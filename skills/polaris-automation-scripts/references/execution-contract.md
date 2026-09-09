# Execution contract

What actually happens between "the automation fires" and "the script's exit code is recorded".
Read this before choosing an interpreter or a *Runs on* value.

## Two hosts, two very different environments

| | **Runs on: the Polaris server** | **Runs on: the triggering asset's agent** |
|---|---|---|
| Identity | the polaris service account | **root / LocalSystem** |
| Platform | the Polaris host — Linux (RHEL 9 or the container image) | whatever the asset runs |
| Reaches | the Polaris host's network position | the asset itself, locally |
| Context env | `POLARIS_ALERT_ID`, `POLARIS_RULE`, `POLARIS_ASSET` | **`POLARIS_RUN_ID` only** |
| Requires | nothing | an installed Polaris Agent ≥ **0.13.0** on the triggering asset, and an alert that *has* an asset |
| Test-runnable from the Scripts tab | yes | no — trigger it through an automation on a real asset |

`either` means the action picks; the script must therefore satisfy **both** columns, which in
practice means "take all context through the args template" (rule 2 in SKILL.md).

An agent run fails the action with a clear message — recorded as an `automation.action_error`
Event — when the alert has no asset, when the asset has no agent, or when the agent is older
than 0.13.0. A `runTarget` that disagrees with the action's `runOn` is also rejected.

## The server path

1. The action enqueues an `AutomationScriptRun` row (`pending`). Execution is **never** inline
   in the engine.
2. The `runAutomationScripts` job ticks every **5 s** (web/all role): it sweeps stuck rows,
   then claims up to **10** pending rows `pending → running`, then executes at concurrency **2**.
3. The body is written to a **0600** temp file under the state dir with an extension matching
   the interpreter, and handed to the interpreter via `execFile` — the rendered args string is
   **one argv entry, never shell-interpolated**.
4. The interpreter binary resolves to a known absolute path where one exists
   (`/bin/bash`, `/usr/bin/python3`, `/usr/bin/pwsh`, …), falling back to a bare-name PATH
   lookup so a non-standard layout still works.
5. Timeout kills with **SIGKILL**. stdout/stderr are captured to a 64 KB cap. The temp file is
   always deleted.
6. The row records `status` / `exitCode` / `stdout` / `stderr` / `completedAt`, plus one
   `automation.script.run` audit Event (level `warning` on failure or timeout).
7. Completed runs are pruned after **90 days**.

A row still `running` more than **timeout + 60 s** after it started — a process restart or a
wedge — is swept to `timeout` with `"run abandoned (process restart or wedge)"`.

## The agent path

The action enqueues the run row *and* an `AgentCommand` with `action = "run_script"` and
payload `{ runId, interpreter, body, sha256, args, timeoutSec }`; a best-effort WebSocket wake
nudges the agent to fetch immediately instead of waiting for its next poll.

The agent **verifies the sha256 of the body before executing** and refuses on mismatch. It
writes the body to a 0700 temp file in its own temp dir, builds the same argv shape, runs it
under a context timeout, caps each output stream at 64 KB (a chatty script keeps running —
only the capture is truncated), and pushes back status / exit code / output.

The agent is a satellite: it refuses any action it does not know, and `run_script` is the only
action it accepts. Process and service start/stop/restart control was removed from the agent's
action set — a script is the sanctioned path for that kind of remediation, which means **the
script body is the audit record** for it.

## Interpreters

`bash` · `sh` · `powershell` · `cmd` · `python3`

| Interpreter | Argv the runner builds | Notes |
|---|---|---|
| `bash` / `sh` | `<bin> <script> [args]` | `sh` is POSIX — no arrays, no `[[`, no `declare -A` |
| `python3` | `python3 <script> [args]` | args is `sys.argv[1]` |
| `powershell` | `<bin> -NoProfile -NonInteractive -ExecutionPolicy Bypass -File <script> [args]` | resolves to `powershell.exe` on Windows, **`pwsh` on Linux** |
| `cmd` | `cmd.exe /d /s /c <script> [args]` | **Windows only** — refused outright on a Linux host or agent |

Consequences worth stating when you pick one:

- **Server target on a Linux host**: `bash`, `sh` and `python3` are safe. `cmd` will always
  fail. `powershell` needs `pwsh` installed on the Polaris host — do not assume it.
- **Agent target**: match the asset. Windows assets get `powershell` (or `cmd`); Linux assets
  get `bash` / `sh` / `python3`.
- `-NonInteractive` means any prompt is a failure, and `-NoProfile` means profile-defined
  functions, aliases and modules do not exist. Fully qualify what you call.

## Limits

| Thing | Limit |
|---|---|
| Script body | 64 KB |
| Args template | 2000 characters |
| Timeout | 1–600 s, default 60 (the action may override the script's default) |
| stdout / stderr captured | 64 KB **per stream** |
| Run retention | 90 days |
| Server drain rate | 10 claimed per 5 s tick, 2 executing concurrently |

## Run statuses

| Status | Cause |
|---|---|
| `succeeded` | exit 0 |
| `failed` | non-zero exit, spawn failure, disabled or deleted script, unavailable interpreter, **or output over the 64 KB cap** |
| `timeout` | killed at the timeout, or swept as abandoned past timeout + 60 s |

Exceeding the output cap is classified as **`failed`, not `timeout`** — a run that looks like
a mystery failure with a truncated log is usually a chatty script, not a hung one.

## Environment

The server runner hands the child `process.env` **minus** every key matching
`SECRET|TOKEN|PASSWORD|PASSWD|DATABASE_URL|SESSION|CREDENTIAL|PRIVATE_KEY|_KEY$`
(case-insensitive), then adds the three context vars. `PATH`, `HOME`, proxy vars and locale
are inherited, so ordinary scripts work — but the Polaris server's own database URL, secret-box
key and tokens are deliberately **not** reachable. Do not write a script that expects them.

The agent hands the child its own full environment plus `POLARIS_RUN_ID`.

## Audit

Creating a script, and **every change to its body**, writes a `warning`-level Event carrying
the old and new sha256 — script tampering is visible in the audit trail and syslog. Pasting a
revised version is an audited act; say so when handing over an update to an existing script.

Each run writes one `automation.script.run` Event with the run id, script id, rule id,
notification id, exit code and target.

## The disabled and deleted cases

A script with *Enabled* unchecked fails its runs with `"script is disabled"`. A script deleted
out from under a queued run fails with `"script no longer exists in the registry"`. Both are
`failed` status and therefore warning Events — disabling a noisy script does not silence it,
it converts it into a different warning. Remove the action instead.
