# Isolated ARM64 kernel-main evaluation

The release1.20 port of upstream [Newt450](https://github.com/fosrl/newt/pull/450)
was tested against released native-main on November15,2026 under Homelab
outcome164. This was a separate Community1.26/Gerbil1.6/Traefik3.7.16 deployment;
no production connector, Enterprise database or license was changed.

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

The [upstream limits](kernel-main.md) remain: Linux only, IPv4 inner addresses and non-default IPv4 remote routes, unscoped outer endpoints, exclusive interface ownership and no silent fallback.

## Measured result

| Backend | HTTP GET | HTTP POST |
| --- | ---: | ---: |
| Released1.20 native-main | 1.257Gb/s | 1.382Gb/s |
| Experimental kernel-main | 1.042Gb/s | 1.295Gb/s |

Each cell used four streams, five seconds warmup, sixty receiver-measured seconds,
verified complete64MiB bodies, symmetric tunnel MTU8940, private DRG/veth9000,
CUBIC, owned-veth GRO and verified encrypted-UDP software OFFLOAD. One cell per
backend/direction was accepted; earlier mismatched-MTU setup diagnostics were
excluded. These shared-host samples demonstrate no throughput improvement;
they do not establish a product ceiling or a causal regression. CPU steal varied.
The existing raw native/NetBird transport references include different processing
layers and are not interchangeable with full HTTP delivery.

The actual Linux WireGuard interface, private peer and routes were read back;
there was no TUN device or privileged runtime. Full HTTP hashes and UDP echoes
passed. Connector restart regenerated the key and preserved default and host
routes. A Community control-plane restart recovered transport and target health.
Backend stop/start produced healthy→unhealthy→healthy. An immediate HTTP request
failed after the health API became healthy; fresh proxy readiness followed by a
complete HTTP/UDP check passed. API health alone does not prove route convergence. Graceful connector stop removed
the owned kernel interface while a temporary namespace keeper preserved the
network namespace for direct inspection; the keeper was then released.

Newt cgroup usage fell from56.1→6.6 core-seconds on GET and75.4→19.1 on POST.
This is not a total CPU saving claim: kernel work is not fully charged to that
cgroup. Whole-host execution and softirq/steal were observed on both shared hosts.

Go tests, race tests, real-kernel namespace lifecycle and the existing build
matrix passed. Nine affected ARM64 suites passed in restricted containers.
Nix packaging failed: the inherited lock selects Go1.25.4 for a module requiring
Go1.26. No Nix consumer, Enterprise multi-node failover, live Olm/browser session,
media playback or production readiness is established. The default backend is
unchanged. The experiment does not support adopting this fork as a speed fix.

Homelab owns exact controller manifests, sanitized numerical evidence and
retirement evidence under
[fleet/pilots/pangolin-kernel-main](https://github.com/tacodong/homelab/tree/codex/pangolin-kernel-main/fleet/pilots/pangolin-kernel-main).
The binary was built from `e9f0fbc8e92e9a98116f67e721c7673ff163db68` with
Go1.26.10; SHA256 is
`c812dd88fc9c2ed80f70ab1103da3d1152f1e2a6d9840c1586ae241907213e88`.
