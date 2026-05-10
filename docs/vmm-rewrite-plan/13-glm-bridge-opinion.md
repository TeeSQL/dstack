# Third opinion: per-org Linux bridge vs shared virbr0 for v1

**Author:** GLM-5.1 outsider review, 2026-05-06.
**Input:** `00-decisions.md`, `12-f3-vlan-isolation.md`, source code as cited below.

---

## 1. Is cross-org bridge-side leakage a real attack surface?

**Yes, it is real — not theoretical.**

The F3 investigation (`12-f3-vlan-isolation.md:96-107`) already established this conclusively, and I agree with its analysis. To restate the concrete attack: two bridge-mode CVMs from different orgs share `virbr0` (or `dstack-br0`). Org A's CVM runs a container bound to `0.0.0.0:8080`. Org B's CVM can reach it at the bridge IP directly — no WireGuard involved, no gateway proxy, no port-policy check. This is ARP + Ethernet, not IP routing; the kernel bridge code floods unknown-unicast frames to every port.

Is this likely to be exploited? That depends on what tenants run. A misconfigured database, an unauthenticated metrics endpoint, a container orchestrator API — all common. The F3 doc correctly notes (`12-f3-vlan-isolation.md:106`) that operators running real multi-tenant workloads will eventually flip to bridge mode for the >25K-connection performance benefit, so you can't assume bridge mode stays unused.

The threat model in the question — "SaaS platform, orgs don't trust each other" — makes this a first-class security requirement, not a defense-in-depth nice-to-have. A single bridge with no filtering would be negligent for that threat model.

## 2. Host-side complexity of per-org bridge orchestration

The F3 investigation sketches the touch-points at `12-f3-vlan-isolation.md:134-148`. Let me assess the real complexity honestly.

### What you'd need to build

| Piece | Complexity | Notes |
|---|---|---|
| Bridge lifecycle (create/delete) | Low | `ip link add ... type bridge` + `ip link set up`. Lazy create on first bridge-mode CVM in an org. ~20 lines of shell or Rust via `nix`. |
| Per-bridge IP + /24 allocation | Low | `10.42.<N>.0/24` from a `10.42.0.0/16` supernet. Store `<N>` in the org row in Postgres. 256 orgs before you need to widen the supernet or re-think. For v1 single-host, 256 is generous. |
| DHCP per bridge (dnsmasq) | Medium | One dnsmasq instance per bridge is wasteful. Better: single dnsmasq with multiple `interface=` directives and per-interface `dhcp-range=` lines, or dnsmasq's `--bridge-interface` feature. But dnsmasq hot-reload requires a SIGHUP or restart, which briefly pauses DHCP across all bridges. Alternatively, spawn one dnsmasq per bridge (each on a distinct port 0, so no DNS conflict). Either way, this is the most fiddly piece. |
| NAT/masquerade per bridge | Low | One nftables masquerade rule per subnet: `ip saddr <subnet> oifname != "dstack-org-*" masquerade`. Can be templated, same shape as the existing rules in `docs/bridge-networking.md:138-141`. |
| Cross-bridge FORWARD deny | Low | One rule: `iifname "dstack-org-*" oifname "dstack-org-*" drop`. By default Linux doesn't forward between interfaces without explicit rules anyway; this is a belt-and-suspenders deny. |
| `/etc/qemu/bridge.conf` update | Low | Append `allow dstack-org-<slug>` on create. Remove on delete. `qemu-bridge-helper` reads this file on every invocation (no daemon to restart). |
| DNS resolution for guests | Low | dnsmasq already handles this. Per-bridge dnsmasq instances each serve their own subnet. |
| Garbage collection | Medium | When last bridge-mode CVM in an org is stopped, tear down the bridge, kill the dnsmasq, remove the nftables rules, remove from bridge.conf. Requires tracking "active bridge-mode CVM count per org." If the control plane crashes mid-teardown, you get orphaned bridges — needs reconcile-on-startup logic. |

### IPAM: is a /24 per org from a supernet workable?

Yes, for v1. A `10.42.0.0/16` gives 256 orgs each getting 254 usable host addresses. A single host running hundreds of bridge-mode orgs, each with dozens of CVMs, is well within this budget. If you hit 256 orgs, you widen to `10.42.0.0/12` (4096 orgs) — just a config change, no code change. The allocation is trivial: `org_id → u8` mapping stored in the `organizations` table.

The concern is not the IP space; it's the per-bridge dnsmasq instances. If you have 200 orgs, that's 200 dnsmasq processes. Each is lightweight (~1 MB RSS), but it's still a process-management surface. A single dnsmasq with `--dhcp-range=10.42.N.10,10.42.N.254,255.255.255.0,12h` per bridge interface is cleaner but requires careful config generation and atomic reload.

## 3. Failure modes

| Failure | Likelihood | Blast radius | Mitigation |
|---|---|---|---|
| Orphaned bridge after crash | Medium | Wasted /24, stale nftables rules | Reconcile loop on control-plane startup: list `dstack-org-*` interfaces, compare against orgs with active bridge-mode CVMs, tear down orphans. |
| DHCP collision (two orgs get same /24) | Low (if IPAM is in Postgres with a unique constraint) | IP conflicts, broken connectivity | UNIQUE constraint on the subnet allocation column. Allocate inside a transaction. |
| Bridge deleted but CVM still running | Low (only if GC races with a running CVM) | CVM loses bridge connectivity, TAP device orphaned | Reference-count: only delete bridge when `active_bridge_mode_cvm_count = 0`. Re-check count under a lock. |
| MTU mismatch | Low | Path MTU black holes | Set MTU on the bridge at creation time (default 1500, same as today). Document that operators should match their physical MTU. |
| Conntrack scaling | Low-medium | Table exhaustion, packet drops | 2M conntrack entries already configured (`basefiles/sysctl.d/99-dstack.conf:6`). Each org's NAT adds entries proportional to its egress traffic, not to the number of bridges. N bridges don't multiply conntrack entries — the entries are per-connection, not per-bridge. This is a non-issue for single-host scale. |
| dnsmasq process leak | Medium | Stale DHCP servers on deleted bridges | Track dnsmasq PIDs per bridge. Kill on teardown. Reconcile on startup. |

The failure modes are real but manageable. The most likely operational pain point is the dnsmasq lifecycle, not the bridge lifecycle. Bridges are idempotent kernel objects; dnsmasq is a userspace daemon with state.

## 4. Third options

### 4a. Shared bridge + nftables/ebtables per-TAP filtering

One shared bridge (`dstack-br0`), but install nftables `bridge`-family rules on each TAP device that DROP all traffic except to/from the host (gateway, DNS, DHCP) and to/from the gateway's WireGuard endpoint. This is essentially port isolation at the L2 level.

**Pros:** Single bridge, single subnet, single dnsmasq. No IPAM per org. No bridge lifecycle at all.
**Cons:**
- You'd need to identify the TAP device name after QEMU creates it. QEMU's bridge helper names TAP devices predictably (`tapN`), but the name is not guaranteed. You'd need to scrape `/sys/class/net/<bridge>/brif/` after QEMU start to find the new TAP, then install rules. Brittle.
- `nftables` bridge-family rules are per-port, not per-bridge. You'd need a rule per TAP. With 100 CVMs, that's 100 rules, each allowing traffic to the host IP and the gateway, denying everything else. Manageable but fiddly.
- Actually implementing this correctly is harder than it looks. You need to allow ARP (otherwise DHCP breaks), allow DHCP to the bridge IP, allow DNS to the bridge IP, and deny everything else. The rule set is: `ether daddr <bridge-mac> accept` (host-bound), `ether daddr ff:ff:ff:ff:ff:ff accept` (broadcast for DHCP), `drop`. This is ~3 rules per TAP, but the `ether daddr` match means you need to know the bridge MAC at rule-install time.
- Linux bridge `isolated` flag (`brmanage`) exists but is per-port and only prevents hairpin/reflection, not cross-port unicast. Not sufficient.

**Verdict:** Simpler in theory, fiddlier in practice than per-org bridges. The TAP-discovery step is the weak point — it's a race between QEMU's bridge helper creating the TAP and your code installing rules. If you miss the window, the CVM has unrestricted access until the rule lands.

### 4b. Network namespaces per org

Each org gets its own network namespace. The bridge and dnsmasq live inside the namespace. QEMU runs in the host namespace but its TAP is moved into the org namespace.

**Pros:** Strong isolation (namespaces are the Linux kernel's primary isolation primitive). Familiar pattern (used by Docker, Kubernetes, libvirt).
**Cons:**
- Moving a TAP into a namespace requires `CAP_NET_ADMIN` at the time of TAP creation. QEMU's bridge helper runs setuid but doesn't move interfaces into namespaces. You'd need a post-start hook.
- The gateway's WireGuard tunnel terminates in the host namespace. Routing from host-namespace `wg0` into an org-namespace bridge requires a veth pair per org. This doubles the per-org interface count (bridge + veth + veth-peer).
- DNS and DHCP become harder to manage across namespaces.
- Debugging is miserable: `ip netns exec dstack-org-foo tcpdump -i dstack-br0`. Operators will hate this.

**Verdict:** Overengineered for v1 single-host. Network namespaces are the right tool for multi-host overlay networks or for isolating the control plane itself, but they add complexity disproportionate to the gain over per-org bridges.

### 4c. macvlan / ipvlan

Put each CVM on its own macvlan slave under a shared parent interface. Each CVM gets its own MAC and IP on the parent's subnet.

**Pros:** L2 isolation by default (macvlan slaves in `bridge` mode can talk to each other but not to the parent; in `vepa` mode they can't talk to each other at all). No bridge needed.
**Cons:**
- macvlan slaves cannot communicate with the host (the parent interface). This means DHCP from the host (dnsmasq) doesn't work. You'd need an external DHCP server or static IPs.
- The gateway needs to reach the CVM to proxy traffic. With macvlan, the host can't reach its own macvlan slaves by default. You'd need a workaround (a separate macvlan on the host side, or routing tricks).
- QEMU doesn't natively support macvlan — you'd create the macvlan interface yourself and pass it as a TAP/file descriptor.
- ipvlan (L3 mode) solves the host-communication problem but breaks L2 (no ARP, no DHCP).

**Verdict:** Not suitable. The host-to-CVM communication requirement (gateway proxy, DHCP) is fundamentally at odds with macvlan's isolation model.

### 4d. OVS (Open vSwitch)

Replace the Linux bridge with OVS. Use OVS flows to enforce per-port isolation.

**Pros:** Production-grade flow filtering, VLAN support, monitoring.
**Cons:** New dependency. OVS is a significant daemon with its own kernel module. For a single-host setup, it's bringing a bazooka to a knife fight. The current codebase has zero OVS references, and adding it is a large packaging/operational burden.

**Verdict:** Overkill for v1 single-host. Worth revisiting for multi-host v2 if you need overlay tunnels.

### 4e. eBPF/XDP on the bridge

Attach a BPF program to each TAP's XDP hook that filters packets.

**Pros:** Fine-grained, programmable, high-performance.
**Cons:** Complex to develop and test. Requires a BPF compiler/toolchain. Debugging is hard. Same TAP-discovery race as option 4a. Overkill for v1.

**Verdict:** Not appropriate for v1.

### 4f. Single VLAN-aware bridge with per-port VLAN tags

One bridge (`dstack-br0`) with `vlan_filtering=1`. Each CVM's TAP gets a VLAN tag (vid) assigned per-org. Cross-VLAN traffic is dropped by the bridge.

**Pros:** Single bridge object. Single dnsmasq (if using a trunk + per-VLAN sub-interface for DHCP, or dnsmasq with 802.1Q support). Scales to 4094 orgs.
**Cons:**
- The F3 investigation (`12-f3-vlan-isolation.md:118-120`) already argues against this: three places to misconfigure (VLAN-aware bridge flag, `bridge vlan add vid N dev X pvid untagged`, per-TAP tag assignment). I agree.
- The `pvid untagged` dance is subtle. If you get it wrong, CVMs either can't communicate at all (tagged frames sent to an untagged port) or communicate freely (untagged frames bypass the VLAN filter).
- Same TAP-discovery race as 4a: you need to find the TAP after QEMU creates it and run `bridge vlan add` before traffic flows.
- Debugging VLAN misconfigurations is notoriously painful.

**Verdict:** The F3 doc's argument here is sound. VLAN tags are correct in principle but fragile in practice compared to per-org bridges.

## 5. Recommendation

**Ship per-org Linux bridges, lazily created, in v1. Accept the dnsmasq lifecycle complexity.**

Justification: Per-org bridges are the simplest mechanism that provides the isolation the threat model demands *by default, with no configuration surface for the operator to get wrong*. The bridge IS the isolation boundary — there is no filter to install, no VLAN tag to assign, no rule to misorder. Traffic between bridges is not forwarded unless the operator explicitly routes it. This matches the F3 investigation's recommendation (`12-f3-vlan-isolation.md:118-123`) and I concur.

The complexity is real but bounded. The bridge lifecycle (create/delete, IP assignment, nftables, bridge.conf) is ~200-300 lines of Rust. The dnsmasq lifecycle is the hardest part; I recommend a single dnsmasq process with a generated config file (one `interface=` / `dhcp-range=` block per org bridge), reloaded via SIGHUP on org create/delete, rather than N dnsmasq processes. This avoids process sprawl and makes the DHCP state inspectable in one place. The SIGHUP-induced DHCP pause (~100ms) is acceptable for bridge creation/deletion events that happen at org-provisioning cadence, not at CVM start/stop cadence.

The IPAM question — `10.42.<N>.0/24` from `10.42.0.0/16` — is trivially workable for single-host v1. If you exceed 256 orgs, widen to /12. The allocation is a single `SMALLINT` column with a UNIQUE constraint in the `organizations` table.

The alternative — shared bridge with per-TAP nftables filtering — is tempting in theory but has a structural weakness: the TAP-discovery race. You cannot install filtering rules on a TAP that doesn't exist yet, and QEMU's bridge helper creates it asynchronously. Per-org bridges avoid this race entirely because isolation is a property of the bridge topology, not of rules installed after the fact.

### What to monitor / measure to revisit later

1. **Bridge-mode adoption rate.** If <5% of v1 CVMs use bridge mode, the isolation work is purely defensive. Track `mode` in the `cvms` table and report in `/metrics`.
2. **Active org count with bridge-mode CVMs.** This drives dnsmasq config size and bridge interface count. Alert if >200.
3. **dnsmasq SIGHUP latency.** Measure time from SIGHUP to first DHCP response on the new bridge. If >1s, consider switching to per-bridge dnsmasq instances.
4. **Orphaned bridge count at startup.** The reconcile loop should log how many bridges it cleaned up. If consistently >0, investigate crash frequency.
5. **Per-org egress conntrack entries** (`conntrack -C`). The 2M global limit (`basefiles/sysctl.d/99-dstack.conf:6`) is shared across all orgs. Track per-org usage to detect a noisy-neighbor org exhausting the table.

### When to revisit

If v1 hits 500+ orgs on a single host (unlikely but possible for a popular SaaS platform), per-org bridges become unwieldy. At that point, migrate to a single VLAN-aware bridge with per-port tags (option 4f above) or to OVS. The per-org bridge abstraction is a clean internal interface; the migration would be transparent to CVMs and to the gateway because the QEMU netdev string (`bridge,id=net0,br=<name>`) only changes the `<name>` value, and the gateway routes to WG IPs, not bridge IPs.
