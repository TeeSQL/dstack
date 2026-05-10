# CVM Domain Model — Conceptual Entities, Lifecycle, Invariants

**Prepared:** 2026-05-05
**Scope:** A clean, conceptual domain model distilled from `dstack-vmm` to anchor the multi-tenant control-plane rewrite.
**Stance:** Conceptual, not a Rust struct dump. Where useful, the report points at `file:line` in the existing code.

---

## How to read this document

This is the domain layer the new control-plane API will be designed against. The current `dstack-vmm` mixes three unrelated things in one process: (1) a **control plane** (what should exist, who owns it, what config it has), (2) a **scheduler / placement engine** (which host runs it, what CID/MAC/port it gets), and (3) a **runtime** (a QEMU process, a vsock CID, a tap interface, a port-forward NAT rule). For the rewrite, we want to split those layers cleanly. This document defines the control-plane domain — the *control* layer and what it must persist about the *runtime* layer — but explicitly does **not** model the scheduler/runtime concepts as first-class user entities.

Section 5 lists which existing concepts are runtime/scheduler artefacts and should *not* leak into the new domain model.

---

## 1. Core entities

Each entity is described in terms of the conceptual fields the new domain needs, what role each field plays, and where in the current code the concept maps.

### 1.1 CVM

A **CVM** (confidential virtual machine) is the central tenant-owned entity — the dstack analogue of a "droplet" or "instance". Conceptually:

| Field | Role | Notes |
|---|---|---|
| `id` | Stable, opaque identifier (UUID). Surface name for the CVM. | Today: `Manifest.id` (`uuid::Uuid::new_v4()`), set at create time. `vmm/src/main_service.rs:173`, `vmm/src/app.rs:48–73`. |
| `name` | Human label. | Validated to a restricted character set. `vmm/src/main_service.rs:58–69`. |
| `owner_org` | The organization that owns this CVM. | **Does not exist today.** All CVMs are global to the host. Multi-tenant rewrite requires this field on every entity. |
| `created_by_user` | The user whose action produced this CVM (audit, not auth). | Does not exist today. |
| `created_at` | Wall-clock create time. | Today: `Manifest.created_at_ms`. |
| `compose_ref` | Pointer to the **AppCompose** revision currently in effect. | Today: implicit; the on-disk `app-compose.json` file in the workdir is the single source of truth. The compose hash is recomputed on demand from that file. |
| `image_ref` | The **Image** (guest OS) the CVM is bound to. | Today: `Manifest.image` (string image-folder name). `vmm/src/app.rs:56`. |
| `app_id` | The on-chain application identity. Derived from compose hash on first creation, or supplied for an upgrade-of-existing-app flow. | Today: `Manifest.app_id`, computed as `truncate40(sha256(compose_file))` if not provided. `vmm/src/main_service.rs:46–55, 169–172`. |
| `instance_id` | A second on-chain identity scoped to a single boot/run, written by the guest after first contact with KMS. | Today: read from `INSTANCE_INFO` (`.instance_info`) inside the workdir, populated by the guest. `vmm/src/app/qemu.rs:76–82, 1126–1130`. |
| `resources.vcpu` | Requested vCPUs. | `Manifest.vcpu`. |
| `resources.memory_mb` | Requested RAM in MB. | `Manifest.memory`. |
| `resources.disk_gb` | Requested disk in GB. | `Manifest.disk_size`. |
| `resources.gpus` | GPU attach intent (mode + slot list). | `Manifest.gpus`. **Today this is host-coupled** (PCI slots are bare-metal addresses). For multi-tenant scheduling the user should ask for *count + class*, not specific slots. |
| `resources.hugepages` | Whether QEMU should use hugepages. | `Manifest.hugepages`. |
| `resources.pin_numa` | Whether QEMU should be NUMA-pinned. | `Manifest.pin_numa`. Both are technically host-runtime concerns, but they affect tenant-visible performance, so they belong on the tenant config. |
| `network.mode` | `bridge` / `user` / `custom`. | `Manifest.networking`. Custom mode is operator-only. |
| `network.port_bindings` | List of `(host_addr, host_port) → vm_port` per protocol. | `Manifest.port_map`. **Today these are host ports**. In the multi-tenant model they should be expressed as ingress intent — see §1.6 Gateway Binding. |
| `attestation.no_tee` | Allow falling back to a non-TEE VM (debug only). | `Manifest.no_tee`. |
| `kms_endpoints` | Override KMS URLs. | `Manifest.kms_urls`. |
| `gateway_endpoints` | Override gateway URLs. | `Manifest.gateway_urls`. |
| `desired_state` | What the user has asked for (`running` / `stopped` / `deleted`). | Today: persisted as the `started: bool` flag in `vm-state.json`. `vmm/src/app/qemu.rs:108–111, 1013–1028`. |
| `status` | The system's last-known actual lifecycle state (see §2). | Today: derived live from a join over the supervisor process state, the `started` flag, and the in-memory `removing` flag. `vmm/src/app/qemu.rs:312–331`. |
| `runtime` | Opaque, scheduler-owned runtime placement (which host, which CID, which guest IP, which tap). | Should not leak into the tenant-facing API. Today these are inlined into `VmConfig` (CID) and `VmStateMut` (guest IP, devices). `vmm/src/app.rs:1212–1228`. |
| `events` | Append-only event log (boot progress, errors, lifecycle). | Today: in-memory bounded `VecDeque<GuestEvent>`. `vmm/src/app.rs:912–946`. |

**Encryption-key handling.** The CVM never holds a private key in the control plane; it only holds:
- A reference to the KMS endpoint (or KMS contract) it should fetch keys from at boot.
- An optional `encrypted_env` blob (already encrypted client-side using the app's env-pubkey from KMS — see `docs/encrypted-env-spec.md`). Today this lives at the path `shared/.encrypted-env` inside the VM workdir. `vmm/src/app.rs:952–953, 974–977`.
- An `app_id` and `instance_id` that bind it to the right key derivations once the guest contacts the KMS.

The control plane MUST NOT custody KMS-derived keys (disk-crypt-key, env-crypt-key, k256-key). Those live only inside the CVM after boot.

---

### 1.2 Image

An **Image** is the guest OS image (kernel + initrd + rootfs ± BIOS). Conceptually:

| Field | Role | Source |
|---|---|---|
| `name` | Operator-facing image name (e.g. `dstack-0.5.8`). | Image folder name. |
| `version` | Semver-style version string. | `ImageInfo.version`. `vmm/src/app/image.rs:25`. |
| `is_dev` | Whether this is a dev-mode image (less strict checks). | `ImageInfo.is_dev`. |
| `digest` | SHA256-style content digest of the image. | `digest.txt` next to `metadata.json`. `vmm/src/app/image.rs:75–77`. |
| `os_image_hash` | The TDX-relevant "OS image hash" submitted to KMS / on-chain whitelist. | Decoded from `digest`. `vmm/src/app.rs:1170–1175`. |
| `mr_fingerprints` | Pre-computed MR values (MRTD, RTMR0–3, mrAggregated, mrSystem) for verification. | **Not currently a first-class field on Image.** Today these are computed externally via `dstack-mr` and lifted into the on-chain whitelist independently. The new domain should track them on Image. |
| `kernel_path`, `initrd_path`, `rootfs_path`, `bios_path`, `cmdline` | Boot artefacts. | These are runtime concerns and only the *control-plane* needs to know they exist (so it can validate referenced images exist on the host). |
| `shared_ro` | Whether the host-shared dir is mounted read-only inside the guest. | `ImageInfo.shared_ro`. Affects guest behaviour and the boot-progress feature gate. |

**Conceptual ownership.** Images are operator-scoped (or platform-scoped), not org-scoped. Tenants pick from a whitelist provided by the operator. The on-chain `DstackKms.allowedOsImages` mapping is the security boundary; the operator's image catalogue is the UX layer.

---

### 1.3 Compose / AppCompose

The **AppCompose** is the docker-compose-derived spec that defines what runs *inside* the CVM. Conceptually:

| Field | Role | Source |
|---|---|---|
| `manifest_version` | AppCompose schema version. | `AppCompose.manifest_version`. `dstack-types/src/lib.rs:14`. |
| `name` | App name (independent of CVM name). | `AppCompose.name`. |
| `runner` | How the guest runs the app (`docker-compose` is the only meaningful value today). | `AppCompose.runner`. |
| `docker_compose_file` | The actual compose YAML (or a reference to it). | `AppCompose.docker_compose_file`. |
| `key_provider` | Which KMS-style key provider the app expects (`none` / `kms` / `local` / `tpm`). | `AppCompose.key_provider`. `dstack-types/src/lib.rs:93–100`. |
| `key_provider_id` | Provider-specific identifier (e.g. KMS root pubkey, local-MR hash). | `AppCompose.key_provider_id`. |
| `gateway_enabled` | Whether the app expects to be exposed via dstack-gateway. | `AppCompose.gateway_enabled`. |
| `allowed_envs` | Whitelist of env-var names the app is allowed to receive. | `AppCompose.allowed_envs`. |
| `public_logs` / `public_sysinfo` / `public_tcbinfo` | Whether logs/sysinfo/TCB info are exposed publicly via guest agent. | `AppCompose.*`. |
| `port_policy` | Per-port gateway behaviour (PROXY-protocol opt-in, restrict mode). | `AppCompose.port_policy`. `dstack-types/src/lib.rs:55–72`. |
| `compose_hash` | SHA256 of the canonical (deterministic) JSON serialization of the AppCompose. **The on-chain authority.** | Today: computed at request time and again from disk on demand. `vmm/src/main_service.rs:27–32, 565–572`; `vmm/src/app/qemu.rs:1038–1043`. |
| `env_encryption_pubkey` | The X25519 pubkey clients use to encrypt env vars. **Derived from KMS, not the AppCompose itself.** | Fetched from KMS via `GetAppEnvEncryptPubKey(app_id)`. `vmm/src/main_service.rs:455–467`. Conceptually the env-pubkey hangs off the *App registration* (§1.4), not the AppCompose document — but tenants always think of it as "the public key for this compose", so the API should expose it on the CVM-create flow. |

**Multiple-revision case.** A single CVM can be upgraded to a new AppCompose (`UpdateVm` with a new `compose_file`). The new `compose_hash` must already be on the on-chain whitelist before the upgrade is accepted. The historical succession of compose revisions for a CVM is currently *not* tracked — only the latest is on disk. The new domain should keep an audit trail of compose revisions per CVM.

---

### 1.4 KeyProvider / on-chain App registration

This is the bridge between dstack-vmm and dstack-kms. Conceptually it is **not** a property of the VMM at all; the VMM is just a client that points at it. But the control-plane API must surface it because users need to manage it.

A **KmsApp** (on-chain App registration) has:

| Field | Role | Source |
|---|---|---|
| `app_id` | Ethereum address that uniquely identifies this app on-chain. | The `address` of the deployed `DstackApp` proxy. `kms/auth-eth/contracts/DstackApp.sol`. |
| `kms_contract` | Address of the parent `DstackKms` registry. | `DstackKms` (`kms/auth-eth/contracts/DstackKms.sol`). |
| `allowed_compose_hashes` | Set of compose hashes this app is authorized to boot with. | `DstackApp.allowedComposeHashes` mapping. |
| `allow_any_device` | If true, any KMS-allowlisted host can boot this app; otherwise restricted to `allowed_device_ids`. | `DstackApp.allowAnyDevice`. |
| `allowed_device_ids` | Set of device IDs (host fingerprints) allowed to boot this app. | `DstackApp.allowedDeviceIds`. |
| `require_tcb_up_to_date` | Reject boots when TCB is stale. | `DstackApp.requireTcbUpToDate`. |
| `upgrades_disabled` | If true, the contract is frozen — no more compose-hash additions. | `DstackApp._upgradesDisabled`. |
| `owner_eoa` | EOA that controls compose-hash whitelist + device whitelist + upgrade. | `DstackApp` is `OwnableUpgradeable`. |

A **KmsRegistry** (the parent `DstackKms`) has:

| Field | Role |
|---|---|
| `address` | Parent contract address. |
| `chain_id` | Which chain it lives on (Phala / Sepolia / mainnet / …). |
| `allowed_os_image_hashes` | OS image whitelist. |
| `allowed_kms_aggregated_mrs`, `allowed_kms_device_ids` | KMS bootstrap whitelists. |
| `gateway_app_id` | The dstack-gateway's own on-chain app ID (so it can fetch its keys). |

**Why this matters for the rewrite.** The bridge is currently *implicit*: `Manifest.app_id` is just a hex string, and the only enforcement is "KMS will refuse to provision keys if the compose hash isn't whitelisted on this app". The new domain should:

1. Make the on-chain `KmsApp` a **first-class control-plane entity** (one row in the control-plane DB, with `address`, `chain`, `kms_registry`, `owner_eoa`).
2. Tie `CVM.app_id → KmsApp.address` as a foreign-key relationship.
3. Surface compose-hash management as control-plane operations (the CP either calls the contract on the user's behalf or generates the right unsigned tx for the user to sign).
4. Decide who custodies `owner_eoa` — see open question #2 in `00-overview.md`.

---

### 1.5 Host / Node

A **Host** is a bare-metal TDX machine running a `dstack-vmm` process. Today the VMM does not have a first-class concept of a host beyond "the machine I am running on", but the multi-tenant rewrite plan opens the door to multi-host. Conceptually:

| Field | Role | Source |
|---|---|---|
| `id` | Stable identifier (UUID, or a deterministic ID derived from device-id). | Today the `discovery::VmmInstanceInfo.id` random UUID is the closest analogue. `vmm/src/discovery.rs:26–48`. |
| `name` | Operator label. | `Config.node_name`. `vmm/src/config.rs:333–334`. |
| `address` | Where the CP can reach this host's VMM agent. | `discovery::VmmInstanceInfo.address`. |
| `version` | VMM software version. | `discovery::VmmInstanceInfo.version`. |
| `device_id` | TDX/platform device fingerprint, used for on-chain device-allowlist checks. | Computed by the KMS bootstrap, not by the VMM. `kms/rpc/proto/kms_rpc.proto:139`. |
| `tcb_status` | "UpToDate" / etc. | Reported by the verifier from the latest quote. |
| `capacity` | Total CPU, RAM, GPU inventory. | Today: derived from `lspci` (GPUs) + system limits; no central tracking. |
| `available` | Headroom after subtracting all assigned CVMs. | Today: implicit; the `CvmConfig.max_allocable_*` fields exist but are not enforced. `vmm/src/config.rs:163–165`, `vmm/src/main_service.rs:548–552`. |
| `images_available` | Which Image revisions this host has on disk and can boot. | Today: `App.list_images()`. `vmm/src/app.rs:878–888`. |
| `gpu_inventory` | Per-GPU slot, product ID, in-use flag. | Today: `App.list_gpus()`. `vmm/src/app.rs:1030–1048`. |
| `attached_kms_endpoints`, `attached_gateway_endpoints` | Defaults the host advertises to CVMs. | `CvmConfig.kms_urls`, `CvmConfig.gateway_urls`. |
| `status` | `online` / `draining` / `offline`. | New in multi-host design. |

**Single-host vs multi-host shape.** The current vmm's `discovery` module is a hint of where this is heading: each VMM writes a JSON pointer to `$XDG_RUNTIME_DIR/dstack-vmm/<id>.json` so CLI tools can find it (`vmm/src/discovery.rs`). For multi-host, the control plane is the directory of hosts; the per-host VMM agent is a thin remote runner. CVMs gain a **placement** (which Host[s] they are scheduled on / replicated to).

---

### 1.6 Gateway Binding

A **Gateway Binding** is an ingress-mapping entry tying a CVM to dstack-gateway. Conceptually:

| Field | Role | Source |
|---|---|---|
| `id` | Surrogate ID for the binding. | New. |
| `cvm_id` | Owning CVM. | New (today implicit). |
| `gateway_app_id` | The on-chain ID dstack-gateway uses to fetch its own keys. | `kms/rpc/proto/kms_rpc.proto:46`. Read-only at the CVM level — set by the gateway operator. |
| `instance_id` | The CVM's instance-id (registered with the gateway as a WireGuard peer). | Set when the CVM calls `Gateway.RegisterCvm`. `gateway/rpc/proto/gateway_rpc.proto:12–60`. |
| `wg_pubkey` | The CVM's WireGuard public key. | Sent in `RegisterCvmRequest.client_public_key`. |
| `wg_ip` | The IP address the gateway assigned to this CVM in the WireGuard mesh. | `WireGuardConfig.client_ip`. |
| `port_policy` | Per-port (PROXY-protocol opt-in, restrict-mode) policy for this binding. | `RegisterCvmRequest.port_policy`. Originates from `AppCompose.port_policy`, can be admin-overridden at the gateway. |
| `app_url` | The public URL the gateway exposes for this CVM (e.g. `https://<id>-<port>.<base_domain>`). | Computed by the VMM from gateway config. `vmm/src/app/qemu.rs:276–303`. |

**Lifecycle.** A gateway binding is created when the CVM first calls `Gateway.RegisterCvm` (initiated by the guest, not the VMM). It is invalidated when the CVM is deleted. Today the binding's existence is *implied* by `gateway_enabled = true` on the AppCompose; there is no first-class ID. Multi-tenant rewrite should make bindings explicit so users can see "this CVM is reachable at https://x.gateway.example.com on these ports".

**Per-host port forwarding (bridge mode) is NOT a Gateway Binding.** When a CVM is in `bridge` networking mode the host runs userspace port-forwards as well (`ForwardRule` in `vmm/src/app.rs:454–528`). That is a *runtime artefact* of the placement decision; a Gateway Binding is an external-facing ingress mapping. The two should not be conflated in the new model.

---

### 1.7 Org / User / Membership / ApiToken

These are the control-plane identity entities. **None of these exist today** — `dstack-vmm` has only a flat list of static API tokens (`vmm/src/config.rs:288–293`, `vmm/src/main.rs:208`). The new model needs:

| Entity | Fields | Notes |
|---|---|---|
| **Org** | `id`, `name`, `slug`, `created_at`, `plan/quota` (out of scope for v1), `default_kms_app_owner_eoa?` | An org is the top-level tenancy boundary. Every CVM, every API token, every audit event is scoped to exactly one Org. |
| **User** | `id`, `email`, `display_name`, `auth_method` (password / OAuth / SIWE), `auth_credentials` (provider-specific), `created_at`, `last_login` | A user is an authenticatable principal. Users can be members of multiple orgs. |
| **Membership** | `(org_id, user_id, role)` where role ∈ `{owner, admin, developer, viewer}` | Join entity. Roles drive RBAC checks. |
| **ApiToken** | `id`, `org_id`, `name`, `created_by_user`, `token_hash`, `scopes`, `last_used_at`, `expires_at?`, `revoked_at?` | Tokens are org-scoped service-account credentials. The token is hashed at rest; only the prefix is shown in the UI. |
| **AuditEvent** | `(org_id, actor_user_id, actor_token_id, action, target_kind, target_id, payload, occurred_at)` | Every state-changing operation produces one. Required for "who deleted my CVM at 3am". |
| **Invite** | `(org_id, invited_email, role, invited_by, token, expires_at, accepted_at?)` | Owner-initiated invites for new members. |

**Out of scope for the domain model itself but worth flagging:**
- **Quotas**: `(org_id, resource_kind, limit)` — vCPU, RAM, GPU, CVM count. Enforced by the CP at create-time. The current `CvmConfig.max_allocable_*` fields are host-wide, not org-scoped.
- **Wallet bindings**: if SIWE is supported, a User may have one or more wallet addresses; an Org may have a designated EOA (multisig) for on-chain ownership of `KmsApp`s.

---

## 2. CVM lifecycle states and transitions

The current implementation has a small set of **statuses** computed live from three signals: the user's `started` flag, the supervisor process's `running?` flag, and the in-memory `removing` flag. See `vmm/src/app/qemu.rs:317–331`:

```
status = if removing      → "removing"
         else (started, is_running) match {
           (true,  true)  → "running"
           (true,  false) → "exited"
           (false, true)  → "stopping"
           (false, false) → "stopped"
         }
```

This is conceptually thin and conflates several concerns. The new domain model should distinguish **desired_state** (what the user asked for) from **status** (the system's last-known actual state) and add intermediate states for in-flight operations. Below is the proposed lifecycle.

### 2.1 States (proposed)

| State | Meaning |
|---|---|
| `requested` | The CVM record has been created in the CP DB, but no host has been chosen yet. (Multi-host only; single-host can skip this.) |
| `scheduled` | A target host has been picked. Resources reserved. Workdir not yet materialized. |
| `provisioning` | Workdir being created on host (compose, encrypted env, sys-config written). Image being verified/staged. |
| `starting` | QEMU process spawned. TDX boot in progress. Boot-progress events streaming. |
| `running` | Guest has reached steady state (boot-progress event "running" or supervisor reports running and started flag true). |
| `degraded` | Running but with a problem the system has detected (e.g. KMS unreachable, gateway unreachable, container restart-looping). New state, not present today. |
| `stopping` | User-initiated shutdown in flight. |
| `stopped` | Process exited at user request, workdir preserved. Can be restarted. |
| `exited` | Process exited unexpectedly (the user did not ask for it). Equivalent to today's `(started=true, running=false)`. Eligible for auto-restart. |
| `failed` | Provisioning or first-boot failed; record retained for diagnosis. |
| `deleting` | User asked to delete; cleanup in progress (port forwards being torn down, supervisor process being stopped, workdir being removed, CID being freed). Today: the `removing: true` flag + `.removing` marker file. `vmm/src/app.rs:291–392`. |
| `deleted` | Tombstoned. Workdir removed, runtime resources freed. Kept as a tombstone in the CP DB for audit. |

### 2.2 Transitions

**Operator-initiated** (i.e. caused by a CP user action):

| From | → To | Trigger |
|---|---|---|
| (none) | `requested` | `CreateVm` |
| `requested` | `scheduled` | scheduler picks a host |
| `scheduled` | `provisioning` | CP instructs target host to materialize workdir |
| `provisioning` | `starting` | workdir ready, QEMU spawned |
| `provisioning` / `starting` | `failed` | error during provisioning or first boot |
| `running` / `degraded` | `stopping` | `StopVm` or `ShutdownVm` |
| `stopping` | `stopped` | guest shutdown observed |
| `stopped` / `exited` | `starting` | `StartVm` |
| `stopped` / `exited` / `running` / `degraded` / `failed` | `deleting` | `RemoveVm` |
| `deleting` | `deleted` | cleanup finishes |
| `stopped` (or `running` if hot-update is allowed) | `provisioning` | `UpdateVm` with a new compose / new image / new resources |
| `running` / `stopped` | `provisioning` | `ResizeVm` (must currently be stopped first; `vmm/src/main_service.rs:251–254`) |

**System-initiated** (no user action):

| From | → To | Trigger |
|---|---|---|
| `running` | `exited` | QEMU process disappears unexpectedly. Detected on the next supervisor poll. `vmm/src/app/qemu.rs:325–328`. |
| `exited` | `starting` | Auto-restart loop, when enabled. `vmm/src/main.rs:132–146`, `vmm/src/app.rs:1050–1078`. |
| `running` | `degraded` | Health probe sees KMS unreachable / gateway disconnected / container failing. (New — no equivalent today.) |
| `degraded` | `running` | health probe recovers. |
| any | `failed` | host disappears, workdir is corrupted, image is invalid. |

**Crash-recovery transitions.** When the VMM process restarts, it does NOT have an in-memory state — it reconstructs each CVM by reading workdirs from disk and joining against the supervisor's notion of running processes. The reconstruction logic is in `vmm/src/app.rs:545–622` (`reload_vms`). This is itself a state-restore transition that the new model should formalize: `(unknown) → starting | running | exited | deleting | …` based on persisted markers (`vm-manifest.json`, `vm-state.json`, `.removing`, supervisor process registration).

### 2.3 Desired state vs. observed state

The new model should keep these as separate fields, both on the CVM record:

- `desired_state ∈ {running, stopped, deleted}` — what the user asked for. Persisted, only mutated by user-initiated transitions.
- `status ∈ {…full enum above…}` — the system's last reconciled view.

The reconciliation loop's job is to drive `status → desired_state`. Today's `started` flag is a sloppy version of `desired_state`; today's "status" string is computed live with no persistence. Splitting them lets the CP answer "the user wants this running but it isn't — what's wrong?" without ambiguity.

---

## 3. Invariants

Invariants the new domain model must enforce (or surface as alerts when violated):

### 3.1 Process-state coherence

> **A CVM in `running` state has exactly one live QEMU process owned by the placement host's supervisor; no other state has a live process.**

Today this is *not* enforced — it's checked best-effort by `merged_info` joining the supervisor list with the in-memory map (`vmm/src/app/qemu.rs:317–331`), and orphaned supervisor processes are GC'd at reload time (`vmm/src/app.rs:594–603`). The new model should make this an enforceable invariant via the reconciler.

### 3.2 Compose-hash authority

> **A CVM in any active state (`requested` … `running`) has a `compose_hash` that, if `key_provider = kms`, is on the on-chain whitelist of its `app_id`.**

Today the VMM does not check this — the KMS does, at key-provisioning time. This means a CVM can be created and even started with a not-yet-whitelisted compose, and only fail when the guest tries to fetch keys. The new CP should pre-validate against the on-chain state at create-time and refuse with a clear error.

### 3.3 Image–on-chain consistency

> **A CVM's `image.os_image_hash` is on the parent `DstackKms.allowedOsImages` whitelist.**

Same enforcement gap as 3.2 today. Same fix in the new CP.

### 3.4 Compose-derived gateway state

> **`CVM.gateway_binding` exists IFF `CVM.compose.gateway_enabled` is true.**

Today the VMM derives `gateway_enabled` once at load (`vmm/src/app.rs:204`, `vmm/src/app/qemu.rs:105`) and uses it to decide whether to compute an `app_url`. The actual binding is created by the *guest* calling the gateway; the VMM is not in the loop. The new CP should track gateway bindings explicitly and ensure they're created/torn down with the CVM.

### 3.5 Deletion is total

> **When a CVM transitions to `deleted`, all associated runtime artefacts are freed:**
>
> - QEMU process is gone (`stop` then `remove` from supervisor)
> - Workdir is removed
> - CID is returned to the pool
> - Port-forward NAT rules are removed
> - GPU slots are released
> - Gateway binding (if any) is removed at the gateway

Today's deletion logic in `vmm/src/app.rs:291–392` covers the first four; gateway-binding cleanup is *not currently performed by the VMM* — the gateway garbage-collects bindings when handshakes age out. The new CP should drive explicit binding deletion to avoid stale entries.

### 3.6 Idempotent removal

> **Asking to delete a CVM that is already `deleting` or `deleted` is a no-op, not an error.**

Today's `remove_vm` is idempotent against in-progress removal (`vmm/src/app.rs:296–300`). Preserve this.

### 3.7 No cross-org access

> **All read and write operations on CVMs / Images / Compose / KmsApp / Gateway-Binding entities are scoped to the calling principal's org membership.**

Does not exist today (no orgs). Every API handler must check this in the new design.

### 3.8 Resource-allocation invariants

> **The sum of vCPU / RAM / GPU committed to non-deleted CVMs on a Host does not exceed the Host's capacity.**

Today not enforced (`max_allocable_*` is exposed but not checked). New CP should enforce at scheduling time.

> **No two non-deleted CVMs on the same Host hold the same CID, the same MAC address, or the same GPU slot.**

Today: CID uniqueness via `IdPool` (`vmm/src/app/id_pool.rs`); MAC uniqueness via deterministic SHA256(vm_id) derivation (`vmm/src/app/qemu.rs:40–56`); GPU-slot uniqueness implicit (the user lists slots, no de-dup check). New CP should enforce all three explicitly at scheduling time.

### 3.9 Resize-while-stopped

> **`vcpu`, `memory`, `disk_size`, `image` can be modified only while the CVM is in `stopped` or `exited`.**

Already enforced today (`vmm/src/main_service.rs:251–254`). Disk size cannot shrink. Preserve.

### 3.10 Audit completeness

> **Every state-changing API call produces exactly one AuditEvent with the actor, target, action, and outcome.**

Does not exist today. Required in the rewrite.

---

## 4. Relationships (textual ER diagram)

```
Org 1 ─── * User       (via Membership)
Org 1 ─── * ApiToken
Org 1 ─── * CVM
Org 1 ─── * KmsApp     (orgs control the on-chain registrations they own)
Org 1 ─── * AuditEvent

User * ─── * Org       (via Membership; a user can be in multiple orgs)
User 1 ─── * AuditEvent (as actor)
User 1 ─── * ApiToken  (as creator)
ApiToken 1 ── * AuditEvent (as actor)

CVM * ─── 1 Image       (exactly one image at a time, can be changed via Update)
CVM 1 ─── 1 AppCompose  ("current revision"; logically also 1 ─ * if we keep history)
CVM * ─── 1 KmsApp      (via app_id; multiple CVMs can share an app_id, e.g. blue/green)
CVM 1 ─── 0..1 GatewayBinding   (depends on compose.gateway_enabled)
CVM 1 ─── 1 Host        (placement; multi-host only — single-host is implicit)
CVM 1 ─── * AuditEvent  (as target)

Image * ─── * Host      (an image may be staged on multiple hosts)
Image 1 ─── 0..* CVM    (multiple CVMs can share an image)

KmsApp 1 ─── 1 KmsRegistry          (parent contract)
KmsApp 1 ─── * ComposeHash          (allowed compose hashes)
KmsApp 1 ─── * DeviceId             (allowed devices)
KmsApp 1 ─── 1 OwnerEoa             (on-chain owner)

KmsRegistry 1 ─── * Image            (allowed_os_image_hashes ↔ Image.os_image_hash)
KmsRegistry 1 ─── * KmsApp           (registered apps)

GatewayBinding 1 ── 1 CVM
GatewayBinding * ── 1 Gateway        (the gateway cluster the binding is on; multi-gateway only)

Host 1 ─── * CVM        (placements)
Host 1 ─── * Image      (locally-staged images)
Host 1 ─── * GpuDevice
Host 1 ─── 1 DeviceId   (the on-chain device fingerprint for this host)
```

**Cardinality cheatsheet:**
- An Org owns many CVMs; a CVM has exactly one owning Org.
- A User can belong to many Orgs; an Org has many Users.
- A CVM has one Image at a time, one current AppCompose revision, one KmsApp registration, at most one GatewayBinding, exactly one Host placement.
- A KmsApp can back many CVMs (e.g. you have 3 replicas of the same app — same app_id, three CVMs).
- A KmsApp belongs to one KmsRegistry (parent contract); a KmsRegistry holds many KmsApps.

---

## 5. Implementation artefacts that should NOT leak into the new domain model

The following concepts are present in the current vmm but are **placement / runtime / file-format concerns**, not domain concepts. The new control-plane API should hide them, expose only their effects, or model them as opaque per-host implementation details.

### 5.1 CID allocation and the CID pool

- **Where**: `vmm/src/app/id_pool.rs`, `App.cid_pool`, `Manifest.image`-side use throughout `vmm/src/app.rs` and `vmm/src/app/qemu.rs:552`.
- **Why it leaked**: vsock requires a per-VM CID; the VMM allocates from a configured pool (`CvmConfig.cid_start`, `cid_pool_size`, `vmm/src/config.rs:158–160`).
- **Why it shouldn't leak**: CIDs are a host-local Linux/QEMU concept. The control plane has no business showing "CID 1234" to a tenant. The number and identity of guest-CIDs is a scheduler-level decision.
- **What to expose instead**: nothing. If something is reachable from the host it should go through the `Host → Vmm-agent → CVM` path; if it is reachable from outside the host it goes through the gateway.

### 5.2 Port-mapping pool / host port allocation

- **Where**: `Manifest.port_map` (host-port → vm-port), `CvmConfig.port_mapping` allowed-port-ranges (`vmm/src/config.rs:122–142`), `ForwardService` for bridge-mode userspace forwarding (`vmm/src/app.rs:454–543`).
- **Why it leaked**: in user-mode networking QEMU does NAT internally, in bridge mode the VMM runs userspace forwards.
- **Why it shouldn't leak**: a tenant should ask for "expose port 8080 of my container to the internet" or "to the org-private network". *Where* the host listens for that traffic is a placement decision the scheduler/gateway should make.
- **What to expose instead**: an **ingress intent** field on the CVM (or on the AppCompose) of the form `{vm_port, protocol, visibility ∈ {public, private}}`. The CP/gateway translates that into either a Gateway Binding or a host port-forward.

### 5.3 QEMU PID file, QMP socket, serial PTY, stdout/stderr files

- **Where**: `VmWorkDir::pid_file()`, `qmp_socket()`, `serial_pty()`, `stdout_file()`, `stderr_file()`, `serial_file()`, `serial_history_file()` (`vmm/src/app/qemu.rs:1072–1107`).
- **Why it leaked**: the VMM started one QEMU process per CVM and tracked it via standard process artefacts.
- **Why it shouldn't leak**: these are runtime telemetry sinks. The CP should expose **logs as a stream** and **events as records**, not file paths.
- **What to expose instead**: a `Logs` API (today already partially exposed via `/logs` HTTP endpoint, `vmm/src/main_routes.rs:107–183`) and an `Events` API. Keep the underlying files as a host-internal implementation detail.

### 5.4 Workdir files (`vm-manifest.json`, `vm-state.json`, `app-compose.json`, `.encrypted-env`, `.user-config`, `.sys-config.json`, `.instance_info`, `.removing`, `hda.img`, `shared.img`, `guest-ip`)

- **Where**: `vmm/src/app/qemu.rs:983–1137`, `dstack-types/src/shared_filenames.rs`.
- **Why it leaked**: the VMM uses the on-disk workdir as its source of truth — there is no separate database.
- **Why it shouldn't leak**: filesystem layout is an implementation detail of the VMM. The CP DB should be the source of truth for everything except the volatile contents passed into the guest (compose JSON, encrypted env). Even those should live in the CP DB and be projected onto the host at provisioning time.
- **What to expose instead**: nothing on the API surface. The `.removing` marker is a particularly fragile mechanism (filesystem-backed flag that says "if the process crashes, resume cleanup"); it should be replaced by a proper state machine in the CP DB plus reconciliation.

### 5.5 Supervisor processes and `ProcessAnnotation`

- **Where**: `SupervisorClient`, `supervisor_client::ProcessConfig`, `ProcessAnnotation` in `vmm/src/config.rs:361–375`. Used to track each QEMU as a supervised child process; `kind = "cvm"` distinguishes CVMs from other supervised processes.
- **Why it leaked**: `dstack-supervisor` is a generic process supervisor, repurposed as the VM runtime. The VMM dispatches to it.
- **Why it shouldn't leak**: this is a per-host runner abstraction. The CP shouldn't know whether a host runs QEMUs via supervisor, systemd, k8s pods, or anything else.
- **What to expose instead**: the per-host VMM agent's API (`StartCvm`, `StopCvm`, `LogsCvm`) is the contract — what's behind it is the host operator's implementation choice.

### 5.6 MAC address derivation and DHCP-lease bookkeeping

- **Where**: `mac_address_for_vm()` in `vmm/src/app/qemu.rs:40–56`; DHCP-lease handling in `vmm/src/app.rs:423–447`.
- **Why it leaked**: bridge-mode VMs need a stable host-routable IP, obtained by DHCP, and the host needs to discover that IP to set up port forwards.
- **Why it shouldn't leak**: this is a tap/bridge-mode-network plumbing detail. The CP gives the user "my CVM is reachable here"; whether that's via tap+DHCP or vlan+SR-IOV or something else is the host's concern.

### 5.7 GPU PCI slot strings

- **Where**: `GpuSpec.slot` (`vmm/src/app.rs:117–120`), GPU listing at `vmm/src/app.rs:1030–1048`.
- **Why it leaked**: VFIO passthrough requires the precise PCI BDF of the device to bind.
- **Why it shouldn't leak**: a tenant-facing API of the form `gpus.add({slot: "0000:01:00.0"})` is wrong on a multi-host control plane; they don't even know which host they'll land on.
- **What to expose instead**: a request like `gpus = [{class: "h100", count: 4, mode: dedicated}]`. The scheduler translates that into per-host slot assignments.

### 5.8 NUMA node discovery and CPU pinning

- **Where**: `find_numa()`, `find_numa_node()` in `vmm/src/app/qemu.rs:929–964`; `Manifest.pin_numa`.
- **Why it leaked**: GPU-heavy CVMs benefit from being NUMA-pinned to the GPU's host node, so the VMM reads `/sys/...` paths on the host.
- **Why it shouldn't leak**: NUMA topology is a host concern. A tenant API can have a "performance hint" boolean (`prefer_numa_locality = true`), but topology details belong in the host agent.

### 5.9 SMBIOS spoofing for cloud-detection

- **Where**: `ProductConfig` in `vmm/src/config.rs:217–246`; applied at `vmm/src/app/qemu.rs:868–910`.
- **Why it leaked**: some workloads need the VM to look like AWS/GCP/etc. for licensing/detection purposes.
- **Why it shouldn't leak**: this is operator policy on a host, not a tenant choice.

### 5.10 Per-VMM static API tokens in `vmm.toml`

- **Where**: `AuthConfig` in `vmm/src/config.rs:288–293`; wired up in `vmm/src/main.rs:208`.
- **Why it leaked**: the simplest possible auth model: a list of equally-privileged shared-secret tokens.
- **Why it shouldn't leak**: this is the multi-tenancy hole. New model uses Org-scoped, named, audit-trailable, revocable, scoped ApiToken records (§1.7).

### 5.11 The `started` boolean flag

- **Where**: `vm-state.json` per workdir (`vmm/src/app/qemu.rs:108–111, 1013–1028`).
- **Why it leaked**: persisted desired-state for crash recovery.
- **Why it should be reframed**: this flag conflates "user wants it running" with "we successfully reached running once". Replace with explicit `desired_state` and a tracked `status` (§2.3).

### 5.12 Image-folder filesystem layout

- **Where**: `Image::load()` in `vmm/src/app/image.rs:67–92`. An "image" is a directory of `metadata.json + kernel + initrd + rootfs + bios + digest.txt`.
- **Why it leaked**: dstack guest images ship as folders.
- **Why it shouldn't leak**: an Image record in the CP has fields for digest, MR fingerprints, version, etc. *Where* the kernel.bin lives on a host is a host-side concern.

### 5.13 The Auto-Restart loop

- **Where**: `auto_restart_task` in `vmm/src/main.rs:132–146`, `try_restart_exited_vms` in `vmm/src/app.rs:1050–1078`.
- **Why it leaked**: the VMM polls every N seconds and restarts any CVM whose `started` flag is true but whose process is gone.
- **Why it should be reframed**: the *behaviour* (restart on crash) is desirable, but it should be a property of the reconciliation loop, governed by a per-CVM `restart_policy ∈ {never, on_failure, always}`. Today it's a global on/off switch (`AutoRestartConfig`).

### 5.14 Discovery files (`$XDG_RUNTIME_DIR/dstack-vmm/<id>.json`)

- **Where**: `vmm/src/discovery.rs`.
- **Why it leaked**: lets CLI tools find all VMM instances on a host.
- **Why it shouldn't leak**: in a CP-driven world, the CP knows which Hosts are online — it's the directory. Local discovery files become an internal-only debug aid (or go away entirely).

---

## 6. Cross-references for the synthesis pass

When the synthesis pass picks up this report, the most important hand-offs to other reports are:

- **API surface (01)**: Section 5 names which RPCs / API fields are leaking implementation. The new API design should reproject those into domain terms.
- **Coupling (05)**: Section 1.4 (KmsApp) and 1.6 (Gateway Binding) define the boundary between VMM, KMS, and Gateway. Decoupling work should be measured against those entity definitions.
- **Auth (06)**: Section 1.7 lists the missing identity entities. Fill in concrete schemas there.
- **Persistence (07)**: This report's §1 entities are roughly the table list; §2 the state column; §3 the integrity constraints / DB-level checks.
- **Operations (08)**: The reconciler that drives status → desired_state (§2.3) is the central operations primitive; SLO targets attach to states (e.g. P95 `provisioning → running` < N seconds).
- **Migration (11)**: Existing CVMs need `owner_org` retro-assigned, status normalized from current `(started, running, removing)` triple, and `app_id` resolved against on-chain records.

The two open questions in `00-overview.md` that this report most directly bears on:

- **#1 multi-host**: The `Host` entity (§1.5) is real if and only if multi-host is in scope. If it isn't, drop §1.5 and bake its fields into operator-level config.
- **#2 on-chain identity**: The `KmsApp.owner_eoa` field (§1.4) is the pivot. Three options map onto three different `Org → KmsApp` cardinalities and three different custody stories; all three are expressible in the model above, but each picks a different default for `Org.default_kms_app_owner_eoa`.
