# dstack-vmm CVM Lifecycle & State Reference

**Prepared:** 2026-05-05
**Scope:** End-to-end trace of how `dstack-vmm` manages a CVM, from "user clicks deploy" to "running CVM" to "deletion", with focus on persistence, in-memory state, recovery, and single-tenant assumptions.
**Coverage:** `vmm/src/`, `supervisor/src/`, `port-forward/src/`, `dstack-types/src/shared_filenames.rs`, plus `vmm.toml` schema.

---

## 1. Executive Summary

`dstack-vmm` is a single-process Rocket server that manages a flat fleet of CVMs (Confidential VMs) on one TDX host. There is no scheduler, no notion of tenancy, no transactional state store. The entire system is a thin RPC layer over **two persistent surfaces** plus **one in-memory map**, glued together by a child `supervisor` process that owns QEMU lifetimes.

**The two persistent surfaces:**
1. **`run_path/<vm-id>/` workdir** (default: `~/.dstack-vmm/vm/<uuid>/`) — one directory per CVM, holding manifest, compose file, encrypted env, kernel/initrd-derived shared files, the qcow2 disk, and runtime artefacts (PTY, qmp socket, logs, pidfile, `.removing` marker).
2. **The `supervisor` child process** — keeps a `DashMap<id, Process>` of every QEMU it spawned. It is *not* persistent across its own restarts; it dies, every QEMU dies (`kill_on_drop = true`).

**The one in-memory map:**
- `App.state: Arc<Mutex<AppState>>` in `vmm/src/app.rs:128`–157, holding `cid_pool`, `vms: HashMap<id, VmState>`, `active_forwards`. This is rebuilt on every vmm restart by walking `run_path/`.

**Recovery model:** trust-the-disk. On startup, `App::reload_vms` lists every subdirectory of `run_path`, parses each `vm-manifest.json`, queries the supervisor for already-running QEMU processes, reconciles CIDs, then re-spawns any VM whose `vm-state.json` has `started: true` but is not currently running in supervisor. There is no journal; the only marker for "remove was started but never completed" is a 0-byte `.removing` file in the workdir.

**Concurrency:** A single `std::sync::Mutex<AppState>` (not Tokio) guards every field of the global state; locks are taken briefly inside async methods. Port forwarding additionally holds `tokio::sync::Mutex<ForwardService>`. Long-running cleanup is `tokio::spawn`'d off the lock.

**Single-tenant assumption is everywhere:** there is no user/owner/tenant column on any data structure. The CID pool, port-mapping range, GPU pool, and disk path are all global. VM IDs are random UUIDs but ownership is implicit ("whoever can talk to the supervisor socket"). The `setup-user.sh` script even sandboxes *all* CVMs to a single Unix UID (`cvm.user`) at the host level.

The remainder of this document traces every call path, every file path, every lock, and every state transition, with `file:line` citations throughout.

---

## 2. End-to-End Flow Narrative

### 2.1 "User clicks Deploy" → CreateVM

1. **HTTP arrives.** Rocket dispatches to `/prpc/Vmm.CreateVm` → `RpcHandler::create_vm`. Auth is a single shared bearer-token list (`AuthConfig`, `vmm/src/config.rs:288–293`); no caller identity.
2. **Manifest is built.** `create_manifest_from_vm_config` (`vmm/src/main_service.rs:135–201`):
   - Generates a fresh `uuid::Uuid::new_v4()` as the VM id (line 173).
   - If `app_id` is not provided, derives it as `truncate40(sha256(compose_file))` (line 46–55, 169–172).
   - Validates `name` charset and port-mapping ranges against `cvm.port_mapping`.
   - Resolves GPU spec against `cvm.gpu` (allow-attach-all gating).
3. **Workdir is laid out on disk.** `RpcHandler::create_vm` (`vmm/src/main_service.rs:291–327`):
   - `vm_work_dir.put_manifest(&manifest)` writes `vm-manifest.json` (`qemu.rs:1006–1011`).
   - `prepare_work_dir` (`app.rs:964–994`) creates `<workdir>/shared/` and writes `app-compose.json`, `.encrypted-env`, `.user-config`, and an initial `.instance_info` containing only `{ "app_id": "<hex>" }`.
   - `vm_work_dir.set_started(!request.stopped)` writes `vm-state.json` (`qemu.rs:1024–1028`).
4. **VM is registered in memory.** `App::load_vm` (`app.rs:168–219`):
   - Parses the manifest, loads the image metadata via `Image::load` (`image.rs:67`).
   - Allocates a CID from `cid_pool` (`app.rs:198`) — defined as a `BTreeSet` covering `[cid_start, cid_start + cid_pool_size)`, default `[1000, 2000)` (`vmm.toml:30–31`).
   - Builds `VmConfig { manifest, image, cid, workdir, gateway_enabled }` and calls `AppState::add` to insert into `vms: HashMap<String, VmState>`.
5. **VM is started (if `!stopped`).** `App::start_vm` (`app.rs:221–275`):
   - `sync_dynamic_config` (`app.rs:996–1005`) writes `<workdir>/shared/.sys-config.json`, regenerated on every start (KMS URLs, gateway URLs, host_api vsock url, derived os_image_hash, etc., from `make_sys_config` `app.rs:1139–1168`).
   - Optionally rotates `serial.log` → `serial.history.log` and writes a "===== boot @ <ts> =====" separator into `stdout.log`/`stderr.log` (`app.rs:1082–1137`).
   - `VmConfig::config_qemu` (`qemu.rs:388–772`) builds the full QEMU command:
     - Disk: creates `hda.img` qcow2 if missing (using image's `hda` as backing), sets perms `0o660` if `cvm.user` is set (lines 401–406).
     - Networking: derives MAC deterministically from `sha256(vm_id)` with the configured `mac_prefix` (`qemu.rs:40–56`); `user`, `bridge`, or `custom` netdev (`qemu.rs:504–530`).
     - Shared dir delivery: `9p`, `vvfat`, or `vhd` (line 555–599) — the `vhd` mode synthesises a fresh FAT32 image at `shared.img` from `<workdir>/shared/` on each start (`qemu.rs:139–200`).
     - vsock: `-device vhost-vsock-pci,guest-cid=<cid>` (line 552).
     - TDX: builds `-object tdx-guest,...,mrconfigid=<base64(MrConfig)>,quote-generation-socket=...` from `app_compose_hash` plus `instance_info.app_id` (`qemu.rs:774–866`).
     - SMBIOS, GPU/NUMA, hugepages, taskset, optional `sudo -u <cvm.user>` wrapper, all assembled into a `ProcessConfig`.
   - `App::supervisor.deploy(&process)` ships the config over a Unix-socket HTTP-JSON channel to the `supervisor` binary (`supervisor/client/src/lib.rs:117–119`).
6. **Supervisor spawns QEMU.** `Supervisor::deploy` (`supervisor/src/supervisor.rs:54–73`): builds a `Process`, calls `process.start()` which `tokio::process::Command::spawn`s with `kill_on_drop(true)`, writes the PID to the `pidfile`, redirects stdout/stderr to the log paths under the workdir (with logrotate-aware reopen logic, `process.rs:363–423`), and stores the `Process` in `DashMap<String, Process>`.
7. **Inside the guest** (out of scope for this doc): `dstack-guest-agent` posts events back over vsock to the host_api (`HostApiHandler::notify` → `App::vm_event_report`, `app.rs:902–946`), updating in-memory `boot_progress`, `boot_error`, and writing `<workdir>/shared/.instance_info` when the guest reports its derived instance_id.

### 2.2 Steady state

- The fleet view (`Status` RPC, `app.rs:826–876`) merges:
  - In-memory `VmState` (manifest + boot/shutdown progress + recent guest events)
  - Per-VM live process info from `supervisor.list()` (status, pid, started_at)
  - On-disk `vm-state.json` `started` flag (read every call to derive the human "running/stopped/exited/stopping" status, `qemu.rs:312–342`)
- A background `auto_restart_task` ticks every `cvm.auto_restart.interval` seconds (default 20s, `vmm.toml:105–107`) and re-`start_vm`s any VM whose disk says `started=true` but is not in the supervisor's running set (`main.rs:132–146`, `app.rs:1050–1078`).
- DHCP leases for bridge-mode VMs flow in via `report_dhcp_lease` RPC: VMM matches MAC → vm_id, persists IP to `<workdir>/guest-ip`, and reconfigures user-space port forwarding via the `dstack-port-forward` crate (`app.rs:423–528`).

### 2.3 "User clicks Delete" → RemoveVM

1. **Marker is set.** `App::remove_vm` (`app.rs:291–321`):
   - Sets `VmStateMut.removing = true` under the lock (idempotent — second call returns early).
   - Writes `<workdir>/.removing` (zero-byte marker, `qemu.rs:1108–1118`).
   - Tears down active port forwards.
   - Spawns `finish_remove_vm(id, delete_workdir=true)` on a tokio task and returns immediately. The RPC reply is fire-and-forget.
2. **Background cleanup.** `App::finish_remove_vm` (`app.rs:327–392`):
   - `supervisor.stop(id)` (idempotent if already stopped).
   - Polls `supervisor.info(id)` every 2s indefinitely until status != `Running`. Logs every minute. There is **no timeout**: the comment notes some VMs may take 2+ hours.
   - `supervisor.remove(id)` to drop the `Process` from supervisor's `DashMap`.
   - If `delete_workdir || .removing` exists: `fs::remove_dir_all(workdir)`. (Orphan cleanup without the marker preserves the workdir.)
   - Frees the CID from `cid_pool` and removes from `vms` map.
3. **Guest-initiated shutdown.** Separately, `ShutdownVm` RPC (`main_service.rs:509–512`) just calls into the guest agent over vsock and asks it to power off; QEMU exits, `auto_restart_task` will *not* restart it (because `vm-state.json` still says `started=true` — see Section 8 for the subtle interaction).

---

## 3. Topic 1 — Entry Points

### 3.1 RPC entry — pRPC over Rocket

| Surface | Mount path | Handler | Notes |
|---|---|---|---|
| Public/external pRPC | `/prpc` | `RpcHandler` (`vmm/src/main_service.rs:34`) | Bearer-token auth via `rocket-apitoken`. Single shared token list, no per-caller identity. |
| Guest API proxy | `/guest` | `GuestApiHandler` (`vmm/src/guest_api_service.rs:14`) | Forwards calls down vsock to `dstack-guest-agent` at `vsock://<cid>:8000/api`. |
| Host API | `/api` (vsock only) | `HostApiHandler` (`vmm/src/host_api_service.rs:16`) | Bound on **vsock-only**, validated at startup (`config.rs:454–466`). Identifies caller by **vsock CID**. |
| Web UI | `/`, `/v1`, `/v0`, `/beta` | `main_routes.rs:45–63` | Serves embedded HTML/JS. |
| Log streaming | `/logs?id=...&ch=...` | `vm_logs` (`main_routes.rs:107–183`) | Tails `serial.log`, `stdout.log`, or `stderr.log` from the workdir. |
| OpenAPI | `/api-docs` | Mounted by `ra_rpc::rocket_helper::mount_openapi_docs` (`main.rs:102`) | |

The full RPC service (`Vmm` in `vmm/rpc/proto/vmm_rpc.proto:300–360`):

| RPC | `RpcHandler` impl | What it does |
|---|---|---|
| `CreateVm` | `main_service.rs:291–327` | Build manifest, prepare workdir, optionally start. |
| `StartVm` | `main_service.rs:329–335` | `App::start_vm`. |
| `StopVm` | `main_service.rs:337–343` | Persist `started=false`, clean port forwards, supervisor.stop. |
| `RemoveVm` | `main_service.rs:345–351` | Set marker, tokio::spawn cleanup. |
| `UpgradeApp` / `UpdateVm` | `main_service.rs:373–453` | Rewrites `app-compose.json`, optionally rewrites encrypted-env / user-config / ports / vcpu / mem / disk / gpu / no_tee / kms_urls / gateway_urls. Returns new `app_id`. |
| `ShutdownVm` | `main_service.rs:509–512` | Delegates to guest agent over vsock. |
| `ResizeVm` (deprecated) | `main_service.rs:484–507` | Subset of UpdateVm. |
| `GetComposeHash` | `main_service.rs:565–572` | Pure SHA256 over compose_file string (no compose normalization performed by VMM). |
| `Status` | `main_service.rs:353–355` → `App::list_vms` | Cross-joins in-memory VmState + supervisor.list(). Pagination + keyword filter. |
| `ListImages` / `DeleteImage` / `ListRegistryImages` / `PullRegistryImage` | `main_service.rs:357–741` | Image catalog. Pull is via OCI distribution v2 to a temp dir then atomic rename. |
| `GetAppEnvEncryptPubKey` | `main_service.rs:455–468` | Forwards to KMS. |
| `GetInfo` / `GetMeta` / `Version` / `ListGpus` | `main_service.rs:470–563` | Read-only. |
| `ReloadVms` | `main_service.rs:574–577` → `App::reload_vms_sync` | Resync memory ↔ disk ↔ supervisor. |
| `ReportDhcpLease` | `main_service.rs:579–582` → `App::report_dhcp_lease` | Bridge-mode hook. |
| `SvList` / `SvStop` / `SvRemove` | `main_service.rs:584–617` | Direct supervisor escape hatches — leak the supervisor abstraction outright. |

### 3.2 Process entry — `main.rs`

`main.rs:148–236` orchestrates startup:

1. Init tracing.
2. Load `vmm.toml` via `load-config` crate.
3. Validate `host_api` is vsock-only (`config.rs:454–466`).
4. Subcommand dispatch:
   - `serve` (default) — full server.
   - `run <vm_config>` — one-shot mode (`one_shot.rs`), spins up a single QEMU directly without supervisor or RPC. Uses `ps aux` to scan for in-use CIDs (`one_shot.rs:21–50`).
5. `discovery::cleanup_stale_registrations()` (`discovery.rs:113–144`) — delete `$XDG_RUNTIME_DIR/dstack-vmm/<uuid>.json` for any vmm whose pid is dead.
6. `DiscoveryRegistration::register` writes a JSON file describing this vmm process so `vmm-cli.py` can find it (`discovery.rs:56–97`).
7. Construct `ApiToken` from `auth.tokens` config.
8. Start (or connect to) the supervisor child process via `SupervisorClient::start_and_connect_uds` (`supervisor/client/src/lib.rs:26–88`). If the unix socket already pings, reuse; else spawn `supervisor` binary detached.
9. `App::new(config, supervisor)` — empty in-memory state, fresh CID pool.
10. `state.reload_vms()` — walk `run_path/`, rebuild memory state, resume cleanup of `.removing` workdirs, kill orphan supervisor processes.
11. `tokio::spawn(auto_restart_task)`.
12. `tokio::select!` runs `run_external_api` and `run_host_api` in parallel.

---

## 4. Topic 2 — Compose Normalization

`dstack-vmm` does **not** normalize the compose file at all. The VMM treats `compose_file` as an opaque string:

- `RpcHandler::get_compose_hash` (`main_service.rs:565–572`) and `app_id_of` (`main_service.rs:46–55`) both compute `sha256(request.compose_file)` and `truncate40(...)` for the app id. There is no key-sorting, no whitespace normalization, no UTF-8 canonicalization. **The deterministic JSON rules in `docs/normalized-app-compose.md` are the responsibility of the *caller* (CLI / SDK / UI / Hardhat tasks)** — VMM hashes whatever bytes it received.
- The compose file is parsed only via `serde_json::from_str::<AppCompose>` (`main_service.rs:380, 568`) for *validation* (does it parse?). The parsed `AppCompose` struct is in `dstack-types/src/lib.rs:13–52`. After validation the original *string* is written byte-for-byte to disk.
- Storage path: `<run_path>/<vm-id>/shared/app-compose.json` (constants in `dstack-types/src/shared_filenames.rs:5`, joined in `app.rs:948–950`, written in `app.rs:973`).
- The *compose hash* used for TDX `mrconfigid` is computed over the on-disk bytes at boot time: `VmWorkDir::app_compose_hash` (`qemu.rs:1038–1043`) reads the file and SHA256s the raw bytes. It feeds into `MrConfig::V1` or `MrConfig::V2` (`qemu.rs:796–826`) which then becomes the base64 mrconfigid in the QEMU `tdx-guest` object (`qemu.rs:850–864`).
- The user's encrypted env (`.encrypted-env`) and user config (`.user-config`) are likewise opaque blobs written to the same `shared/` directory.

**Implication for rewrite:** if you want a single canonical compose hash (matching what gets registered on-chain), normalization must happen *before* it ever reaches the VMM, or the VMM rewrite must adopt the deterministic-JSON canonicalization itself.

---

## 5. Topic 3 — Resource Allocation

### 5.1 CID (vsock context ID) pool

- **Pool definition:** `cvm.cid_start` and `cvm.cid_pool_size` in `vmm.toml:30–31` (default `[1000, 2000)`).
- **Implementation:** `IdPool<u32>` at `vmm/src/app/id_pool.rs:24–69` — a `BTreeSet<u32>` of allocated IDs, sequential first-fit from `start+1` upward (the implementation skips `start` itself, see line 48–58).
- **Allocation paths:** `App::load_vm` line 198, `App::load_or_update_vm` line 786. Failure mode: returns "CID pool exhausted".
- **Reservation paths:** `App::reload_vms` line 559–562 (`occupy` for each CID returned by `supervisor.list`), `App::reload_vms_sync` line 650–654 (clears + re-occupies).
- **Free paths:** `App::finish_remove_vm` line 386 (`cid_pool.free`), `App::reload_vms_sync` line 729–737 (frees CIDs no longer attached to any VM).
- **One-shot mode** (debug) bypasses the pool entirely: `one_shot.rs:21–57` parses `ps aux` for `guest-cid=` to find a free number.

### 5.2 TCP port mapping

- **Global allowlist:** `cvm.port_mapping` in `vmm.toml:98–103`. Schema: `{ enabled: bool, address: IpAddr, range: [{ protocol, from, to }] }` (`config.rs:122–142`).
- **Per-VM port_map:** array of `PortMapping { address, protocol, from, to }` stored in the manifest (`app.rs:41–46`). Each entry is validated against `pm_cfg.is_allowed` at create/update time (`main_service.rs:144–167`).
- **Two enforcement modes:**
  - **User-mode networking:** ports go into the QEMU `-netdev user,...,hostfwd=...` string (`qemu.rs:512–520`). Kernel-allocated host listener. There is **no host-side allocator** that prevents two VMs from picking the same host port — collisions surface as QEMU bind failures.
  - **Bridge mode + `forward_service_enabled`:** VMM runs userspace TCP/UDP forwarders via `dstack-port-forward` (`port-forward/src/lib.rs:54–125`). Rules are diffed and applied incrementally on DHCP-lease updates (`app.rs:454–528`). Active rules tracked in `AppState.active_forwards: HashMap<vm_id, Vec<ForwardRule>>` (`app.rs:1257–1262`).

### 5.3 Disk paths

All disk paths are derived from `run_path / <vm-id> / ...`. There is no central allocator; the VM ID (UUID) is the only namespace.

- **qcow2 disk:** `<workdir>/hda.img` (`qemu.rs:1096–1098`). Created on first start with `qemu-img create -f qcow2 -o backing_file=<image-hda> <size>G` (`qemu.rs:113–135, 401–406`). Resize is done via `qemu-img resize` while the VM is stopped (`main_service.rs:271–284`); shrink is rejected.
- **Shared-disk image** (vhd mode): `<workdir>/shared.img`, regenerated on every start (`qemu.rs:582–588, 139–200`).
- **Image directory** (kernel/initrd source-of-truth): `<image.path>/<image_name>/` (default `~/.dstack-vmm/image/<image_name>/`, `config.rs:524–528`). Each image dir contains `metadata.json`, `kernel`, `initrd`, optional `bios`/`hda`/`rootfs`, optional `digest.txt` (`image.rs:67–117`).

### 5.4 MAC addresses

- **Deterministic, derived from `sha256(vm_id)` with optional global `mac_prefix`.** `mac_address_for_vm` (`qemu.rs:40–56`).
  - `mac_prefix` is configured globally in `cvm.networking.mac_prefix` (`config.rs:412–414`) — 0–3 hex bytes.
  - The first byte is forced to locally-administered + unicast: `(b & 0xfe) | 0x02`.
  - Remaining bytes come from the VM-id hash.
- No collision check, no allocator, no persistence. The full 6-byte space (minus the first-byte constraint) is treated as effectively collision-free.

### 5.5 GPU allocation

- **Discovery:** `lspci` filtered by `cvm.gpu.listing` product IDs (`config.rs:262–285`).
- **No locking:** the VMM does *not* reserve a GPU. `App::try_allocate_gpus` (`app.rs:1023–1028`) just clones what the manifest says. Two VMs configured for the same `slot` will fail at QEMU spawn time when vfio-pci grabs the device.
- **Attach modes:** `Listed` (per-VM list of slots) or `All` (find every NVIDIA device + NVSwitch bridges, `main_service.rs:86–132`). `All` is gated by `cvm.gpu.allow_attach_all`.

---

## 6. Topic 4 — Process Spawning

### 6.1 QEMU command construction — `VmConfig::config_qemu`

Single source of truth: `vmm/src/app/qemu.rs:388–772`. Roughly 380 lines of `command.arg(...)` assembly. Highlights:

- Always: `-accel kvm -cpu host -nographic -nodefaults`.
- Serial: `-chardev pty,id=com0,path=<workdir>/serial.pty,logfile=<workdir>/serial.log` and `-serial chardev:com0` (lines 420–425).
- Optional: `-qmp unix:<workdir>/qmp.sock,server,wait=off` (lines 426–431).
- Boot: `-bios <bios?>`, `-kernel <kernel>`, `-initrd <initrd>` (lines 432–436). Optional `rootfs` of `.iso` (cdrom) or `.verity` (read-only virtio-blk).
- Disk: `-drive file=<workdir>/hda.img,if=none,id=hd1` + `-device virtio-blk-pci,drive=hd1` (lines 475–479).
- Networking: see §5.4.
- vsock: `-device vhost-vsock-pci,guest-cid=<cid>` (line 552).
- Shared dir: 9p / vvfat / vhd (lines 555–599).
- Hugepages + NUMA: optional (lines 601–653).
- GPUs / NVSwitches: vfio-pci with iommufd (lines 656–701).
- TDX object: confidential-guest-support=tdx, plus a `-object tdx-guest,...` JSON-encoded with mrconfigid + quote-generation-socket (lines 781–865).
- SMBIOS types 0/1/2/3 from `cvm.product` config (lines 868–910).
- Final wrapper: optional `taskset -c <cpus>` for NUMA pin, optional `sudo -u <cvm.user>` (lines 738–750).

The `ProcessConfig` returned to the supervisor (lines 756–769) carries:
- `id` (= vm UUID), `name`, `command`, `args`, `cwd` (= workdir), `stdout`/`stderr` (= log paths in workdir), `pidfile` (= `<workdir>/qemu.pid`), `cid`, and `note` = `{"kind":"cvm","live_for":null}` JSON.

### 6.2 Supervisor

The `dstack-supervisor` crate (`supervisor/`) is a separate binary the VMM auto-starts and connects to via Unix socket (`supervisor/client/src/lib.rs:26–88`).

- **State:** `Supervisor` (`supervisor/src/supervisor.rs:18–138`) holds `Arc<SupervisorState>` with `freezed: AtomicBool` and `processes: DashMap<String, Process>`. `Process` (`process.rs:149–310`) holds `Arc<ProcessConfig>` and `Arc<Mutex<ProcessStateRT>>` where the rt state has `status`, `started`, `pid`, `kill_tx: Option<oneshot::Sender<()>>`, `started_at`, `stopped_at`.
- **API:** REST-over-HTTP (JSON) on a unix socket: `POST /deploy`, `POST /start/<id>`, `POST /stop/<id>`, `DELETE /remove/<id>`, `GET /list`, `GET /info/<id>`, `GET /ping`, `POST /clear`, `POST /shutdown` (`supervisor/src/web_api.rs:40–106`).
- **Lifetime semantics:** `tokio::process::Command::spawn` with `kill_on_drop(true)` (`process.rs:181–201`). When the supervisor process itself exits, its `tokio::process::Child` handles drop and `SIGKILL` every QEMU. **There is no detached/disowned mode.**
- **Stop:** sends a oneshot to the wait-task which `process.kill().await`s (`process.rs:284–301, 323–343`).
- **Crash detection:** the wait-task transitions status to `Exited(code)` when the child returns (`process.rs:251–265`). Nothing notifies the VMM; the VMM polls via `supervisor.list()` and `auto_restart_task` decides whether to restart based on `vm-state.json`.
- **Persistence: NONE.** When supervisor restarts, its `DashMap` is empty and any QEMUs that survived (impossible thanks to `kill_on_drop`, but if process death is unclean) are orphaned — but in practice every QEMU has died by then.

### 6.3 Tracking processes across vmm restarts

The VMM's *only* knowledge of the running QEMUs comes from querying the running supervisor. The supervisor itself has no persistence, but because the VMM auto-spawns the supervisor on first run and `start_and_connect_uds` reuses an existing socket if pings (`client/src/lib.rs:36–39`), the typical pattern is:

1. VMM crashes → supervisor keeps running → QEMUs keep running.
2. VMM restarts → reconnects to existing supervisor → `supervisor.list()` returns the live processes → CIDs reoccupied, in-memory state rebuilt.

If the supervisor *also* dies, every QEMU dies with it, and the VMM will spawn a fresh supervisor, find an empty list, and restart any VM whose `vm-state.json` says `started=true` (via `auto_restart_task` shortly after).

---

## 7. Topic 5 — State Written to Disk

### 7.1 Per-CVM workdir layout

Root: `<config.run_path>/<vm-id>/` (default `~/.dstack-vmm/vm/<uuid>/`). Defined in `Config::abs_path` (`config.rs:378–384`) and `Config::extract_or_default` (`config.rs:506–561`).

| Path | Producer | Consumer | Format | Lifetime |
|---|---|---|---|---|
| `vm-manifest.json` | `VmWorkDir::put_manifest` (`qemu.rs:1006–1011`) | every load (`app.rs:175, 754`) | JSON serialization of `Manifest` struct (`app.rs:48–73`) | Whole VM lifetime |
| `vm-state.json` | `VmWorkDir::set_started` (`qemu.rs:1024–1028`) | every status read + auto_restart | JSON: `{"started": bool}` (`qemu.rs:108–111`) | Whole VM lifetime |
| `.removing` | `VmWorkDir::set_removing` (`qemu.rs:1116–1118`) | `is_removing` checked at reload + cleanup | Empty file (existence is the signal) | Removal in progress |
| `guest-ip` | `VmWorkDir::set_guest_ip` (`qemu.rs:1068–1070`) on DHCP lease | port forward reconfig + reload restore | Plain text, `<ipv4>` | While VM runs in bridge mode |
| `qemu.pid` | Supervisor (`process.rs:218–225`) | external observers | PID as text | While QEMU runs |
| `serial.log` | QEMU (`-chardev ...,logfile=...`) | log streaming, history rotation | Binary serial output | Truncated by QEMU on each start |
| `serial.history.log` | `rotate_serial_log` (`app.rs:1096–1137`) | log streaming | Concatenated past serial sessions, capped at `cvm.serial_history_max_bytes` (default 4 MB) | Across all boots |
| `serial.pty` | QEMU | external attach | PTY device node (symlink) | While QEMU runs |
| `qmp.sock` | QEMU (if `cvm.qmp_socket=true`) | external QMP clients | Unix socket | While QEMU runs |
| `stdout.log`, `stderr.log` | Supervisor `redirect` task (`process.rs:345–423`) | log streaming | Append-only with watcher-driven logrotate detect | Across all boots; gets boot-separator on each start (`app.rs:1082–1092`) |
| `hda.img` | `qemu-img create` (`qemu.rs:113–135`) | QEMU | qcow2, backing-file = image's `hda` | Whole VM lifetime, deleted on remove |
| `shared.img` | `create_shared_disk` (`qemu.rs:139–200`) when `host_share_mode=vhd` | QEMU | FAT32 image, label `DSTACKSHR`, regenerated on every start | Recreated on each start |
| `shared/app-compose.json` | `prepare_work_dir`, `update_vm` | guest agent + `app_compose_hash` | JSON (opaque to VMM) | Whole VM lifetime |
| `shared/.encrypted-env` | `prepare_work_dir`, `update_vm` | guest agent | Opaque bytes | Whole VM lifetime (optional) |
| `shared/.user-config` | `prepare_work_dir`, `update_vm` | guest agent | Plain text or whatever the user wrote | Whole VM lifetime (optional) |
| `shared/.instance_info` | `prepare_work_dir` initial; `vm_event_report("instance.info")` (`app.rs:936–940`) | mrconfigid V2 build (`qemu.rs:805–812`) + status display | JSON: `{ instance_id, app_id (hex bytes) }` (`qemu.rs:76–82`) | Whole VM lifetime |
| `shared/.sys-config.json` | `sync_dynamic_config` (`app.rs:996–1005`) on every start, via `make_sys_config` (`app.rs:1139–1168`) | guest agent | JSON: `{ kms_urls, gateway_urls, pccs_url, docker_registry, host_api_url, vm_config }` (vm_config is itself a JSON string, `app.rs:1170–1195`) | Rewritten every start |

Constants for the `shared/` filenames: `dstack-types/src/shared_filenames.rs:1–20`.

### 7.2 Other persistent state

- **VMM discovery file:** `$XDG_RUNTIME_DIR/dstack-vmm/<random-uuid>.json` (`vmm/src/discovery.rs:18–24, 56–97`). Schema: `VmmInstanceInfo { id, pid, address, working_dir, config_file, image_path, run_path, node_name, version, started_at }`. Removed on graceful shutdown via `Drop` (`discovery.rs:100–110`); stale entries cleaned by `cleanup_stale_registrations` on next start.
- **Supervisor pid file / log file / unix socket:** `cvm.supervisor.{pid_file, log_file, sock}` from `vmm.toml:131–134`. Defaults relative to vmm cwd: `./run/supervisor.{pid,sock,log}`.
- **VMM unix socket:** `unix:./vmm.sock` by default (`vmm.toml:11`).

### 7.3 No database, no journal, no checkpoint

There is no SQLite, no file-locking abstraction, no two-phase commit. Edits to manifest/compose are done with direct `fs::write` (no temp-file-rename pattern except the `.tmp-pull-<tag>` directory for image pulls in `registry.rs:111–147`). A crash mid-write of `vm-manifest.json` or `app-compose.json` would corrupt the workdir.

---

## 8. Topic 6 — Mutable In-Memory State

### 8.1 The big mutex

`vmm/src/app.rs:128–166`:

```rust
pub struct App {
    pub config: Arc<Config>,
    pub supervisor: SupervisorClient,
    state: Arc<Mutex<AppState>>,                   // std::sync::Mutex
    forward_service: Arc<tokio::sync::Mutex<ForwardService>>,
    pull_status: Arc<Mutex<HashMap<String, PullStatus>>>,
}

pub(crate) struct AppState {                       // app.rs:1257–1284
    cid_pool: IdPool<u32>,
    vms: HashMap<String, VmState>,
    active_forwards: HashMap<String, Vec<ForwardRule>>,
}

pub struct VmState {                                // app.rs:1212–1255
    pub(crate) config: Arc<VmConfig>,              // immutable per VM-edit cycle
    state: VmStateMut,
}

struct VmStateMut {                                 // app.rs:1218–1228
    boot_progress: String,
    boot_error: String,
    shutdown_progress: String,
    guest_ip: String,
    devices: GpuConfig,
    events: VecDeque<pb::GuestEvent>,
    removing: bool,
}
```

- The lock is a `std::sync::Mutex` (NOT `tokio::sync::Mutex`), accessed via `App::lock` (`app.rs:139–141`) which `or_panic`s on poison. All async methods hold it briefly. Long-running cleanup tasks use `tokio::spawn` and re-acquire as needed.
- No `RwLock` — readers and writers contend on the same mutex.
- No actor / mailbox pattern — calls are synchronous-under-lock.
- `pull_status` is held in a separate mutex so background image pulls don't block VM operations.

### 8.2 Static counters

`STREAM_CREATED_COUNTER` and `STREAM_DROPPED_COUNTER` in `main_routes.rs:76–105` track open log streams.

### 8.3 Supervisor's in-memory state

`SupervisorState { freezed: AtomicBool, processes: DashMap<String, Process> }` (`supervisor/src/supervisor.rs:31–34`). DashMap shards locking; per-process state is its own `Arc<Mutex<ProcessStateRT>>` (`process.rs:152`).

### 8.4 Forward service state

`ForwardService { cancel: CancellationToken, rules: HashMap<ForwardRule, RunningRule> }` (`port-forward/src/lib.rs:49–52`). Single `tokio::sync::Mutex` wrapper held by `App.forward_service`.

---

## 9. Topic 7 — Recovery on Restart

`App::reload_vms` (`app.rs:545–621`) is called once at startup (`main.rs:224`). The procedure is:

1. **Query supervisor for live processes:** `supervisor.list().await` returns every `ProcessInfo`. For each, parse the `note` JSON to determine if it's a CVM (`note.is_cvm()` in `config.rs:369–376`). Build `occupied_cids: HashMap<vm_id, cid>`.
2. **Reserve those CIDs in the pool:** for each occupied CID, `cid_pool.occupy(cid)` (`app.rs:558–562`). Errors here would mean the supervisor has duplicate CIDs — the `?` propagates and reload fails.
3. **Walk `run_path/`:** for each subdirectory:
   - Read `vm-manifest.json`.
   - Validate image name charset (lines 176–184).
   - Load image metadata.
   - Allocate or reuse a CID:
     - If already in memory: keep its CID.
     - Else if supervisor has it running: use that CID.
     - Else: allocate a fresh CID.
   - Insert into `vms: HashMap`.
   - If `.removing` marker exists, **load the VM into memory but do not auto-start**, then queue for cleanup.
   - Else if `vm-state.json.started == true`, call `start_vm` (which is idempotent for already-running QEMUs because `start_vm` checks `supervisor.info(...).is_running()` first — `app.rs:231–235`).
4. **Resume cleanup of `.removing` workdirs:** `spawn_finish_remove(id)` for each (lines 588–591).
5. **Clean up orphan supervisor processes:** any process the supervisor knows about but for which no workdir exists → `spawn_finish_remove(id)` (lines 593–603). With `delete_workdir=false`, the cleanup respects the `.removing` marker (which it doesn't have, since the workdir is gone, so the workdir is preserved if it ever reappears — see comments at `app.rs:366–380`).
6. **Restore port forwarding:** for each loaded VM with a `guest-ip` file present, set `vm.state.guest_ip` and call `reconfigure_port_forward` (lines 605–618).

**`reload_vms_sync`** (`app.rs:624–744`) is the on-demand variant exposed via the `ReloadVms` RPC — it adds reconciliation passes to remove VMs whose workdir has disappeared and to emit counts of loaded/updated/removed.

**Trust model:** the disk is canonical. There is no integrity check on `vm-manifest.json` (no signature, no checksum). A malicious or corrupt manifest with `vcpu=999` and a hand-crafted image name would be loaded as-is (the only check is image-name charset, lines 176–184). The `.removing` marker is similarly trust-on-read.

---

## 10. Topic 8 — Lifecycle States & Transitions

### 10.1 Effective state set

The status string returned to clients is computed by `VmState::merged_info` (`qemu.rs:312–342`) from three signals:

| Signal | Source |
|---|---|
| `started` | `vm-state.json.started` on disk |
| `is_running` | supervisor reports `ProcessStatus::Running` |
| `removing` | in-memory `VmStateMut.removing` flag |

The displayed status:

| started | is_running | removing | Status string | Meaning |
|---|---|---|---|---|
| – | – | true | `removing` | `.removing` marker is set, cleanup task running |
| true | true | false | `running` | Normal happy path |
| true | false | false | `exited` | QEMU died but `started=true` → auto_restart will revive |
| false | true | false | `stopping` | User asked for stop, QEMU not yet exited |
| false | false | false | `stopped` | Idle |

### 10.2 Explicit transitions (RPC-driven)

| Trigger | Effect |
|---|---|
| `CreateVm` | Workdir created, manifest/compose written, `vm-state.json.started = !req.stopped`. If `!stopped`, `start_vm` queued. → `running` or `stopped`. |
| `StartVm` | `set_started(true)`. If supervisor doesn't show running, build QEMU process and `supervisor.deploy`. → `running` (or `exited` if QEMU dies fast). |
| `StopVm` | `set_started(false)`, cleanup port forwards, `supervisor.stop`. → `stopping` → `stopped`. |
| `RemoveVm` | Set `removing=true`, write `.removing` marker, cleanup port forwards, `tokio::spawn(finish_remove_vm)`. → `removing` → entry deleted from memory + disk. |
| `UpdateVm` / `UpgradeApp` / `ResizeVm` | Rewrite manifest / compose / encrypted-env / user-config. Resize requires `stopped`/`exited`. Calls `load_vm` to refresh in-memory `VmConfig`. Does **not** restart the VM (caller must `StopVm` then `StartVm`). |
| `ShutdownVm` | Sends shutdown to guest agent over vsock. **Does not change `vm-state.json.started`.** Result: QEMU exits cleanly, status becomes `exited`, `auto_restart_task` will restart it shortly. (To prevent restart, the guest must also report a `shutdown.progress = "powering off"` event, which the host_api handler interprets as setting `started=false` — `app.rs:931–934`.) |

### 10.3 Implicit transitions

| Trigger | Effect |
|---|---|
| QEMU exits unexpectedly | Supervisor records `Exited(code)`. VMM still has `vm-state.json.started=true`. `auto_restart_task` (every `cvm.auto_restart.interval` seconds) restarts it. |
| Supervisor reports running, but no workdir | At reload, `spawn_finish_remove` schedules cleanup that does **not** delete the (already-missing) workdir but does call `supervisor.stop/remove`. |
| VM has `.removing` marker but vmm restarts | Reload re-loads it for visibility, then `spawn_finish_remove` resumes cleanup. |
| DHCP lease for unknown MAC | Logged at debug, dropped. |
| Guest reports `boot.error` | Stored in `VmStateMut.boot_error`, surfaced in `Status`. No restart logic uses this. |
| Guest reports `shutdown.progress = "powering off"` | `set_started(false)` is called (`app.rs:931–934`) — this is the *only* implicit `started=false` transition. |

### 10.4 Failure / inconsistent states

- **Crashed-but-not-cleaned-up:** if vmm itself panics during `RemoveVm` *before* the marker is written and the spawn fires, the workdir, supervisor process, and CID are all leaked. On restart, reload picks up the workdir, re-allocates the CID, and the VM appears as a normal stopped VM.
- **Orphan supervisor process** (workdir gone, supervisor still has it): cleaned up at next reload (`app.rs:593–603`).
- **Orphan workdir** (workdir present, supervisor doesn't have it, `started=true`): auto_restart will start it.
- **Half-applied UpdateVm:** if the process dies between writing `app-compose.json` and writing the manifest, the next start will use the new compose with the old manifest. There is no rollback.

---

## 11. Topic 9 — CVM Deletion: Cleanup & Orphans

`App::remove_vm` → `App::finish_remove_vm` (`app.rs:291–392`):

**Always cleaned up:**
- Active port-forward rules (`cleanup_port_forward`, `app.rs:531–543`).
- Supervisor process (`supervisor.stop` then `supervisor.remove`).
- CID returned to the pool.
- In-memory `VmState` removed from the `vms` map.
- Workdir (`fs::remove_dir_all`) — provided either `delete_workdir=true` (user-initiated) or the `.removing` marker is present.

**Potentially orphaned (failure cases):**
- If `fs::remove_dir_all` fails partway, the residue is logged but not retried — on next reload, the partial workdir would be re-loaded if it still has a manifest, or skipped silently if it doesn't.
- If `supervisor.stop` succeeds but `supervisor.remove` fails (warn-and-continue at `app.rs:351`), the supervisor still has a stopped Process entry. The workdir/CID/memory are still cleaned. On the next reload it would be cleaned up by the orphan-supervisor branch.
- The `qemu.pid` pidfile is left in the workdir, but since the workdir is removed, this is a non-issue. If `delete_workdir=false`, the pidfile may point at a dead PID.
- The serial PTY (`serial.pty`) — QEMU creates this as an actual `/dev/pts/N` link; the link inside the workdir is removed with the workdir, the underlying PTY is closed when QEMU exits.
- **GPU passthrough state**: the VMM does not explicitly unbind/rebind vfio drivers. If the host kernel modules don't release the device on QEMU exit, manual intervention is needed. Not tracked anywhere.
- **Encrypted env / instance keys / disk_crypt_key:** the qcow2 disk is destroyed with the workdir, but the KMS (separate component) still holds derived keys for that `app_id`/`instance_id`. Re-creating a VM with the same compose file would re-derive the same `app_id` and unlock the same keys. The VMM does not signal KMS on delete.

**The `.removing` marker is the only crash-recovery signal.** It is a zero-byte file (`qemu.rs:1116–1118`); existence is the signal.

---

## 12. Topic 10 — "Single Operator, Single Tenant" Assumptions

These are the places where multi-tenancy is impossible without surgery. Each item is a concrete file:line reference.

### 12.1 No identity on the wire

- **`pRPC requests carry no caller identity.**` `RpcCall::construct` (`main_service.rs:746–751`) does not propagate any caller info beyond the bearer-token check. `CallContext` doesn't carry a user/tenant claim.
- **Bearer-token auth is a global allowlist:** `AuthConfig { enabled: bool, tokens: Vec<String> }` (`config.rs:288–293`); checked by `rocket-apitoken` (`main.rs:208`). Any token in the list can do anything.
- **Host API uses vsock CID as identity** (`host_api_service.rs:24–31`) — implicitly tied to the VM, but anyone who can talk to the host vsock listener as that CID is authoritative. There's no signed claim.

### 12.2 Global pools, no per-tenant carve-out

- **CID pool is global:** `App::new` (`app.rs:151–166`), `cvm.cid_start` / `cvm.cid_pool_size` (`vmm.toml:30–31`).
- **Port mapping range is global:** `cvm.port_mapping.range` (`vmm.toml:98–103`, `config.rs:122–142`).
- **GPU listing/include/exclude is global:** `cvm.gpu` (`vmm.toml:109–119`, `config.rs:248–285`).
- **MAC prefix is global:** `cvm.networking.mac_prefix` (`config.rs:412–414`).
- **Image directory is global** and shared across all VMs: `image.path` (`config.rs:313–320`, default `~/.dstack-vmm/image`).
- **`run_path` is global:** every workdir is a sibling under it (`config.rs:328`, default `~/.dstack-vmm/vm`).
- **Resource caps are global advisory limits returned by `GetMeta` only:** `cvm.max_allocable_vcpu`, `cvm.max_allocable_memory_in_mb` (`vmm.toml:32–33`, `main_service.rs:548–552`). Comment at `config.rs:163` even says "Not yet implement fully, only for inspect API `GetMeta`".

### 12.3 `vms: HashMap<String, VmState>` has no tenant column

- `AppState.vms: HashMap<String, VmState>` (`app.rs:1259`). Key is the VM UUID. There is **no** owner_id, project_id, org_id, or tenant_id field anywhere on `VmState`, `Manifest`, `VmConfig`, or `VmStateMut`.
- `Manifest` (`app.rs:48–73`) has `id, name, app_id, vcpu, memory, disk_size, image, port_map, created_at_ms, hugepages, pin_numa, gpus, kms_urls, gateway_urls, no_tee, networking` — no ownership.
- The on-disk `vm-manifest.json` therefore contains no ownership either; renaming `app_id` is the closest thing, but two tenants with the same compose file would share an `app_id`.

### 12.4 `Status` and listing are unfiltered

- `App::list_vms` (`app.rs:826–876`) returns every VM. The `StatusRequest` filters are `ids`, `keyword`, `page`, `page_size` — none of them tenant-aware (`vmm_rpc.proto:168–180`).
- Reload (`reload_vms`, `app.rs:545`) walks the entire `run_path` directory.

### 12.5 Single Unix UID for all CVMs

- `cvm.user` (`config.rs:171`, `vmm.toml:37`): one Unix user shared by every QEMU process the VMM spawns. The `setup-user.sh` script (`vmm/src/setup-user.sh:1–230`) explicitly says "create a sandbox user… Edit the user in the `[cvm]` section… `user = \"dstack-prd1\"`" — singular.
- `qemu.rs:743–748` wraps the QEMU command with `sudo -u <cvm.user>` only if it's set. Per-tenant Unix UIDs are not modeled.

### 12.6 Single KMS / Gateway / Registry per VMM

- `kms_url` (top-level config, `config.rs:330`) is one URL.
- `cvm.kms_urls` and `cvm.gateway_urls` (`config.rs:148–151`) are global lists. Per-VM overrides exist (`Manifest.kms_urls`, `Manifest.gateway_urls`) but are merged from the global default.
- `cvm.docker_registry` and `image.registry` (`config.rs:155, 319`) are single global URLs.
- `host_api.address` is `vsock:2` and `host_api.port` is one port (`vmm.toml:140–141`); every VM talks to the same one.

### 12.7 Discovery & CLI assume one host

- `vmm/src/discovery.rs:18–24`: discovery directory is `$XDG_RUNTIME_DIR/dstack-vmm/`, i.e. **per Unix user on the host**. The CLI scans `/run/user/<uid>/dstack-vmm/` for *every* uid (`vmm-cli.py:41–59`) so an operator can see "all instances on the host" — explicitly host-local.
- A multi-host control plane would need a different discovery mechanism entirely.

### 12.8 No quota / no admission control

- The VMM does not pre-check whether host RAM, disk, or CPU has capacity before spawning a QEMU. The supervisor will happily try to allocate hugepages or pin to a NUMA node that's full and let QEMU fail.
- `max_allocable_vcpu` / `max_allocable_memory_in_mb` are *advertised* via `GetMeta` (`main_service.rs:548–552`) but never enforced.

### 12.9 `App::clone` is the global handle

- `App` is `Clone` (line 128) because everything inside is `Arc`. Every RPC handler is constructed with `App::clone` (`main_service.rs:746–750`, `host_api_service.rs:24–32`, `guest_api_service.rs:29–34`). There is no per-request scope.

### 12.10 Direct supervisor escape hatches

- `SvList`, `SvStop`, `SvRemove` (`main_service.rs:584–617`) expose the supervisor directly. Anyone with the API token can stop/remove arbitrary supervisor processes, including ones that *aren't* CVMs (the supervisor is not CVM-only by design — `live_for` field on `ProcessAnnotation` indicates non-CVM ephemeral processes, `config.rs:362–376`). Multi-tenancy would require rebuilding this surface or removing it.

---

## 13. Cross-Reference: Where the Lifecycle Code Lives

| Concern | Primary file | Key entry points |
|---|---|---|
| Process startup | `vmm/src/main.rs` | `main`, `auto_restart_task` |
| Top-level state | `vmm/src/app.rs` | `App`, `AppState`, `VmState`, `VmStateMut` |
| RPC handlers (lifecycle) | `vmm/src/main_service.rs` | `RpcHandler`, `create_manifest_from_vm_config` |
| QEMU command + workdir layout | `vmm/src/app/qemu.rs` | `VmConfig::config_qemu`, `VmWorkDir` |
| Image catalog | `vmm/src/app/image.rs`, `vmm/src/app/registry.rs` | `Image::load`, `pull_and_extract` |
| CID allocator | `vmm/src/app/id_pool.rs` | `IdPool` |
| Config schema | `vmm/src/config.rs` | `Config`, `CvmConfig`, `Networking` |
| Default config | `vmm/vmm.toml` | (see annotated above) |
| Discovery | `vmm/src/discovery.rs` | `DiscoveryRegistration` |
| One-shot mode | `vmm/src/one_shot.rs` | `run_one_shot` |
| HTTP/streaming routes | `vmm/src/main_routes.rs` | `vm_logs` |
| Host API (vsock) | `vmm/src/host_api_service.rs` | `HostApiHandler::notify` |
| Guest API proxy | `vmm/src/guest_api_service.rs` | `GuestApiHandler` |
| Supervisor service | `supervisor/src/supervisor.rs`, `supervisor/src/process.rs`, `supervisor/src/web_api.rs` | `Supervisor`, `Process` |
| Supervisor client | `supervisor/client/src/lib.rs` | `SupervisorClient` |
| Port forwarding | `port-forward/src/lib.rs` | `ForwardService` |
| Shared filenames | `dstack-types/src/shared_filenames.rs` | `APP_COMPOSE`, `INSTANCE_INFO`, etc. |
| Shared types | `dstack-types/src/lib.rs` | `AppCompose`, `SysConfig`, `VmConfig`, `KeyProviderKind` |
| MR-config (mrconfigid) | `dstack-types/src/mr_config.rs` | `MrConfig::V1`, `MrConfig::V2` |

---

## 14. Loose Ends Worth Calling Out for the Rewrite

1. **No transactional semantics** on workdir mutations: a partial write of `vm-manifest.json` corrupts the VM. Switching to `safe_write` (already used for `instance_info`, `app.rs:939`) for *every* mutation would be a cheap win.
2. **Polling-based reconciliation** (`auto_restart_task`, `cleanup_orphan` at reload) is the only consistency mechanism. There is no event bus from supervisor → VMM, no reactive state machine.
3. **`std::sync::Mutex` held across `await` boundaries is avoided** but the lock surface is large; refactoring to a per-VM lock or actor would scale better with concurrent operations.
4. **The supervisor RPC is leaked** through `SvList`/`SvStop`/`SvRemove`. A multi-tenant rewrite must wrap or remove these.
5. **The `app_id` is a SHA256 truncation of the compose file** — collision-free in practice but creates an *implicit* tenancy: any two users with the same compose file (e.g., two devs deploying the same example) would inherit the same KMS keys. This is by design for the current single-operator model and is the single most consequential design choice for a multi-tenant rewrite.
6. **`Manifest.kms_urls` and `gateway_urls`** allow per-VM overrides today. A tenant would naturally have their own KMS contract on-chain, so this hook is the right place to inject tenancy *if* the URL scheme can encode tenant identity.
7. **`one_shot` mode** (`one_shot.rs`) replicates ~70% of `start_vm` outside the App/Supervisor pipeline, including its own ad-hoc CID allocator (lines 21–57). This duplication will need to be reconciled or amputated in a rewrite.
8. **No bound on `boot_progress`/`shutdown_progress` strings; the events buffer is bounded** (`event_buffer_size`, `vmm.toml:14`, default 20). A misbehaving guest can still flood `boot_progress`/`boot_error` since those overwrite a single string each time (`app.rs:923–929`).

---

*End of report.*
