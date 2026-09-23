# FortiOS REST (direct to a FortiGate)

## Authentication and reachability

- Per-gate **api-user token** over `Authorization: Bearer`; HTTPS on the gate's admin port
  (443 or the configured `admin-sport`). Pin or verify the certificate deliberately — most gates
  run a self-signed or factory cert; make verification a per-credential switch, never silently off.
- A fleet where every gate carries its OWN api-user cannot be polled with one shared token.
  Resolve the credential per device (device credential → integration default), and never let a
  credential's configured host override the device address: a wrong host mis-attributes one
  device's data to twenty.
- Under FortiManager proxy transport with no device token, REST reads are impossible; fall
  back to ICMP for reachability and to FMG's `/dvmdb` for state, and say so in the UI rather than
  letting a "rest_api" method silently collect nothing.

## Paths

| Need | Path | Notes |
|---|---|---|
| system status, hostname, serial, version | `/api/v2/monitor/system/status` | serial here is the chassis; HA cluster members differ |
| interfaces + addresses | `/api/v2/cmdb/system/interface`, `/api/v2/monitor/system/interface` | cmdb = configured, monitor = live counters |
| DHCP servers / scopes | `/api/v2/cmdb/system/dhcp/server` | `reserved-address[]` lives INSIDE each server object |
| DHCP leases | `/api/v2/monitor/system/dhcp` | per-scope lease table; supports `?scope=` |
| ARP table | `/api/v2/monitor/network/arp` | L3 neighbor cache (IP, MAC, interface) |
| VIPs | `/api/v2/cmdb/firewall/vip` | device-owned addresses; never "assignable" |
| managed switches | `/api/v2/monitor/switch-controller/managed-switch/status`, `/api/v2/cmdb/switch-controller/managed-switch` | monitor may omit fields via FMG proxy |
| managed APs | `/api/v2/monitor/wifi/managed_ap`, `/api/v2/cmdb/wireless-controller/wtp` | radio[] and client counts are here even when the AP itself has SNMP off |
| wireless clients | `/api/v2/monitor/wifi/client` | |
| address groups (quarantine) | `/api/v2/cmdb/firewall/addrgrp`, `/api/v2/cmdb/firewall/address` | MAC-typed addresses |
| SD-WAN health | `/api/v2/monitor/virtual-wan/health-check`, `/api/v2/monitor/virtual-wan/members`, `/api/v2/cmdb/system/sdwan` | the selected route is not always exposed; label inferred values as inferred |
| DHCP lease release | `/api/v2/monitor/system/dhcp/revoke` | body `{ "ip": [...] }` |

Always pass `vdom=` where the gate has VDOMs; many monitor endpoints default to the
management VDOM only.

## Access-profile groups gate writes per CMDB tree

A REST API admin's access profile is granted per **group**, not per object, and the group
a tree belongs to is not always the one its menu location suggests. The trap that bites:

- `/api/v2/cmdb/system/global` (the device `alias`, hostname and other global settings) is
  in the **System** group (`sysgrp`). `system/interface` — interface descriptions and
  addresses — is in **Network** (`netgrp`). A profile with Network → Configuration Read-Write
  and System read-only lets an interface-description PUT through and refuses the alias PUT
  with a 403. A feature that writes both then looks half-working rather than broken.
- System Read-Write also covers administrators, access profiles and every global setting.
  There is no narrower grant that reaches `system/global`, so a token that needs it is
  admin-grade; scope it with `trusthost`.
- The same split applies when the writer reaches the gate directly while a FortiManager
  manages it: the gate's own access profile authorizes the call, not FMG's admin profile.
- Prove a grant before depending on it: `GET <path>?action=schema` returns the table's
  definition including its `access_group`, which names the profile group for that build.
  A plain GET proves read access only; the write can still be refused.

## CMDB vs monitor

`cmdb` is configuration (what the operator set); `monitor` is state (what is happening). A
reservation is cmdb; a lease is monitor. A managed switch's port descriptions are cmdb; its
port link state is monitor. Keep the two apart in your model — a "who owns this address"
question is answered by cmdb, a "who is on this address right now" question by monitor.

## Admission and connectivity are two different fields

On both managed-device tables the controller reports **two independent axes**, and reading
either one as "is this device healthy" is wrong.

| | admission (has the operator authorized it) | connectivity (is the session up right now) |
|---|---|---|
| `managed-switch/status` | `state` — `"Authorized"` / `"Unauthorized"` | `status` — `"Connected"` / `"Disconnected"` |
| `wifi/managed_ap` | `state` — `"authorized"` / `"discovered"` / … | `status` — `"online"` / `"connected"` / `"offline"` / `"discovered"` |

A device is routinely authorized-but-offline (powered down, cable pulled) or
discovered-but-connected (plugged in, never admitted). Note the case difference between the
two tables, and that `"discovered"` appears as a value on BOTH axes of the AP row meaning
different things.

**The AP connectivity vocabulary varies by firmware**: most releases report `"online"`, some
report `"connected"`. Match both or a healthy fleet reads as entirely offline. The switch
table has no such variance — `"Connected"` is the only healthy value.

**A device absent from the table is not the same as a device reported down.** It ages out
briefly during a config push and after a controller reboot, so "not listed" is genuinely
unknown; treating it as down produces a burst of false alarms on every push. For the same
reason the `cmdb` roster is the stable answer to "which devices does this controller manage"
while the `monitor` status table is only the answer to "what is their state right now" — a
switch can be missing from the second while present in the first.

## Management access

`allowaccess` on a `system/interface` object lists the protocols that interface accepts
(`ping https ssh snmp http …`). Read it, do not infer it from open ports. A managed FortiSwitch
has no per-device `allowaccess`; the controller's
`switch-controller security-policy local-access` object carries `internal-allowaccess`
(and `mgmt-allowaccess` for the out-of-band port) for the whole fleet the controller manages.
When neither can be read, the honest value is **unknown**, not "no access".

## Quarantine by address group

MAC-based quarantine works by adding a MAC-typed `firewall/address` to a quarantine
`firewall/addrgrp` that a deny policy references, on **every gate that has recently seen the
device** (sightings from DHCP/endpoint tables). Push per gate, report per gate, and only flip
your own state when at least one gate accepted. Release removes the addresses; a failed
device-side removal is reported, never allowed to block the release. Verify by re-reading the
group later — drift happens when someone edits the gate by hand.

## Small-branch models

Branch-class gates (60F/61F/91G class) sometimes answer monitor endpoints slowly or partially
under load; use longer per-request timeouts for them, never fewer retries, and prefer the
FMG-side roster for reachability rather than hammering the gate.

## Per-device change tracking

Firmware (`os_ver`/`build`), a client's switch port, its AP, and its gateway are the four
changes operators want as events. Compute them from an end-of-run baseline against the stored
row, not phase by phase — coarse-then-corrected phases within one run ping-pong otherwise.
