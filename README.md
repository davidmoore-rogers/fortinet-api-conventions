# fortinet-api-conventions

A Claude Code plugin carrying the FortiManager / FortiOS / FortiSwitch / FortiAP API and MIB
knowledge learned operating a ~190-gate, ~780-AP fleet from a Node.js management app. Vendor
behavior only — no application-specific paths.

## Use it in another project

```
claude --plugin-dir C:\Users\dmoore\VSCode\fortinet-api-conventions
```

The skill `fortinet-api-conventions` auto-loads whenever code calls a FortiManager,
FortiGate, FortiSwitch or FortiAP API or MIB, or debugs an FMG/FortiOS error. Invoke by hand
with `/fortinet-api-conventions:fortinet-api-conventions`.

## Layout

```
.claude-plugin/plugin.json
skills/fortinet-api-conventions/SKILL.md
skills/fortinet-api-conventions/references/fmg-jsonrpc.md
skills/fortinet-api-conventions/references/fortios-rest.md
skills/fortinet-api-conventions/references/dhcp-vip-shapes.md
skills/fortinet-api-conventions/references/fortiswitch-fortiap-snmp.md
```

Bump `version` in `.claude-plugin/plugin.json` when the content changes materially.
