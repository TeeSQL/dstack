# F3 — Per-org VLAN tag in bridge mode: necessary, or already covered?

**Status:** Investigation report. Drives the F3 decision flagged in `00-decisions.md:225-229` and `00-decisions.md:323-331`.

**Question.** F2/F3 of the planning round committed to (a) per-org Linux uid for QEMU and (b) per-org VLAN tagging in bridge mode. The user explicitly asked, before committing the VLAN layer, whether equivalent network isolation is already provided by another dstack layer (gateway, guest-agent, vmm, or TDX itself). If equivalent, drop VLANs from v1.

**Answer in one line.** No equivalent isolation exists today. WireGuard between gateway and CVMs gives ingress isolation only; the host bridge (when `cvm.networking.mode = "bridge"`) is a single L2 broadcast domain with no per-VM/per-org filtering. **Per-org VLAN tagging is necessary for v1 if and only if v1 supports bridge mode for multi-tenant CVMs.** Default `mode = "user"` is unaffected (QEMU SLIRP gives every CVM its own NAT'd /24 with no L2 reachability between CVMs).

---

## 1. Today's network setup for a CVM in vmm

QEMU is launched with one `-netdev`/`-device` pair per CVM (single NIC, virtio-net-pci). The mode is selected by `cvm.networking.mode` and is one of three:

- **`user`** (SLIRP): `vmm/src/app/qemu.rs:505-522` builds `user,id=net0,net=<cidr>,dhcpstart=<ip>,restrict=<bool>[,hostfwd=...]`. SLIRP is fully userspace and per-VM — every CVM gets its own private 10.0.2.0/24 (configured in `vmm/vmm.toml:90-92`). No two CVMs are on the same L2 broadcast domain. Inter-CVM traffic on this NIC is impossible by construction.
- **`bridge`**: `vmm/src/app/qemu.rs:523-526` builds `bridge,id=net0,br=<bridge>` (no VLAN field). QEMU's setuid `qemu-bridge-helper` creates a TAP and adds it to the named bridge (`docs/bridge-networking.md:170-178`). All bridge-mode CVMs land on the **same** Linux bridge — `virbr0` from libvirt's default network or a single `dstack-br0` (`docs/bridge-networking.md:13-19, 60-74, 148-154`). They share one IPv4 subnet (libvirt's 192.168.122.0/24 or 10.0.100.0/24) and one broadcast domain. There is no kernel-level filter between TAPs on the same bridge by default.
- **`custom`**: `vmm/src/app/qemu.rs:527` passes `networking.netdev` verbatim. Operator-controlled escape hatch.

Networking config struct: `vmm/src/config.rs:386-445`. The flat shape carries bridge name, mac prefix, user-mode CIDR, and a custom netdev string — but **no `vlan` / `vlan_tag` / `vid` field anywhere**. A repo-wide `Grep` for `vlan|VLAN|802.1q|vlan_tag|vlan_id` against `vmm/` returns zero matches; the only hit anywhere in the tree is unrelated text in `kms/auth-eth/package-lock.json`.

MAC addresses are deterministic-per-VM (SHA256(vm_id) with optional 3-byte prefix, `vmm/src/app/qemu.rs:33-56, 500-503`). The host learns the guest IP when dnsmasq calls `scripts/dhcp-notify.sh`, which RPCs `Vmm.ReportDhcpLease`; `vmm/src/app.rs:421-447` writes the MAC→IP mapping. Optional userspace port-forwarding (`vmm/src/app.rs:454-543`, `port-forward/src/lib.rs`) listens on a host address and forwards to the guest IP — *traffic still traverses the host bridge in both directions*; the forwarder just lets QEMU run unprivileged.

vmm.toml's `[cvm.networking]` (`vmm/vmm.toml:86-96`) accepts only `mode`, `bridge`, `forward_service_enabled`, `mac_prefix`, plus user-mode parameters (`net`, `dhcp_start`, `restrict`) and a `netdev` string for custom mode. No VLAN field is parsed (`vmm/src/config.rs:398-427`).

vhost-net is intentionally disabled because TDX-encrypted guest memory cannot be DMA'd by the host kernel (`docs/bridge-networking.md:215-217`). This is unrelated to isolation but worth noting because it precludes some kernel-bypass packet-filter strategies.

---

## 2. dstack-gateway's role in network isolation

The gateway sets up its own WireGuard interface and treats every CVM as a peer (`gateway/src/config.rs:14-26`, `gateway/gateway.toml:34-43`). On `RegisterCvmRequest` (`gateway/src/main_service.rs:399-478, 1434-1480`), the gateway allocates the next free /32 in `client_ip_range` (default `10.0.0.0/25`, `gateway/gateway.toml:40`), regenerates `wg0.conf` from `gateway/templates/wg.conf`, and runs `wg syncconf` (`gateway/src/main_service.rs:1061-1082`). The template emits one `[Peer]` block per CVM with `AllowedIPs = peer.ip/32` (`gateway/templates/wg.conf:5-10`).

**`AllowedIPs = ip/32` is a cryptographic source-IP filter inside the kernel WireGuard implementation:** packets from a peer that claim a source IP outside that single /32 are dropped before they reach userspace. So at the gateway side, only the registered CVM's WG public key can produce traffic claiming that /32, and that public key was attested via RA-TLS during registration. Two CVMs from different orgs, on the same gateway, cannot impersonate each other on the WG interface.

The proxy data path picks a CVM by app_id, then `connect_multiple_hosts` calls plain `TcpStream::connect((ip, port))` (`gateway/src/proxy/tls_passthough.rs:144-178`, `gateway/src/proxy/tls_terminate.rs:286-300`). The TCP destination IP is the WG /32. Because the route to that /32 lives on the gateway's `wg0` interface, the kernel pushes the packet down WireGuard, which encrypts and tunnels it to the CVM's external WG endpoint over UDP.

**What this gives us:**

- Strong, per-CVM, attested ingress channel (gateway → CVM).
- Per-app port policy enforcement at the gateway (`gateway/src/proxy/port_policy.rs:60-90`), with `restrict_mode` fail-close.

**What this does NOT give us:**

- It does not touch the bridge. Two bridge-mode CVMs share `virbr0` regardless of whether either of them registered with the gateway. Inter-CVM packets going *through the bridge* never enter WireGuard.
- Many CVMs will not register with the gateway at all (`gateway_enabled` is opt-in per app, see `dstack-util/src/system_setup.rs:583-587` early-returns if `!app_compose.gateway_enabled()`). For those CVMs, `dstack-wg0` does not even exist.
- WireGuard is for ingress to the CVM. Egress from the CVM (DNS, container pulls, outbound API calls, etc.) goes via the bridge / SLIRP NIC, not WG.

Conclusion: the gateway provides per-CVM authenticated *ingress*, not bridge-level isolation between CVMs.

---

## 3. dstack-guest-agent's role

Guest-side networking is configured by `dstack-util` (system-setup binary inside the CVM) and `dstack-guest-agent`:

- **WireGuard client setup** runs at boot only when `gateway_enabled` (`dstack-util/src/system_setup.rs:583-587`). It calls the gateway's `RegisterCvm` RPC, writes `/etc/wireguard/dstack-wg0.conf` (`dstack-util/src/system_setup.rs:365, 626-666`), installs an iptables `DSTACK_WG` chain that ACCEPTs UDP from each gateway peer's endpoint IP and DROPs everything else into the WG listen port 9182 (`:669-692`), and then `wg-quick up dstack-wg0` (`:694-695`).
- **No iptables / nftables rules on the bridge interface (eth0/enp\*) are installed.** A repo-wide grep for `iptables|nftables` inside `guest-agent/` and `dstack-util/` confirms the only iptables work is the WG-port allowlist above (`dstack-util/src/system_setup.rs:669-692`). There is no cross-NIC firewalling, no DROP on the bridge, no input filter against bridge peers.
- **No network namespaces.** `Grep` for `netns|namespace` in `guest-agent/` and `dstack-util/` returns nothing relevant. The CVM is one big Linux box; Docker creates per-container netns (default), but the CVM itself is monolithic.
- **Containers default to docker-compose**, started by `basefiles/app-compose.sh:18-30` with `docker compose up -d`. Apps that bind to `0.0.0.0` are reachable on the bridge IP from any other CVM on the same bridge. Per-CVM port allowlists in the gateway (`PortPolicy.restrict_mode`) only filter the **gateway's** inbound proxy; they do not stop another CVM from connecting directly across the bridge.
- The `wg-checker.sh` watchdog (`basefiles/wg-checker.sh`) just keeps the WG handshake fresh, no isolation logic.

The guest agent does not provide bridge-level isolation, full stop.

---

## 4. TEE memory isolation — confirmation that it does NOT cover network

TDX protects **guest memory contents and CPU register state** from the host hypervisor and from physical attackers. It is a memory-encryption + register-isolation feature; it does not introspect or filter packet flow.

The QEMU machine config (`vmm/src/app/qemu.rs:781-866`) wires up `tdx-guest` with `confidential-guest-support=tdx` and an `mrconfigid`. None of the TDX flags affect the bridge, the TAP device, or the bridge's L2 broadcast behaviour. The packets leaving the guest's virtio-net frontend are decrypted into the host kernel's bridge code as ordinary Ethernet frames; once they hit the bridge, TDX's role is over.

There is no TDX-related feature in the codebase that constrains the network path. Confirmed.

---

## 5. Per-app firewall / iptables rules

The only per-CVM iptables work in the repo is host-side and serves a different purpose:

- `vmm/src/setup-user.sh:97-182` creates a `DSTACK_SANDBOX_<user>` chain that DROPs traffic from the CVM-runner Linux uid to **host loopback** (127.0.0.1). The hook is `iptables -I OUTPUT -o lo -d 127.0.0.1 -m owner --uid-owner $USERNAME -j $CHAIN_NAME` (`:180-181`). This stops the user-mode QEMU SLIRP "magic gateway" (10.0.2.2) from reaching host services bound to localhost. It is a sandbox against the *single* QEMU uid, not a per-CVM/per-org filter, and it does nothing on the bridge interface.
- `dstack-util/src/system_setup.rs:669-692` (described above) installs `DSTACK_WG` inside the guest. Only filters the WG UDP listen port.

There are no iptables/nftables rules generated by vmm or guest-agent that filter traffic between CVMs on the bridge. None.

---

## 6. VLAN support that already exists

**None.** The bridge netdev string built at `vmm/src/app/qemu.rs:525` is `bridge,id=net0,br={bridge}` — no VLAN field, no QEMU-side tag. The `Networking` config struct (`vmm/src/config.rs:398-427`) has no `vlan` / `vid` field; the per-VM override mechanism at `vmm/src/app/qemu.rs:483-499` only carries `mode` and `bridge`. The proto config (`networking_to_proto`, `vmm/src/app/qemu.rs:62-70`) has only `mode`. Repo-wide grep for `vlan|VLAN` against `/home/fbx/dstack` returns zero matches outside `kms/auth-eth/package-lock.json` (unrelated).

The only loophole is `NetworkingMode::Custom` (`vmm/src/app/qemu.rs:527`), which lets the operator hand-write a netdev string. An operator could pre-build a VLAN sub-interface on the host (e.g. `dstack-br0.100`) and set `mode = "custom"` with a netdev that targets it — but this is global per vmm config, not per-VM per-org, and it does not give the control plane a way to assign a tag at deploy time.

---

## Verdict

**Per-org VLAN tagging *is* necessary, conditionally:**

- **If v1 only ships `mode = "user"` (SLIRP)** for multi-tenant CVMs (the default in `vmm/vmm.toml:87`), VLAN tagging is **unnecessary**. SLIRP gives every CVM a private NAT'd /24 with no L2 reachability to any other CVM. No new isolation work is needed at the network layer beyond the existing WireGuard ingress and gateway port_policy.
- **If v1 ships bridge mode** for multi-tenant CVMs, VLAN tagging (or per-org bridges) **is** necessary. Today, `mode = "bridge"` puts every CVM, regardless of org, on a single L2 broadcast domain. There is no other layer in the stack that filters bridge-side inter-CVM traffic:
  - Gateway WireGuard isolates ingress, not bridge.
  - Guest-agent installs no bridge-side firewall.
  - TDX protects memory, not packets.
  - vmm's per-uid iptables protects host localhost, not bridge.

**Why operators will want bridge mode anyway.** `docs/bridge-networking.md:5-9` explicitly recommends bridge for high-concurrency workloads (passt is CPU-bound past ~25K connections), and 06-vmm-auth-today-and-gaps.md:308 already calls this out as a hard isolation boundary that must be enforced, not "a config knob the operator might forget." Anyone running a real multi-tenant cloud will eventually flip a CVM to bridge for performance reasons.

---

## Recommendation

**Ship v1 with these network-isolation properties, in this order:**

1. **Default `mode = "user"`** for multi-tenant CVMs in v1, unchanged from today (`vmm/vmm.toml:87`). This is already isolated by SLIRP. **No new work.**

2. **Per-org Linux uid for QEMU (F2): keep.** This is independently valuable and the existing `setup-user.sh` template (`vmm/src/setup-user.sh`) only needs extension from one shared uid to N per-org uids. Cited in `00-decisions.md:223-224` as cheap defense-in-depth on top of TEE — that argument stands regardless of the VLAN decision.

3. **Per-org bridge mode (F3):** the cleanest implementation is **per-org bridges**, not VLAN tags. Reasons:
   - Bridge isolation is enforced by Linux at the kernel-bridge layer with no operator configuration to get wrong (no VLAN-aware bridge flag, no `bridge vlan add`, no `vlan_filtering=1`, no per-port tag/pvid).
   - VLAN tagging requires an upstream switch (or kernel bond/VLAN sub-interface) that respects 802.1Q, plus VLAN-aware bridges (`bridge vlan add vid N dev X pvid untagged`) — three places to misconfigure for one isolation guarantee.
   - The v1 control plane already needs to allocate per-org Linux uids and per-org workdir trees; allocating a per-org bridge at the same point in the lifecycle is the same shape of work.
   - Naming becomes `dstack-org-<short-id>` per-bridge with a /24 subnet from a configurable supernet (e.g., `10.42.<org-octet>.0/24`), DHCP via per-bridge dnsmasq.
   - Cross-bridge routing is *operator policy*, not default behaviour. Default is no FORWARD between bridges, and the host can route org→internet via NAT just like today's single bridge.

4. **What to monitor / measure to revisit:**
   - Adoption rate of `mode = "bridge"` in v1. If it stays near zero (because SLIRP suffices for everyone's connection volumes), the bridge isolation work is purely defensive and can stay slim.
   - Conntrack table pressure on the host (`net.netfilter.nf_conntrack_max` is already bumped to 2M in `basefiles/sysctl.d/99-dstack.conf`). If we end up with N per-org bridges each running NAT, watch the conntrack table and `nf_conntrack_buckets`.
   - Bridge count vs. host hardware interface count. If a single host hosts thousands of orgs that all want bridge mode, per-org bridges become unwieldy and we revisit VLAN sub-interfaces or single-VLAN-aware-bridge with per-port tags as a v2 optimization.

**TL;DR for the decisions doc:** drop "VLAN tag" wording from `00-decisions.md:225-229`, replace with "per-org bridge in bridge mode (lazily created on first bridge-mode CVM in that org)." Keep F2 (per-org uid) verbatim.

---

## Concrete code touch-points if v1 ships per-org bridge isolation

Even with the recommendation above (per-org bridges instead of VLAN tags), the work all happens in roughly the same places as VLAN tagging would have:

- **Config schema** — add a `per_org_bridge_template` (e.g. `"dstack-org-{slug}"`) and a `bridge_subnet_supernet` (e.g. `10.42.0.0/16`) to `[cvm.networking]` at `vmm/src/config.rs:398-427`. The legacy single-bridge `bridge` field can stay for non-multi-tenant installs.
- **Bridge lifecycle** — new module (suggested `vmm/src/bridge_manager.rs`) called by the org-creation lifecycle in the new control plane to:
  - Allocate the next /24 from the supernet.
  - Run `ip link add dstack-org-<slug> type bridge` + `ip address add <subnet>.1/24 dev dstack-org-<slug>` + `ip link set dstack-org-<slug> up`.
  - Append `allow dstack-org-<slug>` to `/etc/qemu/bridge.conf` (compare `docs/bridge-networking.md:160-167`).
  - Spawn a per-bridge dnsmasq (or a single dnsmasq with `interface=dstack-org-*` directives) and ensure the lease-script still notifies `Vmm.ReportDhcpLease`.
  - Install nftables rules per `docs/bridge-networking.md:118-145` parameterised on bridge name + subnet, plus a default `FORWARD iifname dstack-org-A oifname dstack-org-B drop` between sibling bridges.
- **vmm netdev assembly** — change `vmm/src/app/qemu.rs:484-499` to resolve the bridge name from `(org_id, manifest.networking.bridge)` rather than the global `cfg.networking.bridge`. The assembled netdev at `vmm/src/app/qemu.rs:525` does not change shape (`bridge,id=net0,br={bridge}`); only the `{bridge}` value is now per-org.
- **DHCP lease lookup** — `vmm/src/app.rs:421-447` resolves MAC→VM by SHA256(vm_id) match. If the same MAC prefix is used across orgs, ensure the vm_id namespace already disambiguates (it does — the vm_id is unique). The lookup itself need not change. If we want extra paranoia, also key on the receiving bridge name (passed by the `dhcp-notify.sh` script as an extra argument).
- **Port-forward target IP** — the userspace forward (`port-forward/src/lib.rs:21-38`) listens on the host address and forwards to the guest IP. Per-org subnets just give different `target_ip` values; no code change.
- **Dropping the VLAN-tag plumbing** — there is none to drop, since none was ever added (Section 6).

If the team does later want VLAN tags instead of per-org bridges (e.g., to support 1000+ orgs on one host efficiently), the additional touch-points are:
- A `vlan_id: Option<u16>` on the per-org config row.
- VLAN-aware bridge setup (single `dstack-br0` with `bridge vlan_filtering=1`) at `bridge_manager.rs`.
- A `bridge vlan add vid <vid> dev <tap> pvid untagged` invocation per CVM TAP after QEMU starts the device. QEMU's bridge helper does not do this; it would have to be a post-start hook reading `/sys/class/net/<bridge>/brif/` to find the new TAP, similar to how libvirt's `<vlan>` element is wired up. This is more brittle than per-org bridges and is the reason the recommendation above prefers per-org bridges for v1.

---

## Source citations summary

- QEMU netdev assembly: `vmm/src/app/qemu.rs:483-530` (`bridge,id=net0,br={bridge}` at `:525`).
- Networking config: `vmm/src/config.rs:386-445` (no VLAN field).
- vmm.toml networking block: `vmm/vmm.toml:86-96`.
- Bridge networking docs: `docs/bridge-networking.md:5-218` (TAP via `qemu-bridge-helper`, single bridge, MAC stable, no inter-VM filter).
- DHCP lease handling: `vmm/src/app.rs:421-447`; userspace port-forward: `vmm/src/app.rs:454-543`, `port-forward/src/lib.rs:21-90`.
- Gateway WireGuard config: `gateway/src/config.rs:14-32`, `gateway/gateway.toml:34-43`.
- Gateway peer registration and `AllowedIPs = ip/32`: `gateway/src/main_service.rs:399-478`, `gateway/templates/wg.conf:5-10`.
- Gateway proxy data path: `gateway/src/proxy/tls_terminate.rs:286-300`, `gateway/src/proxy/tls_passthough.rs:144-178`.
- Gateway port_policy fail-close: `gateway/src/proxy/port_policy.rs:60-90`.
- Guest-side WireGuard setup and DSTACK_WG iptables chain: `dstack-util/src/system_setup.rs:583-696`.
- Guest agent has no bridge firewall, no netns: confirmed by `Grep iptables|netns` returning only the WG hits in `guest-agent/` and `dstack-util/`.
- TDX guest configured at `vmm/src/app/qemu.rs:781-866` (memory only, no networking effect).
- vmm setup-user.sh per-uid localhost guard: `vmm/src/setup-user.sh:97-182`.
- Decisions context: `docs/vmm-rewrite-plan/00-decisions.md:218-229, 323-331`.
- Cross-org isolation expectation: `docs/vmm-rewrite-plan/06-vmm-auth-today-and-gaps.md:302-314` (esp. `:308` "multi-tenant bridge mode needs a per-org bridge (or VLAN tag) — *enforced*, not a config knob the operator might forget").
