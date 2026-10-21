# OpenWrt for the TP-Link OnHub (TGR1900) as an access point

Fixes and build configurations for running OpenWrt 25.12 on the **TP-Link OnHub
TGR1900** (`ipq806x/chromium`, board `tplink,onhub`) as a bridged Wi-Fi access point,
optionally with an OpenThread Border Router (OTBR).

> [!WARNING]
> **Access point use only. Don't use these images on an OnHub that works as a router.**
> The Ethernet bridging workaround ([fix 1](#1-bridging-the-two-ethernet-ports-dsa-workaround))
> puts both Ethernet ports on the same CPU port of the switch. That's what makes bridging
> them work for an AP, but it's expected to break routing between them, and these images
> haven't been tested as a router.

Stock OpenWrt 25.12 boots on the TGR1900 but has several problems that make it a poor AP:

- **The two Ethernet ports can't be bridged.** An AP puts both ports into one bridge. With
  stock 25.12, the LAN port then can't reach the AP or anything behind it, because DSA puts
  each port on a different CPU port of the switch, and that combination is broken on the
  IPQ8064 ([openwrt#17891](https://github.com/openwrt/openwrt/issues/17891)). This repo
  works around it by running both ports through one CPU port.
- On some units, all three radios fail to load their calibration data.
- The 5 GHz radio's throughput collapses to about 70 Mbps.

This repo carries the fixes, plus a few for the OTBR role.

This isn't an official OpenWrt project. The fixes have been running on a small home fleet
(four TGR1900 units). The OnHub is a TP-Link product; the ASUS OnHub isn't covered here.

## Hardware

- SoC: Qualcomm IPQ8064 (dual Krait, 1.4 GHz)
- Switch: QCA8337 (DSA, two CPU ports: `eth0` via RGMII, `eth1` via SGMII)
- Wi-Fi: three QCA988X-family radios on PCIe, driven by `ath10k-ct`
  - `pcie0`: QCA9882 2.4 GHz 1×1
  - `pcie1`: QCA9880 5 GHz 3×3, the main AP radio
  - `pcie2`: QCA9882 2.4/5 GHz 1×1
- Bootloader: ChromeOS depthcharge, which passes the radio calibration data in the ChromeOS
  VPD and in the device tree

## Branches

All branches start from a commit that imports an OpenWrt release's source tree, followed
by this README. Each import is tagged `base/<version>`, and newer releases are merged in
(currently v25.12.6; see [Why there's no upstream history](#why-theres-no-upstream-history)).

| Branch | Role | Adds |
|---|---|---|
| `ap` | Bridged access point | Fixes 1–3, LuCI, tcpdump |
| `ap-diag` | AP with diagnostics tools | `ap` + iperf3, ethtool, sysstat, mosquitto-client-ssl, tmux, lua |
| `otbr` | AP + OpenThread Border Router | Fixes 1–3, OTBR, fixes 4–6, LuCI, tcpdump, strace, lsof, mosquitto-client-ssl, tmux, lua |

Each branch carries its own `.config`.

## The fixes

### 1. Bridging the two Ethernet ports (DSA workaround)

The QCA8337 switch has two CPU ports, wired to the SoC's `eth0` (RGMII) and `eth1`
(SGMII). OpenWrt's `board.d/02_network` assigns the OnHub's ports to different ones:
`wan` uses `eth0` and `lan1` uses `eth1`. For a bridged AP, `wan` and `lan1` both go
into `br-lan`, and then the AP, and everything behind it, can't be reached from the LAN
port, only from the WAN port.

This is OpenWrt bug [#17891](https://github.com/openwrt/openwrt/issues/17891), which
affects several dual-CPU-port ipq806x boards. The switch driver's multi-CPU-port support
comes from OpenWrt's pending `qca8k` patches, not mainline Linux. It programs the host's
forwarding entries for a bridge on only one of the two CPU ports, so only user ports on
that CPU port can reach the host.

A fix for this (PR [#21317](https://github.com/openwrt/openwrt/pull/21317)) was merged
into OpenWrt's development branch in March 2026, but it isn't in any 25.12 release. On
this hardware it made the LAN port reachable but duplicated local traffic: 19 of 20 pings
came back twice, latency doubled and throughput halved. The development branch has since
reworked those patches; they'll be worth retesting at the next major release.

**The workaround:** the `fixboard` init script runs at boot (START=08, before the network
comes up) and rewrites `/etc/board.json` so both ports use `eth0`, as also suggested in the
issue thread. `eth1` goes unused. Both ports then share one CPU port, which is why these
images are for AP use only (see the warning at the top).

*File:* `target/linux/ipq806x/base-files/etc/init.d/fixboard`

### 2. Radio calibration data on units without VPD

Since 25.12.0, the `11-ath10k-caldata` firmware hotplug script reads each radio's
calibration data from the ChromeOS VPD (`/sys/firmware/vpd/ro/wifi_base64_calibration{0,1,2}`,
provided by `kmod-google-firmware`). Whether Linux gets the VPD depends on the unit's
ChromeOS firmware. On my units, the ones updated to the January 2016 firmware (romstage
`coreboot-c4d26d6`) pass it, and the one still on the original May 2015 firmware (romstage
`coreboot-9958ccb`) doesn't: the modules load, but `/sys/firmware/vpd` doesn't exist, and
all three radios fail with `failed to run otp: -22 (download-cal-data)`. Depthcharge also
passes the same data in the device tree, so the script now uses the VPD when it's there and
the device-tree copy otherwise. `grep romstage /sys/firmware/log` shows which firmware a
unit runs.

To check, `dmesg | grep -E 'ath10k_pci .*(htt-ver|cal)'` should show `cal file` for each
radio. `otp calibration failed: 2` on its own is expected: the chips have no OTP calibration
fuses, so they use the data the script supplies.

*File:* `target/linux/ipq806x/base-files/etc/hotplug.d/firmware/11-ath10k-caldata`

### 3. 5 GHz throughput stuck around 70 Mbps

The ath10k-ct beta firmware (build 023) for QCA988X has a rate-control regression: the rate
stays at MCS 3, one spatial stream, even right next to the AP. This pins QCA988X to the
stable build 022 (`firmware-2-ct-full-community-22.bin.lede.022`). With it, iperf3 reaches
300–518 Mbps on the 3×3 radio.

*File:* `package/firmware/ath10k-ct-firmware/Makefile`

### 4. accept_ra on the services interface (`otbr`)

A bridging AP needs `net.ipv6.conf.*.forwarding=1`, and with forwarding on, the kernel
ignores router advertisements unless `accept_ra` is 2. On the interface the border router
uses as its backbone (in my case `br-lan.5`), that means no on-link
prefix or SLAAC address, and hosts on the LAN can't reach the Thread network. A hotplug
script sets `accept_ra=2` when that interface comes up.

*File:* `target/linux/ipq806x/base-files/etc/hotplug.d/iface/95-services-accept-ra`
(the interface name is site-specific; adjust it to yours)

### 5. otbr-agent leaks file descriptors until it crashes (`otbr`)

`otbr-agent` crashed every few days with `otPlatInfraIfHasAddress() ... No file
descriptors available`. Each time mDNS reported a service instance again, otbr-agent's
mDNSResponder backend (`src/mdns/mdns_mdnssd.cpp`) started a new resolution without
dropping the previous one, so the old resolution's `DNSServiceGetAddrInfo` connection to
`mdnsd` was never closed. In my case the trigger was the border router's TREL browse
rediscovering its own `_trel._udp` service, which leaked one connection per browsed
interface each time.

This was fixed upstream in ot-br-posix [#3003](https://github.com/openthread/ot-br-posix/pull/3003)
(August 2025), after the 2025-06-12 commit that OpenWrt 25.12 builds.
`package/net/openthread-br/patches/102-mdns-fix-ServiceInstanceResolutions-leak-in-mdnssd.patch`
backports it. To test it, a `_trel._udp` service was announced and withdrawn 10 times from
another host: otbr-agent saw 30 adds and 30 removes across its three interfaces, and its fd
count didn't change.

Earlier versions of this repo carried my own patch instead. It stopped the leak but also
cut off later address updates for resolved services. My report,
[`fd-leak-report.md`](https://github.com/jeback1/openwrt-onhub-tgr1900/blob/otbr/fd-leak-report.md)
on the `otbr` branch, was submitted as
[ot-br-posix#3649](https://github.com/openthread/ot-br-posix/issues/3649); the maintainer's
reply there explains the actual design.

`openthread-br` and `mdnsresponder` (bumped to 2881.120.11) are kept as in-tree packages
under `package/net/`, so `./scripts/feeds update` doesn't wipe the changes. See
[Building](#building) for the catch.

### 6. Uplink watchdog (`otbr`)

In my use case, the AP running otbr was connected to the network via a WDS link. If the AP
at the far end of a WDS link has its firmware crash and recover, the 4-address
peer may never come back while both sides still look associated. The OTBR's backbone link
then stays dead until the next group rekey (24 h by default). `uplink-watchdog` pings a host
on the far side every 30 s. After 4 failures in a row it restarts the radio carrying the
station interface, then waits 5 minutes before checking again.

The ping target has no default and must be set; until it is, the watchdog doesn't start:

```sh
uci set system.uplink_watchdog=uplink_watchdog
uci set system.uplink_watchdog.target='fe80::...'   # e.g. the router's link-local address
uci commit system
/etc/init.d/uplink-watchdog restart
```

`network` (default `services`), `interval`, `failures`, `holdoff` and `enabled` can be set
the same way.

*Files:* `target/linux/ipq806x/base-files/usr/sbin/uplink-watchdog`,
`target/linux/ipq806x/base-files/etc/init.d/uplink-watchdog`

## AP configuration notes

These live in `/etc/config` on the device, not in the image:

- **IPv6 on the AP itself.** With `forwarding=1` and no `ip6assign`, odhcp6c installs only
  source-specific default routes, and the AP can't reach the IPv6 internet. Remove
  `ip6assign` from `lan` and add an interface with `proto dhcpv6`, `reqaddress try`,
  `reqprefix no` and `sourcefilter '0'`. In `/etc/config/dhcp`, set `ra 'disabled'` and
  `dhcpv6 'disabled'` on `lan` so the AP doesn't send RAs that compete with the router's.
- **Thread dataset.** The `openthread` netifd protocol handler applies the active dataset
  from `option dataset` (hex TLVs) on the `thread` interface in `/etc/config/network`. That
  file includes the Thread network key, which is why no dataset is stored in this repo.
  `sysupgrade` keeps `/etc/config` by default.

## Building

```sh
git clone -b ap https://github.com/jeback1/openwrt-onhub-tgr1900.git   # or ap-diag / otbr
cd openwrt-onhub-tgr1900
./scripts/feeds update -a
./scripts/feeds install -a
make defconfig
make -j$(($(nproc)+1))
```

The image is `bin/targets/ipq806x/chromium/openwrt-ipq806x-chromium-tplink_onhub-squashfs-factory.bin`.

**On `otbr`:** `./scripts/feeds install -a` recreates the
`package/feeds/packages/mdnsresponder` and `package/feeds/packages/openthread-br` symlinks,
which clash with the in-tree copies. Remove them before building:

```sh
rm -f package/feeds/packages/mdnsresponder package/feeds/packages/openthread-br
```

**`.config` hygiene.** The configs started from the official `config.buildinfo`, which sets
`CONFIG_BUILDBOT=y`. That makes `ALL_NONSHARED` (and through it `ALL_KMODS`) default to
`y`. Deleting those lines doesn't stick, since `make defconfig` refills unset symbols from
their defaults. They're set to `# ... is not set` explicitly instead.

## Why there's no upstream history

The repo starts from a commit containing a release's source tree rather than
OpenWrt's full history. It's about 15 MB instead of several hundred, and the history
holds only the changes for this device.

Moving to a new release adds one more import commit, whose parent is the previous import,
and merges it into the branches:

```sh
git remote add upstream https://github.com/openwrt/openwrt.git
git fetch upstream tag v25.12.7 --no-tags
new=$(git commit-tree 'v25.12.7^{tree}' -p base/v25.12.6 -m 'Import OpenWrt v25.12.7')
git tag base/v25.12.7 "$new"
git checkout ap && git merge base/v25.12.7
git checkout ap-diag && git merge ap          # ap-diag builds on ap
git checkout otbr && git merge base/v25.12.7
```

Then run `make defconfig` and commit the `.config` changes.

Each import commit's diff is exactly what changed upstream between the two releases, and
`git diff base/<version> ap` shows everything this repo changes on top of that release.
Merging means published history is never rewritten. (Through v25.12.6 the branches were
rebased instead, so v25.12.6 was published with a one-time force-push.) Don't push the
upstream tags; they carry the full OpenWrt history.

## License

OpenWrt is GPL-2.0 (see `COPYING` and `LICENSES/`). The changes here are under the same
terms as the files they modify.
