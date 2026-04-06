# rgex driver 

`rgex` is a Solaris 11.4 GLDv3 network driver for the Realtek `RTL8125B` 2.5GbE controller (`pci10ec,8125`).

This driver was created by an AI coding agent.

The development and validation NIC used in this repository is `RTL8125B`.
Some implementation notes still mention `RTL8125B` because the low-level register, descriptor, PHY, and vendor-driver behavior is often described as the
`RTL8125B` family path. In this README, the intended hardware name is `RTL8125BG`.

Target platform:

- Solaris 11.4 x86 SRU 90 and later (including CBE 90)
- Realtek `RTL8125B`

Tested throughput on the current stable configuration:

- IPv4 TX: about `2.35 Gbps`
- IPv4 RX: about `2.19 Gbps`
- IPv6 TX: about `2.32 Gbps`
- IPv6 RX: about `2.02 Gbps`
  
Implemented features:

- device attach for `pci10ec,8125`
- link handling for `10/100/1000/2500` Mbps
- autonegotiation and pause / asym-pause advertisement control
- Solaris MAC property support through `dladm`
- VLAN RX/TX handling
- runtime MTU control up to `9000`
- RX checksum offload
- practical TX checksum offload for IPv4 TCP/UDP traffic
- MSI interrupts with FIXED fallback
- MSI-X infrastructure with safe shared mode as the production default
- warm reboot recovery
- private kstats and diagnostic visibility

## Feature Status Matrix

Status legend:

- `Implemented`
- `Partial`
- `Not implemented`

| Feature | Public RTL8125BG capability | Current `rgex` status | Notes |
| --- | --- | --- | --- |
| Basic attach / GLDv3 registration | NIC controller support | Implemented | Driver attaches as `pci10ec,8125`, registers with GLDv3, and exposes normal MAC callbacks. |
| Link speeds | 10/100/1000/2500 | Implemented | The current stable path supports 2.5G advertisement and reports `2500/full`. |
| Auto-negotiation | Advertised publicly | Implemented | The default path keeps autoneg enabled and supports link-property control. |
| Pause / asym-pause | Advertised in live link state | Implemented | Pause advertisement and Solaris `flow-control` handling are available. |
| VLAN RX/TX | Standard NIC feature | Implemented | RX de-tag and TX VLAN tag handling are available. |
| Jumbo frame support | Up to 16K in public materials | Partial | The current runtime MTU cap is `9000`, not the public `16K` maximum. |
| TX HW checksum offload | IPv4, IPv6/TCP, IPv6/UDP checksum offload | Partial | Practical IPv4 TCP/UDP offload is validated. IPv6 TX offload is not fully exercised by the current Solaris 11.4 stack. |
| RX HW checksum offload | IPv4, IPv6/TCP, IPv6/UDP checksum offload | Implemented | RX checksum indication is active for IPv4 and IPv6 receive traffic. |
| MSI interrupts | MSI / MSI-X | Implemented | Attach prefers MSI when available. |
| FIXED interrupts | Platform fallback | Implemented | Clean fallback exists if MSI registration fails. |
| MSI-X interrupts | MSI / MSI-X | Implemented | Mode 0 (safe shared) is the production default. Mode 1 (small-layout) is validated. Mode 2 needs `22+` vectors for the full ISR v2 path. |
| GLDv3 rings / multi-ring exposure | Needed for Solaris-side scaling | Partial | Ring capability is exposed, but production is intentionally single-ring. |
| RSS / receive-side scaling | Publicly advertised | Partial | RSS programming exists, but the current host remains single-ring in production. |
| LSO / large send offload | Publicly advertised | Partial | LSO/TSO is implemented for IPv4 TCP, but remains intentionally disabled in the default production configuration. |
| EEE / 802.3az | Publicly advertised | Partial | A practical subset exists, but this is not yet full public-feature parity. |
| Wake-on-LAN / RealWoW | Publicly advertised | Not implemented | No WoL / RealWoW support is currently provided. |
| 1588v2 / PTP timestamps | Publicly advertised | Not implemented | No timestamp capability is currently exposed. |
| AVB / 802.1Qav | Publicly advertised | Not implemented | No AVB-specific Solaris integration is currently provided. |
| Suspend / resume plumbing | Platform lifecycle support | Partial | Suspend/resume entry points exist, but this is not equivalent to WoL / RealWoW support. |
| Warm reboot recovery | Driver-specific reliability support | Implemented | Stale DMA state is cleared safely after reboot or reload. |
| Observability / kstats | Driver-specific support | Implemented | The driver exposes operational and diagnostic visibility through kstats. |

Cautions:

- production use is currently centered on a single RX/TX ring
- do not enable multi-ring, RSS, LSO, or TSO in deployed configurations
  - multi-ring/RSS remains frozen on the current host until a `22+` MSI-X vector environment is available
  - LSO/TSO is implemented but intentionally disabled in the default production configuration
- jumbo support is currently capped at `9000` MTU
- PTP / IEEE 1588v2, AVB, and WoL / RealWoW are not implemented

This README covers only these two files:

- `rgex` driver binary
- `rgex.conf.sample`

## Files

- driver binary source:
  `build/release/amd64/rgex`
- sample config source:
  `rgex.conf.master`

Recommended installed locations:

- `/kernel/drv/amd64/rgex`
- `/kernel/drv/rgex.conf`

## Manual Install

Install the driver binary:

```sh
# install -m 0755 build/release/amd64/rgex /kernel/drv/amd64/rgex
```

Install the sample configuration as the active driver configuration:

```sh
# install -m 0644 rgex.conf.master /kernel/drv/rgex.conf
```

If you want to keep the sample without activating it yet:

```sh
# cp rgex.conf.master rgex.conf.sample
```

and later install it with:

```sh
# install -m 0644 rgex.conf.sample /kernel/drv/rgex.conf
```

## After Install

Bind the driver to the device:

```sh
# add_drv -i 'pci10ec,8125' rgex
```

If the driver is already installed and you replaced the files, reload it with your normal Solaris driver maintenance procedure.

Use at your own risk.

## Intel CPU Platform Notes

This document records the operational caveats for running the `rgex` driver on
Intel CPU platforms.
All validation to date has been performed on an AMD platform
(AMD Family 17h / Zen family), and operation on Intel platforms remains
unverified.

### Test Environment (AMD)

| Item | Value |
|------|-------|
| CPU | AMD (Zen family / Family 17h) |
| Chipset | AMD root complex |
| MSI-X vector budget | 8 |
| IOMMU | AMD-Vi |
| PCIe | Gen3 x1 (RTL8125BG 2.5GbE) |

### Risk Summary

| Area | AMD behavior | Intel expectation | Risk |
|------|--------------|------------------|------|
| AER CA masking | enabled (AMD gate) | disabled (Intel skip) | **none** - safely skipped |
| MSI-X vectors | 8 vectors allocated | platform dependent | **low** - fallback exists |
| APIC ID retarget | successful | unverified | **low** - standard API used |
| ASPM disable | successful | unverified | **low** - standard PCIe registers |
| IOMMU (VT-d) | AMD-Vi stable | unverified | **low** - DDI abstraction |
| Memory barriers | stable | same | **none** |
| Cache line | 64B | 64B | **none** |
| DMA sync | stable | same | **none** |


