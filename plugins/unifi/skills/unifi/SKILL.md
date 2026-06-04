---
name: unifi
description: >
  Use this skill whenever the user asks about their UniFi network — connected clients, who's
  on the WiFi, which devices are online, access points, switches, gateways, network health,
  site health, active alarms, network events, WiFi configurations (SSIDs), controller sysinfo,
  or their authenticated UniFi identity. This skill covers the rustifi MCP server (read-only
  Rust bridge to the UniFi REST API via X-API-KEY). Trigger phrases include: "UniFi clients",
  "connected clients", "who's on the network", "UniFi devices", "access points", "APs",
  "UniFi switches", "WiFi networks", "WLAN config", "SSIDs", "network health", "UniFi health",
  "site health", "UniFi alarms", "network alerts", "network events", "UniFi events",
  "sysinfo", "controller version", "UniFi me". Always use this skill rather than guessing
  at curl commands or API paths — the UniFi REST API has several gotchas around path prefixes
  and auth that this skill encodes.
---

# UniFi Skill (rustifi)

Read-only access to a UniFi network controller via the **rustifi** MCP server. All data is
fetched from the UniFi REST API using X-API-KEY authentication.

## Quick Reference

All operations use a single `unifi` MCP tool with an `action` parameter:

```
unifi(action="clients")           # who's connected
unifi(action="devices")           # APs, switches, gateways
unifi(action="health")            # site health summary
unifi(action="wlans")             # WiFi network configs
unifi(action="alarms")            # active alarms
unifi(action="events", limit=20)  # recent events (limit optional)
unifi(action="sysinfo")           # controller version/uptime
unifi(action="me")                # authenticated user info
unifi(action="help")              # built-in documentation
```

---

## Tier 1 — MCP Tool (preferred)

**Tool name:** `unifi`  
**Required parameter:** `action` (string)

### Action Reference

| action | description | extra params |
|--------|-------------|--------------|
| `clients` | Connected wireless and wired clients | — |
| `devices` | Network devices: APs, switches, gateways | — |
| `wlans` | WiFi network configurations (SSID/band/security/VLAN) | — |
| `health` | Site health summary (subsystems, AP counts, client counts) | — |
| `alarms` | Active alarms and alerts | — |
| `events` | Recent network events | `limit` (int, optional) |
| `sysinfo` | Controller version, build, hostname, uptime, timezone | — |
| `me` | Authenticated user info (name, email, role) | — |
| `help` | Returns built-in action documentation | — |

### Response Shape

All actions return: `{"meta": {"rc": "ok"}, "data": [...]}`

Always index into `["data"]` for the actual records.

**Exception — `me`:** Returns `{"data": {...}}` (object, not array). The `/api/self` endpoint
it calls does not use the `/proxy/network` prefix — this is intentional and unique to this action.

### Example Calls

```python
# List connected clients
unifi(action="clients")
# → data[].{hostname, mac, ip, is_wired, essid, sw_port}

# List network devices
unifi(action="devices")
# → data[].{name, model, type, mac, ip, state, state_str}

# Recent 10 events
unifi(action="events", limit=10)
# → data[].{key, msg}

# WiFi networks (configurations, not per-SSID client counts)
unifi(action="wlans")
# → data[].{name, band, security, enabled, vlan_enabled, vlanid}

# Health overview
unifi(action="health")
# → data[].{subsystem, status, num_ap, num_disconnected, num_user, num_guest}

# Controller info
unifi(action="sysinfo")
# → data[0].{version, build, hostname, uptime, timezone}

# Current user
unifi(action="me")
# → data.{name, email, role, is_super_admin}
```

---

## Tier 2 — CLI Binary (fallback when MCP is unavailable)

Binary: `/home/jmagar/workspace/rustifi/target/release/runifi`

If the binary does not exist, build it first:
```bash
cd /home/jmagar/workspace/rustifi && cargo build --release
# or run without building:
cargo run --bin runifi -- <command>
```

| command | output |
|---------|--------|
| `runifi clients` | HOSTNAME / MAC / IP / TYPE / SSID or PORT |
| `runifi devices` | NAME / TYPE / MAC / STATE / IP |
| `runifi wlans` | SSID / BAND / VLAN / SECURITY |
| `runifi health` | subsystem status with AP and client counts |
| `runifi alarms` | `[key] message` per alarm |
| `runifi events [--limit N]` | `[key] message` per event |
| `runifi sysinfo` | Version, Build, Hostname, Uptime, Timezone |
| `runifi me` | Name, Email, Role, Super admin flag |

All commands accept `--json` for raw JSON output.

```bash
# Examples
runifi clients
runifi devices --json
runifi events --limit 20
runifi health
```

---

## Tier 3 — Direct REST API (emergency fallback)

Use when neither MCP nor CLI is available. Requires `UNIFI_URL` and `UNIFI_API_KEY` in environment.

**Auth:** `X-API-KEY` header — only works on UniFi OS consoles (UDM, UDR, UCG, UX, UDW).  
**TLS:** Self-signed certs are normal — always use `-sk` with curl.  
**Site:** Defaults to `default`.

**UDM/UniFi OS paths** (include `/proxy/network` prefix):

```bash
SITE=${UNIFI_SITE:-default}

# Clients
curl -sk "$UNIFI_URL/proxy/network/api/s/$SITE/stat/sta" \
  -H "X-API-KEY: $UNIFI_API_KEY" | jq '.data[] | {hostname, mac, ip, is_wired}'

# Devices
curl -sk "$UNIFI_URL/proxy/network/api/s/$SITE/stat/device" \
  -H "X-API-KEY: $UNIFI_API_KEY" | jq '.data[] | {name, type, mac, ip, state}'

# WLANs
curl -sk "$UNIFI_URL/proxy/network/api/s/$SITE/rest/wlanconf" \
  -H "X-API-KEY: $UNIFI_API_KEY" | jq '.data[] | {name, band, security, enabled}'

# Health
curl -sk "$UNIFI_URL/proxy/network/api/s/$SITE/stat/health" \
  -H "X-API-KEY: $UNIFI_API_KEY" | jq '.data'

# Alarms
curl -sk "$UNIFI_URL/proxy/network/api/s/$SITE/rest/alarm" \
  -H "X-API-KEY: $UNIFI_API_KEY" | jq '.data[] | {key, msg}'

# Events (most recent 20)
curl -sk "$UNIFI_URL/proxy/network/api/s/$SITE/rest/event" \
  -H "X-API-KEY: $UNIFI_API_KEY" | jq '.data[:20]'

# Sysinfo
curl -sk "$UNIFI_URL/proxy/network/api/s/$SITE/stat/sysinfo" \
  -H "X-API-KEY: $UNIFI_API_KEY" | jq '.data[0]'

# Me — NOTE: no /proxy/network prefix, even on UDM
curl -sk "$UNIFI_URL/api/self" \
  -H "X-API-KEY: $UNIFI_API_KEY" | jq '.data'
```

**Legacy controllers** (`UNIFI_LEGACY=true`, typically port 8443): use the same paths but
omit the `/proxy/network` prefix entirely.

---

## Key Gotchas

1. **`me` has a unique path.** `/api/self` never uses the `/proxy/network` prefix, even on
   modern UDM hardware. Every other action uses the site-scoped prefix.

2. **`events` limit is client-side.** The server fetches all events then truncates — it does
   not pass the limit to the UniFi API. Large event logs still incur full fetch cost.

3. **`wlans` is configuration, not client counts.** It returns SSID names, band, security
   mode, and VLAN settings. To count clients per SSID, cross-reference `clients` by `essid`.

4. **Wireless vs wired clients.** In `clients` data: `is_wired=false` means wireless — check
   `essid` for the SSID. `is_wired=true` means wired — check `sw_port` for the switch port.

5. **Device state.** In `devices` data: `state==1` means connected. Prefer `state_str` for
   human display; fall back to checking `state==1` when `state_str` is absent.

6. **Self-signed TLS is expected.** The UniFi controller uses a self-signed certificate by
   default. `UNIFI_SKIP_TLS_VERIFY=true` is the default in rustifi; use `-sk` in curl.

7. **`meta.rc` should be `"ok"`.** If the UniFi API returns an error, `meta.rc` will not be
   `"ok"`. The client raises an HTTP error in this case, so you'll see an anyhow error rather
   than an unexpected data shape.

---

## Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `UNIFI_URL` | Controller base URL, e.g. `https://192.168.1.1` | required |
| `UNIFI_API_KEY` | X-API-KEY header value | required |
| `UNIFI_SITE` | Site name | `default` |
| `UNIFI_SKIP_TLS_VERIFY` | Skip TLS certificate check | `true` |
| `UNIFI_LEGACY` | Omit `/proxy/network` prefix (legacy controllers) | `false` |
| `UNIFI_MCP_PORT` | MCP server bind port | `7474` |
| `UNIFI_MCP_TOKEN` | Static bearer token for MCP auth | — |
| `UNIFI_MCP_NO_AUTH` | Disable MCP auth (loopback only) | — |
