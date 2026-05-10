# dstack-vmm RPC/HTTP API Surface Reference

**Prepared:** 2026-05-05  
**Scope:** Exhaustive inventory of dstack-vmm APIs for multi-tenant rewrite planning  
**Coverage:** Proto definitions, service implementations, HTTP routes, listen addresses, and tenancy assumptions

---

## Executive Summary

dstack-vmm currently exposes **three distinct RPC surfaces** plus **HTTP static/streaming routes**, with **no multi-tenancy isolation** at the API level. The architecture assumes single-operator semantics:

1. **External API** (default): TCP/Unix socket, serves web UI, public prpc endpoints, and log streaming
2. **Host API** (vsock-only): Guest-to-host integration for sealing keys and notifications  
3. **Guest API** (proxied): Allows external code to query guest state (system info, containers, network)

**Key findings for multi-tenant rewrite:**
- All VM lists are **global** (no caller identity passed in requests)
- Port configuration, image paths, GPU allocation are **global settings**, not per-tenant
- Configuration stored in single `vmm.toml` file shared across all operators
- **No authentication token** included in prpc requests (only HTTP header-level auth)
- KMS URLs, gateway URLs, resource limits are **global** settings returned by `GetMeta`
- Host API uses **vsock CID as implicit caller identity** (tied to VM, not operator tenant)

---

## Bind Addresses and Ports

### External API (Web UI + Public RPC)

| Component | Protocol | Address | Port | Notes |
|-----------|----------|---------|------|-------|
| **External API** | Unix socket OR TCP | Configurable via `address` config | Configurable via `port` config | Default: `unix:./vmm.sock` |
| | TCP (if configured) | 0.0.0.0 (binds all interfaces) | 9080 (typical) | Rocket default; overridable |

**Source:** `vmm/src/main.rs:77–109`, `vmm/src/config.rs:448–451`, `vmm/vmm.toml:11–12`

### Host API (Guest ↔ Host, vsock-only)

| Component | Protocol | Address | Port | Notes |
|-----------|----------|---------|------|-------|
| **Host API** | vsock | `vsock:2` (fixed in config) | 10000 (configurable) | **MUST be vsock** per validation |

**Source:** `vmm/src/main.rs:111–130`, `vmm/src/config.rs:448–467`, `vmm/vmm.toml:138–141`

**Security note:** Host API binds only to vsock for security. TCP/Unix socket explicitly forbidden by validation at startup (line 454–465 in config.rs).

---

## HTTP Route Handlers (External API)

### Static UI & Streaming Routes

**File:** `vmm/src/main_routes.rs:45–187`

| Endpoint | Method | Handler | Purpose | Auth Required |
|----------|--------|---------|---------|---|
| `/` | GET | `index()` | Serve web console v1 (default) | No |
| `/v1` | GET | `v1()` | Alias for `/` (console v1) | No |
| `/beta` | GET | `beta()` | Alias for `/` (future beta UI) | No |
| `/v0` | GET | `v0()` | Legacy console v0 (file-backed) | No |
| `/res/x25519.js` | GET | `res()` | Cryptography library for web UI | No |
| `/res/<other>` | GET | `res()` | Wildcard (returns 404 if not x25519.js) | No |
| `/logs` | GET | `vm_logs()` | **Streaming**: VM logs (serial/stdout/stderr) | **YES** (Authorized) |

**Streaming endpoint query params:**
- `id`: VM ID (required)
- `follow`: Boolean, tail-f mode (default false)
- `ansi`: Boolean, preserve ANSI colors (default false)
- `lines`: Integer, number of lines to return (default 10000)
- `ch`: Channel name: `serial`, `stdout`, or `stderr` (default `serial`)

**Source:** `vmm/src/main_routes.rs:107–183` for logs endpoint; lines 45–74 for static routes

**Tenancy issue:** The `/logs` endpoint streams a single VM by global ID with no tenant isolation.

---

## pRPC Service Routes (HTTP POST endpoints)

All pRPC methods are auto-generated via the `ra_rpc::prpc_routes!` macro. Endpoints are **HTTP POST** with request/response bodies in protobuf or JSON.

### Mount Points and Services

**File:** `vmm/src/main.rs:82–88`

| Mount Path | Service Handler | Proto Namespace | Trim Prefix | Purpose |
|-----------|-----------------|-----------------|-----------|---------|
| `/guest` | `GuestApiHandler` | `guest_api::` | (none) | Proxied guest agent queries |
| `/api` | `HostApiHandler` | `host_api::` | (none) | Host-to-guest integration (vsock) |
| `/prpc` | `RpcHandler` | `dstack_vmm_rpc::` (vmm) | `Teepod.` | Main VMM lifecycle & metadata |

OpenAPI docs served at `/api-docs` (file: `vmm/src/openapi.rs:11–35`).

---

## pRPC Service: Vmm (Main Lifecycle)

**Source:** `vmm/rpc/proto/vmm_rpc.proto:300–360` (proto definition)  
**Implementation:** `vmm/src/main_service.rs:290–752` (RpcHandler impl VmmRpc)

### CVM Lifecycle Methods

#### CreateVm
- **Proto:** `rpc CreateVm(VmConfiguration) returns (Id)`
- **URL:** `POST /prpc/Vmm.CreateVm`
- **Handler:** `RpcHandler::create_vm()` (main_service.rs:291–327)
- **Request fields:**
  - `name`: VM name (validated for safe chars)
  - `image`: Image name to use
  - `compose_file`: JSON app compose spec
  - `vcpu`, `memory`, `disk_size`: Resource allocation
  - `ports`: Port mappings (array of PortMapping)
  - `encrypted_env`: Encrypted environment variables (bytes)
  - `app_id`: Optional app ID (KMS-backed)
  - `user_config`: Config string for guest at `/dstack/.user-config`
  - `hugepages`, `pin_numa`: Memory optimizations
  - `gpus`: GPU config (mode: listed/all, slots)
  - `kms_urls`, `gateway_urls`: Service URLs
  - `stopped`: If true, create but don't start
  - `no_tee`: Disable TEE mode
  - `networking`: Per-VM networking override
- **Returns:** `Id` with generated UUID
- **Multi-tenancy issue:** No tenant context; VMs are global.

#### StartVm
- **Proto:** `rpc StartVm(Id) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.StartVm`
- **Handler:** `RpcHandler::start_vm()` (main_service.rs:329–335)
- **Request:** VM ID (string)

#### StopVm
- **Proto:** `rpc StopVm(Id) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.StopVm`
- **Handler:** `RpcHandler::stop_vm()` (main_service.rs:337–343)
- **Request:** VM ID (string)

#### RemoveVm
- **Proto:** `rpc RemoveVm(Id) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.RemoveVm`
- **Handler:** `RpcHandler::remove_vm()` (main_service.rs:345–351)
- **Request:** VM ID (string)
- **Note:** Deletes all disk/runtime state.

#### ShutdownVm
- **Proto:** `rpc ShutdownVm(Id) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.ShutdownVm`
- **Handler:** `RpcHandler::shutdown_vm()` (main_service.rs:509–512)
- **Request:** VM ID (string)
- **Note:** Graceful shutdown via guest agent (not force-kill).

---

### Configuration & Update Methods

#### UpdateVm
- **Proto:** `rpc UpdateVm(UpdateVmRequest) returns (Id)`
- **URL:** `POST /prpc/Vmm.UpdateVm`
- **Handler:** `RpcHandler::update_vm()` (main_service.rs:377–453)
- **Request fields:**
  - `id`: VM ID to update
  - `compose_file`: New compose (optional; empty = no change)
  - `encrypted_env`: New encrypted env (optional)
  - `user_config`: New config string (optional)
  - `update_ports`, `ports`: If update_ports=true, replace port mappings
  - `update_kms_urls`, `kms_urls`: Replace KMS URLs
  - `update_gateway_urls`, `gateway_urls`: Replace gateway URLs
  - `gpus`: Update GPU config
  - `vcpu`, `memory`, `disk_size`, `image`: Resource updates (only if VM stopped)
  - `no_tee`: Toggle TEE mode
- **Returns:** `Id` (with possibly new app_id based on compose hash)
- **Notes:** 
  - VM must be stopped for resource/image changes
  - Updates manifest on disk and reloads in supervisor

#### UpgradeApp (deprecated, alias for UpdateVm)
- **Proto:** `rpc UpgradeApp(UpdateVmRequest) returns (Id)`
- **URL:** `POST /prpc/Vmm.UpgradeApp`
- **Handler:** Calls `update_vm()` (line 373–375)

#### ResizeVm (deprecated, use UpdateVm)
- **Proto:** `rpc ResizeVm(ResizeVmRequest) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.ResizeVm`
- **Handler:** `RpcHandler::resize_vm()` (main_service.rs:484–507)
- **Request fields:**
  - `id`, `vcpu?`, `memory?`, `disk_size?`, `image?`
- **Notes:** 
  - Performs disk resize via `qemu-img resize` if `disk_size` > current
  - Cannot shrink disk
  - VM must be stopped

---

### Image & Registry Management

#### ListImages
- **Proto:** `rpc ListImages(google.protobuf.Empty) returns (ImageListResponse)`
- **URL:** `POST /prpc/Vmm.ListImages`
- **Handler:** `RpcHandler::list_images()` (main_service.rs:357–371)
- **Returns:** Array of ImageInfo with name, description, version, is_dev flag
- **Source:** Local image directory (config.image.path)

#### ListRegistryImages
- **Proto:** `rpc ListRegistryImages(google.protobuf.Empty) returns (RegistryImageListResponse)`
- **URL:** `POST /prpc/Vmm.ListRegistryImages`
- **Handler:** `RpcHandler::list_registry_images()` (main_service.rs:619–662)
- **Returns:** Array of RegistryImageInfo with tag, local, pulling, error status
- **Notes:**
  - Requires registry URL configured in `config.image.registry`
  - Filters out sha256-* hash tags
  - Marks images that are locally cached

#### PullRegistryImage
- **Proto:** `rpc PullRegistryImage(PullRegistryImageRequest) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.PullRegistryImage`
- **Handler:** `RpcHandler::pull_registry_image()` (main_service.rs:697–740)
- **Request fields:**
  - `tag`: Image tag to pull (e.g., "0.5.8" or "nvidia-0.5.8")
- **Notes:**
  - Spawns background task (non-blocking)
  - Updates pull_status map to track Pulling/Failed state
  - Downloads and extracts OCI image tarball to local storage

#### DeleteImage
- **Proto:** `rpc DeleteImage(Id) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.DeleteImage`
- **Handler:** `RpcHandler::delete_image()` (main_service.rs:664–695)
- **Request:** Image name (string)
- **Notes:**
  - Validates image name (no path traversal, no slashes)
  - Checks that no running VM uses this image
  - Deletes image directory from disk

---

### Metadata & System Methods

#### GetMeta
- **Proto:** `rpc GetMeta(google.protobuf.Empty) returns (GetMetaResponse)`
- **URL:** `POST /prpc/Vmm.GetMeta`
- **Handler:** `RpcHandler::get_meta()` (main_service.rs:521–554)
- **Returns:**
  - `kms`: URL(s) configured in `config.cvm.kms_urls`
  - `gateway`: Base domain, port, agent_port, URLs from `config.gateway`
  - `resources`: Max CVM count, max allocable vCPU, max allocable memory (from `config.cvm`)
- **Multi-tenancy issue:** Returns global resource limits; no per-tenant quotas.

#### GetMeta (alias of GetMeta)
- **Proto:** `rpc Version(google.protobuf.Empty) returns (VersionResponse)`
- **URL:** `POST /prpc/Vmm.Version`
- **Handler:** `RpcHandler::version()` (main_service.rs:514–519)
- **Returns:** Semantic version and git revision of vmm binary

#### ListGpus
- **Proto:** `rpc ListGpus(google.protobuf.Empty) returns (ListGpusResponse)`
- **URL:** `POST /prpc/Vmm.ListGpus`
- **Handler:** `RpcHandler::list_gpus()` (main_service.rs:556–563)
- **Returns:**
  - Array of GpuInfo (slot, product_id, description, is_free)
  - allow_attach_all flag from config
- **Notes:** Scans PCI bus for NVIDIA devices (vendor ID 10de) and bridges

---

### VM Querying Methods

#### Status (List VMs)
- **Proto:** `rpc Status(StatusRequest) returns (StatusResponse)`
- **URL:** `POST /prpc/Vmm.Status`
- **Handler:** `RpcHandler::status()` (main_service.rs:353–355)
- **Request fields:**
  - `ids[]`: Optional list of specific VM IDs (if empty, list all)
  - `brief`: If true, omit full VmConfiguration
  - `keyword`: Filter by name/ID substring
  - `page`, `page_size`: Pagination (for web UI)
- **Returns:**
  - `vms[]`: Array of VmInfo (see below)
  - `port_mapping_enabled`: Boolean
  - `total`: Total number of VMs (before pagination)
- **Multi-tenancy issue:** Global list of all VMs; no caller identity filtering.

**VmInfo structure** (vmm_rpc.proto:11–40):
- `id`, `name`, `status`: Identifiers and state
- `uptime`: Human-readable uptime string
- `app_url`: URL to guest agent dashboard (if running)
- `app_id`, `instance_id`: App identifiers
- `configuration`: Full VmConfiguration object
- `exited_at`: Timestamp if exited
- `boot_progress`, `boot_error`: Diagnostic strings
- `shutdown_progress`: Graceful shutdown state
- `image_version`: Version of base image
- `events[]`: Array of GuestEvent (log entries)

#### GetInfo
- **Proto:** `rpc GetInfo(Id) returns (GetInfoResponse)`
- **URL:** `POST /prpc/Vmm.GetInfo`
- **Handler:** `RpcHandler::get_info()` (main_service.rs:470–482)
- **Request:** VM ID (string)
- **Returns:**
  - `found`: Boolean (true if VM exists)
  - `info?`: Optional VmInfo (populated if found=true)

#### GetAppEnvEncryptPubKey
- **Proto:** `rpc GetAppEnvEncryptPubKey(AppId) returns (PublicKeyResponse)`
- **URL:** `POST /prpc/Vmm.GetAppEnvEncryptPubKey`
- **Handler:** `RpcHandler::get_app_env_encrypt_pub_key()` (main_service.rs:455–468)
- **Request:** `app_id` (bytes)
- **Returns:**
  - `public_key`: KMS-backed public key for env encryption
  - `signature`: Legacy signature without timestamp
  - `timestamp`: Unix timestamp (seconds)
  - `signature_v1`: Timestamped signature to prevent replay
- **Notes:** Forwards request to KMS RPC client; used during CreateVm to encrypt env vars

#### GetComposeHash
- **Proto:** `rpc GetComposeHash(VmConfiguration) returns (ComposeHash)`
- **URL:** `POST /prpc/Vmm.GetComposeHash`
- **Handler:** `RpcHandler::get_compose_hash()` (main_service.rs:565–572)
- **Request:** Full VmConfiguration
- **Returns:** SHA256 hex hash of compose_file (used as default app_id)
- **Note:** Debugging/development aid; validates compose JSON first

---

### Supervisor Management

#### SvList
- **Proto:** `rpc SvList(google.protobuf.Empty) returns (SvListResponse)`
- **URL:** `POST /prpc/Vmm.SvList`
- **Handler:** `RpcHandler::sv_list()` (main_service.rs:584–607)
- **Returns:** Array of SvProcessInfo
  - `id`, `name`: Process identifiers
  - `status`: "running", "stopped", "exited(<code>)", or "error(<msg>)"
  - `pid?`: Process ID (if running)
  - `command`, `note`: Supervisor metadata
- **Note:** Calls supervisor RPC client (supervisor_client::SupervisorClient)

#### SvStop
- **Proto:** `rpc SvStop(Id) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.SvStop`
- **Handler:** `RpcHandler::sv_stop()` (main_service.rs:609–612)
- **Request:** Process ID (string)

#### SvRemove
- **Proto:** `rpc SvRemove(Id) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.SvRemove`
- **Handler:** `RpcHandler::sv_remove()` (main_service.rs:614–617)
- **Request:** Process ID (string)
- **Note:** Removes stopped supervisor process

---

### VM Lifecycle Management

#### ReloadVms
- **Proto:** `rpc ReloadVms(google.protobuf.Empty) returns (ReloadVmsResponse)`
- **URL:** `POST /prpc/Vmm.ReloadVms`
- **Handler:** `RpcHandler::reload_vms()` (main_service.rs:574–577)
- **Returns:**
  - `loaded`: Number of VMs loaded from disk
  - `updated`: Number of VMs updated
  - `removed`: Number of VMs removed
- **Note:** Syncs in-memory VM state with disk; used during startup or recovery

#### ReportDhcpLease
- **Proto:** `rpc ReportDhcpLease(DhcpLeaseRequest) returns (google.protobuf.Empty)`
- **URL:** `POST /prpc/Vmm.ReportDhcpLease`
- **Handler:** `RpcHandler::report_dhcp_lease()` (main_service.rs:579–582)
- **Request fields:**
  - `mac`: MAC address of guest NIC (e.g., "02:ab:cd:ef:01:23")
  - `ip`: IPv4 address assigned by DHCP
- **Note:** Called by host DHCP server (dnsmasq) via --dhcp-script hook; triggers port forwarding reconfiguration

---

## pRPC Service: ProxiedGuestApi (Guest State Queries)

**Source:** `guest-api/proto/guest_api.proto:147–153` (proto definition)  
**Implementation:** `vmm/src/guest_api_service.rs:36–58` (GuestApiHandler impl ProxiedGuestApiRpc)

All methods forward requests to the in-guest agent via a gRPC client connected to the guest's vsock port.

| Method | Proto | URL | Handler | Purpose |
|--------|-------|-----|---------|---------|
| Info | `rpc Info(Id) returns (GuestInfo)` | POST /guest/ProxiedGuestApi.Info | line:37–39 | Fetch guest attestation & registration data |
| SysInfo | `rpc SysInfo(Id) returns (SystemInfo)` | POST /guest/ProxiedGuestApi.SysInfo | line:41–43 | Fetch OS/kernel/resource metrics |
| NetworkInfo | `rpc NetworkInfo(Id) returns (NetworkInformation)` | POST /guest/ProxiedGuestApi.NetworkInfo | line:45–47 | Dump NIC/gateway config |
| ListContainers | `rpc ListContainers(Id) returns (ListContainersResponse)` | POST /guest/ProxiedGuestApi.ListContainers | line:49–53 | Enumerate running containers |
| Shutdown | `rpc Shutdown(Id) returns (google.protobuf.Empty)` | POST /guest/ProxiedGuestApi.Shutdown | line:55–57 | Graceful guest shutdown |

**Request:** All take `Id` (VM ID string).

**Multi-tenancy issue:** Each method proxies to guest by VM ID; no tenant-level filtering.

---

## pRPC Service: HostApi (Host ↔ Guest Integration)

**Source:** `host-api/proto/host_api.proto:30–34` (proto definition)  
**Implementation:** `vmm/src/host_api_service.rs:35–63` (HostApiHandler impl HostApiRpc)  
**Transport:** vsock-only (no TCP/HTTP)

| Method | Proto | Handler | Purpose |
|--------|-------|---------|---------|
| Info | `rpc Info(google.protobuf.Empty) returns (HostInfo)` | line:36–42 | Report host ("dstack VMM") name and version |
| Notify | `rpc Notify(Notification) returns (google.protobuf.Empty)` | line:44–47 | Guest reports event to host; routed to event buffer |
| GetSealingKey | `rpc GetSealingKey(GetSealingKeyRequest) returns (GetSealingKeyResponse)` | line:49–62 | Retrieve sealing key from key provider (TSM integration) |

**Transport details (vmm/src/main.rs:111–130):**
- Host API runs in separate Rocket instance
- Binds to vsock address (configured in host_api.address, default "vsock:2")
- Uses RocketVsockListener
- Validates at startup (config.rs:454–465) that address MUST start with "vsock:"

**Tenancy issue:** Endpoint identifies caller via vsock CID (implicit from connection), which maps to a specific VM; no multi-tenant logic.

---

## Configuration Parameters (Global, Single-Tenancy)

**File:** `vmm/vmm.toml` (defaults), loaded via Rocket Figment

### Listen/Transport
- `address`: Unix socket OR TCP address (default: `unix:./vmm.sock`)
- `port`: TCP port if address is TCP (Rocket default: 8000)
- `workers`: Async worker thread pool size (default: 8)

### Authentication
- `auth.enabled`: Boolean (default: false)
- `auth.tokens`: Array of API tokens (header-based auth, not in prpc payload)

### Host API (vsock-only)
- `host_api.address`: MUST start with "vsock:" (e.g., "vsock:2")
- `host_api.port`: vsock port (default: 10000)

### CVM Resources (global caps)
- `cvm.max_allocable_vcpu`: Global vCPU cap (default: 20)
- `cvm.max_allocable_memory_in_mb`: Global memory cap (default: 100,000)
- `cvm.cid_pool_size`: Number of available vCPU context IDs (default: 1000)
- `cvm.cid_start`: Starting CID (default: 1000)

### KMS & Gateway (global endpoints)
- `cvm.kms_urls`: Array of KMS server URLs (default: `["http://127.0.0.1:8081"]`)
- `cvm.gateway_urls`: Array of gateway server URLs (default: `["http://127.0.0.1:8082"]`)
- `cvm.pccs_url`: PCCS server URL for quote verification (empty by default)

### Image Management (global paths)
- `image.path`: Directory where guest images stored locally (default: `~/.dstack-vmm/image`)
- `image.registry`: OCI registry URL for pulling images (empty by default, disables PullRegistryImage)

### Network (global per mode)
- `cvm.networking.mode`: "user", "bridge", or "custom" (default: "user")
- `cvm.networking.bridge`: Bridge interface name (for bridge mode)
- `cvm.networking.net`: CIDR (for user mode, default: "10.0.2.0/24")
- `cvm.networking.dhcp_start`: DHCP pool start IP (default: "10.0.2.10")
- `cvm.networking.forward_service_enabled`: Enable userspace port forwarding (default: false)

### Port Mapping (global policy)
- `cvm.port_mapping.enabled`: Boolean (default: false)
- `cvm.port_mapping.address`: Bind address (default: "127.0.0.1")
- `cvm.port_mapping.range[]`: Allowed port ranges (protocol, from, to)

### GPU (global discovery & policy)
- `cvm.gpu.enabled`: Boolean (default: false)
- `cvm.gpu.listing`: Array of product IDs to include (e.g., "10de:2335" for H200)
- `cvm.gpu.exclude`: Array of PCI slots to exclude
- `cvm.gpu.include`: Array of PCI slots to include (if non-empty, only these)
- `cvm.gpu.allow_attach_all`: Allow "attach all GPUs" mode (default: true)

### Gateway Config (global)
- `gateway.base_domain`: Base domain for VMs (default: "localhost")
- `gateway.port`: Gateway port (default: 8082)
- `gateway.agent_port`: Agent-facing port (default: 8090)

### QEMU & Supervisor (process-level)
- `cvm.qemu_path`: Path to QEMU binary
- `cvm.user`: User to run VMs as (empty = current user)
- `supervisor.exe`: Supervisor binary path
- `supervisor.sock`: Supervisor IPC socket path
- `supervisor.auto_start`: Auto-start supervisor on VMM startup

### Key Provider (for TEE sealing keys)
- `key_provider.enabled`: Boolean (default: true)
- `key_provider.address`: Key provider IP (default: "127.0.0.1")
- `key_provider.port`: Key provider port (default: 3443)

**Multi-tenancy issue:** No per-tenant config; all VMs share global settings.

---

## Single-Tenancy Assumptions in Current API

The following design patterns **assume single operator/tenant** and will require refactoring for multi-tenancy:

1. **Global VM list without tenant filtering**
   - `Status(StatusRequest)` returns all VMs; no caller identity in request
   - No `caller_tenant_id` or similar field in any proto message

2. **Global resource limits returned to all callers**
   - `GetMeta()` returns same limits to every caller
   - No per-tenant quotas or overrides
   - File: main_service.rs:521–554

3. **Global configuration without namespacing**
   - Single `vmm.toml` shared across all VMs
   - KMS URLs, gateway URLs, image paths apply to all VMs
   - No per-tenant config sections

4. **Image paths and registry configuration are global**
   - `config.image.path` is single directory for all VMs
   - `config.image.registry` is single registry for all pulls
   - File: main_service.rs:619–740 (image management)

5. **Port mapping policy is global**
   - Single allowed port range for all VMs
   - Single bind address for all forwarded ports
   - File: config.rs:110–143, main_service.rs:141–167 (validation in CreateVm)

6. **GPU allocation has no tenant isolation**
   - `ListGpus()` shows all available GPUs to all callers
   - No tracking of GPU assignment per tenant
   - File: main_service.rs:556–563

7. **Host API identifies caller only by vsock CID**
   - CID maps to a specific VM (guest), not a tenant operator
   - `host_api_service.rs:24–27` extracts CID from remote_endpoint
   - No multi-tenant logic in Notify or GetSealingKey

8. **Authentication is header-only, not in proto**
   - `rocket_apitoken::Authorized` middleware checks auth (line 109 in main_routes.rs)
   - No `auth_token` or `tenant_id` field in RPC request messages
   - Auth state is implicit; can't be forwarded to guest agent

9. **VM names are not namespaced**
   - `validate_label()` checks for safe chars only; no tenant prefix enforcement
   - File: main_service.rs:57–70

10. **Event reporting has no tenant context**
    - `ReportDhcpLease()` identifies VM by MAC address only
    - Guest events buffered globally in `app.event_buffer_size`
    - No per-tenant event filtering or routing

---

## API Maturity & Deprecation Status

| RPC Method | Status | Notes |
|-----------|--------|-------|
| CreateVm | ✓ Stable | Fully implemented; used in production |
| StartVm | ✓ Stable | Standard operation |
| StopVm | ✓ Stable | Standard operation |
| RemoveVm | ✓ Stable | Standard operation |
| UpdateVm | ✓ Stable | Replaces UpgradeApp and ResizeVm |
| UpgradeApp | ⚠ Deprecated | Alias for UpdateVm (line 373–375) |
| ResizeVm | ⚠ Deprecated | Alias for UpdateVm; no direct use |
| ShutdownVm | ✓ Stable | Graceful guest shutdown |
| GetInfo | ✓ Stable | Single VM query |
| Status | ✓ Stable | Fleet status & filtering |
| ListImages | ✓ Stable | Local images |
| ListRegistryImages | ✓ Stable | Registry images with pull status |
| PullRegistryImage | ✓ Stable | Async background pull |
| DeleteImage | ✓ Stable | Image deletion with use-check |
| GetMeta | ✓ Stable | System metadata |
| Version | ✓ Stable | Build info |
| ListGpus | ✓ Stable | GPU inventory |
| GetAppEnvEncryptPubKey | ✓ Stable | KMS integration |
| GetComposeHash | ✓ Stable | Debugging aid |
| ReloadVms | ✓ Stable | Startup/recovery |
| ReportDhcpLease | ✓ Stable | DHCP integration |
| SvList, SvStop, SvRemove | ✓ Stable | Supervisor process control |

---

## Web UI Assets & JavaScript

**Files:**
- `vmm/src/main_routes.rs`: Routes (index, v0, v1, beta, res, vm_logs)
- `vmm/console_v1.html`: Embedded (via OUT_DIR); default landing page
- `vmm/console_v0.html`: Legacy version (fallback if file not found)
- `vmm/x25519.js`: Cryptography library for key exchange (bundled in build)

**Build:** Assets embedded at compile time. Served as static content.

---

## Error Handling & Status Codes

### pRPC Error Format

On error, all pRPC endpoints return HTTP 400+ with a pRPC error frame (defined in `vmm/rpc/proto/prpc.proto:9–12`):

```protobuf
message PrpcError {
  string message = 1;
}
```

**Source:** prpc.proto (framework-level); handled by ra_rpc library (not in vmm code).

### HTTP Status Codes

| Code | Scenario |
|------|----------|
| 200 OK | Successful pRPC or log stream |
| 400 Bad Request | Invalid request or pRPC error |
| 401 Unauthorized | Missing/invalid auth token (header) |
| 404 Not Found | Unknown endpoint or log file |
| 500 Internal Server Error | Server error (e.g., supervisor unreachable) |

---

## Multi-Tenant Rewrite Checklist

### Proto Changes Required

- [ ] Add `tenant_id: string` to every Request message (or wrap in Envelope)
- [ ] Add `tenant_id: string` to RPC service context (if framework supports)
- [ ] Version proto files; maintain backward compatibility via new service methods if needed
- [ ] Consider separate proto packages per tenant (less likely; tenancy via field is simpler)

### Service Handler Changes

- [ ] Implement TenantContext extraction from request (frame, header, or cert)
- [ ] Add authorization checks before every VM/image query (filter by tenant_id)
- [ ] Implement quota enforcement in CreateVm, UpdateVm, ResizeVm
- [ ] Segment image paths per tenant (e.g., `image_path/{tenant_id}/`)
- [ ] Segment run directories per tenant
- [ ] Add per-tenant KMS/gateway URL overrides (or keep global if allowlist-based)
- [ ] Add per-tenant authentication tokens (or mTLS certs per tenant)

### Config & Storage

- [ ] Migrate from single vmm.toml to per-tenant config (file-based or key-value store)
- [ ] Implement config lookup by tenant_id
- [ ] Namespace VM directories by tenant (e.g., `run_path/{tenant_id}/{vm_id}`)
- [ ] Add tenant row/table to persistent state (if tracking tenants)

### HTTP Layer

- [ ] Require Authorization header in all prpc + guest/api endpoints
- [ ] Validate tenant_id extracted from auth token matches request tenant_id
- [ ] Add request/response interceptor to inject tenant context
- [ ] Segment log streaming by tenant (ensure callers can only see their logs)

### Testing & Rollout

- [ ] Add integration tests for cross-tenant access denial
- [ ] Verify quota enforcement under load
- [ ] Test key material rotation per tenant
- [ ] Plan migration path for single-tenant → multi-tenant upgrades

---

## References

### Proto Files
- `vmm/rpc/proto/vmm_rpc.proto` — Main VMM service & messages
- `vmm/rpc/proto/prpc.proto` — pRPC framework error types
- `host-api/proto/host_api.proto` — Host API for TEE integration
- `guest-api/proto/guest_api.proto` — Guest agent proxy & direct API

### Implementation Files
- `vmm/src/main.rs` — Server setup, route mounting (lines 77–109, 111–130)
- `vmm/src/main_service.rs` — RpcHandler impl VmmRpc (all 25 RPC methods)
- `vmm/src/guest_api_service.rs` — GuestApiHandler impl ProxiedGuestApiRpc
- `vmm/src/host_api_service.rs` — HostApiHandler impl HostApiRpc
- `vmm/src/main_routes.rs` — Web UI & log streaming routes
- `vmm/src/config.rs` — Configuration structures & validation
- `vmm/vmm.toml` — Default configuration
- `vmm/src/openapi.rs` — OpenAPI spec generation

### Build & Version
- Cargo version: As of 2025-01-01
- Supported Rocket version: 0.5.x (async, managed state)
- Protocol buffers: proto3 syntax

---

**Document generated:** 2026-05-05  
**Scope:** vmm/ crate only; guest-agent and other components out of scope  
**Next steps:** Use this inventory as baseline for multi-tenant architecture design
