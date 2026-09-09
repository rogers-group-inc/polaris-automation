# Authoring rules

Run through these before emitting a script. Each one exists because the alternative produces
a script that appears to work and then misbehaves in production.

## 1. Parse `$1`; never reach for `$2`

The rendered args template is one argv entry. `$2` / `%2` / `sys.argv[2]` / `$args[1]` is
always empty. See context-and-args.md for the parser shapes.

## 2. Portability decides where context comes from

*Runs on: server* ⇒ you may read `POLARIS_ALERT_ID` / `POLARIS_RULE` / `POLARIS_ASSET`.
*Runs on: agent* or *either* ⇒ those do not exist; everything comes through args. Pick the
narrowest *Runs on* that does the job, and if you pick `either`, do not touch the alert env
vars except as an optional extra.

## 3. Exit 0 means "handled"

Non-zero ⇒ `failed` ⇒ a warning audit Event. So:

- Nothing to do, guard tripped, target already in the desired state ⇒ **exit 0** with a
  printed reason.
- A real failure the operator must see ⇒ non-zero.
- Do not let an incidental command failure decide the exit code. In bash prefer explicit
  checks over blanket `set -e`; if you use `set -e`, make sure every expected-to-fail command
  is guarded with `|| true` or an `if`.
- **PowerShell**: `Write-Error` does *not* set a non-zero exit code. `exit 1` explicitly, or
  the run is recorded as succeeded.

## 4. Be idempotent, and expect to run twice

Repeat-until-handled and escalation re-run the same action against the same alert. A script
that opens a ticket, sends a page or bounces a service must either be safe to repeat or use
the once-per-alert guard (recipe 2). "It only fires once" is not true of any automation that
repeats or escalates.

## 5. Scale-check the fan-out

One automation matching a large fleet enqueues one run per asset, and the server drains ~2
concurrently per 5 s tick. 200 assets is therefore minutes of queue, not seconds; the 90-day
run table grows by one row per asset per fire. Consequences to design around:

- Keep per-run wall time to seconds. A 60 s script × 200 assets is over an hour of drain.
- Do not put a retry loop with sleeps inside the script; the queue is already the buffer.
- If the work is fleet-wide rather than per-device, a single scheduled job outside Polaris is
  the better tool than a per-asset script action.
- Agent runs do not share the server drain — they execute on each asset in parallel — but each
  still needs an agent online and a command round-trip.

## 6. Never put a secret in the body

The body is stored plaintext, returned by the registry list endpoint to anyone with
`automationScripts:read`, shipped to agents over the wire, and captured in backups. Read
secrets at run time from a file the polaris service account (or LocalSystem) can read, mode
0600, outside the repo and outside the state dir. Keep them out of argv too — `ps` is readable
by other processes on the host, so pass a token via stdin or a config file, not as a
command-line flag.

## 7. Assume the output is published

stdout and stderr are stored unencrypted for 90 days, rendered in the Scripts tab, and
included in backups. Print a few structured lines an operator can act on. Do not dump `env`,
credentials, full API responses, or personal data. Stay well under the 64 KB per-stream cap —
exceeding it fails the run outright.

## 8. Finish inside the timeout; expect no cleanup

The kill is a SIGKILL (server) or a context cancellation (agent). Traps and `finally` blocks do
not run on timeout. Set the *Default timeout* to a little more than the realistic worst case,
never the maximum "to be safe" — a hung script holds one of two server execution slots for
the whole window. Give every outbound network call its own shorter timeout
(`curl --max-time`, `requests(timeout=…)`) so the script fails cleanly instead of being killed.

## 9. Treat `$1` as untrusted data

Values are substituted raw and unescaped. Never `eval` it, never pass it to `sh -c`, never
interpolate it into a shell command string, a SQL statement or a URL without encoding. Quote
every expansion. Build JSON with a serializer inside the script, never in the args template.

## 10. Guard the empty and the malformed

Handle: no asset context at all (test run, or an event automation with no asset); an args
string that is empty; an args string still containing `{` (a mistyped token). Report and
exit 0 rather than acting on nothing.

## 11. Do not reach into Polaris's own storage

Server scripts run as the service account on the Polaris host. Do not touch the Postgres
database, the state directory, backups or config files directly — schema and layout are not a
contract, and the runner deliberately strips the database URL and secret key from the
environment. Go through the REST API with a least-privilege bearer token.

## 12. Say what needs a human

Every emitted script closes with the review reminder: scripts execute with full privileges,
and a human must review and test-run one before enabling it in production. Server-target
scripts should be test-run from the Scripts tab first — that path passes no alert context,
which is exactly what rule 10's guard is for. Anything that changes device state, opens
tickets, or notifies people outside the team needs an owner who has read it.

## Checklist

- [ ] parses `$1`, no `$2`
- [ ] context source matches *Runs on*
- [ ] exit 0 on "nothing to do"; non-zero only on real failure (PowerShell: explicit `exit`)
- [ ] safe to run twice, or guarded
- [ ] finishes in seconds; every network call has its own timeout
- [ ] no secrets in the body, none in argv, none printed
- [ ] output is a few useful lines, far below 64 KB
- [ ] `$1` quoted everywhere, never evaluated
- [ ] empty-context and unrendered-token guards present
- [ ] the three deliverables are all present: body, registry settings, args template
- [ ] the human-review reminder is on the handover
