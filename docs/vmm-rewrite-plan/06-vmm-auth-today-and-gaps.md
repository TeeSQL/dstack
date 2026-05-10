# dstack-vmm Auth & Authz: Today + Multi-Tenant Gaps

**Prepared:** 2026-05-05
**Scope:** Authentication and authorization in dstack-vmm as it ships today, followed by the gap analysis for a multi-tenant production rewrite.
**Coverage:** All HTTP / pRPC / vsock surfaces, the `Authorized` request guard, vmm-cli auth, audit logging, plus the forward-looking requirements for orgs, RBAC, audit, isolation, and abuse handling.

> The "today" half (Part 1) is descriptive — every claim has a `file:line` citation. The "gaps" half (Part 2) is forward-looking; it describes what a multi-tenant production system *needs* and is **not** describing existing dstack code.

---

## TL;DR

dstack-vmm is a **single-operator management plane**. It has the *skeleton* of a token-based auth system (a `[auth]` config block, a `rocket-apitoken` dependency, an `Authorized` Rocket guard) but the skeleton is barely wired:

- `[auth]` defaults to **`enabled = false`** with an empty token list, so a stock vmm rejects nothing (`vmm/vmm.toml:126-128`, `vmm/src/main.rs:208`).
- The `Authorized` request guard is applied to **exactly one** route — the VM log streamer (`vmm/src/main_routes.rs:107-116`). The web UI, all pRPC RPCs (CreateVm, RemoveVm, …), the proxied guest-API, and the OpenAPI docs are reachable with no auth at all (`vmm/src/main.rs:81-109`, see route table below).
- Even when `[auth] enabled = true`, the scheme is one shared list of bearer tokens — no users, no scopes, no audit identity, and the official deployment guide tells you to bind it on `tcp:0.0.0.0:9080` (`docs/deployment.md:139`).
- `vmm-cli.py` sends **HTTP Basic** while the server expects **`Authorization: Bearer …`**; the two halves don't actually interoperate (`vmm/src/vmm-cli.py:413-419` vs. fetched `rocket-apitoken/src/lib.rs`).
- Inter-component calls between vmm ↔ KMS, gateway, guest agents are not RA-TLS-pinned at this hop. The vmm's KMS client is constructed with `tls_no_check = true` (`vmm/src/app.rs:1012`), and the vsock host-API trusts the `cid` of the connection as the caller identity (`vmm/src/host_api_service.rs:24-33`). DHCP lease notifications come in as plain unauthenticated POSTs from a shell script (`scripts/dhcp-notify.sh:24-33`).
- There is **no audit log**. Standard `tracing::info!` lines record what the *server* did, never *who* asked for it. There is no `principal`/`actor`/`user` field anywhere in vmm code.

For multi-tenant production this is a from-scratch problem. None of the structures one needs (orgs, members, roles, refresh tokens, scoped API tokens, per-tenant audit retention, suspension primitives, hard cross-org isolation) exist today.

---

## Part 1 — Auth Today (descriptive, with citations)

### 1.1 Bind addresses and listener types

The vmm process is a single Rocket server with two listeners (external + host-api), plus a discovery sidecar.

| Surface | Listener | Default | Protocol | TLS? |
|---|---|---|---|---|
| **External API** (web UI, pRPC, guest-proxy, logs, OpenAPI docs) | Rocket | `unix:./vmm.sock` (`vmm/vmm.toml:11`) — operator switches to `tcp:0.0.0.0:9080` per `docs/deployment.md:139` | HTTP/1.1 | **No.** No TLS config keys exist; Cargo enables the `rocket["mtls"]` feature only so `ra_rpc` can compile against `rocket::mtls::Certificate` types (`vmm/Cargo.toml:13`). The `mtls` feature is **never activated at the listener**. |
| **Host API** (vsock-only) | `rocket-vsock-listener` | `vsock:2`, port `10000` (`vmm/vmm.toml:139-141`, `vmm/src/main.rs:111-130`) | pRPC over vsock | N/A. `HostApiConfig::validate` *enforces* that the address starts with `"vsock:"` and panics otherwise (`vmm/src/config.rs:453-466`). |
| **Discovery file** | filesystem | `$XDG_RUNTIME_DIR/dstack-vmm/<uuid>.json` (`vmm/src/discovery.rs:18-24,67-93`) | filesystem | Anyone with FS access reads/writes |

Cargo features make `rocket["mtls"]` compile but the binary never calls `tls = ...`, never loads certs, and never wraps the listener in TLS. Search for `tls`, `rustls`, `cert`, `pem` inside `vmm/src/` returns no listener-side hits (`Grep "rustls|tls.*pem|tls.*cert|tls\."key|certs"` → no matches).

### 1.2 The `[auth]` block — what it does and doesn't do

#### Config schema

```toml
# vmm/vmm.toml:126-128
[auth]
enabled = false
tokens = []
```

```rust
// vmm/src/config.rs:287-293
#[derive(Debug, Clone, Default, Deserialize)]
pub struct AuthConfig {
    /// Whether to enable API token authentication
    pub enabled: bool,
    /// The API tokens
    pub tokens: Vec<String>,
}
```

#### Wiring in `main`

```rust
// vmm/src/main.rs:208
let api_auth = ApiToken::new(config.auth.tokens.clone(), config.auth.enabled);
…
// vmm/src/main.rs:81-100
let external_api = rocket::custom(figment)
    .mount("/", main_routes::routes())                           // ← unauthenticated except for /logs
    .mount("/guest", ra_rpc::prpc_routes!(App, GuestApiHandler)) // ← unauthenticated
    .mount("/api",   ra_rpc::prpc_routes!(App, HostApiHandler))  // ← unauthenticated
    .mount("/prpc",  ra_rpc::prpc_routes!(App, RpcHandler, trim: "Teepod."))  // ← unauthenticated
    .manage(app)
    .manage(api_auth)
```

The `ApiToken` is `manage`d in Rocket state, but Rocket only consults it where `Authorized` appears as a request guard.

#### `rocket-apitoken` semantics

The crate is an external git dep (`Cargo.toml:171`: `rocket-apitoken = { git = "https://github.com/kvinwang/rocket-apitoken", branch = "dev" }`). Its `lib.rs` (fetched, ~95 lines) implements:

- Constructor prepends literal `"Bearer "` to every token: `tokens.into_iter().map(|t| format!("Bearer {}", t)).collect()` so the only accepted scheme is `Authorization: Bearer <token>`.
- `Authorized` `FromRequest` impl: if `enabled == false` → success unconditionally. Otherwise compare the raw `Authorization` header value to the `HashSet<String>` of `"Bearer …"` strings (constant-time? — no, plain `HashSet::contains`).
- 401 with body `"invalid token"` or `"Authorization header not found"` on failure.
- No expiry, no rotation, no scopes, no per-token rate limits, no audit trail.

#### Where `Authorized` is actually used

```
Grep -r "Authorized" vmm/  →
  vmm/src/main_routes.rs:13   use rocket_apitoken::Authorized;
  vmm/src/main_routes.rs:109      _auth: Authorized,
```

Exactly **one** route — `vm_logs` (`#[get("/logs?<id>&<follow>&<ansi>&<lines>&<ch>")]`). Everything else mounted under `/`, `/guest`, `/api`, `/prpc`, `/api-docs` runs through the `prpc_routes!` macro (`ra-rpc/src/rocket_helper.rs:212-270`), which generates raw `#[rocket::post]` / `#[rocket::get]` handlers with **no `Authorized` guard injected**.

#### Route × auth matrix (external listener)

| Mount | Route | Handler | Guarded by `Authorized`? | Notes |
|---|---|---|---|---|
| `/` | `GET /` | `index` (web UI v1 console) | **No** (`vmm/src/main_routes.rs:45-48,185-187`) | Serves console HTML |
| `/` | `GET /v1`, `/beta`, `/v0` | `v1`, `beta`, `v0` | **No** | Same UI, alternate paths |
| `/` | `GET /res/<path>` | `res` | **No** | Serves `x25519.js` |
| `/` | `GET /logs?…` | `vm_logs` | **Yes** (`main_routes.rs:107-110`) | The lone protected route |
| `/prpc` | `POST /prpc/<method>` | macro-generated `prpc_post` | **No** (`ra-rpc/src/rocket_helper.rs:234-251`) | All `Vmm` service RPCs: `CreateVm`, `StartVm`, `StopVm`, `RemoveVm`, `UpdateVm`, `ShutdownVm`, `ResizeVm`, `Status`, `ListImages`, `GetAppEnvEncryptPubKey`, `GetInfo`, `GetMeta`, `Version`, `ListGpus`, `ReloadVms`, `ReportDhcpLease`, `SvList`, `SvStop`, `SvRemove`, `ListRegistryImages`, `PullRegistryImage`, `DeleteImage`, `GetComposeHash` (`vmm/rpc/proto/vmm_rpc.proto:300-360`) |
| `/prpc` | `GET /prpc/<method>` | macro-generated `prpc_get` | **No** | Same RPCs, GET form (query string body) |
| `/guest` | `POST /guest/<method>` | `ProxiedGuestApi` (proxy → guest agent) | **No** | `Info`, `SysInfo`, `NetworkInfo`, `ListContainers`, `Shutdown` (`vmm/src/guest_api_service.rs:36-58`) |
| `/api` | `POST /api/<method>` | `HostApi` *also mounted on the external listener* | **No** | This is the same `HostApiHandler` mounted twice — once on external (`vmm/src/main.rs:84`), once on the vsock host listener (`vmm/src/main.rs:116`). The external mount means a tcp client of `:9080` can hit `Notify`, `Info`, `GetSealingKey` even though they are intended to be guest→host calls. `HostApiHandler::construct` (`vmm/src/host_api_service.rs:24-33`) `bail!`s if `remote_endpoint != Vsock { … }`, so external callers get a 400 from the construct step rather than a 401 — but the route is still *reachable* without auth. |
| `/api-docs` | `GET /api-docs/openapi.json`, `GET /api-docs/docs` | `mount_openapi_docs` | **No** (`ra-rpc/src/rocket_helper.rs:613-639`) | Spec + Swagger UI |

**Bottom line:** with the default `vmm.toml` (`enabled = false`), every external route is open. Even with `enabled = true` and a tokens list, only `/logs` would actually require a bearer token; every state-mutating RPC is still unauthenticated.

### 1.3 Web UI (port 9080) — what protects it

Nothing on the server side that you've read above does. Specifically:

- **Cookies / sessions:** none. `Grep "cookie|session"` in `vmm/src` and `vmm/ui/src` returns only an unrelated CSS keyframe (`dialogIn`) and `credentials: 'same-origin'` in `vmm/ui/src/lib/vmmRpcClient.ts:74` (the browser will *send* cookies if present, but the server never sets any).
- **JWT / OAuth:** no JWT lib, no OAuth flow anywhere in the workspace.
- **HTTP Basic:** the server does not parse `Authorization: Basic …` at all; only `rocket-apitoken`'s Bearer parser exists, and only for `/logs`.
- **mTLS:** Cargo feature is enabled (`vmm/Cargo.toml:13`) but the listener never activates TLS (no `tls = …` figment provider, no certs in `vmm.toml`).
- **IP allowlist / firewall:** none in code. Operators rely on `tcp:127.0.0.1:9080` + a reverse proxy or VPN, but `docs/deployment.md:139` literally instructs `address = "tcp:0.0.0.0:9080"` which binds *all* interfaces.
- **CORS / CSRF / rate limiting / origin check:** no matches for `cors|csrf|rate.?limit|throttle` anywhere in `vmm/`.
- **HTTP basic in vmm-cli only:** `vmm-cli.py:413-419` builds an `Authorization: Basic …` header — **but the server has no Basic-auth handler**. If `auth.enabled = true` on the server, every CLI call returns 401 (server expects Bearer); if `auth.enabled = false`, the Basic header is silently ignored. The `--auth-user` / `--auth-password` flags and `DSTACK_VMM_AUTH_USER`/`DSTACK_VMM_AUTH_PASSWORD` env vars (`vmm-cli.py:1518-1539`, `docs/vmm-cli-user-guide.md:71-90`) are dead code against a stock vmm; they look like a stub for a reverse-proxy front-end the operator is expected to set up themselves.

The web UI itself is a Vue/TypeScript SPA that calls `/prpc/<Method>` over `fetch` (`vmm/ui/src/lib/vmmRpcClient.ts:65-79`). It has no login screen, no token field, no "current user" surface. There is also a **legacy console** (`vmm/src/console_v0.html`) at `/v0` mounted unguarded.

The HTTP UI also embeds an off-Cloudflare CDN script with `crossorigin="anonymous"` (`vmm/src/console_v0.html:18`); not an auth issue, but worth noting in a hardening pass.

### 1.4 Is there an "admin token" anywhere?

No.

There is no notion of admin-vs-user. Every entry in `[auth] tokens = […]` is equivalent — any holder can call any *guarded* route, which today is just `/logs`. There is no hierarchy, no role tag, no expiry, no key ID. `rocket-apitoken` deliberately keeps a `HashSet<String>`.

`Grep -i "admin.*token|root.*token|master.*token"` returns no matches.

### 1.5 How VMM authenticates calls TO/FROM gateway, KMS, guest agents

#### vmm → KMS (outbound)

```rust
// vmm/src/app.rs:1007-1014
pub(crate) fn kms_client(&self) -> Result<KmsClient<RaClient>> {
    if self.config.kms_url.is_empty() { bail!("KMS is not configured"); }
    let url = format!("{}/prpc", self.config.kms_url);
    let prpc_client = RaClient::new(url, true)?;
    Ok(KmsClient::new(prpc_client))
}
```

The `true` is `tls_no_check` (`ra-rpc/src/client.rs:87-94` + `RaClientConfig::into_client` line 49). So the vmm calls KMS over plain HTTP (its config example uses `http://127.0.0.1:8081`, `vmm/vmm.toml:13,25`) and even if `https://` were used, certificate verification is **disabled**. There is no RA-TLS attestation pinning here, no `cert_validator`, no client cert. The vmm uses this client only for `GetAppEnvEncryptPubKey` (`vmm/src/main_service.rs:455-468`).

The signature on the returned public key is verified *client-side by the depositor* (the dstack-vmm UI / vmm-cli, see `dstack-env-encrypt-pubkey` flow in `vmm/rpc/proto/vmm_rpc.proto:211-221` and `kms/src/main_service.rs`), not by the vmm itself. The vmm is a pure proxy here.

#### vmm → gateway

The vmm does not call the gateway. `Grep "gateway" vmm/src/` finds only configuration: gateway URLs and base domain are pushed *into* CVMs via `make_sys_config` (`vmm/src/app.rs:1139-1168`) so guests can talk to the gateway directly. There is no vmm-to-gateway control channel.

#### vmm ← gateway (inbound)

Likewise, the gateway does not call the vmm. `Grep "vmm|VMM" gateway/src/` returns no matches in source code. The deployment scripts (`gateway/dstack-app/deploy-to-vmm.sh`) call vmm via `vmm-cli.py` from the operator's shell — at deploy time only, not runtime.

#### vmm ← guest agents (inbound)

Guest agents call the vmm's host API on **vsock**:

```rust
// vmm/src/host_api_service.rs:21-33
impl RpcCall<App> for HostApiHandler {
    fn construct(context: CallContext<'_, App>) -> Result<Self> {
        let Some(RemoteEndpoint::Vsock { cid, port }) = context.remote_endpoint else {
            bail!("invalid remote endpoint: {:?}", context.remote_endpoint);
        };
        Ok(Self { endpoint: VsockEndpoint { cid, port }, app: context.state.clone() })
    }
}
```

The "authentication" is: *the connection arrived on vsock, with a CID*. The CID is then used to identify which VM is talking (`HostApiHandler::notify` → `App::vm_event_report(cid, …)` at `vmm/src/host_api_service.rs:44-47` and `vmm/src/app.rs:902-946`, which scans `state.vms.values_mut().find(|vm| vm.config.cid == cid)`).

Implications:

- **No RA-TLS** on this hop. The host trusts the kernel-enforced vsock CID.
- The CID is assigned by the *host* VMM (CID pool, `vmm/vmm.toml:30-31`, `vmm/src/app.rs:151-154`), so a malicious guest cannot forge a different CID — but a *misbehaving guest* can spam `Notify` events for its own CID, including arbitrary `event` strings (`vmm/src/app.rs:923-944` accepts unknown events with just an `error!` log).
- The `GetSealingKey` call on this surface forwards the guest's TDX quote to the local key provider (`vmm/src/host_api_service.rs:49-62`); the only check is `key_provider.enabled`. There is no allowlist here — any guest with a vsock connection can ask.

#### vmm ← DHCP notifier (inbound)

Listed for completeness. The host's dnsmasq calls `POST /prpc/ReportDhcpLease` over plain HTTP (`scripts/dhcp-notify.sh:24-33`, route handler `vmm/src/main_service.rs:579-582`). No auth. If port 9080 is reachable from the network, anyone can spoof DHCP lease entries and rewrite port-forwarding rules for bridge-mode VMs (`vmm/src/app.rs:454-528`).

### 1.6 How vmm-cli authenticates to vmm

It doesn't, in any working way:

- **Default case** (`auth.enabled = false`): cli sends an HTTP Basic header *if `--auth-user`+`--auth-password` are set*; the server ignores it. No auth is performed. (`vmm/src/vmm-cli.py:413-419` and `vmm/src/main.rs:208` defaulting `enabled` from config.)
- **`auth.enabled = true`**: server requires `Authorization: Bearer <token>` — but only on `/logs`. cli's `rpc_call` (`vmm-cli.py:489-503`) hits `/prpc/<method>`, which is unguarded; cli's `show_logs` would 401 against the server because cli sends Basic, not Bearer. So:
  - cli **`logs` command** → 401 every time auth is on. (No code path constructs a Bearer header.)
  - cli **everything else** (`lsvm`, `deploy`, `start`, `stop`, `remove`, `update`, `resize`, `lsimage`, `lsgpu`, …) → works regardless of auth setting because the `/prpc` routes never check anything.

Connection mechanism (orthogonal to auth):

- cli auto-discovers vmm instances by reading `$XDG_RUNTIME_DIR/dstack-vmm/*.json` and `/run/user/<uid>/dstack-vmm/*.json` (`vmm-cli.py:41-105`, written by `vmm/src/discovery.rs:50-97`). Anyone with FS read on `/run/user/<their own uid>` can find their vmm; reading another user's discovery files needs cross-uid FS access, which is the only reason `vmm-cli.py:_get_discovery_dirs` succeeds — it scans `/run/user/*` indiscriminately and silently skips dirs it can't read (`vmm-cli.py:54-59`).
- cli prefers Unix socket over TCP when both are available (`vmm-cli.py:148-165`). For Unix, the only protection is filesystem permissions on `vmm.sock` (handed out via Unix peer creds in `ra-rpc/src/rocket_helper.rs:71-182`, but vmm itself does not consult the peer cred — it's exposed to handlers that ignore it).

### 1.7 Audit logging — what vmm records about "who did what"

There is no audit log. There is `tracing::info!` for *what the server did* but no actor identity is ever recorded.

Concretely:

- No fields named `audit`, `actor`, `principal`, `user_id`, `client_ip` anywhere in `vmm/src/` (`Grep` returns 0 hits each).
- `tracing::instrument` decorators on RPC handlers (`ra-rpc/src/rocket_helper.rs:235,254`) capture only `id` (auto-incrementing request id) and `method`. They do not capture remote address, peer cred, header, or any token identifier.
- The few mutation handlers that do log (`vmm/src/main_service.rs:484` `info!("Resizing VM: {:?}", request)`, `vmm/src/main_service.rs:574` `info!("Reloading VMs directory…")`, `vmm/src/main_service.rs:721` `info!("starting background pull for {tag}")`) print only the request payload, not the caller.
- `vm_event_report` logs guest-emitted events with the CID (`vmm/src/app.rs:902-903`: `info!(cid, event, "VM event")`) — this is for guest-side telemetry, not operator audit.
- VM lifecycle records (`Manifest`, `vmm/src/app.rs:48-73`) carry `created_at_ms` but **no `created_by`**. No `last_modified_by` on update. The persisted on-disk shape (`vmm/src/app.rs:48-73`) has nowhere to record an operator/owner.

The `tracing::info!` stream is useful for forensics about *what* happened (which VM, which RPC, what error); it tells you nothing about *who* triggered the change. With shared bearer tokens (`auth.enabled = true`, multiple operators using the same list), even a token-id-tagged log wouldn't be useful — every operator looks the same.

### 1.8 Tenancy primitives that exist today (essentially: none)

Just to enumerate what *does* exist that could be repurposed:

- **VM workdir layout** (`vmm/src/app.rs:147-149`, `vmm/src/app/qemu.rs`): each CVM gets a directory under `run_path/<vm-id>/`. There is no per-tenant grouping.
- **`name` field validation** (`vmm/src/main_service.rs:58-69`): bounded charset for VM names, but global namespace.
- **App ID** (`vmm/src/main_service.rs:46-55`, `vmm/rpc/proto/vmm_rpc.proto:81-86`): SHA-256 of the compose file (truncated to 40 chars), used to address keys in KMS. Globally unique by construction; no link to "who created it".
- **CID pool** (`vmm/src/app/id_pool.rs`): single global pool. No reservation per tenant.
- **Discovery files** (`vmm/src/discovery.rs`): one file per vmm *process*. The discovery JSON does carry `pid`, `working_dir`, `node_name` — those are operator/process-level, not tenant-level.
- **Linux user / iptables sandbox** (`vmm/src/setup-user.sh`): an out-of-band ops convention to run *the qemu child* under a dedicated user. This is unrelated to authenticating vmm-API callers.

---

## Part 2 — Multi-Tenant Production Gap Analysis (forward-looking)

> Everything below is a *requirement* for a multi-tenant rewrite. None of it exists in the current codebase; treat it as design input, not as documentation of behaviour.

### 2.1 User sessions (browser auth)

Today: nothing — open web UI on whatever port `address` resolves to.

A multi-tenant control plane needs:

- **Login endpoint** issuing a session. Credential factors should include at least one of: (a) email + password with mandatory MFA enrolment, (b) OAuth/OIDC (GitHub, Google), (c) SIWE (Sign-In with Ethereum) — fits the dstack ethos and matches the on-chain identity already used by `DstackKms` / `DstackApp` (see `00-overview.md` open question #4).
- **Session storage**:
  - Short-lived **access cookie** (HttpOnly, Secure, SameSite=Strict, ~15 min). JWT or opaque-and-server-side; opaque + server-side is friendlier for instant revocation, JWT is friendlier for stateless distribution. Recommend **opaque with a fast cache** so suspension is immediate.
  - Long-lived **refresh token** (HttpOnly, Secure, ~30 days, rotating on use, revocable per device). Sliding expiry.
- **CSRF defence**: SameSite=Strict cookies + double-submit token on state-changing requests, or Origin/Referer check on all `POST /prpc/*`. The current vmm has none.
- **Device list / "active sessions"** UI page so users can revoke a stolen cookie.
- **Step-up auth** for destructive actions (delete CVM, billing change, transfer ownership): re-prompt for password / WebAuthn / SIWE within the last N minutes.
- **Session-bound XSRF for the SPA**: the existing UI uses `credentials: 'same-origin'` (`vmm/ui/src/lib/vmmRpcClient.ts:74`); pair this with a CSRF token in a non-HttpOnly cookie that the SPA echoes in a header.
- **Transport**: HTTPS-only externally. mTLS for east-west between control-plane components (vmm ↔ scheduler ↔ KMS ↔ gateway). Serve UI over a stable cert (Let's Encrypt or operator-provided), independent of the per-CVM gateway certs.

### 2.2 API tokens (machine auth, with scopes)

Today: `[auth] tokens = ["raw-string", …]` — single class, no scopes, no expiry, no audit. The shipped `rocket-apitoken` is a placeholder, not a system.

A production token model needs:

- **Per-token identity**: `(token_id, owner_user_id, owner_org_id, name, created_at, last_used_at, expires_at, scopes, status)` persisted in a real datastore. Hash-on-store (Argon2id or HMAC-SHA-256 with per-server pepper) so leaks of the DB don't leak tokens.
- **Scopes** modelled like GitHub PATs / AWS IAM:
  - `cvm:read`, `cvm:write`, `cvm:delete`
  - `image:read`, `image:write` (pull, delete)
  - `secret:read`, `secret:write` (encrypted env)
  - `org:admin`, `org:billing`, `org:audit-read`
  - `node:operate` (operator-level: reload, restart-exited, supervisor sv_*)
  - `webhook:dhcp` (the `ReportDhcpLease` surface deserves a dedicated narrow scope or be moved to a Unix socket)
- **Expiry & rotation**: mandatory `expires_at` (default 90d, max 1y). Rotation endpoint issues new+old in a 24h overlap window.
- **Token prefix** so the value is self-describing in logs/leaks (e.g. `dst_p_…` for personal, `dst_o_…` for org-machine, `dst_d_…` for deploy-bot). Easier secret scanning.
- **Per-token rate limits and IP allowlists** for high-privilege tokens.
- **Audit on every use**: token_id + scope-checked + result.
- **Revoke immediately** propagation (use the cache-with-TTL pattern: revocation pushes a kill-list, tokens checked against in-memory bloom filter on hot path).
- **Auth header**: keep `Authorization: Bearer <token>` for tokens; `Cookie: …` for user sessions. Don't reuse cookies for machine auth.

### 2.3 RBAC inside an organization

Today: no organizations, no roles.

Proposed role set (matches the user-facing shape in `00-overview.md`: organizations, members, droplet-like CVMs):

| Role | CVMs | Images | Secrets / encrypted env | Membership | Billing | Audit log | Org settings |
|---|---|---|---|---|---|---|---|
| **Owner** (≥1 per org, can transfer) | full | full | full | invite/remove anyone, change roles incl. owner-transfer | full | read | full |
| **Admin** | full | full | full | invite members, change non-owner roles | read | read | read |
| **Member** | create/start/stop/update **own** CVMs; read all org CVMs; cannot delete others' | pull/use; cannot delete shared | manage own CVM secrets; cannot read others' | none | none | read events for own CVMs | none |
| **Viewer** | read-only on all org CVMs | read-only | none | none | none | none | none |
| **Billing** (optional split) | none | none | none | none | full | read billing-related | none |
| **Service account** (token-only, no UI login) | scoped via API token scopes | same | same | none | none | none | none |

Key design notes:

- Roles are **per-org**, not global. A user in `org A` with role *Admin* might be *Member* in `org B`.
- Owner-transfer is its own ceremony (re-auth + email confirmation + 24h cooldown to undo).
- "Own CVM" vs "any CVM": for member tier, ownership is tracked on the CVM record (`created_by`, `owner_user_id`). Deletion requires Owner/Admin OR `created_by == self`.
- **Resource quotas** ride on roles: vCPU, memory, disk, GPU count, max CVMs per member, max CVMs per org. Enforced at `CreateVm` time (`vmm/src/main_service.rs:291-327` is the choke point) against `(org_id, user_id)`.
- **Image visibility**: org-private vs public. Today the `image` registry path is global (`vmm/src/config.rs:313-320`); this becomes per-org with shared base images.

### 2.4 Cross-org isolation guarantees — hard vs soft

Multi-tenancy implies an explicit threat model: a malicious tenant must not be able to read, modify, or DoS another tenant's CVMs or secrets.

#### Hard boundaries (must be enforced by the system, not by convention)

- **Compute isolation**: each CVM is already a TDX guest (`docs/security/security-model.md:9-25`) — TEE memory protection is the strongest hard boundary dstack offers. Use it.
- **Storage isolation**: per-CVM workdirs (`run_path/<vm-id>/`, `vmm/src/app.rs:147-149`) need per-org subdirs (`run_path/<org-id>/<vm-id>/`) on a filesystem layout that forbids cross-org reads. Run the qemu child under a per-org Linux user (extension of `setup-user.sh`).
- **Network isolation**:
  - User-mode networking (`vmm/vmm.toml:86-92`) is per-VM by default — fine.
  - Bridge mode (`vmm/vmm.toml:93-96`) currently shares one bridge across all CVMs; multi-tenant bridge mode needs a per-org bridge (or VLAN tag) — *enforced*, not a config knob the operator might forget.
  - Port-mapping ranges (`vmm/vmm.toml:98-103`) are global; need per-org subranges with hard caps.
- **Database / metadata isolation**: every query against the persistence layer must include `org_id` in the WHERE clause. Use row-level security in Postgres or a dedicated schema/db per org. *Single bug = full breach* if the choke point is a single shared `vms` table without RLS — design accordingly.
- **KMS keying**: dstack already derives keys per app-id (`docs/security-guide/security-guide.md`, `kms/src/main_service.rs`); make the org-id part of the derivation context so key collisions across orgs are impossible. *App-id scoping is a soft boundary if app-id is just a SHA-256 of the compose file* — the rewrite should prefix or salt with org-id.
- **Image isolation**: image layer caches must be content-addressable per-org *or* shared but immutable (no per-org write to a shared layer).
- **CID pool**: today single pool (`vmm/src/app/id_pool.rs`); split per host *or* keep global but ensure CID alone never carries authority — the `(cid → vm-id → org-id)` lookup must be guarded by an explicit mapping check, not an implicit "I see this CID, the request is fine" pattern (which is what `host_api_service.rs:24-33` does today).
- **Audit-log isolation**: org A's audit log should not be readable by org B even at the storage layer.

#### Soft conventions (nice-to-have, do not rely on for security)

- Naming conventions like `<org>-<vm>` — useful for ops, must not be the basis for any access check.
- Per-org reverse-proxy hostnames — useful, but the auth check still happens server-side after routing.
- "Don't peek at other tenants' logs" admin policy — humans are the weakest link; encode the boundary, don't write it down.

### 2.5 Audit logging requirements for shared / production deployments

Today: zero. Building this from scratch.

Required events (write to an append-only sink):

- **Authentication**: login success/failure, MFA challenge, MFA success/failure, session created/refreshed/revoked, token created/used/expired/revoked, password change, OAuth bind/unbind, SIWE signature received.
- **Membership**: invite sent, invite accepted/declined, member added, member removed, role changed, ownership transferred.
- **CVM lifecycle**: created (with full request snapshot — vcpu, memory, image, gpus, network mode, port map, app_id), started, stopped, shut down, removed, resized, env updated, compose updated, gpus changed, suspended, resumed.
- **Image / registry**: image pulled, image deleted, registry whitelisted.
- **Secrets**: encrypted-env uploaded (record only metadata, never the ciphertext payload itself), kms-public-key requested.
- **Org settings**: quota changed, billing payment method changed, abuse flag set/cleared.
- **Operator (cross-tenant) actions**: cluster suspend, force-stop, force-remove, manual key revocation. These are the most sensitive — add `operator_id`, `target_org_id`, `target_vm_id`, free-form `reason` (mandatory).

Each record should carry:

- `event_id` (uuid), `timestamp` (UTC, monotonic-friendly), `event_type`, `org_id`, `actor_type` (`user|service|operator|system`), `actor_id`, `actor_session_id` or `actor_token_id`, `client_ip`, `user_agent`, `target_type`, `target_id`, `result` (`success|failure|denied`), `error_code` (if any), `request_id` (correlate with `tracing` span), `payload_diff` (canonical JSON-Patch where applicable).

Storage / retention:

- **Hot store**: 30–90 days, queryable by org (the org-admin UI shows recent events).
- **Cold store**: 7 years for compliance, append-only, integrity-hashed (Merkle log or signed batches). Off-host (S3 + Object Lock, or similar).
- **Tamper-evidence**: each batch signed by a control-plane key; on read, re-verify. Useful in disputes ("we didn't suspend you" / "yes you did, here's the signed record").
- **Privacy**: no secrets, no encrypted-env values, no compose env vars, no KMS-key bytes; only metadata/hashes.
- **Per-org export**: GDPR-style export ("show me everything you logged for my org").

Operationalising:

- Build the audit hook at the **request-guard layer**, not in each handler — that way, missing the call is impossible. Today the choke point would be a Rocket fairing (or rewriting `prpc_routes!` macro).
- Async write to a buffered sink; never block the RPC on the audit write *but never drop* — use a local WAL with retries.
- Stream audit events to operators' SIEM (syslog/JSON over TLS) as well as the in-product UI.

### 2.6 Suspension / kill-switch primitives for abuse handling

Today: nothing — operator must SSH in and kill processes manually, or call `RemoveVm` (which is not reversible).

Needed:

- **Per-CVM suspend** that:
  - Stops the qemu child (already exists via `App::stop_vm`, `vmm/src/app.rs:284-289`),
  - Marks the CVM record `status = "suspended_by_operator"` so members see *why* it's down,
  - Disables `start_vm` until cleared,
  - Optionally retains state vs. destroys disk (two flavours).
- **Per-org suspend** that suspends every CVM in the org and blocks new `CreateVm`. Should also block UI login or surface an "account suspended" page.
- **Per-user suspend** (lighter — disables that user's tokens/sessions, leaves the rest of the org intact, useful for a single compromised account).
- **Global rate-limit / circuit-breaker** for: token-creation, VM-creation, image-pull, gateway-route-creation. Most abuse vectors look like "create thousands of CVMs in a minute".
- **Compose / image content blocks**: hash-based deny list of malicious compose hashes or image digests. Today compose hash lives in `app_id` (`vmm/src/main_service.rs:46-55`) — easy hook point.
- **Egress / port restrictions**: tighten port-mapping range (`vmm/vmm.toml:98-103`) per-org-suspended to e.g. zero. Today the range is global.
- **Mandatory operator note** on every kill-switch action (free-form `reason`, written to audit). Don't allow silent suspends.
- **Reversal SLO**: every kill-switch must have an undo command and a documented expected unsuspend latency (so abuse-team operators have a runbook, not just a hammer).
- **Tamper-resilience**: kill-switch state is per-org and persisted; a vmm restart must re-apply it before reload-vms (`vmm/src/main_service.rs:574-577`) ever starts a CVM.
- **Out-of-band emergency stop**: a control-plane-level "freeze region" that the host operator can flip if a tenant's load is wedging the entire host. Today the only such tool is `kill -9` at the supervisor level (`vmm/src/main_service.rs:609-616` `SvStop`).

### 2.7 Cross-cutting threat-model checks the rewrite needs to satisfy

Rolled up so reviewers see them in one place:

- **No silent fail-open**: today `auth.enabled = false` is the default. The rewrite should fail-*closed* — refuse to start if the auth backend isn't configured.
- **No "trust the network"**: every internal hop (vmm ↔ scheduler, vmm ↔ KMS, vmm ↔ gateway) must be mutually authenticated (mTLS or RA-TLS). The current pattern of `RaClient::new(url, true)` with `tls_no_check = true` (`vmm/src/app.rs:1012`) must die.
- **No "trust the CID"**: the host-API surface must validate that a guest's vsock CID matches a CVM owned by the org claiming the call, *before* taking any state-mutating action. Today's `vm_event_report` (`vmm/src/app.rs:902-946`) does the lookup but applies no policy beyond "is there a VM with this CID".
- **No DHCP-by-shell-script**: `scripts/dhcp-notify.sh` should call a Unix socket protected by peer creds (already supported by `ra-rpc/src/rocket_helper.rs:71-182`, currently unused), or be replaced by an internal vmm component watching the lease file. It must not be a publicly reachable HTTP endpoint when the vmm binds to `0.0.0.0`.
- **No "operators are trusted"** as the entire authz model: today the security doc explicitly says "infrastructure operators can deny service" but does not address "what if the operator is the abuser". Multi-tenant changes that — operators are now untrusted by *each other's* tenants. RBAC plus audit plus kill-switch primitives must be designed assuming a malicious operator within a single org.

---

## Appendix A — File:line reference index

Ordered so a reader can jump to the source of any claim.

| Claim / topic | File:line |
|---|---|
| `[auth] enabled = false` default | `vmm/vmm.toml:126-128` |
| `AuthConfig` struct | `vmm/src/config.rs:287-293` |
| `ApiToken::new` invocation | `vmm/src/main.rs:208` |
| External listener mounts (no `Authorized` on `/`, `/guest`, `/api`, `/prpc`) | `vmm/src/main.rs:81-100` |
| Host API listener (vsock-only, validated) | `vmm/src/main.rs:111-130` |
| `HostApiConfig::validate` enforces vsock | `vmm/src/config.rs:453-466` |
| `Authorized` only on `vm_logs` | `vmm/src/main_routes.rs:13,107-110,185-187` |
| `prpc_routes!` macro generates raw POST/GET, no `Authorized` | `ra-rpc/src/rocket_helper.rs:212-270` |
| `rocket-apitoken` crate (Bearer-only, single-list) | external git, fetched: prepends `"Bearer "` to tokens, `HashSet::contains` check |
| `rocket-apitoken` Cargo dep | `Cargo.toml:171` |
| Production deploy guide instructs `tcp:0.0.0.0:9080` | `docs/deployment.md:139` |
| Web UI fetch with `credentials: 'same-origin'` | `vmm/ui/src/lib/vmmRpcClient.ts:74` |
| Legacy `console_v0.html` mounted at `/v0` unguarded | `vmm/src/main_routes.rs:60-63,185-187` |
| vmm-cli sends Basic instead of Bearer | `vmm/src/vmm-cli.py:413-419` |
| vmm-cli auth flag wiring | `vmm/src/vmm-cli.py:1518-1539,1942` |
| vmm-cli auth instructions in user guide | `docs/vmm-cli-user-guide.md:71-90` |
| KMS client uses `tls_no_check=true` | `vmm/src/app.rs:1007-1014` |
| `RaClient::new` semantics | `ra-rpc/src/client.rs:87-94` |
| Host API trusts vsock CID as identity | `vmm/src/host_api_service.rs:24-47` |
| `vm_event_report` — CID-lookup-and-go | `vmm/src/app.rs:902-946` |
| `GetSealingKey` only checks `key_provider.enabled` | `vmm/src/host_api_service.rs:49-62` |
| Discovery JSON written to runtime dir | `vmm/src/discovery.rs:50-97` |
| `DhcpLeaseRequest` route is unauthenticated POST | `vmm/src/main_service.rs:579-582`, `scripts/dhcp-notify.sh:24-33` |
| `ReportDhcpLease` reconfigures port-forward rules | `vmm/src/app.rs:454-528` |
| Manifest has `created_at_ms`, no `created_by` | `vmm/src/app.rs:48-73` |
| `App::start_vm` / `stop_vm` / `remove_vm` | `vmm/src/app.rs:221-321` |
| `RpcHandler::create_vm` (single choke point for resource quotas in a rewrite) | `vmm/src/main_service.rs:291-327` |
| App-id derivation = `sha256(compose_file)[..40]` (no org salt today) | `vmm/src/main_service.rs:46-55` |
| `setup-user.sh` — Linux-user / iptables sandbox for qemu child | `vmm/src/setup-user.sh:1-230` |
| `UnixPeerCredListener` exists, vmm doesn't enforce on it | `ra-rpc/src/rocket_helper.rs:71-182` |
| No `audit|actor|principal|user_id|client_ip` in vmm/src | confirmed via Grep — 0 hits |
| No `cors|csrf|rate.?limit|throttle` in vmm | confirmed via Grep — 0 hits in vmm code |

---

## Appendix B — Open questions surfaced for the synthesis pass

These are not gaps in the today-state document; they are decisions the rewrite needs to make and that this report can't decide unilaterally.

1. **Are operators tenants?** I.e. does the host operator have a "host-admin" identity inside the same RBAC system (one big graph), or does the host operator live outside the RBAC system entirely (root SSH, separate console)? I'd lean *outside* — host-admin actions go through a different binary and a different audit stream — but it changes the schema.
2. **Identity binding to on-chain wallets.** dstack's KMS authority model uses an EOA per app (`docs/onchain-governance.md`, `kms/auth-eth/`). Multi-tenant orgs need a story: org owns one wallet (multisig), each user owns their own and signs as themselves, or the control plane custodies. Different choice → different audit/auth flows.
3. **Self-hosted vs SaaS.** Self-hosted means audit retention is the operator's problem and abuse-handling primitives can be lighter; SaaS means everything in §2.5 / §2.6 is mandatory.
4. **Do we keep pRPC?** The current `prpc_routes!` macro is the right place to put a single auth/audit choke point, but the macro structure also makes adding scopes per-method awkward. Either extend the macro (`#[prpc(scope = "cvm:write")]` attribute) or move to a dedicated framework (axum + tower middleware) where layered auth is idiomatic.
5. **Where do we draw the line between "vmm" and "control plane"?** Multi-tenant work could either (a) thicken the existing vmm with users/orgs/RBAC, or (b) extract a control-plane service in front of vmm, leaving vmm as a single-host worker daemon called only over a mutually-attested channel. (b) is a much cleaner separation of concerns and is the option `00-overview.md` hints at — but it means the work in this report mostly happens in *the new control plane*, and the *vmm* gets a much narrower, mTLS-only RPC.
