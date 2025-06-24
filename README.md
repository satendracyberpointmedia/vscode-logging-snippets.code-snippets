# vscode-logging-snippets.code-snippets
# ⚡ VS Code Logging Snippet Pack

A productivity-oriented snippet pack to boost your logging and debugging experience for **Node.js**, **JavaScript**, and **TypeScript** development.

## 💡 What's Included?

✔️ Console Logs (`log_info`, `log_err`, `log_warn`, `log_debug`)  
✔️ Object and JSON logs  
✔️ Logger utility snippets for `logger.info`, `logger.error`, etc.  
✔️ Function tracing logs  
✔️ Performance timers  
✔️ Visual breaks and TODO reminders

---

## ✨ Snippets Overview

| Prefix         | Output Example                                     |
|----------------|----------------------------------------------------|
| `log_info`     | `[INFO]: some message`                             |
| `log_err`      | `[ERROR]: error.message`                           |
| `log_warn`     | `[WARNING]: some warning`                          |
| `log_debug`    | `[DEBUG]: some debug data`                         |
| `log_json`     | Pretty print an object as JSON                     |
| `log_obj`      | Deep inspect object using `console.dir`            |
| `log_time`     | Start `console.time`                               |
| `log_time_end` | End `console.timeEnd`                              |
| `logger_info`  | `logger.info(...)`                                 |
| `logger_err`   | `logger.error(...)`                                |
| `logger_debug` | `logger.debug(...)`                                |
| `log_entry`    | Log function entry with timestamp                  |
| `log_exit`     | Log function exit with timestamp                   |
| `log_break`    | Log visual separator (good for sectioning logs)    |
| `log_todo`     | Warn log styled as `[TODO]` reminder               |

---

## 📦 Installation

### 🔹 VS Code (Manual Install)

1. Copy the file `vscode-logging-snippets.code-snippets`
2. Paste it into `.vscode/logging-snippets.code-snippets` in your project
3. Or, for global snippets:
   - Open Command Palette → `Preferences: Configure User Snippets`
   - Choose global/user snippets
   - Paste the full snippet JSON

---

## 🛠 Example Usage

```js
log_info → console.log(`[INFO]: User created`);
log_json → console.log(JSON.stringify(user, null, 2));
logger_err → logger.error(`[ERROR]: Failed to connect to DB`);
log_entry → console.log(`[ENTRY] Function: createUser at 2025-06-24T06:00:00Z`);

Uses :- Open your vs code, press ctrl+p, search user snippets, select JS or TS file, than paste below json code in file.

{
  // 🔹 BASIC CONSOLE LOGGING
  "Print to console (Info)": {
    "prefix": "log_info",
    "body": [
      "console.log(`[INFO]: ${1:message}`);"
    ],
    "description": "Log info to console"
  },
  "Print to console (Error)": {
    "prefix": "log_err",
    "body": [
      "console.error(`[ERROR]: ${1:error}`);"
    ],
    "description": "Log error to console"
  },
  "Print to console (Warning)": {
    "prefix": "log_warn",
    "body": [
      "console.warn(`[WARNING]: ${1:warning}`);"
    ],
    "description": "Log warning to console"
  },
  "Print to console (Debug)": {
    "prefix": "log_debug",
    "body": [
      "console.debug(`[DEBUG]: ${1:message}`);"
    ],
    "description": "Log debug output to console"
  },

  // 🔹 OBJECT & STRUCTURED LOGGING
  "Log object deeply": {
    "prefix": "log_obj",
    "body": [
      "console.dir(${1:object}, { depth: null, colors: true });"
    ],
    "description": "Log an object deeply with full structure"
  },
  "Pretty JSON log": {
    "prefix": "log_json",
    "body": [
      "console.log(JSON.stringify(${1:data}, null, 2));"
    ],
    "description": "Log data as pretty JSON"
  },

  // 🔹 PERFORMANCE TIMING
  "Start performance timer": {
    "prefix": "log_time",
    "body": [
      "console.time('${1:TimerLabel}');"
    ],
    "description": "Start a performance timer"
  },
  "End performance timer": {
    "prefix": "log_time_end",
    "body": [
      "console.timeEnd('${1:TimerLabel}');"
    ],
    "description": "End a performance timer"
  },

  // 🔹 LOGGER UTILS (e.g., winston, pino, bunyan)
  "Logger info": {
    "prefix": "logger_info",
    "body": [
      "logger.info(`[INFO]: ${1:message}`);"
    ],
    "description": "Log info using logger"
  },
  "Logger error": {
    "prefix": "logger_err",
    "body": [
      "logger.error(`[ERROR]: ${1:error}`);"
    ],
    "description": "Log error using logger"
  },
  "Logger warn": {
    "prefix": "logger_warn",
    "body": [
      "logger.warn(`[WARN]: ${1:warning}`);"
    ],
    "description": "Log warning using logger"
  },
  "Logger debug": {
    "prefix": "logger_debug",
    "body": [
      "logger.debug(`[DEBUG]: ${1:details}`);"
    ],
    "description": "Log debug using logger"
  },
  "Logger trace": {
    "prefix": "logger_trace",
    "body": [
      "logger.trace(`[TRACE]: ${1:traceInfo}`);"
    ],
    "description": "Log trace using logger"
  },

  // 🔹 FUNCTION ENTRY/EXIT
  "Log function entry": {
    "prefix": "log_entry",
    "body": [
      "console.log(`[ENTRY] Function: ${1:functionName} at ${2:new Date().toISOString()}`);"
    ],
    "description": "Log function entry"
  },
  "Log function exit": {
    "prefix": "log_exit",
    "body": [
      "console.log(`[EXIT] Function: ${1:functionName} at ${2:new Date().toISOString()}`);"
    ],
    "description": "Log function exit"
  },

  // 🔹 MARKERS
  "Log section break": {
    "prefix": "log_break",
    "body": [
      "console.log('================ ${1:SECTION} ================');"
    ],
    "description": "Log visual break in console output"
  },
  "Log TODO": {
    "prefix": "log_todo",
    "body": [
      "console.warn('[TODO] ${1:description}');"
    ],
    "description": "Log a TODO note"
  }
}

