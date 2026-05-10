# dstack-vmm rewrite final plan v2

Status: synthesis v2, based on the revised locked decisions in
[`00-decisions.md`](00-decisions.md). This document replaces the stale
[`99-final-plan.md`](99-final-plan.md). The prior plan is useful history, but it
included a v1 in-house OCI registry, signed audit batches, VLAN-oriented bridge
language, three org roles, and no host_id-ready schema. Those are no longer the
plan.

## 1. Executive Summary

`dstack-control-plane` is a new Rust binary for fresh dstack installs. It
replaces `dstack-vmm` as the management process for new deployments and
subsumes the user-facing API, multi-tenant authorization, Postgres-backed state,
audit/quota logic, Safe orchestration, and the host-side QEMU supervisor driver
in one process. The v1 shape is deliberately one control-plane process on one
TDX bare-metal host, with Postgres in a sibling Docker container on the same
machine. Existing `dstack-vmm` installs stay on the legacy binary; v1 has no
import path, no `vmm-cli.py` compatibility, and no prpc-v1 compatibility
(`00-decisions.md:10-15`, `00-decisions.md:41-44`,
`00-decisions.md:333-339`).

The headline design choices are:

- Identity is Privy, using embedded wallets plus BYO external wallet links.
  There is no password auth in v1 (`00-decisions.md:55-61`).
- Org authorization is four-role RBAC: Admin, Deploy, Billing, Viewer, plus a
  separate global Platform Admin (`00-decisions.md:62-80`).
- API tokens are org-bound bearer tokens. Token role must be no more privileged
  than the creator's role at issue time. Creator demotion does not revoke or
  downgrade already-issued tokens (`00-decisions.md:81-93`).
- On-chain authority is one Safe per org on Base mainnet, with current admins
  synced as 1-of-N Safe signers (`00-decisions.md:95-126`).
- State is Postgres. Secrets are sodium-secretbox encrypted in a `secrets`
  table under an operator-owned KMK (`00-decisions.md:127-183`).
- Audit is a hash chain only. There are no operator-signed hourly batches in v1
  (`00-decisions.md:185-203`).
- Container app images are pulled from org-configured external private
  registries. There is no in-house OCI registry server, no OCI registry API,
  and no OCI blob store (`00-decisions.md:218-241`).
- Bridge-mode CVM isolation is per-org Linux bridge plus per-org Linux uid.
  User-mode SLIRP remains the default and does not use this bridge plumbing
  (`00-decisions.md:243-286`).
- The schema is multi-host-ready through `host_id` FKs even though the v1
  implementation has one row in `hosts` and one in-process host worker
  (`00-decisions.md:30-40`).

Current code baseline: today's `dstack-vmm` is Rocket + pRPC mounted under
`/prpc`, `/guest`, and `/api`, with a static web UI and a single authenticated
logs route (`vmm/src/main.rs:81-90`, `vmm/src/main_routes.rs:45-186`). Its
networking config is global and has user, bridge, and custom modes with no org
or VLAN field (`vmm/src/config.rs:386-427`). Bridge mode passes
`bridge,id=net0,br=<bridge>` directly to QEMU (`vmm/src/app/qemu.rs:523-526`),
and the default config is user-mode SLIRP (`vmm/vmm.toml:86-96`). The new control
plane keeps the useful host-driver pieces but makes Postgres, org ownership,
RBAC, audit, quotas, Safe state, and network isolation first-class.

Differs from prior plan: removes the embedded OCI registry and signed-batch
audit pipeline, adds Deploy, adds `host_id` across org-scoped host resources,
adds `org_registry_creds` and `org_networks`, and replaces VLAN tagging with
per-org Linux bridges.

## 2. Architecture

### 2.1 Process Layout

| Process | Runs as | Purpose |
|---|---|---|
| `dstack-control-plane` | `dstack` system uid | REST/JSON API, Privy session validation, RBAC, quota admission, audit writes, Safe orchestration, secrets encryption/decryption, in-process host worker, QEMU/supervisor driver, bridge/dnsmasq/nftables reconciliation through a small privileged helper. |
| Postgres 16 container | container user mapped to host `dstack` socket ownership | Durable metadata. Control plane connects over Unix-socket peer auth; no DB password in normal install (`00-decisions.md:127-135`). |
| `dstack-net-helper` | root, invoked by systemd/sudoers with fixed argv schema | Owns privileged network mutations: `ip link`, `nft`, dnsmasq unit writes/restarts, and `/etc/qemu/bridge.conf` allowlist edits. The control-plane process does not retain ambient `CAP_NET_ADMIN`. |
| QEMU processes | per-org sandbox uid, e.g. `dstack-o-<short>` | Run CVMs. One Linux uid per org. Uid owns only that org's CVM workdirs and has no route or filesystem access to control-plane secrets. |
| dnsmasq | root or `dnsmasq`, one process per org bridge | DHCP/DNS for that org bridge only. Lease notification goes to a private local socket with bridge identity. |
| dstack-gateway / dstack-kms | existing service users | Remain adjacent services. Gateway continues to provide public ingress. KMS continues to verify boot authority and issue guest secrets. |

The v1 binary contains an in-process host worker trait rather than a separate
node process. The trait boundary should still look like the future multi-host
RPC boundary:

```text
trait HostWorker {
    fn ensure_org_sandbox(org_id, uid, bridge_spec) -> Result<()>;
    fn deploy_cvm(cvm_id) -> Result<()>;
    fn start_cvm(cvm_id) -> Result<()>;
    fn stop_cvm(cvm_id) -> Result<()>;
    fn remove_cvm(cvm_id) -> Result<()>;
    fn reconcile() -> Result<HostReport>;
}
```

That lets v2 split the worker into a node agent without changing the database
model or user-facing API. This reflects the accepted skeptic push-back for
host_id-ready schema while keeping the implementation single-host
(`00-decisions.md:37-40`, `99-skeptic.md:933-939`).

### 2.2 Listening Sockets

| Endpoint | Bind | Auth | Purpose |
|---|---|---|---|
| `https://api.<base-domain>/v1/...` | behind gateway or local reverse proxy | Privy-backed session cookie or `Authorization: Bearer dst_...` | User-facing REST/JSON API. |
| `GET /metrics` | localhost by default; optionally behind admin-auth reverse proxy | Platform Admin session or local scrape token | Prometheus metrics. |
| `GET /healthz` | localhost and external | none or local allowlist | Liveness. |
| `GET /readyz` | localhost and external | none or local allowlist | Readiness after DB, KMK, supervisor, network reconciler, and startup CVM scan succeed. |
| `vsock:CID=2:port=10000` | vsock | guest CID/context plus existing RA patterns | Host API for guest agents. Today the host API is already constrained to vsock by config validation (`vmm/src/config.rs:447-466`) and handler construction (`vmm/src/host_api_service.rs:21-33`). |
| `/run/dstack/dhcp-leases.sock` | Unix datagram or seqpacket socket | filesystem permissions, root/dnsmasq writer only | Private DHCP lease channel. Replaces today's unauthenticated HTTP-ish `ReportDhcpLease` path with bridge identity. |
| Postgres socket | `/run/dstack/postgres/.s.PGSQL.5432` | peer auth | DB access for `dstack` uid only. |
| Supervisor socket | `/run/dstack/supervisor.sock` or in-process replacement | local uid permissions | QEMU process control. |

There is intentionally no `registry.<base-domain>/v2/...` listener. The control
plane is not an OCI registry in v1.

### 2.3 Filesystem Layout

```text
/etc/dstack/
  control-plane.toml
  master.key                         # 0600 dstack:dstack; KMK, never in DB
  privy.toml                         # non-secret app id / public config
  network.toml                       # bridge supernet, uplink iface, DNS defaults

/run/dstack/
  control-plane.sock                 # optional local admin socket
  dhcp-leases.sock                   # dnsmasq lease notifications
  postgres/                          # Postgres Unix socket bind mount
  dnsmasq/
    dstack-org-<short>.pid

/var/lib/dstack/
  postgres/                          # Docker volume bind mount
  images/<image-name>/               # global guest OS image catalog
  orgs/<org-id>/
    cvms/<cvm-id>/
      vm-manifest.json               # materialised from cvms + artifacts
      vm-state.json                  # materialised runtime desired flag
      hda.img
      shared/
        app-compose.json
        .encrypted-env
        .user-config
        .instance_info
        .docker/config.json          # deploy-time materialised registry creds
        .sys-config.json
      serial.log
      serial.history.log
      stdout.log
      stderr.log
      qmp.sock
      guest-ip
      .removing

/var/lib/dstack/network/
  dnsmasq/dstack-org-<short>.conf     # generated, do not edit
  nftables/dstack.nft                 # generated desired rules

/etc/qemu/bridge.conf                 # managed allowlist entries:
                                      # allow dstack-org-<short>
```

Today's VMM already stores per-CVM workdir state such as `vm-manifest.json`,
`vm-state.json`, `shared/app-compose.json`, `.encrypted-env`, `.user-config`,
`.instance_info`, `guest-ip`, logs, `hda.img`, and `qmp.sock`
(`vmm/src/app/qemu.rs:990-1118`; `vmm/src/app.rs:964-1004`). In v2, Postgres is
authoritative for metadata and artifact revisions; the host worker rematerialises
those files before start. Large blobs stay on disk.

### 2.4 Boundaries

The control plane is outside the TEE. It is trusted to orchestrate but not
trusted by the TEE for guest memory confidentiality. This matches today's host
orchestrator trust boundary (`00-decisions.md:45-50`). The TEE protects guest
memory and CPU state; it does not filter traffic after virtio-net frames reach a
host bridge, which is why bridge-mode network isolation is explicit
(`12-f3-vlan-isolation.md:66-72`).

The `dstack` uid owns the control-plane config, KMK file, DB socket, and DB
volume. Per-org sandbox uids own only their org's CVM workdir tree. The network
helper is the only root boundary and accepts fixed subcommands with validated
org bridge names, subnets, dnsmasq config paths, and nftables set entries.

No in-house registry exists. No `/var/lib/dstack/registry`, no `oci_*` tables,
no OCI bearer-token endpoint, no registry blob GC. No signed-batch audit
pipeline exists. Audit rows are hash-chained in `audit_log` only.

### 2.5 Diagram

```text
                              Internet
                                  |
                         dstack-gateway / TLS
                                  |
                  +---------------+----------------+
                  |                                |
          api.<base>/v1/...              <cvm-id>.<base> ingress
                  |                                |
          +-------v--------------------------------v------+
          |            dstack-control-plane               |
          |  - REST/JSON API + OpenAPI                    |
          |  - Privy sessions + API tokens                |
          |  - org RBAC, quotas, audit hash chain         |
          |  - Safe orchestration on Base                 |
          |  - external registry credential materialiser  |
          |  - in-process host worker                     |
          |  - bridge/dnsmasq/nftables reconciler         |
          +-----------+---------------------+-------------+
                      |                     |
              Unix socket peer auth        | fixed sudo/systemd calls
                      |                     |
              +-------v------+       +------v-------------+
              | Postgres     |       | dstack-net-helper  |
              | in Docker    |       | root network ops   |
              +--------------+       +------+-------------+
                                             |
                         +-------------------+-------------------+
                         |                                       |
              dstack-org-a bridge                     dstack-org-b bridge
              10.42.17.1/24                           10.42.18.1/24
              dnsmasq(a), NAT                          dnsmasq(b), NAT
                         |                                       |
              +----------v----------+                +-----------v---------+
              | QEMU as uid org A   |                | QEMU as uid org B   |
              | CVMs, TAPs enslaved |                | CVMs, TAPs enslaved |
              +---------------------+                +---------------------+
```

## 3. Database Schema

This schema is concrete enough to start migrations. It uses UUID primary keys
for domain objects, `bigserial` only for ordered audit rows, `jsonb` for
structured but not relationally queried payloads, `inet/cidr` for network
state, and Postgres enums via `CHECK` constraints to keep migrations simple.

The single-host v1 install inserts exactly one row into `hosts` during init.
Org-scoped host resources still carry `host_id` FKs so v2 can add more hosts
without table rewrites (`00-decisions.md:37-40`).

Intentionally absent: `signed_batches`, `oci_blobs`, `oci_manifests`,
`oci_repos`, `oci_tags`, and `oci_upload_sessions`.

```sql
CREATE EXTENSION IF NOT EXISTS citext;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE hosts (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    hostname            text NOT NULL UNIQUE,
    display_name        text NOT NULL,
    status              text NOT NULL CHECK (status IN ('initializing','online','draining','offline','retired')),
    role                text NOT NULL DEFAULT 'single_host' CHECK (role IN ('single_host','worker')),
    region              text,
    base_domain         text,
    control_plane_version text NOT NULL,
    capabilities        jsonb NOT NULL DEFAULT '{}',
    cid_start           int NOT NULL,
    cid_pool_size       int NOT NULL,
    total_vcpu          int NOT NULL,
    total_memory_mb     int NOT NULL,
    total_disk_bytes    bigint NOT NULL,
    bridge_supernet     cidr,
    bridge_prefix_len   int NOT NULL DEFAULT 24,
    uplink_iface        text,
    last_heartbeat_at   timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    privy_user_id       text NOT NULL UNIQUE,
    email               citext UNIQUE,
    email_verified_at   timestamptz,
    display_name        text NOT NULL DEFAULT '',
    avatar_url          text,
    status              text NOT NULL DEFAULT 'active'
        CHECK (status IN ('active','suspended','deleted')),
    is_platform_admin   boolean NOT NULL DEFAULT false,
    primary_wallet_id   uuid,
    last_login_at       timestamptz,
    deleted_at          timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE user_wallets (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    privy_wallet_id     text,
    address             bytea NOT NULL CHECK (length(address) = 20),
    chain_type          text NOT NULL DEFAULT 'evm' CHECK (chain_type IN ('evm')),
    wallet_kind         text NOT NULL CHECK (wallet_kind IN ('privy_embedded','external')),
    connector           text,
    is_primary_signer   boolean NOT NULL DEFAULT false,
    verified_at         timestamptz NOT NULL DEFAULT now(),
    created_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (user_id, address)
);
CREATE UNIQUE INDEX user_wallets_one_primary_idx
    ON user_wallets (user_id)
    WHERE is_primary_signer;
ALTER TABLE users
    ADD CONSTRAINT users_primary_wallet_fk
    FOREIGN KEY (primary_wallet_id) REFERENCES user_wallets(id) DEFERRABLE INITIALLY DEFERRED;

CREATE TABLE organizations (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    host_id             uuid NOT NULL REFERENCES hosts(id),
    slug                citext NOT NULL UNIQUE,
    display_name        text NOT NULL,
    personal_org        boolean NOT NULL DEFAULT false,
    created_by_user_id  uuid NOT NULL REFERENCES users(id),
    status              text NOT NULL DEFAULT 'active'
        CHECK (status IN ('active','suspended','deleting','deleted')),
    safe_address        bytea CHECK (safe_address IS NULL OR length(safe_address) = 20),
    deleted_at          timestamptz,
    tombstone_expires_at timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    CHECK (slug ~ '^[a-z0-9-]{2,32}$')
);
CREATE INDEX organizations_host_idx ON organizations(host_id);
CREATE INDEX organizations_active_idx ON organizations(id) WHERE deleted_at IS NULL;

CREATE TABLE memberships (
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id             uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role                text NOT NULL CHECK (role IN ('admin','deploy','billing','viewer')),
    status              text NOT NULL DEFAULT 'active'
        CHECK (status IN ('active','pending_removal')),
    pending_safe_tx_id  uuid,
    invited_by_user_id  uuid REFERENCES users(id),
    accepted_at         timestamptz NOT NULL DEFAULT now(),
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, user_id)
);
CREATE INDEX memberships_user_idx ON memberships(user_id);
CREATE INDEX memberships_org_role_idx ON memberships(org_id, role);

CREATE TABLE invitations (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email               citext NOT NULL,
    role                text NOT NULL CHECK (role IN ('admin','deploy','billing','viewer')),
    token_hash          bytea NOT NULL CHECK (length(token_hash) = 32),
    status              text NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','accepted','revoked','expired')),
    invited_by_user_id  uuid NOT NULL REFERENCES users(id),
    accepted_by_user_id uuid REFERENCES users(id),
    expires_at          timestamptz NOT NULL,
    accepted_at         timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX invitations_pending_email_idx
    ON invitations(org_id, lower(email))
    WHERE status = 'pending';
CREATE UNIQUE INDEX invitations_token_hash_idx ON invitations(token_hash);

CREATE TABLE api_tokens (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                text NOT NULL,
    token_prefix        text NOT NULL,
    token_hash          bytea NOT NULL CHECK (length(token_hash) = 32),
    role                text NOT NULL CHECK (role IN ('admin','deploy','billing','viewer')),
    created_by_user_id  uuid REFERENCES users(id) ON DELETE SET NULL,
    creator_role_at_issue text NOT NULL CHECK (creator_role_at_issue IN ('admin','deploy','billing','viewer')),
    last_used_at        timestamptz,
    expires_at          timestamptz,
    revoked_at          timestamptz,
    revoked_by_user_id  uuid REFERENCES users(id) ON DELETE SET NULL,
    created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX api_tokens_hash_idx ON api_tokens(token_hash);
CREATE INDEX api_tokens_org_active_idx ON api_tokens(org_id) WHERE revoked_at IS NULL;

-- Role ordering is application-enforced in the token creation transaction:
-- admin=40, deploy=30, billing=20, viewer=10. The selected token role must be
-- <= the creator's current membership role at issue time. No parent-token column
-- and no cascading revocation in v1.

CREATE TABLE sessions (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    privy_session_id    text,
    cookie_hash         bytea NOT NULL CHECK (length(cookie_hash) = 32),
    user_agent          text,
    ip                  inet,
    created_at          timestamptz NOT NULL DEFAULT now(),
    last_seen_at        timestamptz,
    expires_at          timestamptz NOT NULL,
    revoked_at          timestamptz
);
CREATE UNIQUE INDEX sessions_cookie_hash_idx ON sessions(cookie_hash);
CREATE INDEX sessions_user_active_idx ON sessions(user_id) WHERE revoked_at IS NULL;

CREATE TABLE images (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    host_id             uuid REFERENCES hosts(id),
    name                text NOT NULL UNIQUE,
    version             text NOT NULL,
    description         text NOT NULL DEFAULT '',
    digest              bytea NOT NULL CHECK (length(digest) = 32),
    path                text NOT NULL,
    metadata            jsonb NOT NULL DEFAULT '{}',
    is_dev              boolean NOT NULL DEFAULT false,
    enabled             boolean NOT NULL DEFAULT true,
    size_bytes          bigint NOT NULL DEFAULT 0,
    seeded_by           text NOT NULL DEFAULT 'operator',
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX images_digest_idx ON images(digest);

CREATE TABLE cvms (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT,
    host_id             uuid NOT NULL REFERENCES hosts(id),
    image_id            uuid NOT NULL REFERENCES images(id),
    created_by_user_id  uuid REFERENCES users(id) ON DELETE SET NULL,
    name                text NOT NULL,
    app_id              bytea NOT NULL CHECK (length(app_id) = 20),
    compose_hash        bytea NOT NULL CHECK (length(compose_hash) = 32),
    vcpu                int NOT NULL CHECK (vcpu > 0),
    memory_mb           int NOT NULL CHECK (memory_mb > 0),
    disk_size_gb        int NOT NULL CHECK (disk_size_gb > 0),
    gpus                jsonb NOT NULL DEFAULT '{}',
    networking_mode     text NOT NULL DEFAULT 'user'
        CHECK (networking_mode IN ('user','bridge')),
    port_map            jsonb NOT NULL DEFAULT '[]',
    kms_urls            text[] NOT NULL DEFAULT '{}',
    gateway_urls        text[] NOT NULL DEFAULT '{}',
    no_tee              boolean NOT NULL DEFAULT false,
    hugepages           boolean NOT NULL DEFAULT false,
    pin_numa            boolean NOT NULL DEFAULT false,
    desired_state       text NOT NULL DEFAULT 'stopped'
        CHECK (desired_state IN ('running','stopped','deleted')),
    observed_state      text NOT NULL DEFAULT 'pending'
        CHECK (observed_state IN ('pending','provisioning','starting','running','stopping','stopped','exited','removing','deleted','error')),
    cid                 int,
    sandbox_uid         int,
    bridge_name         text,
    guest_ip            inet,
    instance_id         text,
    boot_progress       text,
    boot_error          text,
    last_error          text,
    workdir             text NOT NULL,
    started_at          timestamptz,
    stopped_at          timestamptz,
    last_seen_at        timestamptz,
    deleted_at          timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);
CREATE INDEX cvms_org_idx ON cvms(org_id);
CREATE INDEX cvms_host_idx ON cvms(host_id);
CREATE INDEX cvms_state_idx ON cvms(observed_state);
CREATE UNIQUE INDEX cvms_host_cid_idx ON cvms(host_id, cid) WHERE cid IS NOT NULL;
CREATE INDEX cvms_bridge_idx ON cvms(host_id, bridge_name) WHERE networking_mode = 'bridge';

CREATE TABLE cvm_artifacts (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    cvm_id              uuid NOT NULL REFERENCES cvms(id) ON DELETE CASCADE,
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    host_id             uuid NOT NULL REFERENCES hosts(id),
    kind                text NOT NULL CHECK (kind IN ('app-compose','encrypted-env','user-config','instance-info','docker-config')),
    version             int NOT NULL CHECK (version > 0),
    sha256              bytea NOT NULL CHECK (length(sha256) = 32),
    ciphertext          bytea,
    plaintext           bytea,
    storage_uri         text,
    created_by_user_id  uuid REFERENCES users(id) ON DELETE SET NULL,
    created_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (cvm_id, kind, version),
    CHECK (
        (kind IN ('encrypted-env','docker-config') AND ciphertext IS NOT NULL AND plaintext IS NULL)
        OR (kind NOT IN ('encrypted-env','docker-config') AND plaintext IS NOT NULL)
        OR storage_uri IS NOT NULL
    )
);
CREATE INDEX cvm_artifacts_latest_idx ON cvm_artifacts(cvm_id, kind, version DESC);

CREATE TABLE safes (
    org_id              uuid PRIMARY KEY REFERENCES organizations(id) ON DELETE CASCADE,
    chain_id            bigint NOT NULL DEFAULT 8453,
    safe_address        bytea NOT NULL CHECK (length(safe_address) = 20),
    safe_version        text NOT NULL DEFAULT '1.4.1',
    threshold           int NOT NULL DEFAULT 1,
    signer_cache        jsonb NOT NULL DEFAULT '[]',
    recovery_enabled    boolean NOT NULL DEFAULT false,
    recovery_signer_address bytea CHECK (recovery_signer_address IS NULL OR length(recovery_signer_address) = 20),
    deployed_tx_hash    bytea,
    deployed_at         timestamptz,
    last_synced_at      timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE safe_transactions (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    safe_address        bytea NOT NULL CHECK (length(safe_address) = 20),
    chain_id            bigint NOT NULL DEFAULT 8453,
    kind                text NOT NULL CHECK (kind IN ('deploy_safe','add_owner','remove_owner','set_threshold','app_add_compose_hash','app_remove_compose_hash','app_add_device','app_remove_device','recovery_intent','recovery_execute')),
    target_address      bytea CHECK (target_address IS NULL OR length(target_address) = 20),
    calldata            bytea NOT NULL,
    nonce               numeric(78,0),
    status              text NOT NULL DEFAULT 'proposed'
        CHECK (status IN ('proposed','awaiting_signature','signed','submitted','confirmed','failed','expired','cancelled')),
    proposed_by_user_id uuid REFERENCES users(id) ON DELETE SET NULL,
    related_user_id     uuid REFERENCES users(id) ON DELETE SET NULL,
    tx_hash             bytea,
    error               text,
    not_before          timestamptz,
    confirmed_at        timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX safe_transactions_org_status_idx ON safe_transactions(org_id, status, created_at DESC);

CREATE TABLE safe_signatures (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    safe_transaction_id uuid NOT NULL REFERENCES safe_transactions(id) ON DELETE CASCADE,
    signed_by_user_id   uuid REFERENCES users(id) ON DELETE SET NULL,
    signer_address      bytea NOT NULL CHECK (length(signer_address) = 20),
    signature           bytea NOT NULL,
    signature_kind      text NOT NULL CHECK (signature_kind IN ('privy_embedded','external_wallet','recovery_signer')),
    created_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (safe_transaction_id, signer_address)
);

CREATE TABLE kms_apps (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    cvm_id              uuid REFERENCES cvms(id) ON DELETE SET NULL,
    chain_id            bigint NOT NULL DEFAULT 8453,
    app_id              bytea NOT NULL CHECK (length(app_id) = 20),
    dstack_app_address  bytea CHECK (dstack_app_address IS NULL OR length(dstack_app_address) = 20),
    dstack_kms_address  bytea CHECK (dstack_kms_address IS NULL OR length(dstack_kms_address) = 20),
    compose_hash        bytea NOT NULL CHECK (length(compose_hash) = 32),
    allow_any_device    boolean NOT NULL DEFAULT false,
    require_tcb_up_to_date boolean NOT NULL DEFAULT false,
    status              text NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','registered','updating','error','retired')),
    last_safe_tx_id     uuid REFERENCES safe_transactions(id) ON DELETE SET NULL,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (org_id, app_id, compose_hash)
);
CREATE INDEX kms_apps_app_id_idx ON kms_apps(app_id);

CREATE TABLE quotas (
    org_id              uuid PRIMARY KEY REFERENCES organizations(id) ON DELETE CASCADE,
    max_cvms_hard       int NOT NULL DEFAULT 5,
    max_cvms_soft       int NOT NULL DEFAULT 4,
    max_vcpu_hard       int NOT NULL DEFAULT 16,
    max_vcpu_soft       int NOT NULL DEFAULT 13,
    max_ram_mb_hard     int NOT NULL DEFAULT 32768,
    max_ram_mb_soft     int NOT NULL DEFAULT 26214,
    max_disk_gb_hard    int NOT NULL DEFAULT 200,
    max_disk_gb_soft    int NOT NULL DEFAULT 160,
    max_ingress_ports_hard int NOT NULL DEFAULT 32,
    max_ingress_ports_soft int NOT NULL DEFAULT 26,
    max_image_storage_bytes_hard bigint NOT NULL DEFAULT 53687091200,
    max_image_storage_bytes_soft bigint NOT NULL DEFAULT 42949672960,
    deploys_per_hour_hard int NOT NULL DEFAULT 20,
    deploys_per_hour_soft int NOT NULL DEFAULT 16,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE quota_overrides (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    dimension           text NOT NULL CHECK (dimension IN ('max_cvms','max_vcpu','max_ram_mb','max_disk_gb','max_ingress_ports','max_image_storage_bytes','deploys_per_hour')),
    hard_value          bigint NOT NULL,
    soft_value          bigint,
    granted_by_user_id  uuid NOT NULL REFERENCES users(id),
    reason              text NOT NULL,
    expires_at          timestamptz,
    revoked_at          timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX quota_overrides_effective_idx
    ON quota_overrides(org_id, dimension, expires_at)
    WHERE revoked_at IS NULL;

CREATE TABLE resource_usage (
    id                  bigserial PRIMARY KEY,
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    host_id             uuid NOT NULL REFERENCES hosts(id),
    cvm_id              uuid REFERENCES cvms(id) ON DELETE SET NULL,
    window_start        timestamptz NOT NULL,
    window_seconds      int NOT NULL DEFAULT 30,
    vcpu_declared       int NOT NULL DEFAULT 0,
    memory_mb_declared  int NOT NULL DEFAULT 0,
    disk_bytes_used     bigint NOT NULL DEFAULT 0,
    network_rx_bytes    bigint NOT NULL DEFAULT 0,
    network_tx_bytes    bigint NOT NULL DEFAULT 0,
    cpu_time_ms         bigint,
    memory_current_bytes bigint,
    samples             int NOT NULL DEFAULT 1,
    created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX resource_usage_org_window_idx ON resource_usage(org_id, window_start DESC);
CREATE INDEX resource_usage_cvm_window_idx ON resource_usage(cvm_id, window_start DESC);

CREATE TABLE audit_log (
    id                  bigserial PRIMARY KEY,
    org_id              uuid REFERENCES organizations(id) ON DELETE SET NULL,
    host_id             uuid REFERENCES hosts(id) ON DELETE SET NULL,
    actor_user_id       uuid REFERENCES users(id) ON DELETE SET NULL,
    actor_token_id      uuid REFERENCES api_tokens(id) ON DELETE SET NULL,
    actor_platform_admin boolean NOT NULL DEFAULT false,
    actor_ip            inet,
    user_agent          text,
    request_id          uuid,
    action              text NOT NULL,
    target_kind         text,
    target_id           text,
    result              text NOT NULL CHECK (result IN ('success','failure','denied')),
    error_code          text,
    payload             jsonb NOT NULL DEFAULT '{}',
    prev_hash           bytea NOT NULL CHECK (length(prev_hash) = 32),
    row_hash            bytea NOT NULL CHECK (length(row_hash) = 32),
    created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX audit_log_org_time_idx ON audit_log(org_id, created_at DESC, id DESC);
CREATE INDEX audit_log_actor_time_idx ON audit_log(actor_user_id, created_at DESC);
CREATE UNIQUE INDEX audit_log_row_hash_idx ON audit_log(row_hash);

CREATE TABLE secrets (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              uuid REFERENCES organizations(id) ON DELETE CASCADE,
    scope               text NOT NULL CHECK (scope IN ('platform','org')),
    name                text NOT NULL,
    version             int NOT NULL DEFAULT 1,
    cipher              text NOT NULL DEFAULT 'xsalsa20poly1305',
    nonce               bytea NOT NULL,
    ciphertext          bytea NOT NULL,
    aad                 bytea,
    key_id              text NOT NULL DEFAULT 'kmk-v1',
    created_by_user_id  uuid REFERENCES users(id) ON DELETE SET NULL,
    rotated_from_secret_id uuid REFERENCES secrets(id) ON DELETE SET NULL,
    revoked_at          timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (scope, org_id, name, version)
);
CREATE INDEX secrets_lookup_idx ON secrets(scope, org_id, name, version DESC)
    WHERE revoked_at IS NULL;

CREATE TABLE org_registry_creds (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    display_name        text NOT NULL,
    registry_url        text NOT NULL,
    registry_host       text NOT NULL,
    auth_method         text NOT NULL CHECK (auth_method IN ('basic','bearer','dockerhub_pat','ghcr_pat','gitlab_pat','ecr_sts','gar_service_account','gar_workload_identity')),
    username            text,
    secret_id           uuid NOT NULL REFERENCES secrets(id),
    ecr_role_arn        text,
    ecr_region          text,
    gar_project         text,
    last_verified_at    timestamptz,
    last_error          text,
    created_by_user_id  uuid REFERENCES users(id) ON DELETE SET NULL,
    revoked_at          timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (org_id, registry_host, display_name)
);
CREATE INDEX org_registry_creds_active_idx
    ON org_registry_creds(org_id, registry_host)
    WHERE revoked_at IS NULL;

CREATE TABLE org_networks (
    org_id              uuid PRIMARY KEY REFERENCES organizations(id) ON DELETE CASCADE,
    host_id             uuid NOT NULL REFERENCES hosts(id),
    bridge_name         text NOT NULL,
    subnet              cidr NOT NULL,
    gateway_ip          inet NOT NULL,
    dhcp_start          inet NOT NULL,
    dhcp_end            inet NOT NULL,
    mtu                 int NOT NULL DEFAULT 1500,
    state               text NOT NULL DEFAULT 'allocated'
        CHECK (state IN ('allocated','provisioning','ready','stale_empty','stale_in_use','gc_pending','error')),
    dnsmasq_pid         int,
    dnsmasq_config_hash bytea CHECK (dnsmasq_config_hash IS NULL OR length(dnsmasq_config_hash) = 32),
    nftables_generation bigint NOT NULL DEFAULT 0,
    qemu_allowlisted    boolean NOT NULL DEFAULT false,
    last_reconciled_at  timestamptz,
    last_lease_at       timestamptz,
    last_error          text,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    UNIQUE (host_id, bridge_name),
    UNIQUE (host_id, subnet)
);
CREATE INDEX org_networks_state_idx ON org_networks(host_id, state);
```

`safe_transactions` is referenced by `memberships.pending_safe_tx_id`; add that
FK after both tables are created:

```sql
ALTER TABLE memberships
    ADD CONSTRAINT memberships_pending_safe_tx_fk
    FOREIGN KEY (pending_safe_tx_id) REFERENCES safe_transactions(id)
    DEFERRABLE INITIALLY DEFERRED;
```

## 4. REST/JSON API Surface

### 4.1 Conventions

The user-facing API is REST/JSON, implemented with Axum. OpenAPI is generated
from route definitions. Stable paths use `/v1`; behavior is pinned with
`Dstack-Api-Version: 2026-05-01` and deprecations get 12 months
(`00-decisions.md:288-299`). Every response includes `Dstack-Request-Id`.

Authentication:

```http
Cookie: dstack_session=<opaque>          # browsers
Authorization: Bearer dst_<token>        # API tokens
Dstack-Api-Version: 2026-05-01
Idempotency-Key: <uuid-or-client-key>    # all create operations
```

Cursor pagination:

```json
{
  "data": [],
  "pagination": {
    "next_cursor": "eyJjcmVhdGVkX2F0IjoiLi4uIn0=",
    "has_more": true,
    "limit": 50
  }
}
```

Default limit is 50, max is 200 (`00-decisions.md:295-295`). Cursors encode the
stable sort key and last ID, not page numbers.

Error shape:

```json
{
  "error": {
    "type": "quota_exceeded",
    "code": "quota.max_vcpu.hard",
    "message": "Requested 8 vCPU would exceed the org hard limit of 16 vCPU.",
    "request_id": "018f2f3a-...",
    "details": {
      "dimension": "max_vcpu",
      "current": 12,
      "requested": 8,
      "limit": 16
    }
  }
}
```

Idempotency keys are stored for 24 hours keyed by `(org_id, actor_id, method,
path, key)`. A replay with the same body returns the original status/body. A
replay with a different body returns `409 idempotency_key_reused`.

Role summary:

| Role | Key permissions |
|---|---|
| Admin | Full org control including members, registry creds, quotas view, audit export, Safe admin changes, CVM destructive actions. |
| Deploy | Pull/use org images, deploy/start/stop/restart CVMs, update non-destructive deploy config, view CVM logs/state. Cannot delete CVMs, manage members, manage org settings, manage registry credentials, or read org secrets. |
| Billing | Read usage, quotas, audit, invoices/billing placeholders. No mutation. |
| Viewer | Read non-financial org resources. |

Differs from prior plan: no `/v2/` OCI registry endpoints and no
`registry.<base-domain>` token server. Registry support is an org credential
CRUD surface only.

### 4.2 Auth

```http
POST /v1/auth/privy/exchange
Content-Type: application/json

{ "privy_access_token": "..." }
```

Response:

```json
{
  "user": {
    "id": "uuid",
    "email": "alice@example.com",
    "display_name": "Alice",
    "is_platform_admin": true,
    "primary_wallet": { "address": "0xabc...", "kind": "privy_embedded" }
  },
  "session": { "expires_at": "2026-06-05T12:00:00Z" },
  "orgs": [{ "id": "uuid", "slug": "alice", "role": "admin" }]
}
```

Sets `Set-Cookie: dstack_session=...; HttpOnly; Secure; SameSite=Strict`.

```http
GET    /v1/auth/session
POST   /v1/auth/logout
GET    /v1/users/me
PATCH  /v1/users/me
GET    /v1/users/me/orgs
GET    /v1/users/me/wallets
POST   /v1/users/me/wallets/primary
```

`POST /v1/users/me/wallets/primary` body:

```json
{ "wallet_id": "uuid" }
```

### 4.3 Organizations

```http
POST /v1/orgs
{ "slug": "acme", "display_name": "Acme" }
```

Response:

```json
{
  "id": "uuid",
  "slug": "acme",
  "display_name": "Acme",
  "personal_org": false,
  "role": "admin",
  "safe": { "status": "not_deployed" },
  "created_at": "2026-05-06T12:00:00Z"
}
```

```http
GET    /v1/orgs/:slug
PATCH  /v1/orgs/:slug
DELETE /v1/orgs/:slug
```

Delete response when CVMs exist:

```json
{
  "error": {
    "type": "conflict",
    "code": "org.delete.cvms_exist",
    "message": "Delete all CVMs before deleting this organization.",
    "request_id": "..."
  }
}
```

Org deletion soft-deletes for 30 days and marks `org_networks.state =
'gc_pending'` after the tombstone expires (`00-decisions.md:343-354`).

### 4.4 Members And Invitations

```http
GET  /v1/orgs/:slug/members
POST /v1/orgs/:slug/invitations
{ "email": "bob@example.com", "role": "deploy" }
GET  /v1/orgs/:slug/invitations
POST /v1/invitations/:token/accept
DELETE /v1/orgs/:slug/invitations/:id
```

Role change:

```http
PATCH /v1/orgs/:slug/members/:user_id
{ "role": "admin" }
```

If the role change affects Safe signers, response is `202`:

```json
{
  "member": {
    "user_id": "uuid",
    "role": "viewer",
    "status": "pending_removal"
  },
  "safe_transaction": {
    "id": "uuid",
    "kind": "remove_owner",
    "status": "awaiting_signature",
    "sign_url": "/v1/orgs/acme/safe/transactions/uuid/sign"
  }
}
```

Removing the last admin returns `409 org.last_admin`.

### 4.5 API Tokens

```http
GET  /v1/orgs/:slug/tokens
POST /v1/orgs/:slug/tokens
{
  "name": "github-actions-prod",
  "role": "deploy",
  "expires_in_days": 90
}
DELETE /v1/orgs/:slug/tokens/:id
```

Create response returns the secret once:

```json
{
  "id": "uuid",
  "name": "github-actions-prod",
  "role": "deploy",
  "token": "dst_6W5...",
  "token_prefix": "dst_6W5",
  "expires_at": "2026-08-04T12:00:00Z"
}
```

The create transaction reads the creator's current role with `FOR UPDATE`,
compares role rank, and inserts only if `requested_rank <= creator_rank`.
Creator role is recorded as `creator_role_at_issue`. Later demotion does not
cascade or revoke (`00-decisions.md:86-93`).

### 4.6 CVMs

List:

```http
GET /v1/orgs/:slug/cvms?state=running&limit=50&cursor=...
```

Create:

```http
POST /v1/orgs/:slug/cvms
Idempotency-Key: deploy-2026-05-06-prod

{
  "name": "api-prod",
  "image": "dstack-0.5.6",
  "compose": { "services": { "...": "..." } },
  "encrypted_env": "base64...",
  "user_config": "base64-or-string",
  "vcpu": 4,
  "memory_mb": 8192,
  "disk_size_gb": 80,
  "networking": { "mode": "user" },
  "ports": [
    { "protocol": "tcp", "host_port": 443, "vm_port": 443 }
  ],
  "registry_credential_ids": ["uuid"],
  "gateway_enabled": true,
  "no_tee": false
}
```

Response:

```json
{
  "id": "uuid",
  "name": "api-prod",
  "app_id": "0x...",
  "compose_hash": "0x...",
  "desired_state": "running",
  "observed_state": "provisioning",
  "operation": {
    "id": "uuid",
    "status_url": "/v1/orgs/acme/cvms/uuid"
  }
}
```

Other routes:

```http
GET    /v1/orgs/:slug/cvms/:id
PATCH  /v1/orgs/:slug/cvms/:id
DELETE /v1/orgs/:slug/cvms/:id
POST   /v1/orgs/:slug/cvms/:id/start
POST   /v1/orgs/:slug/cvms/:id/stop
POST   /v1/orgs/:slug/cvms/:id/restart
GET    /v1/orgs/:slug/cvms/:id/logs?channel=serial&follow=true
GET    /v1/orgs/:slug/cvms/:id/events
GET    /v1/orgs/:slug/cvms/:id/attestation
```

`Deploy` may create/start/stop/restart and view logs. Only `Admin` may delete.

### 4.7 Images

These are guest OS images, not app container images.

```http
GET  /v1/images
GET  /v1/images/:name
POST /v1/admin/images/seed
POST /v1/admin/images/pull
PATCH /v1/admin/images/:id
DELETE /v1/admin/images/:id
```

`POST /v1/admin/images/pull`:

```json
{
  "registry_ref": "ghcr.io/dstack/guest-image:0.5.6",
  "name": "dstack-0.5.6"
}
```

Today's VMM already has an OCI client for pulling guest OS images from external
registries into the global image catalog (`vmm/src/app/registry.rs:5-20`,
`vmm/src/app/registry.rs:89-151`). That is distinct from hosting an OCI registry
for tenant application containers.

### 4.8 External Registry Credentials

```http
GET    /v1/orgs/:slug/registry-creds
POST   /v1/orgs/:slug/registry-creds
GET    /v1/orgs/:slug/registry-creds/:id
PATCH  /v1/orgs/:slug/registry-creds/:id
DELETE /v1/orgs/:slug/registry-creds/:id
POST   /v1/orgs/:slug/registry-creds/:id/verify
```

Create basic/PAT:

```json
{
  "display_name": "ghcr-prod",
  "registry_url": "https://ghcr.io",
  "auth_method": "ghcr_pat",
  "username": "acme-bot",
  "password": "ghp_..."
}
```

Create ECR:

```json
{
  "display_name": "aws-prod",
  "registry_url": "123456789012.dkr.ecr.us-east-1.amazonaws.com",
  "auth_method": "ecr_sts",
  "ecr_region": "us-east-1",
  "ecr_role_arn": "arn:aws:iam::123456789012:role/dstack-pull",
  "external_id": "dstack-org-uuid"
}
```

Response redacts secrets:

```json
{
  "id": "uuid",
  "display_name": "ghcr-prod",
  "registry_host": "ghcr.io",
  "auth_method": "ghcr_pat",
  "username": "acme-bot",
  "last_verified_at": null,
  "created_at": "2026-05-06T12:00:00Z"
}
```

### 4.9 Safe / On-Chain

```http
GET  /v1/orgs/:slug/safe
GET  /v1/orgs/:slug/safe/transactions
GET  /v1/orgs/:slug/safe/transactions/:id
POST /v1/orgs/:slug/safe/transactions/:id/sign
POST /v1/orgs/:slug/safe/transactions/:id/submit
POST /v1/orgs/:slug/safe/recovery-intents
POST /v1/orgs/:slug/safe/recovery-intents/:id/execute
```

`POST /sign` response:

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

### 4.10 Audit

```http
GET /v1/orgs/:slug/audit?limit=50&cursor=...
GET /v1/orgs/:slug/audit/export?format=json&from=...&to=...
GET /v1/admin/audit?org_id=...&action=...
GET /v1/admin/audit/export?format=json
```

Org admins can export their org slice. Platform admins can export global audit.
No signed-batch export endpoint exists in v1.

### 4.11 Quotas And Usage

```http
GET /v1/orgs/:slug/quotas
GET /v1/orgs/:slug/usage?window=1h
GET /v1/admin/orgs/:slug/quota-overrides
POST /v1/admin/orgs/:slug/quota-overrides
DELETE /v1/admin/orgs/:slug/quota-overrides/:id
```

Override create:

```json
{
  "dimension": "max_vcpu",
  "hard_value": 64,
  "soft_value": 52,
  "reason": "paid pilot contract",
  "expires_at": "2026-08-01T00:00:00Z"
}
```

### 4.12 Platform Admin

```http
GET   /v1/admin/hosts
GET   /v1/admin/hosts/:id
GET   /v1/admin/orgs
PATCH /v1/admin/orgs/:slug
GET   /v1/admin/users
PATCH /v1/admin/users/:id
GET   /v1/admin/secrets
PUT   /v1/admin/secrets/:name
GET   /v1/admin/network/org-networks
POST  /v1/admin/network/reconcile
GET   /v1/admin/metrics-config
PATCH /v1/admin/metrics-config
```

## 5. Auth Flow

### 5.1 Privy Sign-In To Session Cookie

```text
Browser                  Privy                  Control Plane                 Postgres
  |                        |                           |                          |
  | Sign in email/OAuth    |                           |                          |
  |----------------------->|                           |                          |
  | Privy access token     |                           |                          |
  |<-----------------------|                           |                          |
  | POST /auth/privy/exchange {token}                  |                          |
  |--------------------------------------------------->|                          |
  |                        | Verify token / fetch user |                          |
  |                        |<------------------------->|                          |
  |                        |                           | UPSERT users/wallets     |
  |                        |                           | create personal org if new
  |                        |                           | create session row       |
  |                        |                           |------------------------->|
  | Set-Cookie dstack_session                          |                          |
  |<---------------------------------------------------|                          |
```

First successful user sign-in after install atomically sets
`users.is_platform_admin = true` and writes `audit.platform_admin.bootstrap`
(`00-decisions.md:75-80`, `00-decisions.md:343-347`).

### 5.2 API Token Use

```text
Client                         Control Plane                         Postgres
  | Authorization: Bearer dst_...     |                                  |
  |---------------------------------->| sha256(token), lookup active row |
  |                                   |--------------------------------->|
  |                                   | org active? token not expired?   |
  |                                   | load token role as AuthContext   |
  |                                   | update last_used_at asynchronously
  |                                   | route RBAC check                 |
  |<----------------------------------| JSON response                    |
```

Tokens are hashes only. Token values never enter audit payloads.

### 5.3 Token Issuance With Role ≤ Creator

```text
Admin/Deploy user             Control Plane                         Postgres
  | POST /orgs/acme/tokens {role}     |                                  |
  |---------------------------------->| BEGIN                            |
  |                                   | SELECT membership FOR UPDATE     |
  |                                   |--------------------------------->|
  |                                   | compare requested rank <= current |
  |                                   | insert api_tokens(token_hash,     |
  |                                   |   role, creator_role_at_issue)   |
  |                                   | insert audit auth.api_token.created
  |                                   | COMMIT                           |
  |<----------------------------------| token shown once                 |
```

If a Deploy member requests Admin, return `403 token.role_exceeds_creator`.
Later creator demotion has no cascade by design (`00-decisions.md:86-93`).

### 5.4 Admin Role Change To Safe Reconfiguration

Admin addition:

```text
Admin A                  Control Plane                 Privy Wallet              Base / Safe
  | PATCH member role=admin   |                              |                         |
  |-------------------------->| create Safe addOwner tx      |                         |
  |                           | status=awaiting_signature    |                         |
  |<--------------------------| 202 sign required            |                         |
  | open sign flow            |                              |                         |
  |--------------------------------------------------------->| sign SafeTx typed data  |
  |                           | store safe_signature         |                         |
  |                           | submit tx if threshold met   |------------------------>|
  |                           | poll receipt confirmed       |<------------------------|
  |                           | update membership role admin |
  |                           | audit member.role_changed    |
```

Off-chain role is promoted only after the Safe update confirms, because admins
are the Safe owners (`00-decisions.md:100-109`).

### 5.5 Admin Removal With Pending Removal

```text
Admin A                  Control Plane                 Privy Wallet              Base / Safe
  | DELETE member Admin B     |                              |                         |
  |-------------------------->| verify not last admin        |                         |
  |                           | set membership.status=pending_removal
  |                           | create Safe removeOwner tx   |                         |
  |<--------------------------| 202 pending_removal          |                         |
  | sign SafeTx               |----------------------------->|                         |
  |                           | submit + confirm             |------------------------>|
  |                           | delete membership row        |
  |                           | audit member.removed         |
```

While pending, Admin B cannot perform new Admin-only actions but remains visible
as a pending Safe signer until the chain catches up. Last-admin removal is
blocked before any Safe transaction is created (`00-decisions.md:105-112`).

## 6. On-Chain Integration

Chain is Base mainnet, chain ID 8453. Operators can configure an alternate EVM
RPC for test installs (`00-decisions.md:95-99`). Use `alloy` as the Rust EVM
stack: `alloy-provider` for JSON-RPC, `alloy-primitives` for `Address/B256`,
`alloy-sol-types` for ABI encoding, and hand-rolled bindings for Safe v1.4.1
and the existing dstack contracts. The synthesis target remains alloy 0.8 per
the decisions implementation note (`00-decisions.md:389-397`); pin the exact
crate versions when implementation starts.

Safe version: Safe contracts v1.4.1 on Base, using the canonical Safe singleton,
SafeProxyFactory, CompatibilityFallbackHandler, and MultiSendCallOnly addresses
for chain 8453. The control plane stores these in config, validates bytecode on
first use, and refuses to deploy if addresses do not match expected code hashes.

Existing dstack contracts:

- `DstackKms` is upgradeable Ownable/UUPS and stores registered app contracts in
  `registeredApps` (`kms/auth-eth/contracts/DstackKms.sol:16-22`,
  `kms/auth-eth/contracts/DstackKms.sol:38-40`).
- It can deploy and register a `DstackApp` proxy via `deployAndRegisterApp`,
  passing `initialOwner`, upgrade flags, device policy, and initial compose hash
  (`kms/auth-eth/contracts/DstackKms.sol:143-168`).
- `DstackApp` is Ownable/UUPS and gates compose hash and device mutations with
  `onlyOwner` (`kms/auth-eth/contracts/DstackApp.sol:16-27`,
  `kms/auth-eth/contracts/DstackApp.sol:152-190`).
- `DstackApp.isAppAllowed` checks the compose hash and device policy during
  boot authorization (`kms/auth-eth/contracts/DstackApp.sol:193-217`).

### 6.1 app_id Derivation

The control plane derives:

```text
compose_hash = sha256(canonical_app_compose_json)
app_id       = first_20_bytes(sha256(org_id_uuid_bytes || compose_hash))
```

Represent `app_id` as a 20-byte EVM address-shaped value. This fixes cross-org
collisions without contract changes because the existing KMS flow already treats
`appId` as a 20-byte address-like field in boot checks
(`kms/auth-eth/src/ethereum.ts:34-52`; `kms/auth-eth/contracts/IAppAuth.sol`
defines `address appId`, found via report references). Today's VMM derives an
app id from compose when not provided (`vmm/src/main_service.rs:169-172`); v2
makes org_id part of the derivation.

### 6.2 Safe Deployment

The Safe is deployed lazily on the org's first on-chain action:

1. Load current Admin memberships and primary signer wallets.
2. Build Safe proxy deployment with owners = admin signer addresses plus the
   optional recovery signer if enabled.
3. Threshold is `1`.
4. Store `safes` row after deployment transaction is confirmed.
5. Insert `audit.safe.deployed`.

The first `DstackApp` for an org is then deployed with `initialOwner =
safe_address`, so all future `addComposeHash`, `removeComposeHash`,
`addDevice`, and `removeDevice` actions route through Safe transactions.

### 6.3 Admin Sync

The source of truth for signers is current Admin memberships. The control plane
polls Safe owners and stores `safes.signer_cache` only as a cache. A mismatch
between DB admins and Safe owners is surfaced as an org setting warning and
repair action. Admin additions/removals follow the sequences in §5.

### 6.4 Recovery Signer

Recovery signer is opt-in at install (`00-decisions.md:118-125`). If enabled:

- Generate one secp256k1 key at install.
- Encrypt private key in `secrets` under `name = 'platform.recovery_signer'`.
- Add its address as a Safe owner for new org Safes.
- Enforce a 24-hour timelock through Safe transactions:
  - `recovery_intent`: a Safe transaction or module call records target org,
    intended action, calldata hash, and `not_before = now + 24h`.
  - `recovery_execute`: after `not_before`, the recovery signer can sign/submit
    the exact calldata.

The implementation should prefer a minimal audited timelock helper contract
owned by the recovery signer rather than an off-chain-only wait. The DB
`safe_transactions.not_before` mirrors the on-chain timelock for UX, but the
chain enforces the delay. If no helper contract is available at v1 build time,
ship recovery signer disabled by default and keep the config flag hidden behind
an explicit "experimental recovery" warning.

Differs from prior plan: Safe remains in v1 despite the skeptic suggesting
deferral; the revised decisions kept Privy and Safe. Signed audit batches were
dropped; Safe is not used for audit signing.

## 7. CVM Lifecycle In The New World

### 7.1 Deploy Flow

```text
API request
  |
  v
Auth/RBAC: Admin or Deploy
  |
  v
Validate compose, image, resources, registry refs
  |
  v
BEGIN DB transaction
  - lock org quota rows
  - derive compose_hash and app_id
  - ensure Safe/KMS app transaction if needed
  - insert cvms row observed_state='provisioning'
  - insert cvm_artifacts revisions
  - insert audit cvm.create.accepted with prev_hash/row_hash
COMMIT
  |
  v
Host worker task
  - ensure per-org uid
  - if bridge mode: ensure org_networks + bridge/dnsmasq/nftables
  - materialise workdir from DB
  - materialise .docker/config.json from org registry creds
  - invoke QEMU/supervisor
  - update cvms observed_state
  |
  v
Guest boots, reports instance.info and boot progress over host API
```

Today's VMM writes compose, encrypted env, user config, and instance info into
the shared directory before launch (`vmm/src/app.rs:964-993`) and builds QEMU
from a workdir (`vmm/src/app/qemu.rs:388-432`). v2 preserves that guest contract
but changes the source of truth: DB rows are authoritative, files are
materialised artifacts.

### 7.2 DB State Versus Disk State

Authoritative in Postgres:

- Org ownership, creator, host placement.
- Desired state and observed state.
- Resource declaration.
- Compose hash and app id.
- Artifact revision history.
- Registry credential references.
- Guest IP and lease metadata.
- Audit trail.

Authoritative on disk:

- `hda.img`, because it is a large sparse disk.
- QEMU sockets and pid files.
- Serial/stdout/stderr logs.
- Materialised shared files at the instant of start.

Reconciliation rule: if DB says a CVM exists and disk is missing, the worker
attempts to rematerialise everything except `hda.img`. Missing `hda.img` is a
hard error requiring operator recovery because tenant disk data is gone. If disk
has a workdir with no DB row in greenfield v1, classify as orphan and do not
adopt automatically.

### 7.3 Per-Org Sandbox Uid

On first CVM for an org:

1. Compute uid from an operator-managed uid range, store on `cvms.sandbox_uid`
   and eventually on an org sandbox cache.
2. Create system user `dstack-o-<short>` with no login shell.
3. Add to `kvm` group and any required device groups.
4. Create `/var/lib/dstack/orgs/<org-id>` owned by that uid, mode `0750`.
5. Install owner-matched localhost guard rules analogous to today's
   `setup-user.sh`, which creates a `DSTACK_SANDBOX_<user>` chain and drops
   non-allowed TCP/UDP traffic to host loopback for that uid
   (`vmm/src/setup-user.sh:97-181`).

This extends today's single-sandbox-uid idea to per-org sandboxes
(`00-decisions.md:248-249`).

### 7.4 Per-Org Bridge Provisioning

If `networking.mode = "bridge"`:

1. Allocate or load `org_networks` for the org on this host.
2. Reconcile bridge link, bridge address, dnsmasq, nftables, and
   `/etc/qemu/bridge.conf`.
3. Set `cvms.bridge_name`.
4. Pass the per-org bridge name into the QEMU networking resolver.

The QEMU argument shape remains `bridge,id=net0,br=<bridge>`; the important
change is that `<bridge>` comes from `org_networks`, not global config. This is
the clean code touch-point identified by the F3 report
(`12-f3-vlan-isolation.md:134-148`) and Codex opinion
(`13-codex-bridge-opinion.md:26-43`).

User-mode remains the default and uses QEMU SLIRP. Current user-mode assembly is
per-VM `user,id=net0,net=...,dhcpstart=...,restrict=...`
(`vmm/src/app/qemu.rs:504-522`), which is why it does not join the host bridge.

### 7.5 Registry Credential Materialisation

The deploy validator parses image references in compose. For every registry
host requiring auth:

1. Find an active `org_registry_creds` row for that host, or fail admission with
   `registry.credentials_missing`.
2. Decrypt the referenced `secrets` row under the KMK.
3. For cloud registries, mint short-lived pull credentials if applicable
   (ECR STS, GAR workload identity).
4. Build a minimal Docker config:

```json
{
  "auths": {
    "ghcr.io": {
      "username": "acme-bot",
      "password": "redacted-at-rest",
      "auth": "base64(username:password)"
    }
  }
}
```

5. Store it as encrypted `cvm_artifacts.kind = 'docker-config'`.
6. Materialise it at
   `/var/lib/dstack/orgs/<org-id>/cvms/<cvm-id>/shared/.docker/config.json`
   immediately before start.
7. Include a path hint in `.user-config` or the existing encrypted-env channel
   so the guest agent/container runtime uses it. The revised decisions explicitly
   call for deploy-time `.docker/config.json` materialisation through the guest
   shared/encrypted channel, not a host registry (`00-decisions.md:225-235`).

### 7.6 Recovery On Restart

Startup order:

1. Load config and KMK. Refuse to start API if KMK cannot decrypt a canary
   secret.
2. Connect to Postgres; run migrations.
3. Load the single `hosts` row and mark `status='initializing'`.
4. Reconcile host networking from `org_networks` before starting bridge-mode
   CVMs.
5. Query supervisor for running QEMU processes and their annotations.
6. Query `cvms` for this `host_id`.
7. For each CVM:
   - If desired running and QEMU running, set observed running.
   - If desired running and QEMU absent, rematerialise artifacts and start.
   - If desired stopped and QEMU running, stop unless the host crashed mid-stop
     and `.removing` exists.
   - If observed removing or `.removing`, resume removal.
8. Restore port forwards for bridge-mode CVMs with `guest_ip`.
9. Mark host online; `/readyz` can return 200.

Today's reload path scans workdirs, preserves running CIDs, resumes `.removing`
cleanup, and restores port forwarding from persisted guest IPs
(`vmm/src/app.rs:560-620`). v2 keeps that reconciliation behavior but drives it
from DB desired state first.

## 8. External Registry Credential Handling

This section replaces the old OCI registry implementation entirely.

### 8.1 Credential Model

Org admins create one or more registry credentials. The credential metadata
lives in `org_registry_creds`; the secret payload lives in `secrets` as
sodium-secretbox ciphertext. The audit payload records registry host, auth
method, creator, and verification status, but never the password/token.

Supported v1 auth methods:

| Registry | Method | Secret payload |
|---|---|---|
| Docker Hub | `dockerhub_pat` or `basic` | username + PAT/password |
| GHCR | `ghcr_pat` | username + PAT with package read scope |
| GitLab Registry | `gitlab_pat` | username + deploy token/PAT |
| Generic OCI | `basic` or `bearer` | username/password or bearer token |
| ECR | `ecr_sts` | role ARN, external ID, optional bootstrap credentials; runtime gets ECR auth token through STS |
| GAR | `gar_service_account` or `gar_workload_identity` | service account JSON encrypted, or workload identity config |

### 8.2 Deploy Pull Flow

```text
POST /orgs/acme/cvms
        |
        v
Parse compose image refs
        |
        v
registry host -> org_registry_creds lookup
        |
        v
decrypt secrets with KMK / mint cloud token
        |
        v
write shared/.docker/config.json for this CVM
        |
        v
QEMU boots guest
        |
        v
guest agent / docker compose pulls directly from GHCR/ECR/GAR/etc.
```

The control plane never proxies image layers and never stores app image blobs.
External registry downtime fails deploys fast with a clear registry-host error;
running CVMs continue unaffected (`00-decisions.md:316-329`).

### 8.3 Rotation And Revocation

Rotation creates a new `secrets` version and updates `org_registry_creds.secret_id`.
Existing running CVMs are not mutated. New deploys and restarts use the new
version. Revocation sets `org_registry_creds.revoked_at`; deploy admission fails
for image refs that only match revoked credentials. An admin can force a CVM
restart after rotation to rematerialise the new Docker config.

Audit events:

```text
registry_cred.created
registry_cred.verified
registry_cred.rotated
registry_cred.revoked
registry_cred.deploy_materialized
registry_cred.deploy_missing
```

Differs from prior plan: saves the prior registry-server effort and removes
registry blob backup/GC/security-review concerns, but adds careful secret
materialisation and cloud-provider token adapters.

## 9. Audit Log

Audit is append-only at the API layer and hash-chained at write time. There are
no signed batches in v1 (`00-decisions.md:185-203`).

### 9.1 Canonical Row Encoding

For row `n`, compute:

```text
prev_hash = row_hash of row n-1, or 32 zero bytes for the first row

canonical_payload =
  "dstack-audit-v1\n" ||
  id decimal ascii || "\n" ||
  created_at RFC3339 nanos UTC || "\n" ||
  org_id or "" || "\n" ||
  host_id or "" || "\n" ||
  actor_user_id or "" || "\n" ||
  actor_token_id or "" || "\n" ||
  actor_platform_admin "true|false" || "\n" ||
  actor_ip or "" || "\n" ||
  request_id or "" || "\n" ||
  action || "\n" ||
  target_kind or "" || "\n" ||
  target_id or "" || "\n" ||
  result || "\n" ||
  error_code or "" || "\n" ||
  canonical_json(payload) || "\n" ||
  hex(prev_hash)

row_hash = sha256(canonical_payload)
```

`canonical_json` means RFC 8785 JSON Canonicalization Scheme or an equivalent
deterministic serializer with sorted object keys, UTF-8, no insignificant
whitespace, and stable number formatting. Use one implementation everywhere.

The insert function locks a singleton audit cursor row or uses `SELECT id,
row_hash FROM audit_log ORDER BY id DESC LIMIT 1 FOR SHARE` under serializable
transaction isolation. The goal is no forks in the hash chain under concurrent
requests.

### 9.2 Events

Capture at least:

```text
auth.signin.success / auth.signin.failure / auth.logout / auth.session.revoked
auth.api_token.created / auth.api_token.revoked / auth.api_token.denied
org.created / org.updated / org.deleted / org.suspended
member.invited / invitation.accepted / invitation.revoked
member.role_change.proposed / member.role_changed / member.pending_removal / member.removed
cvm.create.accepted / cvm.provisioned / cvm.started / cvm.stopped / cvm.restarted
cvm.updated / cvm.deleted / cvm.crashed / cvm.boot.error
registry_cred.created / registry_cred.rotated / registry_cred.revoked
safe.transaction.proposed / safe.transaction.signed / safe.transaction.submitted
safe.transaction.confirmed / safe.transaction.failed
quota.soft_warning / quota.hard_denied / quota.override.created / quota.override.revoked
network.org_bridge.allocated / network.org_bridge.ready / network.org_bridge.gc
platform_admin.promoted / platform_secret.updated / host.reconciled
```

Report 10's audit inventory is the baseline (`10-multitenant-design-space.md:735-804`).
Report 08 calls out that today's VMM has no audit log and cannot answer "who
deleted CVM X" (`08-production-ops-gaps.md:646-705`).

### 9.3 Optional Anchor Row

An hourly `audit.anchor` row may be inserted with payload:

```json
{
  "anchor_kind": "hourly_local",
  "covered_first_id": 123,
  "covered_last_id": 456,
  "tip_hash": "0x..."
}
```

This makes expected continuity easier to inspect in exports, but it is not a
signature and does not create a separate trust root. If implementation time is
tight, omit anchor rows; the hash chain is sufficient for v1.

### 9.4 Export

`GET /v1/orgs/:slug/audit/export?format=json` streams JSON lines:

```json
{"id":123,"created_at":"...","action":"cvm.started","payload":{},"prev_hash":"0x...","row_hash":"0x..."}
```

CSV export can follow, but JSON is v1's verification-friendly format. Include a
small verifier command in the CLI that recomputes the chain over exported rows.

## 10. Quota Enforcement

Quota dimensions are max CVMs, max vCPU, max RAM, max disk, max ingress ports,
max image storage, and deploys per hour (`00-decisions.md:205-217`).

### 10.1 Effective Limit Query

Use one COALESCE-style query per org/dimension:

```sql
WITH override AS (
    SELECT hard_value, soft_value
    FROM quota_overrides
    WHERE org_id = $1
      AND dimension = $2
      AND revoked_at IS NULL
      AND (expires_at IS NULL OR expires_at > now())
    ORDER BY created_at DESC
    LIMIT 1
)
SELECT
    COALESCE((SELECT hard_value FROM override),
             CASE $2
               WHEN 'max_cvms' THEN q.max_cvms_hard::bigint
               WHEN 'max_vcpu' THEN q.max_vcpu_hard::bigint
               WHEN 'max_ram_mb' THEN q.max_ram_mb_hard::bigint
               WHEN 'max_disk_gb' THEN q.max_disk_gb_hard::bigint
               WHEN 'max_ingress_ports' THEN q.max_ingress_ports_hard::bigint
               WHEN 'max_image_storage_bytes' THEN q.max_image_storage_bytes_hard
               WHEN 'deploys_per_hour' THEN q.deploys_per_hour_hard::bigint
             END) AS hard_limit,
    COALESCE((SELECT soft_value FROM override),
             CASE $2
               WHEN 'max_cvms' THEN q.max_cvms_soft::bigint
               WHEN 'max_vcpu' THEN q.max_vcpu_soft::bigint
               WHEN 'max_ram_mb' THEN q.max_ram_mb_soft::bigint
               WHEN 'max_disk_gb' THEN q.max_disk_gb_soft::bigint
               WHEN 'max_ingress_ports' THEN q.max_ingress_ports_soft::bigint
               WHEN 'max_image_storage_bytes' THEN q.max_image_storage_bytes_soft
               WHEN 'deploys_per_hour' THEN q.deploys_per_hour_soft::bigint
             END) AS soft_limit
FROM quotas q
WHERE q.org_id = $1;
```

### 10.2 Admission-Time Checks

At API create/update/start admission:

1. Begin transaction.
2. Lock org quota row and relevant CVM rows.
3. Compute proposed resource usage:
   - CVM count: non-deleted CVMs.
   - vCPU/RAM/disk: declared resources for non-deleted CVMs plus request delta.
   - ingress ports: requested mapped ports.
   - deploys/hour: audit or dedicated counter rows in the trailing hour.
   - image storage: global guest image storage is platform-level in v1; org app
     images are external, so this dimension mostly guards future per-org guest
     image features and registry credential metadata.
4. Compare hard limit. If exceeded, insert `quota.hard_denied` audit and return
   403/409 depending on policy vs transient host exhaustion.
5. Compare soft limit. If crossed, allow request but emit in-app notification
   and `quota.soft_warning` audit.
6. Insert/modify CVM rows and commit.

Report 10 places tenant quota admission in the control plane and host caps in
the node/worker (`10-multitenant-design-space.md:706-731`).

### 10.3 Allocation-Time Checks

Immediately before QEMU allocation, the host worker rechecks:

- Host free vCPU/RAM/disk.
- CID pool availability.
- GPU slots not already attached.
- Bridge subnet/lease availability for bridge mode.
- Disk pressure on `/var/lib/dstack/orgs` and image paths.

This double-check catches races and host drift. Today's VMM has in-memory CID
allocation and declared resources but no org accounting (`vmm/src/app.rs:151-165`,
`08-production-ops-gaps.md:603-643`).

### 10.4 Soft Warning Delivery

Soft warnings write:

- `audit_log` action `quota.soft_warning`.
- `notifications` can be added in implementation if UI ships; otherwise expose
  warnings in `GET /v1/orgs/:slug/quotas`.
- Structured log with `event=quota.soft_warning`.

No CVMs are killed when an override expires or quota is reduced. New deploys are
blocked until usage falls under hard limit.

## 11. Per-Org Linux Bridge Plumbing

This is the largest new workstream relative to the original rewrite. The F3
investigation and Codex second opinion agree that bridge-mode CVMs on one shared
bridge can reach each other at L2/L3, while gateway WireGuard only protects
ingress (`12-f3-vlan-isolation.md:11-25`, `13-codex-bridge-opinion.md:9-24`).
Per-org bridges are the v1 isolation primitive.

### 11.1 IPAM

Config:

```toml
[network.bridge_ipam]
enabled = true
supernet = "10.42.0.0/16"
per_org_prefix_len = 24
gateway_offset = 1
dhcp_start_offset = 10
dhcp_end_offset = 254
bridge_name_template = "dstack-org-{short_org_id}"
uplink_iface = "auto"
mtu = 1500
```

Allocation algorithm:

1. API admits first bridge-mode CVM for org.
2. Begin transaction.
3. `SELECT * FROM org_networks WHERE org_id=$org AND host_id=$host FOR UPDATE`.
4. If absent, allocate the first free `/24` inside host supernet:

```sql
-- Implementation can use inet/cidr arithmetic in Rust for clarity, but the
-- allocation must be committed under a uniqueness constraint on (host_id, subnet).
INSERT INTO org_networks (...)
VALUES (...)
ON CONFLICT (host_id, subnet) DO NOTHING;
```

5. Retry on unique violation until success.
6. Bridge name is deterministic from org id, e.g. `dstack-org-018f2f3a`.
7. Store `gateway_ip = first usable`, DHCP range `.10-.254`.

A `/16` yields 256 `/24` orgs, so supernet must be operator-configurable. The
Codex opinion explicitly warns not to hard-code `/16` as the only option
(`13-codex-bridge-opinion.md:34-35`).

### 11.2 Bridge Lifecycle

Lazy create on first bridge-mode CVM:

```bash
ip link add name dstack-org-018f2f3a type bridge
ip addr add 10.42.17.1/24 dev dstack-org-018f2f3a
ip link set dev dstack-org-018f2f3a mtu 1500
ip link set dev dstack-org-018f2f3a up
```

Then update `/etc/qemu/bridge.conf`:

```text
allow dstack-org-018f2f3a
```

The bridge must exist and be allowlisted before QEMU starts because QEMU's
bridge helper creates a TAP and attaches it to the bridge
(`docs/bridge-networking.md:156-178`, `13-codex-bridge-opinion.md:38-39`).

Never mutate bridge lifecycle from the QEMU child. The host network reconciler
owns bridge state.

### 11.3 DHCP

Run one dnsmasq per bridge for v1:

```ini
interface=dstack-org-018f2f3a
bind-interfaces
except-interface=lo
dhcp-range=10.42.17.10,10.42.17.254,255.255.255.0,12h
dhcp-option=option:router,10.42.17.1
dhcp-option=option:dns-server,10.42.17.1
dhcp-authoritative
leasefile-ro
dhcp-script=/usr/lib/dstack/dhcp-lease-notify
```

The generated wrapper sends a message to `/run/dstack/dhcp-leases.sock`:

```json
{
  "bridge": "dstack-org-018f2f3a",
  "org_id": "uuid",
  "action": "add|old|del",
  "mac": "02:...",
  "ip": "10.42.17.21",
  "hostname": "optional",
  "lease_expires_at": "..."
}
```

Today's bridge docs use dnsmasq `dhcp-script` to notify VMM
(`docs/bridge-networking.md:87-113`), and current VMM matches MAC to VM and
writes `guest-ip` (`vmm/src/app.rs:421-447`). Codex flagged the existing
notification shape as spoofable/cross-org-drift-prone and recommended a private
local channel carrying bridge identity (`13-codex-bridge-opinion.md:55-56`).

On bridge changes, write a new config file, validate it with `dnsmasq
--test --conf-file=...`, then restart only that bridge's dnsmasq unit. Record
`org_networks.dnsmasq_config_hash` and `dnsmasq_pid`.

### 11.4 NAT And Forwarding

Use nftables sets/maps rather than one rule block per org:

```nft
table inet dstack {
  set org_bridges {
    type ifname
    elements = { "dstack-org-018f2f3a", "dstack-org-..." }
  }

  set org_subnets_v4 {
    type ipv4_addr
    flags interval
    elements = { 10.42.17.0/24, 10.42.18.0/24 }
  }

  chain input {
    type filter hook input priority filter; policy accept;
    iifname @org_bridges udp dport 67 accept
    iifname @org_bridges udp dport 53 accept
    iifname @org_bridges tcp dport 53 accept
  }

  chain forward {
    type filter hook forward priority filter; policy drop;
    ct state established,related accept
    iifname @org_bridges oifname @org_bridges drop
    iifname @org_bridges oifname $uplink accept
    oifname @org_bridges ct state established,related accept
  }
}

table ip dstack_nat {
  chain postrouting {
    type nat hook postrouting priority srcnat; policy accept;
    ip saddr @org_subnets_v4 ip daddr 224.0.0.0/24 return
    ip saddr @org_subnets_v4 ip daddr 255.255.255.255 return
    ip saddr @org_subnets_v4 oifname $uplink masquerade
  }
}
```

The default drop between sibling org bridges is mandatory. Existing bridge docs
show the single-bridge DHCP/DNS/FORWARD/NAT requirements
(`docs/bridge-networking.md:118-146`); v2 parameterizes them by org bridge.

### 11.5 Reconciliation Loop

On startup and every 60 seconds:

1. Read desired bridges from `org_networks` for this host.
2. Read actual links matching `dstack-org-*`.
3. For each desired bridge:
   - ensure link exists, address/MTU match, link up.
   - ensure qemu allowlist line exists.
   - ensure dnsmasq config and process are healthy.
   - ensure nftables sets include bridge/subnet.
   - update `state='ready'` or `state='error'`.
4. For actual bridges not desired:
   - if no enslaved TAPs and no active dnsmasq, mark stale-empty.
   - if enslaved TAPs exist, mark stale-in-use in memory/metrics and never
     delete automatically.
5. For desired bridges with no active bridge-mode CVMs:
   - keep allocated until org tombstone expiry or GC grace period.

Never delete a bridge with enslaved TAPs. Bridge deletion while VMs run breaks
guest networking; current bridge docs warn that destroying/recreating a bridge
detaches TAPs and requires VM restart (`docs/bridge-networking.md:193-198`).

### 11.6 Garbage Collection

GC can reclaim a subnet only when:

- org status is `deleted`;
- `tombstone_expires_at < now()`;
- no CVMs exist for the org;
- bridge has zero enslaved TAPs;
- dnsmasq is stopped;
- nftables sets have been updated;
- audit row `network.org_bridge.gc` is written.

After GC, delete `org_networks` or mark it `gc_pending` with a `deleted_at`
field if historical allocation inspection is useful. Because audit has the
allocation and GC event, deletion is acceptable.

### 11.7 Opt-In Bridge Mode

Bridge mode is opt-in per CVM. User-mode SLIRP does not allocate `org_networks`,
does not start dnsmasq, and does not touch nftables beyond normal host egress.
If bridge orchestration overruns v1, the explicit fallback is to ship only
`mode = "user"` for tenants and hide bridge mode behind a platform-admin unsafe
flag (`00-decisions.md:280-286`, `13-codex-bridge-opinion.md:73-74`).

Differs from prior plan: no VLAN ID columns, no bridge VLAN filtering, no
post-start TAP mutation. The per-org bridge itself is the isolation boundary.

## 12. Observability

### 12.1 Metrics

Expose Prometheus at `/metrics`. Label names must be stable. Avoid high
cardinality labels like CVM name; `org_id` is acceptable for tenant accounting
in this single-host v1.

Core API:

```text
dstack_control_plane_build_info{version,git_rev} 1
dstack_http_requests_total{method,path_template,status,auth_kind}
dstack_http_request_duration_seconds_bucket{method,path_template,status}
dstack_auth_signins_total{provider,outcome}
dstack_api_tokens_total{org_id,state,role}
dstack_sessions_active{state}
```

CVM / host worker:

```text
dstack_cvm_total{org_id,host_id,state,networking_mode}
dstack_cvm_transitions_total{org_id,host_id,from_state,to_state,outcome}
dstack_cvm_start_duration_seconds_bucket{org_id,host_id,networking_mode}
dstack_cvm_deploy_duration_seconds_bucket{org_id,host_id,networking_mode}
dstack_cvm_declared_vcpu{org_id,host_id}
dstack_cvm_declared_memory_mb{org_id,host_id}
dstack_cvm_declared_disk_gb{org_id,host_id}
dstack_host_cid_pool{host_id,state="allocated|free"}
dstack_host_disk_bytes{host_id,path,role="images|orgs|postgres",state="free|used"}
dstack_qemu_processes_total{host_id,state}
dstack_supervisor_request_duration_seconds_bucket{method,outcome}
dstack_supervisor_errors_total{method}
```

Quotas / audit / secrets:

```text
dstack_quota_usage{org_id,dimension}
dstack_quota_limit{org_id,dimension,kind="soft|hard"}
dstack_quota_denials_total{org_id,dimension}
dstack_audit_rows_total
dstack_audit_hash_chain_tip{id}
dstack_audit_write_duration_seconds_bucket
dstack_secret_decrypt_failures_total{scope,name}
```

External registry:

```text
dstack_registry_creds_total{org_id,registry_host,auth_method,state}
dstack_registry_credential_verify_total{org_id,registry_host,outcome}
dstack_registry_materializations_total{org_id,registry_host,outcome}
dstack_registry_cloud_token_refresh_total{provider,outcome}
```

Bridge-mode metrics required by the revised decisions
(`00-decisions.md:300-314`):

```text
dstack_org_bridges_total{host_id,state}
dstack_org_bridge_active{org_id,host_id,bridge}
dstack_dnsmasq_up{org_id,host_id,bridge}
dstack_dnsmasq_restarts_total{org_id,host_id,bridge,outcome}
dstack_dhcp_lease_events_total{org_id,host_id,bridge,action,outcome}
dstack_dhcp_lease_success_rate{org_id,host_id,bridge}
dstack_nftables_set_size{host_id,set}
dstack_nftables_apply_total{host_id,outcome}
dstack_conntrack_entries{host_id}
dstack_conntrack_insert_failures_total{host_id}
dstack_mtu_incidents_total{org_id,host_id,bridge}
dstack_orphaned_bridges_total{host_id,state="stale_empty|stale_in_use"}
dstack_idle_bridge_orgs_total{host_id}
```

Today's VMM has no metrics endpoint and no OTLP tracing
(`08-production-ops-gaps.md:83-140`), so this is new infrastructure.

### 12.2 Logs

Default log format is JSON lines through `tracing-subscriber`. Required fields:

```text
timestamp
level
message
target
request_id
trace_id
span_id
org_id
user_id
actor_token_id
host_id
cvm_id
safe_tx_id
bridge
registry_host
http.method
http.path_template
http.status
error.code
error.message
```

The decisions doc says logs should carry `request_id`, `org_id`, `user_id`, and
`node_id`; use `host_id` as the v1/v2-compatible name while accepting `node_id`
as an alias in dashboards (`00-decisions.md:302-306`).

### 12.3 OTLP

OpenTelemetry is off by default and auto-enabled when
`OTEL_EXPORTER_OTLP_ENDPOINT` is set (`00-decisions.md:300-306`). Use
`tracing-opentelemetry`, propagate W3C TraceContext on outbound HTTP to Privy,
EVM RPC, KMS, gateway, and registry token endpoints. The host worker creates
child spans for QEMU launch, artifact materialisation, bridge reconcile,
dnsmasq restart, and nftables apply.

## 13. Install / First-Run Flow

### 13.1 docker-compose

The installer writes a compose file similar to:

```yaml
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
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

The actual production compose should use peer auth over the Unix socket rather
than TCP trust. The above is illustrative; the build task must produce a tested
Postgres container config where only the host `dstack` uid can access the socket.

### 13.2 Init Command

`dstack-control-plane init`:

1. Create `dstack` system uid if missing.
2. Create `/etc/dstack`, `/var/lib/dstack`, `/run/dstack`.
3. Generate or import KMK.
4. Prompt for base domain.
5. Prompt for Privy app id/secret and webhook secret.
6. Prompt for EVM RPC URL and Base chain config.
7. Prompt for bridge supernet:

```text
Bridge-mode CVMs need per-org private subnets.
Choose an RFC1918 supernet that does not overlap your LAN/VPN.
Default: 10.42.0.0/16 with /24 per org.
```

8. Prompt whether recovery signer is enabled. Default no.
9. Start Postgres.
10. Run migrations.
11. Insert single `hosts` row.
12. Seed base-image catalog from `/var/lib/dstack/images`.
13. Start service.

### 13.3 KMK Wall Of Text

On auto-generation:

```text
+------------------------------------------------------------------------------+
| BACK UP YOUR DSTACK MASTER KEY NOW                                           |
|                                                                              |
| A new 32-byte master key was generated at:                                    |
|   /etc/dstack/master.key                                                     |
|                                                                              |
| This key encrypts Privy secrets, session keys, EVM RPC credentials, optional  |
| recovery signer keys, SMTP credentials, and per-org registry credentials.     |
|                                                                              |
| Losing this key means the deployment can no longer decrypt those secrets.     |
| Existing Postgres data and CVM disks remain present, but registry pulls,      |
| Privy sessions, recovery signer use, and other integrations will break until  |
| secrets are re-provisioned. The key is not stored anywhere else by dstack.    |
|                                                                              |
| Copy it to your operator password manager or offline backup before creating   |
| customer organizations.                                                       |
+------------------------------------------------------------------------------+
```

This implements the decisions doc's "BACK THIS UP NOW" requirement
(`00-decisions.md:163-183`).

### 13.4 First User Promotion

When the first Privy sign-in succeeds and no platform admin exists, the control
plane promotes that user in the same transaction that creates their session.
Audit action: `platform_admin.promoted`.

### 13.5 Recovery Signer

If enabled:

1. Generate secp256k1 key.
2. Store encrypted secret.
3. Show recovery address.
4. Require operator to acknowledge the 24-hour timelock behavior.
5. Store `platform.recovery_signer.enabled=true`.

### 13.6 Base Image Catalog

Seed `images` from existing guest OS image directories. Today's image catalog is
a global filesystem path in config and VMM validates image names against that
path (`vmm/src/app.rs:175-187`). v2 keeps guest OS images global and
operator-managed (`00-decisions.md:239-241`).

## 14. Work Breakdown And Effort Sizing

Sizing: S = 1-2 weeks, M = 3-5 weeks, L = 6-9 weeks, XL = 10-14 weeks for one
engineer, including implementation and focused tests.

### Milestone 0: Foundation

| Workstream | Size | Justification |
|---|---:|---|
| Repo skeleton for `dstack-control-plane` | S | New Axum binary, config loader, service wiring. |
| DB schema + migrations | M | Tables in §3, constraints, indexes, init seed; host_id-ready schema adds care. |
| sqlx integration + transaction helpers | S | Query layer, migrations, test DB harness. |
| KMK + sodium secret store | M | Key loading/generation, per-row encryption, canary, rotation groundwork. |
| Structured errors, idempotency, cursor pagination | M | Cross-cutting API behavior needed before endpoints scale. |

### Milestone 1: Auth, Orgs, RBAC

| Workstream | Size | Justification |
|---|---:|---|
| Privy server integration | M | Token validation, user sync, wallet sync, webhook verification, no Rust SDK assumption. |
| Session cookie auth | S | Cookie HMAC, sessions table, logout/revoke. |
| Org lifecycle + personal orgs | M | Slugs, personal org auto-create, tombstones. |
| RBAC middleware with Admin/Deploy/Billing/Viewer | M | Route policy matrix and tests; Deploy is new but cheap now. |
| API token issuance/use/revocation | M | Hash-only storage, role≤creator issue-time check, no cascade. |
| Invitations + membership flows | M | Email-token model plus Safe pending states for admins. |

### Milestone 2: Audit, Quotas, Observability

| Workstream | Size | Justification |
|---|---:|---|
| Audit hash chain | M | Canonical encoding, concurrent insert correctness, export verifier. |
| Quotas + overrides | M | Admission checks, soft warnings, override effective-limit query. |
| Resource usage sampler | M | Declared usage plus host probes, 30-second rollups. |
| Prometheus metrics | M | Full metric set including bridge metrics. |
| JSON logs + request IDs | S | `tracing` layers and middleware. |
| OTLP integration | S | Conditional exporter and span propagation. |

### Milestone 3: Safe / On-Chain

| Workstream | Size | Justification |
|---|---:|---|
| alloy EVM client + config | M | Base RPC, chain config, tx submit/poll, error mapping. |
| Safe v1.4.1 ABI/calldata | L | Deploy Safe, EIP-712 SafeTx signing, owner sync. |
| Privy signing flow for Safe txs | L | Embedded wallet signing requests and callbacks. |
| DstackKms/DstackApp orchestration | M | app_id derivation, deploy/register, compose/device txs. |
| Admin add/remove Safe sync | M | Pending removal/promote flows, repair mismatches. |
| Optional recovery signer timelock | L | Secure key handling plus on-chain timelock helper integration. |

### Milestone 4: Host Worker / CVM Lifecycle

| Workstream | Size | Justification |
|---|---:|---|
| QEMU/supervisor host worker rewrite/refactor | XL | Extract useful VMM logic, DB-driven desired state, robust reconciliation. |
| Workdir materialisation | M | Generate manifests/shared files from artifacts before start. |
| CVM API endpoints | L | Create/start/stop/restart/delete/update/logs/events with RBAC/audit. |
| Guest host API preservation | M | Keep vsock host API and guest event paths. |
| Restart recovery | L | DB/disk/supervisor reconciliation and edge cases. |
| Disk pressure and host capacity checks | M | Allocation-time safety. |

### Milestone 5: Network Isolation

| Workstream | Size | Justification |
|---|---:|---|
| Per-org uid provisioning | M | User creation, filesystem permissions, owner firewall rules. |
| IPAM + `org_networks` allocation | M | Atomic subnet allocation and collision tests. |
| Bridge lifecycle helper | M | `ip link`, bridge address/MTU, qemu allowlist. |
| dnsmasq per bridge | M | Config generation, process supervision, health checks. |
| nftables sets/maps NAT/drop policy | L | Correct rules, idempotent apply, conntrack awareness. |
| DHCP private lease socket | M | Replace current spoofable channel with bridge-aware local socket. |
| Startup reconciler + GC | L | Desired/stale-empty/stale-in-use classification, safe deletion. |

This is the major new workstream accepted after F3/Codex bridge review. It adds
roughly 6-8 weeks versus a user-mode-only v1.

### Milestone 6: External Registry Credentials

| Workstream | Size | Justification |
|---|---:|---|
| Registry credential CRUD | S | Metadata plus secret writes. |
| Docker config materialiser | M | Compose image-ref parsing and `.docker/config.json` artifact. |
| Docker Hub/GHCR/GitLab/generic support | M | Basic/PAT variants and verification. |
| ECR STS support | M | AssumeRole/token flow, expiry handling. |
| GAR support | M | Service account/workload identity flow. |
| Rotation/revocation handling | S | Secret versioning and deploy behavior. |

This replaces the old OCI registry. Net savings versus the prior plan: about
6-10 weeks from dropping registry server, blob upload/download, token server,
and registry GC (`99-skeptic.md:388-465`, `00-decisions.md:218-241`).

### Milestone 7: Install, CLI, Tests

| Workstream | Size | Justification |
|---|---:|---|
| Installer + first-run automation | L | Postgres Docker, KMK, prompts, service files, network config. |
| Minimal CLI | M | Login/token helpers, deploy/start/stop/logs, audit verify. |
| End-to-end tests | XL | Privy mock, Postgres, Safe test chain, QEMU smoke, registry pulls, bridge isolation. |
| Documentation | M | Operator guide, backup/restore, network troubleshooting, API docs. |

### Web UI Decision

v1 should ship API + CLI only unless a separate UI engineer is available. The
current UI is a Vue app embedded into the VMM binary and assumes a single global
VM list, static token auth, and no org switcher (`02-vmm-web-ui.md:13-56`,
`02-vmm-web-ui.md:698-900`). A production multi-tenant UI needs auth screens,
org switcher, member management, Safe signing prompts, registry credentials,
quota/audit screens, and bridge diagnostics. That is an L/XL product surface by
itself. Recommend v1.5 for the UI, with v1 exposing a complete API, OpenAPI
docs, and a narrow CLI.

### Fresh Total Estimate

One engineer:

- Foundation/Auth/Audit/Quota/Observability: 18-24 weeks.
- Safe/on-chain: 14-20 weeks.
- Host worker/CVM lifecycle: 14-18 weeks.
- Bridge networking: 8-12 weeks.
- External registry credentials: 5-8 weeks.
- Install/CLI/E2E/docs: 12-16 weeks.

Total: 71-98 engineer-weeks, roughly 16-23 months for one engineer after
coordination overhead and hardening. With two parallel engineers splitting
control-plane/API/on-chain from host/network/tests, realistic calendar is 9-12
months.

Delta versus old plan: old synthesis estimated about 14 months one-engineer /
7 months two-parallel before the cuts. Dropping the OCI registry saves roughly
6-10 weeks and dropping signed batches saves roughly 2-3 weeks, but the revised
per-org bridge work adds roughly 8-12 weeks and the estimate here is more honest
about E2E testing and Safe/Privy integration. The net plan is not dramatically
shorter; it is safer and better scoped around the right hard parts.

## 15. Open Implementation Questions

From `00-decisions.md:384-397` plus synthesis:

1. Exact crate versions: Axum, sqlx, alloy, libsodium wrapper, OpenAPI generator.
   The direction is Axum + sqlx + alloy; pin versions at implementation start.
2. Whether guest-agent needs changes to consume `.docker/config.json` from the
   shared path, or whether existing `.user-config`/encrypted env is sufficient.
3. Exact Privy API surface for embedded-wallet EIP-712 SafeTx signing and
   whether server-driven signing requires a webhook or browser callback.
4. Exact Safe v1.4.1 deployment addresses/code hashes on Base and testnet config.
5. Recovery signer timelock contract choice. If no audited minimal helper is
   available, keep recovery disabled in v1.
6. Postgres peer auth from Docker: exact socket ownership and auth config that
   works repeatably across Ubuntu/Debian host installs.
7. dnsmasq process model at scale: one process per bridge for v1, but define the
   threshold where a single managed process becomes necessary.
8. IPv6: v1 bridge IPAM is IPv4-only RFC1918. Decide whether to explicitly drop
   IPv6 router advertisements on org bridges or support ULA prefixes.
9. Network helper interface: sudoers fixed commands vs systemd-run transient
   units vs a long-running root daemon with a Unix socket.
10. Metrics cardinality policy for `org_id` labels on large SaaS operators.
11. Whether to add a small `notifications` table for quota warnings in API-only
   v1 or expose warnings only through quota/audit endpoints.
12. Backup automation remains out of scope by decision, but the install guide
   must still document `pg_dump` and filesystem snapshot ordering.

## 16. Risks

1. **Bridge orchestration bugs break tenant networking.** Mitigation: declarative
   reconciler, never delete enslaved bridges, E2E tests with two orgs, explicit
   user-mode fallback.
2. **Cross-org bridge leakage if any CVM reaches a shared bridge.** Mitigation:
   hide legacy global bridge for tenant deploys, resolve bridge only from
   `org_networks`, audit any platform-admin unsafe override.
3. **dnsmasq process scaling.** One dnsmasq per active bridge is simple but may
   become noisy at hundreds of orgs. Mitigation: metrics, process-count alert,
   future single-process config generator.
4. **Conntrack pressure from NAT.** Per-org NAT still shares host conntrack.
   Mitigation: monitor count/insert failures, document sysctl sizing, enforce
   per-org deploy/connection policy later.
5. **MTU/path-MTU blackholes.** Mitigation: default 1500, configurable lower MTU,
   MTU incident metric, bridge health check that validates outbound packet path.
6. **Privy lock-in/cost.** The revised decisions accept Privy despite skeptic
   concerns (`99-skeptic.md:981-987`). Mitigation: store normalized users and
   wallets locally, keep auth boundary behind a provider trait, support BYO
   external wallets from day one.
7. **KMK loss.** Losing the master key bricks encrypted secrets. Mitigation:
   first-run wall-of-text, canary decrypt check, documented re-provisioning
   procedure, never imply dstack can recover it.
8. **Safe signing UX delays membership changes.** Admin changes are two-phase and
   can get stuck pending. Mitigation: explicit pending states, repair/retry
   endpoint, clear audit events.
9. **External registry availability affects deploys.** No host cache in v1.
   Mitigation: verify creds at configuration time, fail fast with registry host
   in error, add pull-through cache in v2 if demand appears.
10. **Single-host process remains SPOF.** Existing CVMs keep running if control
    plane dies, but management stops (`00-decisions.md:316-329`). Mitigation:
    readiness/health, restart policy, DB WAL, reconciliation on startup.

## 17. Bibliography

Primary decisions:

- `00-decisions.md:8-29`: TL;DR with Privy, Safe, Postgres, external registries,
  per-org bridges.
- `00-decisions.md:30-54`: one binary, one host, host_id-ready schema,
  greenfield-only, trust boundary.
- `00-decisions.md:55-94`: Privy, RBAC roles, Platform Admin, API tokens.
- `00-decisions.md:95-126`: Safe/Base and recovery signer.
- `00-decisions.md:127-183`: Postgres persistence and secrets/KMK.
- `00-decisions.md:185-217`: audit and quotas.
- `00-decisions.md:218-241`: external registry credentials, no in-house OCI.
- `00-decisions.md:243-286`: networking, per-org uids, per-org Linux bridges.
- `00-decisions.md:288-315`: API conventions and observability.
- `00-decisions.md:331-382`: greenfield release and deferred items.
- `00-decisions.md:384-397`: implementation choices left to synthesis.
- `00-decisions.md:398-403`: skeptic/Codex sign-off.

Investigation reports:

- `01-vmm-api-surface.md`: current Rocket/pRPC API inventory and single-tenant
  API assumptions.
- `02-vmm-web-ui.md`: current embedded Vue UI structure and multi-tenant UI
  blockers.
- `03-vmm-cli.md`: current `vmm-cli.py` capabilities and why v1 does not keep
  compatibility.
- `04-vmm-lifecycle-and-state.md`: current CVM lifecycle, workdir state,
  in-memory state, and recovery behavior.
- `05-vmm-coupling-map.md`: VMM coupling to KMS, gateway, guest agent, host OS,
  QEMU, supervisor, and registry client.
- `06-vmm-auth-today-and-gaps.md`: current auth gaps, RBAC requirements, and
  cross-org hard-boundary analysis.
- `07-persistence-today-and-multitenant.md:187-210`: what moves to DB versus
  stays on disk.
- `07-persistence-today-and-multitenant.md:212-470`: initial Postgres schema
  proposal used as a base and revised here.
- `08-production-ops-gaps.md:28-160`: missing logs/metrics/tracing/health.
- `08-production-ops-gaps.md:603-705`: missing resource accounting and audit.
- `09-cvm-domain-model.md`: domain entities, desired/observed state, and
  invariants for CVMs/images/hosts/orgs.
- `10-multitenant-design-space.md:253-323`: original four-role RBAC analysis,
  adapted to Admin/Deploy/Billing/Viewer.
- `10-multitenant-design-space.md:326-459`: on-chain identity design space.
- `10-multitenant-design-space.md:533-657`: REST/JSON API recommendation.
- `10-multitenant-design-space.md:659-735`: quota enforcement model.
- `10-multitenant-design-space.md:735-804`: audit event requirements.
- `11-migration-strategy.md:476-525`: migration recommendation that the revised
  decisions explicitly decline in favor of greenfield-only.
- `12-f3-vlan-isolation.md:11-25`: today's user/bridge/custom networking and
  lack of VLAN/org fields.
- `12-f3-vlan-isolation.md:95-130`: per-org bridge recommendation.
- `12-f3-vlan-isolation.md:134-153`: concrete code touch-points for bridge
  isolation.
- `13-codex-bridge-opinion.md:3-8`: independent recommendation for per-org
  Linux bridges.
- `13-codex-bridge-opinion.md:26-56`: bridge/DHCP/IPAM/NAT/failure details.
- `13-codex-bridge-opinion.md:75-79`: bridge metrics and revisit thresholds.
- `99-skeptic.md:316-386`: accepted Deploy role push-back.
- `99-skeptic.md:388-465`: accepted external registry push-back.
- `99-skeptic.md:468-552`: accepted drop-signed-batches push-back.
- `99-skeptic.md:730-808`: token role≤creator discussion; revised decisions
  accept issue-time role limit but decline cascading revocation.
- `99-skeptic.md:810-992`: Postgres and Privy push-backs declined.

Current code citations:

- `vmm/src/main.rs:81-90`: current external Rocket mounts.
- `vmm/src/main.rs:111-129`: current host API vsock launch.
- `vmm/src/main_routes.rs:45-186`: static UI and authenticated log route.
- `vmm/rpc/proto/vmm_rpc.proto:63-110`: current VM configuration and networking
  proto shape.
- `vmm/src/config.rs:386-427`: current networking config has user/bridge/custom,
  no org/VLAN field.
- `vmm/vmm.toml:86-96`: current default networking mode is user and bridge is
  global if enabled.
- `vmm/src/app/qemu.rs:33-56`: deterministic MAC from VM ID.
- `vmm/src/app/qemu.rs:388-432`: QEMU command setup from workdir.
- `vmm/src/app/qemu.rs:504-526`: user-mode and bridge-mode netdev assembly.
- `vmm/src/app/qemu.rs:774-866`: TDX QEMU machine/object config.
- `vmm/src/app/qemu.rs:990-1118`: current workdir file layout helpers.
- `vmm/src/app.rs:151-165`: in-memory App state and CID pool.
- `vmm/src/app.rs:421-447`: current DHCP lease handling by MAC/IP only.
- `vmm/src/app.rs:560-620`: current reload/recovery of workdirs and port
  forwarding.
- `vmm/src/app.rs:964-1004`: current shared file materialisation.
- `vmm/src/host_api_service.rs:21-63`: host API remote endpoint handling and
  sealing key path.
- `vmm/src/setup-user.sh:97-181`: current per-uid localhost firewall guard.
- `vmm/src/app/registry.rs:5-20`, `vmm/src/app/registry.rs:89-151`: current
  external OCI client for global guest image pulls.
- `docs/bridge-networking.md:1-29`: bridge mode is a global bridge config today.
- `docs/bridge-networking.md:87-113`: dnsmasq DHCP notify design today.
- `docs/bridge-networking.md:118-146`: single-bridge firewall/NAT requirements.
- `docs/bridge-networking.md:156-178`: qemu-bridge-helper behavior.
- `docs/bridge-networking.md:193-198`: bridge deletion/recreation breaks running
  VM networking.
- `gateway/templates/wg.conf:5-10`: WireGuard peer `AllowedIPs = ip/32`.
- `gateway/src/main_service.rs:399-478`: gateway CVM registration and WG config.
- `gateway/src/proxy/port_policy.rs:55-89`: gateway port-policy fail-close.
- `gateway/gateway.toml:34-43`: gateway WireGuard defaults.
- `kms/auth-eth/contracts/DstackKms.sol:16-22`,
  `kms/auth-eth/contracts/DstackKms.sol:38-40`,
  `kms/auth-eth/contracts/DstackKms.sol:143-168`: DstackKms app registry and
  factory.
- `kms/auth-eth/contracts/DstackApp.sol:16-27`,
  `kms/auth-eth/contracts/DstackApp.sol:152-190`,
  `kms/auth-eth/contracts/DstackApp.sol:193-217`: DstackApp ownership and boot
  policy checks.
- `kms/auth-eth/src/ethereum.ts:34-52`: current app boot check maps appId and
  composeHash into contract calls.
