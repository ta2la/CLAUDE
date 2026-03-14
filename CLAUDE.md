# SUPERCLAUDE — cvz/CLAUDE/CLAUDE.md

## NO INNOVATIONS!!!!!!!!!! Follow existing patterns in other modules and repos. Copy naming, structure, and code style from what already exists. Do not invent new approaches when a working pattern is already in the codebase.

Shared conventions for all cvz projects. Referenced as "SUPERCLAUDE" or "../../CLAUDE/CLAUDE.md" from module CLAUDE.md files.

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

STDIO_BRIDGE is Claude's tool — use it to test, query, and interact with running apps. Don't ask the user to test via bridge; connect and verify yourself.

STDIO_BRIDGE (`APPS/STDIO_BRIDGE/`) is a console app that connects Claude to a running application (e.g. PROMPT_ASSEMBLER) via WebSocket. Claude executes the same commands as the user does in the UI — both go through the same CMD_SYS.

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

**Windows (PowerShell sync pattern):**
Write a .ps1 script to /tmp/bridge_X.ps1, then run via `powershell -NoProfile -File /tmp/bridge_X.ps1`:
```powershell
$env:PATH = "D:\Qt\DQt5.15.2\x64-windows-msvc2017\bin;" + $env:PATH
$pinfo = New-Object System.Diagnostics.ProcessStartInfo
$pinfo.FileName = "D:\KADLUB\cvz\APPS\STDIO_BRIDGE\release\STDIO_BRIDGE.exe"
$pinfo.RedirectStandardInput = $true
$pinfo.RedirectStandardOutput = $true
$pinfo.RedirectStandardError = $true
$pinfo.UseShellExecute = $false
$proc = [System.Diagnostics.Process]::Start($pinfo)
Start-Sleep -Milliseconds 500

$proc.StandardInput.WriteLine("./ws_connect")
$line = $proc.StandardOutput.ReadLine()   # wait for response

$proc.StandardInput.WriteLine("command here")
$line = $proc.StandardOutput.ReadLine()   # wait for response

$proc.Kill()
```

**Linux (fifo pattern):**
```bash
rm -f /tmp/bridge_in; mkfifo /tmp/bridge_in
tail -f /tmp/bridge_in | APPS/STDIO_BRIDGE/debug/STDIO_BRIDGE 2>&1
# Connect: echo "./ws_connect" >> /tmp/bridge_in
# Send:    echo "command args" >> /tmp/bridge_in
```

**Rules:**
- ALWAYS wait for response after each command before sending the next one. Read stdout and verify the result. Never batch multiple commands without waiting.
- Connection can be opened and closed as needed — no need to stay connected permanently.
- The app broadcasts all command results to all WS clients. This means Claude sees results of user's UI actions too — use this to stay aware of app state.
- Diagnostic/query commands replace the UI for Claude — use them to inspect state instead of asking the user what they see.

**`pa_file` convention:**
When the user writes `pa_file`, connect via bridge and call `file_preview_path` to get the path of the file currently previewed in PROMPT_ASSEMBLER. Then read that file from disk.

**Known constraints:**
- StdoutCmdOutput must be disabled in apps launched from Qt Creator (stdout→stdin echo loop)
- StdinMonitor filters lines starting with `qt.` (Qt debug message noise)
- WsClientBridge::send() uses QueuedConnection for thread safety (stdin thread → main thread)

**Potential enhancements:**
- Reconnect on disconnect (currently no retry if WS drops)
- Command-response correlation (match sent command to its result)
- Selective broadcast filtering (receive only results of own commands, not all)
- Binary data channel (QByteArray pass-through for file content)

## Qt Creator integration

- Open file at line in running Qt Creator: `qtcreator -client /path/to/file.cpp:120`
- `-client` flag sends command to already running Qt Creator instance
- Linux: qtcreator not in PATH, use full path from Qt installation

## Uniform project structure

`cvz/` is the root. Structure:

```
cvz/                          ← root (../../ from any CLAUDE.md)
├── base2/                    ← repo
│   ├── cmd_sys/              ← module (static lib)
│   ├── file_manager/         ← module (static lib)
│   └── ...
├── infrastructure/           ← repo
│   ├── command_registry/     ← module (static lib)
│   └── ...
├── APPS/                     ← repo
│   ├── PROMPT_ASSEMBLER/     ← exe module
│   │   └── prompt_assembler/ ← subdirs .pro (builds all deps + exe)
│   ├── CAD_EXE/              ← exe module
│   │   └── cad_exe/          ← subdirs .pro
│   └── STDIO_BRIDGE/         ← exe module
│       └── stdio_bridge/     ← subdirs .pro
└── BUILD/                    ← build output for base2 modules
```

- **Module** = directory with `.pro`, `.gitignore`, `.cpp`, `.h`. Builds as static lib.
- **Exe module** = same, `TEMPLATE = app`. Contains subdirectory of same name with subdirs `.pro` that lists all dependency modules and the exe itself.
- **Commands** = functions registered in `main()` via `CMD_SYS.add()`, not classes.
- **Guards/Filters** = registered via `CMD_SYS.reg()`, extend execution pipeline.
- **Bridge**: stdin → WS → CmdSys → execute → guards — same for all apps.
- **Config**: `config.t2l` script executed at startup, same commands as runtime.

When adding anything new — find the closest existing example and copy its pattern exactly.

## Architecture notes

- OregPool: singleton registry, solveChanges() batches updates
- OregObject → oo_delete() → pending list → pool deletes in solveChanges
- OregObserver: two-step notification pattern (observer → container)
- OregUpdateLock: RAII trigger for pool solve
- mutableContainment_: flag on container for re-evaluating CHANGED objects
- TestModelItem: Q_GADGET with value copies (not pointers)

## Commit Messages

Format:
```
[module] Short description

Details of changes.

Petr Talla: <contribution> (<N> lines) | Claude: <contribution> (<N> ins, <N> del)
```

- Last line documents authoring: who did what (design, direction, diagnosis, code, etc.)
- Include line counts for transparency.

---
**Note for context compression:** The STDIO_BRIDGE section (connection patterns, commands, rules) and the Command system section must NOT be summarized away during compression — they contain exact procedures needed for bridge interaction.
