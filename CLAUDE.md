## Command system (CLI)

- Commands available via stdin, output to stdout.
- Each command logs itself; additional info is appended to the command line in output.
- First argument is positional (unnamed); named args use `--name value`.
- `--name` can repeat.
- Arg value formats: plain, `"quoted string"`, or `{"json"}`.
- Example: `file_add path/to/file.cpp --lines 10 20`

## Logging (logcmd)

- `logcmd` is a no-op command — its execution IS the log entry.
- Usage: `logcmd <message>`
- Example: `logcmd prompt_registry_test start`

## Code Analysis Output Format

When asked to analyze code:
- Output code MUST include context: 10 lines before and 10 lines after each change.
- Mark each change with: `//## beg` and `//## end`
- For each change, print a header in comment: `// path/to/file.ext LINES: N..M`
- Frame modifications using:
```
//--- original lines
 new lines
```

## Coding Rules

- Prefer conservative modifications consistent with the existing code.
- Do NOT add unnecessary validations; validations only upon explicit request.
- Keep solutions simple and focused—do not over-engineer.
- Avoid backwards-compatibility hacks for unused code—delete completely if unused.

## STDIO_BRIDGE — Claude's interface to running apps

STDIO_BRIDGE is a lightweight console app that connects Claude's stdin/stdout to a running application (e.g. PROMPT_ASSEMBLER) via WebSocket.

**Architecture:**
```
Claude stdin → StdinMonitorBridge → WsClientBridge → WS (port 12345) → App's WsServerLite → CMD_SYS
                                                                                              ↓
Claude stdout ← WsClientBridge.onMessage_ ← WS broadcast ← WsServerLiteGuard ← command result
```

**Usage:**
- Lines without `./` prefix → forwarded to the app via WebSocket
- Lines with `./` prefix → executed locally in the bridge's CMD_SYS (`./` stripped before execution)
- App broadcasts every command result back to all connected WS clients
- Binary: `APPS/STDIO_BRIDGE/debug/STDIO_BRIDGE`
- Source: `APPS/STDIO_BRIDGE/`

**Bridge commands (local, prefix `./`):**
- `./ws_connect` — connect to the app via WebSocket. Must be called before sending commands. Reports `--STATUS connected` or `--ERROR` in command output. If connection fails, inform the user that the app is not running.
- `./ws_disconnect` — close the WebSocket connection. Reports `--STATUS disconnected`.
- `./ws_status` — check current connection state. Reports `--STATUS connected|disconnected`.

**Key files:**
- `WsClientBridge.h` — WS client, sends commands, receives results to stdout
- `StdinMonitorBridge.h` — stdin reader, routes `./` locally, rest to WS
- `Main.cpp` — wires components, enables StdoutCmdOutput

**How Claude uses the bridge (essential workflow):**

The bridge is Claude's primary way to interact with a running application. Both the user and Claude execute the same commands — the user via UI buttons, Claude via the bridge. The bridge stdout contains the result of every executed command (both Claude's and user's).

1. Start bridge in background:
   ```
   rm -f /tmp/bridge_in; mkfifo /tmp/bridge_in
   tail -f /tmp/bridge_in | APPS/STDIO_BRIDGE/debug/STDIO_BRIDGE 2>&1
   ```
   Save the background task ID.

2. Connect: `echo "./ws_connect" >> /tmp/bridge_in`

3. Read output and verify `--STATUS connected` before proceeding.

4. Send command: `echo "command args" >> /tmp/bridge_in`

5. Read output and verify the command result before sending the next command.

**Rules:**
- ALWAYS read and verify stdout after every command. The output tells you exactly what happened — success, error, data. Never guess, never skip.
- Connection can be opened and closed as needed — no need to stay connected permanently.
- The app broadcasts all command results to all WS clients. This means Claude sees results of user's UI actions too — use this to stay aware of app state.
- Diagnostic/query commands replace the UI for Claude — use them to inspect state instead of asking the user what they see.

**Known constraints:**
- StdoutCmdOutput must be disabled in apps launched from Qt Creator (stdout→stdin echo loop)
- StdinMonitor filters lines starting with `qt.` (Qt debug message noise)
- WsClientBridge::send() uses QueuedConnection for thread safety (stdin thread → main thread)

**Potential enhancements:**
- Reconnect on disconnect (currently no retry if WS drops)
- Command-response correlation (match sent command to its result)
- Selective broadcast filtering (receive only results of own commands, not all)
- Binary data channel (QByteArray pass-through for file content)

## Commit Messages

Format:
```
[module] Short description

Details of changes.

Petr Talla: <contribution> (<N> lines) | Claude: <contribution> (<N> ins, <N> del)
```

- Last line documents authoring: who did what (design, direction, diagnosis, code, etc.)
- Include line counts for transparency.
