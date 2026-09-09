---
name: polaris-automation-scripts
description: "Writes scripts for Polaris automation script actions that an operator can paste straight into Automations → Scripts and save: the server vs agent execution contract, the {token} args vocabulary and the single-argv trap, the authoring rules (idempotence, exit codes, no secrets, storm safety), and a recipe library in bash / python3 / PowerShell. Load whenever someone asks for a Polaris automation script, a script action, a webhook/ticket/syslog/remediation hook on an alert, or asks what a script receives when an automation fires."
---

# Polaris automation scripts

Polaris automations can run an operator-authored script as an action. This skill produces
that script — **finished, pasteable, and correct about what it will actually receive.**

The registry and the runner are RCE-equivalent surfaces (`automationScripts` RBAC key,
seeded fullwrite for admin-equivalent roles only). Server scripts run as the polaris service
account on the Polaris host; agent scripts run as **root / LocalSystem** on the triggering
asset. Every script this skill emits carries the reminder that **a human must review it
before it is enabled in production** — that is the app's own posture, printed in the modal.

## A deliverable is three things, not one

Handing over a script body alone produces a script that runs with no context. Always emit
all three:

| # | What | Where the operator puts it |
|---|---|---|
| 1 | **Script body** | Automations → **Scripts** tab → **New script** → *Script body* |
| 2 | **Registry settings**: Name, Interpreter, *Runs on*, *Default timeout* | the same modal |
| 3 | **Args template** | the automation's **script action**, *args template* field — a separate screen |

Nothing reaches the script except (2)'s environment and (3)'s single rendered string.
If you omit the args template, the script gets `$1` empty and, on the agent, no context at all.

## Which file

| You need… | Read |
|---|---|
| where it runs, interpreters and their argv, limits, statuses, timeouts, the env each host sets, audit + retention | [references/execution-contract.md](references/execution-contract.md) |
| the full `{token}` vocabulary, the single-argv rule, the args conventions to generate, empty/typo'd context guards | [references/context-and-args.md](references/context-and-args.md) |
| the numbered authoring rules — run through these before emitting anything | [references/authoring-rules.md](references/authoring-rules.md) |
| ready-to-paste recipes: bash skeleton, once-per-alert guard, syslog forward, diagnostics collector, webhook/ticket (python3), Polaris API callback (python3), Windows agent remediation (PowerShell) | [references/recipes.md](references/recipes.md) |

## The eight things that are always true

1. **Args arrive as ONE positional parameter.** The rendered args template is passed as a
   single argv entry — `$1` in bash/python3, `%1` in cmd, the first positional param in
   PowerShell. `{asset.ip} {severity}` does **not** become `$1` and `$2`; it becomes one
   string in `$1`. Generate a parser, never `$2`.
2. **Environment context is server-only.** The server runner sets `POLARIS_ALERT_ID`,
   `POLARIS_RULE`, `POLARIS_ASSET`. The agent sets **only** `POLARIS_RUN_ID`. A script whose
   *Runs on* is `agent` or `either` must take everything it needs through the args template.
3. **Those env values are IDs, not names.** `POLARIS_ASSET` is the asset's id. Hostnames, IPs
   and every human-readable field come only from `{token}`s in the args template.
4. **The exit code is the verdict.** Non-zero ⇒ run status `failed` ⇒ a **warning** audit
   Event. Exit 0 for "handled, nothing to do"; reserve non-zero for a real failure, or the
   automation becomes a warning generator.
5. **stdout and stderr are stored and displayed.** Capped at 64 KB per stream, kept
   unencrypted on the run row for 90 days, rendered to anyone with `automationScripts:read`,
   and included in backups. Print what an operator needs; never print a secret.
6. **No secrets in the body.** The body is stored in plaintext, returned by the registry list
   endpoint, shipped over the wire to agents, and lands in backups. Read credentials from a
   file on the host at run time (mode 0600) and keep them out of argv.
7. **Timeout is a SIGKILL.** No trap runs, no cleanup happens. 1–600 s, default 60. Write
   scripts that finish well inside the window and need no graceful shutdown.
8. **It will re-fire.** Repeat-until-handled and escalation re-run the same action, and one
   automation over a large fleet enqueues one run per asset against a drain of ~2 concurrent
   runs per 5 s tick. Scripts must be idempotent, short, and safe to run twice — recipe 2 is
   the guard.

## Emitting the answer

Lead with the body in one fenced block the operator can copy whole. Then a short settings
block (Name / Interpreter / Runs on / Default timeout), then the args template on its own
line, then one line naming what a successful run prints. Close by reminding them to test-run
from the Scripts tab first — server-target scripts can be test-run there with no alert
context, which is exactly the path the empty-context guard is written for — and that a human
review is required before enabling it in production.
