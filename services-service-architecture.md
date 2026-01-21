# Services Service - System Architecture Document

## Important Note for Developers

### Integration with Other Services
All variables, flows, and integration patterns mentioned in this document for external services (Billing Service, Referral Service, Booking Service, Notification Service, Wallet Service, etc.) should be integrated based on current code and existing flows in those services. The changes, API endpoints, event schemas, and integration details suggested in this document are for representational purposes only and serve as a reference for:
- Understanding the required functionality
- Identifying integration points
- Clarifying data flow and responsibilities

### Before Implementation
1. Review the actual codebase of each external service
2. Verify existing API endpoints, event schemas, and data structures
3. Adapt the suggestions in this document to match the current implementation
4. Ensure backward compatibility with existing integrations
5. Follow existing patterns and conventions in each service

### Key Integration Points to Verify
- Billing Service: Fee calculation logic, invoice generation, event schemas
- Referral Service: Commission slab lookup APIs, commission tracking, event schemas
- Notification Service: Notification templates, delivery channels, event consumption
- Wallet Service: Hold/release/refund flows, event schemas
- Booking Service: Time-slot metadata integration (if applicable)

This document focuses on the Services Service architecture and assumes integration points exist or will be created in other services. Actual implementation should align with the current architecture and patterns of each service.

## Table of Contents
- Part 1: Overview & High-Level Architecture
  - 1. System Overview
  - 2. High-Level Architecture
  - 3. Request Flow Diagrams (with Outbox)
  - 4. Component Architecture
  - 5. Scalability & Performance Overview
- Part 2: Database Architecture & Data Models
  - 2.1 Modeling Principles
  - 2.2 Collections
- Part 3: API Specifications (with PBAC, Idempotency, Contracts)
  - 3.1 Auth & PBAC
  - 3.2 API Conventions
  - 3.3 Client APIs
  - 3.4 Expert APIs
  - 3.5 Shared & Internal APIs
- Part 4: Event Schemas & Cross-Service Integration
  - 4.1 Event Envelope & Versioning
  - 4.2 Outbox & Idempotency
  - 4.3 Domain Event Flows
  - 4.4 Cross-Service Contracts
- Part 5: Security, Monitoring & Scalability
  - 5.1 Security
  - 5.2 Performance & Scaling
  - 5.3 Observability & SLOs
  - 5.4 Reliability, DR & Runbooks
  - 5.5 UI Requirements & User Experience

## Part 1: Overview & High-Level Architecture

### 1. System Overview

#### 1.1 Purpose
The Services Service is a core microservice that enables experts to monetize their expertise by listing service services (standard and custom) that clients can purchase. It orchestrates the complete lifecycle from:
- Service creation
- Custom request and negotiation
- Order creation and payment hold
- Delivery, dispute window, and payout
- Integration with feedback, wallet, billing, booking, messaging, and notifications

#### 1.2 Technology Stack
- Database: MongoDB 6.0+ (replica set, future sharding)
- Message Queue: RabbitMQ 3.12+ (quorum queues only)
- Workflow Engine: Temporal
- Cache / Rate limiting: Redis 7.0+
- Runtime: Node.js 20+
- API Protocol: REST (primary), OpenAPI 3.1 spec; GraphQL optional
- Time Semantics: All server-side timestamps in UTC (ISO-8601); clients responsible for local conversion
- Currency Semantics: Monetary values stored as integer minor units (e.g., paise) + currency code; rounding performed at boundaries (UI, billing)

#### 1.3 Key Capabilities
- Standard Service creation and management: Experts create services, clients purchase directly
- Client-initiated custom requests: Streamlined custom service request and order flow
  - Client creates custom request (isRequest=true) with optional timeline/cost
  - Expert accepts request and negotiates via free conversation
  - Expert creates custom order (isRequest=false) after negotiation with required timeline/cost
  - Client accepts order to proceed with payment
- Clear separation: clients create requests, experts create orders
- Flexible cancellation: requests can be cancelled/rejected, orders can only be disputed by client or cancelled by expert
- Escrow-like payment integration via Wallet and Billing (hold -> release -> refund)
- Outbox-based event publishing (fixes dual-write risk)
- Temporal-based workflows for:
  - Custom request lifecycle (7 days auto-reject) - time configurable via environment variable CUSTOM_REQUEST_NEGOTIATION_TIMEOUT_DAYS (default: 7)
  - Order lifecycle (payment hold, delivery, dispute window, auto-completion, payout)
  - Dispute resolution workflow
  - Service auto-unpause workflows
- Standard service auto-acceptance: Orders for standard services are automatically accepted by expert upon creation
- Service availability management: Individual services or all services can be paused/unpaused based on expert availability
- Referral commission integration: Commission slabs similar to audio/video calls, with transaction fee handling for 0% platform fee scenarios
- Profile-Based Access Control (PBAC) using Profile Service
- Free order-related conversations: Messages linked to orders/requests are free of charge
- Complete audit trail and analytics
- Robust rate limiting, idempotency, and event versioning

### 2. High-Level Architecture

#### 2.1 Microservice Boundaries (Figure 2.1)
```
+-------------------------------------------------------------+
| API GATEWAY / BFF                                            |
| (AuthN, Routing, Rate Limiting)                             |
+-------------------------------------------------------------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
        +-----------+   +-----------+   +-----------+
        | Client    |   | Expert    |   | Admin     |
        | APIs      |   | APIs      |   | APIs      |
        +-----------+   +-----------+   +-----------+
              |               |               |
              +---------------+---------------+
                              |
                     +-------------------------+
                     | SERVICES SERVICE        |
                     | (Core Business Logic)   |
                     | - Service Management    |
                     | - Custom Request Flow   |
                     | - Order Orchestration   |
                     | - Transactional Outbox  |
                     +-----------+-------------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
        +-----------+   +-----------+   +-----------+
        | MongoDB   |   | Redis     |   | RabbitMQ  |
        | (Domain + |   | (Cache +  |   | (Quorum   |
        | Outbox)   |   | Rate Lim) |   | Queues)   |
        +-----------+   +-----------+   +-----------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
        +-----------+   +-----------+   +-----------+
        | Temporal  |   | External  |   | CDN/S3    |
        | (Workflows|   | Services  |   | (Attachs) |
        +-----------+   +-----------+   +-----------+
```

#### 2.2 External Service Dependencies
Synchronous (REST/gRPC, with timeouts + circuit breakers):
- Auth Service: JWT issuing and verification parameters (iss, aud, alg, kid)
- Profile Service: Client/Expert profile lookup for PBAC (with Redis caching, timeouts, fallback)
- Wallet Service: Check balance, create holds, release funds, refunds
- Billing Service: Invoice creation, financial calculations, commission calculations
- Referral Service: Commission slab lookup and referral tracking for service orders
- Booking Service (optional): Time-slot metadata if service is scheduled
- File/Media Service (if separate): Presigned uploads, AV scan status

Asynchronous (RabbitMQ quorum queues):
- Notification Service: All user-facing alerts
- Messaging Service: Negotiation and chat-related events
- Billing Service: Downstream billing events
- Wallet Service: Hold, refund, payout events
- Feedback Service: Post-completion rating flow (includes cancellation feedback)

Important Integration: Messaging Service must support order-linked conversations
- Add orderId field to conversation schema (optional, blank for normal conversations)
- Conversations with orderId are free (no messaging charges)
- Conversation closes when order is closed/deleted/rejected/completed
- UI should show order link in conversation for easy navigation

### 3. Request Flow Diagrams (with Outbox)

#### 3.1 Standard Service Purchase Flow (Outbox + Temporal)
```
Client            Services Service      MongoDB (Orders+Outbox)   Outbox Worker      RabbitMQ       Wallet          Billing        Notification
  |                      |                        |                    |                |               |               |                |
  |--Create Order------->|                        |                    |                |               |               |                |
  |                      |--Validate & Build----->| (Tx: insert order + outbox event)   |               |               |                |
  |                      |<------OK (orderId)-----|                    |                |               |               |                |
  |<--Order Created------|                        |                    |                |               |               |                |
  |                      |  (Start OrderLifecycleWorkflow in Temporal) |                |               |               |                |
  |                      |                        |                    |                |               |               |                |
  |                      |                    [Outbox Poller]          |                |               |               |                |
  |                      |                        |--read outbox------>|                |               |               |                |
  |                      |                        |                    |--publish------>|               |               |                |
  |                      |                        |                    |                |--event------->|               |                |
  |                      |                        |                    |                |               |--hold funds-->|                |
  |                      |                        |                    |                |               |<--result------|                |
  |                      |<---wallet.hold.* events via RabbitMQ and workflow signals---|               |                |
  |                      |                        |                    |                |               |                |--invoice------>|
  |                      |                        |                    |                |               |                |                |
  |                      |--Notify Expert------------------------------------------------------------->| (Notification) |
  |                      |                        |                    |                |               |               |                |
[Expert delivers service]|                        |                    |                |               |               |                |
  |                      |--Mark Delivered------->| (update order + outbox events)      |               |               |                |
  |                      |                        |                    |--publish------>|               |               |                |
  |                      |                        |                    |                |--notify------>| (client notified)             |
  |                      |                        |                    |                |               |               |                |
[48h dispute window passes via Temporal timer]    |                    |                |               |               |                |
  |                      |--Release Funds-------->| (outbox event)     |                |               |               |                |
  |                      |                        |                    |--publish------>|               |--release----->|                |
  |                      |                        |                    |                |               |                |--update------>|
  |                      |<--order.completed event via workflow signals and events---------------------|                |
  |<--Order Completed----|                        |                    |                |               |               |                |
```

#### 3.2 Custom Service Request & Negotiation Flow
```
Client        Services        MongoDB(customRequests+outbox)   Outbox Worker   Messaging   Expert     Temporal
  |              |                        |                        |             |           |           |
  |--CustomReq-->|                        |                        |             |           |           |
  |              |--create req+outbox---> | (Tx)                   |             |           |           |
  |              |<-----requestId-------- |                        |             |           |           |
  |<--Created----|                        |                        |             |           |           |
  |              | (start CustomRequestWorkflow)                   |             |           |           |
  |              |                        |                        |--event----->|--notify-->|
  |              |                        |                        |             |           |
[Expert reviews/negotiates max 2 rounds]  |                        |             |           |
  |<--msgs via Messaging Service--------->|<-------message events via outbox & MQ---------->|
  |              |                        |                        |             |           |
  |--AcceptOffer->|                       |--create order+update req+outbox (Tx)           |
  |              |                        |                        |--order.created------->|
  |              | (start OrderLifecycleWorkflow, mark req converted)                      |
  |<--Order Id---|                        |                        |             |           |
```

#### 3.3 Auto-Rejection Flow (7 Days Timeout via Temporal)
```
Temporal Workflow    Services        MongoDB(customRequests+outbox)   Outbox Worker   Notification    Client
       |                |                       |                          |             |            |
       |--7d timer----->|                       |                          |             |            |
       |                |--set status+outbox--> | (idempotent: only if still PENDING)    |
       |                |                       |                          |--event----->|--notify--->|
       |<--completed----|                       |                          |             |            |
```

Key Points:
- Client can ONLY create custom requests (isRequest=true), NOT custom orders
- Client can cancel request before expert acceptance
- Expert can reject request before acceptance
- Expert creates the actual order (isRequest=false) after negotiation
- Only expert can modify order details during request phase
- Client accepts the final order to proceed with payment

State Transitions:
```
+-------------------------------------------------------------+
| CUSTOM SERVICE REQUEST & ORDER FLOW                         |
+-------------------------------------------------------------+
CLIENT CREATES REQUEST
|
v
+-----------+ <-----------------+
| PENDING   |                   | Client can CANCEL (CLIENT_CANCELLED)
| isRequest=T                   |
| Status: PENDING               |
+-----+-----+                   |
      |                         |
      | <---- Expert REJECTS ---+
      |      (EXPERT_REJECTED)
      |
      v
+-----------+ ------------------+
| ACCEPTED  |                   |
| isRequest=T                   |
| isExpertAccepted=T            |
| Status: PENDING [7-Day Timer: Auto-Reject]
+-----+-----+                   |
      |                         |
      v                         v
 (AUTO_REJECTED)        [Negotiation Phase]
                        [Free Conversation]
      |
      v
EXPERT CREATES ORDER
      |
      v
+------------------+ <---- CLIENT can DECLINE
| ORDER CREATED    |
| isRequest=F      |
| Status: CREATED (back to negotiation or end)
| Timeline/Cost REQUIRED
+-----+------------+
      |
      | Client ACCEPTS
      v
+--------------------------+
| PAYMENT HOLD             |
| isRequest=F              |
| isClientAccepted=T       |
| Status: PAYMENT_HOLD_PENDING
+-----+--------------------+
      |
      v
+--------------------------+ <---- Expert can CANCEL (EXPERT_CANCELLED + Refund)
| IN PROGRESS              |
| isRequest=F              |
| Status: IN_PROGRESS      | Client CANNOT cancel (only DISPUTE after delivery)
+-----+--------------------+
      |
      v
+-----------+
| DELIVERED |
| Status: DELIVERED
| Dispute Window (48 hours)
+-----+-----+
      |
      +----> DISPUTE_OPENED (if client raises dispute)
      |
      v
+-----------+
| COMPLETED |
| Status: COMPLETED
| Payout Released
+-----------+
```

Status Summary:

| Status | isRequest | Description | Client Actions | Expert Actions |
| --- | --- | --- | --- | --- |
| PENDING | true | Request awaiting expert review | Cancel | Accept / Reject |
| PENDING | true | Expert accepted, negotiating | Cancel | Create Order |
| CREATED | false | Order created, awaiting client | Accept / Decline | Modify Order |
| PAYMENT_HOLD_PENDING | false | Client accepted, payment processing | - | - |
| IN_PROGRESS | false | Payment held, service in progress | Dispute (after delivery) | Deliver / Cancel |
| DELIVERED | false | Service delivered, dispute window | Dispute / Accept | - |
| COMPLETED | false | Order completed, payout released | Rate/Feedback | - |
| CLIENT_CANCELLED | true | Client cancelled request | - | - |
| EXPERT_REJECTED | true | Expert rejected request | - | - |
| EXPERT_CANCELLED | false | Expert cancelled order (refund) | Rate/Feedback | - |
| DISPUTE_OPENED | false | Client opened dispute | Provide Info | Provide Info |
| REFUNDED | false | Order refunded | - | - |

#### 3.4 Standard Service Auto-Acceptance Flow
For standard services (non-custom), orders are automatically accepted by the expert upon creation. If the expert is unavailable, they can cancel and refund the order.
```
Client        Services Service    MongoDB (Orders+Outbox)   Outbox Worker    RabbitMQ    Wallet     Billing    Notification
  |                |                      |                    |               |         |         |           |
  |--Create Order->|                      |                    |               |         |         |           |
  |                |--Validate & Build--->| (Tx: insert order + outbox event)   |         |         |           |
  |                |--Auto-accept (isExpertAccepted=true)      |               |         |         |           |
  |                |<------OK (orderId)--|                    |               |         |         |           |
  |<--Order Created|                      |                    |               |         |         |           |
  |                | (Start OrderLifecycleWorkflow in Temporal) |              |         |         |           |
  |                |                      |                    |               |         |         |           |
[If expert unavailable, can cancel and refund]
  |                |--Cancel & Refund--->| (update order + outbox events)      |         |         |           |
  |                |                      |--publish------------------------>|         |         |           |
  |                |                      |                                  |--refund->|         |           |
  |                |                      |                                  |         |--notify->|           |
```

#### 3.5 Service Pause/Unpause Flow
Experts can pause individual services or all services for a period (e.g., when unavailable for appointments). Paused services cannot receive new orders.
```
Expert     Services Service     MongoDB (Services)     Outbox Worker     RabbitMQ     Notification
  |              |                      |                  |              |             |
  |--Pause Service--------------------->|                  |              |             |
  | (or Pause All)                      |--Update status-->| (update service(s) + outbox event) |
  |              |                      |                  |--publish---->|             |
  |<--Service Paused--------------------|                  |              |             |
  |              |                      |                  |              |             |
[Service unavailable for new orders]
  |              |                      |                  |              |             |
  |--Unpause Service------------------->|                  |              |             |
  |              |--Update status------>| (update service(s) + outbox event) |            |
  |              |                      |--publish-------->|              |             |
  |<--Service Active--------------------|                  |              |             |
```

### 4. Component Architecture
```
services-service/
├── api/
│   ├── controllers/
│   │   ├── service.controller.ts
│   │   ├── customRequest.controller.ts
│   │   ├── order.controller.ts
│   │   └── internal.controller.ts
│   ├── middlewares/
│   │   ├── auth.middleware.ts // JWT validation (iss, aud, alg, kid)
│   │   ├── pbac.middleware.ts // Profile lookup + Redis cache + timeout
│   │   ├── rateLimit.middleware.ts // Redis-based sliding window
│   │   └── validation.middleware.ts // schema validation & sanitization
│   ├── routes/
│   │   ├── client.routes.ts
│   │   ├── expert.routes.ts
│   │   ├── admin.routes.ts
│   │   └── internal.routes.ts
├── domain/
│   ├── entities/
│   │   ├── Service.ts
│   │   ├── CustomRequest.ts
│   │   ├── Order.ts
│   │   └── OutboxEvent.ts
│   ├── repositories/
│   │   ├── ServiceRepository.ts
│   │   ├── CustomRequestRepository.ts
│   │   ├── OrderRepository.ts
│   │   └── OutboxRepository.ts
│   ├── services/
│   │   ├── ServiceService.ts
│   │   ├── CustomRequestService.ts
│   │   ├── OrderService.ts
│   │   └── OutboxService.ts
├── infrastructure/
│   ├── database/
│   │   ├── mongodb.connection.ts
│   │   └── models/*.ts
│   ├── cache/
│   │   └── redis.service.ts
│   ├── messaging/
│   │   ├── rabbitmq.connection.ts
│   │   ├── publishers/ (only used by outbox worker)
│   │   └── consumers/
│   ├── temporal/
│   │   └── workflows/
│   │       ├── OrderLifecycleWorkflow.ts
│   │       ├── CustomRequestWorkflow.ts
│   │       ├── DisputeResolutionWorkflow.ts
│   │       └── ServiceAvailabilityWorkflow.ts // Auto-unpause workflows
│   └── config/
│       └── env.config.ts // Environment variable validation and defaults
├── workers/
│   └── outboxWorker.ts // reads outbox, publishes to RabbitMQ, handles idempotency
├── services/
│   ├── ReferralService.ts // Referral commission calculation
│   └── FeeCalculationService.ts // Platform fee and transaction fee calculation
└── shared/
    ├── constants/
    ├── utils/
    └── types/
```

### 5. Scalability & Performance Overview
- MongoDB:
  - Replica set (3 nodes), sharding planned by { expertId, _id } (compound, to avoid hotspots on single experts).
  - Secondary indexes for client-centric queries.
- RabbitMQ:
  - Only quorum queues, no mirrored queues.
- Redis:
  - Used for cache, PBAC cache, rate limiting. Not in Temporal signaling path.
- Temporal:
  - Used for long-running workflows with timers and signals; payloads kept small (IDs, not large documents).
  - Workflows include:
    - Custom request auto-rejection (7 days timer, configurable)
    - Dispute window expiration (48 hours timer, configurable)
    - Service auto-unpause (when pausedUntil expires)

## Part 2: Database Architecture & Data Models

### 2.1 Modeling Principles
1. Single Source of Truth: Domain entities in Mongo, events emitted via outbox.
2. Bounded Embedded Arrays: negotiation.history and orders.timeline are capped (e.g., last 20 entries). Full history is reconstructable from outbox/event logs and auditLogs.
3. Currency Precision: Store as amountMinor: Number + currency: String (e.g., INR, USD).
4. Versioning and OCC: Every mutable document has version: Number. APIs expose ETag / If-Match to enforce optimistic concurrency.
5. Soft Delete and Audit: isDeleted, deletedAt, plus auditLogs for critical changes.
6. Sharding Strategy (future): Primary shard key candidate: { expertId, _id } for high-cardinality distribution.

### 2.2 Collections

#### 2.2.1 services (categories/verticals)
Unchanged from your original design (service categories / verticals).

Key points:
- slug unique.
- Hierarchical via parentId, ancestors.

Indexes:
- { slug: 1 }
- { parentId: 1 }
- { isDeleted: 1, name: 1 }

#### 2.2.2 services (expert offerings)
```
{
  "_id": ObjectId,
  "expertId": ObjectId,
  "serviceTitle": String,
  "description": String,
  "image": String | null,
  "priceMinor": Number, // amount in minor units (paise)
  "currency": String, // "INR"
  "deliveryTimeDays": Number,
  "isCustomAllowed": Boolean,
  "status": "ACTIVE" | "PAUSED", // service availability status
  "pausedUntil": Date | null, // optional: pause until specific date/time
  "version": Number,
  "isDeleted": Boolean,
  "deletedAt": Date | null,
  "createdAt": Date,
  "updatedAt": Date,
  "createdBy": ObjectId | null,
  "updatedBy": ObjectId | null
}
```

Constraints:
- Unique per expert: (expertId, serviceTitle, isDeleted=false).

Indexes:
- { expertId: 1, isDeleted: 1 }
- { serviceId: 1, isDeleted: 1 }
- { expertId: 1, serviceTitle: 1, isDeleted: 1 } unique
- { expertId: 1, status: 1, isDeleted: 1 }
- { status: 1, pausedUntil: 1 } (for auto-unpause queries)

#### 2.2.3 customRequests
No separate CustomRequest collection required. Custom requests are created as orders with isRequest: true flag.
- Client creates custom service request -> order with isRequest=true, timeline/cost optional
- A new conversation is created for negotiation and discussion
- Custom request can be:
  - Cancelled by client before expert acceptance
  - Rejected by expert before acceptance
  - Converted to order by expert after negotiation (sets isRequest=false)

#### 2.2.4 orders
```
{
  "_id": ObjectId,
  "type": "STANDARD" | "CUSTOM",
  "isRequest": Boolean, // true = custom request (negotiation phase), false = confirmed order
  "expertId": ObjectId,
  "clientId": ObjectId,
  "serviceId": ObjectId,
  "serviceVersionNumber": Number,
  "conversationId": ObjectId | null, // Linked conversation for order-related discussions (free messaging)
  "amount": {
    "grossMinor": Number, // Optional when isRequest=true, required when isRequest=false
    "currency": String,
    "walletHoldId": String | null
  },
  "deliveryTimeDays": Number | null, // Optional when isRequest=true, required when isRequest=false
  "deliveryDeadline": Date | null,
  "proofOfDelivery": [
    {
      "fileKey": String,
      "originalName": String,
      "uploadedAt": Date,
      "mimeType": String
    }
  ],
  "status":
    "PENDING" | // Initial state for custom requests (isRequest=true)
    "CREATED" | // Order confirmed (isRequest=false)
    "PAYMENT_HOLD_PENDING" |
    "PAYMENT_HOLD_FAILED" |
    "IN_PROGRESS" |
    "DELIVERED" |
    "DISPUTE_OPENED" |
    "COMPLETED" |
    "REFUNDED" |
    "CLIENT_CANCELLED" | // Only for custom requests (isRequest=true)
    "EXPERT_REJECTED" | // Expert rejects custom request
    "EXPERT_CANCELLED", // Expert cancels confirmed order
  "timeline": [
    {
      "status": String,
      "timestamp": Date,
      "actorUserId": ObjectId | null,
      "meta": Object
    }
  ],
  "disputeWindowEndsAt": Date | null, // 48 hours after delivery (configurable via DISPUTE_WINDOW_HOURS)
  "payoutReleasedAt": Date | null,
  "workflowId": String | null,
  "isClientAccepted": Boolean, // true when client accepts (auto-true when client creates order); false for custom request creation
  "isExpertAccepted": Boolean, // true when expert accepts (auto-true for STANDARD, set by expert for CUSTOM requests)
  "version": Number,
  "isDeleted": Boolean,
  "deletedAt": Date | null,
  "createdAt": Date,
  "updatedAt": Date,
  "createdBy": ObjectId | null,
  "updatedBy": ObjectId | null
}
```

Notes:

Standard Service Orders:
- Client creates order directly (no custom request needed)
- isRequest = false from creation
- Automatically accepted by expert (isExpertAccepted = true) upon creation and sets isClientAccepted = true
- Expert can cancel and refund if unavailable
- Timeline and cost are required fields

Custom Service Request Flow:
1. Client raises custom service request:
   - Creates order with isRequest = true
   - Timeline and cost are optional at this stage
   - isClientAccepted = false (client is creator)
   - status = PENDING
   - New conversation is created and linked via conversationId
   - Client can cancel the request before expert acceptance
2. Expert reviews and accepts request:
   - Expert can reject the request (status = EXPERT_REJECTED)
   - Expert can accept the request (sets isExpertAccepted = true)
   - Messaging/negotiation can start in linked conversation
3. Negotiation phase:
   - Both parties negotiate via the linked conversation
   - Conversation is free (no charges for order-related messages)
4. Expert creates custom order (after negotiation):
   - Expert fills complete order details (timeline and cost now required)
   - Sets isRequest = false (converts request to confirmed order)
   - status = CREATED
   - Only the expert can create the order (client cannot create custom orders)
5. Client accepts custom order:
   - Client reviews and accepts the order
   - Sets isClientAccepted = true
   - Order proceeds to payment hold and fulfillment

Order Cancellation Rules:
- Custom Requests (isRequest = true):
  - Can be cancelled by client before expert acceptance
  - Can be rejected by expert before acceptance
- Confirmed Orders (isRequest = false):
  - Cannot be cancelled by client - only dispute can be raised
  - Expert can cancel the order and initiate refund
  - If client feels disheartened, they can give rating/feedback

Conversation Management:
- New conversation created for each order/request (linked via conversationId)
- Messaging Service should add orderId field to conversation schema (optional, blank for normal conversations)
- Conversations linked to orders are charge-free
- Conversation closes when order is closed/deleted/rejected/escalated
- In Messaging Service UI, show order link for easy navigation to order details

Dispute Window:
- 48 hours after service is marked delivered (configurable via DISPUTE_WINDOW_HOURS environment variable, default: 48)
- Client is notified upon delivery with clear message that after dispute window, order will be automatically marked completed and no refunds can be requested
- Only applicable to confirmed orders (isRequest = false)

Referral Integration:
- Commission slabs similar to audio/video calls
- When platform fee is 0% (100% commission), a minimum transaction fee (2-3%, configurable via MIN_TRANSACTION_FEE_PERCENTAGE) is still charged to sustain payment gateway costs
- UI should display this clearly when order has 0% platform fee

Bounded timeline:
- Application cap: last 50 entries; full history available in auditLogs and outbox/event logs.

Indexes:
- { expertId: 1, status: 1, isDeleted: 1 }
- { clientId: 1, status: 1, isDeleted: 1 }
- { serviceId: 1, isDeleted: 1 }
- { isRequest: 1, status: 1, isDeleted: 1 } (for filtering requests vs orders)
- { expertId: 1, isRequest: 1, status: 1 } (expert's requests and orders)
- { clientId: 1, isRequest: 1, status: 1 } (client's requests and orders)
- { conversationId: 1 } (link to messaging conversation)
- { status: 1, deliveryDeadline: 1 }
- { disputeWindowEndsAt: 1, status: 1 }
- { isRequest: 1, createdAt: 1 } (for auto-rejection workflow queries)

#### 2.2.5 attachments
As before, with explicit security usage (see Part 5).

#### 2.2.6 serviceStats
As before, fed by Feedback Service events.

#### 2.2.7 auditLogs
Redaction guidance (fixing PII issue):
- Do NOT store full before / after documents.
- Store:
  - changedFields: [String]
  - Optional partial diffs for non-PII fields.
- Never log raw JWTs, email, phone, or file contents.

#### 2.2.8 expertAvailabilitySettings
Optional collection for expert-level availability settings (can pause all services for a period).
```
{
  "_id": ObjectId,
  "expertId": ObjectId,
  "allServicesPaused": Boolean,
  "pausedUntil": Date | null,
  "reason": String | null, // e.g., "Appointment booking", "Temporary unavailability"
  "createdAt": Date,
  "updatedAt": Date
}
```

Indexes:
- { expertId: 1 } unique
- { allServicesPaused: 1, pausedUntil: 1 } (for auto-unpause queries)

#### 2.2.9 outboxEvents
New collection for transactional outbox.
```
{
  "_id": ObjectId,
  "aggregateType": "ORDER" | "CUSTOM_REQUEST" | "SERVICE",
  "aggregateId": ObjectId,
  "eventType": String, // e.g., "services.order.created"
  "schemaVersion": Number,
  "payload": Object, // small, ID-centric payload
  "status": "PENDING" | "PUBLISHED" | "FAILED",
  "retryCount": Number,
  "lastError": String | null,
  "createdAt": Date,
  "updatedAt": Date
}
```

Indexes:
- { status: 1, createdAt: 1 }
- { aggregateType: 1, aggregateId: 1, eventType: 1 }

## Part 3: API Specifications (with PBAC, Idempotency, Contracts)

### 3.0 Custom Service Request & Order Flow Summary

#### 3.0.1 Key Concepts
Custom Request vs Custom Order:
- Custom Request (isRequest=true): Initial negotiation phase where client proposes a custom service to expert. Timeline and cost are optional. Can be cancelled by client or rejected by expert.
- Custom Order (isRequest=false): Confirmed order after negotiation. Timeline and cost are required. Cannot be cancelled by client (only disputed). Can be cancelled by expert with refund.

Key Rules:
1. Client can ONLY create custom requests, NOT custom orders
2. Expert creates the custom order after negotiation is complete
3. Custom requests can be cancelled by client or rejected by expert
4. Custom orders cannot be cancelled by client - only disputes can be raised
5. Expert can cancel custom orders and initiate refunds

#### 3.0.2 Complete Flow
Step 1: Client Creates Custom Request
- Endpoint: POST /client/custom-request
- Creates order with:
  - isRequest = true
  - status = PENDING
  - isClientAccepted = false (auto-set)
  - Timeline and cost are optional
  - New conversation created (conversationId)
- Client can cancel before expert accepts

Step 2: Expert Accepts Request
- Endpoint: POST /expert/custom-request/:orderId/accept
- Updates order:
  - isExpertAccepted = true
  - Still isRequest = true (still in negotiation)
  - status = PENDING
- Messaging/negotiation can start
- Expert can also reject: POST /expert/custom-request/:orderId/reject

Step 3: Negotiation Phase
- Both parties negotiate via linked conversation
- Conversation is free (no messaging charges)
- Expert can modify request details

Step 4: Expert Creates Custom Order
- Endpoint: POST /expert/custom-order/create/:orderId
- Converts request to order:
  - isRequest = false (now a confirmed order)
  - status = CREATED
  - Timeline and cost are now required
  - isExpertAccepted = true (already accepted)
  - isClientAccepted remains as is (waiting for client)

Step 5: Client Accepts Custom Order
- Endpoint: POST /client/custom-order/:orderId/accept
- Confirms order:
  - isClientAccepted = true
  - status = PAYMENT_HOLD_PENDING
- Payment hold initiated
- OrderLifecycleWorkflow starts

Step 6: Order Fulfillment
- Same as standard service orders
- Expert delivers, dispute window, auto-completion, payout

Cancellation and Rejection:
- Custom Request Phase (isRequest=true):
  - Client can cancel: POST /client/custom-request/:orderId/cancel
  - Expert can reject: POST /expert/custom-request/:orderId/reject
- Custom Order Phase (isRequest=false):
  - Client cannot cancel - can only raise dispute
  - Expert can cancel: POST /expert/orders/:orderId/cancel (triggers refund)

#### 3.0.3 Comparison: Standard vs Custom Service Orders

| Aspect | Standard Service Order | Custom Service Request -> Order |
| --- | --- | --- |
| Who creates order? | Client | Client creates request -> Expert creates order |
| Timeline/Cost required from start | Required | Optional in request -> Required in order |
| Expert acceptance | Auto-accepted | Manual acceptance required |
| Negotiation | No | Yes (via free conversation) |
| Client cancellation | Not allowed (only dispute) | Allowed in request phase only |
| Expert cancellation | Allowed (with refund) | Request: Reject / Order: Cancel with refund |
| isRequest flag | false from creation | true -> false after negotiation |

### 3.1 Auth & PBAC

#### 3.1.1 JWT Authentication
Header: Authorization: Bearer <jwt>

JWT validation:
- iss = Auth Service
- aud = services-service
- alg = RS256 (no "none")
- kid -> select public key from JWKS cache
- Token revocation handled by Auth service (token introspection / blacklist)

#### 3.1.2 Profile-Based Access Control (PBAC)
For each request, PBAC middleware:
- Extracts userId from JWT.
- Checks Redis for cached profile capabilities.
- On cache miss, calls Profile API (GET /profile/by-user/:userId) with:
  - Timeout (e.g., 200ms)
  - Circuit breaker (opens on repeated failures)
  - Fallback: last-known capabilities from cache; if none, fail with 403

### 3.2 API Conventions

#### 3.2.1 Standard Response Envelope
Success:
```
{
  "success": true,
  "data": {},
  "error": null,
  "meta": {
    "requestId": "uuid",
    "traceId": "trace-id"
  }
}
```

Error:
```
{
  "success": false,
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid price",
    "details": [{ "field": "price", "message": "Must be > 0" }]
  },
  "meta": {
    "requestId": "uuid",
    "traceId": "trace-id"
  }
}
```

#### 3.2.2 Pagination
Query params: ?limit=20&cursor=<opaque>

Response meta:
```
{
  "meta": {
    "limit": 20,
    "nextCursor": "opaque-or-null",
    "hasMore": true
  }
}
```

#### 3.2.3 Idempotency
For all non-idempotent POST operations (orders, custom requests, negotiation, delivery), support:
- Header: Idempotency-Key: <uuid>
- Server stores mapping of (userId, route, method, idempotencyKey) -> result.
- On duplicate with same payload: return same response.
- On mismatch payload: return 409 CONFLICT.

#### 3.2.4 Optimistic Concurrency
- For resource reads: include ETag header based on version (e.g., "v-5").
- For updates: accept If-Match: "v-5". If mismatch, return 412 PRECONDITION_FAILED.

### 3.3 Client APIs

#### GET /client/services/:expertId
Auth: JWT, PBAC: client profile required.

Query params: serviceId, pagination, sorting (price, rating).

Response: list of services with stats.

#### POST /client/custom-request
Purpose: Client creates a custom service request (NOT a custom order)

Headers:
- Authorization
- Idempotency-Key

Body (validated via JSON Schema):
- expertId (required)
- serviceId (required)
- name (required)
- description (required)
- proposedPriceMinor (optional - can be negotiated)
- proposedDeliveryTimeDays (optional - can be negotiated)
- attachments[] (optional)

Validation:
- Price > 0 (if provided)
- DeliveryTimeDays >= 1 (if provided)
- Description <= 10,000 chars
- Attachments <= configured limit, allowed MIME types only

Flow:
- Validate and PBAC
- Create Mongo transaction:
  - Insert order with isRequest=true, status=PENDING
  - Create new conversation and link via conversationId
  - Set isClientAccepted=false (client is creator)
  - Insert outboxEvents for notifications
- Start CustomRequestWorkflow in Temporal (7-day auto-reject timer)
- Return 201 Created with orderId (not requestId)

Response:
```
{
  "success": true,
  "data": {
    "orderId": "...",
    "conversationId": "...",
    "status": "PENDING",
    "isRequest": true
  }
}
```

#### POST /client/custom-request/:orderId/cancel
Purpose: Client cancels custom request before expert acceptance

Headers:
- Authorization
- Idempotency-Key

Validation:
- Order must have isRequest=true
- Order must be in PENDING status
- Expert must not have accepted yet (isExpertAccepted=false)

Flow:
- Update order status to CLIENT_CANCELLED
- Close linked conversation
- Emit outbox events for notifications
- Return 200 OK

#### POST /client/custom-order/:orderId/accept
Purpose: Client accepts the custom order created by expert

Headers:
- Authorization
- Idempotency-Key

Validation:
- Order must have isRequest=false (expert has converted to order)
- Order must have isExpertAccepted=true
- Order must be in CREATED status
- Timeline and cost must be present (required for orders)

Flow:
- Set isClientAccepted=true
- Update order status to PAYMENT_HOLD_PENDING
- Start OrderLifecycleWorkflow in Temporal
- Emit outbox events for Wallet/Billing (payment hold)
- Calculate referral commission (if applicable) and transaction fees
- If platform fee is 0% (100% commission), apply minimum transaction fee (configurable via MIN_TRANSACTION_FEE_PERCENTAGE, default: 2-3%)
- Return 200 OK with order details

Response:
```
{
  "success": true,
  "data": {
    "orderId": "...",
    "status": "PAYMENT_HOLD_PENDING",
    "isClientAccepted": true,
    "isExpertAccepted": true,
    "amount": {
      "grossMinor": 100000,
      "currency": "INR"
    }
  }
}
```

#### POST /client/orders
Purpose: Client creates a standard service order (NOT custom order)

Important: Clients can ONLY create standard service orders. For custom services, clients must create a custom request (see /client/custom-request endpoint).

Headers:
- Authorization
- Idempotency-Key

Body:
```
{
  "serviceId": "...",
  "expertId": "...",
  "deliveryTimeDays": 7,
  "amount": {
    "grossMinor": 100000,
    "currency": "INR"
  }
}
```

Validation:
- Service must be type=STANDARD
- Service must be status=ACTIVE (not paused)
- All required fields (timeline, cost) must be present

Flow:
- Create order with isRequest=false (confirmed order)
- Auto-set isExpertAccepted=true (standard services auto-accepted)
- Set isClientAccepted=true (client is creator)
- Set status=CREATED
- Create new conversation and link via conversationId
- Start OrderLifecycleWorkflow in Temporal
- Calculate referral commission (if applicable) and transaction fees
- If platform fee is 0% (100% commission), apply minimum transaction fee (configurable via MIN_TRANSACTION_FEE_PERCENTAGE, default: 2-3%)
- Emit outbox events for Wallet/Billing (payment hold)
- Return 201 Created with order details

Response:
```
{
  "success": true,
  "data": {
    "orderId": "...",
    "conversationId": "...",
    "status": "CREATED",
    "isRequest": false,
    "isExpertAccepted": true,
    "isClientAccepted": true
  }
}
```

Note: UI should display transaction fee information when platform fee is 0%.

#### POST /client/orders/:id/dispute
Purpose: Client raises a dispute for a confirmed order

Important: For confirmed orders (isRequest=false), clients CANNOT cancel - they can only raise disputes.

Headers:
- Authorization
- Idempotency-Key

Body:
```
{
  "reason": "SERVICE_QUALITY" | "NOT_AS_DESCRIBED" | "INCOMPLETE" | "OTHER",
  "description": "Detailed explanation...",
  "attachments": []
}
```

Validation:
- Order must be in DELIVERED status
- Dispute window must not have expired (disputeWindowEndsAt > now)
- Description required (min 50 chars, max 2000 chars)

Flow:
- Update order status to DISPUTE_OPENED
- Create dispute record
- Start DisputeResolutionWorkflow in Temporal
- Emit outbox events for notifications to expert and admin
- Return 200 OK

### 3.4 Expert APIs

#### POST /expert/custom-request/:orderId/accept
Purpose: Expert accepts a custom service request from client

Headers:
- Authorization
- Idempotency-Key

Validation:
- Order must have isRequest=true
- Order must be in PENDING status
- Expert must not have accepted yet (isExpertAccepted=false)

Flow:
- Set isExpertAccepted=true
- Order remains in PENDING status (still a request, not an order)
- Messaging can start in linked conversation
- Emit outbox events for notifications to client
- Return 200 OK

Response:
```
{
  "success": true,
  "data": {
    "orderId": "...",
    "status": "PENDING",
    "isRequest": true,
    "isExpertAccepted": true,
    "conversationId": "..."
  }
}
```

#### POST /expert/custom-request/:orderId/reject
Purpose: Expert rejects a custom service request from client

Headers:
- Authorization
- Idempotency-Key

Validation:
- Order must have isRequest=true
- Order must be in PENDING status

Flow:
- Update order status to EXPERT_REJECTED
- Close linked conversation
- Emit outbox events for notifications to client
- Return 200 OK

#### POST /expert/custom-order/create/:orderId
Purpose: Expert creates the actual custom order after negotiation (converts request to order)

Important: Only expert can create custom orders. This endpoint converts a custom request (isRequest=true) to a confirmed order (isRequest=false).

Headers:
- Authorization
- Idempotency-Key

Body (all fields REQUIRED):
```
{
  "amount": {
    "grossMinor": 100000,
    "currency": "INR"
  },
  "deliveryTimeDays": 7,
  "description": "Final agreed terms...",
  "attachments": []
}
```

Validation:
- Order must have isRequest=true (still a request)
- Order must have isExpertAccepted=true (expert must have accepted request first)
- Order must be in PENDING status
- All required fields must be present (amount, timeline)
- Amount must be > 0
- DeliveryTimeDays must be >= 1

Flow:
- Update order: set isRequest=false (convert to confirmed order)
- Update order: set all order details (amount, timeline, etc.)
- Update status to CREATED
- Keep isExpertAccepted=true, isClientAccepted remains (waiting for client acceptance)
- Emit outbox events for notifications to client (client needs to accept)
- Return 200 OK

Response:
```
{
  "success": true,
  "data": {
    "orderId": "...",
    "status": "CREATED",
    "isRequest": false,
    "isExpertAccepted": true,
    "isClientAccepted": false,
    "amount": {
      "grossMinor": 100000,
      "currency": "INR"
    },
    "deliveryTimeDays": 7
  }
}
```

#### POST /expert/orders/:id/deliver
Validates attachments (size, type, AV scan status).
Updates orders + outbox.
Sets disputeWindowEndsAt to current time + DISPUTE_WINDOW_HOURS (default: 48 hours).
Triggers notification to client with clear message about dispute window and auto-completion.
Return new ETag.

#### POST /expert/services/:id/pause
Headers: Authorization, If-Match

Body: { "pausedUntil": Date | null } (optional: pause until specific date/time)

Updates service status to PAUSED + outbox event.
If pausedUntil is provided, schedules auto-unpause via Temporal workflow.

#### POST /expert/services/:id/unpause
Headers: Authorization, If-Match

Updates service status to ACTIVE + outbox event.

#### POST /expert/services/pause-all
Headers: Authorization

Body: { "pausedUntil": Date | null, "reason": String | null }

Updates expertAvailabilitySettings.allServicesPaused = true + pauses all active services + outbox events.
If pausedUntil is provided, schedules auto-unpause via Temporal workflow.

#### POST /expert/services/unpause-all
Headers: Authorization

Updates expertAvailabilitySettings.allServicesPaused = false + unpauses all paused services + outbox events.

#### POST /expert/orders/:id/cancel
Purpose: Expert cancels a confirmed order and initiates refund

Important: Expert can cancel confirmed orders (isRequest=false). For custom requests (isRequest=true), expert should use the /reject endpoint instead.

Headers:
- Authorization
- Idempotency-Key

Body:
```
{
  "reason": "UNAVAILABLE" | "UNABLE_TO_DELIVER" | "OTHER",
  "description": "Explanation for cancellation..."
}
```

Validation:
- Order must have isRequest=false (confirmed order)
- Status allows cancellation: CREATED, PAYMENT_HOLD_PENDING, IN_PROGRESS
- Cannot cancel if status is DELIVERED, COMPLETED, DISPUTE_OPENED

Flow:
- Update order status to EXPERT_CANCELLED
- Trigger refund via Wallet Service
- Close linked conversation (optional)
- Emit outbox events for:
  - Wallet (refund)
  - Billing (update invoice)
  - Notifications (client notification)
  - Feedback Service (client can provide rating/feedback about cancellation)
- Return 200 OK

Response:
```
{
  "success": true,
  "data": {
    "orderId": "...",
    "status": "EXPERT_CANCELLED",
    "refundAmount": {
      "grossMinor": 100000,
      "currency": "INR"
    },
    "cancellationReason": "UNAVAILABLE"
  }
}
```

Note:
- For standard services, expert can cancel and refund if unavailable
- For custom services, expert can cancel anytime before completion
- If client feels disheartened by cancellation, they can provide rating/feedback through Feedback Service
- Repeated cancellations by expert may affect their rating and reputation

### 3.5 Shared & Internal APIs
Internal callbacks (Wallet, Billing) secured with:
- x-service-signature (HMAC of body + timestamp)
- x-service-timestamp
- x-service-id

Replay protection:
- Reject if timestamp older than e.g. 5 minutes.
- nonce stored in Redis for short TTL to prevent re-use.

## Part 4: Event Schemas & Cross-Service Integration

### 4.1 Event Envelope & Versioning
All events share this canonical envelope:
```
{
  "id": "uuid",
  "eventType": "services.order.created",
  "schemaVersion": 1,
  "occurredAt": "2025-01-01T00:00:00Z",
  "source": "services-service",
  "traceId": "otel-trace-id",
  "correlationId": "wf-or-order-or-request",
  "payload": {}
}
```

Versioning Policy:
- schemaVersion incremented on breaking changes.
- Consumers must support at least 2 versions during migration.
- Only additive changes allowed without version bump.

### 4.2 Outbox & Idempotency
Outbox worker:
- Polls outboxEvents where status=PENDING.
- Publishes to RabbitMQ quorum queues.
- Marks as PUBLISHED if ACK'd.
- On errors: increments retryCount, with exponential backoff; moves to FAILED for manual intervention after N attempts.

Consumers (other services) must be idempotent:
- Use id as idempotency key.
- Store processed IDs in their own dedup table/Redis to avoid reprocessing.

### 4.3 Domain Event Flows

#### Order Creation -> Billing / Wallet
services.order.created (to Billing + Wallet + Referral):
```
"payload": {
  "orderId": "...",
  "expertId": "...",
  "clientId": "...",
  "serviceId": "...",
  "amountMinor": 75000,
  "currency": "INR",
  "type": "STANDARD",
  "source": "SERVICES"
}
```

Notes:
- Referral Information: If client was referred, referral service contains details and commission slab. Billing Service should apply commission before platform fee calculation.
- Transaction Fee: When platformFeePercentage is 0% (100% commission scenario), requiresTransactionFee is true and transactionFeePercentage contains the minimum transaction fee (default: 2-3%, configurable via MIN_TRANSACTION_FEE_PERCENTAGE). Billing Service must apply this transaction fee.

Fee Calculation Order:
1. Apply referral commission (if exists)
2. Apply platform fee
3. Apply transaction fee on total order amount (if platform fee is 0%)

Wallet/Billing respond with events:
- billing.payment.hold.success
- billing.payment.hold.failed
- wallet.hold.success

All mapped to workflow signals on OrderLifecycleWorkflow.

#### Custom Request Events
Custom Request Lifecycle:
1. services.custom_request.created
   - Client creates custom request (order with isRequest=true)
   - Payload: { orderId, clientId, expertId, serviceId, conversationId, proposedAmount?, proposedDeliveryDays? }
2. services.custom_request.accepted_by_expert
   - Expert accepts the custom request (isExpertAccepted=true)
   - Payload: { orderId, expertId, clientId, acceptedAt }
3. services.custom_request.rejected_by_expert
   - Expert rejects the custom request
   - Payload: { orderId, expertId, clientId, rejectionReason, rejectedAt }
4. services.custom_request.cancelled_by_client
   - Client cancels custom request before expert acceptance
   - Payload: { orderId, clientId, expertId, cancelledAt }
5. services.custom_order.created_by_expert
   - Expert creates the actual order after negotiation (converts isRequest=true to isRequest=false)
   - Payload: { orderId, expertId, clientId, amountMinor, currency, deliveryTimeDays, createdAt }
6. services.custom_order.accepted_by_client
   - Client accepts the custom order created by expert
   - Payload: { orderId, clientId, expertId, amountMinor, currency, acceptedAt }
7. services.custom_request.auto_rejected
   - Triggered after 7 days timeout (configurable via CUSTOM_REQUEST_NEGOTIATION_TIMEOUT_DAYS)
   - Payload: { orderId, clientId, expertId, autoRejectedAt, timeoutDays }

Each event only carries IDs and deltas, not the full document.

#### Service Availability Events
- services.service.paused (individual or all services)
- services.service.unpaused (individual or all services)
- services.service.auto_unpaused (when pausedUntil expires)

#### Order Delivery & Dispute Window Events
services.order.delivered (includes disputeWindowEndsAt timestamp):
```
{
  "id": "uuid",
  "eventType": "services.order.delivered",
  "schemaVersion": 1,
  "occurredAt": "2025-01-01T00:00:00Z",
  "source": "services-service",
  "payload": {
    "orderId": "order-id",
    "clientId": "client-id",
    "expertId": "expert-id",
    "serviceTitle": "Service Title",
    "deliveredAt": "2025-01-01T00:00:00Z",
    "disputeWindowEndsAt": "2025-01-03T00:00:00Z",
    "disputeWindowHours": 48,
    "orderLink": "https://app.example.com/orders/order-id",
    "disputeLink": "https://app.example.com/orders/order-id/dispute"
  }
}
```

Notification Requirements:
- Notification Service must send email, SMS, and in-app notifications.
- Email must include clear messaging about 48-hour dispute window.
- Must emphasize: "After dispute window expires, order will be automatically marked completed and no refunds can be requested."
- Include countdown timer information in UI.

services.order.dispute_window_expired (triggered after 48 hours, configurable via DISPUTE_WINDOW_HOURS):
```
{
  "id": "uuid",
  "eventType": "services.order.dispute_window_expired",
  "schemaVersion": 1,
  "occurredAt": "2025-01-03T00:00:00Z",
  "source": "services-service",
  "payload": {
    "orderId": "order-id",
    "clientId": "client-id",
    "expertId": "expert-id",
    "deliveredAt": "2025-01-01T00:00:00Z",
    "disputeWindowEndsAt": "2025-01-03T00:00:00Z"
  }
}
```

services.order.completed (auto-completed after dispute window expires):
```
{
  "id": "uuid",
  "eventType": "services.order.completed",
  "schemaVersion": 1,
  "occurredAt": "2025-01-03T00:00:00Z",
  "source": "services-service",
  "payload": {
    "orderId": "order-id",
    "clientId": "client-id",
    "expertId": "expert-id",
    "completedAt": "2025-01-03T00:00:00Z",
    "completedReason": "DISPUTE_WINDOW_EXPIRED",
    "payoutReleasedAt": "2025-01-03T00:00:00Z"
  }
}
```

Notification Requirements:
- Client must be notified that order is completed.
- Must state: "Payment has been released to expert. No refunds can be requested."
- Expert must be notified of payout release.

### 4.4 Cross-Service Contracts

#### 4.4.1 Services Service -> Billing Service Contract
Event: services.order.created

Responsibilities of Billing Service:
1. Receive order creation event with referral and fee information
2. Calculate fees in this order:
   - If referral exists: Apply commission first
   - Calculate platform fee (on amount after commission)
   - If platform fee is 0%: Apply transaction fee (2-3%, configurable)
3. Create invoice with complete fee breakdown
4. Publish events:
   - billing.invoice.created (includes all fees)
   - billing.commission.applied (if referral exists)
   - billing.transaction_fee.applied (if transaction fee applied)

Fee Calculation Logic (Billing Service):
Update Billing Service accordingly.

#### 4.4.2 Services Service -> Referral Service Contract
API Call: Call relevant APIs.

Responsibilities of Referral Service:
1. Lookup commission slab for client's referral (if exists)
2. Return commission details:
   - referrerId: ID of person who referred the client
   - commissionSlab: Tier/slab (e.g., "TIER_1")
   - commissionPercentage: Commission percentage for this slab
   - tierDetails: Additional tier information
3. Track commission when order is created
4. Mark commission as earned when order is completed (after dispute window)
5. Reverse commission if order is refunded

Events to Consume:
- services.order.created: Track commission
- services.order.completed: Mark commission as earned
- services.order.refunded: Reverse commission

Events to Publish:
- referral.commission.calculated: When commission is calculated
- referral.commission.earned: When commission is earned
- referral.commission.reversed: When commission is reversed

#### 4.4.3 Services Service -> Notification Service Contract
Events Published:
- services.order.created
- services.order.delivered (critical - requires immediate notification)
- services.order.dispute_window_expiring (reminders at 24h, 12h, 1h)
- services.order.completed
- services.order.cancelled
- services.custom_request.auto_rejected

Responsibilities of Notification Service:
1. Receive events via RabbitMQ
2. Send notifications via multiple channels:
   - Email (with detailed content)
   - SMS (for critical events)
   - In-app notifications
   - Push notifications (mobile app)
3. Format messages according to templates
4. Handle notification preferences (but cannot disable critical notifications)
5. Track delivery status

Critical Notifications (cannot be disabled):
- Service delivered (with dispute window information)
- Dispute window expiration warnings
- Order completion
- Payment/refund updates

#### 4.4.4 Services Service -> Messaging Service Contract
Purpose: Integrate order-linked conversations for custom request negotiation and order discussions.

Required Changes in Messaging Service:
1. Conversation Schema Update:
   - Add optional orderId field to conversation schema:
```
{
  "_id": ObjectId,
  "participants": [ObjectId],
  "orderId": ObjectId | null, // NEW: Link to order (null for normal conversations)
  "isFree": Boolean, // NEW: true if linked to order (no charges)
  "closedReason": String | null, // NEW: "ORDER_COMPLETED", "ORDER_CANCELLED", etc.
  // ... existing fields ...
}
```

2. Conversation Creation:
   - When Services Service creates an order/request, it should:
     - Create conversation via Messaging Service API: POST /messaging/conversations/create
     - Provide: { participants: [clientId, expertId], orderId: "...", isFree: true }
     - Receive: { conversationId: "..." }
   - Store conversationId in order document

3. Conversation Charging Logic:
   - Messaging Service must check isFree flag:
     - If isFree=true (order-linked): No charges for messages
     - If isFree=false (normal): Apply regular messaging charges

4. Conversation Closure:
   - Services Service publishes events when order closes:
     - services.order.completed
     - services.order.cancelled
     - services.order.expert_cancelled
     - services.custom_request.rejected
     - services.custom_request.cancelled
   - Messaging Service should consume these events and:
     - Close the conversation (set status=CLOSED)
     - Set closedReason based on event type
     - Optionally notify participants that conversation is closed

5. UI/UX Requirements:
   - Show order link in conversation: "This conversation is linked to [Order #12345]"
   - Show badge: "Free Conversation - Order Related"
   - Show order status in conversation header
   - Provide "View Order" button for easy navigation
   - Show notification when conversation closes due to order closure

6. API Endpoints Required:
```
// Create order-linked conversation
POST /messaging/conversations/create
Body: {
  participants: [ObjectId],
  orderId: ObjectId,
  isFree: boolean
}

// Close conversation when order closes
POST /messaging/conversations/:conversationId/close
Body: {
  closedReason: "ORDER_COMPLETED" | "ORDER_CANCELLED" | "ORDER_REJECTED"
}

// Check if conversation is order-linked
GET /messaging/conversations/:conversationId/metadata
Response: {
  conversationId: ObjectId,
  orderId: ObjectId | null,
  isFree: boolean,
  orderStatus: string | null
}
```

7. Events to Consume:
   - services.order.created: Show order created notification in conversation
   - services.order.completed: Close conversation
   - services.order.cancelled: Close conversation
   - services.custom_request.accepted_by_expert: Notify in conversation
   - services.custom_order.created_by_expert: Notify in conversation

8. Events to Publish:
   - messaging.conversation.created: When order-linked conversation is created
   - messaging.conversation.closed: When conversation is closed due to order closure
   - messaging.message.sent: Regular message events (even for free conversations, for audit)

#### 4.4.5 Services Service -> Feedback Service Contract
Purpose: Collect feedback when expert cancels order and client feels disheartened.

Required Changes in Feedback Service:
1. Feedback Types:
   - Add new feedback type for order cancellations: ORDER_CANCELLATION
2. Events to Consume:
   - services.order.expert_cancelled: Trigger feedback request to client
3. API Endpoints:
```
// Create cancellation feedback request
POST /feedback/cancellation-feedback
Body: {
  orderId: ObjectId,
  clientId: ObjectId,
  expertId: ObjectId,
  cancellationReason: string
}

// Submit cancellation feedback
POST /feedback/cancellation-feedback/:feedbackId/submit
Body: {
  rating: number, // 1-5
  comment: string,
  wouldRecommend: boolean
}
```

4. UI Requirements:
   - Show feedback form when order is cancelled by expert
   - Ask questions like:
     - "How disappointed were you with this cancellation?" (1-5)
     - "Was the cancellation reason clear and justified?"
     - "Would you work with this expert again?"
   - Free text feedback
   - Display feedback in expert's profile (with context: "Cancellation feedback")

#### 4.4.6 General Contract Principles
- Made everything ID-based.
- Added schemaVersion.
- Added idempotency requirements.
- Clarified Wallet/Billing callback security.
- All services must be idempotent when processing events.
- Services must support at least 2 schema versions during migration.
- Order-linked conversations are free and close when order closes.
- Cancellation feedback helps maintain service quality and expert accountability.

## Part 5: Security, Monitoring & Scalability

### 5.1 Security

#### 5.1.1 JWT
- Validate alg, iss, aud, exp, nbf.
- Use JWKS with kid.
- Reject tokens with "none" or wrong alg.

#### 5.1.2 Service-to-Service Auth
- Use signed service tokens with:
  - iss = "platform"
  - aud = "services-service"
  - short exp (e.g., 5-15 minutes)
- Rotate signing keys via KMS.
- Use scopes/claims to restrict access.

#### 5.1.3 Webhook/Callback Security
As described above with HMAC, timestamp, nonce.

#### 5.1.4 Attachments Security
Upload flow:
- Client gets presigned URL from File Service.
- Uploads file directly to storage.
- File Service runs AV scan (e.g., ClamAV) asynchronously.
- Only after scanStatus=PASSED can Services Service attach file to Order/Request.
- MIME allowlist (e.g., images, PDFs, ZIP).
- Max size per file and per request.

#### 5.1.5 PII Redaction
- Logging: never log raw body for sensitive endpoints.
- auditLogs store only field names and safe deltas.
- Use centralized logger with redaction rules.

### 5.2 Performance & Scaling

#### 5.2.1 Configuration & Environment Variables
Key environment variables for service configuration:
- CUSTOM_REQUEST_NEGOTIATION_TIMEOUT_DAYS: Custom request negotiation timeout in days (default: 7)
- DISPUTE_WINDOW_HOURS: Dispute window duration in hours after service delivery (default: 48)
- MIN_TRANSACTION_FEE_PERCENTAGE: Minimum transaction fee percentage when platform fee is 0% (default: 2.5, range: 2-3%)
- REFERRAL_COMMISSION_ENABLED: Enable/disable referral commission for services (default: true)

#### 5.2.2 Mongo Connection Pooling
Pool size per pod tuned:
- Example: maxPoolSize=50 per app instance.
- HPA limit and replica count used to cap total connections.
- Use connection pooling at app layer, not per request.

#### 5.2.3 Rate Limiting (Complete)
Redis sliding window per:
- userId
- IP address (normalized via X-Forwarded-For)

Limits:
- Custom requests: 5/min/user
- Negotiation messages: 20/min/user
- Cancellations: 3/hour/user
- Global fallback limit per IP (/min) to prevent abuse

### 5.3 Observability & SLOs

#### 5.3.1 SLOs
- Availability: 99.9% for core APIs.
- Latency:
  - p95 < 300ms for read APIs.
  - p95 < 600ms for write APIs (excluding external dependencies).
- Error rate:
  - 5xx < 0.5% over 30-minute windows.

#### 5.3.2 Alerts
- High 5xx rate alarm.
- Increased payment failures alarm.
- Workflow backlog alarm (Temporal).
- RabbitMQ queue depth alarm.

#### 5.3.3 Runbooks
For each alert: a documented runbook with:
- Symptoms
- Possible causes
- Diagnostics commands/queries
- Rollback/mitigation steps

### 5.4 Reliability, DR & Runbooks

#### 5.4.1 DR Objectives
- RPO: <= 15 minutes.
- RTO: <= 60 minutes.

#### 5.4.2 DR Strategy
MongoDB:
- Backup schedule (daily full, oplog streaming).
- Cross-region replica.

RabbitMQ:
- Cluster replication + snapshotting.

Temporal:
- Multi-cluster configuration.

Regular DR drills:
- At least quarterly restore tests with documented results.

### 5.5 UI Requirements & User Experience

#### 5.5.1 Order Creation & Confirmation UI
For Standard Service Orders:
1. Order Creation Page:
   - Display service details (title, description, price, delivery time)
   - Show service availability status (ACTIVE/PAUSED)
   - If service is paused, show clear message: "This service is currently paused and not accepting new orders"
   - Display expert information and ratings
2. Order Summary Before Confirmation:
   - Order Amount: Gross amount in major currency units (e.g., INR 750.00)
   - Platform Fee: Display platform fee percentage and amount
   - If platform fee is 0% (100% commission scenario):
     - Show prominent notice: "Platform Fee: 0% (100% commission to expert)"
     - Transaction Charge Notice (highlighted):
       - "A minimum transaction charge of [X]% ([amount]) will be applied to cover payment gateway costs. This charge is necessary to sustain payment processing operations."
   - Show breakdown:
     - Order Amount: INR 750.00
     - Platform Fee: INR 0.00 (0%)
     - Transaction Charge: INR 18.75 (2.5%) - highlighted
     - Net Amount to Expert: INR 731.25
     - Referral Commission (if applicable):
       - "Referral Commission: [X]% ([amount])"
       - Display commission slab/tier if visible to user
     - Total Amount: Amount that will be held from client's wallet
   - Auto-Acceptance Notice: For standard services, show: "This order will be automatically accepted by the expert"
3. Order Confirmation Page:
   - Show order ID
   - Display all fee breakdowns clearly
   - Show expected delivery date
   - Link to order details page
   - Show conversation link (order-related conversations are free)

For Custom Service Requests and Orders:
Phase 1: Client Creates Custom Request
1. Custom Request Creation Page:
   - Show custom request form with:
     - Service description (required)
     - Proposed price (optional - can be discussed)
     - Proposed delivery time (optional - can be discussed)
     - Attachments (optional)
   - Show message: "Timeline and cost are optional at this stage. You can negotiate with the expert."
   - Display negotiation timeout: "You have 7 days (configurable) to complete negotiations"
   - Show "Submit Request" button
2. Custom Request Confirmation:
   - Show request ID (orderId)
   - Display status: "PENDING - Waiting for Expert Review"
   - Show conversation link (free messaging)
   - Show "Cancel Request" button (available until expert accepts)
   - Display countdown timer for 7-day auto-rejection

Phase 2: Expert Reviews Request
3. Expert View - Custom Request:
   - Show client's request details
   - Display proposed price/timeline (if provided)
   - Show conversation link for discussion
   - Show two action buttons:
     - "Accept Request" (green) - allows negotiation to proceed
     - "Reject Request" (red) - declines the request
   - If accepted: show "Create Order" button (after negotiation)

Phase 3: Negotiation
4. Negotiation Phase UI (Both Client and Expert):
   - Show request status: "ACCEPTED - In Negotiation"
   - Display conversation interface (free messaging)
   - Show badge: "Free Conversation - Order Related"
   - Show negotiation countdown timer (7 days)
   - For client: Show "Cancel Request" button
   - For expert: Show "Create Order" button (when ready to finalize)

Phase 4: Expert Creates Order
5. Expert - Create Custom Order Form:
   - Show all request details from negotiation
   - Require final details (all required now):
     - Final price (required)
     - Delivery timeline (required)
     - Service description
   - Show breakdown of fees (platform fee, referral commission, transaction fee if applicable)
   - Show "Create Order" button
   - Display message: "Client will need to accept this order before payment is processed"
6. Expert - Order Created Confirmation:
   - Show status: "CREATED - Waiting for Client Acceptance"
   - Display all order details
   - Show conversation link
   - Cannot cancel at this stage (order is now firm)

Phase 5: Client Accepts Order
7. Client - Review Custom Order:
   - Show notification: "Expert has created your custom order"
   - Display complete order details:
     - Service title and description
     - Final price breakdown (with all fees)
     - Delivery timeline
     - Expert information
   - Show comparison (if applicable): Original request vs Final order
   - Show "Accept Order" and "Decline" buttons
   - Display message: "Once you accept, payment will be held and order cannot be cancelled (only disputes allowed)"
8. Client - Order Acceptance Confirmation:
   - Show order accepted confirmation
   - Display payment hold information
   - Show expected delivery date
   - Show conversation link (continues to be free)
   - Remove "Cancel" button (now can only raise dispute after delivery)
   - Show "Raise Dispute" button (only after delivery)

Status Indicators Throughout Flow:
- PENDING (yellow): Custom request awaiting expert review
  - Client can cancel
  - Expert can accept/reject
- ACCEPTED (blue): Expert accepted, negotiation in progress
  - Still a request (isRequest=true)
  - Both parties can negotiate
  - Client can cancel
  - Expert can create order
- CREATED (orange): Expert created order, waiting for client acceptance
  - Now an order (isRequest=false)
  - Client must accept
  - No cancellation allowed (firm commitment)
- PAYMENT_HOLD_PENDING (purple): Client accepted, payment processing
  - Order confirmed
  - Payment being held
  - Cannot cancel (only dispute after delivery)
- IN_PROGRESS (green): Payment held, expert working on delivery
  - Normal order flow continues
  - Expert can cancel (with refund)
  - Client cannot cancel (only dispute after delivery)

Custom Request Timeline View:
- Display visual timeline showing:
  1. Request Created
  2. Expert Accepted
  3. Negotiation
  4. Order Created by Expert
  5. Client Accepted
  6. Payment Hold
  7. In Progress
  8. Delivered
  9. Completed

Cancellation & Rejection UI:
- Client Cancel Request (before expert creates order):
  - Show confirmation dialog
  - Display message: "Are you sure you want to cancel this request? This action cannot be undone."
  - Reason dropdown (optional)
  - Confirm/Cancel buttons
- Expert Reject Request:
  - Show rejection reason form
  - Display message to client: "Expert has declined your request. You may create a new request or contact support."
- Expert Cancel Order (after order created):
  - Show cancellation reason form
  - Display refund information
  - Notify client: "Expert has cancelled your order. Full refund will be processed."
  - Allow client to rate/feedback: "Share your experience with this cancellation"

#### 5.5.2 Service Delivery & Dispute Window UI
When Expert Marks Service as Delivered:
1. In-App Notification (Real-time):
   - Show prominent notification banner/toast:
     - "Your service has been delivered!
       You have 48 hours (configurable) to review and raise a dispute if needed.
       IMPORTANT: After the dispute window expires, this order will be automatically marked as completed and no refunds can be requested."
   - Include "View Order" button
   - Include "Raise Dispute" button (available during dispute window only)
2. Order Details Page - Post Delivery:
   - Status Badge: "DELIVERED" with visual indicator
   - Dispute Window Countdown Timer:
     - Large, prominent countdown showing: "Time remaining: [X] hours [Y] minutes"
     - Visual progress bar showing time elapsed
     - Color coding: Green -> Yellow -> Red as time approaches expiration
   - Dispute Window Information Box:
     - "Dispute Window: 48 hours (configurable)
       You can raise a dispute if:
       - Service quality doesn't match description
       - Delivery doesn't meet requirements
       - Any other valid concerns
       After 48 hours, this order will be automatically completed and payment will be released to the expert. No refunds can be requested after completion."
   - Action Buttons:
     - "Raise Dispute" (prominent, available during window)
     - "Mark as Satisfied" (optional, can complete early)
     - "View Delivery Proof" (if attachments provided)
   - Delivery Proof Section: Display all proof of delivery files
3. Email Notification (via Notification Service):
   - Subject: "Your service has been delivered - Review within 48 hours"
   - Body should include:
     - Service title and order ID
     - Delivery confirmation message
     - Clear dispute window information (48 hours)
     - Auto-completion warning
     - Direct link to order details page
     - Link to raise dispute
4. SMS Notification (via Notification Service):
   - "Your service [Service Title] has been delivered. Review within 48 hours. [Order Link]"
5. Dispute Window Expiration Warning:
   - 24 hours before expiration: Show reminder notification
   - 12 hours before expiration: Show urgent reminder
   - 1 hour before expiration: Final warning notification
   - All reminders should emphasize: "No refunds after completion"
6. After Dispute Window Expires:
   - Status Change: Order status changes to "COMPLETED"
   - Notification to Client:
     - "Order Completed
       The dispute window has expired. This order has been automatically marked as completed. Payment has been released to the expert.
       If you have concerns, please contact support."
   - UI Changes:
     - Remove dispute button
     - Show "Order Completed" badge
     - Display completion timestamp
     - Show payment released information

#### 5.5.3 Transaction Charge Explanation (Detailed)
Understanding Transaction Charges:
When Platform Fee is 0% (100% Commission Scenario):
1. Why Transaction Charge Exists:
   - Payment gateways (Razorpay, Stripe, etc.) charge processing fees (typically 2-3%)
   - When platform fee is 0%, there's no platform revenue to cover gateway costs
   - Transaction charge ensures payment gateway costs are covered
   - This is a standard practice in payment processing
2. How It Works:
   - Normal Scenario (Platform Fee > 0%):
     - Order Amount: INR 1000
     - Platform Fee: INR 200 (20%)
     - Gateway costs covered from platform fee
     - Transaction Charge: INR 0
     - Net to Expert: INR 800 - GST etc (This is for just platform fees scenario. Other charges etc are as existing)
   - 100% Commission Scenario (Platform Fee = 0%):
     - Order Amount: INR 1000
     - Platform Fee: INR 0 (0%)
     - Transaction Charge: INR 25 (2.5%, configurable via MIN_TRANSACTION_FEE_PERCENTAGE)
     - Gateway costs covered by transaction charge
     - Net to Expert: INR 975 - GST etc (This is for just platform fees scenario. Other charges etc are as existing)
3. UI Display Requirements:
   - On Order Creation Page:
     - Show detailed breakdown in a collapsible section
     - Include explanation tooltip: "Transaction charge covers payment gateway processing fees"
     - Show calculation: "Transaction Charge = Order Amount x [X]%"
4. Transparency Requirements:
   - Transaction charge must be visible before order confirmation
   - Cannot be hidden in fine print
   - Must be explained in user-friendly language
   - Should be consistent across all UI screens (order creation, confirmation, invoice, order details)

#### 5.5.4 Service Availability Status UI
Individual Service Status:
1. Service Listing Page:
   - Active Service: Normal display with "Available" badge
   - Paused Service:
     - Show "Paused" badge (orange/yellow)
     - Disable "Order Now" button
     - Show message: "This service is currently paused"
     - If pausedUntil is set: "Will be available again on [Date]"
     - Show countdown if resuming soon
2. Expert Profile/Service Page:
   - Show all services with status indicators
   - Group paused services separately (optional)
   - Show pause reason if provided by expert

All Services Paused:
1. Expert Dashboard:
   - Show banner: "All your services are currently paused"
   - Display pause reason and resume date (if set)
   - Show "Unpause All Services" button
2. Client View:
   - When viewing expert's services:
     - Show banner: "This expert has temporarily paused all services"
     - Display resume date if available
     - Option to get notified when services resume
3. Auto-Unpause Notification:
   - When pausedUntil expires and services auto-unpause:
     - Notify expert: "Your services have been automatically resumed"
     - Optionally notify clients who had services in cart/wishlist

#### 5.5.5 Referral Commission Display UI
1. Order Details Page:
   - Show referral information in a dedicated section
   - Display: "Referred by: [Referrer Name/ID]" (if visible)
   - Show commission slab/tier
   - Show commission amount
2. Expert Dashboard:
   - Show total referral commissions earned
   - Breakdown by commission tier
   - Link to referral program details

#### 5.5.6 Order Status & Timeline UI
1. Timeline View:
   - Show order timeline with status changes
   - Display timestamps in user's local timezone
   - Show actor (client/expert/system) for each change
   - Highlight important milestones (delivery, completion)

#### 5.5.7 Notification Preferences UI
1. Critical Notifications (cannot be disabled):
   - Service delivered
   - Dispute window expiration warnings
   - Order completion
   - Payment/refund updates

### 5.6 Notification Requirements (Detailed)

#### 5.6.1 Client Notifications via Notification Service
All client notifications must be sent through the Notification Service (asynchronous via RabbitMQ). Services Service publishes events via outbox, and Notification Service handles delivery.

Notification Types:
1. Order Created:
   - Trigger: When order is created (standard or custom)
   - Channels: Email, In-app, Push (if mobile app)
   - Content:
     - Order ID, service title, expert name
     - Order amount and fee breakdown
     - Expected delivery date
     - Link to order details
     - For standard services: "Order automatically accepted by expert"
     - For custom services: "Waiting for expert acceptance"
2. Order Accepted (Custom Services):
   - Trigger: When expert accepts custom service order
   - Channels: Email, In-app, Push
   - Content:
     - Order accepted confirmation
     - Updated delivery timeline
     - Link to order details
3. Service Delivered (Critical - Highest Priority):
   - Trigger: When expert marks service as delivered
   - Channels: Email, SMS, In-app, Push
   - Email Subject: "Your service has been delivered - Review within 48 hours"
   - Email Body:
     - Dear [Client Name],
       Your service "[Service Title]" (Order #[OrderID]) has been delivered by [Expert Name].
       DISPUTE WINDOW: 48 HOURS
       You have 48 hours (configurable) to review the service and raise a dispute if needed.
       IMPORTANT NOTICE:
       After the dispute window expires, this order will be automatically marked as completed and payment will be released to the expert.
       No refunds can be requested after completion.
       [View Order] [Raise Dispute]
     - Dispute Window Countdown: [Real-time countdown in UI]
   - SMS Content:
     - "Service delivered! Order #[OrderID]. Review within 48 hours. [Order Link]"
   - In-app: Show prominent notification with countdown timer
4. Dispute Window Reminders:
   - 24 Hours Remaining:
     - Channels: Email, In-app
     - Subject: "Reminder: 24 hours left to review your service"
     - Content: Remind about dispute window and auto-completion
   - 12 Hours Remaining:
     - Channels: Email, SMS, In-app, Push
     - Subject: "Urgent: 12 hours left to review your service"
     - Content: Emphasize urgency and consequences
5. Order Completed (Auto):
   - Trigger: When dispute window expires and order auto-completes
   - Channels: Email, In-app, Push
   - Content:
     - "Order Completed
       Your order #[OrderID] has been automatically marked as completed as the dispute window has expired.
       Payment has been released to [Expert Name].
       Thank you for using our services!"
6. Order Cancelled/Refunded:
   - Trigger: When order is cancelled or refunded
   - Channels: Email, SMS, In-app, Push
   - Content: Cancellation reason, refund amount, refund timeline
7. Custom Request Auto-Rejected:
   - Trigger: When 7-day negotiation window expires
   - Channels: Email, In-app
   - Content: "Your custom request was not accepted within 7 days and has been automatically rejected"
8. Service Paused/Unpaused (if client has order in progress):
   - Trigger: When expert pauses/unpauses service that client has ordered
   - Channels: In-app, Email (optional)
   - Content: Service status change notification

#### 5.6.2 Expert Notifications via Notification Service
1. New Order Received:
   - Trigger: When order is created
   - Channels: Email, In-app, Push
   - Content: Order details, client information, delivery deadline
2. Service Paused/Unpaused:
   - Trigger: When service auto-unpauses (if pausedUntil was set)
   - Channels: Email, In-app
   - Content: "Your services have been automatically resumed"
3. Order Disputed:
   - Trigger: When client raises dispute
   - Channels: Email, SMS, In-app, Push
   - Content: Dispute details, required actions

### 5.7 Billing Service Integration Requirements

#### 5.7.1 Changes Required in Billing Service
The Billing Service must be updated to handle transaction charges and referral commissions for service orders.

##### 5.7.1.1 Transaction Charge Calculation
When Platform Fee is 0% (100% Commission Scenario):
1. Transaction Charge Logic:
   - Billing Service receives services.order.created event
   - Check if platformFeePercentage is 0%
   - If yes, apply minimum transaction fee:
     - Read MIN_TRANSACTION_FEE_PERCENTAGE from environment (default: 2.5%, range: 2-3%)
     - Calculate: transactionFeeAmountMinor
     - Round to nearest minor unit (paise)
2. Invoice Generation:
   - Include transaction charge as separate line item
   - Show breakdown:
     - Order Amount
     - Platform Fee: INR 0.00 (0%)
     - Transaction Charge: INR X.XX (X%)
     - Net Amount to Expert
   - Add note: "Transaction charge covers payment gateway processing costs"
3. API Changes Required:
   - Update invoice creation API to accept transaction fee parameters
   - Add transaction fee calculation endpoint (if separate)
   - Update invoice schema to include transactionFeeAmountMinor and transactionFeePercentage

##### 5.7.1.2 Referral Commission Integration
1. Commission Calculation:
   - Billing Service receives services.order.created event with referral information
   - If referral exists:
     - Extract referrerId, commissionSlab, commissionPercentage
     - Calculate commission: commissionAmountMinor = (orderAmountMinor * commissionPercentage) / 100
   - Apply commission
   - Platform fee is calculated
2. Commission Flow:
   - As per existing flow of Referral service
3. Invoice Updates:
   - Do necessary changes if required
4. Events to Publish:
   - billing.commission.calculated: When commission is calculated
   - billing.invoice.created: Include referral and transaction fee details

#### 5.7.2 Billing Service Event Schema Updates
Updated services.order.created event payload for Billing Service:
```
{
  "id": "uuid",
  "eventType": "services.order.created",
  "schemaVersion": 2,
  "occurredAt": "2025-01-01T00:00:00Z",
  "source": "services-service",
  "payload": {
    "orderId": "order-id",
    "expertId": "expert-id",
    "clientId": "client-id",
    "serviceId": "service-id",
    "amountMinor": 100000,
    "currency": "INR",
    "type": "STANDARD",
    "source": "SERVICES"
  }
}
```

Billing Service Response Events:
- billing.invoice.created: Includes all fee breakdowns
- billing.commission.applied: When referral commission is applied
- billing.transaction_fee.applied: When transaction fee is applied

### 5.8 Referral Service Integration Requirements

#### 5.8.1 Changes Required in Referral Service
The Referral Service must be updated to support service orders with commission slabs similar to audio/video calls.

##### 5.8.1.3 Events to Consume
- services.order.created: Extract referral info and track commission
- services.order.completed: Mark commission as earned (after dispute window)
- services.order.refunded: Reverse commission if order refunded

##### 5.8.1.4 Events to Publish
- referral.commission.calculated: When commission is calculated for an order
- referral.commission.earned: When commission is earned (order completed)
- referral.commission.reversed: When commission is reversed (order refunded)

#### 5.8.2 Referral Service API Updates
Endpoints:
- Add support for service order commissions
- Track orderType in commission records
- Store commission slab/tier
- Link to service order ID
- Track commission status (pending, earned, reversed)

### 5.9 Quick Reference & FAQs

#### 5.9.1 Custom Service Request & Order - Quick Reference
Q: What's the difference between a custom request and a custom order?
A:
- Custom Request (isRequest=true): Initial proposal/negotiation phase. Timeline and cost are optional. Can be cancelled by client or rejected by expert.
- Custom Order (isRequest=false): Confirmed order after negotiation. Timeline and cost are required. Cannot be cancelled by client (only disputed). Can be cancelled by expert with refund.

Q: Who can create custom orders?
A: Only experts can create custom orders. Clients can only create custom requests. After negotiation, the expert creates the actual order with final terms.

Q: Can a client create a custom order directly?
A: No. Clients can ONLY create custom requests. The expert must accept the request, negotiate if needed, and then create the custom order.

Q: Can a client cancel a custom order?
A: No. Once an order is confirmed (isRequest=false), clients cannot cancel it. They can only:
- Raise a dispute after delivery (within 48-hour dispute window)
- Wait for expert to cancel (which triggers automatic refund)
However, clients CAN cancel a custom request (isRequest=true) before the expert accepts it.

Q: Can an expert cancel an order?
A: Yes. Experts can cancel confirmed orders at any time before completion. This triggers:
- Automatic refund to client
- Conversation closure
- Opportunity for client to provide cancellation feedback/rating

Q: Are conversations linked to orders free?
A: Yes. All conversations linked to orders/requests (conversationId populated) are completely free - no messaging charges apply. This encourages open communication during negotiation and order fulfillment.

Q: When does a conversation close?
A: Conversations automatically close when:
- Order is completed
- Order is cancelled by client (for requests)
- Order is rejected by expert
- Order is cancelled by expert (for confirmed orders)
- Request is auto-rejected (7-day timeout)

Q: What happens if expert doesn't respond to custom request?
A: After 7 days (configurable via CUSTOM_REQUEST_NEGOTIATION_TIMEOUT_DAYS), the request is automatically rejected. Client receives notification and can create a new request if desired.

Q: What are the required fields for custom requests vs custom orders?
A:
- Custom Request (isRequest=true):
  - Required: expertId, serviceId, description
  - Optional: timeline, cost
- Custom Order (isRequest=false):
  - Required: expertId, serviceId, description, timeline, cost
  - All fields must be populated

Q: How does the dispute window work?
A:
- Applies only to confirmed orders (isRequest=false)
- Starts when expert marks service as delivered
- Default: 48 hours (configurable via DISPUTE_WINDOW_HOURS)
- Client receives multiple notifications (delivery, 24h reminder, 12h reminder, 1h warning)
- After dispute window expires: order auto-completes, payment released to expert, no refunds possible
- Client must raise dispute within this window if there are issues

Q: Can standard service orders be cancelled?
A:
- By client: No, clients cannot cancel standard service orders. Only disputes can be raised after delivery.
- By expert: Yes, experts can cancel standard service orders (e.g., if unavailable). Full refund is issued automatically.

Q: What is the transaction fee for 0% platform fee scenarios?
A: When platform fee is 0% (100% commission to expert), a minimum transaction fee of 2-3% (configurable via MIN_TRANSACTION_FEE_PERCENTAGE) is charged to cover payment gateway costs. This is clearly displayed to users before order confirmation.

Q: How do referral commissions work for service orders?
A:
- Commission slabs work similar to audio/video calls
- Referral Service tracks referrer and commission percentage
- Billing Service applies commission before calculating platform fee
- Commission is earned when order completes (after dispute window)
- Commission is reversed if order is refunded

Q: What notifications do clients receive for custom requests/orders?
A:
1. Custom request created -> Expert notified
2. Expert accepts/rejects -> Client notified
3. Expert creates order -> Client notified (must accept)
4. Client accepts order -> Both parties notified
5. Service delivered -> Client notified (with dispute window info)
6. Dispute window reminders -> Client notified (24h, 12h, 1h)
7. Order completed -> Both parties notified
8. Order cancelled -> Both parties notified

#### 5.9.2 Implementation Checklist
For Services Service:
- Add isRequest field to orders collection
- Add conversationId field to orders collection
- Implement custom request creation endpoint (client)
- Implement custom request accept/reject endpoints (expert)
- Implement custom order creation endpoint (expert)
- Implement custom order acceptance endpoint (client)
- Implement request cancellation endpoint (client)
- Implement order cancellation endpoint (expert)
- Update order validation logic (timeline/cost optional for requests, required for orders)
- Add indexes for isRequest, conversationId
- Implement 7-day auto-rejection workflow (Temporal)
- Update event schemas for all new events
- Integrate with Messaging Service for conversation creation

For Messaging Service:
- Add orderId field to conversation schema
- Add isFree field to conversation schema
- Update charging logic to check isFree flag
- Implement conversation creation API for order-linked conversations
- Implement conversation closure logic when order closes
- Add order link in conversation UI
- Consume Services Service events (order created, completed, cancelled)
- Publish conversation events for audit

For Billing Service:
- Implement transaction fee calculation for 0% platform fee scenarios
- Read MIN_TRANSACTION_FEE_PERCENTAGE from environment
- Update invoice generation to include transaction fee line item
- Integrate with Referral Service for commission calculation
- Apply commission before platform fee
- Publish commission and transaction fee events

For Referral Service:
- Add support for service order commission tracking
- Implement commission slab lookup for service orders
- Track commission status (pending, earned, reversed)
- Consume order events (created, completed, refunded)
- Publish commission events

For Notification Service:
- Create templates for all custom request/order events
- Implement dispute window reminder logic (24h, 12h, 1h)
- Emphasize "no refunds after completion" in all delivery notifications
- Add countdown timers to email templates
- Mark critical notifications as non-optional

For Feedback Service:
- Add cancellation feedback type
- Implement cancellation feedback request API
- Create feedback form for order cancellations
- Display cancellation feedback in expert profiles (with context)

End of Architecture Document
