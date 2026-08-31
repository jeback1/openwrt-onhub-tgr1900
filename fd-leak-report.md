# Report to openthread/ot-br-posix regarding an otbr-agent fd leak

## Subject: `ServiceInstanceResolution` never released in `mdns_mdnssd.cpp` — otbr-agent leaks one `mdnsd` connection per browsed interface on every mDNS re-discovery of an already-resolved instance

## Hardware and software setup

TP-Link TGR1900 "OnHub" running OpenWrt 25.12.5, `ipq806x/chromium` target, configured as
an OpenThread Border Router bridging a Thread mesh to a LAN (no NAT64). ot-br-posix pinned
at commit `2f3c799c7463c8f674754e65c53f78bc0bbcbd58` (2025-06-12, package version `0.3.0`
via OpenWrt's `openthread-br` feed package), built with:

```
-DOTBR_MDNS=mDNSResponder -DOTBR_ENABLE_TREL=1 -DOTBR_BACKBONE_ROUTER=ON
-DOTBR_BORDER_ROUTING=ON -DOTBR_ENABLE_SRP_ADVERTISING_PROXY=1
-DOTBR_ENABLE_DNSSD_DISCOVERY_PROXY=1
```

mDNSResponder (`mdnsd`) version at time of testing: `2200.100.94.0.2` (also reproduced
after upgrading to `2881.120.11` — same behavior on both, so the bug is not
mDNSResponder-version-dependent).

Symptom: `otbr-agent` crashes roughly once a day-to-week with:
```
[C] Platform------: otPlatInfraIfHasAddress() at infra_if.cpp:72: No file descriptors available
```
`netifd` (OpenWrt's network manager) auto-restarts the `thread` interface immediately
after, which masked the symptom until fd counts were logged directly.

**Reproduction condition, which is probably why this hasn't been widely reported:** this
border router is the *only* one on its network, and `otbr-agent` browses `_trel._udp` for
TREL peers as part of normal operation. With no other border router to find, it
periodically rediscovers *itself* as its own mDNS record's TTL cycles (ordinary mDNS
behavior — ~6-7 minutes in our testing) — and each self-rediscovery leaks. A deployment
with other border routers on the same network, or where TREL/backbone routing isn't in
this exact configuration, may not exercise this code path the same way.

We also determined (empirically, by toggling it off and back on) that our own
`accept_ra`/routing configuration on the backbone interface was gating whether this
churn happened at all — worth mentioning only because it's *why* this surfaced for us at
a specific point in time, not because it's part of the actual bug. The underlying bug is
purely in how `otbr-agent` handles a repeat mDNS "add" event for an instance it has
already resolved.

## Diagnostics

fd counts sampled every 5 minutes (`otbr-agent`, `mdnsd`), both processes climbing in
lockstep:
```
timestamp,otbr_pid,otbr_fd_count,mdnsd_pid,mdnsd_fd_count
2026-08-28 09:51:07,10309,73,1930,50
...steady climb, +3/+6/+12 per sample, matching multiples of 6...
```
`ubusd` (OpenWrt's local RPC bus, also connected to by otbr-agent) stayed perfectly flat
throughout every test — ruling it out as the leak's peer and confirming `mdnsd` is the
counterpart.

`strace -f -tt -e trace=socket,connect,close -p <otbr-agent pid>` across a live leak event
(fd numbers reused from a prior burst, hence starting at 415):
```
15:14:10.043471 socket(AF_UNIX, SOCK_STREAM, 0) = 415
15:14:10.043856 connect(415, {sa_family=AF_UNIX, sun_path="/var/run/mdnsd"}, 110) = 0
15:14:10.046854 socket(AF_UNIX, SOCK_STREAM, 0) = 416
15:14:10.047226 connect(416, {sa_family=AF_UNIX, sun_path="/var/run/mdnsd"}, 110) = 0
... (416..420, six total) ...
15:14:10.064802 close(415)        = 0
15:14:10.065362 socket(AF_UNIX, SOCK_STREAM, 0) = 415
15:14:10.065737 connect(415, {sa_family=AF_UNIX, sun_path="/var/run/mdnsd"}, 110) = 0
... (repeats for 416..420) ...
```
Six connections opened and closed cleanly (415-420), immediately followed by six more to
the same six fd numbers/same socket path — and that second batch is never closed for the
remainder of every capture we took (15+ minutes, multiple independent captures).

Correlated syslog at the same timestamp:
```
[INFO]-MDNS----: DNSServiceBrowse reply: remove <instance>._trel._udp. inf 9, flags=1, error=0
[INFO]-MDNS----: Service <instance>._trel._udp is removed from netif 9.
... (repeated for the other 5 browsed interfaces) ...
[INFO]-MDNS----: DNSServiceBrowse reply: add <instance>._trel._udp. inf 9, flags=3, error=0
[INFO]-MDNS----: DNSServiceResolve <instance> _trel._udp. inf 9
... (repeated for the other 5 interfaces) ...
```
`<instance>` is this border router's own TREL advertisement — this is genuine
self-rediscovery, not a second BR. Six browsed interfaces, six leaked connections, every
occurrence.

## Analysis

`src/mdns/mdns_mdnssd.cpp`:

```cpp
void PublisherMDnsSd::ServiceSubscription::Resolve(uint32_t aInterfaceIndex, ...)
{
    mResolvingInstances.push_back(
        MakeUnique<ServiceInstanceResolution>(*this, aInstanceName, aType, aDomain, aInterfaceIndex));
    mResolvingInstances.back()->Resolve();
}
```
Called on every mDNS browse "add" event, once per matching interface, with **no check
for an existing/stale resolution of the same instance already in `mResolvingInstances`.**

```cpp
void PublisherMDnsSd::ServiceInstanceResolution::FinishResolution(void)
{
    ServiceSubscription   *subscription = mSubscription;
    std::string            serviceName  = mSubscription->mType;
    DiscoveredInstanceInfo instanceInfo = mInstanceInfo;

    // NOTE: The `ServiceSubscription` object may be freed in `OnServiceResolved`.
    subscription->mPublisher.OnServiceResolved(serviceName, instanceInfo);
}
```
Called once a resolution completes. **Never erases `this` from `mResolvingInstances`,
and never deallocates the `DNSServiceGetAddrInfo` connection** (the first-phase
`DNSServiceResolve` connection *is* correctly deallocated earlier, in
`HandleResolveResult()`, via `DeallocateServiceRef()` — only the second phase leaks).

Since `mResolvingInstances` is a `std::vector<std::unique_ptr<ServiceInstanceResolution>>`
and finished entries are never erased, every repeat "add" for an already-resolved
instance appends a new entry on top of the old one — a leaked `mdnsd` connection *and* a
leaked heap object, forever, once per still-browsed interface per occurrence.

We also noticed a second instance of the identical *pattern* in the same file:
`PublisherMDnsSd::SubscribeHost()` unconditionally `push_back`s a new `HostSubscription`
onto `mSubscribedHosts` with no check for an existing subscription of the same hostname,
whereas `UnsubscribeHost()` (which correctly finds-and-erases by hostname) already exists
and could be called first. This looked initially like it might leak the same way if
`SubscribeHost()` is ever called again for an already-subscribed host without an
intervening `UnsubscribeHost()` — we saw a small, one-time fd increase correlating with a
host address-change event shortly after applying the patch above. However, after 25+
hours of continued monitoring spanning further node reflashes, a border-router restart,
and another host address change, fd counts have stayed completely flat with no recurrence.
We're now fairly confident this code path, while still missing a defensive check that
`ServiceInstanceResolution`'s equivalent (`UnsubscribeHost`) already has, is not actually
being exercised as a live leak in normal operation — flagging it as a spotted-but-dormant
code smell worth a defensive fix, not a confirmed second bug.

## Patch

```diff
--- a/src/mdns/mdns_mdnssd.cpp
+++ b/src/mdns/mdns_mdnssd.cpp
@@ -1316,6 +1316,26 @@
     std::string            serviceName  = mSubscription->mType;
     DiscoveredInstanceInfo instanceInfo = mInstanceInfo;

+    // Move `this` out of `mResolvingInstances` before calling `OnServiceResolved` below
+    // (which may free `subscription`, per the existing NOTE) so that this resolution's
+    // DNSServiceGetAddrInfo ref gets deallocated via ~ServiceRef() once `self` goes out
+    // of scope, instead of leaking forever: without this, a new "add" browse event for
+    // the same instance (e.g. this BR periodically rediscovering its own _trel._udp
+    // record as its mDNS cache entry refreshes) pushes another ServiceInstanceResolution
+    // onto mResolvingInstances on top of this now-orphaned one -- one leaked fd per
+    // still-browsed interface, every such cycle.
+    auto &instances = subscription->mResolvingInstances;
+    auto  it        = std::find_if(instances.begin(), instances.end(),
+                                   [this](const std::unique_ptr<ServiceInstanceResolution> &aInstance) {
+                                       return aInstance.get() == this;
+                                   });
+    std::unique_ptr<ServiceInstanceResolution> self;
+    if (it != instances.end())
+    {
+        self = std::move(*it);
+        instances.erase(it);
+    }
+
     // NOTE: The `ServiceSubscription` object may be freed in `OnServiceResolved`.
     subscription->mPublisher.OnServiceResolved(serviceName, instanceInfo);
 }
```

The move-then-erase-before-callback pattern mirrors the one `Publisher::RemoveServiceRegistration()`
already uses in `mdns.cpp` for the identical self-destruction hazard (`this` potentially
being freed as a side effect of a callback it's still executing inside). When the local
`unique_ptr` goes out of scope at the end of the function, the existing (and already
correct) `~ServiceRef() -> Release() -> DeallocateServiceRef()` chain runs and closes the
connection — no new cleanup logic needed, it just needed to actually be reached.

## Result

Hardware-validated on the affected device: fd counts (`otbr-agent`, `mdnsd`) have held
flat for 25+ hours post-flash (as of this writing, still running), spanning roughly 215
TREL self-rediscovery cycles — the unpatched build would have leaked on the order of a
thousand fds in that window, well past the ~1024 default soft limit that was crashing the
process every day-to-week before the fix. No `otbr-agent` restart. That window also
included three node reflashes, a border-router restart, and a host address change — the
same class of activity that briefly touched fd counts by ±1 right after patching (see the
`SubscribeHost()` note above) — with no recurrence since. We separately confirmed the
`ServiceInstanceResolution` leak is gated by our `accept_ra` configuration by deliberately
reverting it: with TREL self-rediscovery suppressed, fd counts stayed flat for 75 minutes
(~10-11 cycles) on the *unpatched* build too — consistent with the patched build's
stability being the fix working, not coincidence.
