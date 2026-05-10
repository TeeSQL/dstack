# dstack-vmm Migration & Compatibility Strategy

**Prepared:** 2026-05-05
**Scope:** how we get from today's single-tenant `dstack-vmm` to a multi-tenant
production control plane without breaking existing operators or stranding their
on-chain state.
**Companion docs:** `00-overview.md`, `01-vmm-api-surface.md`.

---

## 0. Starting state — what we are migrating

Today, every existing dstack deployment is shaped like this:

- One Linux user (the **operator**) runs `dstack-vmm` with a single
  `vmm.toml`. Auth is opt-in HTTP API tokens, off by default
  (`vmm/vmm.toml:126-128`, `vmm/src/main.rs:208`).
- The web UI is served on the same listener (default `unix:./vmm.sock`,
  often forwarded to `0.0.0.0:9080`); console v1 is mounted at `/`,
  v0 at `/v0`, log streaming at `/logs`
  (`vmm/src/main_routes.rs:45-74`, `:107-183`, `:185-187`).
- All CVMs live as sibling directories under `config.run_path`
  (default `~/.dstack-vmm/vm`, `vmm/src/config.rs:527-528`). Each CVM
  has a workdir with `vm-manifest.json`, `vm-state.json`, a `shared/`
  bundle, qemu pid + serial logs, and an optional `.removing` marker
  (`vmm/src/app/qemu.rs:966-1123`).
- VMs are reloaded from disk at startup (`vmm/src/app.rs:545-621`).
- `vmm-cli.py` discovers running VMM processes by reading
  `$XDG_RUNTIME_DIR/dstack-vmm/*.json` files written by
  `discovery::DiscoveryRegistration::register`
  (`vmm/src/discovery.rs:56-97`, `vmm/src/vmm-cli.py:39-105`).
- Some operators have already run `npx hardhat kms:create-app` /
  `app:add-hash` (`kms/auth-eth/hardhat.config.ts:346`, `:431`), giving
  them `DstackKms` registrations and per-app `DstackApp` proxies whose
  upgrade authority is an EOA they control (`kms/auth-eth/contracts/
  DstackApp.sol:153,158,170,182,220`).
- The vmm prpc service is consumed by **only** the web UI, `vmm-cli.py`,
  and ad-hoc operator scripts. Gateway, KMS, and guest-agent do **not**
  call the vmm prpc surface (verified: only `vmm/`, `tests/docs`, and
  `sdk/go` reference the `Vmm.*` / `Teepod.*` namespace; gateway/KMS only
  share *names* like `GetMeta` for their own services). This dramatically
  scopes BC discussion in §3.

This is the surface area we have to keep working — or visibly retire — as we
introduce orgs, users, RBAC, multi-host scheduling, and audit logging.

---

## 1. Migration strategies — pros / cons

### (a) Hard cut

> Ship the new control plane as a separate process. Existing operators
> spin down their old `dstack-vmm`, deploy the new control plane, and
> redeploy their CVMs. On-chain registrations still exist — operators
> import them by pasting their app-id / `DstackApp` address into the new UI.

**Pros**

- Cleanest internal model: no legacy code paths in the new control plane,
  no schema-version branches, no "node-mode" stub for the old vmm.
- Smallest test matrix — there is exactly one production shape.
- Forces every operator through a known migration moment, so the
  audit-log / multi-tenant invariants are correct from day one.
- The on-disk layout in `~/.dstack-vmm/vm/<uuid>/` is already
  self-describing (`vm-manifest.json` carries app-id, image, ports, gpu,
  kms_urls, gateway_urls, networking — `vmm/src/app.rs:48-73`), so an
  *import* tool can re-adopt CVMs without a real "compatibility shim";
  see §5.

**Cons**

- Existing operators have to redeploy their CVMs. Even if state is
  preserved on disk, the moment of cutover is downtime per CVM.
- KMS-backed apps are tied to a `DstackApp` proxy whose owner is an EOA;
  if we change the org→wallet model later (overview Q2) we will have to
  re-import again, so a hard cut once doesn't actually solve the
  identity-bridging question.
- Operators who self-host *for themselves only* perceive this as
  pure cost — they get features they don't immediately want.

**Sweet spot:** small operator population, willingness to accept a one-time
migration, hostility to maintaining legacy code in the rewrite.

### (b) Coexistence — old vmm becomes a "node"

> Ship a separate control-plane process. Treat each existing
> `dstack-vmm` host as a registered **node** that the control plane talks
> to over the existing prpc surface. Adopt all CVMs on a node into a
> default org (the migrating operator's personal org), one node at a time.

**Pros**

- Zero CVM downtime: control plane just *talks to* the existing vmm.
- Lets us land multi-host scheduling primitives early — the control
  plane is multi-host-shaped from day one because it must be.
- Isolates the messy parts (auth/orgs/audit) into a new codebase
  without rewriting the QEMU lifecycle code that already works.
- The vmm prpc surface is small and stable enough to be a node-driver
  protocol: `Status / GetInfo / CreateVm / StartVm / StopVm / RemoveVm /
  UpdateVm / ShutdownVm / Get*Hash / Get*EncryptPubKey / List*Images /
  *Vm`, plus log streaming via `/logs`
  (`vmm/rpc/proto/vmm_rpc.proto:300-360`, `vmm/src/main_routes.rs:107`).

**Cons**

- The old vmm has **no caller identity** at the prpc layer
  (`vmm/src/main_service.rs:743-751` — `construct` ignores caller),
  and per-VM auth tokens don't exist. The control plane has to be the
  *only* client the old vmm trusts; we enforce that by:
  (1) turning on `auth.enabled = true` and rotating tokens
  (`vmm/vmm.toml:126-128`, `vmm/src/main.rs:208`), and
  (2) firewalling the old vmm's TCP port so only the control plane
  can reach it. Direct CLI / UI bypass of the control plane is now
  a security concern, not a feature.
- Host-API plumbing **stays where it is**: `host_api` is vsock-only by
  design — `HostApiConfig::validate` rejects anything that doesn't start
  with `vsock:` (`vmm/src/config.rs:454-466`), and `run_host_api` binds
  via `VsockListener::bind_rocket` (`vmm/src/main.rs:111-130`). So
  guest↔host integration is still terminated on the same host as the
  CVM, regardless of where the control plane runs. That's correct
  (vsock cannot be remoted) but it means the control plane cannot
  proxy host-API itself; it must trust the node.
- Two binaries, two release cadences, two upgrade stories. The "node"
  protocol becomes a public surface we have to version.
- Quotas are unenforceable on the old vmm — `GetMeta` reports global
  caps (`vmm/src/main_service.rs:521-554`) and CreateVm has no quota
  hook. The control plane has to enforce quotas *itself* before
  forwarding to a node.

**Sweet spot:** existing fleet has many CVMs; downtime cost dominates
internal-complexity cost; multi-host is in scope.

### (c) Gradual evolution — multi-tenancy bolted into the existing vmm

> No new process. Add a SQLite/Postgres state file next to `vmm.toml`,
> add user/org tables, gate every prpc method behind an auth middleware
> that resolves a caller into a `(user, org)` pair, tag every VM with
> an `org_id` column, then ship.

**Pros**

- One binary, one config file, one deployment topology — *exactly* the
  shape existing operators already run. Single-host-only operators get
  multi-tenancy with zero re-architecture.
- Discovery / `vmm ls` UX in `vmm-cli.py` keeps working unchanged
  (`vmm/src/vmm-cli.py:79-105`, `:177-256`).
- We keep the existing CVM lifecycle code, port-forwarding, supervisor
  integration, and disk layout — battle-tested
  (`vmm/src/app.rs:221-621`).
- Incremental: each capability (auth → orgs → quotas → audit) can land
  in its own minor release without a flag day.

**Cons**

- The `App` struct's `state: Arc<Mutex<AppState>>` holds *all* VMs in
  a single `HashMap` (`vmm/src/app.rs:128-136`, `:151-166`); retrofitting
  org-scoped reads means touching every `state.get(id)` /
  `state.vms.iter()` path including the DHCP lease lookup
  (`vmm/src/app.rs:421-447`), the supervisor reconciliation
  (`vmm/src/app.rs:545-621`), and the port-forward bookkeeping
  (`vmm/src/app.rs:454-543`). Easy to miss one and leak cross-org info.
- The CID pool, image pool, and GPU inventory are physically global on
  the host (`vmm/src/app.rs:151-166`, `vmm/src/main_service.rs:556-563`).
  "Multi-tenant" here means *fair-share inside one host*, not isolation.
  Some operators (and most regulatory stories) want a sharper boundary
  than that, which requires either §1(b)-style nodes or per-org cgroups.
- Multi-host scheduling is essentially out of reach without re-introducing
  a control-plane shape later. So we may end up doing both this and
  some flavour of (b) anyway.
- The existing `rocket_apitoken::ApiToken` is *flat* — a list of tokens,
  no association with a user (`vmm/src/main.rs:208`). We have to layer
  a real auth system *under* it (or replace it) without breaking
  current `--auth-user` / `--auth-password` flag callers
  (`vmm/src/vmm-cli.py:1530-1539`).
- The web UI is embedded HTML at compile time (`vmm/src/main_routes.rs:19`,
  `:46-48`); making it tenant-aware means rebuilding it as a real SPA
  with a session cookie or token, which is most of the effort of a
  rewrite anyway.

**Sweet spot:** the operator population genuinely is "one host, a few
people sharing it" and will stay that way. Bad fit if the rewrite is
also meant to enable multi-host scheduling.

---

## 2. `vmm-cli.py` compatibility

Today the CLI:

- Reads `~/.dstack-vmm/config.json` for `url`, `auth_user`, `auth_password`
  and `active_vmm` (`vmm/src/vmm-cli.py:33-76`, `:1515-1539`).
- Auto-discovers running vmms via the registration files and either
  uses the singleton, the `active_vmm`, or `localhost:8080`
  (`vmm/src/vmm-cli.py:79-145`).
- Talks to `/prpc/Vmm.*` over HTTP/Unix-socket with optional Basic
  auth.

Recommendation, in priority order:

1. **Keep the binary working against single-host vmm forever**, even
   after the new control plane ships. It is the operator's
   "out of band" diagnostic tool when the control plane itself is sick.
   Concretely: keep the discovery code (`vmm/src/discovery.rs`,
   `vmm/src/vmm-cli.py:39-105`) and the existing `--url / --auth-user /
   --auth-password` flags untouched; document them as the
   "node-direct mode."
2. **Add a parallel control-plane mode**: `--control-plane URL`
   (or env `DSTACK_CONTROL_PLANE_URL`) plus token auth
   (`--token` / `DSTACK_TOKEN`). When present, every command is rerouted
   to the control plane, which fans out to nodes by VM-id ownership.
   This is the recommended day-to-day mode under strategies (a) and (b).
3. **Do not deprecate the existing CLI in favour of a new one.** Most
   `subparsers.add_parser` blocks (`vmm/src/vmm-cli.py:1541-1799`+)
   are direct mappings to prpc methods that the control plane will
   also expose. Forking the binary doubles the surface area for almost
   no gain. Instead, ship the same `vmm-cli.py` binary with a
   `--control-plane` flag and the same subcommand grammar. Where a
   control-plane-only command makes sense (e.g. `org`, `user invite`,
   `quota set`), add it as a subparser that errors out cleanly when no
   control-plane URL is set.
4. **Defer SIWE / OAuth in the CLI.** First release: bearer token only,
   minted by the control-plane web UI under a "Personal Access Tokens"
   screen. SIWE / OIDC happen browser-side; the CLI just consumes
   the token.

If we go with strategy (c), only step (1) applies; the CLI's
`--auth-user` / `--auth-password` already works against
`rocket_apitoken` and the only diff is adding `--token` (alias for
`--auth-password` with `--auth-user=__token__`).

---

## 3. prpc / wire-protocol compatibility

The vmm prpc surface today is consumed by:

- `vmm-cli.py` (operator tool).
- The embedded web UI (HTML/JS shipped inside the binary;
  `vmm/src/main_routes.rs:19`).
- Whatever ad-hoc scripts each operator wrote.

It is **not** consumed by gateway, kms, or guest-agent. (Verified by
grep: gateway/kms/guest-agent each define their own `GetMetaResponse`
and never import `dstack_vmm_rpc` — the vmm-only crate is referenced
only in `vmm/src/...`, `tests/docs`, and `sdk/go`. The Gateway / KMS
references that look similar are their own `GetMeta` services.)

That changes what we owe each consumer:

| Consumer | Stability promise | Versioning |
|---|---|---|
| Old `vmm-cli.py` against an old `dstack-vmm` | Keep the existing `Teepod.` / `Vmm.*` proto frozen for 2 minor releases after the rewrite ships, then mark deprecated. | None needed — it's the same binary. |
| Old `vmm-cli.py` against the new control plane | Speak the same `Vmm.*` methods at `/prpc/` for the subset that maps cleanly, and add control-plane-only services at `/prpc/v2/`. | Mount both prefixes (`/prpc` for v1, `/prpc/v2` for v2). |
| Web UI | The embedded UI dies with the old binary; the new control plane ships its own SPA. No wire compat owed. | n/a |
| External scripts | "Best effort, on the v1 path." Document the deprecation window in release notes. | Tag the old proto with `option deprecated = true` per RPC once a v2 equivalent exists. |
| Host API (`vsock://`, `host_api.address` validated to start with `vsock:` — `vmm/src/config.rs:454-466`) | **Fully stable**, never versioned. The guest VM image and the vmm instance on the same host are tightly coupled by definition. | n/a — we keep `host-api/proto/host_api.proto` and `guest-api/proto/guest_api.proto` exactly as is. |

Practical rules for v2:

- **New methods only** in `dstack_control_rpc` package, mounted at
  `/prpc/v2/` (parallel to the existing `/prpc/` mount in
  `vmm/src/main.rs:82-88`). No silent semantic changes.
- **`tenant_id` / `org_id` is implicit from the auth token**, not a
  request field. (Otherwise honest clients accidentally cross orgs.)
  Wrong-org IDs return 404, never 403, so we don't leak existence.
- **Anything multi-host** (node addressing, scheduling hints, fleet-wide
  status) is v2-only. The v1 surface stays single-host-shaped.
- **`GetMeta` semantics**: in v1 it returns global host limits
  (`vmm/src/main_service.rs:521-554`). In v2 it returns
  *org-scoped quotas*. Do not overload the v1 message — different
  endpoints, even if the wire types look identical.
- **`ReportDhcpLease`** (`vmm/src/main_service.rs:579-582`) is an
  internal hook called by `dnsmasq --dhcp-script`. It already doesn't
  belong on the public API surface. Move it to a private socket in v2;
  remove it from the v1 proto in the next major.

---

## 4. On-chain implications

Existing on-chain state, per operator:

- A `DstackKms` registration whose `registeredApps[appId] = true`
  was set by `registerApp` or `deployAndRegisterApp`
  (`kms/auth-eth/contracts/DstackKms.sol:130-167`).
- A per-app `DstackApp` UUPS proxy whose `owner()` is the operator's
  EOA (`kms/auth-eth/contracts/DstackApp.sol:110-117`), and whose
  `addComposeHash` / `addDevice` / `setAllowAnyDevice` /
  `setRequireTcbUpToDate` / `disableUpgrades` are all `onlyOwner`
  (`DstackApp.sol:158,164,170,176,182,188,220`).

What this means for orgs:

- **The control plane never holds the EOA private key** unless the
  operator explicitly opts in. The natural mapping is
  `org_id → set_of_owner_addresses`, where the org *records* which
  on-chain apps it considers part of itself, but `addComposeHash` and
  friends still go through the owner wallet (e.g. via WalletConnect /
  SIWE-signed transactions from the UI). This matches the existing
  Hardhat tasks `app:add-hash` (`kms/auth-eth/hardhat.config.ts:431`),
  `app:add-device` (`:451`).
- **Multi-user orgs need a multisig.** A single EOA cannot mean
  "the org acts as one." For orgs with >1 owner, we recommend Safe (
  Gnosis Safe). The control plane's role is just to surface the
  pending transaction in the UI; the Safe contract enforces the
  threshold. New apps deployed via `kms:create-app` should accept a
  `--owner <safe-address>` flag (already supported — `initialOwner`
  is the only required arg in `deployAndRegisterApp`,
  `DstackKms.sol:144-167`); existing apps can `transferOwnership` to
  a Safe.
- **For migration** of existing apps: nothing on-chain has to change.
  The control plane just needs an "import on-chain app" flow:
  user pastes `(chain_id, app_id_address)`, signs a message proving
  control of the current `owner()`, and the org adds the app to its
  registry. No on-chain transaction is required to *import*; only
  ongoing operations like `addComposeHash` need a tx (already true
  today).
- **Open question (overview Q2)**: do we want the control plane to
  *custody* the owner key for hosted-only orgs, so users without a
  wallet still get the upgrade story? Default proposal: **no** for
  v1 — hosted-only orgs simply can't upgrade compose hashes without
  bringing a wallet. Revisit in a later milestone if needed.

---

## 5. Data migration — adopting existing CVMs into a default org

The on-disk layout already carries everything we need.
`Manifest` (`vmm/src/app.rs:48-73`) has `id`, `name`, `app_id`,
resources, `port_map`, `kms_urls`, `gateway_urls`, `networking`,
`gpus`. `vm-state.json` has the running flag
(`vmm/src/app/qemu.rs:1013-1028`). The shared bundle has `app-compose.json`
+ `encrypted-env` + `instance-info`
(`vmm/src/app/qemu.rs:1034-1055`).

So adoption is mostly *bookkeeping*: tag every existing workdir with an
`org_id` and an `owner_user_id`. Two implementation shapes,
depending on chosen strategy.

### 5.1 Strategy (a) or (b) — control plane does the import

One-shot CLI shipped as `vmm-cli.py vmm import` (or
`dstack-control-plane import-node`):

1. **Discover** running vmms via the existing discovery files:
   `_get_discovery_dirs()` already enumerates
   `/run/user/<uid>/dstack-vmm/*.json` (`vmm/src/vmm-cli.py:41-59`,
   produced by `discovery::DiscoveryRegistration::register`,
   `vmm/src/discovery.rs:56-97`). Pick the target.
2. **List CVMs** via `Vmm.Status` (`vmm/src/main_service.rs:353-355`,
   `vmm/rpc/proto/vmm_rpc.proto:321`) on that vmm using its existing
   token auth. We do NOT touch disk directly — Status returns the
   authoritative `VmInfo` (manifest + status + ports + image_version
   + events) for every running and stopped CVM.
3. **For each VmInfo**, insert a row in the control plane's CVM
   table with:
   - `vm_id` = manifest.id
   - `node_id` = the discovered vmm's id (from discovery file)
   - `org_id` = the migrating operator's personal org (created
     server-side at first login)
   - `owner_user_id` = same operator
   - `app_id` = manifest.app_id (already 40-hex, stable —
     `vmm/src/main_service.rs:46-55`)
   - `state_snapshot` = current `VmInfo.status`,
     `boot_progress`, `image_version`
4. **Run a verification pass**: call `Vmm.GetInfo` per VM, compare
   `manifest.app_id` against the on-chain app registry, optionally
   prompt the user to bind each `app_id` to a `DstackApp` proxy
   address.
5. **Mark the node as adopted** in the control plane's nodes table.
   From this point on, the CLI/UI sends operations through the
   control plane (token-gated) which forwards to the node.

The on-disk layout is untouched. If the operator rolls back the
control plane, the old vmm and old CLI keep working against the same
files. This is what makes (a)/(b) low-risk.

### 5.2 Strategy (c) — vmm gains org awareness in place

Single binary, one DB next to `vmm.toml`. On boot:

1. **DB bootstrap.** First run with the new schema creates the
   `default` org (id `org_default`) and a single user from
   `auth.tokens[0]` (`vmm/vmm.toml:126-128`) labelled "imported".
2. **Walk `config.run_path`** (the same `reload_vms` loop —
   `vmm/src/app.rs:545-621`). For every workdir, insert a `(vm_id,
   org_id=org_default, owner_user_id=imported)` row. The existing
   reload code already iterates each subdir, reads its manifest,
   rebuilds memory state, and clears stale CIDs — that's the right
   moment to also persist the bookkeeping rows.
3. **`vm-manifest.json` schema is already forward-compatible** —
   `Manifest` derives `Serialize/Deserialize` (`vmm/src/app.rs:48-73`)
   and uses `#[serde(default)]` for newer fields. Adding an optional
   `org_id` field with `#[serde(default)]` and a sentinel
   `"org_default"` keeps the disk format readable by old binaries
   (forward-compat: pre-rewrite vmm just ignores the field).

In both shapes, the migration script is **idempotent** (re-running
just no-ops on already-imported VMs), and the operator can roll back
by unsetting the new flags in `vmm.toml`.

---

## 6. Concrete migration steps in order

Effort key: **s** ≤ 1 week, **m** 1–4 weeks, **l** > 4 weeks for one
focused engineer. "Independent" = can ship before the rewrite as part
of normal vmm releases. "Coupled" = needs the control plane to exist.

### Phase 0 — preparation (independent, ships into existing vmm now)

| # | Step | Effort | Touch points |
|---|---|---|---|
| 0.1 | Add per-VM `org_id` and `owner_user_id` fields to `Manifest` with `#[serde(default)]`; default `org_default` / `owner_imported`. New code reads/writes the field; old binaries ignore it on read. | s | `vmm/src/app.rs:48-73`, `vmm/src/app/qemu.rs:1006-1011` |
| 0.2 | Document the `Vmm.*` prpc surface as **frozen** for the deprecation window. Move `ReportDhcpLease` off the public proto onto a private socket. | s | `vmm/rpc/proto/vmm_rpc.proto:300-360`, `vmm/src/main_service.rs:579-582` |
| 0.3 | Add structured audit logging to every method on `RpcHandler` (record `caller_token_id` from `rocket_apitoken`, vm_id, action, result). Useful for both single-tenant ops and as the substrate for org-scoped audit later. | s | `vmm/src/main_service.rs:743-751` |
| 0.4 | Token auth metadata: extend `AuthConfig` from `tokens: Vec<String>` to `tokens: Vec<{token, label, scopes?}>` (kept backward-compatible — `tokens: ["abc"]` still parses). Lets us mint "personal access tokens" later without a schema migration. | s | `vmm/src/config.rs:287-293`, `vmm/vmm.toml:126-128`, `vmm/src/main.rs:208` |
| 0.5 | Make discovery file include a `node_id` independent of the random per-process UUID, so a vmm that restarts keeps the same identity. (Today the file uses `Uuid::new_v4()` per launch — `vmm/src/discovery.rs:69-71`.) | s | `vmm/src/discovery.rs:18-97` |
| 0.6 | Add a read-only `Vmm.ExportInventory` rpc returning the list of `(vm_id, manifest, status, image_version, events)` in a stable JSON shape. Used by the importer in §5.1; also handy for backups. | s | `vmm/rpc/proto/vmm_rpc.proto:300`, `vmm/src/main_service.rs` |

Phase 0 turns the existing single-tenant vmm into something the future
control plane can adopt cleanly. It is shippable without committing
to a strategy.

### Phase 1 — core control plane skeleton (coupled, only needed for (a)/(b))

| # | Step | Effort | Notes |
|---|---|---|---|
| 1.1 | Stand up the new `dstack-control-plane` binary (HTTP + DB). One node, one default org, no scheduler. Auth: bearer tokens, signup/login by email+password and SIWE. | l | New crate. |
| 1.2 | Implement node registration: a node (= existing `dstack-vmm`) checks in with its `node_id`, listen address, and a shared secret. Control plane stores it. | m | Adds a `nodes` table; reuses Phase-0 stable `node_id` from `vmm/src/discovery.rs`. |
| 1.3 | Implement `node-driver` shim that translates control-plane CVM operations into `Vmm.*` prpc against a node. Initial verbs: list, create, start, stop, remove, update, status, logs (proxied SSE from `/logs`, `vmm/src/main_routes.rs:107-183`). | m | No vmm changes needed. |
| 1.4 | Implement org / user / membership tables and RBAC. Personal org auto-created on signup. | m | |
| 1.5 | Implement quota enforcement at the control plane (vCPU, memory, vm-count per org). The existing `GetMeta` (`vmm/src/main_service.rs:521-554`) is *advisory only* — control plane decides admission. | s | |
| 1.6 | Implement `import-node` (§5.1). | s | |
| 1.7 | Implement web UI (SPA): login, org switcher, CVM list, deploy form, logs. | l | |

Phases 0 and 1 can ship independently; an operator who hasn't yet
upgraded their vmm beyond Phase 0 still works against the new control
plane (the control plane just doesn't get audit logs in the structured
form yet).

### Phase 2 — multi-host scheduling (coupled, only for full SaaS shape)

| # | Step | Effort | Notes |
|---|---|---|---|
| 2.1 | Add a `placement` decision in CreateVm: control plane picks the node based on free vCPU / memory / GPU / network mode. | m | Uses Phase 0.6 inventory + per-node `GetMeta`. |
| 2.2 | Per-node networking config flows from control plane: pick a node whose `networking.mode` (`vmm/src/config.rs:387-427`) and port range (`vmm/src/config.rs:122-127`) accept the request. | s | |
| 2.3 | Optional: live migration story. Out of scope for this doc. | l | |

### Phase 3 — on-chain bridging (coupled)

| # | Step | Effort | Notes |
|---|---|---|---|
| 3.1 | "Import on-chain app" flow: paste app-id address, sign a message proving `owner()` (`DstackApp.sol:110-117`), record `(org_id, app_id_address)`. | s | |
| 3.2 | Surface pending owner-only operations in the UI (`addComposeHash`, `addDevice`, `setAllowAnyDevice`, `setRequireTcbUpToDate`, `disableUpgrades` — `DstackApp.sol:158-220`) as transactions to be signed by the user's wallet (or Safe). | m | |
| 3.3 | Optional Safe (Gnosis) integration for multi-owner orgs. | m | |

### Phase 4 — deprecation (independent, ships into the old vmm)

| # | Step | Effort | Notes |
|---|---|---|---|
| 4.1 | Mark v1 prpc methods that have a v2 equivalent with `option deprecated = true`. Add a startup banner pointing operators at the control plane. | s | `vmm/rpc/proto/vmm_rpc.proto:300-360` |
| 4.2 | Two minor versions later: stop building the embedded web UI in the old vmm — leave just `/api-docs` and the prpc endpoints. (The CLI continues to work.) | s | `vmm/src/main_routes.rs:185-187`, `vmm/src/main.rs:82-88` |

### Independence summary

- **Ship now, before any strategy decision:** Phase 0 (0.1 - 0.6).
  All small, all backward-compatible. Each step makes the rewrite
  cheaper without forcing it.
- **Ship after strategy decision (a) or (b):** Phase 1.
- **Ship later:** Phases 2, 3, 4.

---

## 7. Recommendation

**Pick strategy (b) — coexistence with old vmm as a node — built on top
of Phase 0 in the existing vmm.**

Reasoning:

1. **It is the only strategy that makes the multi-host story
   honest from day one.** The overview's question 1 explicitly flags
   multi-host as "dramatically changes scheduler / state design";
   committing to the node abstraction now means we don't have to
   re-architect later. Strategy (c) defers this question and likely
   has to re-pay it.
2. **The existing vmm prpc surface is small and is already the right
   shape for a node driver** (~25 stable methods,
   `vmm/rpc/proto/vmm_rpc.proto:300-360`). The CVM lifecycle code
   in `app.rs` / `app/qemu.rs` is battle-tested and the disk layout
   carries enough metadata to be re-adopted without a separate
   migration database (§5.1). Throwing this code away — strategy (a)
   — buys us nothing.
3. **It contains the rewrite blast radius.** Auth, RBAC, audit, orgs,
   quotas, scheduler, and the SPA all live in a fresh codebase. The
   old vmm only needs the small Phase-0 changes (org_id field,
   stable node_id, ExportInventory rpc, audit log shape, token
   metadata). That makes the rewrite testable in isolation and
   means an operator can roll back to "just the old vmm" if the
   control plane has a regression.
4. **No CVM downtime during migration.** Once the old vmm is
   registered as a node, every running CVM is *immediately* visible
   in the new UI without restarting QEMU. Operators can take their
   time moving day-to-day operations from `vmm-cli.py --url ...` to
   `vmm-cli.py --control-plane ...`.
5. **On-chain stays exactly where it should.** Operator EOAs keep
   their `DstackApp` ownership; the control plane is bookkeeping plus
   a UI for transactions the user already had to sign anyway. We are
   not entangling the rewrite with the custody / Safe-integration
   discussion (overview Q2), which is a separate, slower decision.
6. **Strategy (c) is genuinely tempting for the smallest deployments,
   but Phase 0 captures everything (c) would have shipped first
   anyway** (auth-token labels, audit logs, manifest org_id field).
   So Phase 0 keeps single-host operators happy while we build the
   control plane in parallel; we never have to commit to (c) to get
   value out.

The deprecation window for the v1 prpc + the embedded web UI is two
minor releases after the control plane GAs, per Phase 4.

---

## Appendix — file-line index of migration touch-points

| Concern | File:line |
|---|---|
| Listener address / port (single global listener) | `vmm/src/main.rs:77-109`, `vmm/src/config.rs:323-359` |
| Token-only auth (no user concept) | `vmm/src/main.rs:208`, `vmm/src/config.rs:287-293`, `vmm/vmm.toml:126-128`, `vmm/src/main_routes.rs:107-109` |
| prpc handler ignores caller identity | `vmm/src/main_service.rs:743-751` |
| All-VMs-global state | `vmm/src/app.rs:128-166`, `:545-621` |
| CVM workdir layout (manifest, state, shared, removing-marker) | `vmm/src/app/qemu.rs:966-1123`, `:1125-1137` |
| Disk reload at startup | `vmm/src/app.rs:545-621` |
| Manifest schema (already serde-stable + `#[serde(default)]`) | `vmm/src/app.rs:48-73` |
| App-id derivation (already deterministic 40-hex) | `vmm/src/main_service.rs:46-55` |
| GetMeta returns global limits | `vmm/src/main_service.rs:521-554` |
| GPU inventory is global | `vmm/src/main_service.rs:556-563` |
| Port-mapping policy is global | `vmm/src/config.rs:122-143`, `vmm/src/main_service.rs:141-167` |
| Discovery file (used by CLI for `vmm ls`) | `vmm/src/discovery.rs:18-97`, `vmm/src/vmm-cli.py:41-105` |
| CLI auth flags | `vmm/src/vmm-cli.py:1515-1539` |
| `Vmm.Status` (per-VM read used by import flow) | `vmm/src/main_service.rs:353-355`, `vmm/rpc/proto/vmm_rpc.proto:321` |
| Log streaming endpoint (proxied by control plane) | `vmm/src/main_routes.rs:107-183` |
| Host API vsock-only constraint (cannot be remoted) | `vmm/src/config.rs:454-466`, `vmm/src/main.rs:111-130` |
| ReportDhcpLease (move off public surface) | `vmm/src/main_service.rs:579-582`, `vmm/rpc/proto/vmm_rpc.proto:344-345` |
| `DstackKms.registerApp` / `deployAndRegisterApp` | `kms/auth-eth/contracts/DstackKms.sol:130-167` |
| `DstackApp` owner-only ops | `kms/auth-eth/contracts/DstackApp.sol:110-117,158-220` |
| Hardhat tasks operators run today | `kms/auth-eth/hardhat.config.ts:346,431,451` |
