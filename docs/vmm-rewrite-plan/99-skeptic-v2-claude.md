# 99 — Skeptic pass v2 on the dstack-vmm rewrite plan (Claude Opus 4.7)

**Prepared:** 2026-05-06
**Author:** Claude Opus 4.7 (1M ctx) — independent of the synthesis author (Codex GPT-5.5).
**Scope:** Loyal opposition to the *revised* plan in `99-final-plan-v2.md` after the v1
skeptic pass and Codex bridge synthesis. Does not rehash the v1 skeptic items in
`99-skeptic.md` except where v2 made them worse.

Format per item:
- **Consensus** — one sentence.
- **Case against** — code/document-grounded.
- **Alternative.**
- **Cost of being wrong.**
- **Verdict** — change or hold.

---

## 1. Web UI deferred to v1.5

**Consensus.** v1 ships API + CLI only; web UI lives in v1.5 because the embedded
Vue VMM UI is single-tenant and rebuilding it is L/XL work
(`99-final-plan-v2.md:2310-2319`).

**Case against — fatal coupling between Privy embedded wallets, Safe signing,
and "no UI."**

This is the single biggest scoping error in the v2 plan, and it is not even a
prioritization debate; it is internally inconsistent.

The v2 admin add/remove flow returns this from `POST /sign`
(`99-final-plan-v2.md:1093-1105`):

```json
{
  "privy_signing_request": {
    "wallet_id": "privy-wallet-id",
    "chain_id": 8453,
    "to": "0xSafe",
    "data": "0x...",
    "typed_data": { "...": "SafeTx EIP-712 payload" }
  }
}
```

Privy's embedded-wallet signing API is a **browser-side** SDK call. The whole
point of "embedded wallet" is that the private key is custodied behind Privy's
iframe and the only legitimate way to invoke `wallet.signTypedData()` is from a
DOM context the user authenticated to via Privy's auth flow. You cannot script
this with curl. You cannot script it with the CLI. There is no Privy server-side
"sign on behalf of user" API for embedded wallets that doesn't either (a)
require per-user JWT plus a browser-context callback or (b) defeat the entire
custody model.

So a v1 "API + CLI only" cut means:

- `PATCH /v1/orgs/:slug/members/:user_id` to promote a Deploy to Admin returns
  a `safe_transaction.sign_url` (`99-final-plan-v2.md:887-902`) — which is a URL
  to a UI that doesn't exist.
- `POST /v1/orgs/:slug/safe/transactions/:id/sign` returns a
  `privy_signing_request` payload — which has no consumer.
- The org cannot add its second admin. The org cannot rotate admins. The org
  cannot whitelist a new compose hash, because that goes through the same Safe
  flow (`99-final-plan-v2.md:1318-1320`).

The plan acknowledges this in the Risks section:

> Open implementation question 3: Exact Privy API surface for embedded-wallet
> EIP-712 SafeTx signing and whether server-driven signing requires a webhook
> or browser callback (`99-final-plan-v2.md:2353-2354`).

The honest answer is "browser callback." Privy's documented EIP-712 signing
flow for embedded wallets is `useSignTypedData()` from `@privy-io/react-auth`.
That requires a React app. There is no "OpenAPI client signs Privy embedded
wallets" path.

The CLI cannot fill the gap either. v1's CLI is "minimal" and the listed
features are "login/token helpers, deploy/start/stop/logs, audit verify"
(`99-final-plan-v2.md:2306`). That doesn't include "render an EIP-712 dialog
that drives a Privy iframe."

Beyond Safe signing: the audience for this product (per `00-decisions.md`
TL;DR — "the operator's customers") includes billing roles and viewers who, by
construction, are non-engineers. Asking a billing user to read OpenAPI docs to
view their own audit trail and run cursor pagination by hand is silly.

**Alternative — a minimum-viable UI footprint that ships in v1:**

Keep the v1 CLI plan, but add a small SPA for the *handful* of flows that
require a browser context:

1. **Privy auth** (sign-in landing). Pure Privy SDK widget; ~50 lines.
2. **Wallet/signer management** (list wallets, set primary).
3. **Safe signing UI** — pending Safe transactions list, "Sign with Privy"
   button that drives the embedded-wallet API, status display. This is the
   load-bearing screen.
4. **Member management** (invite, role change) — these create Safe txs that
   need #3 anyway, so building them as one screen is cheap.
5. **Org switcher + create org**.

That's it. No CVM UI, no audit dashboard, no quota dashboards, no registry-cred
UI in v1. Those can come in v1.5 because their flows are scriptable from CLI.

Cost: probably +M (3–5 weeks) with one frontend-aware engineer. Compare
report 02's blocker list (`02-vmm-web-ui.md:13-56,698-900`) — a *full*
multi-tenant UI is L/XL. The minimum-viable list above is a fraction of that
because it shares ~no surface with the existing Vue UI and can be fresh.

**Cost of being wrong, in either direction.**

- Wrong toward consensus (no UI in v1): you ship an API-only v1 in which the
  Privy/Safe flow is unreachable, which means v1 cannot promote a second admin
  or whitelist a new compose hash. v1 is functionally a single-admin platform.
  Either you reverse course and add the UI mid-cycle, or v1 customers operate
  as single-admin orgs forever (unhappy product reality).
- Wrong toward alternative (slim UI in v1): you carry a small frontend bundle.
  Maintenance burden ~1 person-month/year. Marginal.

**Verdict. Change.** Keep CVM/audit/quota dashboards in v1.5, but pull
Privy auth + Safe-signing + member-management screens into v1. The
consensus is internally inconsistent: you cannot ship Privy embedded wallets
*and* ship "no UI" at the same time. Pick one.

If the team really wants no UI in v1, then drop Privy embedded wallets in v1
and require external-wallet-only auth (WalletConnect/MetaMask), which moves
the signing experience to wallet apps you don't ship. That is a coherent
position; "API-only with Privy embedded wallets" is not.

---

## 2. Bridge mode in v1 at all

**Consensus.** Per-org Linux bridges, dstack-net-helper, IPAM, per-bridge
dnsmasq, nftables sets, and a reconciler — Milestone 5, ~6–8 calendar weeks
of focused work plus contingencies (`99-final-plan-v2.md:2271-2284`).

**Case against — Codex's own fallback is the right v1 choice.**

The plan documents the fallback explicitly (`99-final-plan-v2.md:1979-1983`,
`00-decisions.md:283-286`):

> Fallback if per-org bridge orchestration overruns v1 budget: ship
> `mode = "user"` only for v1 (today's vmm.toml default per
> `vmm/vmm.toml:86-96`) and defer multi-tenant bridge mode entirely. Codex
> flagged this as the cleanest fallback.

What does the user persona actually need bridge mode for?

- **Web/SaaS workloads.** SLIRP works. Today's default is already SLIRP
  (`vmm/vmm.toml:86-96`).
- **High-concurrency workloads where SLIRP becomes CPU-bound past ~25K
  connections** (`docs/bridge-networking.md:5-9`, cited in F3
  `12-f3-vlan-isolation.md:106`). This is real but it's a minority workload.
- **LAN discovery / multicast / raw sockets.** Real but extremely niche; if
  you need them you usually need a dedicated host anyway.

Now look at what bridge mode in v1 actually pulls in:

| Workstream from M5 | Code surface |
|---|---|
| Per-org Linux uid + setup-user.sh extension | `vmm/src/setup-user.sh` |
| IPAM + `org_networks` allocation | new module |
| Bridge lifecycle | `ip link add`, `ip addr`, `bridge.conf` |
| dnsmasq supervisor | new module + privileged binary |
| nftables sets/maps | new module + privileged binary |
| Private DHCP lease socket | replace `dhcp-notify.sh` HTTP |
| Reconciliation loop (every 60s) | new module |
| Startup classifier (desired/stale-empty/stale-in-use) | new module |
| GC after tombstone | new module |
| `dstack-net-helper` privileged binary | new binary, see §4 below |
| Bridge metrics (12 distinct metrics) | `99-final-plan-v2.md:2049-2062` |

Plus the new failure modes catalogued by Codex (`13-codex-bridge-opinion.md:44-56`):
orphaned bridges, DHCP collisions, MTU mismatches, conntrack pressure, rule
drift, lease notification spoofing.

This is a substantial subsystem with security-critical correctness requirements
(cross-tenant L2 leakage if you get it wrong) for a feature that the explicit
fallback says you can drop without losing the user persona.

The v2 plan even hedges further inside itself:

> If bridge orchestration overruns v1, the explicit fallback is to ship only
> `mode = "user"` for tenants and hide bridge mode behind a platform-admin
> unsafe flag (`99-final-plan-v2.md:1981-1983`).

If that's a viable fallback, it should be the default for v1, not the panic
button.

**Alternative.** Ship v1 with `mode = "user"` only for tenant CVMs. Keep
`mode = "bridge"` as a platform-admin-only escape hatch with a single shared
bridge for trusted internal workloads (this is how today's VMM works), with
loud documentation that it is *not* multi-tenant safe. Cut M5 entirely. Defer
all bridge plumbing — IPAM, per-org bridges, dnsmasq, nftables, reconciler —
to v1.5. Save 6–8 calendar weeks plus the security-review tail.

**Cheap intermediate** (if §1 is rejected and you really want some bridge
in v1): platform-admin-issued "bridge mode pass" — a per-CVM flag that admin
explicitly sets, gated by the operator-side override. Single shared bridge,
no IPAM, no per-org dnsmasq, no nftables sets — just the existing single
bridge + an audit log entry that this CVM is on the shared bridge. Tenants
who need it sign a side agreement with the operator. ~1 week of work.

**Cost of being wrong, in either direction.**

- Wrong toward consensus (bridge in v1): you ship 6–8 weeks of network
  plumbing for a feature whose adoption rate is unknown. If real adoption
  is <5% of CVMs, the engineering effort is dead weight and you've added
  hundreds of dnsmasq processes and a privileged net-helper binary's
  attack surface for nothing. The Codex opinion specifically says
  "monitor bridge-mode adoption rate ... if it stays near zero, the
  bridge isolation work is purely defensive and can stay slim"
  (`12-f3-vlan-isolation.md:125-127`) — that's a tell that nobody on
  the planning side knows whether tenants will use this.
- Wrong toward alternative (no bridge in v1): customers with high-concurrency
  workloads get the existing SLIRP perf cliff (~25K conns). Some bounce
  to dedicated hosts or competitors. You build it in v1.5 with real
  customer data telling you which features matter (per-org bridge?
  VLAN-aware single bridge? OVS?).

**Verdict. Change.** Defer bridge mode to v1.5. The investigation reports
gave you the per-org-bridge design, but they did not give you a use-case
intensity measurement. Build the cheaper thing first; let real adoption
data drive the v1.5 design instead of guessing.

If the team genuinely cannot defer it for product reasons, ship the
"platform-admin-issued single-shared-bridge override" intermediate and skip
the per-org plumbing.

---

## 3. Stuck-Safe edge case

**Consensus.** Safe `removeOwner` runs first, DB removal second
(`00-decisions.md:104-108`, `99-final-plan-v2.md:1244-1259`). Last-admin
removal is blocked at the API layer. Recovery signer is opt-in, off by
default (`00-decisions.md:118-125`, `99-final-plan-v2.md:1331-1347`).

**Case against — bricked-org scenario.**

Consider this state: an org has one admin. Admin's Privy account is lost
(forgot the email used to sign in; lost OAuth provider; Privy account got
suspended; Privy's embedded-wallet keys are inaccessible because of a bug
in Privy or the user wiped their browser without exporting). The admin
can't sign Safe transactions. No second admin can be promoted because that
also requires the current admin to sign the `addOwner` Safe tx
(`99-final-plan-v2.md:1222-1237`). No deploys can update compose hashes
because those route through the same Safe.

This is not a hypothetical. Embedded-wallet recovery is exactly the failure
mode the v1 skeptic flagged in §3 (Privy lock-in,
`99-skeptic.md:236-313`). The v2 plan declined to address that risk.

If recovery signer is enabled, the operator can break-glass after the 24h
timelock. If recovery signer is disabled (default), the org is permanently
bricked: the Safe owner is a wallet nobody controls; the operator cannot
override the on-chain authority because they specifically chose to *not*
have that override.

The plan says (`99-final-plan-v2.md:2393-2396`):

> Privy lock-in/cost. The revised decisions accept Privy despite skeptic
> concerns. Mitigation: store normalized users and wallets locally, keep
> auth boundary behind a provider trait, support BYO external wallets from
> day one.

That mitigation is for *cost/lock-in*. It doesn't help the user with
a lost embedded-wallet account. Their `users.privy_user_id` row is fine;
their `user_wallets` row is fine; the Privy-custodied private key is
lost. The local DB cannot reconstruct it.

The Safe-first ordering compounds the problem. With DB-first:
"membership marked deleted; you can promote a new admin and clean up the
on-chain owner later" — a remediation path exists. With Safe-first:
the on-chain state must change before the off-chain one, and changing
on-chain state requires a signer that doesn't exist.

**Three alternatives, ordered by recommendation strength:**

**(a) Make recovery signer mandatory in v1.** The operational cost of an
optional 24h-timelock break-glass is small. The cost of a bricked org is
catastrophic (you lose the customer entirely; their CVMs run forever
unmanaged until the host shuts down). Default-on, opt-out only with a
loud install-time warning. Cost: same as today + one bool flip in the
defaults.

**(b) Relax Safe-first ordering for admin removal.** Use DB-first with
quarantine: mark the membership `pending_removal`, immediately revoke
the user's session and API tokens, delete the DB row after a grace
period. The on-chain `removeOwner` happens lazily and asynchronously;
if it fails, the on-chain owner becomes a stale signer that any other
admin can clean up later. Audit log records the divergence. Cost: a
nightly reconciler job that reports DB↔Safe drift.

**(c) Add a platform-admin "stuck Safe rescue" flow.** If recovery signer
is enabled, the operator can break glass after 24h. If recovery signer
is *disabled*, there should still be a documented rescue: platform-admin
acknowledges the org is stuck, generates a new Safe with new
admin set + recovery signer enabled, marks the old Safe address with
`abandoned_at`. Compose hashes whitelisted under the old Safe survive
because they're indexed by app_id, not Safe address. Audit row
`platform_admin.safe_rescue` with full justification.

**Cost of being wrong.**

- Wrong toward consensus (recovery signer optional, Safe-first ordering):
  one customer with a single-admin org loses their Privy account and
  their CVMs become unmanageable. They are an example to other potential
  customers who notice the structural risk in your governance model.
- Wrong toward alternative (a) — mandatory recovery signer: you ship with
  a 24h-timelock-able operator key in `secrets`. If KMK leaks AND
  the timelock period is bypassed somehow, attacker can hijack any org
  Safe. The mitigation is exactly what the plan already documents: the
  timelock is on-chain, so attacker also has to wait 24h while customers
  watch. Real but small.

**Verdict. Change.** Make recovery signer **mandatory in v1** (alternative a).
Add platform-admin rescue flow for legacy orgs created before the mandate
(alternative c). Keep Safe-first ordering for now, but document the
divergence-repair procedure. The "opt-in, off by default" framing of the
recovery signer is risk-shifting onto users who don't understand the
implication; mandatory + visible-during-install is the right default.

---

## 4. `dstack-net-helper` design choice

**Consensus.** A new privileged binary, `dstack-net-helper`, owns `ip link`,
`nft`, dnsmasq writes, and `/etc/qemu/bridge.conf` edits. Invocation model
is "sudoers fixed commands / systemd-run / long-running daemon" — left as
open implementation question 9 (`99-final-plan-v2.md:71`,
`99-final-plan-v2.md:2363-2364`).

**Case against — this binary was never reviewed; the simplest answer was
not considered.**

`dstack-net-helper` does not appear in `00-decisions.md`. It was introduced
by the synthesis pass to address the question "how does the control-plane
process do `ip link add` without running as root or holding ambient
`CAP_NET_ADMIN`?" The plan punts on the implementation choice but the
choice meaningfully affects security and operational surface.

CVE patterns by option:

**Sudoers fixed commands.** `sudo` is a notable CVE-rich attack surface.
CVE-2021-3156 (Baron Samedit, heap overflow in `sudoedit` argv parsing)
affected ~all sudo before 1.9.5p2 and was 10/10 severity. CVE-2023-22809
(sudoedit env-var-injection on edit lists) affected 1.8.0–1.9.12p1.
CVE-2023-28486/28487 (sudo log syslog escape sequence injection). The
attack pattern is consistent: the parser around argv/env/log strings is
where bugs live. *Anything* that turns user-controlled bytes into
fixed-argv sudo invocations is a sudo-CVE-amplifying surface. The
control plane converts org IDs → bridge names → sudo argv; the org ID
comes from the user's slug at org-create time. That's exactly the kind
of input boundary where a sudo escape would be material.

**systemd-run transient units.** Smaller CVE surface than sudo, but adds
DBus dependency and unit-name collisions, and `systemd-run` has its own
parser issues (CVE-2024-... unit-name escape). Race conditions between
concurrent `systemd-run --wait` invocations are real. PolicyKit/polkit
CVEs (CVE-2021-4034 Pwnkit) live in this neighborhood.

**Long-running root daemon with a Unix socket.** Standing root privilege.
Every byte that crosses the socket is parsed by code running as root.
Same surface as old `dhcp-notify.sh` HTTP path — exactly the surface
Codex flagged as spoofable (`13-codex-bridge-opinion.md:55`). You're
explicitly building the thing you said was wrong.

**The simplest answer was not on the table:** give the control-plane process
`CAP_NET_ADMIN` (and `CAP_NET_RAW`, `CAP_NET_BIND_SERVICE` if needed) via
systemd unit `AmbientCapabilities=` / `CapabilityBoundingSet=`. The
`dstack` uid runs as a regular user but the capabilities are inherited
into the process at exec time. No sudo, no daemon, no fixed-argv parser.
The control plane already has to be highly trusted because it directly
manages QEMU lifecycle, owns the KMK, and decrypts secrets — adding
`CAP_NET_ADMIN` to the same trust boundary is consistent with what's
already there.

The argument for net-helper is "least privilege" — but the control plane
is *already* the security boundary that decides what bridges exist.
Splitting the bytes-that-decide from the bytes-that-execute via a sudo
shim does not reduce the trusted code base; it adds a parser between two
parts of the same brain.

Today's VMM precedent: `vmm/src/setup-user.sh` is invoked with sudo
(presumably) and runs `iptables -I OUTPUT ...` for the per-uid sandbox
chain (`vmm/src/setup-user.sh:97-181`). Even that script is one shot at
install/uid-create time, not the steady-state mutation surface the
v2 plan envisions.

**Alternative — keep `CAP_NET_ADMIN` on the control plane process via
systemd capabilities.**

```ini
# /etc/systemd/system/dstack-control-plane.service
[Service]
User=dstack
Group=dstack
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_RAW
NoNewPrivileges=true
RestrictNamespaces=user mnt net pid uts cgroup ipc
RestrictRealtime=true
RestrictSUIDSGID=true
ProtectClock=true
ProtectHostname=true
ProtectKernelLogs=true
ProtectKernelModules=true
ProtectKernelTunables=true
ProtectControlGroups=false
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
```

`/etc/qemu/bridge.conf` is the only file edit not covered by capabilities;
that is solved by an open-handle held at startup or a small per-mutation
write that doesn't need root because the file's parent dir is owned by
`dstack`.

This is what every distro QEMU/networking daemon does (firewalld, NM,
etc.). It's the boring, well-understood answer.

**Cost of being wrong.**

- Wrong toward consensus (`dstack-net-helper`): you carry a privileged-binary
  attack surface, you decide between three suboptimal invocation models,
  and you spend implementation time on the parser between the two halves
  of the same trust domain. If a sudo CVE drops, your install is exposed.
- Wrong toward alternative (`CAP_NET_ADMIN` on the control plane): if the
  control plane is RCE'd via the API surface, attacker has `CAP_NET_ADMIN`
  and can fully reconfigure host networking. They also have the KMK, which
  decrypts every org's registry credentials and the recovery signer key.
  At that point the network capability is the second worst thing they
  have. If you actually care about reducing blast radius, the answer is
  to move secrets handling into a separate process — *not* network ops.

**Verdict. Change.** Drop `dstack-net-helper`. Run the control plane with
`CAP_NET_ADMIN`+`CAP_NET_RAW` via systemd `AmbientCapabilities`. Tight
unit-file sandboxing. If post-v1 you want to split secrets out of the
control plane (the actual blast-radius win), do that — but invented
sudo shims are a CVE-magnet for very little real benefit.

The "no ambient `CAP_NET_ADMIN` for the control plane" framing in
`99-final-plan-v2.md:71` is privilege-separation cosplay, not real
privilege separation, when the same process holds the KMK.

---

## 5. dnsmasq-per-bridge

**Consensus.** One dnsmasq process per org bridge, for failure isolation
and config simplicity. "Future single-process config generator" deferred
(`00-decisions.md:262-264`, `99-final-plan-v2.md:1853-1889`,
`13-codex-bridge-opinion.md:32`).

**Case against.**

At hundreds of orgs with bridge mode, hundreds of dnsmasq processes are
running. Each has a small RSS (~1–2 MiB) and a config file watched by
the reconciler. Per-process overhead at 500 orgs is ~1 GiB RAM and
~500 inodes for config files, ~500 PID files, ~500 lease files
(`99-final-plan-v2.md:122-126`).

That's not a memory crisis, but here are the actual problems:

**(a) Migration tax.** The plan acknowledges (`99-final-plan-v2.md:2360-2362`):

> dnsmasq process model at scale: one process per bridge for v1, but
> define the threshold where a single managed process becomes necessary.

Migrating from N processes to 1 process *later* requires:

- DB schema change: drop `org_networks.dnsmasq_pid` and replace with
  a host-singleton field.
- Reconciler rewrite: the per-bridge supervisor logic becomes a
  per-host `interface=` config builder.
- Lease socket protocol: today's plan sends lease events with bridge
  identity from the per-bridge dhcp-script wrapper. In single-process
  mode, dnsmasq fires the script with the same identity but only one
  process is spawning the script; nothing fundamentally changes, but
  the "lease event came from process X for bridge Y" trust path
  changes.
- All the per-process metrics (`dstack_dnsmasq_up`,
  `dstack_dnsmasq_restarts_total`) lose their meaning.

So you're paying the migration cost in v1.x and v2 anyway, when the
single-process design is *already* the documented endgame.

**(b) Config-reload semantics.** A single dnsmasq with `interface=`
stanzas reloads on `SIGHUP`. The single-process worry is "if I reload
for org B's new bridge, does the SIGHUP affect org A's lease state?"

Answer: dnsmasq's SIGHUP behavior is documented:

> Upon receipt of SIGHUP dnsmasq will re-read /etc/hosts, /etc/ethers
> and any file given by `--dhcp-hostsfile`, `--dhcp-optsfile`, or
> `--addn-hosts`. Re-read the leases file to honour explicitly added
> dynamic leases. Re-read the configuration file (if `-C` not used).
> Active leases are NOT affected. (`man dnsmasq`)

So org A's leases survive the org-B-bridge add. The "failure isolation"
argument loses force: dnsmasq process death is rare, and SIGHUP-on-config-
change does not disturb existing leases.

**(c) The actual failure mode is the opposite.** With one dnsmasq per
bridge, a config error in org B's dnsmasq config crashes only org B's
dnsmasq. With a single dnsmasq, a config error breaks DHCP for every
bridge. This is a genuine failure-isolation argument for per-bridge.

But mitigation is cheap: validate every config with `dnsmasq --test --conf-file=...`
before reload (the v2 plan already does this,
`99-final-plan-v2.md:1887-1888`). If validation passes, SIGHUP. If it
fails, don't reload, mark `org_networks.state='error'`, alert.

**Alternative — single dnsmasq with `interface=` stanzas from day one.**

```ini
# /etc/dstack/dnsmasq/main.conf
log-facility=/var/log/dstack/dnsmasq.log
domain-needed
bogus-priv
no-resolv
bind-interfaces

# generated block — included once per active org bridge
{% for bridge in active_bridges %}
interface={{ bridge.name }}
dhcp-range={{ bridge.name }},{{ bridge.dhcp_start }},{{ bridge.dhcp_end }},255.255.255.0,12h
dhcp-option=tag:{{ bridge.name }},option:router,{{ bridge.gateway }}
dhcp-option=tag:{{ bridge.name }},option:dns-server,{{ bridge.gateway }}
{% endfor %}
dhcp-script=/usr/lib/dstack/dhcp-lease-notify
```

Reload algorithm: regenerate file → `dnsmasq --test --conf-file=…` →
on success, SIGHUP single process; on failure, leave existing config in
place and surface the error.

Cost: about the same v1 implementation effort as per-bridge, slightly
*less* operational surface (one process to monitor, one log file, one
metric), and no migration in v1.x.

**Cost of being wrong.**

- Wrong toward consensus (per-bridge): hundreds of processes, ~1 GiB
  RAM at scale, migration tax later when you implement the single-process
  endgame the doc already foreshadows. No real-world failure-isolation
  benefit because SIGHUP semantics don't disturb live leases.
- Wrong toward alternative (single process): a config-validation bypass
  bug in the reconciler crashes DHCP for every bridge. Mitigation is
  the validate-before-reload pattern, which is straightforward.

**Verdict. Change.** Single-dnsmasq-with-interface-stanzas from day one.
The v2 plan already telegraphs this as the endgame; building it later
is more expensive than building it now.

---

## 6. Postgres-in-Docker vs native install

**Consensus.** Postgres 16 in a sibling Docker container, peer auth over a
bind-mounted Unix socket (`00-decisions.md:127-135`,
`99-final-plan-v2.md:2113-2135`). The v1 skeptic argued for SQLite; that
push-back was declined. The Docker-vs-native choice was never independently
scrutinized.

**Case against.**

The v2 plan documents the install compose snippet (`99-final-plan-v2.md:2113-2135`):

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: dstack
      POSTGRES_USER: dstack
      POSTGRES_HOST_AUTH_METHOD: trust
    command:
      - postgres
      - -c
      - listen_addresses=
      - -c
      - unix_socket_directories=/run/postgresql
    volumes:
      - /var/lib/dstack/postgres:/var/lib/postgresql/data
      - /run/dstack/postgres:/run/postgresql
```

Then immediately admits this won't actually work as written:

> The actual production compose should use peer auth over the Unix socket
> rather than TCP trust. The above is illustrative; the build task must
> produce a tested Postgres container config where only the host `dstack`
> uid can access the socket. (`99-final-plan-v2.md:2133-2135`)

Open question 6 (`99-final-plan-v2.md:2357-2359`):

> Postgres peer auth from Docker: exact socket ownership and auth config
> that works repeatably across Ubuntu/Debian host installs.

This is non-trivial because:

1. Inside the container, Postgres runs as the `postgres` user (UID 999 in
   the official image). Peer auth checks the *connecting* uid against
   `pg_ident.conf`. The connecting uid is the host `dstack` uid (let's
   say 982). The container must map host 982 ↔ container "dstack" user,
   and `pg_ident.conf` must accept it. This works but requires either
   userns-remap configuration or a `pg_ident.conf` that accepts the
   numeric uid-as-string.
2. The bind-mount `/run/dstack/postgres:/run/postgresql` only works if
   the host directory's uid/gid match what the container's `postgres`
   user can write to, *and* the host's `dstack` uid can read from. SELinux
   contexts (Fedora/RHEL hosts) fight this without `:Z` flags.
3. Docker daemon dependency: install order is "install Docker Engine →
   pull `postgres:16` → start container → start control plane." Three
   moving parts vs one. Docker daemon outage = control plane outage.
4. Backup story: `pg_dump` runs from inside the container or via host
   `pg_dump` against the socket. Either works but requires explanation.
5. Upgrade story: bumping to Postgres 17 means a new container image, a
   `pg_upgrade` ceremony, and ordering across the control-plane unit and
   Docker.

Compare native install via `apt install postgresql-16` (or dnf/whatever):

- Postgres runs as `postgres:postgres` system uid by package convention.
- Socket lives at `/run/postgresql/.s.PGSQL.5432` by default; peer auth
  via `pg_hba.conf`'s `local all dstack peer` line works as
  documented; `pg_ident.conf` may need a one-liner mapping if the
  control plane uid is named `dstack` differently from `postgres`.
- `systemctl enable postgresql` → done. No Docker. No userns. No SELinux
  bind-mount fights.
- Backups: `pg_dumpall -U postgres` (sudo to postgres) or via the socket.
- Upgrades: `pg_upgradecluster` is a one-liner on Debian/Ubuntu; on RHEL
  the documented `pg_upgrade` with the official packages.
- HA when v2 wants it: pgbouncer + repmgr or whatever, all native.

There is no v1 benefit to containerization. v2's claimed HA path
("two stateless API replicas pointing at the same Postgres,"
`00-decisions.md:51-53`) doesn't change with the choice — both replicas
still connect over a socket or TCP, native or containerized.

The only case where containerization might be defended is "we want the
Postgres version to be pinned regardless of the host distro." Native
install can also pin a version (PGDG repos are well-maintained on
Debian/Ubuntu/RHEL). The difference is small and the operational
familiarity with `systemctl` is huge.

**Alternative — native install via PGDG/distro packages.**

Install steps in the installer (`dstack-control-plane init`,
`99-final-plan-v2.md:2138-2160`):

1. Detect host distro.
2. Add PGDG repo.
3. `apt install postgresql-16` (or equivalent).
4. Configure `pg_hba.conf`: one line `local dstack dstack peer`.
5. Configure `pg_ident.conf` if needed.
6. `systemctl enable --now postgresql`.
7. `createuser dstack && createdb -O dstack dstack`.
8. Run migrations.

Cost: distro detection logic in the installer. Marginal. Saves the
"Postgres in Docker actually works on these distros" testing matrix.

**Cost of being wrong.**

- Wrong toward consensus (Postgres in Docker): you spend installer time
  on Docker bind-mount + userns wrangling, you carry a Docker daemon
  dependency, your "single binary, one host" story has a Docker asterisk,
  operators who have a strict no-Docker policy can't use you.
- Wrong toward alternative (native install): you carry per-distro
  installer logic. PGDG covers Debian/Ubuntu/RHEL/CentOS Stream, which
  covers ~all real production Linux. Operators on niche distros do
  manual setup.

**Verdict. Change.** Native install is operationally simpler, has no v1
benefit penalty, and removes an open question. The decision was
"Postgres" (locked, `99-skeptic.md:810-925` declined); the *delivery
mechanism* was never debated. Pick the distro-package answer.

---

## 7. The 16–23 month / 9–12 month estimate; identifying a 6-month v0.5

**Consensus.** 71–98 engineer-weeks for one engineer (16–23 months);
9–12 months for two engineers (`99-final-plan-v2.md:2321-2342`).

**Case against — that's a lot for a v1, and the cuts are obvious.**

The v2 plan's milestone budget breakdown:

| Milestone | Size estimate (this plan) |
|---|---|
| M0 Foundation | ~M+M+S+M+M ≈ 14–20 weeks |
| M1 Auth/Orgs/RBAC | ~M+S+M+M+M+M ≈ 18–24 weeks |
| M2 Audit/Quotas/Obs | ~M×4+S×2 ≈ 14–20 weeks |
| M3 Safe/On-chain | ~M+L+L+M+M+L ≈ 26–35 weeks |
| M4 Host worker | ~XL+M+L+M+L+M ≈ 31–43 weeks |
| M5 Network isolation | ~M+M+M+M+L+M+L ≈ 24–34 weeks |
| M6 Registry creds | ~S+M+M+M+M+S ≈ 16–22 weeks |
| M7 Install/CLI/E2E/docs | ~L+M+XL+M ≈ 23–32 weeks |

Total summed individual: ~166–230 weeks for one engineer doing all of
it strictly serial. The plan compresses this to 71–98 weeks via
parallelism / shared components / gut estimate. Even taking the
optimistic number, 16–23 months is a long v1.

**A 6-month v0.5 cut.** Pick a target of 25 engineer-weeks (one
engineer for ~6 calendar months including leave and overhead) or 50
engineer-weeks for two engineers in parallel. Defer:

| Item | Saved | Justification |
|---|---:|---|
| Bridge mode (M5 entirely) | 8–12 wk | §2 above; minority workload, fallback documented |
| Recovery signer timelock helper contract | 4–6 wk | Make recovery signer mandatory but use a simple Safe-module pattern; defer the full audited timelock contract to v1.5 |
| ECR + GAR cloud token adapters | 4–6 wk | Ship Docker Hub + GHCR + generic basic in v0.5; cloud-registry IAM flows are non-trivial each |
| Audit hash chain → simple append-only | 1–2 wk | Re-add hash chain in v1; the threat model in v0.5 doesn't include "operator rewriting their own DB" |
| Per-org Linux uid (M5 partial) | 3–5 wk | Keep today's single sandbox uid for v0.5 (TEE is the hard boundary); add per-org uid in v1.0 |
| OpenTelemetry tracing | 1–2 wk | Enable Prometheus only in v0.5; OTLP in v1.0 |
| OpenAPI generation | 1–2 wk | Hand-write the API docs initially; OpenAPI in v1.0 |
| `Dstack-Api-Version` header versioning | 1 wk | Stable `/v1` paths only; revisit when first breaking change is contemplated |
| Resource usage 30-second sampler | 2–3 wk | Keep last-60s in-memory; persistent rollups in v1.0 |
| Idempotency keys | 1–2 wk | Document "client must dedupe" for v0.5; build the table in v1.0 |
| Privy embedded wallets | 3–5 wk | v0.5 = email + WalletConnect (BYO external wallet). Embedded wallets in v1.0. (See §8 below for why this *also* mitigates the Privy single-vendor risk.) |
| Cursor pagination | 1 wk | Offset pagination is fine for the v0.5 row counts |
| Quota soft-warning notifications | 1 wk | Hard caps only in v0.5 |

That's roughly 30–50 weeks of deferral. v0.5 lines up at ~25–35
engineer-weeks single engineer or ~12–17 weeks two-engineer-parallel.

**v0.5 is still genuinely multi-tenant** because what's left is:
multi-host-ready Postgres schema, Privy auth (sans embedded wallets) /
Safe / RBAC / API tokens / quotas / audit (append-only) / CVM lifecycle
(SLIRP only) / external Docker Hub + GHCR registry creds / install +
slim UI for Privy auth + Safe signing.

You ship the *governance & multi-tenancy* primitives (which is what
distinguishes the new product from the old VMM) and defer the *features
that incrementally improve a working system*.

**Cost of being wrong.**

- Wrong toward consensus (16–23 months): the long timeline lets the
  market or the threat model drift; competitors ship; the team burns
  out; the eventual v1 is a bigger forklift than necessary because
  features built in month 5 must match features built in month 17.
- Wrong toward alternative (6-month v0.5): you ship a smaller product;
  some customers wait for v1.0 because they need ECR/GAR or bridge
  mode. You ship faster, get real adoption telemetry, and the v1.0
  features are built with usage data.

**Verdict. Change.** Adopt a v0.5 cut along these lines. The current
plan reads as "ship everything we want eventually, sequenced." A v1
cut should ship "the smallest product that is genuinely the new
product." Bridge mode, embedded wallets, ECR/GAR, OTLP, per-org uid,
hash-chained audit are all real features but none of them are what
makes this product *different from the old VMM*. The differentiators
are: org RBAC, on-chain Safe per org, external pull credentials,
Postgres state, audit trail. Ship those first.

---

## 8. Privy single-vendor critical-path risk

**Consensus.** Privy is the identity provider; Safe sign-flow drives Privy
embedded-wallet signing (`00-decisions.md:55-61`,
`99-final-plan-v2.md:1093-1105`). v1 skeptic flagged Privy lock-in;
declined (`99-skeptic.md:236-313`).

**Case against — v2 made it worse, not better.**

The v1 skeptic was about *cost* and *user portability*. v2 elevated Privy
to load-bearing for **on-chain governance**:

- Every membership change goes through `addOwner`/`removeOwner` Safe txs
  signed by Privy embedded wallet (`99-final-plan-v2.md:1222-1259`).
- Every compose-hash whitelist change goes through `app_add_compose_hash`
  Safe txs signed by Privy embedded wallet
  (`99-final-plan-v2.md:1262-1322`).
- Every device add/remove on `DstackApp` goes the same way.

So during a 1-hour Privy outage:
- No org can add or remove an admin.
- No org can deploy a new compose (because deploy creates a new compose
  hash that needs whitelisting).
- No org can add a new device.

The "store user records locally + provider trait" mitigation
(`99-final-plan-v2.md:2390-2392`) helps with **switching providers later**.
It does **not** help with an active outage, because the user's embedded
wallet private key is custodied by Privy. The local DB has the
`privy_user_id` and the wallet address; it does not have the key.

Privy outage history (publicly reported): rare but real. Auth0 has had
outages. WorkOS has had outages. Every BaaS auth/wallet vendor has had
outages. Pretending Privy will never have one is the planning version of
"this won't happen to us."

**Three mitigations, ordered by recommendation strength:**

**(a) Always-allow external-wallet bypass for Safe signing.** The signer
the user picks for Safe txs can be either the Privy embedded wallet *or*
a BYO external wallet (MetaMask, WalletConnect, Rabby). The plan already
supports BYO external wallets *as a primary signer*
(`00-decisions.md:60-63`). Make sure the Safe-signing screen offers both
paths every time, even if the user's *primary* is set to Privy embedded.

This means a user who can't reach Privy embedded wallets in the moment
can connect MetaMask and sign the same SafeTx EIP-712 payload. The
control plane only needs to verify the signature came from a known
admin's wallet address, which it already does
(`99-final-plan-v2.md:521-530`, `safe_signatures` table).

**(b) "Paste the SafeTx hash" recovery path.** A user copies the hex
SafeTx hash from the control-plane UI, signs it on any device with any
wallet (e.g., a hardware wallet via Frame or Rabby Desktop), pastes the
signature back. Control plane validates and submits. This is exactly
the Safe Transaction Service workflow from non-Privy clients. It costs
~1 week of work and removes Privy from the critical signing path
entirely — Privy becomes auth-only, not signing.

**(c) Operator-side break-glass Safe-tx bundling.** Platform admin can,
under audit, queue a SafeTx that's signed by the recovery signer
during a documented Privy-outage emergency. This stacks with the §3
recovery-signer-mandatory recommendation.

**Cost of being wrong.**

- Wrong toward consensus (Privy on critical path): a 1-hour Privy outage
  freezes governance for every org on every dstack install globally
  *that day*. Customers who were trying to deploy notice. A 24-hour
  Privy outage (rare but possible) cascades into compose-hash-whitelist
  staleness, which means CVMs that try to update compose during that
  window can't.
- Wrong toward alternative (a/b): you ship two signing UIs and a small
  amount of "paste the hash" plumbing. Real but small.

**Verdict. Change.** Adopt (a) immediately — Safe-signing UI offers
both Privy embedded *and* BYO-external wallet for every signing
operation. Adopt (b) as a fallback in v1.x. Privy stays as the auth
layer; it does not stay on the critical signing path.

This also addresses §1 partially — if BYO external wallet is supported
day-one for signing, the v1 UI can be even thinner because the signing
flow is "show user the calldata; they sign in their wallet of choice"
exactly the way `vmm-cli.py` already works (`99-skeptic.md:179-203`).

---

## 9. API token revocation propagation latency

**Consensus.** Token validation is "sha256(token), lookup active row"
(`99-final-plan-v2.md:1188-1199`). v2 mentions caching as an open
implementation question (no, actually — searching the plan, **caching
is not mentioned**; it's `last_used_at update asynchronously` only).
The skeptic prompt said v2 mentions caching but doesn't pin staleness;
on read, the plan does *not* describe a token cache at all.

**Case against — under-engineered for the threat model.**

Two failure modes:

**(a) The plan as written is implicitly synchronous.** Every API request
does a Postgres lookup keyed by `token_hash`. At thousands of req/s on
a busy SaaS, that's a hot index path:
`CREATE UNIQUE INDEX api_tokens_hash_idx ON api_tokens(token_hash)`
(`99-final-plan-v2.md:372`). It works in v0.5; it does not scale to
thousands of req/s without a cache.

**(b) The plan as actually deployed will end up caching.** Once metrics
show p99 auth latency hitting Postgres, someone adds an in-memory
LRU with a TTL. The TTL becomes the token revocation window. If the
TTL is left at "default 5 min" (typical Rust LRU crate default), a
revoked token works for 5 minutes after revocation. For a fired
employee scenario that is the wrong default.

The v2 plan punts the question.

**For a multi-tenant SaaS where "fired employee revokes API token used
by their off-boarded laptop" matters:**

- GitHub PAT default revocation propagation is documented as "near
  real-time, can take up to a minute" — that's the 30s-cache figure.
- AWS IAM revocation can take up to 15 minutes for cached creds.
- HashiCorp Vault default is configurable per-secret.

For this product's user persona (operator running SaaS for customers,
where individual customer companies fire individual people regularly),
v1 should pin a strict propagation guarantee.

**Alternative — synchronous DB lookup, no cache, in v1.**

Concretely:

1. Every request does a `SELECT role, expires_at, revoked_at FROM
   api_tokens WHERE token_hash = $1` indexed lookup.
2. No cache. Documented as "revocation is effective within the time of
   one DB round-trip after `DELETE FROM api_tokens` commits" — sub-100ms
   typical, sub-1s worst-case.
3. Document that the system "becomes auth-DB-bound" past some throughput
   threshold; if you hit that, that's a v1.x scale-up signal.
4. If a cache ever ships, pin the TTL to ≤5s and document it
   prominently. Anything higher is a security regression.

Postgres can do this comfortably. A pgbench-style benchmark on commodity
hardware shows ~50–200k point-lookup QPS on a small indexed table. The
v1 traffic is going to be in the low thousands at most.

If a per-process LRU cache becomes necessary later, *invalidate via
LISTEN/NOTIFY*: when a token is revoked, the DB sends a Postgres NOTIFY;
every control-plane process (in v2 active-active mode) listens and
purges. Sub-second propagation, scales to many replicas.

**Cost of being wrong.**

- Wrong toward consensus (cache with default TTL): a fired admin's
  token works for the TTL window after revocation. If the TTL is
  silently set at 60s, it's tolerable; if it's silently 5 min or
  worse, it's a security incident waiting to happen.
- Wrong toward alternative (sync only): you cap throughput at
  Postgres-point-lookup-rate. Easy win to revisit in v1.x with a
  benchmark-driven cache + NOTIFY invalidation.

**Verdict. Change.** Pin the v1 design as **synchronous DB lookup, no
cache.** Document this in `00-decisions.md` as part of the API tokens
section. When/if a cache is added later, it MUST use LISTEN/NOTIFY for
sub-1s invalidation. Caches with stale TTLs are a multi-tenant-SaaS
anti-pattern.

---

## 10. IPAM supernet sizing

**Consensus.** Default `10.42.0.0/16` with `/24` per org → 256 orgs of
~250 hosts each. Operator-tunable (`99-final-plan-v2.md:1788-1825`,
`13-codex-bridge-opinion.md:34-35`).

**Case against — the documented default does not survive real SaaS scale.**

The plan acknowledges the issue:

> A `/16` yields 256 `/24` orgs, so supernet must be operator-configurable.
> The Codex opinion explicitly warns not to hard-code `/16` as the only
> option (`99-final-plan-v2.md:1823-1825`).

But the *default* is still `/16`, and defaults are what 90% of operators
ship with.

The user persona is "operator running multi-tenant SaaS." A serious SaaS
ramp pattern:
- Soft launch: 10–20 orgs.
- Year 1: 100–500 orgs.
- Year 2 if it works: 1k–5k orgs.

256 orgs is the *first 6–12 months*. Then you hit the wall.

Once you hit the wall, what happens? Operator must re-IPAM their bridges,
which means renumbering every running CVM's bridge. You can't migrate a
running bridge's subnet without restarting every CVM on it. (Bridges with
enslaved TAPs cannot be deleted and recreated without breaking guest
networking — the v2 plan acknowledges this:
`99-final-plan-v2.md:1955-1959`.)

So the operator's choice at the wall is:
- (i) Bring up a new supernet alongside, allocate new orgs there. Now
  you have a tagged "old" and "new" allocation, plus reconciliation
  complexity. This is what AWS did with VPC IP exhaustion (10.0.0.0/16
  default → 100.64.0.0/10 secondary), and it's a pain.
- (ii) Schedule a maintenance window per org, tear down + recreate
  bridges. Customer-visible downtime.

Better default avoids both. The two real candidates:

**(a) `100.64.0.0/10` (RFC 6598 — Carrier-Grade NAT).** 4 million
addresses. Specifically reserved for "shared address space among
multiple subscribers" — *exactly* this use case. Will not collide with
operator's home/office RFC1918 LAN, which is the actual configuration
risk: the operator may be SSH'd in from a 10.42.x.x corporate office
network and their tunnel breaks. /10 with /22 per org → 16k orgs of
1022 hosts each. /10 with /24 per org → 16k orgs of 254 hosts each.
The /10 is overkill on a single host but the point is that the *default*
choice doesn't surprise you in year 2.

**(b) `10.0.0.0/8` with `/22` per org.** 16k orgs of 1022 hosts each.
Simpler optics but more likely to collide with operator LAN.

**(c) Status quo with bigger default supernet.** `10.0.0.0/12`
(10.0.0.0/12 = 10.0.0.0–10.15.255.255) at `/24` per org → 4096 orgs of
~250 hosts. Less collision risk than (b), still RFC1918, more headroom
than `/16`.

The `/24` per-org sizing is fine for the per-org-host count (real CVM
counts per org will be in low double digits typical, hundreds in
extreme cases). The problem is the *count of orgs*, not the count of
CVMs per org.

**Alternative — pin default at `100.64.0.0/10` with `/22` per org.**

Reasoning:
- 100.64.0.0/10 will not collide with the operator's corporate LAN
  (which is almost always in the 10.x or 192.168.x range).
- /22 per org gives 1022 host bits, which leaves room for sub-org
  segmentation later (per-deploy-env subnets, dev/staging/prod,
  whatever).
- 1024 orgs per host at /22, 16k orgs per host at /24. Either keeps the
  v2 plan honest at scale.
- The plan can still expose `bridge_supernet` and `per_org_prefix_len`
  as operator-tunable (`99-final-plan-v2.md:1791-1800`).

If you really want RFC1918 (operators sometimes have policy reasons),
use `10.0.0.0/9` (10.0.0.0/9 = 10.0.0.0–10.127.255.255, 8 million
addresses) at /22 per org. Same headroom, RFC1918.

**Cost of being wrong.**

- Wrong toward consensus (default `/16`): half the operators who actually
  use bridge mode hit the limit in year 2. You issue an advisory.
  Operators who haven't read the doc carefully ramp into the wall and
  blame the platform.
- Wrong toward alternative (default 100.64.0.0/10 with /22): you use a
  block that some operators will find unfamiliar. Documentation in the
  installer prompt explains it. No real operational cost.

**Verdict. Change.** Default supernet to `100.64.0.0/10` with /22 per
org. Operator can override at install time. This is a one-line default
change in the installer prompt (`99-final-plan-v2.md:2147-2153`); cost
~zero now, saves a customer-visible re-IPAM later.

---

# Additional skeptical lenses

## YAGNI risks

**(Y1) Hourly audit anchor rows (`99-final-plan-v2.md:1657-1672`).** Pure
ceremony in a hash-chain-only world. The chain is its own integrity
proof; an hourly anchor adds nothing without an external verifier (which
is explicitly deferred to v2). The plan even says "If implementation
time is tight, omit anchor rows; the hash chain is sufficient for v1"
(`99-final-plan-v2.md:1670-1672`). Drop them entirely from the v1 plan.

**(Y2) `cvms.pin_numa`, `cvms.hugepages` columns (`99-final-plan-v2.md:432-434`).**
Today's VMM doesn't expose these as per-CVM tenant knobs; they're
operator-side performance config. Adding them to the per-org CVM API
surface invites tenants to ask for non-default values, which becomes a
support burden. Default to operator-config-only and don't surface in
the API for v1.

**(Y3) `Dstack-Api-Version` header versioning + 12-month deprecation policy
(`99-final-plan-v2.md:288-299`).** You don't have a v1 yet. Stripe-style
date versioning is a convention that pays off after years of API
evolution. For v1, ship plain `/v1/...` and revisit when the first
breaking change is contemplated. The deprecation-policy SLO is a
liability you don't need at launch.

**(Y4) Idempotency key storage table for 24h with `(org_id, actor_id, method,
path, key)` (`99-final-plan-v2.md:766-768`).** Real future need, but for
v1, document "client must dedupe" and skip the table. The CVM-create
case is the only really expensive replay; that's protected anyway by
`UNIQUE (org_id, name)` on `cvms` (`99-final-plan-v2.md:453`).

**(Y5) `safes.signer_cache` jsonb column (`99-final-plan-v2.md:489`).** The
plan describes it as cache-only with on-chain as source of truth
(`99-final-plan-v2.md:1322-1327`). Drop the column; query on-chain on
demand. The latency matters in the UI (which doesn't ship in v1 anyway
under consensus §1) and not elsewhere.

**(Y6) Resource usage 30-second sampler with rolling table
(`99-final-plan-v2.md:588-606`).** Useful for billing primitives, but
billing is a v2 concern (per `00-decisions.md` "Billing role is
read-only access to billing, usage metrics, and the audit log" without
ever defining what billing-the-product does). Defer the sampler; keep
the schema column.

## Under-engineered risks

**(U1) KMK loss = bricked deployment.** The "BACK THIS UP NOW" wall of text
(`99-final-plan-v2.md:2162-2184`) is a docs-shaped hope, not a recovery
mechanism. For a serious SaaS, KMK should be **Shamir-split at install
time** (e.g., 2-of-3 shares: operator password manager, encrypted backup
to a customer-chosen cloud, printed paper in a safe). The cost is one
crate (`shamir`) and a 10-minute install ceremony. Without this, "lose
your KMK and you lose Privy creds, EVM RPC, recovery signer, SMTP, and
every org's registry creds" — all your secrets, not just one. Document
the restore procedure with Shamir reassembly.

**(U2) Hash-chain insertion under SERIALIZABLE isolation
(`99-final-plan-v2.md:1628-1631`).** Every audit write contends on the
audit-tip lock. In a busy multi-tenant install with N concurrent
deploys, every deploy writes ~6 audit rows; deploys serialize on the
audit chain, not on the deploy itself. This is correct but slow and
will surface as latency under load. Mitigation: a dedicated single-row
"audit cursor" advisory lock, plus batched-insert with sub-millisecond
hold time. Or, more honestly, document the throughput ceiling.

**(U3) `dhcp-leases.sock` lease channel trust model
(`99-final-plan-v2.md:104-105`).** "filesystem permissions, root/dnsmasq
writer only." But dnsmasq's `dhcp-script` is invoked with environment
inherited from dnsmasq's process. If dnsmasq is compromised (the
attack surface for which is "any CVM on the bridge can broadcast a
DHCP request"), the script runs with whatever privileges dnsmasq's
script-invocation context grants. The lease socket needs message
authentication, not just FS permissions. A 32-byte HMAC keyed off the
control plane secret store works.

**(U4) Cross-tenant leak via shared host DNS resolver.** Per-bridge dnsmasq
forwards upstream queries (`13-codex-bridge-opinion.md:40-41`). If
operator's upstream resolver logs queries, every tenant's DNS becomes
visible to whoever holds those logs (operator, ISP). Document this in
the threat model and recommend operators run a local recursive resolver
that does not log per-query.

**(U5) Bridge reconciler 60s loop (`99-final-plan-v2.md:1939-1956`)** + lazy
bridge create + 30-day org tombstone window means an org marked for
deletion can have its bridge's CIDR re-allocated to a new org during
the GC window if the reconciler classifies wrong. The plan says GC
requires zero-CVMs + zero-TAPs + grace, but the order-of-operations
under crash-recovery is non-trivial. Worth a written "GC invariant"
section instead of just listing the steps.

**(U6) `org_registry_creds.last_verified_at` is advisory only.** A
verified-at-config-time credential can be revoked at the registry
side without dstack noticing until the next deploy fails. Add a
periodic re-verification (daily) and surface "credential health" in
the org dashboard (when there is a dashboard, see §1).

## Things synthesis split-when-merge / merged-when-split

**(S1) "Network helper" is one open question (#9) when it should be a
locked decision.** §4 above. The plan can't ship the rest of M5
without choosing the helper model; treating it as "implementation
question" understates its design impact.

**(S2) "External registry credentials" is one milestone (M6) but ECR-STS
and GAR-workload-identity are entirely different code paths from
Docker Hub / GHCR / generic-basic.** The simple cases are S+M each
(~1 week per registry); ECR-STS requires AWS SDK integration,
AssumeRoleWithWebIdentity flow, token expiry handling and refresh
(M each); GAR requires Google Cloud SDK plus workload-identity
federation (M each). M6 should be split: v0.5 ships
basic/PAT/Bearer; v1.0 adds ECR; v1.1 adds GAR. Treating them as
one milestone hides the difficulty.

**(S3) "M3 Safe / On-chain" merges six workstreams that have very different
risk profiles.** "Recovery signer timelock" is a Solidity-contract
audit-grade workstream; "alloy EVM client + config" is plumbing.
Splitting them lets you ship plumbing in v0.5 while deferring the
audited contract. Currently they sit in one milestone with one
estimate.

**(S4) The "host_id-ready schema" decision is correctly scoped, but the
matching control-plane↔host-worker trait boundary
(`99-final-plan-v2.md:80-94`) is mentioned as a sketch and not specced.**
That trait is the v2 unblock; if it isn't designed precisely in v1,
v2 has the forklift v1 skeptic warned about. Move it to a locked
decision with the exact method signatures.

**(S5) "Audit log" merges hash chain (cheap, real) with anchor rows
(ceremony) with export verifier (UX) into one section.** Split: hash
chain is locked; anchor rows are deferred (Y1); export verifier is
v1.0+.

---

# Recommended changes (priority order)

1. **Pull a slim Privy/Safe-signing UI into v1 (§1).** *Why this matters:*
   the consensus is internally inconsistent — embedded-wallet signing
   requires a browser context, so "API + CLI only with Privy" is not
   shippable. Either build the slim UI or drop Privy embedded wallets.

2. **Defer bridge mode to v1.5; ship `mode = "user"` only for tenant CVMs
   in v1 (§2).** *Why this matters:* saves 6–8 weeks of network plumbing
   and a privileged binary's attack surface for a feature whose
   adoption rate is unknown. Real adoption data drives the v1.5
   design instead of guessing.

3. **Make recovery signer mandatory in v1 (§3).** *Why this matters:*
   a single-admin org that loses Privy access is permanently bricked
   under the current default. The cost of mandatory + 24h timelock is
   low; the cost of one bricked customer is high.

4. **Drop `dstack-net-helper`; run control plane with `CAP_NET_ADMIN` via
   systemd `AmbientCapabilities` (§4).** *Why this matters:* every
   net-helper invocation model carries CVE baggage, and the control
   plane already holds the KMK — splitting "ip link add" from
   "decrypt secrets" inside the same trust boundary is privilege-sep
   cosplay.

5. **Single-dnsmasq-with-`interface=`-stanzas from day one if bridge
   mode survives item 2 (§5).** *Why this matters:* the v2 plan calls
   single-process the endgame; building it later is more expensive
   than building it now, and SIGHUP semantics don't disturb live
   leases so the failure-isolation argument is weak.

6. **Native Postgres install via PGDG, not Docker (§6).** *Why this
   matters:* removes Docker daemon dependency, removes "Postgres
   peer auth from Docker" open question, simplifies install path.

7. **Adopt a 6-month v0.5 cut (§7).** *Why this matters:* the 16–23
   month one-engineer estimate is too long for what is essentially a
   governance + multi-tenancy product. Defer features that don't
   distinguish the new product from the old VMM.

8. **Always-allow external-wallet bypass for Safe signing (§8).** *Why
   this matters:* removes Privy from the critical signing path.
   A 1-hour Privy outage no longer freezes governance globally.

9. **Pin synchronous-DB-lookup, no-cache token validation in v1; require
   LISTEN/NOTIFY invalidation if a cache is ever added (§9).** *Why
   this matters:* prevents silent revocation-latency regression. v1
   load is well within Postgres point-lookup throughput.

10. **Default IPAM supernet `100.64.0.0/10` with `/22` per org (§10).** *Why
    this matters:* saves the "256-org wall" rebuild in year 2, avoids
    operator-LAN collision, free to ship.

**Hold (consensus survives scrutiny):**

- Multi-host-ready schema with `host_id` FKs (already accepted; do not
  unwind).
- Per-org Linux uid as defense-in-depth (cheap, real; keep — though it
  can move to v1.0 from v0.5 per §7).
- `app_id = truncate40(sha256(org_id || compose))` derivation
  (`00-decisions.md:113-117`). Solves cross-org collisions cleanly.
- Hash-chained audit log (cheap; real tamper detection). Keep, drop
  signed batches and anchor rows per Y1.
- Greenfield-only release (the v1 skeptic's import-path push-back was
  declined; reasonable user-side tradeoff).
- 4-role RBAC (Admin/Deploy/Billing/Viewer). v1 skeptic recommended;
  consensus accepted; correct.
- External registry creds instead of in-house OCI registry. v1 skeptic
  recommended; consensus accepted; correct. (Split ECR/GAR per S2.)

---

**Closing.** The v2 plan is a substantial improvement over v1 — Deploy
role, external registries, dropped signed batches, host_id-ready
schema, per-org bridges over VLAN tagging are all the right calls. The
remaining issues are not in *what* the plan picks but in:

- **Internal consistency** (§1: API+CLI vs Privy embedded wallets).
- **Scope discipline** (§2, §7: too much for v1).
- **Operational defaults** (§5, §6, §10: defaults that age badly).
- **Critical-path single-vendor risk** (§3, §8: Privy on Safe path).
- **Privilege-design over-engineering** (§4: net-helper is a
  parser-between-halves-of-the-same-brain).
- **Under-specified caching/recovery primitives** (§9, U1, U3).

The pattern across these is similar to the v1 skeptic's: the synthesis
agent consistently picks the more-ambitious-for-v1 choice over
ship-less-defer-more, and consistently builds for "production SaaS at
scale" before "first 100 customers." Pull the v1 cut back; the rest is
sound.
