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
