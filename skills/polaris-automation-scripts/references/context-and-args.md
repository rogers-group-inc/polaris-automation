# Context and args

A script gets exactly two channels. There is no stdin, no JSON payload, no config file
handed in, and no callback into the engine.

| Channel | Server target | Agent target |
|---|---|---|
| **Environment** — `POLARIS_ALERT_ID`, `POLARIS_RULE`, `POLARIS_ASSET` (all **ids**) | set | **not set** |
| **Environment** — `POLARIS_RUN_ID` | not set | set |
| **Args** — the action's args template, rendered at fire time | one argv entry | one argv entry |

So: **the args template is the only channel that works on both targets, and the only one that
carries human-readable values at all.** Everything a script needs to know about the device
that fired must be listed there.

## The single-argv rule

The args template is rendered and then passed as **one** argv element.

```
args template:   {asset.ip} {severity}
rendered:        10.14.3.20 warning
the script sees: $1 = "10.14.3.20 warning"     $2 = ""      $# = 1
```

This is the single most common way a generated script fails. Never emit a script that reads
`$2`, `%2`, `sys.argv[2]` or `$args[1]`. Emit a parser for `$1`.

## Rendering has no escaping, and typos survive

Token values are substituted **raw**. A description, tag list, event message or trigger
summary can contain spaces, quotes, semicolons, `|`, `$`, and in principle newlines. The
runner never lets a shell parse the args string, so this is not a command-injection path into
the interpreter — but it absolutely will break a naive parser, and it *would* become an
injection if your script feeds `$1` to `eval`, `sh -c`, or a shell-interpolated SQL/HTTP
string. Treat `$1` as untrusted data.

An **unknown or mistyped token is left literal**, by design, so a typo stays visible: an args
template of `{asste.ip}` delivers the seven characters `{asste.ip}` to the script. Generated
scripts should notice — see the guard below.

## Args conventions to generate

**Primary: newline-delimited `key=value`.** One pair per line, structured values first, free
text last or omitted. Robust, self-describing in the run record, and trivial to parse in all
five interpreters.

```
asset_id={asset}
asset_ip={asset.ip}
severity={severity}
metric={metric}
value={value}
```

**Alternative: pipe-delimited positional**, for a one-value or two-value script where a parser
is overkill:

```
{asset.ip}|{severity}
```

**Interpreter decides which shape.** `bash` and `python3` read a multi-line argv value
cleanly, so use newline-delimited `key=value` there. For `powershell` and especially `cmd`,
keep the template to **one line** and use the pipe-delimited shape — multi-line argv values
are unreliable through Windows argument handling.

Rules for either shape:

- Pass **structurally safe** values when the script parses them: ids, IPs, MACs, severity,
  status, type, model, serial, times, numbers.
- **Free-text tokens** — `{message}`, `{asset.description}`, `{trigger.summary}`,
  `{event.message}`, `{asset.tags}`, `{rule.description}` — go **last**, or not at all. A
  newline inside one would forge a `key=value` line; a `|` would shift a positional field.
- Never build JSON in the args template. Unescaped quotes in a description produce invalid
  JSON. Build JSON **inside** the script, with a real serializer.
- Keep it under 2000 characters.

## Guards every generated script needs

**Empty context.** A Scripts-tab test run passes no alert, rule or asset, so all three env
vars are `""` and the args string is whatever the tester typed (often nothing). Event-triggered
automations can also fire with no asset. Guard, report, and `exit 0` — never divide by empty
or act on a blank target.

**Unrendered token.** If the args string still contains a `{`, the args template has a typo.
Log a warning and keep going (or exit 0), rather than acting on the literal text.

## The token vocabulary

Single-brace `{token}`. Not `{{ }}`.

### Alert / trigger

| Token | Value |
|---|---|
| `{asset}` | asset hostname (or id, or `"host"`) |
| `{metric}` | metric / field / event action that triggered |
| `{value}` | observed value at fire time |
| `{threshold}` | configured threshold / comparison value |
| `{dimension}` | sub-asset dimension — interface / mount / sensor / tunnel |
| `{conditions}` | multi-condition summary, e.g. `2 of 3 conditions met` (composite triggers only) |
| `{trigger.summary}` | the trigger in the builder's words, with the observed value — free text |
| `{message}` | the rendered in-app notification message — free text |
| `{severity}` | rule severity, e.g. `warning` |
| `{severity.upper}` | `WARNING` |
| `{severity.color}` | hex colour for the severity |
| `{time}` | trigger time, ISO-8601 |
| `{time.local}` | trigger time in the server's timezone, human-readable |
| `{link}` | Automations page URL (empty when `POLARIS_PUBLIC_URL` is unset) |
| `{ack}` | this alert's acknowledge URL — **filled at delivery expansion, not at args render time; do not use it in an args template** |

### Rule

| Token | Value |
|---|---|
| `{rule}` | rule name |
| `{rule.description}` | rule description — free text |

### Asset

| Token | Value |
|---|---|
| `{asset.ip}` | primary IP |
| `{asset.mac}` | MAC |
| `{asset.type}` | asset type, e.g. `firewall` |
| `{asset.status}` | lifecycle status, e.g. `active` |
| `{asset.location}` | location (operator-set, falling back to learned) |
| `{asset.description}` | description — free text |
| `{asset.manufacturer}` · `{asset.model}` · `{asset.serial}` | hardware identity |
| `{asset.os}` · `{asset.osVersion}` | operating system |
| `{asset.department}` · `{asset.assignedTo}` | ownership |
| `{asset.tags}` | tags, comma-joined — free text |
| `{asset.connectedSwitch}` | switch/port last seen on, e.g. `FS-248E-01/port15` |
| `{asset.connectedAp}` | AP last seen on |
| `{asset.link}` | URL that opens the device in Polaris (empty without `POLARIS_PUBLIC_URL`) |

### Event-triggered automations only

`{event.action}` · `{event.resource}` · `{event.resourceType}` · `{event.actor}` ·
`{event.message}` (the reason — free text) · `{event.level}`

Empty on every other trigger type.

### Escalation

`{escalation.tier}` (empty on the initial send) · `{escalation.elapsed}` (e.g. `1h 30m`)

Useful in an args template when the script should behave differently on a second or third
attempt — but prefer the once-per-alert guard in recipes.md for suppressing repeats.

### Not for args

`{chart.cpu}`, `{chart.memory}`, `{chart.responseTime}`, `{chart.sensor}`,
`{interface.lldp}`, `{brand.header}` — these render inline HTML or a multi-line plain-text
block for email. They will make a mess of an args string.

## Turning an id into something useful

`POLARIS_ASSET` is an id, and `{asset}`/`{asset.ip}` may be all the script needs. When a
script genuinely needs more of the record than the args template can carry, call the Polaris
REST API with a role-bound bearer token — see the API callback recipe. Two constraints: the
token is a secret, so it lives in a file and not in the body; and it should be bound to a
least-privilege role, not an admin one.
