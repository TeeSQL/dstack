# Codex opinion: bridge-mode isolation for v1

## Recommendation

Ship v1 multi-tenant bridge mode with **per-org Linux bridges, lazily created on the first bridge-mode CVM in that org**. Do not ship shared `virbr0` as the default for independent tenant orgs. If per-org bridge orchestration misses v1, then bridge mode should be disabled for multi-tenant org workloads or exposed only as an explicitly unsafe platform-admin escape hatch.

The short version: the leakage is real, and per-org bridges are the simplest isolation primitive that matches the v1 scope. A shared bridge makes every bridge-mode CVM a peer on the same L2 segment, while the existing WireGuard/gateway path isolates ingress only. Per-org bridges add operational work, but it is ordinary host-network reconciliation work: allocate a subnet, create a bridge, bind DHCP, set NAT, and clean it up when unused. That is easier to reason about than retrofitting correct L2 flow filtering on one shared bridge, and much less brittle than VLAN tagging through QEMU bridge-helper-created TAPs.

## 1. Is cross-org bridge-side leakage real?

Yes. In this threat model it is an actual attack surface, not a theoretical purity issue.

The current bridge path resolves each VM's networking against the global config and then passes a plain `bridge,id=net0,br=<bridge>` netdev to QEMU (`vmm/src/app/qemu.rs:480-525`). The config shape has one bridge name and no org, VLAN, filtering, or namespace field (`vmm/src/config.rs:398-427`). The bridge networking doc says the per-VM override selects only the mode and the bridge interface comes from global config (`docs/bridge-networking.md:21-29`), with `virbr0` as the libvirt default (`docs/bridge-networking.md:32-35`) or one manual `dstack-br0` (`docs/bridge-networking.md:148-154`).

That means two bridge-mode CVMs from different orgs are on the same Ethernet broadcast domain. Tenant B can ARP-scan the subnet, connect directly to Tenant A's services bound to the bridge IP, observe broadcast/multicast service discovery, attempt ARP/NDP/DHCP disruption, and generally bypass the gateway port policy. This matters because apps commonly bind admin APIs, metrics endpoints, databases, or internal HTTP listeners to `0.0.0.0` inside the CVM and rely on the platform boundary to keep other tenants away.

The other layers do not close this gap:

- Gateway WireGuard peers are restricted to per-CVM `/32` `AllowedIPs` (`gateway/templates/wg.conf:5-10`), but bridge-side CVM-to-CVM packets do not go through that tunnel.
- Guest setup skips gateway work entirely when the app has gateway disabled (`dstack-util/src/system_setup.rs:583-587`).
- The guest-side `DSTACK_WG` chain only filters UDP packets to the WireGuard listen port by gateway endpoint IP, then drops the rest (`dstack-util/src/system_setup.rs:669-692`). It is not a firewall on the bridge NIC.
- The host-side sandbox script inserts an owner-matched localhost rule on `OUTPUT -o lo -d 127.0.0.1` (`vmm/src/setup-user.sh:179-181`), not a bridge inter-tenant filter.

TDX does not change this conclusion. It protects guest memory and CPU state, not packets after the virtio-net path hands Ethernet frames to the host bridge.

## 2. Host-side complexity of per-org bridges

Per-org bridge orchestration is real work, but it is workable for a single-host v1.

**Lazy creation.** Create a bridge only when the first bridge-mode CVM is admitted for an org. The QEMU argument does not need a new shape; only the resolved bridge name changes before `vmm/src/app/qemu.rs:523-525` builds the netdev. The privileged part should live in a small host-network manager/reconciler, not in the QEMU child.

**DHCP per bridge.** This is the touchiest part because today's bridge mode assumes an external DHCP server: dnsmasq runs a `dhcp-script`, which notifies VMM of MAC/IP leases (`docs/bridge-networking.md:87-113`), and VMM matches MAC to VM and persists the guest IP (`vmm/src/app.rs:421-446`). For v1, either one dnsmasq per bridge or one generated dnsmasq config with explicit `interface=` stanzas is acceptable. I would pick one dnsmasq per bridge initially for failure isolation and simpler config generation; move to a single managed process only if process count becomes a real problem. Include bridge/org context in the lease notification even if MAC remains globally derived from VM ID.

**IPAM.** A `/24` per org from a configured supernet is workable on one host. A `/16` only gives 256 org subnets, so do not hard-code that as the only option; make the supernet operator-configurable and reserve allocations in Postgres with a uniqueness constraint. A `/24` gives about 250 leases, which should exceed v1 per-org CVM quotas. If an operator wants thousands of bridge-mode orgs, they can choose a larger RFC1918 block or a smaller per-org prefix later, but `/24` is the right default for DHCP simplicity and human debugging.

**NAT and routing.** Existing manual bridge docs already require DHCP/DNS INPUT rules, FORWARD rules, and NAT masquerade for a standalone bridge (`docs/bridge-networking.md:118-146`). Per-org bridges multiply this by N. Use nftables sets/maps keyed by bridge name or subnet instead of emitting a long independent rule block per org. Default policy should allow org bridge to uplink/internet and established return traffic, allow DHCP/DNS to the host service, and drop forwarding between sibling org bridges.

**QEMU and dnsmasq interactions.** QEMU bridge-helper creates and attaches TAPs (`docs/bridge-networking.md:156-178`), so the bridge must exist and be allowed in `/etc/qemu/bridge.conf` before QEMU starts. Avoid designs that require discovering the just-created TAP and mutating it after QEMU launch; that is one reason per-org bridges are cleaner than VLAN tags.

**DNS.** Treat per-org bridge DNS as host-provided recursive DNS, not tenant service discovery. dnsmasq can hand out the bridge gateway as DNS and forward upstream, or hand out operator-configured resolvers. Do not publish cross-org names from dnsmasq. If per-CVM names are useful later, scope them to the org bridge.

**Garbage collection.** Org deletion is already blocked while CVMs exist (`docs/vmm-rewrite-plan/00-decisions.md:281-282`), which makes network reclamation tractable. Still, implement a reconciler: desired bridge subnets come from DB state; actual bridge links, dnsmasq processes, nftables sets, and qemu bridge-helper allowlist entries are repaired on startup. Delete only after zero bridge-mode CVMs, no TAPs enslaved to the bridge, and a short grace period.

## 3. Failure modes

The main failure modes are operational, and they are manageable if reconciliation is first-class:

- **Orphaned bridges.** Host crashes or control-plane bugs can leave `dstack-org-*` links behind. Startup reconciliation should classify them as desired, stale-empty, or stale-in-use. Never delete a bridge with enslaved TAPs unless the owning CVMs are known stopped.
- **DHCP collisions.** Avoid by reserving non-overlapping CIDRs in Postgres and generating DHCP ranges from that allocation. Do not let dnsmasq serve a wildcard interface set where two bridge configs can overlap.
- **Deleted org but bridge stays.** This is acceptable as a leak of host resources, not tenant data, if the bridge is empty and no NAT/DHCP path remains. Reclaim it in GC and expose it in diagnostics.
- **Bridge deleted while CVMs run.** This breaks guest networking; the current docs already warn that recreating a bridge detaches TAPs and requires VM restart (`docs/bridge-networking.md:193-198`). Guard deletion on zero active ports.
- **MTU mismatches.** Keep bridge/TAP MTU at the host default unless the operator configures a lower uplink MTU. Monitor DHCP success and guest connectivity; mismatched MTU will show up as path-MTU blackholes, not isolation failure.
- **NAT/conntrack pressure.** Per-org NAT still shares the host conntrack table. This is not fundamentally worse than one shared bridge with NAT, but high-connection tenants can exhaust shared conntrack. Monitor conntrack count, insert failures, and per-org flow estimates.
- **Rule drift.** nftables, dnsmasq, and `/etc/qemu/bridge.conf` can drift from DB state. Treat generated host networking as declarative state and repair it, rather than only mutating on create/delete events.
- **Lease notification spoofing/drift.** Today's `dhcp-notify.sh` posts MAC/IP to VMM over HTTP (`scripts/dhcp-notify.sh:24-33`) and `ReportDhcpLease` writes the matched IP (`vmm/src/main_service.rs:579-581`, `vmm/src/app.rs:421-446`). The rewrite should move this onto a private local channel and include bridge identity so a bad or stale DHCP event cannot cross org context.

## 4. Third options

I would not choose a third option for v1, but the alternatives are worth naming.

**Shared bridge plus nftables/ebtables filtering.** Possible, but less simple than it looks. You must filter L2 and L3 between TAP ports while still allowing DHCP, DNS, ARP, IPv6 neighbor discovery where needed, gateway/uplink traffic, and return traffic. You also need a reliable mapping from QEMU-created TAP port to org at runtime. A missed protocol path becomes a tenant escape; an overbroad drop breaks networking. This is harder to reason about than separate bridges.

**Linux bridge VLAN filtering.** This is the original VLAN-family idea. It can scale better than one bridge per org, but QEMU's bridge-helper path creates TAPs and attaches them to the bridge (`docs/bridge-networking.md:172-178`) without assigning per-port VLAN state. The control plane would need a post-start hook to find the TAP and run `bridge vlan add ... pvid untagged`. That creates ordering and stale-port failure modes. I would keep this as a v2 optimization if thousands of bridge-mode orgs become normal.

**Network namespaces.** A namespace per org with a bridge inside it is strong isolation, but it complicates QEMU launch, qemu-bridge-helper access, dnsmasq placement, host forwarding, observability, and debugging. It is not simpler than per-org bridges on the root namespace for a single-host v1.

**macvlan/ipvlan.** These are good for direct attachment to an uplink, but they usually trade away convenient host-to-guest communication, NAT, DHCP control, and local service hooks. They also push more assumptions onto the operator's L2 environment. Not a good default for developer-friendly v1.

**OVS.** Open vSwitch can model this cleanly with ports, tags, and flows, but it is a new dependency and operational surface. It is appropriate if dstack later needs richer virtual networking, not for the first single-host control plane.

**eBPF/XDP/TC filtering.** Powerful but the most bespoke option here. It increases implementation and debugging risk, and it still needs correct attachment to dynamically-created TAPs. Not a v1 isolation primitive.

**Disable bridge mode.** This is the only simpler security option. If bridge mode is not required for v1, defaulting all multi-tenant CVMs to `mode = "user"` preserves isolation because QEMU user networking is per VM (`vmm/src/app/qemu.rs:504-522`) and the default config is already user mode (`vmm/vmm.toml:86-96`). But if bridge mode is available to tenants, it must not use a shared bridge.

## 5. What to monitor and revisit

Monitor bridge-mode adoption rate, per-org bridge count, dnsmasq process count or reload failures, DHCP lease latency/failure rate, nftables rule/set size, conntrack count and insert failures, NAT throughput, MTU/path-MTU incidents, and any operator reports that per-org `/24` allocation is too coarse. Also track how many orgs have zero running bridge-mode CVMs but still hold a bridge allocation; that will tell you whether GC policy is too conservative.

Revisit the design if a single host commonly carries hundreds or thousands of active bridge-mode orgs. At that point, a single VLAN-aware bridge, OVS, or smaller-prefix IPAM may be worth the added machinery. For v1, per-org Linux bridges are the right balance: they remove the cross-org L2 attack surface with primitives Linux operators already understand, and the complexity is bounded by single-host scope.
