---
date: 2026-09-05
categories:
  - XGS-PON
  - XS-220X-A
  - WAS-110
  - Nokia
  - X-ONU-SFPP
  - TDS Telecom
description: Masquerade as the Nokia XS-220X-A with the WAS-110 or X-ONU-SFPP
slug: masquerade-as-the-nokia-xs-220x-a-with-the-was-110
---

# Masquerade as the Nokia XS-220X-A with the WAS-110 or X-ONU-SFPP

Replace a **Nokia XS-220X-A** ONT with a WAS-110 (or other MaxLinear PRX126 / 8311
SFP+ ONT stick) on an XGS-PON network. Verified on TDS Telecom.

<!-- more -->
<!-- nocont -->

!!! tip "The one non-obvious setting"

    On this ONT you must run with **`8311_fix_vlans` enabled**. With it disabled you
    reach O5 and get a public IP over DHCP, but no traffic passes — the OLT's raw
    extended-VLAN table strips the internet VLAN tag on the downstream, so a tagged
    router WAN silently drops every return frame. Enabling the fix re-applies the tag.

## Prerequisites

- A WAS-110 or X-ONU-SFPP running the **8311 community firmware**. If it is not
  flashed yet, follow *Install the 8311 community firmware on the WAS-110* first.
- Use a current 8311 release — this guide was verified on **v2.8.3 (basic)**. Older
  builds also expose `fix_vlans`, but staying current is recommended.
- Apply every `8311_*` setting below in the 8311 WebUI configuration page, or over
  SSH with `fw_setenv <name> <value>`. Reboot after changing them.

## Mandatory PON settings

Read these off the ONT's back label. The CLEI code and ONT P/N are identical for
every XS-220X-A; only the serial number is unique to your unit.

| 8311 setting          | Value                     | Source / notes                  |
|-----------------------|---------------------------|---------------------------------|
| `8311_gpon_sn`        | `ALCLxxxxxxxx`            | S/N on the label                |
| `8311_equipment_id`   | `BVMN300DRAXS220XA`       | CLEI `BVMN300DRA` + model        |
| `8311_hw_ver`         | `3TN00617AAAA`            | ONT P/N                         |
| `8311_cp_hw_ver_sync` | `1`                       | sync circuit-pack HW version    |
| `8311_pon_slot`       | `10`                      | internet UNI slot (see notes)   |
| `8311_mib_file`       | `/etc/mibs/prx300_1U.ini` | single Ethernet UNI personality |
| `8311_pon_mode`       | `xgspon`                  |                                 |
| `8311_override_active`| `A`                       | keep bank A active during SW DL |
| `8311_override_commit`| `A`                       | keep bank A committed           |

The override flags stop the OLT's OMCI software-image download from switching or
flashing banks while it aligns the reported version (next section).

## Firmware version

The ONT's local web UI is not reachable behind this deployment, so you cannot read
the shipped software version directly. You do not need to — the OLT will hand it to
you:

1. Seed a syntactically valid placeholder in both banks, e.g.
   `8311_sw_verA` / `8311_sw_verB` = `3TN00669AOCK59`.
2. Set the firmware-match regex for this vendor:
   `8311_fw_match_b64` = `KDNUTlswLTlBLVpdezExfSkk` (decodes to `(3TN[0-9A-Z]{11})$`).
   The default `3FE...` match will not validate a Nokia `3TN...` build.
3. On first association the OLT runs an OMCI software-image download and rewrites
   `8311_sw_verA`/`B` to the value it expects. With the override flags set, the 8311
   firmware persists the string to the environment rather than flashing anything.
   Allow ~10-15 minutes.
4. Read it back with `fw_printenv | grep sw_ver`. That value now persists.

## ISP fixes

!!! warning "Enable Fix VLANs"

    Set `8311_fix_vlans` to `1`. The `vlansd` daemon then installs a downstream
    tag-modify rule on the internet p-mapper so return traffic keeps the internet
    VLAN tag. This is re-applied automatically on every boot from the environment
    flag, so it is permanent.

Set the internet VLAN to match your line (TDS uses `474`):

```
8311_fix_vlans      1
8311_internet_vlan  474
```

## Router configuration

Configure the router WAN interface for **DHCP** and tag it with the internet VLAN
(`474` on TDS). MAC cloning is not required.

## Notes

???+ info "Only one Ethernet UNI"

    The XS-220X-A has both a 1GE and a 10GE port, so the OLT provisions two Ethernet
    UNIs (OMCI instances 257 and 2561). An SFP+ stick has a single physical port and
    can satisfy only one; the second UNI fails to bind, which is harmless. Internet is
    carried on the slot-10 UNI (instance 2561), hence `8311_pon_slot=10` with the 1U
    MIB.

???+ info "Managing the stick"

    The stick's management address (`192.168.11.1`) is reachable only while the
    router WAN is untagged. For hands-on debugging, temporarily set the router WAN to
    a static `192.168.11.2/24` with no gateway (untagged).

## Verify

- `pon psg` should report state `5` (O5, Associated).
- `8311-extvlan-decode.sh` should show an extended VLAN table once the OLT has
  provisioned the subscriber.
- After enabling Fix VLANs, `tc filter show dev <internet-pmapper> ingress` shows the
  downstream `vlan modify id <internet-vlan>` rule.
