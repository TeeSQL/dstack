# dstack-vmm CLI (`vmm-cli.py`) — Capability and Multi-Tenant Audit

**Prepared:** 2026-05-05
**Scope:** Behaviour, capabilities, security model, and multi-tenant blockers in the operator-side CLI shipped at `vmm/src/vmm-cli.py`.
**Source under audit:** `/home/fbx/dstack/vmm/src/vmm-cli.py` (2024 lines, 73 KB), companion docs `docs/vmm-cli-user-guide.md`.

---

## 1. Executive Summary

`vmm-cli.py` is a **single-file Python 3 client** for `dstack-vmm`'s `/prpc/<method>` endpoints plus the streaming `/logs` route. It is the operator's most-feature-complete control surface: it implements every capability the web UI offers, plus several the UI does not (resize, multi-aspect `update`, app-compose generation, multi-VMM discovery, KMS signer whitelisting). Effectively it doubles as **dstack-vmm's reference admin tool** and is referenced from production tutorials.

That status is at odds with three structural realities of the rewrite:

1. **The CLI is a single-operator god-mode tool.** It scans `/run/user/*/dstack-vmm/*.json` across **every UID on the host** to discover VMM instances (`vmm-cli.py:38–59, 85–106`). It carries no caller identity into RPC calls — the request body is the entire input to the server. There is no concept of "my VMs" vs "your VMs" anywhere in the client or server today.

2. **Authentication is broken end-to-end.** The CLI sends `Authorization: Basic <base64(user:password)>` (`vmm-cli.py:413–419`). The server's only auth guard, `rocket-apitoken::Authorized`, accepts only `Authorization: Bearer <token>` strings (`~/.asdf/installs/rust/1.84.0/git/checkouts/rocket-apitoken-bd4044808b9d81fc/7b5c1e8/src/lib.rs:50–58, 91–101`). The two literally never match — Basic credentials are silently a no-op. Worse, the guard is only mounted on `/logs` (`vmm/src/main_routes.rs:107–116`); the entire `/prpc/*`, `/guest/*` and `/api/*` RPC surface has no auth guard at all (`vmm/src/main.rs:82–88` via `ra_rpc::prpc_routes!`, see `ra-rpc/src/rocket_helper.rs:212–270`). The default `vmm.toml` further disables auth entirely (`vmm/vmm.toml:126–128`: `[auth] enabled = false`). The CLI's `--auth-user`/`--auth-password` machinery is therefore vestigial.

3. **TLS verification is disabled.** When connecting via HTTPS the CLI installs a context with `check_hostname = False` and `verify_mode = ssl.CERT_NONE` (`vmm-cli.py:431–436`). Any MITM on the operator → VMM hop succeeds.

In a multi-tenant rewrite, every CLI subcommand must be re-modelled around an authenticated principal, an org-scoped resource set, and proper TLS — which means the existing CLI cannot be ported, only replaced. The CLI's affordances (compose generation, multi-aspect updates, signer whitelisting) are still useful patterns to bring forward.

---

## 2. CLI Structure

### 2.1 File layout

`vmm-cli.py` is one file, divided as follows:

| Lines | Section | Contents |
|---|---|---|
| 1–35 | Module preamble | Optional `cryptography` / `eth_keys` / `eth_utils` import via `try/except` toggling `CRYPTO_AVAILABLE` (`vmm-cli.py:21–31`); default config paths (`vmm-cli.py:33–35`). |
| 38–105 | VMM discovery | `_get_discovery_dirs()` (`vmm-cli.py:41–59`) and `discover_vmm_instances()` (`vmm-cli.py:79–105`). |
| 108–174 | URL resolution / active-instance config | `resolve_vmm_url()` (`vmm-cli.py:108–145`), `vmm_address_to_url()` (`vmm-cli.py:148–165`), `save_active_vmm()` (`vmm-cli.py:168–174`). |
| 177–257 | `vmm ls` / `vmm switch` handlers | `cmd_ls_vmm()` (`vmm-cli.py:177–226`), `cmd_switch_vmm()` (`vmm-cli.py:229–257`). |
| 260–319 | Env-var encryption helper | `encrypt_env()` X25519+AES-GCM (`vmm-cli.py:260–319`). |
| 322–346 | Misc parsing helpers | `parse_port_mapping()` (`vmm-cli.py:322–340`), `read_utf8()` (`vmm-cli.py:343–346`). |
| 349–363 | Unix socket transport | `UnixSocketHTTPConnection` (`vmm-cli.py:349–363`). |
| 366–472 | HTTP client | `VmmClient` (`vmm-cli.py:366–472`). |
| 475–1249 | Command dispatcher | `VmmCLI` class (`vmm-cli.py:475–1249`). |
| 1252–1374 | Output / size helpers | `format_table()` (`vmm-cli.py:1252–1275`), `parse_env_file()` (`vmm-cli.py:1278–1298`), `parse_size()`/`parse_memory_size()`/`parse_disk_size()` (`vmm-cli.py:1301–1374`). |
| 1377–1501 | KMS signature verification | `verify_signature_v1()` (`vmm-cli.py:1377–1419`), `verify_signature()` (`vmm-cli.py:1422–1470`), `load_whitelist()` / `save_whitelist()` (`vmm-cli.py:1473–1501`). |
| 1504–2024 | Argparse + dispatch | `main()` (`vmm-cli.py:1504–2024`). |

### 2.2 Entry point

`main()` at `vmm-cli.py:1504` does, in order:

1. Loads `~/.dstack-vmm/config.json` via `load_config()` (`vmm-cli.py:62–76, 1509`).
2. Discovers all running VMMs on the host via `discover_vmm_instances()` (`vmm-cli.py:1512`).
3. Resolves defaults for `--url`, `--auth-user`, `--auth-password` from environment variables `DSTACK_VMM_URL`, `DSTACK_VMM_AUTH_USER`, `DSTACK_VMM_AUTH_PASSWORD` (`vmm-cli.py:1514–1521`).
4. Builds an `argparse` tree (`vmm-cli.py:1523–1927`).
5. Short-circuits the `vmm ls` / `vmm switch` commands without ever opening a socket (`vmm-cli.py:1931–1938`).
6. For all other commands, calls `resolve_vmm_url()` to pick a server, instantiates `VmmCLI(url, auth_user, auth_password)`, and dispatches by `args.command` (`vmm-cli.py:1940–2020`).

### 2.3 Key classes

#### `UnixSocketHTTPConnection` (`vmm-cli.py:349–363`)
A 14-line subclass of `http.client.HTTPConnection` that overrides `connect()` to open `AF_UNIX, SOCK_STREAM` sockets so the same `http.client` request path can speak to either TCP or Unix-domain VMM instances.

#### `VmmClient` (`vmm-cli.py:366–472`)
Thin transport wrapper:

- Detects `unix:` prefix and uses `UnixSocketHTTPConnection` accordingly (`vmm-cli.py:377–387, 428–429`).
- Adds `Authorization: Basic <base64(user:pass)>` if both `auth_user` and `auth_password` are set (`vmm-cli.py:413–419`). **This is the load-bearing line of the broken auth model.**
- Disables hostname/cert verification on HTTPS (`vmm-cli.py:431–436`, comment "TODO: we should verify TLS cert").
- Best-effort JSON sniffing: parses the response as JSON if `Content-Type: application/json` or the body starts with `{`/`[` (`vmm-cli.py:454–462`).
- Supports streaming (used only by `logs --follow`, `vmm-cli.py:447–449, 472`).

#### `VmmCLI` (`vmm-cli.py:475–1249`)
One method per CLI verb. All RPC calls go through `rpc_call()` (`vmm-cli.py:489–503`), which targets `/prpc/<Method>?json` with a JSON body and raises on any non-200 status.

---

## 3. Subcommand Inventory

Every subcommand registered in `main()`'s argparse tree, with the RPC(s) it triggers and side-effects it has on the operator's machine.

| Subcommand | Defined at | Handler | RPC / route(s) | Required positional / flag args | Local side-effects |
|---|---|---|---|---|---|
| `vmm ls` | parser `vmm-cli.py:1551–1556`, dispatch `vmm-cli.py:1932–1933` | `cmd_ls_vmm()` (`vmm-cli.py:177–226`) | **None.** Pure local scan of `/run/user/*/dstack-vmm/*.json`. | none (`--json` optional) | Reads `/run/user/<uid>/dstack-vmm/*.json` for every UID; reads `/proc/<pid>` for liveness check (`vmm-cli.py:96`); reads `~/.dstack-vmm/config.json`. |
| `vmm switch` | parser `vmm-cli.py:1558–1563`, dispatch `vmm-cli.py:1934–1935` | `cmd_switch_vmm()` (`vmm-cli.py:229–257`) | **None.** | `vmm_id` (prefix-matched) | Writes `~/.dstack-vmm/config.json` with new `active_vmm` (`vmm-cli.py:168–174, 255`). |
| `lsvm` | parser `vmm-cli.py:1585–1591`, dispatch `vmm-cli.py:1944–1945` | `VmmCLI.list_vms()` (`vmm-cli.py:505–549`) | `POST /prpc/Status` (`vmm-cli.py:507`) | none (`-v`, `--json` optional) | Reads `~/.dstack-vmm/config.json` for default URL; opens TCP/UDS socket. |
| `info` | parser `vmm-cli.py:1594–1598`, dispatch `vmm-cli.py:1946–1947` | `VmmCLI.show_info()` (`vmm-cli.py:1178–1222`) | `POST /prpc/GetInfo` (`vmm-cli.py:1180`) | `vm_id` | Same network/file reads as above. |
| `start` | parser `vmm-cli.py:1601–1602`, dispatch `vmm-cli.py:1948–1949` | `VmmCLI.start_vm()` (`vmm-cli.py:567–570`) | `POST /prpc/StartVm` (`vmm-cli.py:569`) | `vm_id` | Network only. |
| `stop` | parser `vmm-cli.py:1605–1609`, dispatch `vmm-cli.py:1950–1951` | `VmmCLI.stop_vm()` (`vmm-cli.py:572–579`) | `POST /prpc/ShutdownVm` (default, `vmm-cli.py:578`) **or** `POST /prpc/StopVm` (`-f/--force`, `vmm-cli.py:575`) | `vm_id` | Network only. |
| `remove` | parser `vmm-cli.py:1612–1613`, dispatch `vmm-cli.py:1952–1953` | `VmmCLI.remove_vm()` (`vmm-cli.py:581–584`) | `POST /prpc/RemoveVm` (`vmm-cli.py:583`) | `vm_id` | Network only. Server deletes VM workdir; CLI does not. |
| `resize` | parser `vmm-cli.py:1616–1625`, dispatch `vmm-cli.py:1954–1961` | `VmmCLI.resize_vm()` (`vmm-cli.py:586–611`) | `POST /prpc/ResizeVm` (`vmm-cli.py:610`) | `vm_id`; at least one of `--vcpu`, `--memory`, `--disk`, `--image` (validation `vmm-cli.py:605–608`) | Network only. **Has no UI counterpart.** |
| `logs` | parser `vmm-cli.py:1628–1635`, dispatch `vmm-cli.py:1962–1963` | `VmmCLI.show_logs()` (`vmm-cli.py:613–645`) | **`GET /logs?id=<id>&follow=...&ansi=false&lines=<n>`** (`vmm-cli.py:615`) — the only auth-guarded route on the server (`vmm/src/main_routes.rs:107–116`) | `vm_id`; `-n/--lines`, `-f/--follow` optional | Network only. With `-f`, holds a long-lived connection until Ctrl-C (`vmm-cli.py:629–642`). |
| `compose` | parser `vmm-cli.py:1638–1689`, dispatch `vmm-cli.py:1964–1965` | `VmmCLI.create_app_compose()` (`vmm-cli.py:782–829`) | **None.** Pure local file generation. | `--name`, `--docker-compose`, `--output`; many optional toggles (`--kms`, `--gateway`, `--local-key-provider`, `--key-provider`, `--key-provider-id`, `--public-logs`, `--public-sysinfo`, `--env-file`, `--no-instance-id`, `--secure-time`, `--swap`, `--prelaunch-script`) | Reads `--docker-compose` and optional `--prelaunch-script`; writes `--output` (`vmm-cli.py:826–828`) and prints SHA-256 of the generated JSON. **Has no UI counterpart** (the UI builds compose JSON inline in `vmm/ui/src/composables/useVmManager.ts:712–767`). |
| `deploy` | parser `vmm-cli.py:1692–1770`, dispatch `vmm-cli.py:1966–1967` | `VmmCLI.create_vm()` (`vmm-cli.py:831–907`) | `POST /prpc/CreateVm` (`vmm-cli.py:905`); if `--env-file` set, also `POST /prpc/GetAppEnvEncryptPubKey` against the supplied KMS URL (`vmm-cli.py:670–680, 898–901`) | `--name`, `--image`, `--compose`; many optional flags (`--vcpu`, `--memory`, `--disk`, `--swap`, `--env-file`, `--user-config`, `--app-id`, `--port`, `--gpu`, `--ppcie`, `--pin-numa`, `--hugepages`, `--kms-url`, `--gateway-url`, `--stopped`, `--no-tee`/`--tee`, `--net`) | Reads compose, env-file, user-config from disk. Computes app_id locally as `sha256(compose_file)[:40]` if `--app-id` not provided (`vmm-cli.py:777–780, 896`). Encrypts envs locally using ephemeral X25519 + AES-256-GCM (`vmm-cli.py:260–319`). |
| `lsimage` | parser `vmm-cli.py:1773–1776`, dispatch `vmm-cli.py:1968–1969` | `VmmCLI.list_images()` (`vmm-cli.py:647–663`) | `POST /prpc/ListImages` (`vmm-cli.py:649`) | none (`--json` optional) | Network only. |
| `lsgpu` | parser `vmm-cli.py:1779–1782`, dispatch `vmm-cli.py:1970–1971` | `VmmCLI.list_gpus()` (`vmm-cli.py:1224–1249`) | `POST /prpc/ListGpus` (`vmm-cli.py:1226`) | none (`--json` optional) | Network only. |
| `update-env` | parser `vmm-cli.py:1785–1794`, dispatch `vmm-cli.py:1972–1975` | `VmmCLI.update_vm_env()` (`vmm-cli.py:909–964`) | `POST /prpc/GetInfo` → `POST /prpc/GetAppEnvEncryptPubKey` (KMS) → `POST /prpc/UpgradeApp` (chain at `vmm-cli.py:919, 930, 963`) | `vm_id`, `--env-file`; `--kms-url` required (`vmm-cli.py:914–915`) | Reads env-file. Recomputes `allowed_envs` and (if `APP_LAUNCH_TOKEN` present) `launch_token_hash` (`vmm-cli.py:946–957`); merges into compose JSON; encrypts envs. |
| `update-app-compose` | parser `vmm-cli.py:1816–1822`, dispatch `vmm-cli.py:1978–1979` | `VmmCLI.update_vm_app_compose()` (`vmm-cli.py:971–974`) | `POST /prpc/UpgradeApp` with `compose_file` field (`vmm-cli.py:973`) | `vm_id`, `compose` (path) | Reads compose from disk. |
| `update-user-config` | parser `vmm-cli.py:1825–1831`, dispatch `vmm-cli.py:1976–1977` | `VmmCLI.update_vm_user_config()` (`vmm-cli.py:966–969`) | `POST /prpc/UpgradeApp` with `user_config` field (`vmm-cli.py:968`) | `vm_id`, `user_config` (path) | Reads user-config from disk. |
| `update-ports` | parser `vmm-cli.py:1834–1844`, dispatch `vmm-cli.py:1980–1981` | `VmmCLI.update_vm_ports()` (`vmm-cli.py:976–982`) | `POST /prpc/UpgradeApp` with `update_ports=true` and `ports=[...]` (`vmm-cli.py:978–981`) | `vm_id`, one or more `--port` | None beyond network. |
| `update` (multi-aspect) | parser `vmm-cli.py:1846–1926`, dispatch `vmm-cli.py:1982–2010` | `VmmCLI.update_vm()` (`vmm-cli.py:984–1176`) | `POST /prpc/ResizeVm` if any of `--vcpu`/`--memory`/`--disk`/`--image` set (`vmm-cli.py:1028–1030`); `POST /prpc/GetInfo` if compose-derived or env updates needed (`vmm-cli.py:1043–1049`); `POST /prpc/GetAppEnvEncryptPubKey` if `--env-file` set (`vmm-cli.py:1095–1097`); `POST /prpc/UpgradeApp` aggregating compose / env / user-config / ports / GPUs / TEE flag (`vmm-cli.py:1170–1171`) | `vm_id`; many optional flags (`--vcpu`, `--memory`, `--disk`, `--image`, `--compose`, `--prelaunch-script`, `--env-file`, `--user-config`, `--port`/`--no-ports`, `--swap`, `--gpu`/`--ppcie`/`--no-gpus`, `--no-tee`/`--tee`, `--kms-url`) | Reads compose, prelaunch script, env-file, user-config; rebuilds compose JSON in-process (`vmm-cli.py:1051–1086`); encrypts envs; updates `allowed_envs` and `launch_token_hash` similarly to `update-env`. |
| `kms list` | parser `vmm-cli.py:1797–1801`, dispatch `vmm-cli.py:2011–2018` | `VmmCLI.manage_kms_whitelist('list')` (`vmm-cli.py:733–745`) | **None.** | none | Reads `~/.dstack-vmm/kms-whitelist.json` (`vmm-cli.py:1473–1489`). |
| `kms add` | parser `vmm-cli.py:1804–1807`, dispatch `vmm-cli.py:2011–2018` | `VmmCLI.manage_kms_whitelist('add')` (`vmm-cli.py:758–765`) | **None.** | `pubkey` (33-byte secp256k1 compressed key, hex) | Writes `~/.dstack-vmm/kms-whitelist.json`. |
| `kms remove` | parser `vmm-cli.py:1810–1813`, dispatch `vmm-cli.py:2011–2018` | `VmmCLI.manage_kms_whitelist('remove')` (`vmm-cli.py:766–772`) | **None.** | `pubkey` | Writes `~/.dstack-vmm/kms-whitelist.json`. |

**RPC method total:** 11 distinct VMM RPCs invoked by the CLI — `Status`, `GetInfo`, `StartVm`, `StopVm`, `ShutdownVm`, `RemoveVm`, `ResizeVm`, `CreateVm`, `UpgradeApp`, `ListImages`, `ListGpus`. Plus one external KMS RPC, `GetAppEnvEncryptPubKey` (proxied via VMM by default, or sent direct to a KMS URL when `--kms-url` is provided — see `vmm-cli.py:665–681`). Plus the `/logs` HTTP route. The full server RPC surface is enumerated in `01-vmm-api-surface.md`; methods such as `ReloadVms`, `ListRegistryImages`, `PullRegistryImage`, `DeleteImage`, `Version`, `GetMeta`, `SvList`, `SvStop`, `SvRemove` exist on the server but are **not** exposed by the CLI (the web UI calls some of them — see §7).

---

## 4. User Flows

Three representative flows showing how the CLI composes its primitives.

### 4.1 First-time deploy (compose → deploy)

```bash
# Step 1: build the app-compose.json locally
./vmm-cli.py compose \
  --name secure-app \
  --docker-compose ./docker-compose.yml \
  --kms --gateway \
  --env-file ./secrets.env \
  --output ./app-compose.json

# Step 2: deploy the VM
./vmm-cli.py deploy \
  --name secure-app-vm \
  --image dstack-0.5.3 \
  --compose ./app-compose.json \
  --vcpu 2 --memory 4G --disk 50G \
  --env-file ./secrets.env \
  --kms-url https://kms.example.com:8443
```

What actually happens (file:line references):

1. **`compose`** (`vmm-cli.py:782–829`): reads docker-compose YAML, builds the canonical `AppCompose` JSON dict, computes `sha256` of the resulting JSON bytes, prints the hash, writes the file. **No RPC.** The hash is what the on-chain `DstackApp` whitelist must contain to allow this app to receive keys.
2. **`deploy`** (`vmm-cli.py:831–907`):
   - Reads `app-compose.json` (`vmm-cli.py:837`).
   - Validates that `--env-file` requires `--kms-url` and `kms_enabled=true` in the compose file (`vmm-cli.py:842–854`).
   - Computes `app_id = sha256(compose_file)[:40]` locally if `--app-id` not provided (`vmm-cli.py:777–780, 896`).
   - Calls `GetAppEnvEncryptPubKey(app_id)` against the provided `--kms-url[0]` (or via VMM proxy if no `--kms-url`) to fetch the per-app X25519 public key (`vmm-cli.py:898–901, 665–726`).
   - Verifies the KMS signature on the returned key (preferring `signature_v1` with timestamp, falling back to legacy `signature`, `vmm-cli.py:683–724`); compares the recovered signer's compressed pubkey against `~/.dstack-vmm/kms-whitelist.json` (`vmm-cli.py:713–724`).
   - Encrypts the env vars with ephemeral X25519 → shared secret → AES-256-GCM (`vmm-cli.py:260–319`); the resulting hex blob is `ephemeral_pubkey || iv || ciphertext`.
   - Posts `CreateVm` with the compose, encrypted env, image, vCPU/memory/disk, ports, GPUs, etc. (`vmm-cli.py:861–905`).

**Trust observation.** The compose-hash-as-app-id derivation is deliberate — the server cannot lie about which app the operator is creating, because the same hash function will reproduce on deploy. But it means **every operator who ships an identical compose file will deploy under the same app_id** (see §8).

### 4.2 Rotating a secret (`update-env`)

```bash
./vmm-cli.py update-env <vm_id> \
  --env-file ./new-secrets.env \
  --kms-url https://kms.example.com:8443
```

Trace (`VmmCLI.update_vm_env`, `vmm-cli.py:909–964`):

1. `POST /prpc/GetInfo` to fetch the VM's existing `app_id` and current compose JSON (`vmm-cli.py:919–927`).
2. `POST /prpc/GetAppEnvEncryptPubKey` (via supplied `--kms-url[0]`) for that `app_id` (`vmm-cli.py:930–931`); same signature verification + whitelist check as deploy.
3. Compute new `allowed_envs` from the env-file keys, merge into the existing compose JSON; if `APP_LAUNCH_TOKEN` is one of the keys, recompute `launch_token_hash = sha256(value)` (`vmm-cli.py:946–957`).
4. Encrypt envs locally as before.
5. `POST /prpc/UpgradeApp` with `id`, optional updated `compose_file`, and `encrypted_env` (`vmm-cli.py:937–963`).

**Important:** the compose hash changes when `allowed_envs` or `launch_token_hash` change. Whether the new compose hash is whitelisted on-chain is the operator's responsibility — the CLI does not check.

### 4.3 Multi-instance discovery (`vmm ls` / `vmm switch`)

```bash
./vmm-cli.py vmm ls
./vmm-cli.py vmm switch <id-prefix>
./vmm-cli.py lsvm   # now targets the chosen instance
```

Trace:

1. `vmm ls` calls `discover_vmm_instances()` (`vmm-cli.py:79–105`), which iterates `/run/user/<uid>/dstack-vmm/*.json` for **every** UID present in `/run/user`, parses each registration record (written by the VMM via `vmm/src/discovery.rs:59–97`), and filters out any whose PID no longer exists in `/proc` (`vmm-cli.py:96–97`). Each entry's `username` is resolved via `pwd.getpwuid()` (`vmm-cli.py:53–55`). The active instance (from `~/.dstack-vmm/config.json`) is marked with `*` (`vmm-cli.py:212`).
2. `vmm switch` does prefix-matching across the same set, then writes `active_vmm` into `~/.dstack-vmm/config.json` via `save_active_vmm()` (`vmm-cli.py:168–174, 254–257`).
3. Subsequent commands fall through `resolve_vmm_url()` (`vmm-cli.py:108–145`): explicit `--url` > `DSTACK_VMM_URL` > `config["url"]` > `active_vmm` → matched discovery record's address > sole-discovered-instance > `http://localhost:8080`.

**The host-wide visibility is intentional**, per `vmm-cli.py:39–40`'s comment: *"Each user's instances are in $XDG_RUNTIME_DIR/dstack-vmm. CLI scans all users' directories so operators can see every instance on the host."* This is a single-operator host model; in a multi-tenant rewrite it must be replaced (see §8).

---

## 5. Authentication & TLS

### 5.1 What the CLI sends

`VmmClient.request()` at `vmm-cli.py:413–419`:

```python
if self.auth_user and self.auth_password:
    credentials = f"{self.auth_user}:{self.auth_password}"
    encoded_credentials = base64.b64encode(credentials.encode("utf-8")).decode("ascii")
    headers["Authorization"] = f"Basic {encoded_credentials}"
```

Credentials come from (priority high → low) `--auth-user`/`--auth-password` CLI flags → `DSTACK_VMM_AUTH_USER`/`DSTACK_VMM_AUTH_PASSWORD` env vars → `auth_user`/`auth_password` in `~/.dstack-vmm/config.json` (`vmm-cli.py:1518–1521`).

### 5.2 What the server expects

The server's only auth guard is `rocket-apitoken::Authorized` (`vmm/src/main_routes.rs:13, 109`). Its implementation (`~/.asdf/installs/rust/1.84.0/git/checkouts/rocket-apitoken-bd4044808b9d81fc/7b5c1e8/src/lib.rs:50–101`) does:

```rust
pub fn new(tokens: Vec<String>, enabled: bool) -> Self {
    Self {
        tokens: tokens.into_iter().map(|t| format!("Bearer {}", t)).collect(),
        enabled,
    }
}
// ...
match request.headers().get_one("Authorization") {
    Some(value) => {
        if token.tokens.contains(value) { Outcome::Success(Authorized) }
        else { Outcome::Error((Status::Unauthorized, "invalid token")) }
    }
    _ => Outcome::Error((Status::Unauthorized, "Authorization header not found")),
}
```

The full string `"Bearer <token>"` is compared against the incoming `Authorization` header. A header that begins with `Basic ` literally cannot match.

### 5.3 Net effect on the deployed system

- **Default config disables auth completely.** `vmm/vmm.toml:126–128` ships with `[auth] enabled = false` and `tokens = []`, so even `/logs` is unguarded.
- **Even if auth is enabled**, only `/logs` carries the `Authorized` guard (`vmm/src/main_routes.rs:107–116`). The `/prpc/*`, `/guest/*`, and `/api/*` mounts (`vmm/src/main.rs:82–88`) use `ra_rpc::prpc_routes!` which generates routes without a `FromRequest` auth guard (see `ra-rpc/src/rocket_helper.rs:212–270`). Lifecycle, deploy, env-update, signed-key fetch — all uncredentialed.
- **The CLI's Basic header is therefore decorative.** It will never match Bearer; and in the only place it could be checked (`/logs` with auth enabled), the CLI's command would fail with 401, never succeeding.
- **Tutorials in the repository document Basic auth** (`docs/tutorials/attestation-verification.md:138, 185, 385, 573` use `curl -u "admin:..."`). Those examples likewise can't authenticate against a Bearer-only server; they only work because the actual deployments have `auth.enabled = false`.

### 5.4 TLS

`VmmClient.request()` at `vmm-cli.py:431–436`:

```python
if self.is_https:
    # TODO: we should verify TLS cert.
    context = ssl.create_default_context()
    context.check_hostname = False
    context.verify_mode = ssl.CERT_NONE
    conn = http.client.HTTPSConnection(self.host, context=context)
```

Hostname and chain verification are off. Combined with a cleartext (no auth) RPC body, any network-positioned attacker can take over a CVM by intercepting `CreateVm` / `UpgradeApp` calls.

---

## 6. Config and State on the User's Machine

| Path | Format | Written by | Read by | Notes |
|---|---|---|---|---|
| `~/.dstack-vmm/config.json` | JSON object: `{ url, auth_user, auth_password, active_vmm }` | `save_active_vmm()` (`vmm-cli.py:168–174`); manual edits otherwise | `load_config()` (`vmm-cli.py:62–76`); `main()` (`vmm-cli.py:1509–1521`) | Stores cleartext password if any; per-user (no separation between operators sharing the same UID). |
| `~/.dstack-vmm/kms-whitelist.json` | JSON: `{ "trusted_signers": [<33-byte hex compressed pubkey>, …] }` | `save_whitelist()` (`vmm-cli.py:1492–1501`) | `load_whitelist()` (`vmm-cli.py:1473–1489`); KMS pubkey verification (`vmm-cli.py:713–724`) | Empty whitelist → CLI prompts the operator to confirm any signer it sees. Per-user. |
| `/run/user/<uid>/dstack-vmm/*.json` (one file per running VMM) | JSON: `{ id, pid, address, working_dir, config_file, image_path, run_path, node_name, version, started_at }` | VMM's `discovery::DiscoveryRegistration::register()` (`vmm/src/discovery.rs:59–97`) | CLI's `_get_discovery_dirs()` + `discover_vmm_instances()` (`vmm-cli.py:41–105`) | Files persist for the life of the VMM process; cleaned via `Drop` (`vmm/src/discovery.rs:100`) and `cleanup_stale_registrations()` on next start. CLI scans **all UIDs**, not just its own. |

**Configuration source precedence** for `--url`, `--auth-user`, `--auth-password`:

1. CLI flag (`--url …`) — `vmm-cli.py:1523–1539`.
2. Environment variable (`DSTACK_VMM_URL`, `DSTACK_VMM_AUTH_USER`, `DSTACK_VMM_AUTH_PASSWORD`) — `vmm-cli.py:1515–1521`.
3. `~/.dstack-vmm/config.json` (`url`, `auth_user`, `auth_password`) — `vmm-cli.py:1509, 1516, 1518, 1520`.
4. For URL only: discovery fallback (single-instance auto-pick or `active_vmm`) — `vmm-cli.py:135–144`.
5. For URL only: hardcoded `http://localhost:8080` — `vmm-cli.py:145, 1516`.

**Filesystem permissions are not enforced.** The CLI calls `os.makedirs(..., exist_ok=True)` and `open(..., "w")` without setting restrictive modes (`vmm-cli.py:172–174, 1481, 1499–1501`); the resulting files inherit the caller's umask.

---

## 7. CLI vs Web UI Capability Matrix

The web UI lives at `vmm/ui/` (Vue + protobuf-generated `vmmRpc` client at `vmm/ui/src/composables/useVmManager.ts:6, 39–41`).

| Capability | CLI (file:line) | Web UI (file:line) |
|---|---|---|
| List VMs (`Status`) | `lsvm` (`vmm-cli.py:507`) | `useVmManager.ts:519, 525, 565` |
| VM details (`GetInfo`) | `info` (`vmm-cli.py:1180`) | `useVmManager.ts:565` (folded into status refresh) |
| Start (`StartVm`) | `start` (`vmm-cli.py:569`) | `useVmManager.ts:1344` |
| Graceful stop (`ShutdownVm`) | `stop` (`vmm-cli.py:578`) | `useVmManager.ts:1353, 1384` |
| Force stop (`StopVm`) | `stop -f` (`vmm-cli.py:575`) | `useVmManager.ts:1368, 1384` |
| Remove (`RemoveVm`) | `remove` (`vmm-cli.py:583`) | `useVmManager.ts:1391` |
| Create (`CreateVm`) | `deploy` (`vmm-cli.py:905`) | `useVmManager.ts:990, 1169` |
| Resize (`ResizeVm`) | `resize`, `update --vcpu/--memory/...` (`vmm-cli.py:610, 1030`) | **Absent.** No `resizeVm` reference anywhere in `vmm/ui/src`. |
| Upgrade app (`UpgradeApp`) | `update-env`, `update-app-compose`, `update-user-config`, `update-ports`, `update` (`vmm-cli.py:963, 968, 973, 980, 1171`) | `useVmManager.ts:1068` (`vmmRpc.updateVm`) |
| Image list (`ListImages`) | `lsimage` (`vmm-cli.py:649`) | `useVmManager.ts:601` |
| GPU list (`ListGpus`) | `lsgpu` (`vmm-cli.py:1226`) | `useVmManager.ts:610` |
| `GetAppEnvEncryptPubKey` + signature verify + whitelist | `deploy`, `update-env`, `update` (`vmm-cli.py:665–726`) | `useVmManager.ts:926` (encrypts envs but does **not** verify the KMS signature or consult any whitelist) |
| Compose-file generation (`compose` builder) | `compose` (`vmm-cli.py:782–829`) | Inline in dialog state (`useVmManager.ts:712–767`) — different code path, different result |
| KMS signer whitelist (`kms list`/`add`/`remove`) | `kms ...` (`vmm-cli.py:733–775`) | **Absent.** |
| VMM-instance discovery (`vmm ls`/`switch`) | `vmm ls`/`vmm switch` (`vmm-cli.py:177–257`) | **Absent.** UI is bound to whichever VMM serves it. |
| Server `Version` RPC | **Not called.** | `useVmManager.ts:626` |
| `ReloadVms` RPC | **Not called.** | `useVmManager.ts:1288` |
| Registry: `ListRegistryImages`, `PullRegistryImage`, `DeleteImage` | **Not called.** | `useVmManager.ts:1490, 1514, 1527` |
| Supervisor: `SvList`, `SvStop`, `SvRemove` | **Not called.** | `useVmManager.ts:1576, 1601, 1611, 1622` |
| Live log streaming (`/logs`) | `logs [-f]` (`vmm-cli.py:615`) | Opens `/logs?id=…&follow=true` in a new window (`useVmManager.ts:1399`) |

**Asymmetry summary:**

- **CLI-only:** `resize`, the multi-aspect `update`, on-disk compose generation (`compose`), KMS signer whitelisting (`kms`), cross-VMM discovery (`vmm ls`/`vmm switch`).
- **UI-only:** `Version`, `ReloadVms`, registry image pull/delete, supervisor process listing, image deletion. The UI also signs more compose-file fields (storage_fs at `useVmManager.ts:741`, init_script at `:745`, etc.) than the CLI's `compose` builder produces.
- **Drift risk:** Two independent compose-file constructors, two independent env-encryption code paths, two independent UpgradeApp invocations. Any policy change (e.g. adding an `allowed_envs` allowlist, or rotating the key-derivation function) must be made in both places or behaviour will diverge.

---

## 8. Multi-Tenant Blockers (numbered, with file:line evidence)

These are the structural reasons today's CLI cannot be fitted into a multi-tenant control plane without redesign.

1. **No caller identity is sent on any RPC.** `VmmClient.request()` only adds an `Authorization` header (Basic) and a `Content-Type` header (`vmm-cli.py:410–425`). The body of `CreateVm`, `Status`, `GetInfo`, `UpgradeApp`, etc. carries no actor / org / tenant field. Server methods like `Status` therefore return *all* VMs unconditionally (`vmm/src/main_service.rs:353` — see also report 01 §"pRPC Service: Vmm").

2. **`Status` is server-wide.** `lsvm` calls `Status` (`vmm-cli.py:507`) and renders every row returned. There is no client-side filter and no server-side tenant filter; in a shared host every operator sees every VM.

3. **`GetInfo` / `RemoveVm` / `ShutdownVm` / `StopVm` / `StartVm` / `ResizeVm` / `UpgradeApp` accept a bare VM ID.** No org/tenant ownership check is performed — the call is sent (`vmm-cli.py:569, 575, 578, 583, 610, 919, 963`) and the server acts on whichever VM has that UUID. Any user with network reach to the VMM can issue a destructive op against another user's VM.

4. **App ID is `truncate40(sha256(compose_file))`.** Computed locally at `vmm-cli.py:777–780` and used by `deploy` / `update-env` / `update` for env-encryption key derivation (`vmm-cli.py:896–905, 919–924, 1091`). Two organizations that ship the same `app-compose.json` (which is plausible — common open-source images, identical defaults) will collide on app_id, and both will request encryption against the same KMS-derived public key. The KMS has no concept of tenant either, so this blends env-secret namespaces across orgs.

5. **Discovery is host-wide, with username resolution.** `_get_discovery_dirs()` walks `/run/user/<uid>/dstack-vmm/` for every UID (`vmm-cli.py:46–58`); `discover_vmm_instances()` reads each registration file and tags it with `pwd.getpwuid()` (`vmm-cli.py:53–55, 87–104`). The model is "single trusted operator account inspecting many local VMM processes". A multi-tenant control plane needs the inverse: a centralized scheduler that knows which CVMs each tenant owns, regardless of host.

6. **Auth model is misaligned and effectively absent.** Basic header sent (`vmm-cli.py:413–419`); Bearer-only check on the server (`rocket-apitoken/src/lib.rs:50–58, 91–101`); only `/logs` is gated (`vmm/src/main_routes.rs:109`); default config disables even that (`vmm/vmm.toml:127`). There is currently no CLI-side affordance to pass a Bearer token, so even a server with auth enabled cannot be authenticated by the CLI without code change.

7. **TLS verification is disabled.** `vmm-cli.py:431–436` accepts any cert / hostname. In a multi-tenant SaaS this is unacceptable for control-plane traffic — credentials and compose contents (which often contain secrets even before the env-file step) flow over this channel.

8. **State files are global per UID and unprotected.** `~/.dstack-vmm/config.json` and `~/.dstack-vmm/kms-whitelist.json` are written without explicit modes (`vmm-cli.py:172–174, 1481, 1499–1501`) and contain plaintext passwords (`vmm-cli.py:1518–1521`) and trust roots (`vmm-cli.py:733–775`). Any process running as that UID can read or tamper with them.

9. **Compose-hash whitelist enforcement is on the operator, not the system.** `compose` prints the SHA-256 of the generated JSON (`vmm-cli.py:825–829`); `update-env` / `update` recompute it implicitly. The on-chain `DstackApp.allowedComposeHashes` registry is the actual gate, but neither the CLI nor the server checks it before issuing `CreateVm` / `UpgradeApp`. This is a multi-tenant blocker because in a SaaS the platform — not the operator — needs to enforce the whitelist policy automatically.

10. **`update-env`'s "merge into compose" rewrites compose JSON in the operator's process, not server-side.** `vmm-cli.py:937–963` mutates the existing compose's `allowed_envs` and `launch_token_hash` then submits the new JSON via `UpgradeApp`. A multi-tenant control plane needs a server-side audit trail of *who* changed *which* fields when; the current model loses that information because the server only sees the final blob. Two operators in the same org would race over the contents.

11. **Per-VM logs unauthenticated by VM ownership.** `GET /logs?id=<id>` is the only auth-checked route (`vmm/src/main_routes.rs:109`), but the check is "is the request authenticated at all", not "is this principal allowed to read this VM's serial console". Any token-bearer can read any VM's logs.

12. **No abstraction for "the VMM I'm allowed to talk to".** `resolve_vmm_url()` (`vmm-cli.py:108–145`) auto-picks the only running instance, treats `/run/user/*/dstack-vmm/*.json` as a public registry, and falls back to `localhost:8080`. A multi-tenant client must instead be told "your control plane is at https://api.example.com; you have a credential" and never auto-discover.

---

## 9. Notes for the Rewrite

This section is an opinionated set of recommendations for how a CLI should evolve in the multi-tenant rewrite. Concrete decisions still belong to the synthesis pass.

### 9.1 Authentication and credential handling

- **Replace Basic with a real auth flow.** The most natural targets are: a session token from the new control-plane API (e.g. issued after OAuth/SIWE login), or an issued API key (Bearer). Either way, the CLI needs to send `Authorization: Bearer <jwt-or-opaque-token>` so it can be checked by a `FromRequest` guard on every privileged route, not just `/logs`.
- **Stop carrying passwords on disk.** The replacement for `~/.dstack-vmm/config.json` should store at most a refresh token + endpoint URL, with file mode 0600. A `dstack login` flow that obtains a short-lived access token is the standard pattern.
- **Profiles, not single config.** Multi-tenant operators often have access to multiple orgs. A CLI like `gh` / `aws` profile model fits naturally; `~/.dstack-vmm/config.json` becomes `~/.dstack-vmm/profiles.json` keyed by profile name, with `--profile <name>` flag and `DSTACK_PROFILE` env var.
- **TLS verification on by default**, with an `--insecure` opt-out for local development. Remove the unconditional `verify_mode = ssl.CERT_NONE` (`vmm-cli.py:431–436`).

### 9.2 Control-plane mode vs direct-VMM mode

- The new CLI should default to talking to a **control plane** (a multi-tenant orchestrator that fronts one or more VMM hosts), not to a VMM directly. The control plane resolves `vm_id` → owning host → backing VMM, performs authz, and forwards to the appropriate VMM.
- A `--direct-vmm` (or "host-admin") mode for operators who self-host can preserve the current direct-RPC behaviour. That mode needs the new auth scheme too — it is *not* a free pass to call uncredentialed RPCs.
- The `vmm ls` / `vmm switch` semantics map cleanly to **host operator mode only**. In control-plane mode there is no VMM list; there is an org list and a CVM list. Keep the discovery code, but gate it behind the host-admin profile.

### 9.3 Subcommand reshape

- **`compose` → server-side rendering.** Generating compose JSON locally guarantees drift between the CLI's builder and the UI's builder (see §7). Move compose construction to the control plane, expose it via a `RenderCompose` RPC, and have both clients render server-side.
- **`update-*` family → fold into a single `update`.** The standalone `update-env`, `update-app-compose`, `update-user-config`, `update-ports` commands all post `UpgradeApp` with one or two fields populated (`vmm-cli.py:963, 968, 973, 980`). The aggregated `update` (`vmm-cli.py:984–1176`) supersedes them, but only via flag-by-flag composition. A simpler rewrite: one `update` command, one server-side patch endpoint that accepts a JSON-merge-patch.
- **`kms list/add/remove` → per-org policy.** A signer whitelist is a security policy that should live in the org's settings, not in the CLI user's home directory. Keep the verification logic (`vmm-cli.py:1377–1470`) — it is sound — but bind the trust roots to the authenticated org.
- **`logs`** is the most reusable command and stays as-is, modulo per-VM authz.

### 9.4 App-ID and compose-hash flow

- Deriving `app_id` as `sha256(compose)[:40]` (`vmm-cli.py:777–780`) is wrong for multi-tenant. Move to either:
  - **Server-issued app_id** — the control plane assigns a UUID at first deploy, persists it, and uses it as the KMS lookup key (binding via signed delegation chain). This decouples app identity from compose contents.
  - Or **`app_id = sha256(org_id || compose)[:40]`** if the on-chain `DstackKms.appRegistry` semantics need to be preserved. The org_id is part of the authenticated session.
- Either way, the CLI should not be the source of truth for app_id. Have the server return it from `RegisterApp`/`CreateVm` and persist it in the local profile cache.

### 9.5 Coexistence and deprecation

- **Keep the old CLI working.** Existing tutorials, internal automation, and integration tests use it (e.g. `docs/tutorials/attestation-verification.md`, `docs/tutorials/hello-world-app.md`). Ship the new CLI as a separate binary (`dstack` or `dstack-cli`) and leave `vmm-cli.py` in place pinned to current behaviour.
- **Stage migration.** A reasonable order:
  1. Land the new control-plane API alongside the existing `/prpc` surface (auth-gated from day one).
  2. Ship the new CLI talking to the control plane only.
  3. Add a `dstack legacy-vmm` subcommand that wraps the old `vmm-cli.py` flows for users who self-host.
  4. Mark `vmm-cli.py` as deprecated in `docs/vmm-cli-user-guide.md` and the tool's `--help`. Do not remove until the SDKs and tutorials are updated.
- **Fix the documentation drift first.** `docs/vmm-cli-user-guide.md:77–78` and `docs/tutorials/*.md` describe Basic auth that does not work against a server with auth enabled. The simplest interim fix is a one-line note: "If the VMM has `[auth] enabled = true`, the CLI's `--auth-*` flags are non-functional — disable VMM auth or apply the patch in #X." This shouldn't ship as the rewrite's only fix, but it removes a footgun while the rewrite proceeds.

### 9.6 Things from the current CLI worth preserving

- The **KMS signature verification + whitelist** logic (`vmm-cli.py:1377–1501`) is the only place in the codebase that does end-to-end verification of the KMS-derived encryption key against a trust root. Port this faithfully into the new CLI; ideally promote it to a shared library that both the control plane and the CLI use.
- The **env-encryption envelope** (X25519+AES-256-GCM, `vmm-cli.py:260–319`) is straightforward and well-commented. Keep the wire format.
- The **multi-aspect `update` semantics** (single command, partial flags) (`vmm-cli.py:984–1176`) is the right UX for resource + app + ports updates. Keep the shape, change the underlying RPCs to a single patch endpoint.
- The **Unix-socket transport** support (`vmm-cli.py:349–363, 377–387, 428–429`) is useful for local administration. Keep it under the host-admin profile.

---

## Appendix: Minor issues observed (pre-existing, file:line)

- `is_https` truthiness check at `vmm-cli.py:431` works only because `parsed_url.scheme` is the lowercase string `"https"`; subtle but safe.
- `is_https` is set only when `use_uds` is False (`vmm-cli.py:381–387`), and only on first call (`__init__`); changing the URL after construction is not supported. Not a bug; minor architectural note for the rewrite.
- `parse_env_file()` (`vmm-cli.py:1278–1298`) does not unquote values, does not handle multi-line values, and silently drops malformed lines. This is fine for CI use but surprising in interactive use; a stricter parser belongs in the rewrite.
- `update_vm_env` writes `allowed_envs = list(envs.keys())` (`vmm-cli.py:946`) — this *replaces* the existing allowlist rather than merging. Operators who have run `update-env` once with a partial env-file will lose previously-allowed keys. Expected behaviour given the data model, but worth flagging.
- `parse_size()` rejects fractional results (`vmm-cli.py:1360–1363`). Not a bug; documents the assumption that memory and disk sizes are integer-valued in their target units.
- `confirm_untrusted_signer()` (`vmm-cli.py:728–731`) blocks on stdin even when invoked from automation. The rewrite should allow `--yes` / `--require-trusted-signer` flags.
