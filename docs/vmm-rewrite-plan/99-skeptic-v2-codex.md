# 99 — Skeptic v2 pass on the revised dstack-vmm rewrite plan

**Prepared:** 2026-05-06  
**Stance:** loyal opposition to `99-final-plan-v2.md`, with special attention to
places where the synthesis may have over-architected, over-budgeted, or chosen the
more sophisticated option because it felt cleaner.

This pass does not re-run the v1 skeptic arguments unless v2 changed the risk. The
revised decisions already accepted 8 of 10 v1 push-backs and explicitly declined
Privy and Postgres (`00-decisions.md:398-403`). The point here is to challenge the
new consensus, especially where `99-final-plan-v2.md` added implementation choices
that were not in the locked decisions.

---

## 1. Web UI deferred to v1.5

**Consensus.** v1 ships API + OpenAPI + a narrow CLI only; a production
multi-tenant Web UI is deferred to v1.5 unless a separate UI engineer is available
(`99-final-plan-v2.md:2310-2319`).

**Case against it.** This is the most product-risky decision in v2. The plan
describes a SaaS-style platform where operator customers onboard their own orgs and
members, configure registry credentials, sign Safe prompts, inspect quota and usage,
export audit logs, and debug CVM lifecycle. That persona is not realistically served
by curl, generated OpenAPI docs, and a CLI.

The code evidence cuts both ways:

- The current UI is not reusable as-is. It is a single embedded Vue page with a
  1700-line controller, no real bundler, and no multi-tenant auth model
  (`02-vmm-web-ui.md:13-52`).
- But that same report lists the exact UI surfaces a multi-tenant rewrite needs:
  login, org switcher, scoped VM views, logs authorization, host/operator split,
  signature verification for env encryption, and audit activity
  (`02-vmm-web-ui.md:797-899`).
- The revised backend plan already exposes browser-shaped session auth and cookie
  flows (`99-final-plan-v2.md:783-818`, `99-final-plan-v2.md:1158-1183`). It is
  building the server half of a web product while declining the client half.
- Soft quota warnings become awkward without a UI: v2 says `notifications` can be
  added only "if UI ships", otherwise warnings are hidden in quota/audit endpoints
  (`99-final-plan-v2.md:1767-1774`). That is not acceptable for an org admin trying
  to operate capacity.
- Admin add/remove relies on a "Pending removal" UI state in the locked decisions
  (`00-decisions.md:102-108`) and v2 returns a signing URL for Safe actions
  (`99-final-plan-v2.md:887-903`). A CLI can drive this, but it is not the user
  experience implied by the Safe/Privy design.

The minimum viable UI is not a full cloud console. It is:

- Login/session with Privy.
- Org switcher and member/invitation management.
- Safe transaction queue with sign/submit/retry states.
- Registry credential CRUD, including redacted display and verify result.
- CVM list/detail/start/stop/restart/logs/deploy form.
- Quota/usage view and audit event list/export.
- Platform-admin bootstrap page for first user, secrets status, org suspension, and
  quota overrides.

That omits billing, rich dashboards, custom domains, bridge diagnostics beyond
admin-only status, image catalog management polish, and every marketing page. It is
still likely 8-12 weeks for one frontend-capable engineer, but it converts v1 from
"API substrate" into a usable v1.

**Alternative.** Move a minimal Web UI into v1 and cut elsewhere to pay for it.
Use a separate static SPA served by the control plane; do not reuse the embedded
legacy UI except for visual/flow references. Keep it thin and API-driven. The CLI
remains for automation and local debugging, but not as the primary admin surface.

**Cost of being wrong.**

- Wrong toward the consensus: v1 technically ships but fails the target persona.
  Early org admins have to copy JSON into curl for membership, Safe signing, and
  registry credentials; support load moves to the operator; the platform feels like
  an integration kit, not a SaaS control plane.
- Wrong toward the alternative: v1 slips by 8-12 weeks and the UI may need redesign
  after real usage. But UI work will be needed anyway; doing the minimum now
  validates the API and catches auth/Safe usability bugs earlier.

**Final recommendation: change.** Ship a minimum viable UI in v1. Defer bridge mode,
ECR/GAR, and the optional recovery timelock before deferring the only interface most
tenant admins can reasonably use.

---

## 2. Bridge mode in v1 at all

**Consensus.** v1 supports bridge-mode CVMs through per-org Linux bridges, per-org
IPAM, dnsmasq, nftables, and reconciliation, with user-mode SLIRP still the default
(`00-decisions.md:250-286`; `99-final-plan-v2.md:1779-1987`).

**Case against it.** The isolation finding is correct, but the product conclusion
may be wrong. The F3 report is explicit: if v1 ships only `mode = "user"`, no VLAN
or per-org bridge isolation is needed because QEMU SLIRP is per-VM and has no shared
L2 segment (`12-f3-vlan-isolation.md:95-105`). The Codex bridge opinion says the
same thing: disabling bridge mode is the only simpler secure option
(`13-codex-bridge-opinion.md:71-74`).

The current code strongly supports a user-mode-only v1 cut:

- The default config is already `mode = "user"` (`vmm/vmm.toml:86-96`).
- User mode builds a per-VM QEMU userspace network, while bridge mode attaches to a
  named host bridge (`vmm/src/app/qemu.rs:504-526`).
- The current bridge bridge-name is global; there is no org, VLAN, or namespace in
  config (`vmm/src/config.rs:386-427`).
- Bridge mode is recommended today for high concurrency, L2 access, or LAN
  reachability (`docs/bridge-networking.md:3-9`), not for the ordinary "web app
  behind dstack-gateway" workload.

For web-app workloads, `mode = "user"` plus dstack-gateway covers the normal shape:
outbound registry/API calls, inbound HTTPS/TLS through gateway, per-CVM port policy,
and no tenant L2. Bridge mode mainly serves LAN discovery, raw sockets, multicast,
direct LAN reachability, or very high connection counts. Those are real use cases,
but likely minority v1 use cases.

The v2 plan admits the cost: bridge networking adds roughly 8-12 weeks in the
estimate and 6-8 weeks versus user-mode-only v1 (`99-final-plan-v2.md:2271-2284`,
`99-final-plan-v2.md:2325-2342`). That is nearly the budget for a minimal UI.

**Alternative.** Make v1 tenant networking `mode = "user"` only. Keep bridge mode
behind a platform-admin-only unsafe override for pilots that explicitly accept the
operational risk, or behind a compile/config flag that is not exposed to org users.
Do not build per-org bridge orchestration until v1.5.

Cheap intermediate: allow platform admins to manually pre-provision a bridge for a
single trusted org and mark that org as `bridge_mode_allowed`. That is not
self-service and not a general multi-tenant feature, but it supports exceptional
pilots without committing v1 to a full network control plane.

**Cost of being wrong.**

- Wrong toward the consensus: v1 burns 6-12 weeks on a minority network mode,
  introduces root network mutation, dnsmasq lifecycle, nftables drift, IPAM, and
  more E2E tests before the core product has a UI.
- Wrong toward the alternative: some high-concurrency or LAN-reachability customers
  cannot use v1, or use it only through an admin-approved exception. If those
  customers are the launch target, this cut hurts.

**Final recommendation: change.** Ship user-mode-only tenant networking in v1, with
a platform-admin bridge escape hatch for named pilots. Move full per-org bridge
mode to v1.5 after the core SaaS flows work.

---

## 3. Stuck-Safe edge case

**Consensus.** Safe signer changes are Safe-first: add admin only after
`addOwnerWithThreshold` confirms; remove admin by proposing/signing
`removeOwner` first, then deleting the DB membership. The recovery signer is
optional, opt-in at install, and off by default (`00-decisions.md:100-125`;
`99-final-plan-v2.md:1220-1259`, `99-final-plan-v2.md:1329-1347`).

**Case against it.** Safe-first ordering is coherent only if there is always at
least one usable signer. With recovery disabled, that assumption is false. If the
only reachable admin permanently loses Privy access, or Privy cannot sign for the
embedded wallet, the org can become unable to add/remove admins or perform
on-chain app actions. The DB still has an admin row, but that admin cannot exercise
the Safe signer. The locked decisions explicitly make recovery optional and off by
default (`00-decisions.md:118-125`), and v2 goes further: if no audited timelock
helper is available, recovery remains hidden behind an experimental warning
(`99-final-plan-v2.md:1342-1347`).

The ordering also gives the control plane no clean local escape. Admin removal
sets `pending_removal`, blocks new Admin-only actions, and waits for chain
confirmation before deleting the row (`99-final-plan-v2.md:1242-1259`). That helps
malicious-admin removal, but it does not help "no signer can sign."

**Alternative.** For v1, choose one of these, in order of preference:

1. Make platform recovery mandatory for every org Safe, but simplify it: no custom
   timelock helper in v1. Use a documented platform-admin break-glass flow with
   loud audit events, exportable recovery reports, and an operator-held hardware
   wallet or encrypted key. Add the on-chain timelock contract in v1.5.
2. If mandatory recovery is unacceptable, add a stuck-Safe rescue flow: platform
   admin can quarantine the org, freeze deploy/admin changes, record an audit event,
   and submit a recovery signer action after an explicit waiting period.
3. If neither recovery path is acceptable, relax Safe-first for admin membership:
   allow DB-first quarantine for lost-admin cases, with on-chain cleanup retried
   lazily and visibly flagged as Safe drift.

The plan already has a `safes.signer_cache` and mismatch repair flow
(`99-final-plan-v2.md:1322-1327`), so representing drift is not foreign to the
model.

**Cost of being wrong.**

- Wrong toward the consensus: an org can be bricked in v1, and the operator has no
  auditable product path except telling the customer that on-chain ownership is
  unrecoverable.
- Wrong toward mandatory/break-glass recovery: the operator gets a powerful
  capability that weakens org sovereignty. If abused or compromised, it can take
  over org Safes. That is why it must be highly visible and auditable.

**Final recommendation: change.** Do not ship optional-off recovery with strict
Safe-first ordering. Make recovery mandatory for v1, or provide a platform-admin
stuck-Safe rescue path. The current combination is under-engineered for account-loss
recovery.

---

## 4. `dstack-net-helper` design choice

**Consensus.** v2 introduces a new root `dstack-net-helper` for bridge, nftables,
dnsmasq, and `/etc/qemu/bridge.conf` mutations, but leaves the exact interface open:
sudoers fixed commands, `systemd-run`, or a long-running root daemon
(`99-final-plan-v2.md:67-74`, `99-final-plan-v2.md:175-178`,
`99-final-plan-v2.md:2361-2365`).

**Case against it.** This was not in `00-decisions.md`; the synthesis introduced a
privileged boundary without committee review. That matters because each option has
real security and operational baggage:

- `sudo` fixed-command designs inherit sudo parser/env complexity. Historical
  examples are not theoretical: CVE-2021-3156 was a sudo heap overflow leading to
  root privilege escalation, and CVE-2023-22809 was a sudoedit privilege escalation
  through environment-controlled editor arguments
  (`https://nvd.nist.gov/vuln/detail/CVE-2021-3156`,
  `https://nvd.nist.gov/vuln/detail/CVE-2023-22809`). Even if those exact bugs are
  patched, the pattern is "general-purpose privilege broker plus argument/env
  edge cases."
- `systemd-run` moves the trust boundary into PID1/D-Bus/polkit policy. Polkit's
  pkexec CVE-2021-4034 is the canonical local privilege-escalation pattern for
  this family (`https://nvd.nist.gov/vuln/detail/CVE-2021-4034`). systemd itself
  has also had local D-Bus parsing DoS issues, e.g. CVE-2019-6454
  (`https://nvd.nist.gov/vuln/detail/CVE-2019-6454`).
- A long-running root daemon is bespoke privileged code. It avoids sudo/polkit
  argument injection, but now dstack owns a root protocol, authz, input validation,
  lifecycle, and update story.
- Giving `CAP_NET_ADMIN` to the whole control plane is simpler operationally, but
  it attaches network mutation to the process that also handles Privy sessions,
  registry secrets, KMK decrypts, EVM RPC credentials, and user-facing HTTP
  (`99-final-plan-v2.md:67-71`). A route-level RCE becomes a host-network RCE.

If bridge mode is deferred, this entire boundary can disappear from v1. That is
another argument for item 2.

**Alternative.** If bridge stays in v1, pick one default now: a minimal
long-running root daemon on a Unix socket with:

- peer-credential check accepting only the `dstack` uid;
- a closed enum of operations (`ensure_bridge`, `apply_nftables_generation`,
  `write_dnsmasq_config`, `restart_dnsmasq`, `update_qemu_allowlist`);
- full desired-state payload validation, not shell command passthrough;
- no inherited environment, no shell, no arbitrary argv;
- audit logs in the control plane before and after every helper call.

Do not support three privilege models in v1. Do not put ambient `CAP_NET_ADMIN` on
the main control plane unless the helper cannot be built and bridge is still
required.

**Cost of being wrong.**

- Wrong toward the consensus/open choice: implementation drifts into whichever
  mechanism is easiest late in the schedule, likely sudoers or systemd-run, with
  insufficient threat review.
- Wrong toward the root daemon: we build and maintain a small privileged service.
  If validation is wrong, it is a root bug. The benefit is that the attack surface
  is dstack-specific and reviewable.
- Wrong toward `CAP_NET_ADMIN` on the control plane: fewer moving parts, but a
  user-facing API compromise has more immediate host-network power.

**Final recommendation: change.** If bridge mode is deferred, delete
`dstack-net-helper` from v1. If bridge remains, make a tiny root daemon the only
v1 helper design and drop sudoers/systemd-run as supported options.

---

## 5. dnsmasq per bridge

**Consensus.** v1 runs one dnsmasq process per org bridge for failure isolation and
simple config generation (`00-decisions.md:261-266`;
`99-final-plan-v2.md:1851-1889`).

**Case against it.** This is another place where "simple now, migrate later" may
be backwards. The plan itself flags dnsmasq process scaling as a risk and says a
future single-process config generator may be needed at hundreds of orgs
(`99-final-plan-v2.md:2379-2381`). If hundreds of orgs is within plausible SaaS
scale, designing the single-process model now is cheaper than migrating process
supervision, health metrics, config hashing, and restart semantics later.

dnsmasq has also had real network-reachable memory safety issues, including
CVE-2017-14491 and CVE-2020-25681 heap overflows in DNS handling
(`https://nvd.nist.gov/vuln/detail/CVE-2017-14491`,
`https://nvd.nist.gov/vuln/detail/CVE-2020-25681`). One process per org is better
for blast radius if a tenant can feed dnsmasq malicious packets, but it also
multiplies daemon count, PID files, systemd units, logs, reloads, and per-process
memory.

The single-process design is not exotic: generate one dnsmasq config with repeated
`interface=`, `dhcp-range=tag:<bridge>`, and per-interface options; bind only the
managed `dstack-org-*` interfaces; keep lease notifications bridge-aware as v2
already requires (`99-final-plan-v2.md:1867-1885`). The hard part is safe config
generation, and v2 is already building that hard part for per-bridge configs.

What breaks if a single dnsmasq is hot-reloaded or restarted N times per
org-create per day? New DHCP/DNS requests may fail briefly across all org bridges.
Existing leases keep working; guests retry DHCP; Docker/container DNS retries are
usually tolerant of short outages. If org creation is frequent enough that this is
visible, batch config apply or debounce reloads for a few seconds.

**Alternative.** If bridge mode ships in v1, use a single managed dnsmasq process
from day one, with generated config validation and debounced reload/restart. If
the team insists on per-bridge dnsmasq, define the migration threshold in the v1
decision, not as an open question.

**Cost of being wrong.**

- Wrong toward per-bridge: hundreds of orgs become hundreds of root-ish network
  daemons and systemd units. Migration later touches config layout, metrics, health
  checks, and lease handling.
- Wrong toward single-process: a bad config or restart can affect DHCP/DNS for all
  bridge-mode orgs at once. That is a real shared-failure domain.

**Final recommendation: change if bridge remains.** Start with single dnsmasq if
bridge mode is in v1. If item 2 is accepted and bridge moves to v1.5, defer the
dnsmasq decision with it.

---

## 6. Postgres in Docker versus native install

**Consensus.** v1 uses Postgres 16 in a sibling Docker container on the same host,
with Unix-socket peer auth and no DB password in the normal install
(`00-decisions.md:127-135`; `99-final-plan-v2.md:2107-2135`).

**Case against it.** The v1 skeptic already argued SQLite and the revised
decisions declined that push-back (`99-skeptic.md:810-924`;
`00-decisions.md:398-403`). This v2 pass should not reopen SQLite. But the Docker
part still deserves scrutiny.

The plan gains little from containerizing Postgres:

- There is no HA, replication, managed backup, or isolation story gained by the
  container. v1 is one box and one control-plane process (`00-decisions.md:30-53`).
- The installer already creates system users, writes `/etc/dstack`, manages a
  systemd service, starts Postgres, and runs migrations (`99-final-plan-v2.md:2137-2160`).
  Adding `postgresql.service` is operationally consistent.
- Docker adds a daemon dependency to a product otherwise framed as a single host
  service. The v1 skeptic already called out "Docker daemon dead = control plane
  dead" as a failure mode (`99-skeptic.md:897-903`).
- The v2 compose example is illustrative and not production-correct: it uses
  `POSTGRES_HOST_AUTH_METHOD: trust`, then says the actual build must produce a
  tested peer-auth Unix socket config (`99-final-plan-v2.md:2111-2135`). That is a
  sign the container boundary is adding ceremony.
- Unix-socket peer auth across a container boundary means UID/socket ownership and
  bind mounts must be correct across Ubuntu/Debian variants. v2 lists this as an
  open question (`99-final-plan-v2.md:2355-2358`). Native Postgres makes peer auth
  the default shape, not a bind-mount trick.

**Alternative.** Keep Postgres, but install/run it natively via apt/dnf/systemd in
the recommended production path. The installer can support "use existing Postgres"
for operators who want containers or managed DBs. Docker compose can remain a dev
or demo path, not the v1 production default.

**Cost of being wrong.**

- Wrong toward Docker: production installs depend on Docker, socket bind mounts,
  container UID mapping, and a not-yet-proven peer-auth setup.
- Wrong toward native: packaging differs across distributions, and upgrades are
  handled by the host OS instead of a pinned image tag. Some operators prefer
  containerized Postgres because their runbooks already cover it.

**Final recommendation: change.** Keep Postgres, but make native systemd Postgres
the default production install. Offer Docker only as an alternate path.

---

## 7. 16-23 month solo / 9-12 month two-engineer estimate

**Consensus.** The revised v1 plan is 71-98 engineer-weeks: 16-23 months for one
engineer, 9-12 months for two parallel engineers (`99-final-plan-v2.md:2321-2342`).

**Case against it.** The estimate is defensible for the full v1 plan, but the full
v1 plan may be the wrong product cut. This is where GPT-5-style synthesis may be
over-budgeting risk by preserving too many "correct" surfaces at once:

- Full Safe/Privy on-chain orchestration is 14-20 weeks
  (`99-final-plan-v2.md:2249-2258`, `99-final-plan-v2.md:2325-2327`).
- Bridge networking is 8-12 weeks (`99-final-plan-v2.md:2271-2284`,
  `99-final-plan-v2.md:2327-2329`).
- ECR/GAR adapters add multiple M-sized items (`99-final-plan-v2.md:2286-2295`).
- Optional recovery signer timelock is L-sized and may require a helper contract
  (`99-final-plan-v2.md:2258`, `99-final-plan-v2.md:1331-1347`).
- The plan defers UI even though a minimal UI is likely the best way to validate
  the SaaS workflows (`99-final-plan-v2.md:2310-2319`).

A credible six-month v0.5 can still be genuinely multi-tenant if "multi-tenant"
means org-scoped identity, RBAC, quotas, audit, per-org CVMs, and secret/registry
isolation on one host. It does not need every v1.5 production feature.

**Alternative.** Define a v0.5 six-month cut:

- Keep: Postgres schema, Privy sign-in, orgs/members/invitations, four-role RBAC,
  API tokens, hash-only token storage, per-org CVM ownership, quotas, audit rows,
  sodium secrets, Docker Hub/GHCR/GitLab/generic registry creds, DB-driven CVM
  lifecycle, user-mode networking, gateway ingress, minimal UI, CLI basics,
  installer, backups docs.
- Defer: full bridge mode; ECR/GAR cloud adapters; optional recovery timelock
  contract; M-of-N thresholds; custom domains; pull-through cache; signed audit
  batches; webhooks; per-org guest OS images.
- Simplify Safe for v0.5: either defer Safe entirely and use external wallet
  calldata signing for app registration, or ship a managed/operator Safe path for
  pilots only. If Safe remains mandatory, make recovery mandatory and cut bridge
  plus ECR/GAR to compensate.
- Consider deferring per-org Linux uid only if the filesystem isolation is not
  ready. I would prefer keeping per-org uid because it is a cheap hardening
  extension of today's `setup-user.sh` owner firewall model
  (`vmm/src/setup-user.sh:97-182`) and protects secrets/workdirs better than naming
  alone.

This cut is not "toy multi-tenant." It has users, orgs, roles, tokens, audit,
quotas, tenant-owned CVMs, and tenant registry credentials. The main missing
feature is bridge-mode workloads.

**Cost of being wrong.**

- Wrong toward the full estimate: the project spends a year or more before real
  tenant admins can use it, and architectural assumptions remain untested.
- Wrong toward v0.5: early users hit missing bridge/ECR/GAR/Safe sophistication and
  perceive the product as incomplete. Support must be clear about "web apps and
  GHCR/Docker Hub first."

**Final recommendation: change.** Publish a six-month v0.5 milestone before the
full v1. Cut bridge, ECR/GAR, and optional recovery timelock first; move minimal UI
into the milestone.

---

## 8. Privy single-vendor critical path

**Consensus.** Privy is the identity provider and every Safe interaction routes
through Privy-backed embedded or connected wallet signing; local normalized users
and a future provider trait mitigate lock-in (`00-decisions.md:55-61`;
`99-final-plan-v2.md:1081-1105`, `99-final-plan-v2.md:2387-2390`).

**Case against it.** The v1 skeptic accepted Privy only with reservations
(`99-skeptic.md:236-312`). v2 makes the risk sharper because Privy is now not only
login; it is on the governance path. Admin add/remove, Safe owner sync, and app
compose/device actions all require signing through the Privy flow
(`99-final-plan-v2.md:1220-1259`, `99-final-plan-v2.md:1307-1327`).

The mitigation in v2 does not address outages. Storing normalized users/wallets
locally helps future migration and read-side continuity, but during a one-hour
Privy outage:

- new sign-ins may fail;
- embedded-wallet signing fails;
- external-wallet signing through Privy connectors may fail;
- admin changes cannot confirm on-chain;
- deploys that require Safe/KMS app transactions may block.

The v2 risk section says existing sessions and machine tokens can continue during
Privy trouble only indirectly via local state; it does not provide an outage-safe
signing path (`99-final-plan-v2.md:2387-2390`). That is insufficient for governance.

**Alternative.** Add an outage-resilient signing path in v1:

- The control plane can render the SafeTx EIP-712 payload or Safe transaction hash.
- A user signs it with any external wallet outside Privy.
- The control plane verifies the recovered signer address against
  `user_wallets`/Safe owners and submits the transaction.
- UI supports "sign externally" as a fallback; CLI supports paste-in signature.

This is not a full non-Privy auth provider. It is a break-glass signing path for
governance and deploy continuity. It also validates the data model claim that one
user can have many wallets and one primary signer (`00-decisions.md:60-61`).

**Cost of being wrong.**

- Wrong toward the consensus: Privy outage or API drift pauses governance and some
  deploys platform-wide. The provider trait does not help during the incident.
- Wrong toward the alternative: v1 carries a second signing UX and more tests. It
  may confuse non-crypto users if surfaced too prominently.

**Final recommendation: change.** Keep Privy for the primary UX, but require an
external-signature fallback for SafeTx signing in v1. Privy should not be the only
governance signing path.

---

## 9. API token revocation propagation latency

**Consensus.** API tokens are org-bound bearer tokens, stored as hashes, checked
against Postgres in the flow diagram; v2 mentions caching risk only indirectly and
does not pin revocation staleness (`00-decisions.md:81-93`;
`99-final-plan-v2.md:1185-1200`).

**Case against it.** The plan is ambiguous. The happy-path diagram shows a DB
lookup on every token use (`99-final-plan-v2.md:1185-1197`), which implies
synchronous revocation. But the prompt notes v2 mentions caching without a pinned
staleness budget, and this matters in a fired-employee or leaked-token scenario.

For v1, caching token auth is premature:

- Tokens are sha256 hashes with a unique index (`99-final-plan-v2.md:357-373`).
  One indexed lookup per request is operationally cheap compared with QEMU, Safe,
  registry, and audit work.
- The control plane is single-process and same-host with Postgres
  (`00-decisions.md:30-53`, `00-decisions.md:127-135`), so there is no multi-replica
  cache invalidation problem to solve in v1.
- A 30-second stale token can start/stop/restart CVMs, read logs, create deploys,
  or in the Admin case revoke other tokens and mutate members. For a fired employee
  case, "revoked but still works for 30 seconds" is not a harmless convenience.
- The plan already says `last_used_at` can be updated asynchronously
  (`99-final-plan-v2.md:1193-1195`), which is the right place to save writes. Do
  not save the active-token read.

**Alternative.** In v1, every API-token-authenticated request performs the DB
lookup for `token_hash`, `revoked_at IS NULL`, `expires_at`, org status, and token
role. If caching is later needed, use a strict <5s TTL plus explicit invalidation
on `DELETE /tokens/:id`, but do not start there.

**Cost of being wrong.**

- Wrong toward caching: revoked tokens retain power after the admin believes they
  are dead. The more destructive the token role, the worse this gets.
- Wrong toward synchronous DB lookup: high-QPS API-token workloads spend more time
  in Postgres. For v1's management API, that cost is likely negligible.

**Final recommendation: change/clarify.** Pin v1 to synchronous DB lookup on every
API-token request. No positive auth cache in v1; async `last_used_at` is fine.

---

## 10. IPAM supernet sizing

**Consensus.** v2 uses /24 per org and gives `10.42.0.0/16` as the install default,
while warning that `/16` yields only 256 orgs and the supernet is configurable
(`00-decisions.md:257-260`; `99-final-plan-v2.md:1787-1825`,
`99-final-plan-v2.md:2147-2153`).

**Case against it.** The plan knows the default is too small and still presents it
as the default. For a SaaS-style platform, 256 bridge-capable orgs is not a serious
ceiling. It is fine for a lab; it is not fine as the first-run default.

The harder issue is overlap. `10.0.0.0/8` gives huge capacity, but it collides with
many corporate VPNs, VPCs, home routers, and Kubernetes defaults. The installer
does warn operators to choose a non-overlapping RFC1918 supernet
(`99-final-plan-v2.md:2147-2153`), but good defaults matter.

`100.64.0.0/10` is attractive because it is RFC 6598 shared address space rather
than ordinary RFC1918. Many operators are less likely to route it internally than
`10/8` or `192.168/16`. With `/22` per org, it gives 16,384 org networks, each with
about 1022 usable IPv4s. That is far above v1 quota defaults, and it avoids the
"256 orgs" trap. If an operator environment already uses CGNAT space, they can
override it.

The `/24` per-org default is also generous. v1 quota defaults are 5 CVMs per org
(`99-final-plan-v2.md:552-570`), so `/22` is more than enough; even `/26` would
often suffice. But `/22` plus `100.64/10` gives both easy mental math and scale.

**Alternative.** Default bridge IPAM to:

```toml
supernet = "100.64.0.0/10"
per_org_prefix_len = 22
gateway_offset = 1
dhcp_start_offset = 10
dhcp_end_offset = 1022
```

If the team wants RFC1918 only, default to `10.0.0.0/8` with `/22`, but be explicit
that overlap is more likely and installer validation should warn on detected host
routes.

**Cost of being wrong.**

- Wrong toward `/16` + `/24`: operators hit a hard 256 bridge-org ceiling and need
  disruptive renumbering.
- Wrong toward `100.64/10` + `/22`: some environments already use CGNAT space and
  must override. Per-org subnets are larger than v1 needs, but capacity is still
  high enough.

**Final recommendation: change.** Default to `100.64.0.0/10` with `/22` per org,
or `10.0.0.0/8` with `/22` if RFC1918 is mandatory. Do not ship `10.42.0.0/16`
as the default.

---

## Additional skeptical lenses

### YAGNI risks

- **Full bridge orchestration in v1** is the clearest YAGNI risk. It is correct if
  bridge mode ships, but v1 does not need bridge mode for normal web-app workloads.
- **ECR/GAR in v1** is likely over-scoped. Docker Hub, GHCR, GitLab, and generic
  username/PAT cover enough early users. Cloud registry adapters can be v1.5.
- **Recovery timelock helper contract** is too much if recovery is optional and
  hidden as experimental. Either make recovery mandatory and simple, or defer the
  custom timelock.
- **Large metric surface with `org_id` labels** risks cardinality pain before the
  product has proven scale. Keep core metrics, but bridge/dnsmasq metrics should
  move with bridge mode if bridge is deferred.

### Under-engineered for the threat model

- **Privy outage resilience** is under-engineered. Local user records do not help
  if Safe signing depends on Privy.
- **Stuck-Safe recovery** is under-engineered. Optional-off recovery plus
  Safe-first admin changes can brick org governance.
- **API token revocation** needs a hard staleness guarantee. The simplest guarantee
  is no positive cache.
- **Database tenant isolation** still relies on application `WHERE org_id` checks.
  Report 06 warns that a single missed filter is a full breach and suggests RLS or
  dedicated org storage as a hard boundary (`06-vmm-auth-today-and-gaps.md:310-314`).
  v2 uses application-scoped queries but does not commit to RLS. At minimum, add an
  implementation note to evaluate Postgres RLS for org-scoped tables before v1.

### Split versus merge

- **Network helper and bridge mode should be merged as one decision.** If bridge is
  not in v1, the helper, dnsmasq model, nftables reconciliation, and IPAM default
  should not be in v1 either.
- **Web UI and Privy/Safe should be merged as one product decision.** A Privy
  embedded-wallet Safe signing model without a web UI is internally inconsistent.
  CLI can exist, but Privy signing prompts are browser-native product flows.
- **Postgres and Docker should be split.** Postgres as the database may hold; Docker
  as the production packaging default does not automatically follow.

---

## Recommended changes

1. **Move a minimum viable Web UI into v1.** This is what makes the multi-tenant
   platform usable by customer org admins rather than only by API integrators.
2. **Defer full bridge mode to v1.5; ship tenant `mode = "user"` only plus a
   platform-admin bridge override.** Saves 6-12 weeks and removes the root network
   helper/dnsmasq/nftables work from the v1 critical path.
3. **Make stuck-Safe recovery mandatory or add an auditable break-glass rescue
   flow.** Optional-off recovery plus Safe-first ordering can brick an org.
4. **Add a non-Privy external-signature fallback for SafeTx signing.** Privy should
   not be the only governance signing path during outages.
5. **Run Postgres natively by default, not in Docker.** Keep Postgres, but remove
   Docker daemon, UID mapping, and socket bind-mount complexity from production
   installs.
6. **Pin API token revocation to synchronous DB lookup in v1.** Revoked means dead
   immediately; no positive auth cache until scale proves it is needed.
7. **Define a six-month v0.5 cut.** Keep org/RBAC/quota/audit/CVM lifecycle/UI;
   defer bridge, ECR/GAR, and optional recovery timelock.
8. **If bridge remains in v1, choose one privileged helper model now: a tiny root
   Unix-socket daemon.** Do not leave sudoers/systemd-run/root-daemon as an open
   late-stage decision.
9. **If bridge remains in v1, prefer single dnsmasq from day one.** Avoid migrating
   hundreds of per-org dnsmasq units later.
10. **Change the default bridge IPAM to `100.64.0.0/10` with `/22` per org.** The
    documented `10.42.0.0/16` default is capped at 256 orgs and is not SaaS-scale.
