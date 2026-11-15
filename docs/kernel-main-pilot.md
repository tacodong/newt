# Kernel-main port for isolated ARM64 evaluation

Owner-requested prototype under Homelab outcome164. No production rollout or
default-backend change is authorized by this branch.

## Source and review

The source is [upstream PR450](https://github.com/fosrl/newt/pull/450), frozen at
`220f19e0192066d5348f16f3951ecb45432b7291`, merged onto the1.20 release code.
Its actual merge base is `f6a4be0137bd6d1237e7c1bec68bfe0a297326fb`;
the release has32 upstream commits absent from that feature branch. This port
retains1.20's SSH session logging, client behavior and DNS address-family/upstream
fallback handling, and adds cancellation through that shared resolver path.
Runtime module versions remain pinned to1.20; kernel-adapter dependencies are
resolved through Go's normal module/checksum mechanism. Later upstream CI updates
are retained, with no release-tag or mirror workflow dispatch.

What this change does: the opt-in kernel-main path creates an exclusively owned
Linux WireGuard interface through netlink/wgctrl. Newt continues to run the
control connection and application proxies; released userspace/native-TUN modes
remain available. Interface aliases/index checks, protected control/endpoint
addresses, exclusive route additions and partial-failure rollback bound ownership.

The caller review covers connect/sync/subnet updates, ping/reconnect, shutdown,
native proxy listeners, browser gateway setup and configuration precedence.
Directly importing the old branch would lose released SSH-log wiring and DNS
fallback behavior; those port conflicts are resolved here. No source-visible
blocker was identified for an isolated opt-in evaluation. This is not a claim
of production readiness or complete network-failure coverage.

## Validation boundary

The first build and nine affected package suites passed on Linux ARM64 in
network-disabled, read-only containers without host credentials or privileges:
`internal/kernelwg`, `newt`, `newtconfig`, `proxy`, `util`, `exitnode`, `nativessh`,
`browsergateway`, and `clients`. The optional privileged real-kernel lifecycle
case is separate; a browser-SSH shell assertion skips when the sandbox cannot
spawn its shell. Those skips are not successful compatibility results.

The intended live comparison uses separate synthetic Community1.26 state,
Gerbil1.6 and Newt1.20-based artifacts over the existing private SJC/TYO DRG pair.
It does not reuse or activate the licensed canary database. Verify actual kernel
interface/peer/route state, complete HTTP bodies in both payload directions,
UDP, restart/reconnect and ownership-preserving rollback. Keep shared-host CPU,
kernel-worker accounting and the actual private endpoint explicit. Single-node
Community acceptance does not prove Enterprise cluster reassignment/failover.

The [upstream limits](kernel-main.md) remain explicit: Linux only, IPv4 inner
addresses/non-default IPv4 remote routes, unscoped outer endpoints and no silent
fallback. No existing interface is adopted. This branch does not remove those
limits or assert that performance will match native/NetBird.

Verdict: Ship for isolated evaluation only.
Not checked yet: live ARM64 handshake/delivery and recovery; licensed cluster
failover and production acceptance are not established by the unit results.
