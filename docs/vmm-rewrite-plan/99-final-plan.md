# dstack-control-plane — final plan

**Prepared:** 2026-05-05
**Status:** synthesis pass output. The decisions in `00-decisions.md` are
the source of truth; the 11 investigation reports are the evidence base
this plan is built on. Where this document and `00-decisions.md`
disagree, `00-decisions.md` wins.

This is the document the implementation runs from. It is intentionally
concrete — a "we could start coding from this on Monday" plan — even
where that means picking a specific library version, table column type,
or HTTP path.

---

## 1. Executive summary

`dstack-control-plane` is a single new Rust binary that **replaces**
`dstack-vmm` (`vmm/src/main.rs`) for fresh installs of dstack. Greenfield
only — no migration from existing vmm installs, no `vmm-cli.py` compat,
no prpc-v1 compat, no on-chain re-registration. The legacy `dstack-vmm`
binary stays in the tree for legacy installs but is not part of the new
install path (`docs/vmm-rewrite-plan/00-decisions.md:264-269`).

The new binary combines **the multi-tenant management surface** and **the
host-side QEMU/supervisor responsibilities** of today's vmm into a single
process that manages exactly one TDX bare-metal box. Multi-host
orchestration is explicitly out of scope for v1
(`00-decisions.md:26-32`); operators who want a multi-host cloud run one
control plane per host and federate above us if they want to.

The big design choices, in one paragraph: identity is **Privy**
(per-user embedded wallet + BYO external wallet); off-chain auth is
three-role org-scoped RBAC (Admin / Billing / Viewer) plus a global
Platform Admin; on-chain authority is a **Safe multisig per org** on
**Base mainnet (chain ID 8453)** where current admins are auto-synced as
1-of-N signers; state lives in a sibling **Postgres** container reached
over a Unix socket; secrets are sodium-secretbox-encrypted under an
operator-supplied master key (KMK); audit log is hash-chained and
operator-signed in hourly batches; a **full OCI-compliant per-org
private container registry** ships in v1 (`00-decisions.md:7-23`,
`:46-77`, `:78-108`, `:111-141`, `:144-164`, `:166-186`, `:202-216`).

What this replaces concretely:

- `dstack-vmm` the binary — the management surface and host QEMU
  ownership in one process (`vmm/src/main.rs:148-236`).
- The `dstack-vmm` web console (Vue SPA embedded via `include_str!` at
  `vmm/src/main_routes.rs:19`).
- `vmm-cli.py` for new installs (`vmm/src/vmm-cli.py:1-2024`).
- The flat `/prpc/` HTTP/JSON surface
  (`vmm/rpc/proto/vmm_rpc.proto:300-360`).
- The single shared bearer-token "auth" model
  (`vmm/src/main.rs:208`, `vmm/vmm.toml:126-128`).
- The on-disk `vm-manifest.json` / `vm-state.json` / `.removing`
  source-of-truth (`vmm/src/app.rs:48-73`,
  `vmm/src/app/qemu.rs:108-111,1116-1118`).

What is preserved unchanged:

- The **host-API on vsock** (`vmm/src/host_api_service.rs:35-63`,
  `vmm/src/main.rs:111-130`) — vsock cannot be remoted, and this is the
  guest↔host integration channel; we keep `prpc` over vsock for it
  (`00-decisions.md:240-241`, `06-vmm-auth-today-and-gaps.md:33`).
- The **dstack-kms** boundary
  (`vmm/src/main_service.rs:455-468`, single call to
  `GetAppEnvEncryptPubKey`); the new control plane is the proxy.
- The **dstack-gateway** ingress URL convention
  `<id>[-<port>][s|g].<base>.tld`; org scoping is enforced in API/UI,
  not in the URL (`00-decisions.md:220-222`,
  `vmm/src/app/qemu.rs:276-303`).
- The **supervisor** child process and its `kill_on_drop(true)` model
  for QEMU lifetime (`supervisor/src/process.rs:181-201`,
  `vmm/src/app.rs:209-222`). Wrapped behind a trait, but the binary and
  the wire (HTTP-JSON over UDS) stay.

The result is a single binary that an operator drops onto a bare-metal
TDX host alongside Postgres-in-Docker, gives an EVM RPC URL and a Privy
app secret, and gets a multi-tenant cloud out the other end.

---

## 2. Architecture

### 2.1 Process layout

A dstack install is **one bare-metal TDX host**, running:

| # | What | UID | Process |
|---|------|-----|---------|
| 1 | `dstack-control-plane` (this binary) | `dstack` | Single process. Owns the REST/JSON API, the OCI registry endpoint, the audit/quota/scheduler logic, the on-host QEMU-driver code, the supervisor connection, and the Privy/Safe/KMS clients. |
| 2 | Postgres 16 in Docker | `postgres` (container) → bind-mounted Unix socket owned by `dstack` | Database. Connection over Unix-socket peer auth (no password). |
| 3 | `dstack-supervisor` child | `dstack` | Auto-spawned by the control plane the first time it starts (preserving today's `SupervisorClient::start_and_connect_uds` shape, `supervisor/client/src/lib.rs:26-88`). Owns the `tokio::process::Command::spawn` + `kill_on_drop(true)` for every QEMU. |
| 4 | One QEMU per CVM | `dstack-org-<slug>` (per-org sandbox uid) | Each CVM runs as a child of supervisor, wrapped in `sudo -u <org-uid>` (extension of today's `vmm/src/app/qemu.rs:743-748`). |
| 5 | Host services | root | Bridge interfaces, optional VLAN setup, optional dnsmasq for bridge mode. Operator-managed; we don't run these. |

The trust boundary is **outside the TEE**: the control plane runs as a
regular Linux process; the TEE protects the guest CVMs, not the host
orchestrator (`00-decisions.md:36-41`). A v2 "control plane inside an
attested CVM" mode is explicitly out of scope.

### 2.2 Listening sockets and endpoints

| Endpoint | Bind | Auth | Purpose |
|----------|------|------|---------|
| `https://api.<base>.tld/v1/...` | TCP 443 (TLS terminated by the dstack-gateway, which already does ACME) | Session cookie (browsers) or `Authorization: Bearer dst_<token>` (machines) | The user-facing REST API. |
| `https://registry.<base>.tld/v2/...` | TCP 443 (separate hostname, separate ACME cert) | OCI bearer token | The OCI v2 distribution endpoint. |
| `unix:/var/run/dstack/control.sock` | Unix domain | Peer creds — accepts only the `dstack` uid | Internal admin / health / debug endpoint. The reason metrics is gated to platform-admin and isn't available on the public TLS hostname. |
| `vsock:CID=2:port=10000` | vsock | Caller's vsock CID (`vmm/src/host_api_service.rs:21-33`) | Host API, called by guest agents. **Identical contract to today's `host_api`**, including the `validate()` check that rejects non-vsock addresses (`vmm/src/config.rs:454-466`). |
| `unix:/run/dstack/postgres.sock` | Unix domain (Docker bind-mount) | Peer creds | Postgres connection. The control plane is the only process with access to this socket; CVM uids cannot reach it. |
| `unix:/run/dstack/supervisor.sock` | Unix domain | None (file-mode 0600 owned by `dstack` uid) | Supervisor IPC, unchanged from today (`vmm/vmm.toml:131-135`). |

**No public TCP listener for `control.sock`, `postgres.sock`, or
`supervisor.sock`.** The only public TCP surfaces are 443 (api +
registry, multiplexed by SNI on the gateway) and whatever the operator
has open for the gateway's CVM ingress traffic.

### 2.3 Filesystem layout

```
/etc/dstack/
  control-plane.toml          mode 0640, owned by dstack:dstack
  master.key                  mode 0600, the KMK (32 random bytes,
                              auto-generated on first start, see §12)

/var/lib/dstack/
  pg/                         Postgres data dir, owned by the postgres
                              container's uid; bind-mounted into Docker.
  registry/blobs/<sha256>     OCI content-addressed blob store, mode 0640
                              owned by dstack:dstack (00-decisions.md:208-211).
  vm/<vm-id>/                 per-CVM workdir on the host fs; identical
                              shape to today's run_path/<vm-id>/, with
                              the per-org sandbox uid owning the dir.
                              Subfiles: hda.img, shared/.encrypted-env,
                              shared/.user-config, shared/.instance_info,
                              shared/app-compose.json, shared/.sys-config.json,
                              serial.log, serial.history.log,
                              stdout.log, stderr.log, qemu.pid,
                              qmp.sock, serial.pty, .removing
                              (matching vmm/src/app/qemu.rs:990-1117 except
                              vm-manifest.json + vm-state.json are gone —
                              they live in Postgres now, see §3).
  images/                     guest OS image catalog (operator-managed).
                              Identical to today's image.path —
                              vmm/src/config.rs:506-541.

/run/dstack/
  control.sock                admin/debug Unix domain socket
  postgres.sock               Postgres Unix socket (bind-mount into Docker)
  supervisor.sock             supervisor IPC

/home/dstack/.dstack/         logs, OTLP buffers, etc.
```

The on-disk per-CVM workdir keeps almost the same shape as today —
because that shape is well-tested and QEMU expects files at fixed paths
(`vmm/src/app/qemu.rs:555-599`). Two changes:

1. The `vm-manifest.json` and `vm-state.json` files are **gone**. The
   control plane materialises everything QEMU needs from the
   Postgres-resident `cvms` row before each start. This is the
   "DB is source of truth, host re-materialises artifacts to disk"
   pattern (`00-decisions.md:138-141`,
   `07-persistence-today-and-multitenant.md:472-516`).
2. Each `/var/lib/dstack/vm/<vm-id>/` is owned by the per-org sandbox
   uid, not by `dstack-prd1` (the single-uid model in
   `vmm/src/setup-user.sh:1-230` and `vmm/src/config.rs:171`). Per-org
   uids are pre-provisioned by an install script (see §12) and are
   stored in the `organizations.sandbox_uid` column.

### 2.4 ASCII diagram

```
                          ┌────────────────────────────────────────┐
                          │            dstack-gateway              │
                          │  - ACME, TLS termination, RA-TLS, WG   │
                          │  - SNI: api/registry/<id>-<port>.<bd>  │
                          └─────────┬──────────────────┬───────────┘
                                    │  HTTPS           │ HTTPS (CVM ingress)
       ┌────────────────────────────▼──────────────┐   │
       │           dstack-control-plane             │   │
       │   ┌──────────────────────────────────────┐ │   │
       │   │  axum HTTP server (port via UDS)     │ │   │
       │   │   /v1/...        REST/JSON API       │ │   │
       │   │   /v2/...        OCI registry        │ │   │
       │   └──────────────────────────────────────┘ │   │
       │                                            │   │
       │   ┌──────────────────────────────────────┐ │   │
       │   │  in-process modules                  │ │   │
       │   │   - org/auth/RBAC                    │ │   │
       │   │   - quota + admission                │ │   │
       │   │   - audit + signed batches           │ │   │
       │   │   - Privy server SDK                 │ │   │
       │   │   - Safe/Base orchestration (alloy)  │ │   │
       │   │   - KMS client (existing prpc)       │ │   │
       │   │   - host worker (QEMU command build) │ │   │
       │   │   - OCI registry storage             │ │   │
       │   └──────────┬─────────────────┬─────────┘ │   │
       └──────────────┼─────────────────┼───────────┘   │
                      │ UDS (peer auth)│ UDS           │
                      ▼                ▼               │
              ┌───────────────┐  ┌──────────────┐       │
              │ Postgres 16   │  │ supervisor   │       │
              │ (Docker)      │  │ (binary,     │       │
              │ /run/dstack/  │  │  owned by    │       │
              │ postgres.sock │  │  dstack uid) │       │
              └───────────────┘  └──────┬───────┘       │
                                        │ spawn         │
                                        ▼               │
       ┌──────────────────────────────────────────────┐ │
       │  per-org sandbox uids running QEMU:          │ │
       │   sudo -u dstack-org-acme  qemu-system-x86_64 ...   │
       │   sudo -u dstack-org-foo   qemu-system-x86_64 ...   │
       │                                              │ │
       │   each QEMU is a TDX guest CVM, talks vsock  │ │
       │   to control-plane host_api, exposes apps    │ │
       │   to the gateway via WireGuard               │ │
       └──────────────────────────────────────────────┘ │
                              │ vsock CID                 │
                              ▼                           │
                   ┌────────────────────┐                 │
                   │ control-plane host │◄────────────────┘
                   │ API (vsock:2:10000)│ CVM ingress through
                   │  Notify, GetSealing│ gateway → CVMs
                   │  Key, Info         │
                   └────────────────────┘
```

### 2.5 Boundaries / what goes where

- **Control plane process** owns: HTTP API, OCI registry, all RBAC
  decisions, all DB transactions, audit-row writes, Privy session
  validation, Safe transaction proposal/signing-orchestration, KMS proxy
  calls (`vmm/src/main_service.rs:455-468`), and the host-worker module
  that builds QEMU command lines.
- **Postgres** owns: every metadata fact about users, orgs, CVMs,
  artifacts, audit events, signed batches, secrets, images, OCI
  manifests/blobs/repos/tags, quotas, overrides. `pg_dump`/`pg_restore`
  is the documented backup strategy; no scheduled backups in v1
  (`00-decisions.md:113-117`).
- **Supervisor** owns: each running QEMU's `tokio::process::Child` and
  the `kill_on_drop(true)` semantics (`supervisor/src/process.rs:181`).
  Identical to today.
- **Filesystem** owns: large blobs (qcow2 disks, base image files,
  registry blobs, log files, qmp/serial sockets). All metadata
  *describing* those blobs is in Postgres (`00-decisions.md:138-141`,
  `07-persistence-today-and-multitenant.md:471-489`).
- **dstack-kms (separate)** owns: per-app key derivation and on-chain
  whitelist enforcement. We continue to call exactly one KMS RPC
  (`GetAppEnvEncryptPubKey`) from the control plane
  (`vmm/src/main_service.rs:455-468`,
  `05-vmm-coupling-map.md:75-83`), unchanged.
- **dstack-gateway (separate)** owns: ACME, TLS termination, WireGuard
  fabric, ingress mapping (`<id>-<port>.<bd>`). The control plane never
  calls the gateway (`05-vmm-coupling-map.md:135-188`); we just emit
  URLs into `.sys-config.json` for guest agents to register with.

### 2.6 Network isolation

Per `00-decisions.md:218-229`:

- **Per-org Linux uid for QEMU** is mandatory, defense-in-depth on top
  of TEE isolation. This extends today's single-uid sandbox
  (`vmm/src/setup-user.sh:1-230`,
  `04-vmm-lifecycle-and-state.md:475-477`) to a per-org sandbox.
- **Per-org VLAN tag in bridge mode** is conditional on F3 outcome. F3
  is a focused investigation that runs alongside this synthesis pass to
  confirm that gateway / guest-agent don't already provide equivalent
  isolation. Two outcomes:
  - **F3 says yes (already isolated by gateway):** drop the VLAN layer.
    The per-org sandbox uid + TEE memory isolation + RA-TLS-pinned
    gateway peers is sufficient.
  - **F3 says no:** implement per-org VLAN tagging in bridge mode. Each
    org gets a `vlan_id` column in the `organizations` table. Bridge
    mode QEMU args become `-netdev tap,br=<bridge>,vlan=<vlan_id>` plus
    operator-side dnsmasq scoping. Default behaviour for `mode=user` is
    unchanged (per-VM NAT), since user mode already provides per-VM
    isolation by definition (`vmm/src/app/qemu.rs:512-520`).

The final plan assumes "F3 = yes" until F3 says otherwise; the
`organizations.vlan_id` column is included as nullable in the schema so
the wiring can be added later without a migration.

---

## 3. Database schema

Postgres 16. Migrations are managed by **`sqlx-cli` migrations**
(`sqlx 0.8.x`, see §16) — one numbered SQL file per migration in
`db/migrations/`. The control plane runs `sqlx migrate run` at startup
inside a Postgres advisory lock.

UUIDs are **UUIDv7** (`uuid 1.x` with the `v7` feature, time-ordered for
B-tree locality). Timestamps are `timestamptz`. JSON columns are `jsonb`.
Hashes are `bytea` (raw bytes, not hex strings).

```sql
-- ─────────────────────────────────────────────────────────────────────
-- 1. Identity & access
-- ─────────────────────────────────────────────────────────────────────

CREATE TABLE users (
    id                 uuid PRIMARY KEY,
    -- Privy is the identity provider (00-decisions.md:46-49). The Privy
    -- DID is the canonical identifier; email is captured for display
    -- and for invitation matching but is never the auth credential.
    privy_did          text NOT NULL UNIQUE,
    primary_wallet_id  uuid,                                    -- FK set after wallet rows exist; nullable for new users
    email              citext,
    display_name       text,
    is_platform_admin  boolean NOT NULL DEFAULT false,          -- (00-decisions.md:62-67)
    suspended_at       timestamptz,                              -- platform-admin suspension
    suspended_by       uuid REFERENCES users(id),
    suspended_reason   text,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX users_email_idx ON users (lower(email));

CREATE TABLE user_wallets (
    id              uuid PRIMARY KEY,
    user_id         uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    address         bytea NOT NULL CHECK (octet_length(address) = 20),  -- EVM 20-byte address
    chain_id        int NOT NULL DEFAULT 8453,                          -- Base mainnet (00-decisions.md:80-82)
    -- Privy assigns each linked wallet a "linked_account_id"; we record
    -- it for de-duplication when the user re-links the same wallet.
    privy_account_id text,
    is_embedded     boolean NOT NULL DEFAULT false,             -- true if Privy embedded wallet
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX user_wallets_address_idx ON user_wallets (address);
CREATE INDEX user_wallets_user_idx ON user_wallets (user_id);

ALTER TABLE users
    ADD CONSTRAINT users_primary_wallet_fk
    FOREIGN KEY (primary_wallet_id) REFERENCES user_wallets(id) ON DELETE SET NULL;

CREATE TABLE organizations (
    id               uuid PRIMARY KEY,
    slug             citext NOT NULL UNIQUE                     -- 2-32 chars, [a-z0-9-]
        CHECK (slug ~ '^[a-z0-9][a-z0-9-]{1,30}[a-z0-9]$|^[a-z0-9]{2,32}$'),
    name             text NOT NULL,
    is_personal      boolean NOT NULL DEFAULT false,
    sandbox_uid      int NOT NULL,                              -- Linux uid of dstack-org-<slug>; auto-allocated at create
    vlan_id          int,                                        -- nullable; populated only if F3 says we need VLANs
    -- Soft-delete tombstone (00-decisions.md:281-284): blocked if any
    -- CVMs exist; 30-day grace for audit, then GC removes the row.
    deleted_at       timestamptz,
    deleted_by       uuid REFERENCES users(id),
    created_by       uuid NOT NULL REFERENCES users(id),
    created_at       timestamptz NOT NULL DEFAULT now(),
    updated_at       timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX organizations_active_idx ON organizations (id) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX organizations_sandbox_uid_idx ON organizations (sandbox_uid);

CREATE TABLE memberships (
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            text NOT NULL CHECK (role IN ('admin','billing','viewer')),
    -- "Pending removal": admin role-change is committed only after the
    -- on-chain Safe removeOwner is signed (00-decisions.md:88-91). The
    -- pending_removal flag drives the UI banner.
    pending_removal      boolean NOT NULL DEFAULT false,
    pending_role_change  text CHECK (pending_role_change IN ('admin','billing','viewer')),
    invited_by      uuid REFERENCES users(id),
    accepted_at     timestamptz,
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, user_id)
);
CREATE INDEX memberships_user_idx ON memberships (user_id);

CREATE TABLE invitations (
    id              uuid PRIMARY KEY,
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           citext NOT NULL,
    role            text NOT NULL CHECK (role IN ('admin','billing','viewer')),
    invited_by      uuid NOT NULL REFERENCES users(id),
    token_hash      bytea NOT NULL CHECK (octet_length(token_hash) = 32),  -- sha256 of opaque secret
    status          text NOT NULL CHECK (status IN ('pending','accepted','revoked','expired')),
    expires_at      timestamptz NOT NULL,
    accepted_by     uuid REFERENCES users(id),
    accepted_at     timestamptz,
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX invitations_org_status_idx ON invitations (organization_id, status)
    WHERE status = 'pending';
CREATE UNIQUE INDEX invitations_pending_email_idx
    ON invitations (organization_id, lower(email))
    WHERE status = 'pending';

CREATE TABLE api_tokens (
    id              uuid PRIMARY KEY,
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            text NOT NULL,
    -- Token format: dst_<base32(random 24 bytes)>; we store sha256 only
    -- (00-decisions.md:69-76).
    hash            bytea NOT NULL CHECK (octet_length(hash) = 32),
    role            text NOT NULL CHECK (role IN ('admin','billing','viewer')),
    created_by_user uuid REFERENCES users(id) ON DELETE SET NULL,
    last_used_at    timestamptz,
    expires_at      timestamptz,                                 -- NULL = never (00-decisions.md:71)
    revoked_at      timestamptz,
    revoked_by      uuid REFERENCES users(id) ON DELETE SET NULL,
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX api_tokens_hash_idx ON api_tokens (hash);
CREATE INDEX api_tokens_org_idx ON api_tokens (organization_id) WHERE revoked_at IS NULL;

CREATE TABLE sessions (
    id              uuid PRIMARY KEY,                            -- becomes the session-cookie value (HMAC'd)
    user_id         uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    privy_session   text NOT NULL,                               -- the Privy session token / refresh handle
    created_at      timestamptz NOT NULL DEFAULT now(),
    expires_at      timestamptz NOT NULL,
    last_seen_at    timestamptz NOT NULL DEFAULT now(),
    revoked_at      timestamptz,
    user_agent      text,
    client_ip       inet
);
CREATE INDEX sessions_user_idx ON sessions (user_id) WHERE revoked_at IS NULL;
CREATE INDEX sessions_expires_idx ON sessions (expires_at) WHERE revoked_at IS NULL;

-- ─────────────────────────────────────────────────────────────────────
-- 2. CVMs and artifacts
-- ─────────────────────────────────────────────────────────────────────

-- Replaces the on-disk Manifest struct (vmm/src/app.rs:48-73) and the
-- in-memory VmState (vmm/src/app.rs:1212-1255).
CREATE TABLE cvms (
    id                 uuid PRIMARY KEY,
    organization_id    uuid NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT,
    created_by         uuid NOT NULL REFERENCES users(id),
    image_id           uuid NOT NULL REFERENCES images(id),
    name               text NOT NULL,                            -- charset matches vmm/src/main_service.rs:58-69
    -- app_id derivation (00-decisions.md:96-100):
    --   app_id = truncate40(sha256(org_id || compose))
    -- Stored as 40 lowercase hex chars to match KMS / DstackApp expectations.
    app_id             text NOT NULL CHECK (app_id ~ '^[0-9a-f]{40}$'),
    -- declared resources (mirrors Manifest fields):
    vcpu               int NOT NULL,
    memory_mb          int NOT NULL,
    disk_size_gb       int NOT NULL,
    gpus               jsonb NOT NULL DEFAULT '{}',              -- {attach_mode, gpus: [...], bridges: [...]}
    networking         jsonb,                                     -- nullable, matches Networking struct
    port_map           jsonb NOT NULL DEFAULT '[]',
    kms_urls           text[] NOT NULL DEFAULT '{}',
    gateway_urls       text[] NOT NULL DEFAULT '{}',
    no_tee             boolean NOT NULL DEFAULT false,
    hugepages          boolean NOT NULL DEFAULT false,
    pin_numa           boolean NOT NULL DEFAULT false,
    -- runtime state (replaces vm-state.json + the live status string in
    -- vmm/src/app/qemu.rs:312-342). Two columns: what the user wants
    -- (desired_state), and what the system observes (observed_state).
    desired_state      text NOT NULL CHECK (desired_state IN ('running','stopped','removed')),
    observed_state     text NOT NULL CHECK (observed_state IN
        ('pending','provisioning','starting','running','degraded',
         'stopping','stopped','exited','failed','deleting','deleted')),
    cid                int,                                       -- assigned at start; UNIQUE with workdir present
    instance_id        text,                                      -- guest-reported, populated via host_api Notify
    boot_progress      text,
    boot_error         text,
    shutdown_progress  text,
    guest_ip           inet,                                      -- bridge mode DHCP lease (vmm/src/app.rs:439-441)
    workdir            text NOT NULL,                             -- /var/lib/dstack/vm/<id>
    started_at         timestamptz,
    stopped_at         timestamptz,
    last_seen_at       timestamptz,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX cvms_org_idx                ON cvms (organization_id) WHERE desired_state <> 'removed';
CREATE INDEX cvms_observed_state_idx     ON cvms (observed_state);
CREATE INDEX cvms_org_created_at_idx     ON cvms (organization_id, created_at DESC);
-- CID is host-local (we are single-host); UNIQUE per running VM.
CREATE UNIQUE INDEX cvms_cid_idx ON cvms (cid)
    WHERE cid IS NOT NULL AND observed_state <> 'deleted';
-- Names unique within an org while alive.
CREATE UNIQUE INDEX cvms_org_name_idx
    ON cvms (organization_id, lower(name))
    WHERE desired_state <> 'removed';

-- Versioned per-CVM blobs that the host re-materialises into
-- <workdir>/shared/ before each start. Replaces the on-disk
-- shared/app-compose.json, shared/.encrypted-env, shared/.user-config,
-- shared/.instance_info (today: vmm/src/app.rs:964-994 +
-- dstack-types/src/shared_filenames.rs).
CREATE TABLE cvm_artifacts (
    cvm_id      uuid NOT NULL REFERENCES cvms(id) ON DELETE CASCADE,
    kind        text NOT NULL CHECK (kind IN
        ('app_compose','encrypted_env','user_config','instance_info')),
    version     int NOT NULL,
    sha256      bytea NOT NULL CHECK (octet_length(sha256) = 32),
    body        bytea,                                            -- inline storage; small blobs only
    created_by  uuid REFERENCES users(id),                        -- nullable: guest-initiated writes have no actor
    created_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (cvm_id, kind, version)
);
CREATE INDEX cvm_artifacts_latest_idx ON cvm_artifacts (cvm_id, kind, version DESC);

-- Global guest OS image catalog (operator-managed). Per-org images are
-- explicitly v2+ (00-decisions.md:215-216). The catalog is small
-- enough to live in Postgres entirely; the actual
-- kernel/initrd/rootfs files live in /var/lib/dstack/images/<name>/.
CREATE TABLE images (
    id            uuid PRIMARY KEY,
    name          text NOT NULL UNIQUE,                           -- "dstack-0.5.8"
    version       text NOT NULL,
    is_dev        boolean NOT NULL DEFAULT false,
    digest        bytea NOT NULL CHECK (octet_length(digest) = 32),
    metadata      jsonb NOT NULL,                                  -- copied from metadata.json
    on_disk_path  text NOT NULL,                                   -- absolute path on the host
    uploaded_by   uuid REFERENCES users(id),
    uploaded_at   timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX images_digest_idx ON images (digest);

-- ─────────────────────────────────────────────────────────────────────
-- 3. Safe / on-chain
-- ─────────────────────────────────────────────────────────────────────

CREATE TABLE safes (
    organization_id   uuid PRIMARY KEY REFERENCES organizations(id) ON DELETE CASCADE,
    -- Safe is deployed lazily on the org's first on-chain action
    -- (00-decisions.md:84-85). Until then, safe_address is NULL.
    safe_address      bytea CHECK (octet_length(safe_address) = 20),
    chain_id          int NOT NULL DEFAULT 8453,
    threshold         int NOT NULL DEFAULT 1,                      -- always 1 in v1 (00-decisions.md:92-94)
    -- Cached signers, refreshed by polling the chain. Authoritative
    -- source is on-chain; this is just a cache for the UI and audit.
    signers_cache     jsonb NOT NULL DEFAULT '[]',                  -- [{address: "0x...", added_block: 12345}]
    signers_synced_at timestamptz,
    -- The optional platform recovery signer (00-decisions.md:101-108)
    -- if it's part of this org's Safe.
    has_recovery_signer boolean NOT NULL DEFAULT false,
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now()
);

-- Tracks pending Safe transactions awaiting admin signatures. A txn
-- moves through proposed → signed → submitted → confirmed/failed.
CREATE TABLE safe_transactions (
    id                  uuid PRIMARY KEY,
    organization_id     uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    -- The intent: addOwner / removeOwner / changeThreshold / call to
    -- DstackKms.deployAndRegisterApp, etc.
    intent              text NOT NULL CHECK (intent IN
        ('add_owner','remove_owner','change_threshold','create_app',
         'add_compose_hash','remove_compose_hash','add_device',
         'remove_device','set_allow_any_device','set_require_tcb',
         'disable_upgrades','arbitrary_call')),
    -- The encoded calldata + target. We compute the Safe transaction
    -- hash off-chain and store it; admins sign the hash, not the
    -- calldata, so we serve the hash to Privy's signing UI.
    target_contract     bytea NOT NULL CHECK (octet_length(target_contract) = 20),
    calldata            bytea NOT NULL,
    safe_tx_hash        bytea NOT NULL CHECK (octet_length(safe_tx_hash) = 32),
    nonce               bigint NOT NULL,
    status              text NOT NULL CHECK (status IN
        ('proposed','signed','submitted','confirmed','failed','expired')),
    proposed_by         uuid NOT NULL REFERENCES users(id),
    submitted_tx_hash   bytea CHECK (octet_length(submitted_tx_hash) = 32),
    block_number        bigint,
    error               text,
    expires_at          timestamptz NOT NULL,                      -- defaults to created + 7 days
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX safe_transactions_org_idx ON safe_transactions (organization_id, status);
CREATE INDEX safe_transactions_status_idx ON safe_transactions (status)
    WHERE status IN ('proposed','signed','submitted');

CREATE TABLE safe_signatures (
    safe_transaction_id uuid NOT NULL REFERENCES safe_transactions(id) ON DELETE CASCADE,
    signer_address      bytea NOT NULL CHECK (octet_length(signer_address) = 20),
    signature           bytea NOT NULL,
    signed_by_user      uuid REFERENCES users(id) ON DELETE SET NULL,
    created_at          timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (safe_transaction_id, signer_address)
);

-- DstackApp contract registrations per org. One row per (org, app_id)
-- — typically 1:1 with cvms, but multiple CVMs can share an app_id
-- (e.g. blue/green of the same compose).
CREATE TABLE kms_apps (
    id                  uuid PRIMARY KEY,
    organization_id     uuid NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT,
    app_id              text NOT NULL CHECK (app_id ~ '^[0-9a-f]{40}$'),
    dstack_app_address  bytea NOT NULL CHECK (octet_length(dstack_app_address) = 20),
    chain_id            int NOT NULL DEFAULT 8453,
    -- We mirror the on-chain whitelist for fast UI rendering. Source of
    -- truth is the chain; the column is a cache.
    allowed_compose_hashes_cache jsonb NOT NULL DEFAULT '[]',
    cache_synced_at     timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX kms_apps_app_id_idx ON kms_apps (app_id);
CREATE INDEX kms_apps_org_idx ON kms_apps (organization_id);

-- ─────────────────────────────────────────────────────────────────────
-- 4. Quotas
-- ─────────────────────────────────────────────────────────────────────

-- Default quotas per org. Soft (warn) and hard (block) caps on every
-- dimension (00-decisions.md:189-199).
CREATE TABLE quotas (
    organization_id        uuid PRIMARY KEY REFERENCES organizations(id) ON DELETE CASCADE,
    max_cvms               int NOT NULL DEFAULT 5,
    max_vcpu               int NOT NULL DEFAULT 16,
    max_memory_mb          int NOT NULL DEFAULT 32768,
    max_disk_gb            int NOT NULL DEFAULT 200,
    max_ingress_ports      int NOT NULL DEFAULT 32,
    max_image_storage_mb   int NOT NULL DEFAULT 10240,
    max_deploys_per_hour   int NOT NULL DEFAULT 30,
    -- Soft thresholds (default = 80% of hard).
    soft_pct               int NOT NULL DEFAULT 80 CHECK (soft_pct BETWEEN 1 AND 100),
    updated_at             timestamptz NOT NULL DEFAULT now()
);

-- Platform-admin-granted overrides (00-decisions.md:198-199). Deletable
-- via an `expires_at`. Each row replaces the corresponding column in
-- quotas for one dimension.
CREATE TABLE quota_overrides (
    id              uuid PRIMARY KEY,
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    dimension       text NOT NULL CHECK (dimension IN
        ('max_cvms','max_vcpu','max_memory_mb','max_disk_gb',
         'max_ingress_ports','max_image_storage_mb','max_deploys_per_hour')),
    value           bigint NOT NULL,
    granted_by      uuid NOT NULL REFERENCES users(id),
    reason          text NOT NULL,
    expires_at      timestamptz,
    revoked_at      timestamptz,
    revoked_by      uuid REFERENCES users(id),
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX quota_overrides_active_idx
    ON quota_overrides (organization_id, dimension)
    WHERE revoked_at IS NULL AND (expires_at IS NULL OR expires_at > now());

-- Resource usage rolling table for billing primitives + soft-quota
-- warnings. The host-side measurement loop appends rows every 30s
-- (00-decisions.md:251-253).
CREATE TABLE resource_usage (
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    cvm_id          uuid REFERENCES cvms(id) ON DELETE CASCADE,
    bucket_start    timestamptz NOT NULL,                          -- 30s buckets
    vcpu_seconds    double precision NOT NULL DEFAULT 0,           -- cgroup-derived
    memory_mb_avg   double precision NOT NULL DEFAULT 0,
    disk_gb         double precision NOT NULL DEFAULT 0,
    network_rx_b    bigint NOT NULL DEFAULT 0,
    network_tx_b    bigint NOT NULL DEFAULT 0,
    PRIMARY KEY (organization_id, cvm_id, bucket_start)
);
CREATE INDEX resource_usage_org_time_idx
    ON resource_usage (organization_id, bucket_start DESC);

-- ─────────────────────────────────────────────────────────────────────
-- 5. Audit + signed batches
-- ─────────────────────────────────────────────────────────────────────

CREATE TABLE audit_log (
    id              bigserial PRIMARY KEY,                        -- monotonic; the chain follows id order
    organization_id uuid REFERENCES organizations(id) ON DELETE SET NULL,
    actor_user_id   uuid REFERENCES users(id) ON DELETE SET NULL,
    actor_token_id  uuid REFERENCES api_tokens(id) ON DELETE SET NULL,
    actor_ip        inet,
    actor_ua        text,
    action          text NOT NULL,                                -- "cvm.create", "membership.role.change", ...
    target_kind     text,
    target_id       text,
    request_id      uuid,
    payload         jsonb NOT NULL DEFAULT '{}',
    result          text NOT NULL CHECK (result IN ('success','failure','denied')),
    error           text,
    occurred_at     timestamptz NOT NULL DEFAULT now(),
    -- Hash chain (00-decisions.md:168-171). prev_hash is the row_hash
    -- of the previous audit_log row (by id). row_hash is sha256 of the
    -- canonical encoding (see §9). genesis row has prev_hash = 32×0x00.
    prev_hash       bytea NOT NULL CHECK (octet_length(prev_hash) = 32),
    row_hash        bytea NOT NULL CHECK (octet_length(row_hash) = 32)
);
CREATE INDEX audit_log_org_time_idx ON audit_log (organization_id, occurred_at DESC);
CREATE INDEX audit_log_action_idx   ON audit_log (action);
CREATE INDEX audit_log_actor_user_idx
    ON audit_log (actor_user_id, occurred_at DESC) WHERE actor_user_id IS NOT NULL;
-- Unique row_hash enables fast verification.
CREATE UNIQUE INDEX audit_log_row_hash_idx ON audit_log (row_hash);

-- Hourly signed batches (00-decisions.md:172-179).
CREATE TABLE signed_batches (
    id            bigserial PRIMARY KEY,
    -- Each batch covers audit_log rows where id ∈ [first_id, last_id].
    first_id      bigint NOT NULL,
    last_id       bigint NOT NULL,
    merkle_root   bytea NOT NULL CHECK (octet_length(merkle_root) = 32),
    -- secp256k1 signature over (sha256("dstack-audit-batch-v1" || merkle_root || first_id || last_id || timestamp))
    -- using the audit_signing_key in secrets.
    signer_pubkey bytea NOT NULL CHECK (octet_length(signer_pubkey) = 33), -- compressed
    signature     bytea NOT NULL,
    -- Cumulative tip-hash so a verifier can stitch batches end-to-end.
    tip_hash      bytea NOT NULL CHECK (octet_length(tip_hash) = 32),
    created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX signed_batches_last_id_idx ON signed_batches (last_id);

-- ─────────────────────────────────────────────────────────────────────
-- 6. Secrets (sodium-secretbox encrypted by KMK)
-- ─────────────────────────────────────────────────────────────────────

-- All operator-side secrets (00-decisions.md:144-159). Decryption
-- requires the KMK loaded into the control plane process at startup.
CREATE TABLE secrets (
    -- Stable string key, e.g. "privy.app_secret", "audit.signing_key".
    name        text PRIMARY KEY,
    nonce       bytea NOT NULL CHECK (octet_length(nonce) = 24),    -- libsodium secretbox nonce
    ciphertext  bytea NOT NULL,                                      -- secretbox(plaintext, nonce, KMK)
    -- Free-form description shown in the platform-admin UI.
    description text,
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now()
);

-- Required well-known names (seeded at install):
--   privy.app_id
--   privy.app_secret
--   privy.webhook_secret
--   session.cookie_hmac_key
--   evm.rpc_url
--   evm.rpc_api_key      (optional)
--   recovery.signer_key  (only if §3 opt-in)
--   audit.signing_key    (always; separate from recovery.signer_key)
--   smtp.host / smtp.port / smtp.user / smtp.password / smtp.from_addr

-- ─────────────────────────────────────────────────────────────────────
-- 7. OCI registry
-- ─────────────────────────────────────────────────────────────────────

-- Content-addressed blobs. The bytes themselves live on disk under
-- /var/lib/dstack/registry/blobs/<sha256>/data. Postgres just tracks
-- the index (00-decisions.md:208-211).
CREATE TABLE oci_blobs (
    digest      bytea PRIMARY KEY CHECK (octet_length(digest) = 32),  -- sha256 only in v1
    media_type  text,                                                 -- registered when first referenced
    size_bytes  bigint NOT NULL,
    on_disk     boolean NOT NULL DEFAULT true,                        -- false = soft-deleted, awaiting GC
    created_at  timestamptz NOT NULL DEFAULT now(),
    last_seen_at timestamptz NOT NULL DEFAULT now()
);

-- Repos namespace registry contents per org. URL form:
--   registry.<base>.tld/<org-slug>/<repo>
CREATE TABLE oci_repos (
    id              uuid PRIMARY KEY,
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            text NOT NULL,                                  -- e.g. "my-app/web"; must match OCI rules
    visibility      text NOT NULL DEFAULT 'private'
                    CHECK (visibility IN ('private')),              -- v1: private only (00-decisions.md:213-214)
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX oci_repos_org_name_idx ON oci_repos (organization_id, lower(name));

-- Manifests reference blobs (the manifest itself is a blob too).
-- The (config, layers) tuple is the manifest's *content*; we store the
-- materialised parsed view here so the registry can serve HEAD/PULL
-- without re-parsing every blob.
CREATE TABLE oci_manifests (
    id                   uuid PRIMARY KEY,
    repo_id              uuid NOT NULL REFERENCES oci_repos(id) ON DELETE CASCADE,
    manifest_digest      bytea NOT NULL CHECK (octet_length(manifest_digest) = 32),
    manifest_media_type  text NOT NULL,                              -- application/vnd.oci.image.manifest.v1+json etc.
    config_digest        bytea CHECK (octet_length(config_digest) = 32),
    layer_digests        bytea[] NOT NULL DEFAULT '{}',
    size_bytes           bigint NOT NULL,
    created_by           uuid REFERENCES users(id),
    created_at           timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX oci_manifests_repo_idx ON oci_manifests (repo_id);
CREATE UNIQUE INDEX oci_manifests_digest_idx ON oci_manifests (repo_id, manifest_digest);

CREATE TABLE oci_tags (
    repo_id      uuid NOT NULL REFERENCES oci_repos(id) ON DELETE CASCADE,
    tag          text NOT NULL,                                       -- 1..128 ASCII per OCI spec
    manifest_id  uuid NOT NULL REFERENCES oci_manifests(id) ON DELETE RESTRICT,
    pushed_by    uuid REFERENCES users(id),
    pushed_at    timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (repo_id, tag)
);

-- The blob ↔ manifest reverse index (computed; not authoritative). We
-- compute "is this blob referenced anywhere?" as
--   EXISTS (
--     SELECT 1 FROM oci_manifests
--     WHERE manifest_digest = $1
--        OR config_digest    = $1
--        OR $1 = ANY(layer_digests)
--   )
-- The nightly GC job (§8) walks this query for each oci_blobs.digest
-- where last_seen_at < now() - interval '7 days' AND on_disk = true,
-- and soft-deletes (on_disk = false) plus removes the file.

-- Optional: in-progress upload sessions (the OCI two-step blob push).
CREATE TABLE oci_upload_sessions (
    id             uuid PRIMARY KEY,
    repo_id        uuid NOT NULL REFERENCES oci_repos(id) ON DELETE CASCADE,
    started_by     uuid NOT NULL REFERENCES users(id),
    bytes_received bigint NOT NULL DEFAULT 0,
    -- Path to the in-progress staging file under
    -- /var/lib/dstack/registry/uploads/<id>.
    staging_path   text NOT NULL,
    expires_at     timestamptz NOT NULL,                              -- session TTL, 24h
    created_at     timestamptz NOT NULL DEFAULT now(),
    updated_at     timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX oci_upload_sessions_expires_idx ON oci_upload_sessions (expires_at);
```

Notes on the schema:

- `audit_log.id` is `bigserial` (monotonic). The hash chain is over
  consecutive ids, which lets verification be a streaming pass.
- `cvms.observed_state` enum extends today's five-state model
  (`vmm/src/app/qemu.rs:317-331`) with `provisioning`, `degraded`,
  `failed`, `deleted` as suggested by
  `09-cvm-domain-model.md:218-234`.
- `cvm_artifacts` keeps per-revision history. The host re-materialises
  the latest version (per `kind`) into `<workdir>/shared/` before each
  start, exactly as today's `prepare_work_dir`
  (`vmm/src/app.rs:964-994`) does — but the source bytes come from the
  DB, not from disk.
- `oci_blobs` has no `repo_id`; blobs are content-addressed and shared
  across repos *within the same registry*. Cross-org blob sharing is a
  storage optimisation that's safe because the digest *is* the
  identity; access control is enforced at the manifest/repo layer.
- `safes.signers_cache` is a cache; the on-chain Safe is the source of
  truth. We refresh it lazily on every Safe-touching action and
  proactively every 5 minutes.
- `users.is_platform_admin` is a single bool, not a separate table. The
  first-user-becomes-admin promotion is a one-time UPDATE.

---

## 4. REST/JSON API surface

### 4.1 Stack and conventions

- **Framework:** Axum 0.7 (`axum`, `tower`, `tower-http`). We're moving
  off Rocket — see §16.
- **Versioning:** stable URL path `/v1/...`. Clients pin behaviour with
  the request header `Dstack-Api-Version: 2026-05-01`. Server
  recognises a known set of date-versions; unknown versions are 400ed.
  Date-version semantics: server default is the latest; pinning an
  older date-version asks the server to serialise responses in that
  shape. 12-month deprecation policy
  (`00-decisions.md:233-237`).
- **Pagination:** opaque cursor strings, default 50, max 200
  (`00-decisions.md:238`). Cursor encodes `(created_at, id)` so it's
  stable across inserts. The wire shape:
  ```json
  {
    "items": [...],
    "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNS0wNVQwMDowMDowMFoiLCJpZCI6Ii4uLiJ9",
    "has_more": true
  }
  ```
  Clients pass `?cursor=...&limit=N`.
- **Idempotency:** `Idempotency-Key: <client-supplied UUID>` header on
  every POST that creates a resource. Server caches the
  (org-scoped) request hash + response for 24h; replays return the
  cached response. Mismatched body for the same key returns 409.
- **Auth:**
  - **Browsers:** session cookie set by `/v1/auth/privy/callback`,
    HttpOnly, Secure, SameSite=Strict, signed by
    `secrets["session.cookie_hmac_key"]`. Cookie value =
    `<session_id>.<HMAC-SHA256(session_id, key)>`.
  - **Machines:** `Authorization: Bearer dst_<base32>`. Token's
    `hash` (sha256 of full secret) looked up in `api_tokens`.
- **Error shape:**
  ```json
  {
    "error": {
      "code": "membership.role.demote_self",
      "message": "Admins cannot demote themselves; ask another admin.",
      "request_id": "01HXY...",
      "details": { "membership_id": "..." }
    }
  }
  ```
  HTTP status: 4xx for client problems, 5xx for ours, 402 for "billing"
  (placeholder; v1 has no billing layer), 409 for idempotency
  conflicts, 429 for rate-limit/quota. We do **not** distinguish 401
  vs. 403 vs. 404 to avoid existence leaks: cross-org access returns
  404.
- **Rate limits:** per token + per IP at the gateway
  (`00-decisions.md:239`). The control plane reports
  `X-RateLimit-Remaining` / `X-RateLimit-Reset` based on the gateway's
  bucket headers when present.
- **All bodies are JSON** with strict `Content-Type: application/json`.
  The OCI registry routes are the exception: those use the OCI v2
  binary contract.

### 4.2 Auth

```
POST   /v1/auth/privy/login        # browser-initiated; redirects to Privy
GET    /v1/auth/privy/callback     # Privy redirects here; sets session cookie
POST   /v1/auth/logout             # clears session cookie + revokes Privy session
GET    /v1/auth/session            # current session info; 401 if none
GET    /v1/auth/sessions           # list active sessions for this user
DELETE /v1/auth/sessions/:id       # revoke one
POST   /v1/auth/privy/webhook      # Privy server-side webhook (user deleted etc.)
```

Login does not return a token in the body — only a cookie. Machines
never go through this path; they use API tokens directly.

`GET /v1/auth/session` response:

```json
{
  "user": {
    "id": "01H...",
    "privy_did": "did:privy:...",
    "display_name": "Dan",
    "email": "dan@example.com",
    "is_platform_admin": false,
    "primary_wallet": {
      "id": "01H...",
      "address": "0xabc...",
      "is_embedded": true
    },
    "wallets": [...]
  },
  "expires_at": "2026-05-12T..."
}
```

### 4.3 Organizations

```
POST   /v1/orgs                     # create non-personal org (admin role)
GET    /v1/orgs                     # list orgs the caller is a member of
GET    /v1/orgs/:slug               # org detail
PATCH  /v1/orgs/:slug               # rename (slug or name) — admin only
DELETE /v1/orgs/:slug               # soft-delete; blocked if any non-removed CVMs (00-decisions.md:281-283)

# Personal org is auto-created on first signin. Same shape; UI just
# hides the "create org" affordance for the personal org and shows a
# "personal" badge.
```

`POST /v1/orgs` body:
```json
{ "slug": "acme", "name": "Acme Corp" }
```
201 response:
```json
{
  "id": "01H...",
  "slug": "acme",
  "name": "Acme Corp",
  "is_personal": false,
  "sandbox_uid": 60001,
  "created_at": "2026-05-05T..."
}
```

### 4.4 Members + invitations

```
GET    /v1/orgs/:slug/members
GET    /v1/orgs/:slug/members/:user_id
PATCH  /v1/orgs/:slug/members/:user_id   { role }    # triggers Safe addOwner / removeOwner
DELETE /v1/orgs/:slug/members/:user_id              # admin-only; blocks last-admin removal

POST   /v1/orgs/:slug/invitations    { email, role }
GET    /v1/orgs/:slug/invitations
DELETE /v1/orgs/:slug/invitations/:id
POST   /v1/invitations/:token/accept                # invitee, post Privy signin
POST   /v1/invitations/:token/decline
```

The role-change semantics live in §5 (auth flow). The DELETE on a
member with role=admin returns 202 with a `pending_removal` flag and a
link to the resulting `safe_transactions/:id`. The off-chain row stays
until the Safe `removeOwner` confirms; UI shows the "Pending removal"
state from `memberships.pending_removal`.

### 4.5 API tokens

```
POST   /v1/orgs/:slug/tokens      { name, role, expires_at }
GET    /v1/orgs/:slug/tokens
PATCH  /v1/orgs/:slug/tokens/:id  { name }
DELETE /v1/orgs/:slug/tokens/:id  # revoke
```

`POST` response includes the **plaintext token exactly once**:
```json
{
  "id": "01H...",
  "name": "ci-deploy",
  "role": "admin",
  "token": "dst_NJ4Q3K8H2VWZ...",
  "created_at": "2026-05-05T..."
}
```
Subsequent reads return only the prefix (`dst_NJ4Q3K8...`), never the
full secret.

### 4.6 CVMs

```
GET    /v1/orgs/:slug/cvms                          # paginated, filterable by status
POST   /v1/orgs/:slug/cvms                          # create (admin/viewer can read; admin/admin-equivalent creates)
GET    /v1/orgs/:slug/cvms/:id
PATCH  /v1/orgs/:slug/cvms/:id                      # update compose/env/ports/resources
DELETE /v1/orgs/:slug/cvms/:id                      # remove

POST   /v1/orgs/:slug/cvms/:id/start
POST   /v1/orgs/:slug/cvms/:id/stop                 { graceful: bool }
POST   /v1/orgs/:slug/cvms/:id/restart

GET    /v1/orgs/:slug/cvms/:id/logs                 # SSE stream; ?follow, ?lines, ?ch=serial|stdout|stderr
GET    /v1/orgs/:slug/cvms/:id/events               # paginated; from in-DB events
GET    /v1/orgs/:slug/cvms/:id/info                 # proxied guest-agent Info (vmm/src/guest_api_service.rs:36-47)
GET    /v1/orgs/:slug/cvms/:id/network              # proxied NetworkInfo
GET    /v1/orgs/:slug/cvms/:id/containers           # proxied ListContainers
```

`POST /v1/orgs/:slug/cvms` body:
```json
{
  "name": "my-app",
  "image_id": "01H...",
  "compose": "<json string of the AppCompose>",
  "encrypted_env": "<base64 X25519+AES-GCM blob, optional>",
  "user_config": "<string, optional>",
  "vcpu": 4,
  "memory_mb": 8192,
  "disk_size_gb": 50,
  "gpus": { "attach_mode": "listed", "gpus": [...] },
  "port_map": [
    { "address": "0.0.0.0", "protocol": "tcp", "from": 8080, "to": 80 }
  ],
  "kms_urls": ["https://kms.dstack.cloud:8443"],
  "gateway_urls": ["https://gateway.dstack.cloud:8090"],
  "no_tee": false,
  "hugepages": false,
  "pin_numa": false,
  "networking": { "mode": "user" },
  "stopped": false
}
```
202 response:
```json
{
  "id": "01H...",
  "name": "my-app",
  "app_id": "ab12cd34ef56...",   // truncate40(sha256(org_id || compose))
  "observed_state": "provisioning",
  "operation_id": "01H..."        // poll for progress
}
```

`GET /v1/orgs/:slug/cvms/:id` returns a `Cvm` object with all the
declared fields plus `observed_state`, `cid`, `instance_id`,
`boot_progress`, `boot_error`, `app_url` (computed from
`gateway_urls[0] + base_domain`), `last_seen_at`, etc. The shape mirrors
today's `VmInfo` (`vmm/rpc/proto/vmm_rpc.proto:11-40`) minus the
runtime-only fields (PTY paths, supervisor PIDs).

### 4.7 Images

```
GET    /v1/images                   # list global catalog (anyone signed in)
GET    /v1/images/:id
POST   /v1/admin/images             # platform-admin only; register a new entry
DELETE /v1/admin/images/:id         # platform-admin only; checks no CVM uses it
```

The image catalog stays global, operator-managed (`00-decisions.md:215-216`).

### 4.8 OCI registry

The registry endpoint is on a separate hostname
(`registry.<base>.tld`), but it's the same control-plane process. OCI
v2 distribution spec endpoints, all under `/v2/`:

```
GET    /v2/                                            # the "version check"
GET    /v2/<org-slug>/<repo>/manifests/<reference>
HEAD   /v2/<org-slug>/<repo>/manifests/<reference>
PUT    /v2/<org-slug>/<repo>/manifests/<reference>
DELETE /v2/<org-slug>/<repo>/manifests/<reference>
GET    /v2/<org-slug>/<repo>/blobs/<digest>
HEAD   /v2/<org-slug>/<repo>/blobs/<digest>
DELETE /v2/<org-slug>/<repo>/blobs/<digest>
POST   /v2/<org-slug>/<repo>/blobs/uploads/             # start session
PATCH  /v2/<org-slug>/<repo>/blobs/uploads/<uuid>      # chunked upload
PUT    /v2/<org-slug>/<repo>/blobs/uploads/<uuid>?digest=<d>
DELETE /v2/<org-slug>/<repo>/blobs/uploads/<uuid>
GET    /v2/<org-slug>/<repo>/tags/list
```

Plus the bearer-token endpoint:

```
GET    /v2/auth/token?service=registry.<base>.tld&scope=repository:acme/web:pull
```
(See §8 for token semantics.)

User-facing convenience routes (off the OCI spec, on `api.<base>.tld`):

```
GET    /v1/orgs/:slug/registry/repos
POST   /v1/orgs/:slug/registry/repos    { name }     # create empty repo
DELETE /v1/orgs/:slug/registry/repos/:repo_name
GET    /v1/orgs/:slug/registry/repos/:repo_name/tags
GET    /v1/orgs/:slug/registry/repos/:repo_name/manifests/:digest
DELETE /v1/orgs/:slug/registry/repos/:repo_name/tags/:tag
```

### 4.9 Audit

```
GET    /v1/orgs/:slug/audit             # paginated; org admins can see their org's slice
GET    /v1/admin/audit                  # platform admin; full feed
GET    /v1/orgs/:slug/audit/export      # JSON / CSV download (admin only)
GET    /v1/admin/audit/export           # JSON / CSV download (platform admin)
GET    /v1/admin/audit/batches          # signed batches listing
GET    /v1/admin/audit/batches/:id      # batch + included row range
GET    /v1/admin/audit/batches/:id/export  # JSON download with rows + signature
```

Org admins can export only their own slice (`00-decisions.md:181-182`).

### 4.10 Quotas

```
GET    /v1/orgs/:slug/quotas                # current quotas + usage
POST   /v1/admin/orgs/:slug/quotas/overrides   # platform-admin only
GET    /v1/admin/orgs/:slug/quotas/overrides
DELETE /v1/admin/orgs/:slug/quotas/overrides/:id
```

`GET /v1/orgs/:slug/quotas`:
```json
{
  "limits": {
    "max_cvms": 5, "max_vcpu": 16, "max_memory_mb": 32768,
    "max_disk_gb": 200, "max_ingress_ports": 32,
    "max_image_storage_mb": 10240, "max_deploys_per_hour": 30,
    "soft_pct": 80
  },
  "usage": {
    "cvms": 2, "vcpu": 6, "memory_mb": 12288,
    "disk_gb": 100, "ingress_ports": 4,
    "image_storage_mb": 2400, "deploys_last_hour": 3
  },
  "overrides": [
    {
      "id": "01H...",
      "dimension": "max_vcpu",
      "value": 64,
      "reason": "support ticket #1234",
      "granted_by": { "id": "01H...", "display_name": "Operator" },
      "expires_at": null
    }
  ]
}
```

### 4.11 Platform admin

```
GET    /v1/admin/orgs                              # all orgs, including suspended
PATCH  /v1/admin/orgs/:slug/suspension             { suspended: bool, reason }
GET    /v1/admin/users
PATCH  /v1/admin/users/:id/suspension              { suspended: bool, reason }
GET    /v1/admin/secrets                           # name + description only
PUT    /v1/admin/secrets/:name                     { value }
GET    /v1/admin/recovery-signer                   # status: enabled, address
POST   /v1/admin/recovery-signer/enable
POST   /v1/admin/recovery-signer/intent            { action, target }
POST   /v1/admin/recovery-signer/execute/:id       # 24h after intent (00-decisions.md:106-108)
GET    /v1/admin/metrics-config                    # OTLP endpoint config
GET    /metrics                                    # Prometheus, behind platform-admin auth
GET    /healthz                                    # liveness; no auth
GET    /readyz                                     # readiness; no auth
```

### 4.12 On-chain (Safe transaction queue)

```
GET    /v1/orgs/:slug/safe                          # safe address, threshold, signers
GET    /v1/orgs/:slug/safe/transactions             # all proposed/signed/submitted
GET    /v1/orgs/:slug/safe/transactions/:id
POST   /v1/orgs/:slug/safe/transactions             { intent, params }   # propose
POST   /v1/orgs/:slug/safe/transactions/:id/sign    { signature }        # admin signs via Privy
POST   /v1/orgs/:slug/safe/transactions/:id/submit                       # broadcast once threshold met
```

The intent set in §3 (`safe_transactions.intent`) maps to known Safe
operations. The control plane builds `calldata`, computes
`safe_tx_hash`, and stores the tx; the admin's Privy embedded wallet
signs the hash; we collect signatures into `safe_signatures` until
threshold (always 1 in v1); then `submit` calls
`Safe.execTransaction(...)` via the EVM RPC.

---

## 5. Auth flow

### 5.1 Privy sign-in → session cookie

```
Browser                  control-plane                 Privy
   |                          |                           |
   |  GET /                   |                           |
   |------------------------->|                           |
   |  302 → Privy hosted      |                           |
   |  /v1/auth/privy/login    |                           |
   |<-------------------------|                           |
   |                                                      |
   |  user authenticates at Privy (email/SMS/OAuth +      |
   |  embedded wallet provisioned)                        |
   |--------------------------------------------------->  |
   |                                                      |
   |  302 → /v1/auth/privy/callback?code=... + state      |
   |<---------------------------------------------------- |
   |                          |                           |
   |  GET /v1/auth/privy/callback?code=...                |
   |------------------------->|                           |
   |                          |  POST /sessions/exchange  |
   |                          |  + privy.app_id/secret    |
   |                          |-------------------------->|
   |                          |  { user_did, wallets[] }  |
   |                          |<--------------------------|
   |                          |                           |
   |                          |  upsert users row by      |
   |                          |  privy_did                |
   |                          |  upsert user_wallets      |
   |                          |  if first user ever:      |
   |                          |    is_platform_admin=true |
   |                          |  if no personal org:      |
   |                          |    create personal org    |
   |                          |  match invitations.email  |
   |                          |    accept, create         |
   |                          |    membership rows        |
   |                          |  insert sessions row      |
   |                          |  audit auth.signin        |
   |  Set-Cookie:             |                           |
   |   dstack_session=...     |                           |
   |   HttpOnly; Secure;      |                           |
   |   SameSite=Strict        |                           |
   |  302 → /                 |                           |
   |<-------------------------|                           |
```

Subsequent requests carry the cookie. The middleware verifies the HMAC,
loads the session row, joins to user, and constructs an
`AuthContext { user_id, is_platform_admin, primary_wallet, sessions_id }`
that flows into every handler.

### 5.2 API token use

```
Client                       control-plane
  |                                |
  | POST /v1/orgs/acme/cvms        |
  | Authorization: Bearer dst_...  |
  |------------------------------->|
  |                                | sha256(token) → api_tokens.hash
  |                                | check revoked_at IS NULL
  |                                | check expires_at IS NULL OR > now()
  |                                | UPDATE api_tokens SET last_used_at = now()
  |                                | construct AuthContext {
  |                                |   token_id, organization_id, role,
  |                                |   actor_user_id = api_tokens.created_by_user
  |                                | }
  |                                | role check (admin can create CVM)
  |                                | quota admission
  |                                | ... handler ...
  |                                | audit cvm.create with actor_token_id
  | 202 Accepted                   |
  |<-------------------------------|
```

### 5.3 Admin role change → Safe reconfiguration via Privy embedded wallet

The decision rule from `00-decisions.md:84-91` is: **for admin#2 added,
the Safe `addOwnerWithThreshold` proposal is signed first; off-chain DB
role change happens only after on-chain confirm. For removing an admin,
the Safe `removeOwner` is proposed and signed first; the off-chain row
is then deleted.**

```
Admin1's browser              control-plane          EVM RPC          Safe (on-chain)
    |                              |                    |                   |
    | (Admin1 wants to add Admin2)                      |                   |
    | PATCH /v1/orgs/acme/members/:user2 { role:"admin" }                   |
    |----------------------------->|                    |                   |
    |                              | RBAC: admin1 is admin? yes             |
    |                              | check: target is current member        |
    |                              | check Safe exists; if not, deploy:     |
    |                              |   build Safe deployment calldata       |
    |                              |   (initial signers = [admin1.wallet],  |
    |                              |    threshold = 1)                      |
    |                              |   propose safe_transactions row        |
    |                              |   intent="deploy"                      |
    |                              | else: existing safe_address loaded     |
    |                              |                                        |
    |                              | build addOwnerWithThreshold calldata:  |
    |                              |   target = safe_address                |
    |                              |   data = encode(addOwnerWithThreshold( |
    |                              |          user2.wallet, threshold=1))   |
    |                              | compute safe_tx_hash                   |
    |                              | INSERT safe_transactions               |
    |                              |   (intent='add_owner', status='proposed', ...)
    |                              | UPDATE memberships SET                 |
    |                              |   pending_role_change='admin'           |
    |                              |   WHERE user2 ...                      |
    |                              | audit safe.transaction.proposed        |
    | 202 Accepted                 |                                        |
    | { safe_transaction_id, ... } |                                        |
    |<-----------------------------|                                        |
    |                                                                       |
    | (UI reload shows pending Safe action and prompts admin1 to sign)      |
    |                                                                       |
    | POST /v1/orgs/acme/safe/transactions/:id/sign-request                 |
    |----------------------------->|                                        |
    |                              | look up safe_tx_hash                   |
    |                              | call Privy signTypedData via app_secret|
    |                              |   payload = EIP-712 SafeTx struct      |
    |                              |   wallet  = admin1.primary_wallet      |
    |                              |   user_id = admin1.privy_did           |
    | 200 { signing_url }          |                                        |
    | (redirects browser through  Privy embedded UI for confirmation)       |
    |<-----------------------------|                                        |
    |                                                                       |
    | (admin1 confirms in Privy modal)                                      |
    |                                                                       |
    | POST /v1/orgs/acme/safe/transactions/:id/sign  { signature }          |
    |----------------------------->|                                        |
    |                              | verify signature recovers admin1.wallet|
    |                              | INSERT safe_signatures                 |
    |                              | check threshold (1 v1; always met)     |
    |                              | UPDATE safe_transactions               |
    |                              |   SET status='signed'                  |
    |                              | enqueue submission                     |
    |                              |                                        |
    |                              | POST eth_sendRawTransaction            |
    |                              | (Safe.execTransaction(...))            |
    |                              |------------------->|                   |
    |                              |                    |                   |
    |                              | wait 2 confirmations                   |
    |                              |<------- mined --- |                   |
    |                              | UPDATE safe_transactions               |
    |                              |   SET status='confirmed',              |
    |                              |       block_number, submitted_tx_hash  |
    |                              | UPDATE memberships SET                 |
    |                              |   role='admin',                        |
    |                              |   pending_role_change=NULL             |
    |                              | refresh safes.signers_cache            |
    |                              | audit                                  |
    |                              |   safe.transaction.confirmed,          |
    |                              |   membership.role.change               |
```

The `add_owner` flow makes the off-chain DB role change *after* the
on-chain confirm. The same mechanism handles `change_threshold` if a
future version introduces M-of-N.

### 5.4 Admin removal flow with "Pending removal" state

```
Admin1                     control-plane          Safe          DB
   |                            |                   |             |
   | DELETE /members/:user2     |                   |             |
   |--------------------------->|                   |             |
   |                            | role check        |             |
   |                            | last-admin guard  |             |
   |                            | (refuses if user2 |             |
   |                            |  is the only one) |             |
   |                            | build removeOwner |             |
   |                            | calldata          |             |
   |                            | compute safe_tx_h |             |
   |                            | INSERT safe_transactions intent='remove_owner'
   |                            | UPDATE memberships|             |
   |                            |   SET pending_removal=true       |
   |                            |   WHERE user2 ... |             |
   |                            | audit            |             |
   |                            |  safe.tx.proposed,|             |
   |                            |  membership.removal.requested    |
   | 202 + safe_transaction_id  |                   |             |
   |<---------------------------|                   |             |
   |                                                              |
   | (UI shows user2 with "Pending removal" badge.                |
   |  user2 still has DB membership but the UI denies              |
   |  destructive actions because pending_removal=true.)           |
   |                                                              |
   | (admin1 signs via the same Privy flow as §5.3)               |
   |                                                              |
   | POST /safe/transactions/:id/sign { sig }                     |
   |--------------------------->|                   |             |
   |                            | submit            |             |
   |                            |------------------>|             |
   |                            |                   |  (mined)    |
   |                            |<------------------|             |
   |                            | DELETE FROM memberships         |
   |                            |   WHERE user2 ...              |
   |                            | audit                          |
   |                            |  safe.tx.confirmed,            |
   |                            |  membership.removal.completed  |
```

Refusing to remove the last admin is a server-side guard backed by:
```sql
SELECT COUNT(*) FROM memberships
WHERE organization_id = $1 AND role = 'admin' AND pending_removal = false;
```
which must be ≥ 2 before we accept a removal request. If 1, return
409 with `code = "membership.last_admin_remove_blocked"` and a hint
that another admin must be promoted first.

The "Pending removal" badge is driven by `memberships.pending_removal`.
While true, the affected user can read the org but cannot sign new Safe
proposals (we exclude them from `safes.signers_cache`-derived UI
checks; on-chain they still are a signer, but in v1 with threshold=1
this is fine).

---

## 6. On-chain integration

### 6.1 Stack

- **Chain:** Base mainnet (chain ID 8453)
  (`00-decisions.md:80-83`). The operator can configure their own
  EVM-compatible RPC endpoint via `secrets["evm.rpc_url"]` —
  Sepolia for testing, a self-hosted Base node, or a third-party
  service like Alchemy/Infura.
- **EVM client (server-side, Rust):** `alloy 0.8.x` (the Rust port of
  ethers/viem). Use `alloy::providers::ProviderBuilder` to construct
  the JSON-RPC provider; use `alloy::sol!` to bind contract ABIs.
- **Safe SDK choice:** there is no first-party Rust Safe SDK. We do not
  bring in a Python / JS toolchain just for Safe. Instead we **encode
  Safe transactions ourselves** using the published Safe ABI (the
  `Safe` v1.4 contract has stable ABI). The encoding work is small —
  computing the EIP-712 `SafeTx` typed-data hash, building
  `execTransaction(...)` calldata, and the
  `addOwnerWithThreshold` / `removeOwner` / `changeThreshold` selectors
  — and lives in `crates/dstack-onchain/src/safe.rs`. The module
  vendors the relevant Safe contract ABI snippets via `alloy::sol!` and
  has unit tests against the same `safe_tx_hash` that the JS Safe SDK
  produces (golden-file tests using fixtures from a small helper script
  `tests/fixtures/safe-fixtures.mjs` that runs Safe's TypeScript SDK
  once at test-fixture-write time, then never again).
- **Privy for client signing:** the browser uses Privy's embedded
  wallet to sign EIP-712 messages. The control plane sends the wallet
  the structured `SafeTx` payload via the Privy JS SDK; the embedded
  wallet returns a signature; the browser POSTs it back. The server
  never holds a user's private key.

### 6.2 Contracts deployed

We continue to use the existing contracts (`kms/auth-eth/contracts/`):

- **`DstackKms`** — the parent registry. Deployed once on Base by the
  operator at install time. Its address is stored in the `secrets` table
  under `name = "kms.contract_address"`.
- **`DstackApp`** (proxy + impl) — one per `app_id`. Deployed lazily
  when an org first creates a CVM with KMS-backed env encryption
  (`AppCompose.key_provider == "kms"`). Owner of each `DstackApp` is
  the org's Safe (`safes.safe_address`).

We do **not** modify the Solidity. The `app_id` collision fix is
**off-chain only** (`00-decisions.md:96-100`):

```text
app_id = truncate40(sha256(org_id || compose))
```

(`org_id` is the UUID v7 string form; `compose` is the canonical JSON
serialisation from `dstack-types` — the existing
`docs/normalized-app-compose.md` rules.) This is computed by
`crates/dstack-control-plane/src/app_id.rs`. The 40-char hex string is
what we store in `cvms.app_id` and what we pass to KMS in
`GetAppEnvEncryptPubKey` (proto: `kms_rpc.proto:AppId`).

The existing `DstackKms.registerApp` / `deployAndRegisterApp`
(`kms/auth-eth/contracts/DstackKms.sol:130-167`) accepts whatever
`app_id` we pass; the `truncate40(sha256(...))` derivation lives entirely
client-side and gives perfect cross-org separation
(`00-decisions.md:96-100`).

### 6.3 Lazy Safe deployment

A new org has `safes.safe_address = NULL`. The first time the org
needs an on-chain action (e.g. first CreateVm with `key_provider = kms`,
or admin role change), the control plane:

1. Builds a `SafeProxyFactory.createProxyWithNonce(...)` calldata,
   passing the org's current admin wallets as initial signers and
   `threshold = 1`. If `recovery_signer` is enabled, append the
   recovery wallet to the initial signers as well (then signers count
   is N+1, threshold still 1).
2. Inserts a `safe_transactions` row with `intent = 'deploy_safe'`,
   `status = 'submitted'`, `proposed_by = admin1`. This proposal is
   *self-signed* by the operator (via the platform recovery key) when
   the recovery signer is enabled, or by the proposing admin when it
   isn't. Either way the Safe's first signer is among the org admins.
3. After the deploy tx is mined, populates `safes.safe_address` with
   the predicted address (deterministic via CREATE2 salt).
4. Triggers the original action that needed the Safe.

### 6.4 Recovery-signer 24h timelock

Per `00-decisions.md:101-108`, the recovery signer is opt-in at install
time. When enabled:

- A 32-byte secp256k1 key is generated at install
  (`secrets["recovery.signer_key"]`) and printed to the operator with a
  prominent "back this up" warning.
- The recovery signer's address is added to every new org Safe as the
  N+1 signer.
- For *use*, every recovery action is gated by a documented **24-hour
  on-chain timelock**.

Implementation: a simple "intent" record published to a public
`RecoveryQueue` contract (deployed once on Base), with a
`canExecute(intent, timestamp)` view that returns true 24h after
publication. The control plane:

```
POST /v1/admin/recovery-signer/intent
  { action: "force_remove_owner", target_org: "acme",
    target_address: "0x...", reason: "support ticket" }
  → calls RecoveryQueue.publishIntent(hash) on-chain
  → records intent in `recovery_intents` table

(24 hours later)

POST /v1/admin/recovery-signer/execute/:id
  → checks RecoveryQueue.canExecute(hash, ...) on-chain
  → builds and signs the actual Safe action with recovery key
  → submits
```

The control plane refuses to execute before 24h have elapsed both
on-chain and in-DB. The `RecoveryQueue` contract is small (~40 lines of
Solidity, deployed once) and is part of the install artifacts. The
contract source lives at `crates/dstack-onchain/contracts/RecoveryQueue.sol`.

The `recovery_intents` table:

```sql
CREATE TABLE recovery_intents (
    id              uuid PRIMARY KEY,
    intent_hash     bytea NOT NULL CHECK (octet_length(intent_hash) = 32),
    action          text NOT NULL,
    target_org      uuid REFERENCES organizations(id),
    payload         jsonb NOT NULL,
    reason          text NOT NULL,
    publish_tx_hash bytea CHECK (octet_length(publish_tx_hash) = 32),
    published_at    timestamptz,
    executed_at     timestamptz,
    executed_tx_hash bytea CHECK (octet_length(executed_tx_hash) = 32),
    created_by      uuid NOT NULL REFERENCES users(id),
    created_at      timestamptz NOT NULL DEFAULT now()
);
```

Default state of recovery signer = **off** (`00-decisions.md:108`).

---

## 7. CVM lifecycle in the new world

### 7.1 Deploy flow

```
1. POST /v1/orgs/acme/cvms { ... }
2. AuthN/Z middleware verifies session/token; loads AuthContext.
3. RBAC check: role >= admin (only admins can deploy).
4. Compute org_id from slug; load org row.
5. Validate request body:
   - parse compose JSON via dstack_types::AppCompose deserialize
   - validate name charset (vmm/src/main_service.rs:58-69 logic)
   - validate port_map vs. operator-configured allowed ranges
   - validate gpus vs. operator-configured allow_list
6. Compute app_id = truncate40(sha256(org_id_str || compose_normalized)).
7. BEGIN transaction.
8. Quota admission (§10): SELECT-and-compare against limits.
9. INSERT cvms row with desired_state='running', observed_state='pending',
                     workdir = '/var/lib/dstack/vm/<id>'.
10. INSERT cvm_artifacts (app_compose v1, encrypted_env v1, user_config v1,
                          instance_info v1 with just app_id).
11. If key_provider = kms and there's no kms_apps row for app_id:
    - propose Safe transaction { intent='create_app',
        target=DstackKms, data=encode(deployAndRegisterApp(...)) }
    - the request returns 202 with safe_transaction_id; the actual
      provisioning waits until the Safe action confirms.
12. Else:
    - INSERT host_task { kind='provision_cvm', cvm_id }
13. INSERT audit_log row(action='cvm.create', target_id=cvm.id, ...).
14. COMMIT.
15. Background: host worker picks up host_task:
    a. Create workdir (/var/lib/dstack/vm/<id>/), set owner = sandbox_uid.
    b. Write shared/ files from cvm_artifacts: app-compose.json,
       .encrypted-env, .user-config, .instance_info, .sys-config.json.
       (matches today's prepare_work_dir vmm/src/app.rs:964-994)
    c. UPDATE cvms SET observed_state='provisioning'.
    d. allocate CID from BTreeSet (§7.4).
    e. Build QEMU args (port from vmm/src/app/qemu.rs:388-771):
       - sudo -u dstack-org-<slug>
       - qemu-system-x86_64 ... per existing logic ...
       - -object tdx-guest,...,mrconfigid=<base64>,quote-generation-socket=...
    f. supervisor.deploy(process_config) → spawn QEMU.
    g. UPDATE cvms SET observed_state='starting', cid=<cid>,
                     started_at=now().
    h. (host_api Notify events from guest will drive
        observed_state='running'.)
```

The host_task table:

```sql
CREATE TABLE host_tasks (
    id               uuid PRIMARY KEY,
    cvm_id           uuid REFERENCES cvms(id) ON DELETE CASCADE,
    kind             text NOT NULL CHECK (kind IN
        ('provision_cvm','start_cvm','stop_cvm','remove_cvm',
         'update_cvm','pull_image')),
    payload          jsonb NOT NULL DEFAULT '{}',
    status           text NOT NULL CHECK (status IN
        ('pending','claimed','succeeded','failed','cancelled')),
    claimed_by       text,                                          -- worker thread id
    claim_expires_at timestamptz,
    attempts         int NOT NULL DEFAULT 0,
    last_error       text,
    created_at       timestamptz NOT NULL DEFAULT now(),
    updated_at       timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX host_tasks_dispatch_idx
    ON host_tasks (status, created_at)
    WHERE status IN ('pending','claimed');
```

Workers `SELECT ... FOR UPDATE SKIP LOCKED` to claim. Single-process v1
means one worker pool, but the table is still the right shape (and
makes the recovery story trivial).

### 7.2 DB ↔ disk relation

| Concern | DB | Disk |
|---------|-----|------|
| What CVMs exist | `cvms` rows | (no on-disk inventory file) |
| Compose / env / user-config | `cvm_artifacts` rows | Re-materialised into `<workdir>/shared/*` before each start |
| Instance ID + app ID | `cvms.instance_id`, `cvms.app_id` | `<workdir>/shared/.instance_info` (re-materialised) |
| qcow2 disk | (path in `cvms.workdir`) | `<workdir>/hda.img` (qemu-img backing chain to image's hda) |
| KMS / gateway URLs | `cvms.kms_urls`, `cvms.gateway_urls` | Generated on every start into `<workdir>/shared/.sys-config.json` (matches today's `make_sys_config`, `vmm/src/app.rs:1139-1167`) |
| Running state | `cvms.observed_state` | `qemu.pid` (informational only, not parsed) |
| CID | `cvms.cid` | (not on disk; supervisor reports it) |
| Logs | (not in DB; streamed only) | `<workdir>/serial.log`, `serial.history.log`, `stdout.log`, `stderr.log` (unchanged from today) |
| `.removing` marker | Replaced by `cvms.observed_state = 'deleting'` + `host_tasks.kind = 'remove_cvm'` | Marker file is gone |

### 7.3 Per-org sandbox uid + (optional) VLAN provisioning

When a non-personal org is created (or on the operator-configurable
"first deploy" trigger for personal orgs), a one-shot helper invokes:

```
sudo /usr/lib/dstack/provision-org.sh <slug> <uid> [<vlan_id>]
```

`provision-org.sh` (operator-installed, in the same package):
- `useradd dstack-org-<slug> -u <uid> -M -s /usr/sbin/nologin`
- creates `/var/lib/dstack/vm/by-org/<slug>/` (group-readable by `dstack`)
- if `vlan_id` is passed: `ip link add link <bridge> name <bridge>.<vlan_id>
  type vlan id <vlan_id>` and the corresponding bridge binding
- `visudo` snippet allowing `dstack` to `sudo -u dstack-org-<slug>
  qemu-system-x86_64 ...` without password (analog of today's
  `vmm/src/setup-user.sh:106-181`)

The sandbox uid is allocated server-side from a configurable pool
(default `60001..60999`) and stored in `organizations.sandbox_uid`.
Re-running `provision-org.sh` for an existing org is idempotent.

### 7.4 CID allocation

CIDs are still host-local (vsock CID is host-scoped by design). We keep
the `IdPool<u32>` shape from `vmm/src/app/id_pool.rs:24-69` but back it
with a Postgres-resident counter, so two control-plane processes (e.g.
during a rolling upgrade — out of v1 scope but a free win) cannot
collide:

```sql
-- CID allocation is a single atomic SQL:
UPDATE cvms
SET cid = (
    SELECT COALESCE(MIN(unused), :cid_start)
    FROM (
        SELECT generate_series(:cid_start, :cid_start + :pool_size) AS unused
        EXCEPT
        SELECT cid FROM cvms WHERE cid IS NOT NULL AND observed_state <> 'deleted'
    ) t
    WHERE unused < :cid_start + :pool_size
)
WHERE id = $1
RETURNING cid;
```

If the SELECT returns NULL → pool exhausted, surface as 409
`cvm.cid_pool_exhausted`. The pool config (`cid_start`, `pool_size`) is
in `control-plane.toml`.

### 7.5 Recovery on restart

On startup:

1. `sqlx migrate run` (advisory-lock-protected).
2. Decrypt `secrets` rows using the KMK from `/etc/dstack/master.key`.
3. Connect to `dstack-supervisor` (auto-spawn if not present, same as
   today's `vmm/src/main.rs:209-222`).
4. Reconciliation pass — the new `reload_vms`:
   ```
   For each cvms row WHERE desired_state IN ('running','stopped') AND
                          observed_state <> 'deleted':
     a. Re-materialise <workdir>/shared/ from cvm_artifacts (latest
        version of each kind).
     b. Regenerate <workdir>/shared/.sys-config.json.
     c. Query supervisor.list() for running processes.
     d. If supervisor reports running and our cvms.cid matches:
          UPDATE cvms SET observed_state='running'.
        Else if cvms.desired_state = 'running' and supervisor doesn't
        report it:
          enqueue host_task kind='start_cvm'.
     e. If cvms.observed_state = 'deleting' and host_tasks has a
        completed remove_cvm task: finalise (UPDATE observed_state =
        'deleted' + remove workdir).
   ```
5. Start the auto-restart loop (replaces today's `auto_restart_task`
   in `vmm/src/main.rs:132-146`), now driven by `host_tasks` rather
   than a polling scan. Crash-loop circuit breaker: a CVM that has 5
   consecutive failed starts in 10 minutes is marked
   `observed_state = 'failed'` and the operator gets an audit event.
6. Open API listeners only after step 4 completes (readiness gate).

The DB is the source of truth. If an operator does `pg_restore`, the
host re-materialises every workdir from the restored DB state on next
start. The qcow2 disks need to be backed up separately (see §15
"Risks").

---

## 8. OCI registry implementation

### 8.1 Library choice

We **roll our own** OCI v2 distribution server in Rust, because:

- The Go reference (`distribution/distribution`) would require us to
  pull in CGo + a Go runtime just to host registry endpoints. Bad fit
  for our single-binary aim.
- Zot (the Cloud-Native Computing Foundation registry) is also Go.
- The OCI v2 distribution spec is small (~30 endpoints, all
  well-documented), and the registry contracts the rest of the system
  needs (auth, scoped tokens, garbage collection, content addressing)
  are *exactly* what we need to integrate with our org/RBAC model
  anyway. A library would force us to fork/subclass for Privy and
  org-aware auth.

We use **`oci-spec-rs 0.7.x`** for type definitions (manifest schema,
descriptor, etc.) and **`reqwest` only as an outbound client** for the
optional registry-pull-through case (deferred to v2). The server logic
is ~1500 lines of Rust in `crates/dstack-control-plane/src/oci/`.

### 8.2 Auth flow (token-server)

OCI clients (Docker, podman, skopeo) use a bearer-token flow:

```
docker pull registry.dstack.cloud/acme/web:v1
   ↓
1. GET https://registry.dstack.cloud/v2/
   → 401 with WWW-Authenticate: Bearer realm="https://registry.dstack.cloud/v2/auth/token",
                                service="registry.dstack.cloud",
                                scope="repository:acme/web:pull"

2. GET https://registry.dstack.cloud/v2/auth/token?service=...&scope=repository:acme/web:pull
   Authorization: Basic base64(<username>:<dst_token>)
   (the dst_<...> token from §4.5 plays the password role; username is "_")
   →
   200 OK
   {
     "token": "<jwt>",
     "expires_in": 600,
     "issued_at": "2026-05-05T..."
   }

3. GET https://registry.dstack.cloud/v2/acme/web/manifests/v1
   Authorization: Bearer <jwt>
   →
   200 with manifest JSON
```

The "JWT" token is signed with HS256 using a key derived from
`secrets["registry.token_signing_key"]`. Claims:

```json
{
  "iss": "dstack-control-plane",
  "sub": "<api_token_id or user_id>",
  "aud": "registry.<base>.tld",
  "exp": <now + 10 min>,
  "iat": <now>,
  "nbf": <now>,
  "jti": "<uuid>",
  "access": [
    { "type": "repository", "name": "acme/web", "actions": ["pull"] }
  ]
}
```

`access` is the realised scope after enforcing RBAC: an Admin asking
`pull,push` on `acme/web` gets `["pull","push"]`; a Viewer gets
`["pull"]` only; a non-member gets a 403 (we deliberately use 403 here,
not 404, because the org-slug in the URL has already been seen by the
client — no information leak).

### 8.3 Deploy-time tokens for CVMs

When the host worker provisions a CVM whose `app_compose.docker_compose_file`
references private registry images (heuristic: image refs starting with
`registry.<base>.tld/<org-slug>/...`), it generates a short-lived
deploy-time token:

- `aud = "registry.<base>.tld"`
- `sub = "deploy:<cvm_id>"`
- `exp = now + 30 minutes` (enough for cold pulls)
- `access = [{ type: "repository", name: "<org-slug>/<repo>",
              actions: ["pull"] }]` for each repo referenced

This token is written into `<workdir>/shared/.docker-config.json` as a
`auths` entry, mode 0640, so the guest agent's docker-compose runner
picks it up. The token is single-use-ish (it's still a JWT; we don't
record-and-revoke), but its 30-minute expiry plus its narrow scope is
sufficient.

### 8.4 Storage layout and GC

```
/var/lib/dstack/registry/
├── blobs/
│   └── sha256/
│       └── ab/cd/abcd1234.../    # 2-level fan-out by digest hex prefix
│           ├── data              # the actual blob bytes
│           └── refcount          # not used; we query Postgres
└── uploads/
    └── <upload_session_id>/
        └── partial               # streaming-uploaded bytes
```

Two-level hex fan-out (`ab/cd/abcd...`) keeps any single directory
under ~64K entries even with millions of blobs.

**Garbage collection:**

- Nightly cron job (`schedule = '@daily'`, runs at 02:00 local).
- Pseudocode:
  ```
  for each oci_blobs row WHERE on_disk = true:
      referenced =
        EXISTS (
          SELECT 1 FROM oci_manifests
          WHERE manifest_digest = $1
             OR config_digest    = $1
             OR $1 = ANY(layer_digests)
        )
      if not referenced and last_seen_at < now() - interval '7 days':
          rm /var/lib/dstack/registry/blobs/.../data
          UPDATE oci_blobs SET on_disk = false
          audit registry.blob.gc target=<digest>
  ```
- The 7-day grace is so that an in-progress push (manifest not yet
  PUT) doesn't lose its blobs.
- Upload sessions are GC'd separately: `expires_at < now()` →
  `DELETE FROM oci_upload_sessions` + `rm -rf /var/lib/.../uploads/<id>/`.

### 8.5 Push / pull RBAC

| Action | Admin | Billing | Viewer |
|--------|-------|---------|--------|
| pull (GET manifest, GET blob, GET tags/list) | yes | yes | yes |
| push (POST/PATCH/PUT blob upload, PUT manifest) | yes | no | no |
| delete (DELETE manifest, DELETE blob) | yes | no | no |

This mirrors `00-decisions.md:206-213`. Billing/Viewer pull is needed
for CI/CD with limited access.

---

## 9. Audit log + signed batches

### 9.1 Canonical row encoding

Audit rows are hash-chained. The **canonical encoding** for hashing is
deterministic JSON (sorted keys, no whitespace, `"\u..."` escapes only
for control bytes ≤ 0x1F):

```json
{"a":"<action>","aip":"<actor_ip or empty>","aid":"<actor_user_id or empty>","atid":"<actor_token_id or empty>","aua":"<actor_ua or empty>","err":"<error or empty>","id":<id>,"oc":"<occurred_at iso8601 utc>","org":"<org_id or empty>","p":<payload jsonb canonical>,"ph":"<prev_hash hex>","r":"<result>","rid":"<request_id or empty>","tid":"<target_id or empty>","tk":"<target_kind or empty>"}
```

`row_hash` = `sha256(canonical_encoding)`.

The "genesis" row has `prev_hash = 32 × 0x00`. Each subsequent row's
`prev_hash` is the previous row's `row_hash`. The chain is verified by
walking by ascending `id` and recomputing.

The encoding is implemented in
`crates/dstack-audit/src/canonical.rs` with property tests against
JCS (RFC 8785) for compatibility with off-the-shelf verifiers.

### 9.2 Hourly signed batches

A scheduled task runs every hour at `:00`:

```
1. Find max(id) of audit_log → tip_id
2. Find max(last_id) of signed_batches → last_batch_id
3. If tip_id == last_batch_id → no-op
4. Else:
   a. Stream rows where id > last_batch_id AND id <= tip_id, in ascending id.
   b. Compute their row_hashes (already in DB).
   c. Build a binary Merkle tree: leaves are the row_hashes, internal
      nodes are sha256(left || right). Odd levels duplicate the last
      leaf (Bitcoin-style). The root is the merkle_root.
   d. Compute tip_hash = sha256("dstack-audit-tip-v1" || merkle_root
                                 || u64_be(tip_id)
                                 || (last_signed_batch.tip_hash if any else 32×0x00)).
   e. Compute payload = sha256("dstack-audit-batch-v1" || merkle_root
                               || u64_be(first_id) || u64_be(tip_id)
                               || u64_be(unix_seconds_now)).
   f. Sign payload with secp256k1 audit_signing_key from secrets.
   g. INSERT signed_batches.
```

The Merkle structure means a verifier can prove a row is in a batch
with O(log n) hashes. Public publication of batches is **deferred to
v2** (`00-decisions.md:175-176`), but the structure is built so it's
free to add later.

### 9.3 Export endpoint

```
GET /v1/admin/audit/batches/:id/export
```

Returns:
```json
{
  "batch": {
    "id": 42,
    "first_id": 1003,
    "last_id": 1827,
    "merkle_root": "<hex>",
    "signer_pubkey": "<hex>",
    "signature": "<hex>",
    "tip_hash": "<hex>",
    "created_at": "2026-05-05T01:00:00Z"
  },
  "rows": [
    { "id": 1003, "action": "...", "...": "...", "row_hash": "<hex>", "prev_hash": "<hex>" },
    ...
  ]
}
```

A standalone Rust verifier (`dstack-audit-verify`) accepts a batch
JSON and an expected signer pubkey and re-derives the chain + Merkle
tree + signature, returning OK or a precise mismatch reason.

### 9.4 Retention

Hot retention 90 days in the `audit_log` table; older rows are
truncated by a nightly job. Signed batches retain the merkle root +
signature forever (small) so old batches remain verifiable; raw row
bytes are gone after 90 days but the Merkle proof you exported earlier
is still valid (`00-decisions.md:177-179`).

### 9.5 Events captured

Per `00-decisions.md:181-185`, every state-changing operation produces
exactly one audit row. The taxonomy:

```
auth.signin / auth.signout / auth.session.refresh
membership.invite / membership.invite.accept / membership.invite.decline
membership.role.change / membership.remove
cvm.create / cvm.update / cvm.start / cvm.stop / cvm.restart / cvm.remove
cvm.boot.error / cvm.crash
image.add / image.remove
secret.set / secret.unset
api_token.create / api_token.revoke
quota.override.create / quota.override.revoke
safe.transaction.proposed / safe.transaction.signed / safe.transaction.confirmed / safe.transaction.failed
registry.repo.create / registry.repo.delete / registry.manifest.push / registry.manifest.delete / registry.blob.gc
platform_admin.suspend / platform_admin.unsuspend
recovery.intent.publish / recovery.intent.execute
```

Webhooks on these events are deferred to v2 (`00-decisions.md:186`).

---

## 10. Quota enforcement

Quotas are dual-enforced (`00-decisions.md:194-197`):

### 10.1 Admission-time at the API layer

In the `POST /v1/orgs/:slug/cvms` handler, *inside* the same
transaction that inserts the `cvms` row:

```rust
// Pseudocode
let limits = effective_limits(org_id);  // quotas + active overrides
let usage = current_usage(org_id);       // SELECT SUM(...) FROM cvms ...
let req = (request.vcpu, request.memory_mb, request.disk_size_gb, ...);
for dim in [Cvms, Vcpu, MemoryMb, DiskGb, IngressPorts, DeploysPerHour] {
    let proposed = usage[dim] + req[dim];
    if proposed > limits[dim].hard {
        // hard reject
        audit("quota.deploy.denied", { dim, requested: req[dim], hard: limits.hard });
        return Err(QuotaError::Hard(dim, proposed, limits.hard));
    }
    if proposed > limits[dim].soft {
        // soft warn — collect into a list to emit after commit
        soft_warnings.push((dim, proposed, limits.soft));
    }
}
// continue with cvm insert ...
```

The query for `current_usage` is one SELECT per dimension, all hitting
indexed columns:

```sql
SELECT COUNT(*),
       COALESCE(SUM(vcpu), 0),
       COALESCE(SUM(memory_mb), 0),
       COALESCE(SUM(disk_size_gb), 0)
FROM cvms
WHERE organization_id = $1
  AND desired_state <> 'removed';
```

For `max_ingress_ports`: count distinct `(host_addr, host_port)` pairs
across `port_map` jsonb arrays.

For `max_deploys_per_hour`: COUNT cvm.create audit rows in the last
hour for this org.

### 10.2 Allocation-time at the host worker

When the host worker is processing a `provision_cvm` task:

```rust
// double-check just before spawning QEMU
let (host_total_vcpu, host_used_vcpu) = host_resource_state();
if host_used_vcpu + req.vcpu > host_total_vcpu {
    fail_task("host vcpu exhausted");
    return;
}
// repeat for memory_mb, disk
```

This catches the case where two deploys race past the API's optimistic
admission check (e.g. transaction A and B both pass the check before
either inserts) — at the host worker level we have the in-process
single source of truth and can reject. A failed task triggers
`observed_state = 'failed'` plus an audit `cvm.create.host_rejected`,
not a partial CVM.

### 10.3 Soft-warning delivery

Soft-quota crossings (proposed > soft, not hard) emit:

1. **Audit event** `quota.soft.warning` with the dimension and current
   percentage.
2. **In-app notification** — an in-DB `notifications` row owned by
   each org admin/billing user:
   ```sql
   CREATE TABLE notifications (
       id              uuid PRIMARY KEY,
       organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
       user_id         uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
       kind            text NOT NULL,            -- "quota.soft.warning", ...
       payload         jsonb NOT NULL DEFAULT '{}',
       read_at         timestamptz,
       dismissed_at    timestamptz,
       created_at      timestamptz NOT NULL DEFAULT now()
   );
   ```
   The UI shows unread notifications as a banner. The API
   `GET /v1/users/me/notifications` lists them.

There is no email or push delivery in v1; that's a webhook-style
addition deferred to v2.

### 10.4 Override row handling

`effective_limits(org_id)` is implemented as:

```sql
WITH active_overrides AS (
    SELECT dimension, value
    FROM quota_overrides
    WHERE organization_id = $1
      AND revoked_at IS NULL
      AND (expires_at IS NULL OR expires_at > now())
)
SELECT
    COALESCE((SELECT value FROM active_overrides WHERE dimension = 'max_cvms'), q.max_cvms) AS max_cvms,
    COALESCE((SELECT value FROM active_overrides WHERE dimension = 'max_vcpu'), q.max_vcpu) AS max_vcpu,
    -- ... etc
FROM quotas q
WHERE q.organization_id = $1;
```

When more than one override exists for the same dimension (e.g. two
historical overrides), the most recent unrevoked one wins:

```sql
SELECT value FROM quota_overrides
WHERE organization_id = $1 AND dimension = $2
  AND revoked_at IS NULL AND (expires_at IS NULL OR expires_at > now())
ORDER BY created_at DESC
LIMIT 1;
```

Org admins **cannot** create or modify their own overrides — the API
endpoint is `/v1/admin/orgs/:slug/quotas/overrides` and the middleware
checks `is_platform_admin`. Audit captures the platform admin's id +
the reason.

---

## 11. Observability

### 11.1 Prometheus metrics (`/metrics`)

Behind platform-admin auth (the `/metrics` route is mounted on the same
listener but checks for either `is_platform_admin` session or a special
metrics scrape-token in `secrets["metrics.scrape_token"]`).

Series:

```
# CVMs and lifecycle
dstack_cvm_total{state}                          gauge
dstack_cvm_create_total{outcome}                 counter
dstack_cvm_start_total{outcome}                  counter
dstack_cvm_stop_total{outcome}                   counter
dstack_cvm_remove_total{outcome}                 counter
dstack_cvm_update_total{outcome}                 counter
dstack_cvm_crash_total                           counter
dstack_cvm_boot_seconds_bucket{...}              histogram

# Per-org breakdown (for billing dashboards)
dstack_org_cvm_total{org_slug,state}             gauge
dstack_org_resource_used{org_slug,dim}           gauge   # dim: vcpu, memory_mb, ...

# HTTP
dstack_http_requests_total{method,route,status}  counter
dstack_http_request_seconds{method,route}        histogram

# OCI registry
dstack_registry_pulls_total{org_slug,outcome}    counter
dstack_registry_pushes_total{org_slug,outcome}   counter
dstack_registry_blob_bytes{state="present|deleted"}      gauge
dstack_registry_storage_bytes{org_slug}                  gauge

# Audit
dstack_audit_rows_total                          counter
dstack_audit_chain_verified_at_seconds           gauge
dstack_audit_signed_batch_lag_seconds            gauge

# Database
dstack_db_pool_connections{state="idle|active"}  gauge
dstack_db_query_seconds{stmt}                    histogram

# Supervisor / QEMU
dstack_supervisor_request_seconds{op}            histogram
dstack_qemu_processes{status}                    gauge
dstack_cid_pool{state="allocated|free"}          gauge

# Disk
dstack_disk_bytes{role="image|run|registry",path}  gauge

# Build info
dstack_build_info{version,git_rev,rust_version} gauge
```

### 11.2 Structured logs

JSON output by default. Fields on every line:

```
ts                ISO8601 UTC, RFC3339Nano
level             trace|debug|info|warn|error
target            module path
message           free text (lowercase first letter, per CLAUDE.md)
request_id        UUID v7; injected by middleware on every HTTP request
node_id           UUID; the install's stable id from secrets
org_id            UUID; if available
user_id           UUID; if available
token_id          UUID; if a token-bearer
cvm_id            UUID; if scoped to a CVM
operation_id      UUID; for long-running ops
error.type        error type name (Rust)
error.message     error display
error.cause       chained Display of source errors
```

Rotation is via `tracing-appender::rolling` with daily files;
ship-to-OTLP optional.

### 11.3 OpenTelemetry / OTLP

- Off by default.
- Auto-on when `OTEL_EXPORTER_OTLP_ENDPOINT` env var is set
  (`00-decisions.md:248-250`).
- Implementation: `tracing-opentelemetry 0.27.x` +
  `opentelemetry-otlp 0.18.x`.
- Spans emitted for: every HTTP request (`http.server`), every DB
  query, every supervisor RPC, every KMS RPC, every Safe action, every
  CVM lifecycle transition.
- Resource attributes: `service.name=dstack-control-plane`,
  `service.version=<git-rev>`, `dstack.node_id=<uuid>`,
  `dstack.install_id=<uuid>`.
- Sampling defaults to parent-based; `OTEL_TRACES_SAMPLER=...` overrides.

---

## 12. Install / first-run flow

### 12.1 Operator install

The operator runs:

```bash
git clone https://github.com/Dstack-TEE/dstack-control-plane.git
cd dstack-control-plane
sudo ./install.sh
```

`install.sh` (a shell script we ship) does:

1. Verify the host is TDX-capable (`/sys/firmware/tdx/...`), fail with
   a friendly message if not.
2. Verify Docker is installed, install Postgres image (Postgres 16) if
   needed.
3. Create the `dstack` Linux user (uid 60000, in groups `kvm`, `docker`).
4. `useradd -r dstack-org-pool` for the per-org sandbox uids' shared
   group; pre-allocate 999 sandbox uids in `60001..60999`.
5. Create `/var/lib/dstack/{pg,registry/blobs,vm,images}` with correct
   owners.
6. Render `/etc/systemd/system/dstack-postgres.service` (Docker
   container with `--network none`, bind-mount `/run/dstack/postgres.sock`).
7. Render `/etc/systemd/system/dstack-control-plane.service`.
8. Install the `dstack-control-plane` binary at `/usr/sbin/`.
9. Drop the `provision-org.sh` helper at `/usr/lib/dstack/`.
10. Configure sudoers snippet at `/etc/sudoers.d/dstack`.
11. Print "Run `sudo dstack-control-plane init` to finish setup."

### 12.2 First-run: `dstack-control-plane init`

```
dstack-control-plane init
  - Auto-generates KMK if /etc/dstack/master.key doesn't exist:
      dd if=/dev/urandom of=/etc/dstack/master.key bs=32 count=1
      chmod 0600
      chown dstack:dstack
  - PROMINENTLY prints the wall-of-text:

  ╔══════════════════════════════════════════════════════════════════╗
  ║                                                                  ║
  ║   ⚠  BACK UP YOUR MASTER KEY NOW  ⚠                              ║
  ║                                                                  ║
  ║   We just generated a 32-byte master key and wrote it to:        ║
  ║                                                                  ║
  ║       /etc/dstack/master.key                                     ║
  ║                                                                  ║
  ║   This key encrypts every secret in your install — Privy        ║
  ║   creds, EVM RPC keys, audit signing key, recovery signer       ║
  ║   (if enabled).                                                  ║
  ║                                                                  ║
  ║   IF YOU LOSE THIS KEY:                                          ║
  ║     - all encrypted secrets become unrecoverable                 ║
  ║     - audit-batch lineage breaks (a new audit signing key        ║
  ║       starts a new chain)                                        ║
  ║     - the recovery signer (if enabled) is gone                   ║
  ║                                                                  ║
  ║   YOUR USER DATA STAYS — the Postgres DB itself is unaffected.  ║
  ║   You can re-provision Privy + EVM secrets and continue, but    ║
  ║   you will have lost recoverable on-chain authority.            ║
  ║                                                                  ║
  ║   COPY THIS NOW:                                                 ║
  ║                                                                  ║
  ║   <hex of /etc/dstack/master.key>                                ║
  ║                                                                  ║
  ║   Press [Enter] when you have stored it safely.                  ║
  ║                                                                  ║
  ╚══════════════════════════════════════════════════════════════════╝

  - Asks for required secrets:
      Privy app ID:               (paste)
      Privy app secret:           (paste, hidden)
      Privy webhook secret:       (paste, hidden)
      EVM RPC URL:                (paste; e.g. https://base-mainnet.g.alchemy.com/v2/<key>)
      DstackKms contract address: (paste; or "deploy" to deploy a new one now)
      Base domain (e.g. dstack.cloud): (paste)
      SMTP host / port / user / password / from: (paste each, optional)

  - Asks: "Enable platform recovery signer? [y/N]"
      if y:
        generates secp256k1 key
        prints address; warns "back up the master.key — it encrypts this"
        sets has_recovery_signer = true in install metadata

  - Always generates audit_signing_key (secp256k1, encrypted with KMK).

  - Connects to Postgres, runs migrations.

  - Calls EVM RPC eth_chainId to verify connectivity.

  - Optionally seeds the base-image catalog by scraping
    /var/lib/dstack/images/* directories that look like dstack images
    (have metadata.json + digest.txt + kernel + initrd).

  - Prints:
      "Setup complete.
       Open https://api.<base>/ in your browser.
       The first user to sign in becomes the platform admin."

  - Starts the control plane:
      systemctl start dstack-postgres
      systemctl start dstack-control-plane
```

### 12.3 First-user → platform-admin promotion

In the Privy callback handler, after upserting the user:

```sql
WITH first_user AS (
    SELECT NOT EXISTS (SELECT 1 FROM users WHERE id <> $new_user_id) AS is_first
)
UPDATE users
   SET is_platform_admin = true
   WHERE id = $new_user_id AND (SELECT is_first FROM first_user);
```

This is a one-shot promotion (`00-decisions.md:64`). Subsequent
platform admins must be explicitly added by an existing platform admin
via `PATCH /v1/admin/users/:id { is_platform_admin: true }`.

### 12.4 Optional recovery-signer enablement

If the operator answered "n" at install time, `init` skips it. They can
enable it later:

```
sudo dstack-control-plane recovery enable
  - Generates secp256k1 key
  - Stores encrypted in secrets["recovery.signer_key"]
  - Prints the public address
  - Updates install metadata
  - WARNING: existing org Safes are NOT modified retroactively. New
    orgs from this point on get the recovery signer added.
```

If they want to retrofit existing org Safes, that's a manual `addOwner`
Safe operation per org, which the platform admin can drive through the
admin UI.

### 12.5 Base-image catalog seeding

`dstack-control-plane init` looks at `/var/lib/dstack/images/` for
directories with `metadata.json + digest.txt + kernel + initrd`, and
creates an `images` row per match. This handles the existing operator
shape of "extract release tarball into image dir"
(`docs/deployment.md:163`).

For new images post-install:

```
sudo dstack-control-plane image add /path/to/extracted/dstack-0.5.10/
```

CLI helper that does the same INSERT.

---

## 13. Work-breakdown with effort sizing

Sizing key:

- **S** — ~1 week of focused work.
- **M** — 1–4 weeks.
- **L** — 4–10 weeks.
- **XL** — > 10 weeks.

All sizes are for one experienced Rust engineer.

### Milestone 1 — foundation (M0, weeks 1–6)

| # | Work | Size | Justification |
|---|------|------|---------------|
| 1.1 | Cargo workspace + crate layout + CI | S | New crates: `dstack-control-plane` (binary), `dstack-onchain`, `dstack-audit`, `dstack-secrets`, `dstack-host-worker`. |
| 1.2 | Database schema + migrations (sqlx-cli) | M | Every table in §3, all constraints, FKs, indexes. |
| 1.3 | Secrets module (sodium-secretbox + KMK loader) | S | `crates/dstack-secrets/`. Tested in isolation. |
| 1.4 | Privy server-SDK integration | M | No first-party Rust SDK; we wrap `reqwest` calls to Privy's REST API. The pieces: token exchange, session validation, signing-request kickoff, webhook signature verification. |
| 1.5 | Axum server skeleton + auth middleware + RBAC layer | M | Session cookies + bearer tokens. AuthContext extractor. Per-route policy. Idempotency-key handling. Versioning header. |
| 1.6 | OpenAPI spec generation | S | `utoipa` derive macros on routes. Served at `/api-docs`. |

**M0 delivers:** a binary that boots, connects to Postgres, loads
secrets, accepts Privy signin, and serves stub auth/org endpoints.

### Milestone 2 — control plane core (M1, weeks 7–14)

| # | Work | Size | Justification |
|---|------|------|---------------|
| 2.1 | Org / member / invitation / token CRUD | M | Maps cleanly to the schema. Invitation email send via SMTP. |
| 2.2 | Audit + signed batches | M | Canonical encoding, hash-chain row write trigger, hourly batch builder, signature verifier crate, export endpoint. |
| 2.3 | Quotas + override system | M | Effective-limit computation, admission middleware, soft-warning notification path, override CRUD. |
| 2.4 | Notifications table + read/dismiss endpoints | S | Used by quota soft-warnings and admin actions. |
| 2.5 | Observability — Prometheus metrics + JSON logs | S | `metrics-rs 0.23.x` registry, `prometheus 0.13.x` exposition format, `tracing-subscriber` JSON layer. |
| 2.6 | Observability — OTLP wiring | S | Conditional on env var; `tracing-opentelemetry` integration. |
| 2.7 | Health / readiness endpoints | S | `/healthz` always-200; `/readyz` blocks on DB + supervisor + reload completion. |

**M1 delivers:** a multi-tenant control plane that can manage orgs,
members, tokens, audit, quotas — but does not yet schedule CVMs.

### Milestone 3 — on-chain + Safe (M2, weeks 15–22)

| # | Work | Size | Justification |
|---|------|------|---------------|
| 3.1 | `alloy` provider + RPC client + JWT-secret-managed RPC URL | S | Read EVM RPC URL from secrets, build `Provider`. |
| 3.2 | Safe ABI bindings + EIP-712 SafeTx hashing | M | `alloy::sol!` bindings against Safe v1.4 ABI. Golden-file tests against Safe TS SDK fixtures. |
| 3.3 | Safe deploy + addOwner / removeOwner / changeThreshold orchestration | M | The proposal/sign/submit state machine in `safe_transactions`. |
| 3.4 | DstackKms / DstackApp contract bindings + deployment + whitelist mgmt | M | `deployAndRegisterApp` + `addComposeHash` + `addDeviceId` + `setRequireTcbUpToDate` operations as Safe-targeted intents. |
| 3.5 | Privy embedded-wallet signing flow | M | Browser-side: integrate Privy SDK to request EIP-712 signatures; server-side: store signatures, build full Safe `execTransaction` calldata. |
| 3.6 | Recovery signer + RecoveryQueue contract + 24h timelock | M | Solidity (~40 lines), deployment script, intent/execute endpoints, audit. |
| 3.7 | EVM event watcher (cvm.app_id needs to know when DstackApp is ready) | S | Polling subscription on `Initialized` event from new DstackApp instances; also for Safe `ExecutionSuccess` to drive `safe_transactions.status='confirmed'`. |

**M2 delivers:** an org has a real Safe on Base with signers synced to
admins, and on-chain `DstackApp` contracts that auth KMS env-key
requests.

### Milestone 4 — host worker + CVM lifecycle (M3, weeks 23–30)

This is mostly **rewrite/refactor of existing vmm code** rather than
greenfield, because the QEMU command building and supervisor protocol
are well-tested.

| # | Work | Size | Justification |
|---|------|------|---------------|
| 4.1 | Lift QEMU command-building into a library crate (`dstack-host-worker`) | M | Take `vmm/src/app/qemu.rs:388-771` as-is, extract from the Rocket app, parameterise on a CvmConfig struct. |
| 4.2 | host_tasks worker pool | M | `SELECT FOR UPDATE SKIP LOCKED` claim loop, exponential backoff, crash-loop circuit breaker. |
| 4.3 | Workdir materialisation from cvm_artifacts | S | The DB-to-disk projection. Matches `prepare_work_dir` (`vmm/src/app.rs:964-994`). |
| 4.4 | CID pool via Postgres | S | The atomic SQL allocation in §7.4. |
| 4.5 | Supervisor client (preserve existing UDS protocol) | S | Wrap `supervisor/client/src/lib.rs:26-88` behind a trait. |
| 4.6 | host_api on vsock (preserve unchanged) | S | Reuse existing `vmm/src/host_api_service.rs:35-63`. The handler now writes events to DB instead of in-memory map. |
| 4.7 | Reconciliation pass (replaces `reload_vms`) | M | DB-driven reconciliation, no more on-disk manifest reads. |
| 4.8 | Per-org sandbox uid provisioning | S | `provision-org.sh` + INSERT trigger on `organizations`. |
| 4.9 | Auto-restart loop (DB-driven) | S | Replaces `vmm/src/main.rs:132-146` with a host_tasks worker that periodically inserts start_cvm tasks for crashed CVMs. |
| 4.10 | DHCP lease handler (move off public surface) | S | New private Unix socket; replaces `Vmm.ReportDhcpLease` (`vmm/src/main_service.rs:579-582`). |
| 4.11 | Resource usage measurement loop | S | cgroup polling every 30s, INSERT into `resource_usage`. |
| 4.12 | OCI deploy-time token generation | S | When provisioning a CVM whose compose pulls from our private registry. |

**M3 delivers:** end-to-end CVM creation, start, stop, remove flows, all
talking to QEMU via supervisor exactly as today.

### Milestone 5 — OCI registry (M4, weeks 31–40)

| # | Work | Size | Justification |
|---|------|------|---------------|
| 5.1 | Storage layer (blob FS layout + writes) | M | The fan-out + atomic move-into-place + upload sessions. |
| 5.2 | OCI v2 endpoints (manifests, blobs, tags, version-check) | L | ~30 endpoints. The PUT manifest path validates the JSON, dereferences config + layer digests, INSERT/UPSERT manifest + tag. |
| 5.3 | Token-server endpoint (`/v2/auth/token`) + JWT signer | M | `jsonwebtoken 9.x` for HS256 signing; integrate with API token / session auth as the credential. |
| 5.4 | Blob-upload session protocol (POST/PATCH/PUT) | M | Range-style chunked uploads; resume-on-disconnect support. |
| 5.5 | Garbage-collection job | S | Nightly cron + reference query in §8.4. |
| 5.6 | OCI integration tests against `docker push/pull` and `crane` | M | Real CLIs hitting the registry; verify both push and pull round-trip multi-layer images. |

**M4 delivers:** `docker login registry.<base>.tld -u _ -p dst_<token>` +
`docker push/pull` works for an org's repos.

### Milestone 6 — install / first-run / docs (M5, weeks 41–46)

| # | Work | Size | Justification |
|---|------|------|---------------|
| 6.1 | `install.sh` + systemd units + Docker setup | M | Idempotent reinstall. |
| 6.2 | `dstack-control-plane init` interactive setup | M | KMK auto-gen + secrets prompt + sanity checks + base image seeding. |
| 6.3 | `dstack-control-plane recovery enable/disable` | S | |
| 6.4 | `dstack-control-plane image add/remove` | S | Operator-side image catalog management. |
| 6.5 | First-user → platform-admin promotion | S | One-shot SQL on Privy callback. |
| 6.6 | Documentation: install guide, ops runbook, API reference | M | Replace existing `docs/deployment.md`. |

### Milestone 7 — testing + hardening (M6, weeks 47–55)

| # | Work | Size | Justification |
|---|------|------|---------------|
| 7.1 | Unit tests (target ≥ 80% coverage on RBAC/quota/audit/SafeTx) | M | Per-module in each crate. |
| 7.2 | DB integration tests with `testcontainers` Postgres | M | One canonical schema + replay tests for every migration. |
| 7.3 | End-to-end tests (Privy stub + simulated EVM RPC + SDK simulator) | L | Drives the full lifecycle: signin → org create → deploy CVM → stop → remove. |
| 7.4 | Property tests for audit canonical encoding | S | Use `proptest`. |
| 7.5 | Security review pass (skeptic agent equivalent) | M | Threat model walk: RBAC bypass attempts, audit chain forgery, registry token replay, recovery-signer abuse. |
| 7.6 | Load test (deploys/min, registry pulls/min) | S | One synthetic operator with N orgs + N CVMs + N pulls. |

### Milestone 8 — Web UI question

The user-facing decision: **does v1 ship a web UI, or is it CLI-only?**

**Recommendation:** v1 ships an **API + a thin first-party CLI**, not a
web UI. The web UI is a separate workstream that lands in v1.5.

Reasoning:

1. The web UI work is large (`02-vmm-web-ui.md:797-915` documents how
   much the existing UI carries; a fresh UI for the new model is even
   bigger because it includes orgs, members, Safe transaction queue,
   audit views, registry views, quota UI). That's a 3-month workstream
   on its own.
2. Privy's hosted login UI handles 80% of what a "login page" needs;
   the remaining UI surface is mostly CRUD lists.
3. Operators self-hosting will be technical enough to drive a CLI.
4. The **API + OpenAPI spec** lands in M0–M5; building a UI on top is
   pure frontend work and can run in parallel post-M3 (so it could
   ship simultaneously with the rest of v1 if a separate frontend
   engineer joins).

If the user prefers a UI in v1, the work is:

| # | Work | Size |
|---|------|------|
| UI.1 | React + TypeScript SPA, vite-driven, hosted by the control plane | XL |
| UI.2 | Privy SDK integration for embedded-wallet signing flow | M |
| UI.3 | Org switcher, member mgmt, invitation flow | M |
| UI.4 | CVM list + deploy form + log viewer + status panels | L |
| UI.5 | Safe transaction queue UI (sign / submit) | M |
| UI.6 | Registry repo browser + tag list | M |
| UI.7 | Audit log viewer | S |
| UI.8 | Quota / usage dashboard | S |
| UI.9 | Platform-admin views | M |

Adding **UI.* in parallel** is reasonable; the API contract is fixed
by M2, so a frontend engineer can start on UI.1–UI.9 from week 15.

### Milestone summary / calendar estimate

| Milestone | Weeks (cumulative for one engineer) | Notes |
|-----------|--------------------------------------|-------|
| M0 — foundation | 1–6 | |
| M1 — control plane core | 7–14 | Could overlap M0 with two engineers. |
| M2 — on-chain + Safe | 15–22 | Independent; could overlap M1. |
| M3 — host worker + CVM | 23–30 | Depends on M0 (DB) + M1 (audit/quotas) + M2 (Safe for app deployment). |
| M4 — OCI registry | 31–40 | Independent; could overlap M3. |
| M5 — install / docs | 41–46 | After M4. |
| M6 — test / hardening | 47–55 | After M5. |
| **Total v1 (CLI-first)** | **~14 months for one engineer** | **~7 months for two engineers** working in parallel. |
| UI workstream | +3 months calendar | If a third engineer joins by week 15. |

This is a v1 with a real production posture, multi-tenant from day one,
TEE-rooted, and shippable. The numbers are on the conservative side
(no surprises = ~10 months for one engineer); aggressive single-engineer
estimate would be ~10–12 months.

---

## 14. Open implementation questions deferred to build time

These are questions `00-decisions.md` §16 explicitly defers, plus a few
that surfaced during synthesis:

1. **Frontend framework for the optional UI workstream:** React /
   Vue / Svelte / Solid. Likely React (largest ecosystem of Privy
   examples, Safe SDK examples, OCI registry browsers).
2. **CLI shape:** ship a Rust `dstack` CLI or a Python one? Rust avoids
   the runtime dependency; Python (à la `vmm-cli.py`) is faster to
   iterate. Recommendation: Rust, sharing the OpenAPI client crate
   the control plane already publishes.
3. **Is `prpc` retained internally for any boundary?** The host-worker
   ↔ host_api crossing on vsock is the only place `prpc` adds value
   (mutual familiarity with `dstack-guest-agent`). The
   user-facing API is REST/JSON. Recommendation: keep `prpc` *only*
   for the vsock host_api boundary; everything else is REST/JSON.
4. **Which `prometheus` crate:** `prometheus 0.13` (textfile-based) vs
   `metrics-exporter-prometheus 0.15` (modern, integrates with
   `metrics-rs`). Recommendation: `metrics-rs` + the prometheus
   exporter — it's how `kms/` is moving (recent commit `1a7c59b6:
   "kms: enable metrics by default"`).
5. **`/etc/dstack/master.key` rotation:** v1 has no key rotation. If
   the KMK is compromised, the operator manually re-encrypts every
   secret. A future v1.x adds `dstack-control-plane secrets rotate-kmk`.
6. **Privy Account export / "claim my wallet"** (00-decisions.md:298)
   is deferred. We do log the Privy DID + linked wallets so a future
   migration is possible.
7. **Public-key signature verification of KMS responses in browser**
   (00-decisions.md:306). The existing UI today does not verify
   `signature_v1` on `PublicKeyResponse`
   (`02-vmm-web-ui.md:600-695`). The new UI should verify it before
   using the env-encrypt pubkey. Implementation: pin the KMS root
   pubkey at install time in `secrets["kms.trusted_signer_pubkeys"]`,
   ship it to the browser via `/v1/auth/session`, verify in the
   browser before any encryption call. Decide before UI.4.
8. **Concrete schema migration tool:** `sqlx-cli` migrations vs
   `refinery` vs `atlas`. Recommendation: **`sqlx-cli`** — same crate
   as the runtime DB layer; one toolchain.
9. **Postgres connection pool:** `sqlx::PgPool` defaults are fine
   (max 10). Tuning is observability-driven later.
10. **Backup automation:** v1 leaves `pg_dump` to the operator
    (`00-decisions.md:114-117`). v1.x can ship a "scheduled snapshot
    to S3 / B2 / restic repository" addon. Out of scope here.
11. **Telemetry off by default**: confirmed (`00-decisions.md:269`).
    No phone-home in v1.

---

## 15. Risks

1. **Privy API surface drift / outage.** Privy is a vendor; their REST
   API can change. *Mitigation:* pin Privy SDK / API version; have an
   integration-test suite that runs against Privy Sandbox; include a
   "Privy unreachable → can't sign in but existing sessions and machine
   tokens keep working" path in middleware. Document the dependency
   prominently.

2. **Safe v1.4 ABI changes / threshold semantics.** Our hand-rolled
   Safe encoder is a small surface but a load-bearing one. *Mitigation:*
   golden fixtures from Safe's TS SDK; integration tests against the
   real Safe contracts on Base Sepolia; pin the contract version; do
   not abstract over Safe versions.

3. **Audit hash chain corruption / divergence.** A bug in the canonical
   encoding or in the writer (e.g. concurrent writes assigning the same
   `prev_hash`) breaks the chain. *Mitigation:* `audit_log.id` is
   `bigserial` (single-writer-friendly); the trigger that computes
   `row_hash` runs in the same transaction with `SELECT ... FROM
   audit_log ORDER BY id DESC LIMIT 1 FOR UPDATE` to lock the tail; a
   nightly verifier compares stored `row_hash` to recomputed and pages
   on mismatch.

4. **Master key (KMK) loss.** The operator loses
   `/etc/dstack/master.key` and didn't back it up. *Mitigation:* the
   wall-of-text at install (§12) is the primary defense. The DB itself
   stays intact; only secrets need re-provisioning. Document recovery
   procedure: re-run `init`, re-paste secrets, audit-batch lineage
   resets.

5. **Postgres data loss.** v1 has no scheduled backups. *Mitigation:*
   document `pg_dump` thoroughly, recommend restic + cron, ship a
   sample script. The qcow2 disks aren't in the DB; they need their
   own backup policy. *Acknowledged risk:* a host-disk failure with no
   operator backup loses the install.

6. **OCI registry storage exhaustion.** Aggressive uploads can fill
   `/var/lib/dstack/registry/blobs/`. *Mitigation:* per-org image
   storage quota (`max_image_storage_mb`) enforced at PUT manifest
   time; nightly GC; `df` watch with alerts.

7. **CID pool exhaustion at scale.** Default `cid_pool_size` is 1000.
   *Mitigation:* document the cap; surface it in `/v1/admin/health`;
   simple raise via config.

8. **vsock CID-as-identity attack.** A malicious guest could try to
   spam `Notify` events for its own CID with absurd payloads.
   *Mitigation:* unchanged from today's
   `vmm/src/host_api_service.rs:21-33` — bound the per-VM event
   buffer (default 20 in `vmm.toml:14`); validate event types against
   a whitelist; rate-limit.

9. **Per-org sandbox uid leak between QEMUs.** A bug in `provision-org.sh`
   or in `cvms.workdir` ownership could allow CVM A to read CVM B's
   disk. *Mitigation:* TEE memory isolation already protects against
   in-memory leaks; the workdir ownership is enforced at provision time;
   add a startup-time lint that walks `/var/lib/dstack/vm/*/` and
   verifies ownership matches `cvms.organization_id.sandbox_uid`.

10. **Recovery-signer abuse.** Even with the 24h timelock, an attacker
    who compromises both the KMK *and* the operator's email/Privy can
    use the recovery signer to take over orgs. *Mitigation:* the
    timelock is the primary defense; the audit log records every
    `recovery.intent.publish` so affected orgs see it; off by default
    (`00-decisions.md:108`); document the threat model in the operator
    onboarding.

---

## 16. Bibliography of cited reports

| Section | Source reports |
|---------|----------------|
| §1 Executive summary | `00-decisions.md` (locked decisions, all sections); `00-overview.md`; `10-multitenant-design-space.md:1183-1206` (architectural shape); `11-migration-strategy.md:476-522` (greenfield rationale). |
| §2 Architecture | `00-decisions.md:26-45` (process layout); `04-vmm-lifecycle-and-state.md:121-139` (current process startup we're modelling on); `05-vmm-coupling-map.md:367-457` (host OS coupling we keep in the host worker); `06-vmm-auth-today-and-gaps.md:30-39` (listener model); `08-production-ops-gaps.md:185-232` (today's single-host story). |
| §3 Database schema | `00-decisions.md:111-141` (persistence model); `07-persistence-today-and-multitenant.md:213-457` (proposed schema sketch — extended and committed here); `09-cvm-domain-model.md:18-200` (entity model). |
| §4 REST/JSON API surface | `00-decisions.md:233-241` (API design); `10-multitenant-design-space.md:534-657` (REST sketch); `01-vmm-api-surface.md:80-540` (current API surface we're replacing). |
| §5 Auth flow | `00-decisions.md:46-77` (identity, tokens, sessions); `06-vmm-auth-today-and-gaps.md:236-275` (multi-tenant auth requirements); `09-cvm-domain-model.md:182-198` (role entities). |
| §6 On-chain integration | `00-decisions.md:78-108` (Safe + Base + recovery); `11-migration-strategy.md:280-323` (DstackKms / DstackApp existing surface); `09-cvm-domain-model.md:103-138` (KmsApp / KmsRegistry entity); `10-multitenant-design-space.md:325-460` (custody options). |
| §7 CVM lifecycle | `00-decisions.md:138-141` (DB as source of truth); `04-vmm-lifecycle-and-state.md:30-86` (current end-to-end flow); `04-vmm-lifecycle-and-state.md:340-419` (recovery and state transitions); `09-cvm-domain-model.md:202-273` (lifecycle states); `07-persistence-today-and-multitenant.md:471-621` (DB↔disk projection). |
| §8 OCI registry | `00-decisions.md:202-216`; `08-production-ops-gaps.md:114-128` (today's image-pull story we're displacing). |
| §9 Audit + signed batches | `00-decisions.md:166-186`; `06-vmm-auth-today-and-gaps.md:323-353` (audit requirements); `07-persistence-today-and-multitenant.md:419-437` (audit_log table sketch). |
| §10 Quota enforcement | `00-decisions.md:188-199`; `08-production-ops-gaps.md:393-463` (today's nonexistent quota story); `06-vmm-auth-today-and-gaps.md:295-296` (per-org quota requirements). |
| §11 Observability | `00-decisions.md:243-253`; `08-production-ops-gaps.md:31-141` (current obs gaps). |
| §12 Install / first-run | `00-decisions.md:144-164` (KMK + secrets); `00-decisions.md:62-67` (first-user platform-admin); `08-production-ops-gaps.md:270-393` (install / upgrade gaps today). |
| §13 Work breakdown | All reports inform sizing; specifically `02-vmm-web-ui.md:797-915` for UI complexity and `03-vmm-cli.md:329-389` for CLI shape. |
| §14 Open questions | `00-decisions.md:308-322` (open implementation questions); `00-decisions.md:286-306` (deferred items); §15 of `06-vmm-auth-today-and-gaps.md:428-436` (open questions). |
| §15 Risks | `06-vmm-auth-today-and-gaps.md:298-322` (cross-org isolation requirements); `08-production-ops-gaps.md:466-601` (failure modes today); operator-experience risks from `10-multitenant-design-space.md:357-417`. |

The decisions doc (`docs/vmm-rewrite-plan/00-decisions.md`) is the
source of truth that this plan implements. Each of the 11 investigation
reports
(`docs/vmm-rewrite-plan/01-vmm-api-surface.md` through
`docs/vmm-rewrite-plan/11-migration-strategy.md`) is the evidence
substrate for the respective design choice. Any disagreement between
this final plan and the decisions doc should be resolved in favor of
the decisions doc; any disagreement between this plan and the reports
is the synthesis pass making a concrete pick that the reports
left open.
