# Extreme VOSS by SNMP — Zabbix Template

## Overview

Zabbix **7.4** template for **Extreme Networks VOSS / Fabric Engine** switches via SNMPv2c or SNMPv3.

Uses master `walk[]` / `get[]` items with dependent discovery and metrics, aligned with official network-device template practices (`{$IFCONTROL}`, `{$NET.IF.*}` filters, ICMP suite, chassis inventory/hardware).

| Area | Source |
|---|---|
| Availability | ICMP ping / loss / response time; SNMP agent availability |
| Inventory | ENTITY-MIB chassis (serial, model, firmware) |
| System | SNMPv2-MIB |
| CPU / memory | RAPID-CITY KHI `rcKhiSlotPerfTable` |
| Hardware | Temperature, fans, power supplies |
| Virtual IST | Discovered when `rcVirtualIstVlanId` ≠ 0; session status only |
| Classic IST | Discovered when `rcMltIstSessionEnable` = enable; session status |
| MLT / LACP | Enabled MLTs; aggregation state, aggregatable |
| IS-IS / SPB | Circuits: oper state, adjacency counts, type |
| NTP | Enabled NTPv4 servers: synchronized / reachable (5m) |
| SLPP Guard | Ports with guard enabled; status / timer / timeout / packet count (labels = ifIndex) |
| BPDU Guard | Ports with guard enabled; remaining disable timer / timeout (shared `rcPort` walk w/ Auto-Sense) |
| Auto-Sense | Ports with Auto-Sense enabled; operational state |
| EAPOL / 802.1X | Ports with PortControl=auto; PAE / backend / port status (labels = ifIndex) |
| RADIUS | Enabled `rcRadiusServHost` entries; RTT, pending, retries (5m) |
| Interfaces | IF-MIB (HC counters, util, errors, discards, speed, status) — official-style `Interface {#IFNAME}({#IFALIAS}): …` names |

`rcSysCpuUtil` / `rcSysDram*` are **not supported on VOSS** and are not used.

---

## Setup

1. Import `template_extreme_switch_voss.yaml` (**Configuration → Templates → Import**).
2. Assign **Extreme VOSS by SNMP** to the host.
3. Configure an SNMP interface (IP, port 161, v2c or v3 credentials).
4. To ignore link-down on a specific interface, set `{$IFCONTROL:"ifName"}=0` on the host.
5. To ignore LACP-down on a specific MLT, set `{$MLTCONTROL:"mltName"}=0` on the host.
6. To ignore IS-IS circuit problems, set `{$ISISCONTROL:"circuitIndex"}=0` on the host.
7. To ignore EAPOL issues on a port, set `{$EAPOLCONTROL:"ifIndex"}=0` (context is `{#SNMPINDEX}`).
8. Tune other macros as needed (including `{$NTP.SYNC.PROBLEM.MATCHES}` / `{$NTP.REACHABLE.OK}` if device status strings differ).

---

## Macros

| Macro | Default | Description |
|---|---|---|
| `{$CPU.UTIL.CRIT}` / `{$CPU.UTIL.WARN}` | `90` / `80` | CPU % thresholds (5m avg) |
| `{$MEMORY.UTIL.CRIT}` / `{$MEMORY.UTIL.WARN}` | `90` / `80` | Memory % thresholds (10m avg) |
| `{$IF.UTIL.CRIT}` / `{$IF.UTIL.WARN}` | `95` / `90` | Interface bandwidth % (context: `"{#IFNAME}"`) |
| `{$IF.ERRORS.WARN}` | `2` | Interface error rate eps (context supported) |
| `{$IFCONTROL}` | `1` | Link-down fires only where context equals `1` (`{$IFCONTROL:"ifName"}=0` to ignore) |
| `{$MLTCONTROL}` | `1` | LACP-down fires only where context equals `1` |
| `{$ISISCONTROL}` | `1` | IS-IS triggers fire only where context equals `1` |
| `{$EAPOLCONTROL}` | `1` | EAPOL triggers fire only where context equals `1` (`"{#SNMPINDEX}"`) |
| `{$RADIUSCONTROL}` | `1` | RADIUS triggers fire only where context equals `1` |
| `{$RADIUS.RETRIES.WARN}` | `0.1` | RADIUS client retries/s warning |
| `{$NTP.SYNC.PROBLEM.MATCHES}` | `(?i)^(rejected\|discarded\|not[\s-]?synchronized)` | Unhealthy NTP sync CLI strings (not Candidate/Selected) |
| `{$NTP.REACHABLE.OK}` | `Reachable` | Expected NTP reachability (`eq`) |
| `{$ICMP.LOSS.WARN}` | `20` | ICMP loss % |
| `{$ICMP.RESPONSE.TIME.WARN}` | `0.15` | ICMP RTT seconds |
| `{$SNMP.TIMEOUT}` | `5m` | SNMP availability window |
| `{$NET.IF.IFNAME.NOT_MATCHES}` | Mgmt / loopback regex | Skip Mgmt / loopback-style names |
| `{$NET.IF.IFADMINSTATUS.NOT_MATCHES}` | `^2$` | Skip admin-down |
| `{$NET.IF.IFTYPE.MATCHES}` | `^(6\|161)$` | ethernetCsmacd + LAG; set `.*` for all |
| `{$PSU.STATUS.NOT_MATCHES}` | `^2$` | Skip empty PSU slots |

Additional `{$NET.IF.*.MATCHES}` / `NOT_MATCHES` macros follow the official filter pattern.

---

## Discovery

| Name | Key | Notes |
|---|---|---|
| Virtual IST | `extreme.vist.discovery` | Only if VLAN ID ≥ 1 |
| Classic IST | `extreme.ist.discovery` | Only if `rcMltIstSessionEnable` = enable(1) |
| MLT / LACP | `extreme.mlt.discovery` | Only if `rcMltEnable` = true(1) |
| IS-IS circuits | `extreme.isis.circuit.discovery` | All `rcIsisCircuitTable` entries |
| NTP servers | `extreme.ntp.discovery` | Only if `rcNtpv4ServerEnable` = true(1) |
| SLPP Guard | `extreme.slpp.guard.discovery` | Only if `rcSlppPortGuardEnable` = true |
| BPDU Guard | `extreme.bpdu.guard.discovery` | Only if `rcPortBpduGuardAdminEnabled` = true |
| Auto-Sense | `extreme.autosense.discovery` | Only if `rcPortAutoSense` = enable(2) |
| EAPOL / 802.1X | `extreme.eapol.discovery` | Only if PortControl = auto(2) |
| RADIUS servers | `extreme.radius.discovery` | Only if `rcRadiusServHostEnable` = true(1) |
| Chassis inventory | `system.hw.chassis.discovery` | ENTITY-MIB class chassis(3) |
| KHI slots | `extreme.khi.slot.discovery` | CPU / memory |
| Temperature sensors | `extreme.temp.discovery` | Device warn/crit thresholds |
| Fans | `extreme.fan.discovery` | Status / speed / RPM (excludes notPresent; stable LLD) |
| Power supplies | `extreme.psu.discovery` | Oper status |
| Network interfaces | `net.if.discovery` | Filtered by `{$NET.IF.*}` |

---

## Triggers (summary)

- ICMP unavailable / high loss / high RTT
- No SNMP data collection
- Slot critical/high CPU and memory
- Temperature at device warn/crit thresholds
- Fan / PSU down
- vIST session down (only on discovered/configured vIST)
- Classic IST session down (only when IST enabled)
- MLT LACP aggregation down (`{$MLTCONTROL}`; only when aggregatable)
- IS-IS circuit down / no UP adjacencies (`{$ISISCONTROL}`)
- NTP server sync problem / unreachable
- SLPP Guard blocked interface (status = blocking)
- BPDU Guard shut interface (`rcPortShutdownReason` = bpduGuard)
- Auto-Sense NNI auth fail / NNI loop
- EAPOL held / stuck authenticating / backend fail-timeout (`{$EAPOLCONTROL}`)
- RADIUS pending stuck / high retries (`{$RADIUSCONTROL}`)
- Interface link down (`{$IFCONTROL}`), util, errors (with hysteresis)
- Device restarted (uptime < 10m)

---

## Compatibility

- **Zabbix**: 7.4+
- **SNMP**: v2c or v3
- **Platforms**: Extreme VOSS / Fabric Engine
- **MIBs**: RAPID-CITY, ENTITY-MIB, IF-MIB, IEEE8021-PAE-MIB, SNMPv2-MIB
- **Author**: Jon McLaughlin

**Note:** Macro names use dotted style (`{$CPU.UTIL.CRIT}` etc.). Re-apply host overrides after import if you used older `{$CPU_HIGH}`-style macros.
