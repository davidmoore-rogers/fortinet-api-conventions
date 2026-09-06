---
name: fortinet-api-conventions
description: "FortiManager JSON-RPC and FortiOS REST conventions and traps: api-key Bearer auth and why /sys/logout must never be called, RPC fault classes and retry policy, proxy vs direct device transport, /dvmdb rosters and HA serial flips, CMDB vs monitor reads, DHCP reservation/lease/VIP shapes, quarantine address groups, FortiSwitch and FortiAP SNMP quirks. Load whenever code calls a FortiManager, FortiGate, FortiSwitch or FortiAP API or MIB, or debugs an FMG/FortiOS error."
---

# Fortinet API conventions

Learned operating a fleet of ~190 FortiGates (mostly FortiManager-managed, some standalone),
~780 FortiAPs and their FortiSwitches from a Node.js management application. Vendor behavior
only; the app's own names are left out.

## The ten rules

1. **FortiManager auth is a permanent api-key over the `Authorization: Bearer` header.**
   FMG 7.4.7+ / 7.6.2+ removed `access_token` in the query string. A REST API Admin's api-key
   is permanent and shares ONE session per user — **never call `/sys/logout`** (or `/sys/login`):
   those are for session auth. A periodic logout tears the shared session out from under every
   other process using the key and shows up as RPC `-11` "no valid session" churn.
2. **Classify RPC faults before retrying.** Retry only transient faults (HTTP 5xx, network
   errors) with a small bounded backoff (≤2 retries), serialized per FMG. Fail fast on
   permanent ones: 401/403/404/405 and JSON-RPC `-11`. Retrying `-11` only amplifies the churn.
3. **Two transports to a managed gate, one narrow lane.** FMG's `/sys/proxy/json` relays
   `/api/v2/...` calls to a device through the manager; treat it as a **concurrency-1 lane per
   FMG** and keep native FMG reads (`/dvmdb`, CMDB) in a separate, wider lane. "Direct" mode uses
   FMG only to enumerate devices and talks REST to each gate with its own api-user token.
4. **The roster is `/dvmdb/adom/<adom>/device`.** It answers even for a gate that is offline
   (`conn_status !== 1`), and carries `sn`, `ha_slave[]`, connection state and platform. One
   roster read serves every gate — cache it per tick instead of asking per device.
5. **Identity is the serial, and in HA the serial is a SET.** FMG flips a cluster's top-level
   `sn` to whichever member is active, so compare against `{sn} ∪ ha_slave[].sn`, never a single
   value — or every failover reads as a chassis replacement. A device NAME cannot tell a rename
   from a replacement.
6. **CMDB is the config; monitor is the state.** In proxy mode read config natively from FMG's
   device database (works offline, lags) and live state (`/api/v2/monitor/...`) through the
   proxy (needs the gate reachable). Never take monitor output as configuration truth or vice
   versa. FMG's proxy also filters fields on some objects — see the reference.
7. **Reads that "succeed" with nothing are not answers.** A walk or list that returned an
   error must not be cached as "empty"; an unreadable roster means unknown, not "manages none".
   Distinguish absent / `[]` / populated everywhere you store what a gate manages.
8. **Write through the manager when the device is managed.** Under central management a
   device-side change must be mirrored into FMG's database with JSON-RPC `update` (never
   `set`, which replaces the object); otherwise the next install/push reverts it.
9. **Management access is a per-interface policy on the gate and a per-controller policy for
   switches.** A FortiGate's `allowaccess` lists what an interface accepts; a managed FortiSwitch
   has no profile of its own — its access comes from the controller's
   `switch-controller security-policy local-access` (`internal-allowaccess` /
   `mgmt-allowaccess`). Unknown must never render as "nothing permitted".
10. **FortiSwitch and FortiAP SNMP lie in specific, documented ways** (see the SNMP
    reference): `ifDescr` is the operator's port description, the FDB is indexed by row number,
    trunk names are truncated peer serials, and the FortiAP MIB carries no `UNITS` clause.

## Which file

| Topic | Read |
|---|---|
| JSON-RPC envelope, auth, fault classes, lanes, `/dvmdb`, ADOMs, offline gates, proxy field filtering, mirroring writes | [references/fmg-jsonrpc.md](references/fmg-jsonrpc.md) |
| FortiOS REST: api-user tokens per gate, monitor vs cmdb paths, DHCP/ARP/system reads, managed-switch and managed-AP status, `allowaccess`, quarantine, per-model workarounds | [references/fortios-rest.md](references/fortios-rest.md) |
| the data shapes behind IP management: DHCP server scopes, reserved addresses, leases, VIPs, interface IPs, what is device-owned vs assignable, lease release | [references/dhcp-vip-shapes.md](references/dhcp-vip-shapes.md) |
| FortiSwitch / FortiAP SNMP: which tables, which quirks, which MIB objects have no units, trunk naming, PoE, VLAN bitmaps | [references/fortiswitch-fortiap-snmp.md](references/fortiswitch-fortiap-snmp.md) |

## Parity rule

FortiManager-managed and standalone FortiGates talk to the same FortiOS via different
transports. Build push / release / quarantine / description-sync as a transport abstraction
dispatched on integration type, and ship every feature on both paths unless it is structurally
FMG-only (multi-device filters, ADOM scoping, proxy-lane tuning).
