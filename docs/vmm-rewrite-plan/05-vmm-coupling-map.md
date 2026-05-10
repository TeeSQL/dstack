# 05 — VMM coupling map

How `dstack-vmm` is wired to the rest of the world today, scored by how
hard it would be to swap or to make the boundary multi-tenant aware.
Read this together with `04-vmm-internal-cvm-lifecycle-and-state.md` and
`06-auth-authz-multi-tenant-gaps.md`.

Repo paths in this report are relative to `/home/fbx/dstack`. All
file:line citations are exact at the commit `398c8b9a` (master).

---

## 0. TL;DR — who vmm actually talks to

For each external system, "tightness" is rated 1 (loose, single
adapter) → 5 (pervasive assumption, refactor touches everything).

| # | External system          | Transport / wire format                                  | Tightness |
|---|--------------------------|----------------------------------------------------------|-----------|
| 1 | The host OS              | direct syscalls + spawned child processes                | **5**     |
| 2 | QEMU                     | argv assembly + supervisor-spawned child + qemu-img CLI  | **5**     |
| 3 | dstack-supervisor        | HTTP-over-Unix-socket, JSON                              | **4**     |
| 4 | The operator (human)     | shared bearer-token list, single Rocket app, single host | **4**     |
| 5 | dstack-guest-agent (host_api inbound) | prpc-over-vsock, port 10000, no attestation / no auth | **3**     |
| 6 | Local key-provider (TCP) | raw length-prefixed JSON over loopback TCP               | **3**     |
| 7 | dstack-guest-agent (proxied API outbound) | prpc-over-vsock to `cid:8000/api`                | **2**     |
| 8 | dstack-kms               | prpc-over-HTTPS, single global URL, *attestation off*    | **2**     |
| 9 | OCI image registry       | reqwest HTTPS, OCI distribution API                      | **1**     |
|10 | DHCP server (dnsmasq)    | inbound prpc on the management API: `ReportDhcpLease`    | **1**     |
|11 | dstack-gateway           | **no direct calls** — only string URLs forwarded to guests | **1**   |
|12 | Ethereum / on-chain      | **no direct calls from vmm** — KMS and vmm-cli do that   | **0**     |

Two surprises worth flagging up front:

- **vmm has no on-chain dependency at all.** Smart-contract calls are
  done by the KMS server and by `vmm-cli.py`. The vmm itself never
  imports an Ethereum client and never talks JSON-RPC to a chain. See
  §3 for the search evidence.
- **vmm has no live RPC link to dstack-gateway either.** It only ever
  passes `gateway_urls` as strings to the guest in `.sys-config.json`,
  and constructs `https://{instance_id}-{port}.{base_domain}` URLs for
  the UI to render. See §2 for citations.

---

## 1. dstack-kms

**(a) Protocol / transport.** prpc (protobuf-over-HTTP, JSON-encoded
body, `?json` query suffix) over HTTPS. The client is built in
`vmm/src/app.rs:1007-1014`:

```rust
pub(crate) fn kms_client(&self) -> Result<KmsClient<RaClient>> {
    if self.config.kms_url.is_empty() { bail!("KMS is not configured"); }
    let url = format!("{}/prpc", self.config.kms_url);
    let prpc_client = RaClient::new(url, true)?; // <-- tls_no_check = true
    Ok(KmsClient::new(prpc_client))
}
```

Two things to call out:

1. The second arg to `RaClient::new` is `tls_no_check` — set to `true`,
   so vmm explicitly **disables TLS verification** for KMS calls
   (`ra-rpc/src/client.rs:87-94` and `:48-50`). There's no attestation
   verification on the KMS endpoint either; `verify_server_attestation`
   defaults to `true` in `RaClientConfig` but `RaClient::new` never
   wires a `cert_validator`, so attestation is effectively skipped
   (`ra-rpc/src/client.rs:114-121`).
2. The URL is taken from a **single** top-level config key
   `kms_url`, not the `[cvm] kms_urls` list (`vmm/src/config.rs:330`,
   `vmm/vmm.toml:13`). The `kms_urls` *list* is only ever forwarded
   into the guest's `.sys-config.json` for the guest agent to use.

**(b) Data exchanged.** vmm calls KMS exactly **one** RPC method:

| KMS method | Where called | Request | Response |
|---|---|---|---|
| `GetAppEnvEncryptPubKey(AppId{app_id})` | `vmm/src/main_service.rs:455-468` | hex-decoded app_id bytes | `PublicKeyResponse{public_key, signature, timestamp, signature_v1}` (`kms/rpc/proto/kms_rpc.proto:20-30`) |

That single hop is the *only* KMS dependency baked into vmm itself.
Everything else KMS does (`GetAppKey`, `GetKmsKey`, `SignCert`,
`Bootstrap`, `Onboard`, `GetAttestationInfo`, `ClearImageCache`) is
called by the guest agent or by humans / vmm-cli — never by vmm.

The KMS proto surface that *would* matter to a control plane is in
`kms/rpc/proto/kms_rpc.proto:95-110`.

**(c) Assumptions vmm makes about KMS.**

- Single global KMS endpoint — `Config.kms_url: String`, not a list
  (`vmm/src/config.rs:330`). If you want per-tenant KMS instances you
  have nowhere to put that today.
- Same trust domain — TLS check disabled, attestation check disabled.
  The implicit trust boundary is "the operator vouches for this URL".
- Anyone with vmm bearer-token auth can ask KMS to sign an env-encrypt
  pubkey for *any* app_id they pass in — the call is at
  `main_service.rs:455` and there is no per-tenant filter on `app_id`.
- The `app_id` namespace is global. There is no notion of which org or
  user "owns" an app_id; KMS authorisation is enforced on-chain inside
  KMS itself, not by vmm.
- The deprecated `[cvm] kms_urls` list (passed to guests) is an
  *operator*-set global default — no schema for "tenant X may only use
  KMS Y".

**(d) Failure handling.** Single hop, no retries, no circuit-breaker.
`RaClient` has a 5s connect timeout and 60s overall timeout
(`ra-rpc/src/client.rs:52-53`). On unreachable / 5xx / TLS error the
prpc call returns `Err` and the vmm RPC handler bubbles it up to the
HTTP caller with `?` in `main_service.rs:457-461`. No caching of the
last good response, no soft-degrade. The only place vmm *needs* KMS
during normal operation is when the operator clicks "create VM with
encrypted env" in the UI, so a KMS outage doesn't stop vmm itself —
but the create flow fails hard.

**(e) Tightness rating: 2/5 — loose.**

- Single function (`App::kms_client` at `app.rs:1007`).
- Single call site (`main_service.rs:456`).
- Single proto method.
- Pluggable in principle: `KmsClient<T>` is generic over the transport,
  and the call could be hidden behind a one-method trait
  `EnvEncryptionPubkeySource` without touching anything else.

Refactor cost is small. The much harder part is the *semantic*
coupling: today every VM transparently shares the global KMS, and the
on-chain governance assumes one KMS contract for the whole fleet. A
multi-tenant rewrite that wants per-org KMS instances has to think
about that, but the code change in vmm itself is tiny.

---

## 2. dstack-gateway

**vmm makes zero live RPC calls to dstack-gateway.** Confirmed by
grepping the entire vmm crate for gateway client usage — nothing. The
only "gateway" interactions are:

1. **Config forwarded into the guest.** `make_sys_config` at
   `vmm/src/app.rs:1139-1167` writes `gateway_urls` into
   `.sys-config.json` inside the guest's shared FAT image. The guest
   agent — not vmm — uses that to register with gateway on boot.
   Source of truth precedence: per-VM manifest `gateway_urls` first,
   else global `[cvm] gateway_urls` list (`app.rs:1148-1152`).
2. **URL construction for the UI / app_url.**
   `vmm/src/app/qemu.rs:276-303` builds
   `https://{instance_id}-{agent_port}.{base_domain}` by combining the
   per-VM `gateway_urls[0]` (if present) and the global
   `[gateway]` block (`base_domain`, `port`, `agent_port`,
   `vmm/src/config.rs:305-310`, defaults at `vmm/vmm.toml:121-124`).
   This is purely string-formatting; vmm does not call the gateway.
3. **`GetMeta` returns gateway settings to the UI** — pure passthrough
   of config (`main_service.rs:534-547`).

**(a) Protocol / transport.** None.

**(b) Data exchanged.** None directly. The "communication" is via the
guest agent: the agent reads `.sys-config.json` from the shared disk
and registers itself with gateway via WireGuard / RA-TLS (out of scope
for vmm).

**(c) Assumptions vmm makes about gateway.**

- A single `[gateway]` block is enough — `base_domain`, `port`,
  `agent_port` (`vmm/src/config.rs:305-310`). One vmm instance ⇒ one
  gateway domain. There's no concept of per-tenant subdomains.
- The ingress mapping pattern `{id}-{port}.{base_domain}` is hard-coded
  in `app/qemu.rs:288-302`. A tenant scoping like
  `{org}.{base_domain}` is not representable.
- Custom `gateway_urls` are accepted per-VM but they're free-form
  strings — vmm does no validation of who is allowed to direct
  traffic where.
- `gateway_enabled` per VM is derived from `app_compose.gateway_enabled()`
  (`app.rs:204`, `app/qemu.rs:357`); no policy control here.

**(d) Failure handling.** N/A — vmm doesn't touch gateway.

**(e) Tightness rating: 1/5 — barely coupled, but in a way that
handicaps multi-tenant.**

The *code* coupling is essentially zero. The *naming-scheme* coupling is
the real problem: one base domain per host, ingress derived from
`instance_id`. Fixing that for a multi-tenant SaaS means redesigning
the ingress model in gateway, vmm config, and the guest agent that
actually calls `gateway` to register — but in vmm itself the change is
surface-level (extend `GatewayConfig`, change one string format in
`app/qemu.rs:288-302`).

---

## 3. On-chain (Ethereum)

**vmm makes no on-chain calls.** Verified by grepping `vmm/` for
`ethers`, `web3`, `eth_`, `contract`, `json.?rpc`, `jsonrpc`,
`chain_id` — only the prpc framework and a Python helper in
`vmm-cli.py` use `eth_keys`/`eth_utils`, and *those are the CLI*, not
vmm. The vmm crate does not depend on `ethers-rs`, `alloy`, or any
EVM client (`vmm/Cargo.toml:12-62`).

Where on-chain logic actually lives:

- **dstack-kms** owns the chain interaction. KMS reads the on-chain
  whitelist of OS images and the per-app device-id allow list before
  releasing keys. See `kms/rpc/proto/kms_rpc.proto:51-61` (`GetMeta`
  reports `kms_contract_address`, `chain_id`, `app_auth_implementation`)
  and the auth backends in `kms/auth-eth/`, `kms/auth-mock/`,
  `kms/auth-simple/`.
- **vmm-cli.py** is what humans use to sign Ethereum-style messages
  for app registration / hash whitelist updates (`vmm/src/vmm-cli.py:26-27`
  imports `eth_keys`, `eth_utils`).
- The vmm process never signs, never verifies, and never queries the
  chain.

**(a) Protocol / transport.** N/A — there is no transport.

**(b) Data exchanged.** N/A.

**(c) Assumptions vmm makes about the chain.** Implicitly: that
*someone else* (operator + KMS contract) has already authorised the
`compose_hash`/`app_id` before the user clicks deploy. vmm does not
re-check.

The only on-chain leakage into vmm is in the data model:

- `app_id` is a 40-char hex string (truncated SHA256 of compose,
  `main_service.rs:46-55`). This is the format expected by KMS and the
  `DstackApp` contract. Changing it would ripple.
- KMS settings get a `kms_contract_address`/`chain_id` in their
  metadata (`kms_rpc.proto:58-60`) but vmm never reads them.

**(d) Failure handling.** N/A.

**(e) Tightness rating: 0/5 — no direct coupling.**

vmm is already insulated from chains. A multi-tenant rewrite can
choose its own org-to-EOA mapping without touching vmm code at all,
**provided** the rewrite is willing to keep the global `app_id`
namespace (or to teach KMS about per-org namespaces, which is not a
vmm concern).

---

## 4. dstack-guest-agent

vmm has *two* directions of interaction with guest agents — they're
asymmetric and worth separating.

### 4a. host_api: guest → host (inbound on vmm)

**(a) Protocol.** prpc-over-HTTP, but the listener is bound to a
**vsock** address, not TCP. Configured at
`vmm/vmm.toml:138-141` (`address = "vsock:2"`, `port = 10000`).
Validation lives at `vmm/src/config.rs:447-467`:

```rust
pub fn validate(&self) -> Result<()> {
    if !self.address.starts_with("vsock:") {
        anyhow::bail!("Host API address must be a vsock address ...");
    }
    Ok(())
}
```

i.e. vmm refuses to start unless the host_api binds vsock — TCP/UDS
paths are explicitly rejected for "security reasons".

The handler is mounted in `vmm/src/main.rs:111-130` via
`run_host_api(...)`, separate from the operator-facing API on port
9080 (`main.rs:77-109`).

**(b) Data exchanged.** Three RPC methods (`host-api/proto/host_api.proto:1-34`):

| Method | Direction | Data |
|---|---|---|
| `Info()` | guest pulls vmm version | `HostInfo{name, version}` |
| `Notify(event, payload)` | guest pushes lifecycle events | `boot.progress`, `boot.error`, `shutdown.progress`, `instance.info` (handled at `vmm/src/app.rs:902-946`) |
| `GetSealingKey(quote)` | guest asks for sealing key | calls local key-provider, returns `{encrypted_key, provider_quote}` (`vmm/src/host_api_service.rs:49-62`) |

`Notify` events are bounded by `event_buffer_size` (default 20,
`vmm/vmm.toml:14`, `app.rs:920-922`) and `instance.info` payloads are
written to disk via `safe_write::safe_write` at `app.rs:937-940`.

**(c) Assumptions vmm makes about the guest agent.**

- The guest is on the **same host** (vsock CID:port). No remote /
  cross-host guest agents.
- The guest's vsock CID identifies a VM uniquely. The handler
  trusts the vsock peer CID as authentication
  (`vmm/src/host_api_service.rs:21-33`):
  ```rust
  let Some(RemoteEndpoint::Vsock { cid, port }) = context.remote_endpoint else {
      bail!("invalid remote endpoint: {:?}", context.remote_endpoint);
  };
  ```
  Once a VM has a CID, anything on that CID is treated as that VM. No
  attestation, no key, no token — just CID.
- The set of allowed events is hard-coded in `app.rs:923-944`
  (`boot.progress`, `boot.error`, `shutdown.progress`, `instance.info`).
  Anything else logs an error.

**(d) Failure handling.** Errors propagate back as RPC errors; vmm
does not retry guest events. If `safe_write` for `instance.info` fails
the call returns an error. There is no queue / dead-letter; events
are dropped after `event_buffer_size`.

**(e) Tightness rating: 3/5 — the trust model is the coupling.**

The host_api is a clean prpc surface (one file,
`vmm/src/host_api_service.rs`, 64 lines). What's tight is:

- vmm assumes a single `host_api` listener per host (one vsock CID 2).
  In a multi-host control plane each host has its own CID 2 — that's
  fine — but the *application logic* that handles `Notify` is
  single-process: it walks the in-memory `vms` map by CID
  (`app.rs:909`). Multi-tenant pasting requires an org filter at the
  message-handler level, not the transport.
- The trust model "vsock-peer-CID == VM identity" is non-negotiable
  inside a single host. For a control plane that needs to delegate
  CVMs to a worker, the worker — not the control plane — has to host
  this listener.

Refactor: route `Notify` through a tenant-aware event sink (queue /
db / observer pattern) instead of mutating in-process state.

### 4b. proxied guest API: host → guest (outbound from vmm)

**(a) Protocol.** prpc-over-vsock client. Created on demand in
`vmm/src/app.rs:1016-1021`:
```rust
pub(crate) fn guest_agent_client(&self, id: &str) -> Result<GuestClient> {
    let cid = self.lock().get(id).context("vm not found")?.config.cid;
    Ok(guest_api::client::new_client(format!("vsock://{cid}:8000/api")))
}
```

Used by:

- `RpcHandler::shutdown_vm` (`main_service.rs:509-512`) — issues a
  graceful shutdown to the guest.
- `GuestApiHandler` (`vmm/src/guest_api_service.rs:36-58`) — proxies
  five guest API RPCs (`Info`, `SysInfo`, `NetworkInfo`,
  `ListContainers`, `Shutdown`) to the appropriate VM.

**(b) Data exchanged.** Defined in `guest-api/proto/guest_api.proto:132-153`
(`ProxiedGuestApi`). vmm just forwards.

**(c) Assumptions vmm makes about the guest.** Guest agent listens on
port 8000 inside every CVM. Hard-coded port. The `vsock://` URL scheme
is also hard-coded. The proxy assumes the guest on `cid:8000` is the
VM with that CID in vmm's in-memory state; no per-call attestation
check.

**(d) Failure handling.** Each call is wrapped in `?`; on connect or
RPC error the operator UI sees a generic "guest agent not reachable"
error. No retry, no backoff. `RaClient` defaults
(`ra-rpc/src/client.rs:52-53`) apply: 5s connect, 60s overall.

**(e) Tightness rating: 2/5 — loose.**

Two short functions. Replacing the transport (e.g. with a remote
worker proxying back to a control plane) means rewriting these two
functions and adding a "guest agent reachability" component on the
worker side.

---

## 5. The host OS

This is the source of most of the coupling. vmm assumes:

**(a) Protocol / transport.** Direct syscalls and process spawning.
There is no abstraction layer between vmm and the kernel.

**(b) Data exchanged / files / sockets / mounts.**

| Resource | Used for | Citation |
|---|---|---|
| `/etc/dstack/client.conf` | optional QEMU path override | `vmm/src/config.rs:480-504` |
| `~/.dstack-vmm/image/`, `~/.dstack-vmm/vm/` | image and VM state on disk | `vmm/src/config.rs:506-541` (computed from `dirs::home_dir()`) |
| `$XDG_RUNTIME_DIR/dstack-vmm/` | discovery files for the CLI | `vmm/src/discovery.rs:16-23` |
| `./vmm.sock` | external API listener (Rocket Unix socket) | `vmm/vmm.toml:11` |
| `./run/supervisor.sock` | supervisor IPC | `vmm/vmm.toml:131-135` |
| `vsock:2` | host-api listener | `vmm/vmm.toml:138-141` |
| `/sys/bus/pci/devices/0000:{slot}/numa_node` | NUMA detection for hugepages | `vmm/src/app/qemu.rs:929-950` |
| `/sys/devices/system/node/node{N}/cpulist` | CPU pinning via taskset | `vmm/src/app/qemu.rs:952-964` |
| `/dev/tpmrm0` or `/dev/tpm0` | TPM key provider passthrough | `vmm/src/app/qemu.rs:535-547` |
| `/dev/hugepages` | NUMA-pinned memory backing | `vmm/src/app/qemu.rs:644` |
| `/dev/kvm` | implicit via QEMU `-accel kvm` | `vmm/src/app/qemu.rs:416` |
| `/proc/{pid}` | discovery liveness check | `vmm/src/discovery.rs:134` |
| `/sys/class/dmi/id/...` | SMBIOS values for cloud-env detection | `vmm/vmm.toml:55-79` (config) |
| `lspci` | enumerate GPUs & PCI devices | `vmm/src/main_service.rs:106-110`, `vmm/src/config.rs:262-285` |
| `ps aux` | one-shot mode CID discovery | `vmm/src/one_shot.rs:22-50` |
| `qemu-img` | create / resize disks | `vmm/src/main_service.rs:274`, `vmm/src/app/qemu.rs:117-135` |
| `qemu-system-x86_64` | the actual VMs | `vmm/src/config.rs:534-540` (`which::which`) |
| `taskset` | NUMA pinning wrapper | `vmm/src/app/qemu.rs:739-740` |
| `sudo -u {user}` | optional drop-privileges wrapper | `vmm/src/app/qemu.rs:743-748` |
| `iptables` | sandbox via `setup-user.sh` (operator-run) | `vmm/src/setup-user.sh:106-181` |
| TAP / bridge | `qemu -netdev bridge,br=...` for bridge mode | `vmm/src/app/qemu.rs:523-526` |

**(c) Assumptions vmm makes about the OS.**

- Linux. x86_64 (the default `which::which("qemu-system-{ARCH}")` is
  arch-portable, but everything else — KVM, vsock, TDX — is Linux/Intel
  only).
- KVM access (membership of group `kvm`, see `setup-user.sh:106`).
- vsock kernel support on the host.
- Hugepages mountpoint at `/dev/hugepages` if the user requests
  hugepages.
- TDX-capable host kernel + QEMU (only required for confidential VMs).
- For bridge networking: a pre-existing Linux bridge created by the
  operator (`virbr0` is the default).
- For port-forward (bridge mode): vmm runs **userspace** TCP/UDP
  forwarders — see `port-forward/src/lib.rs:40-89`. No `iptables`
  required for that path because forwarding happens at the
  application layer. No `CAP_NET_ADMIN` needed for the forwarders;
  ordinary listen sockets suffice if the chosen port is unprivileged.
  If the chosen listen port is <1024, listening will fail without
  `CAP_NET_BIND_SERVICE`.
- For DHCP-driven bridge mode: someone else (dnsmasq) calls
  `Vmm.ReportDhcpLease` so vmm can map MAC→IP. vmm does not run a
  DHCP server itself.
- For TPM-backed key provider: a TPM device exists at one of the two
  hard-coded paths above.
- For GPU passthrough: vfio + iommufd kernel support, devices already
  bound to vfio-pci by the operator.
- The "drop privileges" model is `sudo -u <user> qemu-system-x86_64
  ...` — implies the supervisor process has `sudo` rights for that
  user without a password. This requires sudoers config done by the
  operator (`setup-user.sh`).

**(d) Failure handling.** The host OS is treated as authoritative. If
KVM is missing, qemu-img is missing, hugepages aren't allocated, or
the bridge doesn't exist, vmm surfaces the underlying error to the
caller and continues to run (the auto-restart task at
`main.rs:132-146` keeps trying every 20s). There is no preflight
"is this host healthy" check.

**(e) Tightness rating: 5/5 — extremely tight, structurally tight.**

- The whole `qemu.rs` module (1138 LOC) builds raw argv strings
  embedding host paths directly. Replacing QEMU or adding a
  cross-host worker boundary means redesigning this module.
- Configuration is "the operator owns the host". Multi-tenant means
  hosts shared across orgs; the OS-level isolation here is
  **per-OS-user**, not per-tenant — set with a single `cvm.user`
  string, see `vmm/src/config.rs:170-171`.
- The discovery file scheme is per-Linux-user, not per-tenant
  (`discovery.rs:16-23`).
- All filesystem state is in `~/.dstack-vmm/` for the operator's
  account. There is no namespacing for orgs.

Refactor entails: introduce a "host worker" abstraction (the thing
that owns QEMU, supervisor, host_api on a given physical box), make
that worker per-host-tenancy or single-tenant only, and have a
control plane talk to one or many workers. That's a redesign, not a
swap.

---

## 6. QEMU

**(a) How the command line is constructed.** Entirely string
concatenation in `vmm/src/app/qemu.rs:388-771` (`VmConfig::config_qemu`).
~380 lines of `command.arg(...)` calls with conditionals on
networking mode, GPU passthrough, hugepages, TPM, host-share-mode,
TDX-vs-non-TEE, mrconfigid, vsock CID, NUMA, etc.

The resulting `ProcessConfig` (id, command, args, env, cwd, stdout,
stderr, pidfile, cid, note) is sent to the supervisor over its UDS,
which then `tokio::process::Command::spawn`s it. There is no QMP
control today (it's gated by `qmp_socket = false` in `vmm.toml:34`),
so once QEMU is up vmm can't change anything but disk-resize via
`qemu-img resize` (`main_service.rs:274-283`).

**(b) Data exchanged with QEMU.**

- argv (everything that matters)
- pty for serial console (`-chardev pty`, `qemu.rs:420-425`)
- optional QMP UDS for control (off by default)
- stdout / stderr files
- pidfile
- vsock CID (`-device vhost-vsock-pci,guest-cid={cid}`,
  `qemu.rs:550-552`)

**(c) Assumptions vmm makes about QEMU.**

- QEMU version is parsed at startup (`vmm/src/config.rs:18-75`) and
  passed to `make_vm_config` (`app.rs:1183-1191`) so the guest can do
  version-conditional things. The vmm itself does *not* fork on QEMU
  version — but several flags are passed through unchanged
  (`qemu_pic`, `qemu_single_pass_add_pages`, `qemu_pci_hole64_size`,
  `qemu_hotplug_off`).
- `q35` machine type (`qemu.rs:783`).
- `confidential-guest-support=tdx` for TEE mode (`qemu.rs:790`).
- `tdx-guest` object (`qemu.rs:850-858`) with mrconfigid + optional
  `quote-generation-socket` to vsock CID 2 / `qgs_port`.
- 9p / vvfat / vhd file-sharing schemes for the guest's
  `/dev/disk/by-label/host-shared` (`qemu.rs:554-599`). The label is
  literal (`HOST_SHARED_DISK_LABEL` from `dstack-types`, used at
  `qemu.rs:152, 574`).
- Image layout: `kernel`, `initrd`, optional `bios`, optional `hda`,
  optional `rootfs.iso` or `rootfs.verity` (`vmm/src/app/image.rs:55-92`).
- `iommufd` for GPU passthrough (`qemu.rs:657, 668, 682, 696`).

**(d) Failure handling.** Process exit is observed by supervisor
(`supervisor/src/process.rs:240-280`); vmm's auto-restart loop polls
the supervisor every 20s and re-issues `start_vm` for any VM whose
desired state is `started=true` but supervisor reports `not running`
(`main.rs:132-146`, `app.rs:1050-1078`). Disk-resize errors propagate
synchronously (`main_service.rs:278-284`).

**(e) Tightness rating: 5/5 — pervasive.**

Replacing QEMU (e.g. with Cloud Hypervisor or Firecracker), or
abstracting the runtime (e.g. for non-TDX dev VMs on macOS), means
rewriting `qemu.rs`. Every CVM-spec field eventually maps to a QEMU
flag here. There is no `enum HypervisorBackend` to extend.

---

## 7. dstack-supervisor

**(a) Protocol / transport.** HTTP-over-Unix-socket (or TCP, depending
on `base_url`), JSON bodies. Thin wrapper over reqwest in
`supervisor/client/src/lib.rs:90-113`. No streaming / no auth.

vmm spawns supervisor as a side process if it isn't already running
(`supervisor/client/src/lib.rs:26-88`, called from `vmm/src/main.rs:209-222`).
Path / pidfile / log live in `[supervisor]` (`vmm/vmm.toml:130-136`,
`vmm/src/config.rs:295-303`).

**(b) Data exchanged.**

| Endpoint | Method | Used in |
|---|---|---|
| `POST /deploy` | submit a `ProcessConfig` to be tracked | `app.rs:264-267` |
| `POST /start/{id}` | (not currently called from vmm — vmm calls `deploy` which auto-starts) | – |
| `POST /stop/{id}` | stop a tracked process | `app.rs:287-289`, `:329-331`, `main_service.rs:609-611` |
| `DELETE /remove/{id}` | drop tracked entry | `app.rs:350-352`, `main_service.rs:614-616` |
| `GET /list` | list tracked processes | `app.rs:545-547`, `:631`, `:826-832`, `:1052-1056` |
| `GET /info/{id}` | fetch one tracked process | `app.rs:233-235`, `:337`, `:891`, `main_service.rs:586` |
| `GET /ping` | startup probe | `supervisor/client/src/lib.rs:36-39, 80-83` |

`ProcessConfig` carries an opaque `note: String` field that vmm
serialises a `ProcessAnnotation{kind, live_for}` into
(`app.rs:140, 549-551`). This is how vmm distinguishes "this process
is a CVM I own" from "this is something else" during reload
(`app.rs:546-562`, `:631-645`).

**(c) Assumptions vmm makes about supervisor.**

- A **single** supervisor instance per vmm
  (`SupervisorClient` is built once at startup, `main.rs:209-222`,
  stored on `App.supervisor: SupervisorClient`,
  `vmm/src/app.rs:131`).
- Supervisor lives at a Unix socket on the local filesystem.
- Supervisor runs as the same OS user as vmm.
- `note: String` is opaque to supervisor (vmm puts JSON in there).
- `cid: Option<u32>` is preserved by supervisor — used as a sanity
  check at `app.rs:553-556` to recover the CID pool on reload.
- Process IDs are vmm-supplied (UUIDs) and globally unique within the
  supervisor instance.
- Auto-restart is **vmm's** responsibility (`main.rs:132-146`),
  supervisor doesn't restart on its own.

**(d) Failure handling.** Sync API has a 1s timeout
(`supervisor/client/src/lib.rs:185-191`). The async API has none
beyond reqwest defaults. On a missing socket, vmm tries to spawn the
supervisor binary in a thread (`supervisor/client/src/lib.rs:48-87`)
and probes 10× with backoff. Once running, vmm assumes it stays
running — there is no second-time auto-recovery.

**(e) Tightness rating: 4/5 — invasive but interface-shaped.**

- Used in many places (`supervisor.list`, `supervisor.info`,
  `supervisor.deploy`, `supervisor.stop`, `supervisor.remove` are
  scattered across `app.rs` and `main_service.rs`, ~15 call sites).
- But all behind one client object — `App.supervisor`. Replacing the
  process model (e.g. with kubelet, systemd, or a remote worker) is
  feasible because the API is small and well-defined: deploy, list,
  stop, remove, info.
- For a multi-tenant rewrite the natural shape is "one supervisor per
  worker host" and "the control plane proxies through". The hook
  point is `App::supervisor`, no proto changes needed.
- The bigger conceptual coupling: vmm relies on supervisor's
  in-process state for "is this VM running?". If supervisor restarts
  it tries to recover (the `reload_vms` flow at `app.rs:545-621`),
  which assumes it can poll the supervisor and reconcile. Multi-host
  needs that reconciliation to be transactional across the network.

---

## 8. The operator (human)

**(a) Protocol / transport.** Three surfaces:

1. **Web UI (Rocket)** at `address` from `vmm.toml` (default
   `unix:./vmm.sock`, often `0.0.0.0:9080` in production). Static
   HTML/JS in `vmm/src/console_v0.html` and the new
   `vmm/ui/...` Vue app, plus prpc endpoints under `/prpc`. Mount
   points in `vmm/src/main.rs:81-100`.
2. **prpc API**: exposed on the same port. Service definition
   `vmm/rpc/proto/vmm_rpc.proto:299-360`. The big surface — 25 RPC
   methods covering create/start/stop/remove, image management,
   resource queries, etc.
3. **vmm-cli.py**: thin wrapper that POSTs to `/prpc/{Method}?json`
   (`vmm/src/vmm-cli.py:489-499`). Discovers running vmm instances
   via the discovery files in `$XDG_RUNTIME_DIR/dstack-vmm/`
   (`vmm/src/vmm-cli.py:38-105`).

**(b) Data exchanged.** The full prpc surface (covered in detail in
`01-vmm-rpc-http-api-surface.md`).

**(c) Assumptions vmm makes about the operator.**

This is the load-bearing assumption for multi-tenant:

- **Authentication is shared bearer-token list.**
  `[auth] enabled = false, tokens = []` by default
  (`vmm/vmm.toml:127-129`, `vmm/src/config.rs:288-293`). Anyone with
  any token from the list has full operator privileges
  (`vmm/src/main.rs:208`, `main_routes.rs:109`). The token guard is
  applied to the log-streaming route (`vmm/src/main_routes.rs:107-116`)
  but the prpc routes mounted at `/prpc` rely on `rocket-apitoken`'s
  global mounting behaviour — there is no per-method or per-tenant
  authorisation.
- **One operator owns the whole host.** Discovery files live under
  `$XDG_RUNTIME_DIR/dstack-vmm/` (`discovery.rs:16-23`), one per Linux
  user. The CLI scans `/run/user/*/dstack-vmm/` to find every vmm
  instance on the box (`vmm-cli.py:38-59`) — explicitly cross-user.
- **No sessions, no audit log, no rate limiting.** Bearer token in,
  command executes. No record of who invoked what.
- **Config is a single file.** `vmm.toml`. Per-VM overrides are
  expressible (per-VM `kms_urls`, `gateway_urls`, networking mode), but
  the entire admin surface is a single Rocket app with one auth realm.
- **vmm-cli.py implicitly trusts the local user's filesystem** —
  reads `~/.dstack-vmm/config.json` (`vmm-cli.py:34`) for the saved
  URL/token, scans `/run/user` for running instances. No notion of
  org / project / RBAC.

**(d) Failure handling.** Token check fails ⇒ Rocket returns 401.
Beyond that, no rate-limit, no lockout, no replay protection.

**(e) Tightness rating: 4/5 — invasive in a way that is the entire
problem the rewrite is trying to solve.**

The auth model is *not* scattered across many files — it's actually
tightly localised to `main.rs:208`, `main_routes.rs:109`, and the
`rocket-apitoken` crate's mount semantics. So the *insertion point*
for replacing it (with sessions, RBAC, org scoping, audit) is small.

What is invasive is:

- Every prpc method on `RpcHandler` runs with full operator privilege
  (`vmm/src/main_service.rs:290-740`). To add per-method authorisation
  you have to plumb a `principal` through `CallContext` into every
  method.
- Every domain object (manifest, vm, image, app_id) is **un-namespaced**:
  `Manifest{id, name, app_id, ...}` (`app.rs:48-73`). Adding
  `org_id`/`owner_id` is a schema migration that touches the
  on-disk layout (`vm-manifest.json`), the prpc proto, and 25 RPC
  methods.
- The web UI is a single static HTML/JS bundle that has no concept
  of accounts (`vmm/src/console_v0.html` and `vmm/ui/`).

This is the file that determines the rewrite shape. See report
`06-auth-authz-multi-tenant-gaps.md` for the deep dive.

---

## 9. Other adjacent systems (lower coupling)

Brief notes for completeness — these don't dominate the design but
they do exist.

### 9a. Local key-provider service (`[key_provider]`)

Length-prefixed JSON over a TCP loopback connection at
`127.0.0.1:3443` by default (`vmm.toml:143-146`). Code:
`key-provider-client/src/host.rs:24-57`. Used only by `host_api_service::get_sealing_key` (`vmm/src/host_api_service.rs:49-62`)
when the guest-agent asks for a sealing key.

- Single endpoint config (`KeyProviderConfig{address, port}`,
  `config.rs:469-474`). Single function call site.
- No TLS, no auth. Trust = "this is on loopback".
- Tightness 3/5: bespoke wire format means swap requires reworking the
  protocol; but only one call site, so a clean adapter trait would
  isolate it.

### 9b. OCI image registry

reqwest HTTPS, hand-rolled OCI-distribution v2 client at
`vmm/src/app/registry.rs:1-473`. Used for `ListRegistryImages` and
`PullRegistryImage` prpc methods. Single registry per vmm
(`vmm.toml:18-21`, `[image] registry`). Token auth supported.

- Tightness 1/5: self-contained module with a small surface
  (`list_registry_tags`, `pull_and_extract`).

### 9c. DHCP server (inbound on vmm's prpc)

`Vmm.ReportDhcpLease(mac, ip)` (`vmm_rpc.proto:343-345, 363-368`).
Called by an external DHCP server (e.g. `dnsmasq --dhcp-script`).
Handler at `main_service.rs:579-582` → `app.rs:423-447`. No auth
beyond the global bearer token, so the operator is responsible for
exposing this path only to trusted callers.

- Tightness 1/5: one method, one handler, isolated.

### 9d. Self-discovery

`vmm/src/discovery.rs` writes a JSON file into `$XDG_RUNTIME_DIR/dstack-vmm/`
on startup, removes on drop. Read by `vmm-cli.py` and other vmms.

- Tightness 1/5: own crate-private module, ~145 LOC, easily replaced
  with a registry call in a multi-host design.

---

## 10. Coupling heatmap (sorted)

Sorted by tightness (highest first). "Refactor entails" assumes the
rewrite goal of multi-tenant control plane on top of one or many
vmm-style hosts.

| Rank | System | Score | Where it lives | Refactor entails |
|------|--------|-------|----------------|------------------|
| 1 | Host OS | 5 | `app/qemu.rs:388-771`, `config.rs:478-559`, scattered `/sys`, `/dev`, `/proc`, `/etc/dstack` reads | Treat the whole vmm process as a per-host **worker**. Build the new control plane *above* it (talks to one or many workers). The worker has to keep most of the host-OS coupling; the control plane can be host-OS-agnostic. |
| 2 | QEMU | 5 | `app/qemu.rs` (~1100 LOC), `qemu-img` shells in `main_service.rs:274` | Fence off behind a `HypervisorBackend` trait if you ever want non-QEMU support. Otherwise, leave this in the worker and don't try to abstract; the leaks (mrconfigid, TDX object, vsock CID) are too tied to the QEMU + TDX model. |
| 3 | Supervisor | 4 | `App.supervisor` field, ~15 call sites in `app.rs` and `main_service.rs` | Push behind a `ProcessRuntime` trait — `deploy / list / info / stop / remove`. One supervisor per worker host. The control plane never sees supervisor directly. Keep `note: String` as the JSON-blob escape hatch but stop treating it as semi-structured. |
| 4 | Operator (auth + identity) | 4 | `main.rs:208`, `main_routes.rs:109`, every method on `RpcHandler` | This is the rewrite. Add `Principal` to `CallContext`. Add `owner_id`/`org_id` to `Manifest`. Replace `ApiToken` with sessions + a tenant directory. Wrap every existing prpc method in policy enforcement. Web UI becomes a multi-tenant SPA. |
| 5 | guest-agent (host_api inbound, vsock peer = identity) | 3 | `host_api_service.rs:21-33`, event handler `app.rs:902-946` | Re-route `Notify` events through a tenant-aware sink so org-scoped audit/observability is possible. Keep CID-as-VM-identity inside the worker; tag events with `owner_id` before they leave the worker. |
| 6 | Local key-provider | 3 | `key-provider-client/src/host.rs:24-57`, single call from `host_api_service.rs:49-62` | Add an adapter trait so the key provider can be local TCP, vsock, or remote-with-mTLS. Cheap. |
| 7 | guest-agent (proxied API outbound) | 2 | `app.rs:1016-1021`, used in `guest_api_service.rs` and `main_service.rs:509` | Two functions; replace with a `GuestAgentClient` resolved from the worker. Cheap. |
| 8 | dstack-kms | 2 | `app.rs:1007-1014`, `main_service.rs:455-468` | One method, one call site. Hide behind `EnvEncryptionPubkeySource` trait. **However**, the *tenant-to-KMS* mapping is the policy decision: do we have one KMS per fleet, one per org, or one per app? That decision lives in the control plane, not in vmm. |
| 9 | DHCP server (inbound) | 1 | `main_service.rs:579-582`, `app.rs:423-447` | Move auth in front of `ReportDhcpLease` (today it's the same global bearer token). Otherwise unchanged. |
| 10 | OCI registry | 1 | `app/registry.rs` (self-contained) | Already isolated. Adding per-tenant registries is a config / data-model change, not code restructuring. |
| 11 | dstack-gateway | 1 (code) / 4 (semantic) | `app.rs:1148-1152`, `app/qemu.rs:276-303`, `config.rs:305-310` | Code change is one-liner: parameterise the ingress URL template. The hard work — multi-tenant DNS / cert flow / WireGuard peer scoping — is in dstack-gateway itself, out of scope for vmm. |
| 12 | On-chain (Ethereum) | 0 | nowhere | Already insulated. Multi-tenant org-to-EOA mapping is a KMS/control-plane decision. vmm continues not to care. |

---

## 11. Implications for the rewrite

Pulling these threads together:

1. **The rewrite is mostly an auth / identity / namespacing rewrite,
   not a transport rewrite.** Items #1, #2, #5, #7, #11 in the
   heatmap are stable contracts that already have well-defined
   surfaces. The thing that has to change is *who is allowed to do
   what to which CVMs*, which is item #4.

2. **vmm naturally factors into "control plane" + "host worker".**
   The host worker is what owns QEMU, supervisor, host_api, GPUs,
   bridges, the on-disk image cache, the CID pool. Its API surface
   is roughly today's prpc surface minus auth. The control plane is
   what owns auth, orgs, CVM ownership, scheduling onto workers,
   audit, multi-host discovery. Single-host self-hosted deployments
   can run both in one process; SaaS deployments would split them.

3. **The `app_id` namespace is shared with KMS and on-chain
   governance and cannot be made tenant-local without redesigning
   KMS auth.** Either the control plane custodies on-chain
   identities for orgs (org → wallet → app_id), or each user keeps
   their own wallet and the org is purely off-chain bookkeeping.
   This is one of the open framing questions in `00-overview.md`.

4. **Gateway ingress patterns are a separate work-stream.** vmm only
   passes URLs to guests; the actual change to support
   `org.example.com/app/...` style routing is in dstack-gateway and
   the guest agent's registration code. From vmm's side this is one
   format string in `app/qemu.rs:288-302`.

5. **There is no chain dependency to break.** The only thing that
   leaks the chain into vmm is the `app_id` format. Every other
   chain interaction is either in KMS or in the human-facing
   vmm-cli — both can evolve independently.

6. **Discovery is per-Linux-user today.** A multi-tenant control
   plane wants something different (per-org, per-host, federated).
   The discovery scheme in `discovery.rs` is small enough to throw
   out and replace.
