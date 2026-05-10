# Locked decisions — dstack-vmm rewrite

This is the source-of-truth decisions doc. Originally agreed in 4 rounds with
the user (2026-05-05); revised after the skeptic pass + Codex bridge-isolation
second-opinion (2026-05-06). The synthesis pass reads this file as its primary
input.

## TL;DR

A new single binary, `dstack-control-plane` (working name), **replaces**
`dstack-vmm` for fresh installs of dstack. Greenfield only — no backwards
compat with `vmm-cli.py`, the existing prpc surface, or existing on-chain
registrations. The new binary manages **exactly one** TDX bare-metal box,
runs alongside Postgres in Docker on that same box, and is the multi-tenant
control plane for that box's customers.

Identity is **Privy** (per-user embedded wallet + BYO external wallet).
Authorization is four-role org-scoped RBAC (Admin / Deploy / Billing /
Viewer) plus a global Platform Admin role for the operator. On-chain
authority is a Safe multisig on **Base mainnet (chain ID 8453)** per org,
where current admins are auto-synced as 1-of-N signers. State lives in
Postgres. Secrets live in a sodium-secretbox-encrypted table keyed by an
operator-supplied master key. Audit log is hash-chained (tamper-evident by
construction). Container images are pulled from per-org external private
registries (Docker Hub, GHCR, ECR, etc.) using credentials each org
configures — there is no in-house OCI registry server in v1. Multi-tenant
network isolation between bridge-mode CVMs is enforced via **per-org
Linux bridges** (lazily created, /24 per org from a configurable supernet).

## 1. Architectural shape

- **One binary, one host.** `dstack-control-plane` is a single process that
  combines the management surface and the host-side QEMU/supervisor
  responsibilities of today's `dstack-vmm`. There is no separate "node" or
  "scheduler" component — the binary manages exactly one TDX bare-metal box.
  Multi-host orchestration is **out of scope** for v1.
- **Schema is multi-host-ready even though impl isn't.** Org-scoped tables
  carry a `host_id` foreign key from day one (single-row `hosts` table in
  v1) so that v2 multi-host doesn't require a forklift migration. Skeptic
  pass push-back accepted.
- **Greenfield only.** Existing dstack-vmm installs do *not* migrate. They
  keep running on the old binary; new installs install the new binary. No
  data migration, no on-chain re-registration, no `vmm-cli.py` compat,
  no prpc-v1 compat. No import path for existing CVMs.
- **Trust boundary: outside the TEE.** The control plane runs as a regular
  Linux process on the bare-metal TDX host, like today's `dstack-vmm`. The
  TEE protects the guest CVMs, not the host orchestrator. *(Future option,
  explicitly noted: a v2 mode that runs the control plane inside an
  attested CVM, sidestepping operator-side tampering of user data. Out of
  scope for v1; install path must remain easy.)*
- **Single control-plane process.** No HA, no active-active. Operators who
  want HA can run two stateless API replicas pointing at the same Postgres
  in v2; not designed in for v1.

## 2. Identity, auth, RBAC

- **Privy = identity provider** (email / SMS / OAuth via Privy) + per-user
  embedded wallet. Bring-your-own external wallet supported via Privy
  connectors from day one. No password auth.
- **One user → many wallets**, exactly one marked as the **primary signer**;
  the UI never asks "which wallet do you want to sign with."
- **Off-chain RBAC, four roles, org-scoped only:**
  - **Admin** — full control of the org, including Safe signer authority.
    No separate "Owner" role — admins *are* owners.
  - **Deploy** — can push/pull org-scoped images, deploy / start / stop /
    restart CVMs, view CVM logs and state. **Cannot** delete CVMs, change
    org settings, manage members, or manage org-level secrets. The
    least-privilege role for CI/CD pipelines and scoped automation.
  - **Billing** — read-only access to billing, usage metrics, and the
    audit log; no mutation rights.
  - **Viewer** — read-only access to non-financial org resources (CVMs,
    images, settings).
  No per-resource ACLs, no per-user limits inside an org. Four roles is
  the v1 set.
- **Platform Admin = global role**, separate from any org membership.
  - The first user to sign in after install is auto-promoted.
  - Platform admins can grant per-org quota overrides, suspend orgs/users,
    view the global audit log, and configure operator-level secrets
    (KMK, recovery signer, Privy app, EVM RPC, SMTP).
  - Org admins cannot self-promote.
- **API tokens** (org-bound, GitHub-PAT shape):
  - Created by an org member; stored as sha256 hashes only.
  - Fields: `id`, `org_id`, `name`, `hash`, `role` (Admin/Deploy/Billing/
    Viewer), `created_by_user`, `created_at`, `expires_at` (30 / 90 / 365
    days or never), `revoked_at`, `last_used_at`.
  - **Token role ≤ creator's role at issue time.** A creator with role X
    can issue tokens with any role ≤ X. Enforced by a single check at
    issue time. **No cascading revocation** — tokens are org-bound, so a
    creator's later demotion does not affect already-issued tokens.
  - Used as `Authorization: Bearer dst_<token>`.
  - No refresh tokens, no JWTs, no rotation flow.
  - Token outlives its creator's tenure; any current admin can revoke any
    of their org's tokens.

## 3. On-chain authority (Safe + Base)

- **Chain:** Base mainnet (chain ID 8453). Operator can configure their own
  EVM-compatible RPC endpoint (e.g., for Sepolia testing or self-hosted
  Base node).
- **Per-org Safe multisig**, deployed lazily on the org's first on-chain
  action.
- **Signers = current admins, automatically.** Adding admin #2 triggers a
  Safe `addOwnerWithThreshold` proposal that admin #1 signs via Privy on
  next page load; the role change in the off-chain DB happens after the
  on-chain confirm. Removing an admin is the same flow in reverse:
  **first the Safe `removeOwner` tx is proposed and signed (Safe quorum),
  then** the off-chain DB row is deleted. UI shows a "Pending removal" state
  for the in-between window.
- **Threshold is always `1-of-N`** where N = current admin count.
  Multi-signature thresholds (M-of-N for M > 1) are deferred.
- **Refusing to remove the last admin** — UI blocks; org must promote
  another admin first.
- **`app_id` collision fix:** `app_id = truncate40(sha256(org_id || compose))`
  computed off-chain by the control plane. **No contract changes** — the
  existing `DstackKms` / `DstackApp` schema accepts whatever `app_id` we
  derive. This costs nothing on-chain and gives perfect cross-org
  separation.
- **Optional platform recovery signer:**
  - Opt-in at install time. If enabled, every new org Safe gets `N+1`
    signers (the org admins plus the platform recovery signer).
  - Recovery key generated at install, sodium-encrypted in the `secrets`
    table.
  - Use is gated by a documented **24-hour on-chain timelock**: the
    operator publishes a recovery intent, then 24h later can execute.
    Off by default.

## 4. Persistence model

- **Postgres** in a sibling Docker container on the same host.
  Connection over **Unix-socket peer auth** (no password). Both Postgres
  and the control plane run as the dstack uid; CVM sandbox uids
  (different per org) have no FS access to the dstack uid's directories
  and no IP route to the control-plane Unix socket.
- **WAL replay enabled by default** for crash recovery; `pg_dump`/`pg_restore`
  documented for backup. **No scheduled backups in v1.**
- **Schema (sketch — synthesis pass produces the columnar version):**
  - `users` — Privy user id, primary wallet, linked wallets, email, name.
  - `organizations` — slug, display name, personal-org flag, deletion
    tombstone.
  - `memberships` — (`user_id`, `org_id`, `role`).
  - `invitations` — pending memberships keyed by email.
  - `api_tokens` — see §2.
  - `hosts` — single-row in v1; carries `host_id` PK that org-scoped
    tables FK to. Forward-compat for multi-host without a v2 forklift.
  - `cvms` — owned by org, references image + compose, runtime state,
    `host_id` FK.
  - `cvm_artifacts` — versioned blobs (`app-compose`, `.encrypted-env`,
    `.user-config`, `.instance_info`) re-materialised to disk before each
    start.
  - `images` — global guest OS image catalog (operator-managed).
  - `org_registry_creds` — see §8 (per-org external-registry credentials).
  - `org_networks` — see §9 (per-org subnet allocations + bridge state).
  - `quotas` — per-org caps + per-org override rows granted by platform
    admins.
  - `audit_log` — see §6.
  - `secrets` — see §5.
  - `safes` — per-org Safe addresses + signer state cache.
- **Big blobs stay on the filesystem:** qcow2 disks, base image files, qemu
  monitor sockets, log streams.
  All metadata, ownership, declarative config, runtime status, audit, and
  secrets go in Postgres.

## 5. Secrets store

- **Sodium-secretbox** encryption per row in a `secrets` table.
- **Master Key (KMK)** = 32 bytes, supplied to the control plane via
  `DSTACK_MASTER_KEY` env var or `/etc/dstack/master.key` (mode 0600,
  owned by the dstack uid).
- **Auto-generation** on first start: control plane writes a fresh KMK
  to the file, then prints a prominent install-time wall-of-text with
  *"BACK THIS UP NOW — losing this key bricks the deployment."* Operator
  must back up themselves; we do not store the KMK anywhere else.
- **Secrets stored:**
  - Privy app secret
  - Privy webhook signing secret
  - Session cookie HMAC key
  - EVM RPC URL/API key (e.g. Alchemy)
  - Platform recovery signer private key (only if §3 opt-in)
  - SMTP credentials (or third-party transactional service creds)
  - Per-org external-registry credentials (see §8)
- Loss of KMK = inability to decrypt secrets. Loss of secrets ≠ loss of
  user data (DB itself stays intact). Operator can re-provision secrets
  and continue, but the recovery-signer capability resets.

## 6. Audit log

- **Always-on hash chain.** Every audit row has `prev_hash` (sha256 of the
  previous row's canonical encoding) and `row_hash` of itself.
  Tamper-evident by construction. Detects any insertion/edit/deletion of
  prior rows.
- **No operator-signed batches in v1.** Skeptic pass push-back accepted —
  signed batches without a public chain-of-custody for the signing key
  are ceremony, not security. Revisit when public batch publication
  (S3 / L2 anchor / Git) lands in v2.
- **Retention:** 90 days hot in DB; cold storage is operator-managed
  (no built-in archive in v1).
- **Export:** CSV/JSON, **platform-admin only** for global; org admins for
  their org's slice.
- **Events captured:** auth (signin/signout/refresh), CVM lifecycle, image
  ops, secret ops, role changes, invitations, settings changes, on-chain
  proposals/signatures, token creation/revocation, quota override grants,
  platform-admin actions, registry-cred changes.
- **Webhooks on audit events: deferred to v2.**

## 7. Quotas

- **Dimensions:** max CVMs, max vCPU, max RAM, max disk, max ingress ports,
  max image storage, deploys-per-hour.
- **Soft + hard caps on every dimension.**
  - Soft = warning emitted to admins (default 80% of hard).
  - Hard = blocks at admission.
- **Enforced at both layers:** control plane refuses at admission, host
  side double-checks at allocation.
- **Org-level only**, no per-user split.
- **Platform-admin override:** `(org_id, dimension, value, granted_by,
  reason, expires_at?)` row. Org admins cannot grant themselves overrides.

## 8. Container images — external private registries (no in-house registry)

The skeptic-pass recommendation is accepted: ship **no in-house OCI
registry** in v1. The 6–10 weeks of registry-server build effort goes to
zero; users keep using whatever registry they already use. The control
plane just holds per-org credentials so CVMs can pull at deploy time.

- **Per-org external registry credentials.** Each org configures one or
  more registry credential rows (`registry_url`, `username`, `password`,
  optional `auth_method` for ECR/GCR/etc.). Credentials are stored in
  the `secrets` table (sodium-secretbox encrypted under KMK).
- **Pull at deploy time.** When a CVM deploys, the control plane resolves
  which registries the compose pulls from, materialises a `.docker/config.json`
  with the matching org credentials, and hands it to the guest agent
  through the existing `.encrypted-env` channel.
- **Supported registries v1:** Docker Hub, GitHub Container Registry,
  GitLab Container Registry, Amazon ECR (token via STS), Google Artifact
  Registry, generic OCI-compliant registries with username/password.
- **No host-side caching layer in v1.** Each CVM pulls images directly
  from upstream. v2 may add a shared pull-through cache if image-pull
  bandwidth becomes a problem.
- **Host-side image catalog (guest OS images) stays as-is** —
  operator-managed, global, not per-org.
- **Per-org private guest OS images: deferred to v2.**

## 9. Networking & ingress

- **Flat URL pattern preserved**: `<cvm-id>[-<port>][s|g].<base>.tld`. Org
  scoping is enforced in the API + UI, not in the URL.
- **Custom domains (BYOD + ACME): deferred to v2**.
- **Per-org Linux uid for QEMU.** Cheap defense-in-depth on top of TEE
  isolation. Extends today's single-uid sandbox to per-org sandboxes.
- **Per-org Linux bridge for bridge-mode CVMs.** Confirmed by F3
  investigation (`12-f3-vlan-isolation.md`) and independently by Codex
  GPT-5.5 high-reasoning second opinion (`13-codex-bridge-opinion.md`).
  Today's shared `virbr0` lets cross-org CVMs ARP-scan and reach each
  other on L2; gateway WG only protects ingress, not bridge-side traffic.
  - **One Linux bridge per org**, name e.g. `dstack-org-<id>`, lazily
    created on the first bridge-mode CVM in that org.
  - **IPAM:** /24 per org allocated from an operator-configurable
    supernet (RFC1918), reserved in Postgres `org_networks` with a
    uniqueness constraint. /24 is the default per-org prefix; supernet
    is operator-tunable for installs needing more org count.
  - **DHCP:** one dnsmasq process per bridge for failure isolation and
    config simplicity. Reuses the existing `dhcp-notify.sh` →
    `ReportDhcpLease` plumbing, with bridge/org context added to the
    lease notification payload (Codex flagged today's HTTP channel as
    cross-org-spoofable; the rewrite moves it to a private local channel
    and includes bridge identity in the message).
  - **NAT/forwarding:** nftables sets/maps keyed by bridge name (not N
    independent rule blocks). Default policy: org bridge ↔ uplink +
    established return; **drop forwarding between sibling org bridges**.
  - **GC/reconciliation:** Postgres is desired state, host network is
    reconciled on startup (bridge links, dnsmasq processes, nftables
    sets, `/etc/qemu/bridge.conf` allowlist). Bridges are deleted only
    after zero bridge-mode CVMs + zero enslaved TAPs + a short grace
    period.
  - **Drop VLAN tagging entirely.** Per-port VLAN state on
    bridge-helper-created TAPs is fragile (post-start hooks, ordering,
    stale-port failure modes). VLAN-aware single bridge is a v2
    optimization if a single host commonly carries thousands of
    bridge-mode orgs.
- **`mode = "user"` (SLIRP) is unaffected** — per-VM SLIRP isolates by
  construction. Bridge-mode is opt-in per CVM; user-mode CVMs do not
  exercise the per-org bridge plumbing.
- **Fallback if per-org bridge orchestration overruns v1 budget:**
  ship `mode = "user"` only for v1 (today's vmm.toml default per
  `vmm/vmm.toml:86-96`) and defer multi-tenant bridge mode entirely.
  Codex flagged this as the cleanest fallback.

## 10. API design

- **REST/JSON externally**, **OpenAPI spec generated** from server
  definitions.
- **Stripe-style header versioning:** stable path `/v1/orgs/:slug/...`;
  clients pin behaviour with `Dstack-Api-Version: 2026-05-01` header.
  12-month deprecation policy on date-versioned behaviour.
- **Cursor-based pagination**, default 50, max 200.
- **Rate limits:** per token + per IP at the gateway.
- **Idempotency keys** (`Idempotency-Key` header) on every create operation.
- **Internal control-plane↔host RPC** is in-process (single binary).

## 11. Observability

- **Prometheus `/metrics`** behind platform-admin auth.
- **Structured JSON logs** by default, with `request_id`, `org_id`,
  `user_id`, `node_id`.
- **OpenTelemetry tracing** off by default, **auto-on when
  `OTEL_EXPORTER_OTLP_ENDPOINT` is set**, configurable.
- **Per-tenant resource accounting:** the host-side measurement loop
  pushes 30-second aggregates into a rolling table (`resource_usage`)
  for billing primitives; control plane reads it for quota dashboards.
- **Bridge-mode network observability:** monitor per-org bridge count,
  dnsmasq process health, DHCP lease success rate, nftables rule/set
  size, conntrack count + insert failures, MTU/path-MTU incidents,
  orphaned-bridge count, and orgs holding bridge allocations with zero
  active CVMs (signals GC policy needs tuning).

## 12. Failure modes

- **Control plane down:** existing CVMs keep running (QEMU survives
  parent-process death; encrypted disks stay mounted). New deploys,
  updates, and config changes block until restart.
- **Host node is the box:** there's no node-down semantics — if the box is
  down, everything is down.
- **KMS down:** users can read existing CVM state; deploys block.
- **External registry down:** a deploy that pulls from that registry
  fails fast with a clear error referencing the registry URL; existing
  running CVMs unaffected.
- **Orphaned bridge after host crash:** startup reconciler classifies
  bridges as desired / stale-empty / stale-in-use; deletes only stale-empty
  after grace; never deletes a bridge with enslaved TAPs.

## 13. Migration / release

- **Hard cut, fresh installs only.**
- **No backwards compat** with `vmm-cli.py`, the existing prpc surface, or
  existing on-chain registrations.
- **No import path** for existing CVMs in v1 (skeptic push-back declined —
  user wants greenfield clean).
- The legacy `dstack-vmm` binary stays in the tree for legacy installs but
  is not part of the new install path.
- **Telemetry off by default**, opt-in only.
- **License: Apache-2.0** (unchanged).

## 14. Org lifecycle details

- **Personal org auto-created on first signup.** Visible in switcher,
  renamable, deletable (with confirm) — GitHub-style.
- **One user → many orgs.**
- **Org slug:** globally unique, 2–32 chars, `[a-z0-9-]`, case-insensitive.
- **Email invites:** create pending memberships keyed by email; recipient
  claims on first sign-in via Privy.
- **Org deletion:** **block if any CVMs exist** — require explicit teardown
  first. Soft-delete with 30-day tombstone for audit. Bridge GC reclaims
  the org's `/24` after tombstone period.
- **No ownership transfer flow needed** — every admin is an owner.

## 15. Explicitly deferred to later (v2+)

| Item | Notes |
|---|---|
| Multi-host control plane | Single-host is v1 by F1; schema is host_id-ready |
| HA control plane (active-active replicas) | Single process is v1 |
| TEE-protected control plane (control plane inside CVM) | Bootstrap chicken-egg with KMS; revisit later |
| Custom domains (BYOD + ACME) | I2 — operator domain only in v1 |
| Per-org custom guest OS images | J2 — global catalog only in v1 |
| In-house OCI registry server | External-private-registry creds only in v1 |
| Pull-through image cache | v1 pulls upstream every time |
| Webhooks on audit events / CVM lifecycle | H5 — pull-only for v1 |
| Operator-signed audit batches | Hash chain only in v1; signed batches arrive with public publication in v2 |
| Public publication of audit log | Local export only in v1 |
| Scheduled DB backups | Manual `pg_dump`/`pg_restore` in v1 |
| Privy account export / "claim my wallet" | B3 |
| Per-resource ACLs | Org-level roles only in v1 |
| Per-user limits inside an org | Org-level quotas only in v1 |
| Live CVM migration | Encrypted disks tied to host KP |
| Reservation-based GPU allocation | First-come-first-served in v1 |
| M-of-N Safe thresholds where M > 1 | 1-of-N only in v1 |
| Finer per-resource API token scopes | Role-only scopes in v1 |
| CSV/JSON archive automation | Manual export in v1 |
| Public-key signature verification of KMS responses in browser | Currently absent — flagged in report 02 — design for v1 if cheap, else v1.5 |
| VLAN-aware single bridge for >1000 orgs/host | Per-org Linux bridge in v1; revisit when scale demands it |
| Cascading API-token revocation on creator demotion | Tokens are org-bound in v1; not affected by creator role changes |
| Import path for legacy dstack-vmm CVMs | Greenfield only in v1 (user override of skeptic recommendation) |

## 16. Open implementation questions for the synthesis pass

The following are *implementation* questions, not *design* questions —
the synthesis agent decides them:

- Choice of Rust web framework (continue with Rocket vs move to Axum).
  Synthesis recommended Axum 0.7.
- Choice of Postgres client + migration tool (sqlx vs diesel vs ...).
  Synthesis recommended sqlx 0.8 + sqlx-cli.
- Choice of EVM client + Safe ABI bindings. Synthesis recommended alloy
  0.8 with hand-rolled Safe v1.4 calldata.
- Whether to keep `prpc` for the host-side guest-agent boundary and only
  go REST/JSON for user-facing API (recommended), or unify on REST/JSON.

## 17. Sign-off

Decisions above are locked as of 2026-05-06 after one round of skeptic
revisions (8 of 10 skeptic push-backs incorporated; 2 declined — Privy
and Postgres) and one Codex second-opinion confirming the per-org
Linux bridge call. The synthesis pass should re-emit `99-final-plan.md`
based on this revision.
