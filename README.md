# rgex driver 

`rgex` is a Solaris 11.4 GLDv3 network driver for the Realtek `RTL8125B` 2.5GbE controller (`pci10ec,8125`).

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
