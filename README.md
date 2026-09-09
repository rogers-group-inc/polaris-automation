# polaris-automation

A Claude Code plugin that writes **scripts for Polaris automation script actions** — the kind
an operator pastes straight into Automations → Scripts and saves.

Polaris automations can run an operator-authored script when a rule fires, on the Polaris
server or on the triggering asset's agent. Getting one right means knowing what the script
actually receives, which is narrower and stranger than it looks: the args template arrives as
a *single* positional parameter, and the alert environment variables exist only on the server
target. This plugin carries that contract, the authoring rules, and a recipe library.

## Use it

Clone it once, then point Claude Code at the clone:

```
git clone https://github.com/rogers-group-inc/polaris-automation.git
claude --plugin-dir <path-to-clone>/polaris-automation
```

`git pull` in the clone picks up a new version (check `version` in `.claude-plugin/plugin.json`).

The skill `polaris-automation-scripts` auto-loads when someone asks for a Polaris automation
script, a script action, or a webhook / ticket / syslog / remediation hook on an alert. Invoke
by hand with `/polaris-automation:polaris-automation-scripts`.

## What it knows

- **The execution contract** — server (polaris service account, Linux) vs agent (root /
  LocalSystem on the asset), the 5 s / 10-claim / 2-concurrent server drain, sha256
  verification on the agent path, the five interpreters and the argv each one gets, every
  limit (64 KB body, 2000-char args, 1–600 s timeout, 64 KB output, 90-day retention), the
  run statuses and what actually causes each.
- **The context channels** — the full `{token}` vocabulary, the single-argv rule, raw
  unescaped substitution, literal-on-typo rendering, and the args conventions worth
  generating.
- **Twelve authoring rules** — idempotence under repeat and escalation, exit codes as the
  audit verdict, no secrets in a body that is stored plaintext and backed up, output
  discipline, storm behaviour at fleet scale, SIGKILL-on-timeout, treating args as untrusted.
- **Seven recipes** — bash skeleton, once-per-alert guard, syslog forward, diagnostics
  collector, webhook/ticket (python3), Polaris API callback (python3), Windows agent
  remediation (PowerShell) — each with its registry settings and args template.

## Scope

Application behaviour only: what the runner and the agent do with a script, and how to write
one that survives it. No Polaris hostnames, tokens, device names or site data — everything
here is the shape of the contract, not an install's contents.

The script registry is an **RCE-equivalent surface**: authoring requires the
`automationScripts` permission at fullwrite, server scripts run as the polaris service
account, and agent scripts run as root / LocalSystem on the target device. The skill emits the
app's own posture with every script — **a human must review and test-run a script before it is
enabled in production.**

## Layout

```
.claude-plugin/plugin.json
skills/polaris-automation-scripts/SKILL.md
skills/polaris-automation-scripts/references/execution-contract.md
skills/polaris-automation-scripts/references/context-and-args.md
skills/polaris-automation-scripts/references/authoring-rules.md
skills/polaris-automation-scripts/references/recipes.md
```

## Keeping it current

The contract this plugin describes lives in the Polaris repo, chiefly
`services/automationScriptService.ts`, `services/automationScriptRunner.ts`,
`services/automationActionService.ts`, `utils/notificationTemplate.ts` and
`agent/internal/scriptexec/`. A change to any of them — a new interpreter, a new `{token}`, a
changed limit, a new context variable — should be reflected here and the `plugin.json` version
bumped. This is a separate git repo: its commits are its own, never part of a Polaris commit.
