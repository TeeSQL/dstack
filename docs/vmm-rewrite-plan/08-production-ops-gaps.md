# dstack-vmm — Production-Operations Gap Analysis

**Prepared:** 2026-05-05
**Scope:** Topic-by-topic review of dstack-vmm operational primitives that must
exist before a multi-tenant rewrite can ship. Each section names what the code
does today, then names the gap to a production-grade multi-tenant deployment.
**Read-only audit.** No code changes.

---

## TL;DR

Today's vmm is a single-host, single-operator daemon. It boots from a TOML
file, writes per-VM state to a directory, shells out to a sidecar `supervisor`
process to run QEMU, and trusts every API caller equally. There is **no
metrics endpoint**, **no health probe**, **no readiness gate**, **no
multi-host coordination**, **no rate limiting**, **no audit trail**, and
**no resource accounting beyond a CID counter and an unused `max_allocable_*`
config knob**. Backup of CVM disks is "rsync the workdir." Upgrades are
"restart the binary and hope `reload_vms` works."

For a multi-tenant deployment shaped like a typical cloud host, every one of
these primitives has to be designed and built. Concrete priority list at the
end of the doc.

---

## 1. Observability

### 1a. Logs

**Logging library.** `tracing` + `tracing-subscriber::fmt`, plain-text
unstructured. ANSI off. Configured at startup once.

**Source:** `vmm/src/main.rs:150-154`
```rust
use tracing_subscriber::{fmt, EnvFilter};
let filter = EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info"));
fmt().with_env_filter(filter).with_ansi(false).init();
```

**Level config.** Comes from the `RUST_LOG` env var only. The
`log_level = "debug"` field in `vmm.toml:10` is **inert** — it's a Rocket
config key, never read by `tracing_subscriber`. There is no runtime
log-level reload.

**Format.** Default `tracing_subscriber::fmt` text. Not JSON. Not
key/value. There is no log shipper, no syslog, no structured-target
config. CLAUDE.md project rule says log messages should not start with
a capital letter (`.cursorrules`); the codebase is roughly compliant.

**Per-CVM logs** (separate concern — these are guest output, not vmm output):
- Serial console: `<workdir>/serial.log`, captured directly by QEMU via the
  `-chardev pty,...,logfile=...` flag (`vmm/src/app/qemu.rs:420-424`). On VM
  restart the previous `serial.log` is rotated into `serial.history.log`,
  truncated to `serial_history_max_bytes` (default 4 MB) from the front
  on a rough line boundary (`vmm/src/app.rs:1096-1137`).
- QEMU stdout/stderr: `<workdir>/stdout.log`, `<workdir>/stderr.log`,
  appended to (with a `===== boot @ ts =====` separator on each restart at
  `vmm/src/app.rs:1082-1092`). The `supervisor` `redirect()` task watches the
  parent dir with `notify` and reopens the file on rotate/remove
  (`supervisor/src/process.rs:363-424`) — so external `logrotate` works,
  but no log shipping or retention is built in.
- A tail HTTP endpoint exists at `GET /logs?id=...&follow=...&lines=...&ch=serial|stdout|stderr`
  (`vmm/src/main_routes.rs:107-184`); it streams via `tailf` with a 60 s
  heartbeat. This is the only "remote log access" surface vmm provides.

**Request/access logs.** None beyond Rocket's own debug-level access logging.
There is no per-request structured log emitted by vmm itself for create_vm /
start_vm / etc. Only `resize_vm` is wrapped in `#[tracing::instrument]`
(`vmm/src/main_service.rs:484`).

**Multi-tenant gap.**
- No JSON output → can't ship to ELK/Loki/Datadog without parsing text.
- No request log → no per-org "what did this user do" record (see §9 audit).
- No log retention policy on per-VM serial/stdout — disk grows unbounded
  (only `serial.history.log` has a 4 MB cap; `stdout.log` / `stderr.log`
  are append-forever).
- No correlation IDs across vmm → KMS → guest-agent.
- No log-level reload (need to restart the daemon to flip from info → debug,
  which is operationally painful).

### 1b. Metrics

**Vmm has no metrics endpoint.**

- No `/metrics` route. Confirmed via `grep` over `vmm/src/main_routes.rs`,
  `vmm/src/main.rs`, and the entire `vmm/` tree.
- No `prometheus`, `metrics`, `opentelemetry-*`, `tracing-opentelemetry`,
  or similar dependency in `vmm/Cargo.toml:12-62`.
- No counters, gauges, histograms, or atomics that get scraped. (There is a
  pair of `AtomicUsize` counters at `vmm/src/main_routes.rs:76-105` for log
  stream lifecycle — but those are only emitted into the `info!`/`debug!`
  log stream, never exposed.)

**KMS reference.** dstack-kms recently shipped a `/metrics` endpoint
(`kms/src/main.rs:80-83`, `kms/src/main_service.rs:66-95`,
`kms/src/config.rs:49-56`). It exposes only two counters:
`dstack_kms_attestation_requests_total`, `dstack_kms_attestation_failures_total`,
both as `AtomicU64` rendered as Prometheus text. Gateway and verifier have
`/health` (`gateway/src/web_routes.rs:17-29`, `verifier/src/main.rs:70-71`).
**vmm has neither metrics nor health.**

**Multi-tenant gap (everything below is missing).** The bare minimum for an
ops dashboard:
- `dstack_vmm_cvm_total{state}` — counts by state (running, stopped,
  exited, removing, error).
- `dstack_vmm_cvm_create_total{outcome}` — create RPC counter.
- `dstack_vmm_cvm_start_total{outcome}`, `..._stop_total`, `..._remove_total`,
  `..._update_total`.
- `dstack_vmm_kms_request_seconds` — histogram of KMS roundtrips
  (`get_app_env_encrypt_pub_key` is the only direct call today).
- `dstack_vmm_supervisor_request_seconds`, `..._supervisor_errors_total`.
- `dstack_vmm_image_pull_*` — for the OCI registry path
  (`vmm/src/app/registry.rs`).
- `dstack_vmm_cid_pool{state="allocated|free"}`.
- `dstack_vmm_disk_bytes{role="image|run|workdir",path=...}` — from
  `nix::sys::statvfs` or similar.
- `dstack_vmm_qemu_processes_total{status}` — from supervisor list.
- `dstack_vmm_port_forward_rules` — for bridge-mode forwards
  (`vmm/src/app.rs:454-528`).
- Build/version info gauge labeled with `version`, `git_rev`.
- HTTP RPC histogram labeled by method + outcome.

A multi-tenant version of the same series should additionally be labelled
`org_id` (or `tenant_id`) at minimum. That label doesn't exist today
because vmm doesn't know about tenants.

### 1c. Tracing

No distributed tracing anywhere. `tracing` is used as a structured-logging
facade only — no `tracing-opentelemetry`, no W3C TraceContext propagation,
no span export. The single `#[tracing::instrument]` annotation in vmm is
on `resize_vm` (`vmm/src/main_service.rs:484`); spans currently land only
in stderr text logs.

**Multi-tenant gap.** Need OTLP-export of tracing spans, ideally
sampled, ideally labeled with org/CVM identity, propagated to KMS,
gateway, and guest-agent. Today there is no place to plug that in
without writing it from scratch.

---

## 2. Health / readiness probes

**No health endpoint exists in vmm.**

- The external API mounts only the routes in `main_routes::routes()`
  (`vmm/src/main_routes.rs:185-187`): `index`, `v1`, `beta`, `v0`, `res`,
  `vm_logs`. None of these answers a load-balancer health check; they
  either render the web UI or stream logs.
- The pRPC service (`/prpc/...`) does have a `Vmm.Version` method
  (`vmm/src/main_service.rs:514-519`, `vmm/rpc/proto/vmm_rpc.proto`) which
  returns build info. That is the closest thing to a probe today, and it
  requires the auth token if `auth.enabled = true`.
- The `supervisor` sidecar has a `/ping → "pong"` route
  (`supervisor/src/web_api.rs:70-73`) on its own UDS, not the vmm's
  external API.

**Comparison.** dstack-gateway has `/health` (`gateway/src/web_routes.rs:17-19`)
and dstack-verifier has `/health` (`verifier/src/main.rs:70-71`). Vmm should
have one and does not.

**What a probe would need to check (today, none of these is gated):**
- vmm process is up (trivial).
- supervisor UDS is reachable (`SupervisorClient::probe` exists at
  `supervisor/client/src/lib.rs:148-155`, but is not used after startup).
- `run_path` and `image.path` filesystems are writable / not full.
- KMS reachable (vmm needs it for `get_app_env_encrypt_pub_key` only —
  could be a dependency for readiness).
- Discovery file under `$XDG_RUNTIME_DIR/dstack-vmm/<uuid>.json` is still
  present (`vmm/src/discovery.rs:50-110`).

**Multi-tenant gap.** Need:
- `GET /healthz` — liveness, returns 200 if the rocket dispatcher is up.
- `GET /readyz` — readiness, returns 200 only after `reload_vms()` finishes
  (which can take a long time for many CVMs — see §5) and supervisor probe
  succeeds. Today there is *no* signal that "reload finished" — the rocket
  server starts listening before the `tokio::select!` block at
  `vmm/src/main.rs:227-234` fires, so create-vm can race against an
  in-progress reload.
- `GET /startup` (k8s-style startup probe) for slow reload paths.

---

## 3. Multi-host support

**Vmm is a single-host silo.**

- No peer-to-peer code, no cluster membership, no Raft/Paxos/gossip,
  no shared coordinator. `grep -i 'cluster|gossip|raft|consensus|leader|coordinator|peer'`
  in `vmm/src/` yields zero hits other than dependency boilerplate.
- The "discovery" mechanism is **local only** — vmm writes a JSON file to
  `$XDG_RUNTIME_DIR/dstack-vmm/<uuid>.json` so that a CLI on the **same
  host** can list multiple vmm processes (`vmm/src/discovery.rs:16-24,
  50-97`). Stale files are pruned by checking `/proc/<pid>` exists
  (`vmm/src/discovery.rs:113-144`). There is no cross-host equivalent.
- All shared state (CID pool, VM map, port-forward rules) lives in a single
  in-process `Arc<Mutex<AppState>>` (`vmm/src/app.rs:128-166, 1257-1284`).
  Two vmm instances on different hosts cannot read or update the same VM
  state — they don't even know about each other.
- The CID pool is process-local and configured per-host
  (`vmm/vmm.toml:30-31`: `cid_start = 1000`, `cid_pool_size = 1000`). Two
  hosts can both hand out CID 1042 to different VMs — vsock CIDs are only
  unique per host kernel anyway, but if a future control plane wants to
  refer to a CVM by a globally-unique id it must layer something on top.
- There is no cross-host orchestrator. The "scheduler" is whoever calls
  `Vmm.CreateVm` — they pick a host implicitly by which vmm's TCP/UDS
  endpoint they hit. A single vmm cannot offload a CVM onto another host;
  there is no live-migration code.

**Implication.** Two vmms on two hosts today cooperate exactly as much as
two unrelated unix daemons cooperate: zero. Each is its own scheduler, its
own state authority, its own log target. KMS and gateway are the only
fan-in points (and they don't track which host an instance lives on,
either).

**Multi-tenant gap (this is the largest architectural gap).** A multi-host
control plane needs at minimum:
- A **scheduler** that picks a host given an org's CVM request, accounting
  for vCPU/RAM/disk/GPU caps per host and per-org quotas.
- A **shared state store** for the authoritative "this CVM exists on host
  H" fact — likely a Postgres/etcd, with vmm as a worker that reconciles
  the local fs against the desired state.
- A **node registration / heartbeat** loop so the control plane knows
  which hosts are alive and what their free capacity is.
- Per-VM **id namespacing**: today the VM id is a fresh UUID
  (`vmm/src/main_service.rs:173`), unique already, but there's no concept
  of "which org owns this UUID."
- A **placement policy** for GPU/NUMA/hugepage-affine workloads.
- A way to **drain a host** — none exists today.

---

## 4. Backup / restore

**Vmm has no built-in backup or restore.**

### 4a. vmm process state (in-memory)

The `AppState` (`vmm/src/app.rs:1257-1284`) — CID pool, VM map,
port-forward rules — is **not persisted**. It is rebuilt from the
filesystem on every restart by `reload_vms()` (`vmm/src/app.rs:545-621`).
This is fine for a single-host daemon but means there is no point-in-time
backup of "what vmm thought was happening" — the disk is the only source
of truth.

### 4b. Per-VM workdir

Each CVM has a workdir at `<run_path>/<vm_id>/` whose contents define the
VM completely:
- `vm-manifest.json` — name, vcpu, memory, image, port_map, app_id, etc.
  (`vmm/src/app/qemu.rs:990-1011`)
- `vm-state.json` — `{ "started": bool }` only (`vmm/src/app/qemu.rs:994-1028`).
- `shared/app-compose.json`, `shared/encrypted-env`, `shared/user-config`,
  `shared/.instance_info`, `shared/sys-config.json` — the docker-compose
  bundle and dstack-side metadata (`vmm/src/app/qemu.rs:1030-1056`,
  `vmm/src/app.rs:948-994`).
- `hda.img` — the qcow2 system disk, optionally backed by an
  immutable image hda (`vmm/src/app/qemu.rs:113-136, 388-405`).
- `serial.log`, `serial.history.log`, `stdout.log`, `stderr.log`, `qemu.pid`,
  `serial.pty`, `qmp.sock`, `guest-ip`, `.removing` marker.

**Backup story = "tar the workdir."** No documented backup procedure, no
hot-backup hook, no qcow2 snapshot integration, no encrypted-at-rest
guarantee for hda.img (it sits on whatever filesystem `run_path` is on).
There is no built-in QMP `blockdev-snapshot-internal-sync` plumbing
(`qmp_socket = false` by default, `vmm/vmm.toml:35`), even though QEMU is
launched with `-qmp` when enabled (`vmm/src/app/qemu.rs:426-431`).

**Restore story = "untar the workdir before vmm starts, then restart vmm."**
`reload_vms()` will pick it up, allocate a CID, and auto-start it if
`vm-state.json.started == true` (`vmm/src/app.rs:545-621`,
`vmm/src/app.rs:215-218`). There is no validation that the manifest hasn't
been tampered with or that the on-chain registration still matches.

### 4c. On-chain registration metadata

Vmm does **not** touch the chain — there is no `ethers` / `alloy` / web3
dep in `vmm/Cargo.toml`. The chain interaction lives in:
- `kms/auth-eth/` (Hardhat project: `DstackKms`, `DstackApp` contracts).
- The CVM-side guest-agent (which talks to KMS, which talks to the chain).
- The `vmm-cli.py` admin tool (only for `kms:create-app`, `app:add-hash`
  workflows, called via `npx hardhat`).

**There is therefore no vmm-side backup of on-chain state.** App registration
(EOA → app contract → device id whitelist → compose-hash whitelist) is
recoverable only by replaying the Hardhat scripts and reading the chain.
Nothing in vmm checkpoints "this app id was registered at block N with
device id D and whitelist {h1, h2}." A lost Ethereum private key for an
app's upgrade authority is *unrecoverable* from anything vmm has.

### 4d. Backups vmm should have for v1

- A `vmm-backup` command (or RPC) that produces a verifiable archive of:
  - `<run_path>/<vm_id>/` minus volatile sockets/pids
  - QMP-coordinated qcow2 snapshot of `hda.img`
  - A manifest digest signed by the vmm's identity
- Documented restore procedure with vmm offline.
- Per-org export/import (multi-tenant): "give me all CVMs for org X, with
  manifests, encrypted env, app-compose, but not the disk images" so an
  org can be migrated between control planes.
- For the chain side: out of scope for vmm, but the control plane should
  cache the on-chain authorisation graph (org → app → instance) so it can
  detect drift / re-register automatically.

---

## 5. Upgrade story

### 5a. How vmm itself gets upgraded

There is **no online-upgrade primitive**. Operations runbook is:
1. Stop the vmm process. (No SIGTERM handler — `grep` for `signal::|ctrl_c|SIGTERM`
   in `vmm/src/` finds nothing. Rocket's default Ctrl-C handling runs,
   but supervisor has no graceful-shutdown coupling from vmm.)
2. Replace the binary.
3. Start the new vmm.
4. New vmm calls `reload_vms()` (`vmm/src/main.rs:224`) which scans
   `<run_path>/`, reads each `vm-manifest.json`, repopulates the
   in-memory `AppState`, and — for any VM whose `vm-state.json.started ==
   true` — calls `start_vm` (`vmm/src/app.rs:215-218, 815-820`).

### 5b. Are CVMs durable across vmm restarts?

**Yes, mostly.** The QEMU processes are owned by the **supervisor**
sidecar, not by vmm itself (`vmm/src/main.rs:209-222`,
`supervisor/src/main.rs`). So when vmm exits:
- The supervisor process keeps running (started via
  `SupervisorClient::start_and_connect_uds`,
  `supervisor/client/src/lib.rs:26-88`; if `detached = true` it daemonises).
- Each QEMU child is a child of supervisor. When vmm comes back, it
  reconnects to the supervisor's UDS, calls `supervisor.list()`, finds the
  running QEMU, and reconciles
  (`vmm/src/app.rs:545-621`).
- Orphaned supervisor processes (in supervisor but not loaded as a vmm
  VM, e.g. workdir manifest unreadable) are cleaned up by
  `spawn_finish_remove` in `reload_vms()` (`vmm/src/app.rs:594-602`).

**Caveats.**
- If supervisor itself dies (e.g. crash, OOM, segfault), all QEMU
  processes that supervisor spawned die too because of `.kill_on_drop(true)`
  on the `tokio::process::Command` in
  `supervisor/src/process.rs:186`. **Supervisor crashes drop all CVMs.**
- Supervisor has zero on-disk state. Its `processes: DashMap` lives in
  RAM only (`supervisor/src/supervisor.rs:31-34`). On restart it knows
  about no processes — it has to be told via `deploy(...)`. Vmm's
  `reload_vms()` does this on its own startup, but a supervisor restart
  *without* a vmm restart leaves the CVMs running but unmanaged (they
  become orphans of a dead PPID, and a future supervisor restart can't
  recover them — they'll show up in `ps` but not in `supervisor.list()`).
  This is fragile.
- The `auto_restart_task` (`vmm/src/main.rs:132-146`) runs every
  `cvm.auto_restart.interval` seconds (default 20 s, `vmm.toml:106-107`)
  and re-starts any VM that's `started=true` but no longer running.
  This is the only restart-loop. It will retry **forever** with no
  back-off — a continuously-crashing VM will be restart-looped at
  20 s intervals indefinitely, eating CPU and log bandwidth. There is
  no crash-loop circuit breaker.
- Image binary (`dstack-vmm`) is on the host filesystem. There is no
  staged-rollout, no canary, no safe-rollback hook.

### 5c. CVM image upgrade vs vmm upgrade

A CVM image upgrade is a separate, supported flow: `update_vm` /
`upgrade_app` (`vmm/src/main_service.rs:373-453`) can change
`compose_file`, `encrypted_env`, `user_config`, `gpus`, `ports`, and the
guest image (only when stopped, see `apply_resource_updates`
`vmm/src/main_service.rs:235-287`). Disk shrink is rejected. No
attestation re-binding is done at the vmm layer — the new app id will be
recomputed from compose hash if not provided
(`vmm/src/main_service.rs:46-55, 388-389`).

### 5d. Multi-tenant gap

- No graceful-shutdown handling: vmm needs a SIGTERM handler that closes
  the listener, drains in-flight RPCs, and detaches cleanly from
  supervisor.
- No SO_REUSEPORT/zero-downtime hot-swap of the binary.
- No supervisor-side persistent state — make supervisor crash-tolerant by
  having it write its `ProcessConfig` set to disk so it can rebuild
  `processes` on restart without help from vmm.
- Crash-loop detection / exponential backoff in `auto_restart_task`.
- Coordination with the multi-host control plane during upgrades
  (drain → upgrade → uncordon).
- Versioning of the on-disk schema (`vm-manifest.json` is implicit-versioned
  today; serde `#[serde(default)]` lets older files load, but nothing
  enforces "this manifest was written by vmm vN, this vmm is vN+1, do
  migration M").

---

## 6. Rate limiting / quotas / fairness

**None of these exist.**

- No rate limiting middleware in the rocket stack
  (`vmm/src/main.rs:81-100`). No `governor`, `tower::limit`,
  `axum-extra::middleware::rate_limit`, etc. — and rocket-apitoken is just
  a bearer-token gate (`vmm/src/main.rs:208`,
  `vmm/src/main_routes.rs:107-115`).
- No per-tenant primitives at all — there are no tenants. Auth is one
  flat list of bearer tokens (`vmm/src/config.rs:286-293`,
  `vmm/vmm.toml:127-129`). Either you have a token, in which case you can
  do everything, or you don't.

The **only** capacity controls are global config values, none of which
are enforced in the create-VM path:
- `cid_pool_size` (`vmm/vmm.toml:31`, default 1000) — caps total
  concurrent CVMs at 1000 per host. Enforced because `create_vm` allocates
  a CID and `IdPool::allocate` returns `None` when exhausted
  (`vmm/src/app/id_pool.rs:47-60`, `vmm/src/app.rs:194-198`,
  `vmm/src/app.rs:786`). This is a hard cap, not a soft quota, and it's
  per-host.
- `max_allocable_vcpu`, `max_allocable_memory_in_mb` (`vmm/vmm.toml:32-33`,
  defaults 20 / 100_000 MB) — these are **advisory only**. They are
  surfaced through `Vmm.GetMeta` (`vmm/src/main_service.rs:548-552`) so a
  client UI can pre-validate, but **`create_vm` does not enforce them**.
  Confirmed: `grep` for `max_allocable` in vmm src returns five hits, all
  of which only *read* the value to expose it; none of which compare it
  to incoming `vcpu` or `memory`. The `// Not yet implement fully` comment
  at `vmm/src/config.rs:163` is honest.
- `cvm.port_mapping.range` / `enabled` (`vmm/vmm.toml:98-103`) — a
  global allowlist of host ports that can be forwarded. Enforced in
  `create_manifest_from_vm_config` (`vmm/src/main_service.rs:141-167`).
  Not per-tenant.
- `cvm.gpu.enabled` / `cvm.gpu.allow_attach_all` / `include`/`exclude`
  (`vmm/vmm.toml:109-119`, `vmm/src/main_service.rs:72-83`). Global.
- `event_buffer_size` (`vmm/vmm.toml:14`, default 20) caps the per-VM
  in-memory event queue. Drops oldest. Not security-relevant; just
  prevents unbounded growth (`vmm/src/app.rs:920-921`).
- `serial_history_max_bytes` (`vmm/vmm.toml`, default 4 MB) caps per-VM
  serial-log history (`vmm/src/app.rs:1096-1137`). No equivalent cap on
  `stdout.log` / `stderr.log`.

There is no:
- Disk quota per tenant or even per-VM (other than the `disk_size` field
  in the manifest, which is a qcow2 max size, not a quota).
- Image-pull rate limiting (the OCI registry path
  `vmm/src/app/registry.rs` has no `Retry-After` handling).
- Concurrent-deploy rate limiting (a tenant could spam `create_vm` to
  exhaust the CID pool — currently each create allocates a CID
  immediately under the global mutex at `vmm/src/app.rs:191-214`).
- Egress / ingress bandwidth shaping.

### Multi-tenant gap

Need at minimum (per organization):
- Max active CVMs.
- Sum of vCPU across active CVMs.
- Sum of RAM across active CVMs.
- Sum of disk_size across active CVMs (and actual usage, not just declared).
- GPU count.
- Deploy rate (RPS or per-hour) on `create_vm` / `update_vm` /
  `pull_registry_image`.
- Log retention bytes per tenant.
- Image-storage bytes per tenant (`<image.path>` is shared today —
  every tenant pulling an image bloats one global directory).

These need to be enforced in the create_vm / update_vm / pull_image
paths *before* CID allocation or disk I/O happens, plus a periodic
reconciler that revokes runs over quota.

---

## 7. Failure modes

The handling code below is the *entirety* of what vmm does for each
failure today. Anything not cited here is unhandled.

### 7a. QEMU dies

- Detection: supervisor's `wait_on_process` task sees the child exit and
  flips `ProcessStatus` to `Exited(code)` or `Error(msg)`
  (`supervisor/src/process.rs:240-282`).
- Vmm's `auto_restart_task` ticks every `cvm.auto_restart.interval`
  seconds (default 20, `vmm/vmm.toml:106-107`) and finds VMs with
  `vm-state.json.started == true` that aren't in the supervisor's
  running set, then calls `start_vm` for each
  (`vmm/src/main.rs:132-146`, `vmm/src/app.rs:1050-1078`).
- No back-off, no max-retries, no quarantine. A QEMU that segfaults
  every boot will be restart-looped indefinitely. The serial log is
  rotated each restart (`vmm/src/app.rs:1099-1137`) so the *last 4 MB*
  of history is preserved, but stdout/stderr are append-forever.

### 7b. Host reboots

- All QEMU processes die with the host. After the host comes back:
  - If supervisor is run as a systemd service (or whatever), it restarts
    with **empty in-memory state** — supervisor has no on-disk
    `processes` map (`supervisor/src/supervisor.rs:31-34`).
  - When vmm starts, it calls `reload_vms()` which scans
    `<run_path>/`, repopulates AppState, and for VMs marked
    `started=true` calls `start_vm` (`vmm/src/app.rs:545-621,
    215-218, 700-820`).
  - This re-deploys QEMU through supervisor (`vmm/src/app.rs:262-273`).
- **Risk**: if vmm starts before supervisor (or before supervisor's UDS
  is bound), `SupervisorClient::start_and_connect_uds`
  (`supervisor/client/src/lib.rs:26-88`) will retry up to 10 times with
  growing 100–1000 ms backoff and then bail. There's no systemd
  unit-ordering shipped with the project that I can see.
- **Risk**: nothing checks whether the qcow2 image is consistent or
  needs `qemu-img check`. A dirty shutdown that corrupted hda.img is
  re-launched as-is; QEMU may refuse to boot, in which case the
  restart-loop in §7a kicks in.
- No durability guarantee on `vm-state.json` writes (no fsync, no
  atomic-rename — see `vmm/src/app/qemu.rs:1024-1028`,
  `safe_write::safe_write` is used for `instance_info` only at
  `vmm/src/app.rs:937-940`). A power loss between `set_started(true)`
  and the write hitting the platter could lose the started flag.

### 7c. KMS unreachable

- The only direct vmm→KMS call is `get_app_env_encrypt_pub_key`
  (`vmm/src/main_service.rs:455-468`, `vmm/src/app.rs:1007-1014`). It
  is on the synchronous path of the create-VM flow only when a client
  needs to encrypt env vars before deploying.
- Failure mode: `kms_client()` errors with "KMS is not configured" if
  url is empty (`vmm/src/app.rs:1008-1010`); otherwise the RPC call
  returns the underlying connection error and it propagates out the
  pRPC response.
- **No retries, no caching of the public key, no circuit breaker, no
  fallback URL list** (the `manifest.kms_urls` field exists at
  `vmm/src/app.rs:67`, and `cvm.kms_urls` exists at
  `vmm/vmm.toml:25`, but the *vmm* client only uses
  `config.kms_url` — the singular non-list field
  `vmm/src/config.rs:329-330`. The list is only ever passed to the
  guest as part of `make_sys_config` at `vmm/src/app.rs:1143-1152`.
  Inside the CVM, the guest-agent does have failover behaviour, but
  the vmm itself does not.)
- **Critically**, KMS being down does not affect already-running CVMs
  on this vmm — they got their keys at boot via host-API
  (`vmm/src/host_api_service.rs`) or directly. It only affects new
  deploys that need an env-encrypt pubkey.

### 7d. Gateway unreachable

- vmm makes **no direct HTTP call to the gateway**. Confirmed by
  searching `vmm/src/` for gateway client usage. The gateway URLs are
  passed verbatim into `<workdir>/shared/sys-config.json` for the
  guest to read (`vmm/src/app.rs:1148-1152, 1157-1167`).
- Gateway-down therefore manifests as: "guest can't register itself
  with the gateway, app traffic doesn't route." Vmm has no awareness
  of this and exposes no metric/health-signal for it. The only
  derived URL exposed is `app_url` in `VmInfo` which is *constructed*
  from gateway config (`vmm/src/app/qemu.rs:276-303`); a 503 against
  it does not feed back into vmm.

### 7e. On-chain RPC down

- vmm does not talk to the chain. See §4c.
- The downstream effect is borne by KMS (which fails to verify app
  authorization) and by the human operator running `vmm-cli.py`
  (which proxies into Hardhat scripts). A multi-tenant control
  plane will need its own chain-RPC failover and caching layer; vmm
  contributes nothing here today.

### 7f. Disk full

- Not handled. There is no statvfs check before:
  - `create_hd` (`vmm/src/app/qemu.rs:113-136`) — `qemu-img create`
    will error out, the bail propagates from `start_vm`
    (`vmm/src/app.rs:262-268`), the create flow at
    `vmm/src/main_service.rs:291-326` runs the cleanup `if let Err(err)
    { fs::remove_dir_all(work_dir) }` and returns the error.
  - `prepare_work_dir` `fs::write` of `app-compose.json`,
    `encrypted-env`, `user-config` (`vmm/src/app.rs:964-994`).
  - `put_manifest` (`vmm/src/app/qemu.rs:1006-1011`) — partial-write
    risk because there's no atomic rename.
  - The serial / stdout / stderr logfile writes inside QEMU and
    supervisor — these silently fail and the bytes are dropped (the
    `redirect()` task in `supervisor/src/process.rs:345-424` logs the
    error but does not stop the process).
  - The `serial_history.log` rotation reads, then writes, the file
    (`vmm/src/app.rs:1110-1135`); ENOSPC during truncation could
    leave the file empty.
- No metric / alert / readiness-flip on a near-full disk.

**Multi-tenant gap.** Need:
- Pre-flight statvfs at create_vm time, accounting for the requested
  `disk_size` plus a per-VM workdir budget. Refuse with a friendly
  error.
- Periodic background check that emits a metric and refuses new
  deploys past a high-water mark.
- ENOSPC-aware writes for state files; use `safe_write` consistently
  (currently only used for `instance_info`).

### 7g. Supervisor unreachable / crashed

- Vmm calls `supervisor.list()` and `supervisor.info(id)` constantly
  (every 20 s for auto-restart at minimum). All these
  return errors via the HTTP client at
  `supervisor/client/src/lib.rs:90-108`. There is no circuit breaker,
  no backoff. Each error is logged at `error!`/`warn!` level
  (e.g. `vmm/src/app.rs:336-363, 547`) and the operation continues.
- If supervisor's UDS is gone, the next `supervisor.deploy` will bail
  with "Failed to start process", which is observable through the
  `start_vm` return path. Vmm has no auto-recovery — a sysadmin must
  bring supervisor back.

---

## 8. Resource accounting

**Vmm tracks almost nothing in real time.**

What it knows:
- Allocated CIDs (in-memory `IdPool`, `vmm/src/app.rs:1258`).
- Per-VM declared `vcpu`, `memory` (MB), `disk_size` (GB) from the
  manifest (`vmm/src/app.rs:48-72`).
- GPU slots reserved (`vmm/src/app.rs:1023-1028`,
  `vmm/src/main_service.rs:86-132`).
- Whether each CVM is `running|stopped|exited|stopping|removing`
  (`vmm/src/app/qemu.rs:312-361`).
- Per-VM uptime / exited_at, through supervisor's
  `started_at` / `stopped_at` (`supervisor/src/process.rs:51-66`).

What it does **not** know:
- Actual current CPU/RAM consumption per VM (no QMP query loop, no
  cgroup polling, no `proc/<pid>/status` reader).
- Actual disk usage on hda.img (qcow2 is sparse, so declared
  `disk_size` is an upper bound, not actual).
- Actual disk usage on `<workdir>/serial.log` etc.
- Network bytes in / out per VM.
- A summary "this org has used X vCPU-hours."
- An "owner" of any CVM. There is no `org_id` field. There is an
  `app_id` (`vmm/src/app.rs:50`) which is the SHA256 of the
  app-compose, not a tenant identifier.

**How it would learn (forward design):**
- vCPU/RAM live: query QMP `query-memory-size-summary`, `query-cpus-fast`
  on a polling interval, plus cgroup-v2 `memory.current` / `cpu.stat`
  if VMs are placed in per-VM cgroup slices.
- Disk live: `du -sb <workdir>` periodically, plus
  `qemu-img info hda.img` for actual_size.
- Network live: read `/proc/net/dev` for the per-VM tap interface in
  bridge mode, or the userspace forwarder counters in user-mode +
  bridge-with-forward mode (`port-forward/` crate — counters not
  exposed today).
- Owner: introduce an `org_id` field on `Manifest`
  (`vmm/src/app.rs:48-73`) and propagate it through every API and
  every workdir.

---

## 9. Audit / forensics

**There is no audit log.**

- No append-only event store. No "who did what when."
- No structured request log (Rocket's default access log goes to stderr
  alongside everything else).
- No retention policy on the data we *do* keep.
- The only durable per-VM artifacts are:
  - `vm-manifest.json` (`vmm/src/app/qemu.rs:990-1011`) — overwritten
    on update_vm; no history.
  - `vm-state.json` — overwritten on every start/stop.
  - `app-compose.json` / `encrypted-env` / `user-config` —
    overwritten on update_vm.
  - `serial.history.log` — last 4 MB only, truncated from the front.
  - `stdout.log` / `stderr.log` — append-forever, externally rotatable.
  - `instance_info` — written by guest events
    (`vmm/src/app.rs:936-940`).
  - `events: VecDeque<GuestEvent>` — last `event_buffer_size` (20
    by default) per VM, **in memory only** — lost on vmm restart
    (`vmm/src/app.rs:910-921`, `vmm/src/app.rs:1225`).
- The `guest-ip` file (`vmm/src/app/qemu.rs:1057-1069`) is
  point-in-time, no DHCP-lease history.

**On-chain side.** App registration, compose-hash whitelist changes, and
device-id additions *are* on-chain (Phala's KMS contracts at
`kms/auth-eth/`). Those are auditable forever. Anything that *doesn't*
go through KMS-mediated chain calls (every vmm-side action — create_vm,
start_vm, update_vm before chain registration, etc.) has no permanent
record.

### 9a. What a multi-tenant deployment must retain

Minimum forensic trail:
- Every authentication event (token used, source IP, success/failure).
- Every mutating RPC: caller identity, org id, target VM id, before/after
  state diff, timestamp, git-rev of vmm.
- Every CVM lifecycle transition: create → start → exit (with code) →
  remove, with timestamps.
- Every QEMU restart by `auto_restart_task`, with reason.
- Every config change (e.g. `update_vm` with new compose).
- Last N days of serial console + stdout/stderr per CVM, retained even
  after CVM removal for some grace period.
- Cross-component correlation: ideally a `request_id` that tracks an
  org-initiated action through vmm → KMS → guest-agent (today no such
  id exists).

### 9b. Forensics gaps that block multi-tenant ship

- No way to answer "who deleted CVM X" — there is no caller identity
  in `Vmm.RemoveVm`, no audit row, and the workdir is gone.
- No way to answer "which org was running app Y when it crashed at
  03:14 UTC last Tuesday" — events are in-memory only.
- No tamper-evident log. Even if we add an audit table, a malicious or
  buggy vmm could rewrite it. (Optional: append-only on a hashchain or
  off-host storage.)
- No PII redaction. `user_config` and `encrypted_env` may contain
  secrets that we'd inadvertently log.

---

## Prioritised must-haves

### Must-have for v1 production

1. **Health/readiness probes.** `/healthz` (liveness, trivial), `/readyz`
   (returns 200 only after `reload_vms` completes and supervisor probe
   succeeds). Without these, no load balancer or orchestrator can
   safely route traffic. Smallest possible change; biggest immediate
   payoff. *Today: nothing exists.*
2. **Prometheus metrics endpoint** at `/metrics` (gated by config, like
   KMS does it). Minimum series: CVM count by state, create/start/stop
   counters with outcome, supervisor RPC latency, KMS RPC latency,
   disk free per `image.path` / `run_path`, build_info gauge.
   Without this you cannot operate the daemon at any scale.
3. **Per-org identity threaded through the API.** Right now there is
   no `org_id` anywhere. Without it nothing else in this list (quotas,
   audit, multi-host scheduling) is even definable. Practical first
   step: add `org_id` to `Manifest`, persist in `vm-manifest.json`, and
   require it on `Vmm.CreateVm`. Auth tokens map to org. Backwards-compat:
   migrate existing manifests on first reload to a synthetic
   `default` org.
4. **Quota enforcement on create_vm.** At minimum: max CVMs per org,
   sum-of-vcpu, sum-of-memory, sum-of-disk_size. Enforce *before*
   CID allocation. Implement `max_allocable_*` fields properly while
   you're here (`vmm/src/config.rs:163` — currently surfaced but never
   checked).
5. **Audit log.** Per-RPC structured event with caller identity, org,
   target, action, outcome, timestamp, request_id. Sink to a local
   append-only file at minimum; ship to ELK/Loki/etc in prod. Required
   to answer "who did what" for any incident response.
6. **Graceful shutdown + supervisor crash-tolerance.** Vmm needs a
   SIGTERM handler that drains. Supervisor needs to persist its
   `processes` map so a supervisor crash doesn't lose every CVM
   without forewarning. (Currently a supervisor crash kills every
   QEMU and the next supervisor restart has no idea those VMs ever
   existed.)
7. **Crash-loop circuit breaker.** `auto_restart_task` should
   exponential-backoff and quarantine after N failures. A
   continuously-segfaulting QEMU shouldn't burn a CPU.
8. **Disk-pressure pre-flight.** Pre-statvfs check in `create_vm`,
   `update_vm` (resize), and `pull_registry_image`. Refuse with a
   clean error rather than failing mid-write.
9. **Structured (JSON) log output**, gated by config, with a
   `request_id` correlation field. Without this, log shipping to
   any modern stack is a parser-writing exercise.
10. **Backup/restore documentation and a tested procedure.** Even if
    the procedure is "stop vmm, snapshot the workdir, copy it, start
    vmm" — write it down, test it, automate it. Production today has
    *no* documented backup workflow.

### Nice-to-have later

11. **Distributed tracing** with OTLP export and W3C TraceContext
    propagation through vmm → supervisor → KMS → guest-agent. Very
    valuable, but you can ship v1 with structured logs + request IDs.
12. **Cross-host control plane**: scheduler, shared state store, node
    heartbeat, drain/uncordon, live-migration. Massive scope. The
    "multi-host support" question hangs on whether v1 ships single-host
    multi-tenant first (likely yes) and then evolves.
13. **Live resource accounting**: real CPU/RAM/disk/net measurement per
    CVM, exposed both via metrics and via a `Vmm.GetCvmUsage` RPC.
    Distinct from quota enforcement (which only needs declared values).
14. **Image/registry quotas** per tenant, log retention quotas per
    tenant, network egress accounting. Real but secondary.
15. **Hot binary upgrade** (zero-downtime vmm restart). The current
    "stop, swap, start" path is acceptable if `reload_vms` is fast
    enough; revisit if it isn't.
16. **Online log-level reload** via a debug RPC, instead of having to
    restart the daemon to flip from info → debug.
17. **Tamper-evident audit log** (hashchain / off-host append-only).
    Defense-in-depth, not v1 critical.
18. **Per-CVM cgroup placement** so that CPU/RAM accounting is precise
    and enforcement (cgroup limits) is possible. Today QEMU runs as a
    direct child of supervisor with no cgroup-isolation.
19. **Snapshot-aware backup** integrated with QMP
    (`blockdev-snapshot-internal-sync` etc.). Today QMP socket is
    disabled by default.
20. **Multi-region / availability-zone awareness** in the scheduler.
    Only relevant once the cross-host control plane exists.
