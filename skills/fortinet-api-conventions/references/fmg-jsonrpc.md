# FortiManager JSON-RPC

## Envelope

One endpoint (`POST /jsonrpc`), body `{ "id", "method": "get|add|set|update|delete|exec",
"params": [{ "url": "/…", "data": … }], "session"?: … }`. With api-key auth there is no
`session` field; send `Authorization: Bearer <api-key>` on every call. Responses carry
`result[0].status.code` (0 = ok) and `result[0].data`.

## Authentication

- FortiManager 7.4.7+ / 7.6.2+ removed `access_token` query-string support. Use the Bearer
  header exclusively; the same pattern works for standalone FortiOS api-users.
- A predefined **REST API Admin api-key is permanent** and, per Fortinet's FMG API Best
  Practices Guide, shares one session per user. `/sys/login` and `/sys/logout` exist for
  session-based auth only. Calling `/sys/logout` "to be tidy" invalidates the shared session
  for every process using the key; the symptom is `-11` "no valid session" on unrelated calls
  from other workers until they re-establish.
- Prefer the smallest ADOM-scoped admin profile the reads need; discovery is read-only except
  for the explicit push features.

## Fault classes and retry policy

| Class | Examples | Policy |
|---|---|---|
| transient | HTTP 5xx, socket errors, timeouts | retry ≤2 with exponential backoff, serialized per FMG |
| permanent | HTTP 401/403/404/405, JSON-RPC `-11` | fail fast, surface to the caller, do not retry |
| device-side | proxy result with the device's own error object | treat per call; the FMG call itself succeeded |

Serialize retries through the same lane as the original call so a retry storm cannot exceed
the manager's tolerance.

## Two lanes

FMG's `/sys/proxy/json` relays `/api/v2/monitor/...` and `/api/v2/cmdb/...` requests to a
managed device: `params[0].data = { action: "get", resource: "/api/v2/monitor/…", target:
["adom/<adom>/device/<name>"] }`. Run it as a **concurrency-1 lane per FMG**; a second
in-flight proxy call is queued behind the first. Native reads (`/dvmdb/...`, ADOM CMDB
`/pm/config/...`) go in a separate, wider lane. Per-FMG worker with two lanes is the shape that
survived a ~190-gate fleet.

## Roster: `/dvmdb/adom/<adom>/device`

- Lists every managed device with `name`, `sn`, `ip`, `conn_status` (1 = up), `platform_str`,
  `os_ver`/`mr`/`patch`, `ha_mode`, `ha_slave[]` (each with `sn`, `name`, `status`, `prio`).
- Answers for OFFLINE gates too — it is FMG's database, not the device — which is what makes
  it usable as a cheap fleet-wide reachability source (`conn_status`) and as the authoritative
  "which gate exists" list for stale sweeps.
- Read it once per tick and share; ~190 gates cost one round trip.
- The device NAME here is FortiManager's name, which operators rename freely and which need
  not equal the gate's configured hostname. Key everything on **serial**; use the name only to
  ADDRESS the device in proxy calls.

## HA

FMG flips a cluster's top-level `sn` to the active member. Compare identities against the set
`{sn} ∪ ha_slave[].sn`, and stamp per-member rows with the whole set, or every failover looks
like a chassis replacement. A standby's safety is only as good as its cluster answering;
derive standby state from the roster, not from the standby's own (usually unreachable) REST.

## Offline gates and CMDB pulls

A gate with `conn_status !== 1` cannot be proxied to, but its configuration can still be read
from FMG's CMDB (`/pm/config/device/<name>/...` and `/pm/config/adom/<adom>/...`): DHCP servers,
interfaces, VIPs, address groups. Discovery therefore splits: config from CMDB (always),
live state from monitor (only when reachable). Say which one a field came from.

## Proxy field filtering

Some objects come back through `/sys/proxy/json` with fewer fields than a direct REST call
returns (managed-switch/AP status in particular). Do not assume a missing field means the
device does not have it; when a field is load-bearing, read it from the direct path or from
the device's SNMP.

## Writing under central management

A device-side change on a centrally managed gate (a DHCP reservation, an interface
description) must also be written into FMG's database or the next install overwrites it. Use
JSON-RPC **`update`** on the `/pm/config/device/<name>/...` object — `set` replaces the whole
object and drops fields you did not send. Push to the device first, then mirror; treat a
failed mirror as a queued retry, not a silent success.

## Diagnosing `-11` churn

1. `grep` the logs for `code -11` grouped by process — churn from more than one process points
   at something invalidating the shared session (a logout, a duplicate login from another tool
   using the same admin).
2. Confirm no code path calls `/sys/logout` or `/sys/login` with the api-key admin.
3. Check the admin's session count in FMG; if another product uses the same api-user, give
   each product its own.
4. Only after the above: bounce the workers. A restart with the cause still present just
   recreates the churn.
