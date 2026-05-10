# 07 — Persistence model: today & multi-tenant

This report inventories every persistent artefact dstack-vmm reads or writes
today, summarises concurrency assumptions, then proposes how that surface
should be split between Postgres and the local filesystem when the control
plane becomes multi-tenant and (potentially) multi-host.

---

## Part 1 — Today

### 1.1 Top-level paths and config

dstack-vmm has **no database**. All state is on the local filesystem of the
TDX host. Three roots are involved:

| Symbol         | Default                                | Purpose                                                                                                                                           |
| -------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `image.path`   | `~/.dstack-vmm/image`                  | Image catalog (immutable per tag).                                                                                                                |
| `run_path`     | `~/.dstack-vmm/vm`                     | Per-CVM working directories ("inventory" of CVMs).                                                                                                |
| Discovery dir  | `$XDG_RUNTIME_DIR/dstack-vmm` (volatile) | Per-process registration files for the CLI to find running vmm instances.                                                                         |

Defaults live in `vmm/src/config.rs:507-541`. Production deployments override
these to e.g. `/var/lib/dstack/images` and `./run/vm` (`docs/deployment.md:140-150`,
`docs/tutorials/vmm-configuration.md:101,154-156`).

The vmm config file itself is read-only at startup: `load_config_figment`
merges `/etc/vmm/{vmm.toml,vmm.json}` and `./vmm.toml` (`vmm/src/config.rs:77-79`,
`load_config/src/lib.rs:64-72`). vmm never writes back to its own toml.

The **supervisor** is a sibling process owned by vmm (`vmm/src/main.rs:209-222`).
It holds its process map in a `DashMap` in memory only (`supervisor/src/supervisor.rs:31-44`)
— no persistence of its own. On vmm restart, supervisor is re-launched and
its in-memory list is rebuilt by inspecting QEMU pidfiles plus reading
`vm-manifest.json` files. This is why the sole source of truth for "what
CVMs exist" is the `run_path` directory tree.

### 1.2 Persistent artefacts inventory

#### Configuration (read-only by vmm)

| Path                                                | Format | Created by             | Mutated by | Read on restart                                    |
| --------------------------------------------------- | ------ | ---------------------- | ---------- | -------------------------------------------------- |
| `/etc/vmm/vmm.{toml,json}` and `./vmm.toml`         | TOML/JSON | operator (out-of-band)  | operator   | `load_config_figment` at startup (`config.rs:77-79`) |
| `/etc/dstack/client.conf` (optional fallback)       | INI    | operator               | operator   | `read_qemu_path_from_client_conf` (`config.rs:480-504`) — only to find QEMU binary |

Authentication is purely token-based and tokens live inline in vmm.toml
(`config.rs:288-293`, `vmm.toml:127-128`). There is no user/role/account
record on disk.

#### Per-VMM-process volatile registration

| Path                                          | Format | Created          | Mutated         | Read on restart                                    |
| --------------------------------------------- | ------ | ---------------- | --------------- | -------------------------------------------------- |
| `$XDG_RUNTIME_DIR/dstack-vmm/<uuid>.json`     | JSON   | startup (`discovery.rs:59-97`) | written once   | `cleanup_stale_registrations` purges entries whose pid is dead (`discovery.rs:113-144`) |

Schema = `VmmInstanceInfo { id, pid, address, working_dir, config_file, image_path, run_path, node_name, version, started_at }`
(`discovery.rs:26-48`). Unlinked on `Drop` (`discovery.rs:100-110`). Used by
`vmm-cli.py` (`vmm/src/vmm-cli.py:39-101`) to enumerate vmm instances on the
host across users.

#### Image catalog (`image.path`)

Images are flat directories named e.g. `dstack-0.5.5/`. Each contains:

| File                | Format        | Source                                              |
| ------------------- | ------------- | --------------------------------------------------- |
| `metadata.json`     | JSON (`ImageInfo`, `image.rs:12-28`) | tarball / OCI registry pull |
| `digest.txt`        | hex string    | tarball; read by `Image::load` (`image.rs:75-77`)   |
| `kernel`            | raw           | tarball                                             |
| `initrd`            | raw           | tarball                                             |
| `hda` (optional)    | qcow2         | tarball                                             |
| `rootfs` (optional) | `.iso` or `.verity` | tarball                                       |
| `bios` (optional)   | OVMF / TDVF   | tarball                                             |

Population paths:

- Operator extracts a release tarball into `image.path` (`docs/deployment.md:163`).
- Or vmm pulls from an OCI registry on demand: `pull_and_extract` writes to
  `image.path/.tmp-pull-<tag>/` then atomically renames to `image.path/dstack-<tag>/`
  (`vmm/src/app/registry.rs:93-151`). Failure cleans up the temp dir.

Mutations:

- `pull_registry_image` RPC pulls a tag (`main_service.rs:697-740`).
- `delete_image` RPC removes the directory if no VM uses it (`main_service.rs:664-695`).

Read on restart:

- `list_images` walks `image.path` lazily (`app.rs:878-888`).
- Each VM load re-opens its referenced image dir (`app.rs:185-186`).

#### Per-CVM working directory (`run_path/<vm-id>/`)

`<vm-id>` is a v4 UUID assigned at create-time (`main_service.rs:173`).
Layout (paths surface in `vmm/src/app/qemu.rs:990-1117`):

| File / dir                       | Format            | Created                                        | Mutated                                               | Read on restart                                          |
| -------------------------------- | ----------------- | ---------------------------------------------- | ----------------------------------------------------- | -------------------------------------------------------- |
| `vm-manifest.json`               | JSON (`Manifest`, `app.rs:48-73`) | `put_manifest` at create / update (`qemu.rs:1006-1011`, `main_service.rs:296-298`, `441-443`, `499-501`) | `update_vm` / `resize_vm` rewrite via `put_manifest`  | `manifest()` deserialises in `load_vm` (`app.rs:175`)    |
| `vm-state.json`                  | JSON `{started: bool}` (`qemu.rs:108-111`) | `set_started` (`qemu.rs:1024-1028`) | flipped on every start / stop / shutdown (`app.rs:236, 285`, `app.rs:932`) | `started()` (`qemu.rs:1013-1022`) decides whether to auto-restart on reload |
| `shared/`                        | dir               | `prepare_work_dir` (`app.rs:964-993`)          | populated each time guest config changes              | mounted into the CVM via 9p / vvfat / vhd (`qemu.rs:555-599`) |
| `shared/app-compose.json`        | JSON              | `prepare_work_dir`                             | `update_vm` overwrite (`main_service.rs:386`)         | hashed by `app_compose_hash` (`qemu.rs:1038-1043`); read by `app_compose()` |
| `shared/.encrypted-env`          | raw bytes         | `prepare_work_dir` if non-empty                | `update_vm` overwrite                                  | passed through to guest                                  |
| `shared/.user-config`            | raw bytes         | `prepare_work_dir` if non-empty                | `update_vm` overwrite                                  | passed through to guest                                  |
| `shared/.instance_info`          | JSON `{instance_id, app_id}` (`qemu.rs:76-83`) | `prepare_work_dir` (initial app_id) (`app.rs:983-991`); guest event `instance.info` writes the instance_id atomically via `safe_write` (`app.rs:936-940`) | guest can rewrite at runtime          | `instance_info()` deserialises (`qemu.rs:1126-1129`)     |
| `shared/.sys-config.json`        | JSON              | `sync_dynamic_config` on every start (`app.rs:996-1005`) | rewritten before every QEMU launch                    | regenerated each start, not consulted on reload by vmm  |
| `hda.img`                        | qcow2             | `create_hd` on first start (`qemu.rs:113-136, 401-403`) | grown by `qemu-img resize` on disk-size update (`main_service.rs:271-284`) | re-attached to QEMU on restart                          |
| `shared.img` (only if `host_share_mode = "vhd"`) | FAT32 raw image | rebuilt every start (`qemu.rs:579-595`)        | recreated from `shared/` on each start                | not read by vmm; recreated                              |
| `serial.log`                     | text (PTY logfile) | QEMU at start                                | appended by QEMU                                       | rotated into `serial.history.log` before next boot       |
| `serial.history.log`             | text              | `rotate_serial_log` on every restart (`app.rs:1096-1137`) | appended; truncated from front when over `serial_history_max_bytes` | streamed via `/logs` endpoint                            |
| `serial.pty`                     | PTY device path   | QEMU at start                                  | symlink/PTY managed by QEMU; deleted before relaunch (`app.rs:250-254`) | n/a (recreated)                                          |
| `stdout.log`, `stderr.log`       | text              | supervisor's stdio redirector (`supervisor/src/process.rs:345-424`) | appended; boot separator added by `append_boot_separator` (`app.rs:1082-1092`) | streamed via `/logs`                                     |
| `qemu.pid`                       | text (pid)        | supervisor (`supervisor/src/process.rs:218-225`) | written each start                                     | not consulted on vmm restart (process state comes from supervisor's in-memory map) |
| `qmp.sock`                       | unix socket       | QEMU if `cvm.qmp_socket = true`                | held by QEMU                                           | deleted before relaunch                                  |
| `guest-ip`                       | text (IPv4)       | `set_guest_ip` on DHCP lease (bridge mode only) (`qemu.rs:1068-1070`, `app.rs:439-441`) | overwritten on lease refresh                          | `guest_ip()` rebuilds port-forward rules on reload (`app.rs:606-618`) |
| `.removing`                      | empty marker file | `set_removing` when user requests deletion (`qemu.rs:1116-1118`, `app.rs:303-306`) | created once                                          | `is_removing()` makes `reload_vms` resume the cleanup task (`app.rs:564-591`) |

Helper: `safe_write::safe_write` (third-party crate `safe-write = 0.1.2`,
`Cargo.toml:110`) is used **only** for `.instance_info` updates from guest
events (`app.rs:939`). All other writes go through `fs_err::write`, which is a
plain `write+rename`-less truncating write. Crash-during-write of e.g.
`vm-manifest.json` will leave a partially written file.

#### Running-state caches kept off-disk

These are **not** persisted on disk and rebuild each restart:

- The CID pool (`AppState::cid_pool`, `app.rs:1257-1262`). On `reload_vms`,
  vmm interrogates the supervisor for running QEMU pids and re-occupies their
  CIDs (`app.rs:545-563, 624-654`). For VMs the supervisor doesn't know about,
  CIDs are re-allocated when each manifest is loaded.
- `AppState::vms` (`HashMap<String, VmState>`).
- `AppState::active_forwards` (port-forward rules per VM). Recomputed from
  `manifest.port_map` plus the persisted `guest-ip` file at reload time
  (`app.rs:605-618`).
- `App::pull_status` (registry pull progress).

#### Secrets at rest

There are essentially no vmm-owned secrets on disk:

- API tokens are configured statically in `vmm.toml` (`config.rs:288-293`) —
  the operator's tutorial keeps them in `~/.dstack/secrets/` and bakes them
  into the toml at deploy time (`docs/tutorials/vmm-configuration.md:62-71`).
- `.encrypted-env` is encrypted by the client *before* upload (X25519 ECDH +
  AES-256-GCM, see `docs/encrypted-env-spec.md`); vmm only stores the opaque
  ciphertext.
- KMS keys, sealing keys, and TLS keys are held by KMS and Gateway, not by
  vmm.
- The local `key_provider` (`config.rs:469-474`, `host_api_service.rs:49-62`)
  is a *separate* TCP service that vmm forwards quote-bound key requests to;
  vmm does not hold the sealing key itself.

### 1.3 Concurrency model

dstack-vmm assumes **exclusive single-process ownership** of `run_path`,
`image.path`, and the supervisor. Concrete evidence:

- All in-process locking is `std::sync::Mutex` over `AppState`
  (`app.rs:131-141`). No file-locking primitive (`flock`, `fcntl`,
  lockfile) anywhere in `vmm/src`.
- The supervisor is started by vmm and connected via UDS at
  `supervisor.sock` (`vmm.toml:130-136`). Its own state is in-memory only
  (`supervisor/src/supervisor.rs:31-44`).
- The CID pool is a runtime structure keyed off whatever QEMU processes the
  supervisor reports (`app.rs:545-562`). If two vmm processes share a
  `run_path`, they will both allocate from the same pool with no
  coordination, producing CID collisions. QEMU itself fails fast on a
  duplicate vsock CID (`docs/deployment.md:527-538`).
- Discovery files (`discovery.rs`) are an *advisory* registration so
  `vmm-cli.py` can discover sibling instances; they are not used as a
  cross-process lock.
- Inside a single vmm process, the only place a guest can race a host write
  is `.instance_info` (guest event vs. cold start). That is the one place
  `safe_write` is used (`app.rs:939`).

The take-away for the rewrite: today the on-disk inventory is structurally a
single-writer database. Anything that wants to share it across processes —
multi-host, blue/green vmm upgrades, or even an ops sidecar — needs new
coordination.

---

## Part 2 — Multi-tenant production

### 2.1 What is org-scoped vs genuinely shared

| Today's artefact                     | Naturally per-org                        | Genuinely shared                                        |
| ------------------------------------ | ---------------------------------------- | ------------------------------------------------------- |
| `vmm.toml`                           | —                                        | host-wide (operator-owned)                              |
| `image.path/<tag>/` (kernel, initrd, rootfs, hda backing, bios, metadata.json, digest.txt) | — | shared blobs: a global image catalog. Identity-by-content (`digest.txt`). |
| `run_path/<vm-id>/vm-manifest.json`  | yes — describes one CVM owned by one org | —                                                       |
| `run_path/<vm-id>/vm-state.json` (started flag) | yes (operational state)          | —                                                       |
| `run_path/<vm-id>/shared/app-compose.json` | yes — app code identity            | hash is referenced by KMS auth contracts (org reads)    |
| `run_path/<vm-id>/shared/.encrypted-env`, `.user-config` | yes — tenant payload      | —                                                       |
| `run_path/<vm-id>/shared/.instance_info` | yes (per-CVM, per-org)               | —                                                       |
| `run_path/<vm-id>/hda.img`           | yes — tenant disk                        | image's `hda` backing file is shared (qcow2 backing chain) |
| `run_path/<vm-id>/serial*.log`, `stdout.log`, `stderr.log` | yes — observability for one tenant's CVM | —                                                       |
| CID pool                             | host-global                              | shared resource (one bare-metal host's vsock CID space) |
| Port-mapping ranges                  | host-global                              | shared resource (one host's IPv4 ports)                 |
| GPU PCI slots                        | host-global                              | shared resource                                         |
| `auth.tokens`                        | replaced by users/api_tokens             | —                                                       |
| Supervisor process map               | host-global                              | one supervisor per host                                 |
| Discovery files                      | host-global                              | one process registry per host                           |

The pattern: **metadata, ownership, and tenant payloads should move to the
DB; image blobs and disk images stay on the host filesystem; physical
resources (CIDs, ports, GPUs) are owned by a host record in the DB and
allocated through it.**

### 2.2 Proposed Postgres schema

Notation: lowercase snake-case tables, primary keys `id` are UUIDv7 unless
noted. Timestamps are `timestamptz`. JSON-shaped columns use `jsonb`.

```sql
-- ─────────────────────────────────────────────────────────────────────
-- Identity & access
-- ─────────────────────────────────────────────────────────────────────

CREATE TABLE users (
    id              uuid PRIMARY KEY,
    email           citext NOT NULL UNIQUE,           -- nullable if SIWE-only later
    display_name    text NOT NULL,
    password_hash   text,                              -- argon2id; null for SSO/SIWE-only
    eth_address     bytea UNIQUE,                      -- optional SIWE binding (20 bytes)
    status          text NOT NULL CHECK (status IN ('active','disabled','pending')),
    email_verified_at timestamptz,
    last_login_at   timestamptz,
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE organizations (
    id              uuid PRIMARY KEY,
    slug            citext NOT NULL UNIQUE,            -- routable, stable
    name            text NOT NULL,
    created_by      uuid NOT NULL REFERENCES users(id),
    plan            text NOT NULL DEFAULT 'free',
    -- on-chain bridging (see overview Q2):
    eth_address     bytea,                              -- optional org wallet/multisig
    deleted_at      timestamptz,                        -- soft delete
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX organizations_active_idx ON organizations (id) WHERE deleted_at IS NULL;

CREATE TABLE memberships (
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            text NOT NULL CHECK (role IN ('owner','admin','member','viewer')),
    invited_by      uuid REFERENCES users(id),
    accepted_at     timestamptz NOT NULL DEFAULT now(),
    created_at      timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, user_id)
);
CREATE INDEX memberships_user_idx ON memberships (user_id);

CREATE TABLE invitations (
    id              uuid PRIMARY KEY,
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           citext NOT NULL,
    role            text NOT NULL CHECK (role IN ('admin','member','viewer')),
    invited_by      uuid NOT NULL REFERENCES users(id),
    token_hash      bytea NOT NULL,                    -- sha256 of opaque token; bearer is the secret
    status          text NOT NULL CHECK (status IN ('pending','accepted','revoked','expired')),
    expires_at      timestamptz NOT NULL,
    accepted_by     uuid REFERENCES users(id),
    accepted_at     timestamptz,
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX invitations_org_status_idx
    ON invitations (organization_id, status)
    WHERE status = 'pending';
CREATE UNIQUE INDEX invitations_pending_email_idx
    ON invitations (organization_id, lower(email))
    WHERE status = 'pending';

CREATE TABLE api_tokens (
    id              uuid PRIMARY KEY,
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    -- exactly one of (user_id) is null=service-account, set=user-scoped:
    user_id         uuid REFERENCES users(id) ON DELETE CASCADE,
    name            text NOT NULL,
    -- Token format: prefix is shown to users for identification (e.g. "dstk_v1_<random>"),
    -- token_hash is sha256 of the full secret. Never store the secret.
    token_prefix    text NOT NULL,
    token_hash      bytea NOT NULL,
    scopes          text[] NOT NULL DEFAULT '{}',     -- e.g. {'cvm:read','cvm:write','image:pull'}
    last_used_at    timestamptz,
    expires_at      timestamptz,
    revoked_at      timestamptz,
    created_by      uuid NOT NULL REFERENCES users(id),
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX api_tokens_hash_idx ON api_tokens (token_hash);
CREATE INDEX api_tokens_org_idx ON api_tokens (organization_id) WHERE revoked_at IS NULL;

-- ─────────────────────────────────────────────────────────────────────
-- Resources
-- ─────────────────────────────────────────────────────────────────────

CREATE TABLE hosts (                                  -- one row per dstack-vmm node
    id              uuid PRIMARY KEY,
    name            text NOT NULL UNIQUE,             -- operator-supplied; matches `node_name`
    region          text,
    address         text NOT NULL,                    -- internal RPC URL the control plane talks to
    status          text NOT NULL CHECK (status IN ('online','draining','offline','retired')),
    capabilities    jsonb NOT NULL DEFAULT '{}',      -- gpu types, max vcpu, max mem, networking modes
    last_heartbeat_at timestamptz,
    -- physical resource bookkeeping mirrors today's vmm.toml limits:
    cid_start       int NOT NULL,
    cid_pool_size   int NOT NULL,
    max_vcpu        int NOT NULL,
    max_memory_mb   int NOT NULL,
    created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE images (                                 -- global catalog, content-addressed
    id              uuid PRIMARY KEY,
    name            text NOT NULL,                    -- "dstack-0.5.5"
    version         text NOT NULL,
    digest          bytea NOT NULL,                   -- from digest.txt; uniqueness key
    is_dev          boolean NOT NULL DEFAULT false,
    metadata        jsonb NOT NULL,                   -- contents of metadata.json
    -- optional registry source for re-pull:
    registry_ref    text,                             -- "dstacktee/guest-image:0.5.5"
    public          boolean NOT NULL DEFAULT true,
    uploaded_by     uuid REFERENCES users(id),        -- null for system catalog
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX images_digest_idx ON images (digest);
CREATE UNIQUE INDEX images_name_idx ON images (name);

CREATE TABLE host_images (                            -- which hosts have which images on disk
    host_id         uuid NOT NULL REFERENCES hosts(id) ON DELETE CASCADE,
    image_id        uuid NOT NULL REFERENCES images(id) ON DELETE CASCADE,
    path            text NOT NULL,                    -- absolute path on host
    size_bytes      bigint NOT NULL,
    pulled_at       timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (host_id, image_id)
);

CREATE TABLE cvms (
    id              uuid PRIMARY KEY,                  -- replaces today's manifest.id
    organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT,
    created_by      uuid NOT NULL REFERENCES users(id),
    host_id         uuid REFERENCES hosts(id),         -- null = unscheduled / pending
    image_id        uuid NOT NULL REFERENCES images(id),
    name            text NOT NULL,
    app_id          text NOT NULL,                     -- on-chain app identity
    -- declared resources:
    vcpu            int NOT NULL,
    memory_mb       int NOT NULL,
    disk_size_gb    int NOT NULL,
    gpus            jsonb NOT NULL DEFAULT '{}',       -- attach_mode + slots; matches Manifest.gpus
    networking      jsonb NOT NULL DEFAULT '{}',       -- per-VM networking overrides
    port_map        jsonb NOT NULL DEFAULT '[]',
    kms_urls        text[] NOT NULL DEFAULT '{}',
    gateway_urls    text[] NOT NULL DEFAULT '{}',
    no_tee          boolean NOT NULL DEFAULT false,
    hugepages       boolean NOT NULL DEFAULT false,
    pin_numa        boolean NOT NULL DEFAULT false,
    -- runtime state:
    desired_state   text NOT NULL CHECK (desired_state IN ('running','stopped','removed')),
    observed_state  text NOT NULL CHECK (observed_state IN ('pending','starting','running','stopping','stopped','exited','removing','error')),
    cid             int,                              -- assigned by host once scheduled
    instance_id     text,                             -- reported by guest at boot
    boot_progress   text,
    boot_error      text,
    last_seen_at    timestamptz,
    -- persistent operational fields previously inferred from filesystem:
    workdir         text,                             -- absolute path on host_id; mirrors run_path/<id>
    guest_ip        inet,
    started_at      timestamptz,
    stopped_at      timestamptz,
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX cvms_org_idx        ON cvms (organization_id);
CREATE INDEX cvms_host_idx       ON cvms (host_id) WHERE host_id IS NOT NULL;
CREATE INDEX cvms_observed_state ON cvms (observed_state);
CREATE UNIQUE INDEX cvms_host_cid_idx ON cvms (host_id, cid) WHERE cid IS NOT NULL;
CREATE UNIQUE INDEX cvms_org_name_idx ON cvms (organization_id, lower(name)) WHERE desired_state <> 'removed';

CREATE TABLE cvm_artifacts (                          -- mutable per-CVM blobs (compose, env)
    cvm_id          uuid NOT NULL REFERENCES cvms(id) ON DELETE CASCADE,
    kind            text NOT NULL CHECK (kind IN ('app_compose','encrypted_env','user_config','instance_info')),
    -- For audit/version history we keep every revision; latest = max(version):
    version         int NOT NULL,
    sha256          bytea NOT NULL,
    body            bytea,                            -- nullable: see "DB vs disk" below
    body_uri        text,                             -- s3://… or file:// when body is offloaded
    created_by      uuid REFERENCES users(id),        -- null when written by guest
    created_at      timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (cvm_id, kind, version)
);

-- ─────────────────────────────────────────────────────────────────────
-- Quotas, audit, observability
-- ─────────────────────────────────────────────────────────────────────

CREATE TABLE quotas (
    organization_id uuid PRIMARY KEY REFERENCES organizations(id) ON DELETE CASCADE,
    max_cvms        int NOT NULL DEFAULT 5,
    max_vcpu        int NOT NULL DEFAULT 16,
    max_memory_mb   int NOT NULL DEFAULT 32768,
    max_disk_gb     int NOT NULL DEFAULT 200,
    max_gpus        int NOT NULL DEFAULT 0,
    -- soft consumption columns updated transactionally with cvm changes:
    used_cvms       int NOT NULL DEFAULT 0,
    used_vcpu       int NOT NULL DEFAULT 0,
    used_memory_mb  int NOT NULL DEFAULT 0,
    used_disk_gb    int NOT NULL DEFAULT 0,
    used_gpus       int NOT NULL DEFAULT 0,
    updated_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              bigserial PRIMARY KEY,
    organization_id uuid REFERENCES organizations(id) ON DELETE SET NULL,
    actor_user_id   uuid REFERENCES users(id) ON DELETE SET NULL,
    actor_token_id  uuid REFERENCES api_tokens(id) ON DELETE SET NULL,
    actor_ip        inet,
    action          text NOT NULL,                    -- "cvm.create", "image.delete", "membership.add", …
    target_kind     text,                             -- "cvm", "image", "user", …
    target_id       text,                             -- string for portability
    request_id      uuid,
    payload         jsonb NOT NULL DEFAULT '{}',      -- inputs/outputs sans secrets
    result          text NOT NULL CHECK (result IN ('success','failure')),
    error           text,
    created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX audit_log_org_time_idx ON audit_log (organization_id, created_at DESC);
CREATE INDEX audit_log_actor_idx    ON audit_log (actor_user_id, created_at DESC);
CREATE INDEX audit_log_target_idx   ON audit_log (target_kind, target_id);

-- Optional: scheduling work queue (see §2.5 multi-host)
CREATE TABLE host_tasks (
    id              uuid PRIMARY KEY,
    cvm_id          uuid REFERENCES cvms(id) ON DELETE CASCADE,
    host_id         uuid NOT NULL REFERENCES hosts(id) ON DELETE CASCADE,
    kind            text NOT NULL,                    -- 'create','start','stop','remove','update','pull_image'
    payload         jsonb NOT NULL DEFAULT '{}',
    status          text NOT NULL CHECK (status IN ('pending','claimed','succeeded','failed','cancelled')),
    claimed_by      text,                             -- vmm process id / lease holder
    claim_expires_at timestamptz,
    attempts        int NOT NULL DEFAULT 0,
    last_error      text,
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX host_tasks_dispatch_idx
    ON host_tasks (host_id, status, created_at)
    WHERE status IN ('pending','claimed');
```

Notes / open choices left for the synthesis pass:

- `users.email` can be made nullable later for SIWE-only accounts.
- `cvm_artifacts.body` either keeps small JSON (`app_compose`, `instance_info`)
  inline and offloads `encrypted_env` / `user_config` to object storage. The
  `sha256` column is what KMS / on-chain auth references — it is the
  authoritative artefact identity, so the bytes themselves can move
  without renumbering keys.
- `cvms.workdir` is convenient but redundant with `host_id + id`; keep it
  while we still have host-side filesystem layouts.

### 2.3 What stays on disk vs moves to the database

**Stays on the TDX host filesystem.**

- The image catalog (`image.path/`). Kernels, initrds, qcow2 backing files,
  rootfs ISO/verity images. These are large, content-addressed, and
  shared between many CVMs via qcow2 backing chains — they can't live in
  the DB. The `images` row carries the catalog *metadata* and the
  `host_images` row records which host has each one materialised.
- `run_path/<vm-id>/hda.img` — per-CVM qcow2 disk. Stays local to its
  host.
- `run_path/<vm-id>/shared/` while a CVM is starting / running. The 9p /
  vvfat / vhd backend wants files on disk for QEMU to mount; vmm
  re-materialises them from the DB before each start.
- `serial.log`, `serial.history.log`, `stdout.log`, `stderr.log` — high-rate
  log files. Streamed via `/logs` (see report 01) and optionally shipped to
  a log store. Not stored row-by-row in Postgres.
- `qmp.sock`, `serial.pty` — process-local sockets/PTYs.
- `qemu.pid`, supervisor state — process supervision is host-local.
- `$XDG_RUNTIME_DIR/dstack-vmm/*.json` — replaced by `hosts` table heartbeat
  for cross-host discovery; per-process registration can stay or be
  dropped.

**Moves to the database.**

- `auth.tokens` array → `api_tokens`.
- The `vm-manifest.json` payload → `cvms` columns (+ `cvms.gpus`,
  `cvms.networking`, `cvms.port_map` jsonb).
- `vm-state.json` (`{started}`) → `cvms.desired_state`.
- Operational status: started/exited/booting/booterror/instance_id/guest_ip
  → `cvms.observed_state`, `cvms.boot_progress`, `cvms.boot_error`,
  `cvms.instance_id`, `cvms.guest_ip`. Updated via host-side reconciliation,
  not by reading files.
- `app-compose.json`, `.encrypted-env`, `.user-config`, `.instance_info` →
  `cvm_artifacts` rows (with body either inline or offloaded). The host
  re-materialises these into `run_path/<vm-id>/shared/` before each start.
- `digest.txt`, `metadata.json` of each image → `images.digest`,
  `images.metadata`.
- `.removing` marker → `cvms.desired_state = 'removed'` plus a `host_tasks`
  row.
- Per-host capacity envelope (`cvm.cid_start`, `cid_pool_size`,
  `max_allocable_vcpu`, `max_allocable_memory_in_mb`) → `hosts` columns.
  CID allocation becomes a transactional `INSERT … RETURNING` against
  `cvms.cid` with `UNIQUE (host_id, cid)`.

### 2.4 Migration of today's on-disk inventory

Migration runs once per host as the multi-tenant control plane is rolled out.
It is read-mostly on the host:

1. **Bootstrap an org and an owner.** Create one `organizations` row per
   existing legacy install (default: a single "default" org owned by the
   operator's user). All migrated CVMs and images get attached to that
   org. The operator can later split / move CVMs between orgs.
2. **Register the host.** Insert one `hosts` row keyed on the legacy
   `node_name` (or hostname). Capture `cid_start`, `cid_pool_size`, and
   the resource caps from the existing `vmm.toml` — these are the
   per-host envelope.
3. **Import the image catalog.** `for dir in image.path/*: load
   metadata.json + digest.txt; insert images row keyed on digest;
   insert host_images row pointing at the on-disk path`.
4. **Import each CVM.** `for dir in run_path/*:`
   - Read `vm-manifest.json` → populate the `cvms` row (id = manifest.id,
     keep the UUID; organization_id = default; created_by = owner;
     host_id = host row from step 2; image_id = lookup by image
     `manifest.image` name).
   - Read `vm-state.json` → set `desired_state`.
   - Read `shared/app-compose.json`, `shared/.encrypted-env`,
     `shared/.user-config`, `shared/.instance_info` → insert
     `cvm_artifacts` v1 rows (sha256 computed locally; body inline
     or offloaded based on size).
   - Read `guest-ip` if present → `cvms.guest_ip`.
   - Set `cvms.cid` = the supervisor-reported CID for that VM, or NULL if
     the VM isn't running.
   - If `.removing` exists, `desired_state = 'removed'` and enqueue a
     `host_tasks` row with `kind = 'remove'`.
5. **Reconcile.** Start the rewritten vmm-host agent against the DB.
   It should see every existing `cvms` row owned by its `host_id` and
   immediately call `reload_vms` semantics: refresh `observed_state`,
   pin CIDs, restore port-forward rules from `cvms.port_map +
   cvms.guest_ip`. No QEMU restarts are required.
6. **Import quotas.** `INSERT INTO quotas SELECT default values FROM
   organizations`. Operator can tune from there.
7. **Audit retroactively (optional).** A best-effort pass writes one
   `audit_log` row per imported CVM with `action = 'cvm.import'`.

Steps 1–5 are idempotent; running migration twice is a no-op if rows are
upserted on `(id)` for `cvms`, `(digest)` for `images`, `(host_id,
image_id)` for `host_images`, and `(cvm_id, kind, version)` for
`cvm_artifacts`.

### 2.5 Multi-host implications

Today vmm assumes single-process exclusive ownership of the host
filesystem (§1.3). The rewrite splits the system in two:

- **Control plane (stateless API + Postgres).** Owns users, orgs,
  CVMs, images, audit, quotas. The DB is the single source of truth.
- **Host agents (rewritten dstack-vmm, one per TDX host).** Stateless
  workers that own QEMU lifecycle, `run_path`, the local image
  catalog, and the supervisor.

Multi-host coordination lives in the DB. Two reasonable patterns:

**(a) Pull model — host agents claim work.**

`host_tasks` is a queue. A host agent polls / `LISTEN`s on its
`host_id` partition and `UPDATE … RETURNING` to atomically claim a
task with a `claim_expires_at` lease. State changes (CVM created,
started, stopped, removed) are written back as `cvms.observed_state`
plus an audit row. Heartbeats touch `hosts.last_heartbeat_at`. When a
host disappears, its claimed-but-stale tasks are released by a
janitor.

**(b) Push model — control plane RPCs to host agents.**

The host agent exposes a small auth'd RPC (close to today's prpc) and
the control plane calls it directly when a CVM transitions states.
Persistence is still in the DB; the DB keeps `cvms.observed_state` in
sync via heartbeats / status callbacks.

Either way, scheduling is centralised in the control plane:

- **CVM creation.** API allocates an `id`, picks a `host_id` (constraint
  solver: image present, free vcpu/memory/disk/cid/GPU slots, host
  online, networking mode supported), inserts the `cvms` row, enqueues
  a `host_tasks` row.
- **CID allocation.** Becomes a single SQL: `INSERT INTO cvms (…, cid)
  SELECT … FROM hosts h LEFT JOIN cvms c USING (host_id) WHERE …
  RETURNING cid;` enforced by the `UNIQUE (host_id, cid)` partial
  index. No more in-memory `IdPool`.
- **Image distribution.** `host_images` is the index of "what is on
  which host". A schedule that lands on a host without the image
  emits a `pull_image` task first.
- **Failover.** A host going `offline` is a soft failure: its CVMs stay
  pinned (`cvms.host_id` set, `desired_state = 'running'`,
  `observed_state = 'unknown'`). Today there is no live migration of
  CVMs — encrypted disks are bound to host hardware via the local key
  provider, so a re-host requires the user to redeploy. The DB makes
  this visible at least.
- **Multiple control-plane instances.** They share Postgres; Postgres
  advisory locks or `SELECT … FOR UPDATE SKIP LOCKED` keep schedulers
  from racing on the same task.
- **Discovery files** become an internal debugging convenience and stop
  being load-bearing — `hosts` is the source of truth.

The biggest invariant the rewrite gains: a single transaction can mutate
ownership (`cvms.organization_id`), state (`desired_state`), quota
counters (`quotas.used_*`), and audit (`audit_log`). Today these are
spread across `vm-manifest.json`, `vm-state.json`, none, and "nowhere",
respectively, with no atomicity. That's the fundamental capability gap
the multi-tenant rewrite needs to close.
