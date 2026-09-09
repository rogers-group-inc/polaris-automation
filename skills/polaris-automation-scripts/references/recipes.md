# Recipes

Each recipe is a complete deliverable: the body, the registry settings, and the args template.
Adapt rather than invent — the parsers and guards here are the shapes that survive the
single-argv rule, repeats, test runs and mistyped tokens.

**Interpreter and args shape go together.** `bash` and `python3` handle a multi-line
newline-delimited args string cleanly, so prefer `key=value` per line there. For `powershell`
and especially `cmd`, keep the args template to **one line** and use `|` between pairs —
multi-line argv values are unreliable through Windows argument handling.

---

## 1. bash skeleton (server) — start here

The base every server-target bash script should grow from: single-argv parser, empty-context
guard, unrendered-token guard, timestamped logging, honest exit codes.

**Registry**: Name `polaris-skeleton` · Interpreter **bash** · Runs on **the Polaris server** · Timeout **60**

**Args template**
```
asset_id={asset}
asset_ip={asset.ip}
severity={severity}
metric={metric}
value={value}
```

```bash
#!/usr/bin/env bash
# polaris-skeleton — <what this script does>
# Polaris automation script. Runs on: the Polaris server (service account).
# Args arrive as ONE positional parameter ($1); $2 is always empty.
set -uo pipefail

ARGS="${1-}"

log()  { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*"; }
fail() { log "ERROR: $*" >&2; exit 1; }

# --- parse the single argv entry into ARG[key]=value -----------------------
# Needs bash (associative arrays); with Interpreter=sh, use the sh variant below.
declare -A ARG=()
while IFS= read -r line; do
  [ -n "$line" ] || continue
  case "$line" in *=*) ARG["${line%%=*}"]="${line#*=}" ;; esac
done <<< "$ARGS"

asset_id="${ARG[asset_id]-}"
asset_ip="${ARG[asset_ip]-}"
severity="${ARG[severity]-}"

# Server-only context — ids, not names. Empty on a test run and on the agent.
alert_id="${POLARIS_ALERT_ID-}"
rule_id="${POLARIS_RULE-}"

# --- guards ----------------------------------------------------------------
case "$ARGS" in
  *'{'*) log "WARNING: args contain an unrendered token — check the args template" ;;
esac

if [ -z "$asset_id" ] && [ -z "$asset_ip" ]; then
  log "no asset context (test run?) — nothing to do"
  exit 0
fi

# --- the work --------------------------------------------------------------
log "asset=${asset_id:-?} ip=${asset_ip:-?} severity=${severity:-?} alert=${alert_id:-none}"

# ... replace with the action. Quote every expansion; never eval "$ARGS". ...

log "done"
exit 0
```

**sh variant of the parser** (no associative arrays — read the keys you need):

```sh
get_arg() {  # get_arg <key> <args>
  printf '%s\n' "$2" | while IFS= read -r line; do
    case "$line" in "$1="*) printf '%s' "${line#*=}"; break ;; esac
  done
}
asset_ip="$(get_arg asset_ip "$ARGS")"
```

---

## 2. Once-per-alert guard (bash fragment)

Drop this in after the guards in recipe 1 when the action has an external side effect — a
ticket, a page, a service bounce. Repeat-until-handled and escalation re-run the same action
against the same alert; this makes the second and third run a no-op.

```bash
# --- act at most once per alert --------------------------------------------
# An atomic noclobber create is the marker. If /tmp was cleared (reboot), the
# guard simply restarts — the safe direction for a remediation script.
guard_key="${alert_id:-$asset_id}"
if [ -n "$guard_key" ]; then
  guard_dir="${TMPDIR:-/tmp}/polaris-script-guards"
  mkdir -p "$guard_dir" || fail "cannot create $guard_dir"
  guard_file="$guard_dir/$(printf '%s' "$guard_key" | tr -c 'A-Za-z0-9_.-' '_')"
  if ! (set -o noclobber; : > "$guard_file") 2>/dev/null; then
    log "already acted on alert $guard_key — skipping repeat"
    exit 0
  fi
  find "$guard_dir" -type f -mtime +7 -delete 2>/dev/null || true
fi
```

**On an agent target this guard cannot be keyed to the alert.** There is no alert-id token in
the args vocabulary, and the agent sets no `POLARIS_ALERT_ID` — so the best available key is
`{rule}` + `{asset}` (plus a time bucket if you need one per outage rather than one ever).
Say so when handing over an agent-target script with a side effect.

---

## 3. Syslog / SIEM forward (server)

Copies the alert onto the Polaris host's syslog so an existing SIEM pipeline picks it up. No
credentials, nothing to break, finishes in milliseconds — the cheapest useful script action.

**Registry**: Name `polaris-syslog-forward` · Interpreter **bash** · Runs on **the Polaris server** · Timeout **15**

**Args template** (one line; free text last)
```
rule={rule}|severity={severity}|asset={asset}|ip={asset.ip}|metric={metric}|value={value}|summary={trigger.summary}
```

```bash
#!/usr/bin/env bash
# polaris-syslog-forward — copy the alert into the host's syslog for the SIEM.
set -uo pipefail

ARGS="${1-}"
[ -n "$ARGS" ] || { echo "no args — nothing to forward"; exit 0; }

case "$ARGS" in
  *severity=critical*) prio=local0.crit ;;
  *severity=error*)    prio=local0.err ;;
  *severity=warning*)  prio=local0.warning ;;
  *)                   prio=local0.notice ;;
esac

# -- ends option parsing, so a value beginning with '-' is never read as a flag.
logger -t polaris-alert -p "$prio" -- "$ARGS" || { echo "logger failed" >&2; exit 1; }

echo "forwarded to syslog at $prio"
exit 0
```

---

## 4. Diagnostics collector (server)

Captures reachability evidence at the moment a device went down, so the run record holds it
when someone looks an hour later. **Changes nothing and always exits 0** — the output *is* the
deliverable, which is exactly the case rule 3 exists for.

**Registry**: Name `polaris-collect-diagnostics` · Interpreter **bash** · Runs on **the Polaris server** · Timeout **90**

**Args template**
```
asset={asset}
asset_ip={asset.ip}
switch={asset.connectedSwitch}
```

```bash
#!/usr/bin/env bash
# polaris-collect-diagnostics — reachability evidence for a down device.
# Read-only. Always exits 0; the captured output is the point.
set -uo pipefail

ARGS="${1-}"
declare -A ARG=()
while IFS= read -r line; do
  [ -n "$line" ] || continue
  case "$line" in *=*) ARG["${line%%=*}"]="${line#*=}" ;; esac
done <<< "$ARGS"

ip="${ARG[asset_ip]-}"
name="${ARG[asset]-unknown}"

if [ -z "$ip" ]; then
  echo "no IP in args — nothing to probe"
  exit 0
fi

echo "== $name ($ip) at $(date -u +%Y-%m-%dT%H:%M:%SZ) =="
echo "-- last seen switch: ${ARG[switch]-unknown}"

echo "-- ping (5 × 1 s)"
ping -c 5 -W 2 "$ip" 2>&1 | tail -n 4 || echo "   ping: no reply"

echo "-- traceroute (max 12 hops)"
if command -v traceroute >/dev/null 2>&1; then
  timeout 30 traceroute -n -m 12 -w 2 "$ip" 2>&1 | tail -n 14 || echo "   traceroute incomplete"
else
  echo "   traceroute is not installed on the Polaris host"
fi

echo "-- tcp reachability"
for port in 22 443; do
  # $ip and $port are passed as ARGUMENTS to the inner shell, never interpolated
  # into its command string.
  if timeout 2 bash -c 'exec 3<>/dev/tcp/"$1"/"$2"' _ "$ip" "$port" 2>/dev/null; then
    echo "   $port open"
  else
    echo "   $port closed or filtered"
  fi
done

echo "== end =="
exit 0
```

---

## 5. Webhook / ticket POST (server, python3)

Opens a ticket or fires a webhook. Demonstrates the two things that always go wrong here:
JSON built with a real serializer (never in the args template, whose values are unescaped),
and a credential read from a file at run time rather than living in the body.

**Prerequisite** — create on the Polaris host, owned by the polaris service account, mode 0600:

```
/etc/polaris/script-secrets/ticketing.conf
    url=https://ticketing.example.internal/api/incidents
    token=<the API token>
```

**Registry**: Name `polaris-open-ticket` · Interpreter **python3** · Runs on **the Polaris server** · Timeout **45**

**Args template**
```
asset={asset}
asset_ip={asset.ip}
severity={severity}
rule={rule}
link={asset.link}
summary={trigger.summary}
```

```python
#!/usr/bin/env python3
"""polaris-open-ticket - POST the alert to a ticketing or webhook endpoint.

Polaris automation script. Runs on: the Polaris server.
Args arrive as ONE positional parameter (sys.argv[1]).

The endpoint URL and token are read at run time from CONF - never from this
body, which is stored in plaintext, readable through the registry API, and
captured in backups.
"""
import json
import os
import sys
import urllib.error
import urllib.request

CONF = "/etc/polaris/script-secrets/ticketing.conf"   # url= / token=
TIMEOUT_S = 15                                        # well inside the run timeout


def parse_kv(text):
    """Newline-delimited key=value into a dict. Values are kept verbatim."""
    out = {}
    for line in text.splitlines():
        line = line.strip()
        if not line or line.startswith("#") or "=" not in line:
            continue
        key, value = line.split("=", 1)
        out[key.strip()] = value
    return out


def main():
    raw = sys.argv[1] if len(sys.argv) > 1 else ""
    args = parse_kv(raw)

    if "{" in raw:
        print("WARNING: args contain an unrendered token - check the args template")

    asset = args.get("asset", "")
    if not asset and not args.get("asset_ip"):
        print("no asset context (test run?) - nothing to do")
        return 0

    try:
        with open(CONF, "r", encoding="utf-8") as handle:
            conf = parse_kv(handle.read())
    except OSError as exc:
        print(f"cannot read {CONF}: {exc}", file=sys.stderr)
        return 78  # EX_CONFIG - a setup problem, not a transient failure

    url, token = conf.get("url"), conf.get("token")
    if not url or not token:
        print(f"{CONF} must define url= and token=", file=sys.stderr)
        return 78

    alert_id = os.environ.get("POLARIS_ALERT_ID", "")

    payload = json.dumps({
        "title": f"[{args.get('severity', 'alert')}] {asset} - {args.get('rule', '')}",
        "body": args.get("summary", ""),
        "asset": asset,
        "asset_ip": args.get("asset_ip", ""),
        "severity": args.get("severity", ""),
        "link": args.get("link", ""),
        "alert_id": alert_id,
        "rule_id": os.environ.get("POLARIS_RULE", ""),
    }).encode("utf-8")

    request = urllib.request.Request(
        url,
        data=payload,
        method="POST",
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Bearer {token}",       # a header, never argv
            # Repeats and escalation re-run this action against the same alert.
            # Where the endpoint honours it this collapses them into one ticket;
            # where it does not, add the recipe 2 guard.
            "Idempotency-Key": alert_id or f"{asset}:{args.get('rule', '')}",
        },
    )

    try:
        with urllib.request.urlopen(request, timeout=TIMEOUT_S) as response:
            print(f"ticket accepted: HTTP {response.status}")
            return 0
    except urllib.error.HTTPError as exc:
        # The body can echo the request, token included - print the status only.
        print(f"endpoint rejected the ticket: HTTP {exc.code}", file=sys.stderr)
        return 1
    except (urllib.error.URLError, OSError, TimeoutError) as exc:
        print(f"endpoint unreachable: {exc}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

---

## 6. Read the asset back from the Polaris API (server, python3)

When a script needs more of the record than an args template can carry. `POLARIS_ASSET` is an
**id**, so this is the path from that id to the full row.

Two constraints: the bearer token is a secret, so it lives in a file; and it should be bound to
a **least-privilege role** (assets read), never an admin one. Do not read Polaris's Postgres
or state directory directly — go through the API. For write payloads, consult the in-app API
page (`/api.html`) or the `polaris-api-conventions` plugin rather than guessing a shape.

**Prerequisite** — mode 0600, owned by the polaris service account:

```
/etc/polaris/script-secrets/polaris-api.conf
    base_url=https://polaris.example.internal
    token=<a role-bound bearer token with assets:read>
```

**Registry**: Name `polaris-enrich-from-api` · Interpreter **python3** · Runs on **the Polaris server** · Timeout **30**

**Args template**: none required — this one uses `POLARIS_ASSET`, which makes it server-only.

```python
#!/usr/bin/env python3
"""polaris-enrich-from-api - read the alerting asset's full record.

Runs on: the Polaris server (POLARIS_ASSET is set only there).
"""
import json
import os
import sys
import urllib.error
import urllib.parse
import urllib.request

CONF = "/etc/polaris/script-secrets/polaris-api.conf"   # base_url= / token=
TIMEOUT_S = 10
FIELDS = ("hostname", "ipAddress", "macAddress", "assetType", "location",
          "monitorStatus", "manufacturer", "model", "serialNumber", "lastSeen")


def parse_kv(text):
    out = {}
    for line in text.splitlines():
        line = line.strip()
        if not line or line.startswith("#") or "=" not in line:
            continue
        key, value = line.split("=", 1)
        out[key.strip()] = value
    return out


def main():
    asset_id = os.environ.get("POLARIS_ASSET", "")
    if not asset_id:
        print("no asset in context (test run, or an event automation with no "
              "asset) - nothing to do")
        return 0

    try:
        with open(CONF, "r", encoding="utf-8") as handle:
            conf = parse_kv(handle.read())
    except OSError as exc:
        print(f"cannot read {CONF}: {exc}", file=sys.stderr)
        return 78

    base, token = conf.get("base_url", "").rstrip("/"), conf.get("token")
    if not base or not token:
        print(f"{CONF} must define base_url= and token=", file=sys.stderr)
        return 78

    # The id is percent-encoded rather than pasted into the URL.
    url = f"{base}/api/v1/assets/{urllib.parse.quote(asset_id, safe='')}"
    request = urllib.request.Request(url, headers={"Authorization": f"Bearer {token}"})

    try:
        with urllib.request.urlopen(request, timeout=TIMEOUT_S) as response:
            asset = json.load(response)
    except urllib.error.HTTPError as exc:
        print(f"asset read failed: HTTP {exc.code}", file=sys.stderr)
        return 1
    except (urllib.error.URLError, OSError, TimeoutError, ValueError) as exc:
        print(f"asset read failed: {exc}", file=sys.stderr)
        return 1

    # Printed output is stored for 90 days and shown in the Scripts tab - keep
    # it to the fields an operator needs.
    for field in FIELDS:
        print(f"{field}={asset.get(field, '')}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

---

## 7. Windows remediation on the triggering asset (agent, PowerShell)

Restarts a named service on the device that alerted. This is the shape agent scripts take:
**every value comes through the args template**, because the agent sets only
`POLARIS_RUN_ID` — no alert, rule or asset id at all.

Note the posture: the agent refuses service-control *actions* by design, so a script is the
sanctioned path for this, which makes the script body the audit record. It runs as
**LocalSystem**, cannot be test-run from the Scripts tab, and needs an agent ≥ 0.13.0 on the
asset.

**Registry**: Name `polaris-restart-service` · Interpreter **powershell** · Runs on **the triggering asset's agent** · Timeout **120**

**Args template** (one line, pipe-delimited — safest through Windows argument handling)
```
service=Spooler|asset={asset}|severity={severity}|rule={rule}
```

```powershell
# polaris-restart-service - restart a named Windows service on the triggering asset.
# Runs on: the triggering asset's agent (LocalSystem).
# Args arrive as ONE positional parameter; the agent sets no alert context.
param([string]$PolarisArgs = "")

function Write-Log([string]$Message) {
  Write-Output ("{0} {1}" -f (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ"), $Message)
}

# --- parse the single positional parameter --------------------------------
$ctx = @{}
foreach ($pair in ($PolarisArgs -split '\|')) {
  if ($pair -match '^\s*([^=]+)=(.*)$') { $ctx[$Matches[1].Trim()] = $Matches[2] }
}

if ($PolarisArgs -like '*{*') {
  Write-Log "WARNING: args contain an unrendered token - check the args template"
}

$service = $ctx['service']
if ([string]::IsNullOrWhiteSpace($service)) {
  Write-Log "no service name in args - nothing to do"
  exit 0
}

Write-Log ("run={0} asset={1} rule={2} service={3}" -f `
  $env:POLARIS_RUN_ID, $ctx['asset'], $ctx['rule'], $service)

# --- act, idempotently ----------------------------------------------------
try {
  $svc = Get-Service -Name $service -ErrorAction Stop
} catch {
  Write-Log ("service '{0}' does not exist on this host - nothing to fix" -f $service)
  exit 0    # not an operator-visible failure
}

if ($svc.StartType -eq 'Disabled') {
  Write-Log ("service '{0}' is Disabled - refusing to start it" -f $service)
  exit 0
}

try {
  Restart-Service -Name $service -Force -ErrorAction Stop
  Start-Sleep -Seconds 3
  $svc = Get-Service -Name $service
  Write-Log ("service '{0}' is now {1}" -f $service, $svc.Status)
  # Write-Error alone does NOT set the exit code - always exit explicitly.
  if ($svc.Status -ne 'Running') { exit 1 }
  exit 0
} catch {
  Write-Log ("restart failed: {0}" -f $_.Exception.Message)
  exit 1
}
```

Because there is no alert id on the agent, a repeat or escalation **will** restart the service
again. If that is unacceptable, key a marker file under `$env:ProgramData\Polaris` to
`rule + asset` and bucket it by time.
