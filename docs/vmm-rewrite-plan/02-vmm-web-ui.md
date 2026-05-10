# dstack-vmm Web UI: Structure, Flows, and Multi-Tenant Blockers

**Prepared:** 2026-05-05
**Scope:** the v1 web console served by `dstack-vmm` — the source under `vmm/ui/`,
the embedded artifact, the Rocket routes that serve it, the RPCs it talks to,
and what blocks reusing it (or rebuilding it) for a multi-tenant control plane.

> This report is descriptive. Every claim has a `file:line` citation. Section 9
> ("Notes for the rewrite") is the only forward-looking part.

---

## 1. Executive summary

The dstack-vmm web UI is a Vue 3 single-page app, written as plain
`require()`-style TypeScript + global Vue (no `.vue` SFCs, no Vite, no real
bundler), compiled by a hand-rolled CommonJS-walker (`vmm/ui/build.mjs`) into a
**single self-contained `console_v1.html`** that gets `include_str!`-embedded
into the `dstack-vmm` binary at compile time
(`vmm/src/main_routes.rs:19`, `vmm/build.rs:18-63`).

There is **one page** (`vmm/ui/src/templates/app.html`) that doubles as a VM
list, expandable detail panel, deploy modal, update modal, clone-config modal,
image-registry modal, and supervisor-process modal. Logs open in a separate
browser window pointed at a Rocket SSE stream. State is held in **one
1700-line composable**, `useVmManager.ts`
(`vmm/ui/src/composables/useVmManager.ts:250-1726`), which is the entire
"controller" of the app.

From a security / multi-tenant perspective the UI is best understood as a
**thin admin console for a single trusted operator on the same machine**:

- It calls the server with **no authentication header** at all
  (`vmm/ui/src/lib/vmmRpcClient.ts:65-91`, `useVmManager.ts:499-513`).
- It assumes a single global VM fleet — `Status` returns *all* VMs and the UI
  paginates client-side over that single list (`vmm/src/app.rs:826-876`,
  `useVmManager.ts:517-550`).
- It has a "Dev Mode" toggle stored in `localStorage` that bypasses
  destructive-action confirmation prompts (`useVmManager.ts:47, 1273-1281,
  1360, 1376-1389`).
- The lone destructive surface that the *server* protects is the `/logs`
  streamer (`vmm/src/main_routes.rs:107-110`), and even that is gated only by
  a shared `Authorization: Bearer …` token that defaults to disabled
  (`vmm/vmm.toml:126-128`).
- Encrypted environment variables are sealed **client-side**, in the browser,
  using a public key fetched from the unauthenticated VMM and the signature
  on that key is **never verified by the UI**
  (`useVmManager.ts:916-958`, no `signature_v1` reference anywhere in the UI
  source).

For a multi-tenant rewrite this UI cannot be reused incrementally. Sections 8
and 9 spell out why and what the replacement must do differently.

---

## 2. Frontend stack & build pipeline

### 2.1 Source tree

```
vmm/ui/                                             ← the UI workspace
├── package.json                                    ← only dev deps: TS, protobufjs (vmm/ui/package.json:11-16)
├── tsconfig.json                                   ← `module: commonjs`, `target: ES2018` (vmm/ui/tsconfig.json:1-19)
├── build.mjs                                       ← custom bundler (vmm/ui/build.mjs:1-262)
├── scripts/build_proto.sh                          ← pbjs/pbts proto codegen (vmm/ui/scripts/build_proto.sh:1-40)
├── vendor/                                         ← optional spot for vendored vue.global.prod.js (vmm/ui/vendor/README.md:1-4); empty in tree
└── src/
    ├── index.html                                  ← shell page (vmm/ui/src/index.html:1-22)
    ├── main.ts                                     ← `createApp(App).mount('#app')` (vmm/ui/src/main.ts:1-9)
    ├── App.ts                                      ← Vue component wiring (vmm/ui/src/App.ts:1-29)
    ├── styles/main.css                             ← 1750 lines of CSS (vmm/ui/src/styles/main.css)
    ├── templates/app.html                          ← 605-line root template (vmm/ui/src/templates/app.html)
    ├── components/                                 ← 6 dialog / editor components
    │   ├── CreateVmDialog.ts                       (vmm/ui/src/components/CreateVmDialog.ts:1-191)
    │   ├── UpdateVmDialog.ts                       (vmm/ui/src/components/UpdateVmDialog.ts:1-152)
    │   ├── ForkVmDialog.ts                         (vmm/ui/src/components/ForkVmDialog.ts:1-55)
    │   ├── EncryptedEnvEditor.ts                   (vmm/ui/src/components/EncryptedEnvEditor.ts:1-163)
    │   ├── PortMappingEditor.ts                    (vmm/ui/src/components/PortMappingEditor.ts:1-132)
    │   └── GpuConfigEditor.ts                      (vmm/ui/src/components/GpuConfigEditor.ts:1-79)
    ├── composables/useVmManager.ts                 ← the entire controller (vmm/ui/src/composables/useVmManager.ts, 1726 lines)
    └── lib/
        ├── vmmRpcClient.ts                         ← protobuf RPC client (vmm/ui/src/lib/vmmRpcClient.ts:1-92)
        └── x25519.js                               ← bundled curve25519 implementation (vmm/ui/src/lib/x25519.js, 1672 lines)
```

The `App.ts` file (`vmm/ui/src/App.ts:13-27`) declares the root component with
inline registration of all six children and uses
`useVmManager()` as its `setup`.

### 2.2 Vue version & SFC story

- Vue 3, **global** build (`vmm/ui/src/index.html:17` references
  `../vendor/vue.global.prod.js`); the bundler attempts to inline a vendored
  copy and falls back to a pinned CDN URL `vue@3.4.21` if the vendor file is
  missing (`vmm/ui/build.mjs:180-195, 224-236`).
- **No `.vue` files anywhere in the repo.** Templates are string literals or
  raw HTML files imported as text via the bundler's `.html → module.exports =
  JSON.stringify(content)` rule (`vmm/ui/build.mjs:43-50`,
  `vmm/ui/src/App.ts:11`).
- TypeScript is transpiled by `tsc` to CommonJS-style `.js` in
  `vmm/ui/build/ts/` (`vmm/ui/build.mjs:159-163`,
  `vmm/ui/tsconfig.json:3-4`); modules use `require()` and `export =` (CJS
  interop), e.g. `vmm/ui/src/main.ts:7-9`,
  `vmm/ui/src/components/CreateVmDialog.ts:189-191`.
- `Vue` is referenced as a global (`declare const Vue: any` at
  `vmm/ui/src/main.ts:4`, `vmm/ui/src/composables/useVmManager.ts:4`,
  `vmm/ui/src/components/GpuConfigEditor.ts:4`).

### 2.3 The custom bundler

`vmm/ui/build.mjs` is a 262-line in-house bundler. Its main steps
(`vmm/ui/build.mjs:165-211`):

1. Run `scripts/build_proto.sh` (pbjs + pbts on `vmm_rpc.proto` and `prpc.proto`)
   to regenerate `src/proto/{vmm_rpc,prpc}.{js,d.ts}` (`vmm/ui/build.mjs:155-157`,
   `vmm/ui/scripts/build_proto.sh:22-37`).
2. `tsc --project tsconfig.json` (`vmm/ui/build.mjs:159-163`) outputs to
   `build/ts/`. The bundler then copies `src/templates/` into
   `build/ts/templates/` so the `require('./templates/app.html')` form resolves.
3. Walk the require graph from `build/ts/main.js`, collect modules into a
   serialized `(function(){ const modules = {…} … })()` IIFE
   (`vmm/ui/build.mjs:34-103`). It does **not** use esbuild, rollup, webpack, or
   Vite — it's a hand-written CJS resolver using `node:module.createRequire`.
4. Read `src/index.html` (`vmm/ui/src/index.html:1-22`), inline the linked CSS
   stylesheet (`vmm/ui/build.mjs:105-117`), inline (or fall back to CDN)
   `vue.global.prod.js` (`vmm/ui/build.mjs:180-195`), and inline the bundled JS
   (`vmm/ui/build.mjs:197-202`).
5. Write the resulting single HTML to `dist/index.html` and to either
   `$DSTACK_UI_OUT` or `../src/console_v1.html` (`vmm/ui/build.mjs:204-211`).

The watch mode (`vmm/ui/build.mjs:213-253`) prepends an SPDX header before
writing the watch output but the one-shot path does not.

### 2.4 Embedding into the binary

`vmm/build.rs:18-63` runs at `cargo build` time:

- Verifies `node` and `npm` are installed (`vmm/build.rs:27-32`); errors out
  with an install hint if not.
- Conditionally runs `npm ci` based on `package-lock.json` mtime vs.
  `node_modules/.package-lock.json` (`vmm/build.rs:34-42, 118-128`).
- Sets `DSTACK_UI_OUT=$OUT_DIR/console_v1.html` and runs `node build.mjs`
  (`vmm/build.rs:44-53`).
- Asserts the output exists (`vmm/build.rs:55-61`).

The Rust code then includes that file as a static string:

```rust
// vmm/src/main_routes.rs:19
const CONSOLE_V1: &str = include_str!(concat!(env!("OUT_DIR"), "/console_v1.html"));
```

So the v1 HTML is **regenerated on every cargo rebuild** of `dstack-vmm` and
**not** checked into the repo. Only the legacy `console_v0.html` is checked in
(`vmm/src/console_v0.html`, kept for the `/v0` route described in §3).

> Note: an earlier internal note placed `console_v1.html` at
> `vmm/src/console_v1.html`. That path does not exist in the repo
> (`find /home/fbx/dstack -name "console_v*.html"` finds only
> `vmm/src/console_v0.html`). The v1 artifact lives only in `OUT_DIR` after a
> build.

### 2.5 Title interpolation

The build leaves a `{{TITLE}}` placeholder in the template
(`vmm/ui/src/index.html:11`). Rocket replaces it per request based on
`config.node_name` (`vmm/src/main_routes.rs:31-43`).

---

## 3. Backend serving (Rocket routes)

All UI-related routes are defined in `vmm/src/main_routes.rs` and mounted on
the external listener at `vmm/src/main.rs:81-100`.

### 3.1 Route table

| Method | Path | Handler | File:line | Auth guard? |
|---|---|---|---|---|
| GET | `/` | `index` | `vmm/src/main_routes.rs:45-48` | No |
| GET | `/v1` | `v1` (delegates to `index`) | `vmm/src/main_routes.rs:50-53` | No |
| GET | `/beta` | `beta` (delegates to `index`) | `vmm/src/main_routes.rs:55-58` | No |
| GET | `/v0` | `v0` (legacy console from `console_v0.html`) | `vmm/src/main_routes.rs:60-63` | No |
| GET | `/res/<path>` | `res` (only allows `x25519.js`) | `vmm/src/main_routes.rs:65-74` | No |
| GET | `/logs?<id>&<follow>&<ansi>&<lines>&<ch>` | `vm_logs` (SSE-style text stream via `tailf`) | `vmm/src/main_routes.rs:107-183` | **Yes** — `_auth: Authorized` at line 109 |
| (mount) | `routes!` returns `[index, v1, beta, v0, res, vm_logs]` | | `vmm/src/main_routes.rs:185-187` | |

The console also opens `/api-docs/docs` in a new tab via
`useVmManager.ts:1253-1256`; that path is mounted by
`ra_rpc::rocket_helper::mount_openapi_docs` at `vmm/src/main.rs:101-102` and is
also unauthenticated.

### 3.2 Companion data routes the UI consumes

These are not "UI routes" but they are the only routes the UI ever calls:

| Mount | Defined at | Auth guard? | What the UI calls here |
|---|---|---|---|
| `POST /prpc/<method>` and `GET /prpc/<method>` | `vmm/src/main.rs:85-88`, generated by `prpc_routes!` macro at `ra-rpc/src/rocket_helper.rs:212-269` | **No** (`Authorized` is not a parameter of `prpc_post` / `prpc_get`, see `ra-rpc/src/rocket_helper.rs:236-251`) | `Status`, `ListImages`, `ListGpus`, `Version`, `CreateVm`, `UpdateVm`, `StartVm`, `StopVm`, `ShutdownVm`, `RemoveVm`, `GetAppEnvEncryptPubKey`, `ReloadVms`, `SvList`, `SvStop`, `SvRemove`, `ListRegistryImages`, `PullRegistryImage`, `DeleteImage` (see §4 and `vmm/rpc/proto/vmm_rpc.proto:300-360`) |
| `POST /guest/<method>` | `vmm/src/main.rs:83` | **No** (handler is `GuestApiHandler`, no `Authorized`, see `vmm/src/guest_api_service.rs:14-58`) | `NetworkInfo` (only — `useVmManager.ts:1194`) |
| `POST /api/<method>` | `vmm/src/main.rs:84` | **No**, but the `HostApiHandler::construct` rejects non-vsock callers with a `bail!` (`vmm/src/host_api_service.rs:21-33`), so external clients get a 400 from the handler rather than a 401 from a guard. The route remains *reachable* without auth. | The UI does not call this; the path is the same handler as the host listener. |

### 3.3 The lone authenticated route

`vm_logs` is the only Rocket handler that names `Authorized` as a request
guard:

```rust
// vmm/src/main_routes.rs:107-110
#[get("/logs?<id>&<follow>&<ansi>&<lines>&<ch>")]
fn vm_logs(
    _auth: Authorized,
    app: &State<App>,
    …
```

`Authorized` (`rocket-apitoken` v0.1, fetched at
`/home/fbx/.asdf/installs/rust/1.84.0/git/checkouts/rocket-apitoken-bd4044808b9d81fc/7b5c1e8/src/lib.rs:79-103`)
short-circuits to success when `ApiToken::enabled` is `false`, which is the
default (`vmm/vmm.toml:126-128`: `[auth] enabled = false; tokens = []`).

So in stock deployments the "protected" log streamer is also de facto open. In
a deployment with `enabled = true`, the *only* effect is that `/logs` requires
`Authorization: Bearer <token>`; every other route stays open.

### 3.4 Dynamic title and X-Headers

- The UI title combines `config.node_name` with `"dstack-vmm"`
  (`vmm/src/main_routes.rs:31-38`).
- Two response fairings in `vmm/src/main.rs:91-100` add `X-App-Version` and
  `X-Accel-Buffering: no` headers to every response — the latter to keep the
  log stream un-buffered through nginx-style proxies.

---

## 4. UI pages / views

There is one HTML route (`/`) and one SPA (`vmm/ui/src/templates/app.html`,
605 lines). What looks like multiple "pages" is really a stack of conditional
sections and modal overlays driven by reactive flags in `useVmManager.ts`.

### 4.1 Header / system menu

**Where:** `vmm/ui/src/templates/app.html:6-78` (top header). The hamburger
menu (`system-menu` dropdown, `app.html:25-75`) exposes:

- *Reload VMs* → `useVmManager.ts:1283-1320` → `vmmRpc.reloadVms({})` → calls
  `Vmm.ReloadVms` (`POST /prpc/ReloadVms`, `vmm/rpc/proto/vmm_rpc.proto:341`).
- *Images* → opens the Image Registry overlay (§4.6).
- *Processes* → opens the Supervisor Process Manager overlay (§4.7).
- *API Docs* → `useVmManager.ts:1253-1256` opens `/api-docs/docs`.
- *Legacy UI* → `useVmManager.ts:1258-1261` opens `/v0`.
- *Dev Mode toggle* → `useVmManager.ts:1273-1281`; flips
  `localStorage.devMode`. See §6 / §8 for what this disables.

### 4.2 VM list / dashboard

**Where:** `vmm/ui/src/templates/app.html:115-166` (toolbar + table header) and
`app.html:167-243` (one row per VM).

**Data:** `useVmManager.ts:517-550` (`loadVMList`) calls `vmmRpc.status({brief:
true, keyword, page, page_size})` and stores results in `vms.value`. The same
function runs in a 3-second polling loop:

```ts
// vmm/ui/src/composables/useVmManager.ts:1410-1419
async function watchVmList() {
  while (true) {
    try { await loadVMList(); } catch (error) { … }
    await new Promise((resolve) => setTimeout(resolve, 3000));
  }
}
```

**Server:** `Vmm.Status` → `RpcHandler::status` (`vmm/src/main_service.rs:353-355`)
→ `App::list_vms` (`vmm/src/app.rs:826-876`). `list_vms` returns *all* VMs in
the global state, with optional `keyword` substring matching against
`name`/`id`/`app_id`/`image` and `ids` allow-listing. **There is no caller /
owner / tenant filter** (see §8.1).

**Per-row controls** (`app.html:191-241`):

- Logs / Stderr / Board (link buttons): `useVmManager.ts:1398-1408`. Logs and
  Stderr open `/logs?id=…&follow=true&ansi=false&lines=200&ch={serial,stderr}`
  in a new window; Board opens `vm.app_url` (the gateway-facing URL the server
  computed).
- Three-dot dropdown: Start / Shutdown / Kill / Remove / Update / Clone Config
  → `vmmRpc.startVm`, `vmmRpc.shutdownVm`, `vmmRpc.stopVm`, `vmmRpc.removeVm`,
  open the Update modal, open the Clone modal
  (`useVmManager.ts:1342-1396`, `1035-1079`, `1094-1141`). `stopVm` and
  `removeVm` show a `confirm(...)` prompt **only when Dev Mode is off**
  (`useVmManager.ts:1360, 1362-1372, 1376-1389`).

### 4.3 Expandable VM detail row

**Where:** `vmm/ui/src/templates/app.html:244-459` (`v-if="expandedVMs.has(vm.id)"`).

Toggling expansion (`useVmManager.ts:1178-1188`, `toggleDetails`):

- Clears any other expanded VM (this is a single-row-at-a-time accordion).
- Calls `loadVMDetails(vmId)` → `vmmRpc.status({brief: false, ids: [vmId]})`
  to fetch the full configuration including `compose_file`
  (`useVmManager.ts:562-589`).
- Calls `refreshNetworkInfo(vm)` which uses the JSON-mode `?json` form of
  `guestRpcCall('NetworkInfo', { id: vm.id })` →
  `POST /guest/NetworkInfo?json` (`useVmManager.ts:1190-1197, 515`). This is
  the only `/guest/...` call the UI makes.

The detail panel renders: VM ID, Instance ID, App ID, image, vCPUs, memory,
swap, disk size, disk type, TEE on/off, attached GPUs, port mappings, feature
flags (`getVmFeatures`, `useVmManager.ts:1461-1476`), network interfaces with
RX/TX byte counts, optional WireGuard info, app compose file, user config, and
a row of log buttons (`app.html:286-457`). Any change to the VM's underlying
state requires another expand-and-refresh cycle plus the 3-second polled
list refresh.

### 4.4 Deploy form modal (`<create-vm-dialog>`)

**Where:** `vmm/ui/src/components/CreateVmDialog.ts:1-191`, mounted from
`app.html:80-91`.

**State:** `vmForm` ref in `useVmManager.ts:280` (initial shape at
`createVmFormState`, `useVmManager.ts:162-198`).

**Submission flow** (`useVmManager.ts:960-997`, `createVm`):

1. Build the app-compose JSON locally
   (`makeAppComposeFile`, `useVmManager.ts:712-768`). The compose hash is also
   computed locally for live preview (`composeHashPreview`,
   `useVmManager.ts:704-710, 797-821`).
2. If the key provider is `kms` and there are env vars, encrypt them
   client-side (`encryptEnv`, `useVmManager.ts:916-928`):
   - Compute `app_id = sha256(compose).slice(0, 40)` locally
     (`calcAppId`, `useVmManager.ts:911-914`).
   - `vmmRpc.getAppEnvEncryptPubKey({ app_id })` → `POST
     /prpc/GetAppEnvEncryptPubKey` (`vmm/rpc/proto/vmm_rpc.proto:326`,
     handler proxies to KMS, see `vmm/src/main_service.rs:455-468`).
   - Encrypt with X25519+AES-GCM in `encryptEnvWithKey`
     (`useVmManager.ts:930-958`). See §7 for security analysis.
3. Build the VM configuration (`buildCreateVmPayload`,
   `useVmManager.ts:442-464`).
4. `vmmRpc.createVm(payload)` → `POST /prpc/CreateVm`
   (`vmm/rpc/proto/vmm_rpc.proto:302`, handler at
   `vmm/src/main_service.rs:291-327`).

The dialog UI surfaces: name, image (from `Vmm.ListImages`, `useVmManager.ts:599-606`),
vCPUs, memory + unit, swap + unit, disk size, storage filesystem, optional
explicit `app_id`, docker compose file (paste or upload), init/pre-launch
scripts, user config, GPU selection (from `Vmm.ListGpus`,
`useVmManager.ts:608-623`), key provider (`none`/`kms`/`local`), networking
mode, feature checkboxes, encrypted envs editor, port mappings.

### 4.5 Update VM modal (`<update-vm-dialog>`)

**Where:** `vmm/ui/src/components/UpdateVmDialog.ts:1-152`, mounted from
`app.html:93-105`.

**State:** `updateDialog` ref initialized in
`useVmManager.ts:200-225, 287, 859-893`.

**Submission flow** (`useVmManager.ts:1035-1079`, `updateVM`):

- Diffs `vcpu`, `memory`, `disk_size`, `image` against the original VM and
  only sets the ones that changed (`useVmManager.ts:1045-1051`).
- If `resetSecrets` is on, fetches the env encryption pubkey
  *with the existing app_id* (`vm.app_id`) and re-encrypts the new env vars
  (`useVmManager.ts:1056-1060`).
- If `updateCompose` is on, rebuilds the compose JSON from the current
  app-compose with overrides for docker-compose, init script, pre-launch
  script, swap (`useVmManager.ts:770-795`, `makeUpdateComposeFile`).
- Sends `vmmRpc.updateVm(body)` → `POST /prpc/UpdateVm`
  (`vmm/rpc/proto/vmm_rpc.proto:312`, handler at
  `vmm/src/main_service.rs:377-453`). Note that `update_ports = true` is
  always set (`useVmManager.ts:1064`), so port mappings are rewritten on every
  update.

### 4.6 Clone-config modal

There is *no longer a separate "clone" dialog* used in the live flow. Two
artifacts remain:

- `cloneConfigDialog` state (`useVmManager.ts:227-248, 293`) and the
  `cloneConfig()` submit (`useVmManager.ts:1143-1176`) are still defined and
  the `<fork-vm-dialog>` is mounted (`app.html:107-113`).
- But the dropdown's "Clone Config" entry calls `showCloneConfig(vm)`
  (`useVmManager.ts:1094-1141`), which **populates `vmForm` and reuses the
  Create VM dialog** (`showCreateDialog.value = true` at
  `useVmManager.ts:1140`).

So in practice the user clones via the Deploy modal pre-filled with the
selected VM's config; the `ForkVmDialog` component
(`vmm/ui/src/components/ForkVmDialog.ts:1-55`, "Derive VM") is a vestigial
form that never opens.

### 4.7 Image registry modal

**Where:** `vmm/ui/src/templates/app.html:462-547` (`v-if="showImageRegistry"`
overlay).

**State + flow:** `useVmManager.ts:1478-1552`. When opened
(`openImageRegistry`, `useVmManager.ts:1535-1545`):

- `Vmm.ListImages` for local images and `Vmm.ListRegistryImages` for remote
  tags (`useVmManager.ts:1490`, `vmm/rpc/proto/vmm_rpc.proto:355`).
- Sets up `setInterval(loadRegistryImages, 3000)` polling
  (`useVmManager.ts:1538-1544`); cleared on close (`useVmManager.ts:1547-1552`).

**Actions:**

- `pullRegistryImage(tag)` → `Vmm.PullRegistryImage`
  (`useVmManager.ts:1506-1522`, `vmm/rpc/proto/vmm_rpc.proto:357`).
- `deleteImage(name)` → `Vmm.DeleteImage`
  (`useVmManager.ts:1524-1533`, `vmm/rpc/proto/vmm_rpc.proto:359`). Has a
  `confirm()` prompt regardless of Dev Mode.

### 4.8 Supervisor process manager modal

**Where:** `vmm/ui/src/templates/app.html:549-589` (`v-if="showProcessManager"`
overlay).

**State + flow:** `useVmManager.ts:1554-1628`. When opened
(`openProcessManager`, `useVmManager.ts:1586-1590`):

- `baseRpcCall('/prpc/SvList')` → `Vmm.SvList` (`vmm/rpc/proto/vmm_rpc.proto:348`).
- 3-second poll (`useVmManager.ts:1589`).

**Actions:**

- `svStop(id)` → `Vmm.SvStop` (`vmm/rpc/proto/vmm_rpc.proto:350`).
- `svRemove(id)` → `Vmm.SvRemove` (`vmm/rpc/proto/vmm_rpc.proto:352`).
- `svClear()` removes all stopped processes in a loop
  (`useVmManager.ts:1618-1628`).

This modal exposes **every supervised process on the host**, not just the
QEMU VMs — the supervisor also runs gateway/KMS/etc when configured to. There
is no scope filter (see §8.6).

### 4.9 Logs viewer (separate window)

`useVmManager.ts:1398-1400`:

```ts
function showLogs(id: string, channel: string) {
  window.open(`/logs?id=${encodeURIComponent(id)}&follow=true&ansi=false&lines=200&ch=${channel}`, '_blank');
}
```

Opens a new browser tab pointed at the Rocket text stream
(`vmm/src/main_routes.rs:107-183`). The browser renders the raw `text/plain`
stream (Rocket sets the content type via `TextStream`) — there is no
JS-driven viewer. Channels are `serial`, `stdout`, `stderr`
(`vmm/src/main_routes.rs:120-127`).

### 4.10 Toast / message strip

`vmm/ui/src/templates/app.html:591-605` renders three reactive strings —
`updateMessage`, `successMessage`, `errorMessage` — set throughout
`useVmManager.ts` (e.g. `1072` "Compose file updated", `1277` "Dev mode
enabled", `recordError` at `390-397`).

---

## 5. State management

### 5.1 The `useVmManager` composable

Single composable, 1726 lines (`vmm/ui/src/composables/useVmManager.ts`), used
exclusively as the `setup()` of the root component (`vmm/ui/src/App.ts:23-25`).

It holds **everything**:

- VM list + pagination state (`useVmManager.ts:251-262`).
- Form state for create / update / clone / GPU / image-registry / process
  manager (`useVmManager.ts:280-298, 478-1628`).
- Network info per VM (`useVmManager.ts:254`).
- Available images & GPUs (`useVmManager.ts:282-285`).
- Polling timers — three independent 3-second loops:
  - VM list (`useVmManager.ts:1410-1419`, infinite `setTimeout` chain).
  - Image registry (`useVmManager.ts:1538-1544`, `setInterval`).
  - Supervisor (`useVmManager.ts:1589`, `setInterval`).
- Local-storage-backed preferences: `pageSize` (`useVmManager.ts:258, 845-847`)
  and `devMode` (`useVmManager.ts:47, 1273-1281`).
- Compose-hash preview reactivity — two big `watch([…], …, { deep: true })`
  blocks (`useVmManager.ts:797-843`) that recompute the SHA-256 hash on every
  keystroke.
- Imperative DOM manipulation for dropdowns
  (`useVmManager.ts:1227-1247, 1322-1334`).
- Encryption logic (`useVmManager.ts:916-958`) — see §7.

The composable returns 90+ keys/functions (`useVmManager.ts:1637-1722`)
exposed directly to the template.

### 5.2 RPC client

`vmm/ui/src/lib/vmmRpcClient.ts:60-91` builds a singleton protobufjs client
that does:

```ts
fetch(`${basePath}/${methodName}`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/octet-stream' },
  body: payload as unknown as BodyInit,
  credentials: 'same-origin',
})
```

Notable:

- `basePath` defaults to `/prpc` (`vmm/ui/src/lib/vmmRpcClient.ts:60`).
- **No `Authorization` header is ever set.**
- `credentials: 'same-origin'` causes the browser to send any same-site
  cookies, but the server never sets any (see §6).
- Errors are decoded from `prpc.PrpcError` if the body is binary protobuf, or
  fall back to the raw text (`vmmRpcClient.ts:11-28`).

### 5.3 Secondary "JSON RPC" path

`useVmManager.ts:495-515` defines `baseRpcCall(pathname, params)` that uses
JSON instead of binary protobuf:

```ts
async function baseRpcCall(pathname: string, params: Record<string, unknown> = {}) {
  const response = await fetch(makeBaseUrl(pathname), {  // ?json
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(params),
  });
  …
}
const guestRpcCall = (method, params) => baseRpcCall(`/guest/${method}`, params);
```

Used for `Vmm.SvList / SvStop / SvRemove` and the `/guest/NetworkInfo` call
(`useVmManager.ts:1576-1622, 1194`). Same `?json` pathway is enabled by
`ra-rpc/src/rocket_helper.rs:204-208, 371` (server interprets the `?json`
query string as a JSON-mode request).

Like the protobuf client, it sets **no auth headers**.

### 5.4 No state library

There is no Pinia, Vuex, RxJS, or fetch wrapper layer beyond the two helpers
above. Reactive state lives directly in `ref(...)`s inside `useVmManager`.
Components communicate via `props` + `emit('close' | 'submit' | 'load-compose'
| 'update:gpus' | 'update:attachAll')` only (see component definitions in
`vmm/ui/src/components/`).

---

## 6. Auth model from the UI's perspective

### 6.1 What the UI sends

**Nothing.** The UI never:

- Sets `Authorization: Bearer …` (no `Authorization` reference in
  `vmm/ui/src/lib/vmmRpcClient.ts:60-91` or in `baseRpcCall` at
  `useVmManager.ts:499-513`).
- Reads or writes a token from `localStorage` / `sessionStorage` (the only
  `localStorage` reads/writes are `pageSize` and `devMode`,
  `useVmManager.ts:258, 47, 845-847, 1275`).
- Asks the user for credentials. There is **no login screen**, no
  username/password field, no "API key" field, no SIWE button, no OAuth
  button anywhere in `vmm/ui/src/`.
- Sets `credentials: 'include'` (it uses `'same-origin'`,
  `vmmRpcClient.ts:74`); same-site cookies would flow, but none are issued.

### 6.2 What the server requires

From §3 and the audit report at `docs/vmm-rewrite-plan/06-vmm-auth-today-and-gaps.md:91-113`:

- Default `vmm.toml`: `[auth] enabled = false; tokens = []`
  (`vmm/vmm.toml:126-128`).
- Wired in `main.rs:208`:
  `ApiToken::new(config.auth.tokens.clone(), config.auth.enabled)`.
- The `Authorized` request guard from
  `rocket-apitoken/.../lib.rs:79-103` short-circuits on `enabled == false`.
- Only `/logs` carries `_auth: Authorized` (`vmm/src/main_routes.rs:107-110`).
- Therefore, regardless of `enabled`, the entire web UI surface (HTML and
  every JSON / protobuf RPC the UI makes) is **unauthenticated**.

### 6.3 What this means for the UI as it stands

- A reachable web client (anyone who can reach `:9080`) is implicitly an
  admin.
- Even with `enabled = true` and bearer tokens configured, the UI itself
  doesn't know what to do with a token — it has no UI to enter one and never
  attaches one to outgoing requests.
- The `vmm-cli.py` companion sends HTTP **Basic** (`vmm/vmm-cli.py:413-419`,
  per the auth report at `docs/vmm-rewrite-plan/06-vmm-auth-today-and-gaps.md:124-125`),
  which the server doesn't parse. There is no working auth path for the CLI
  against `[auth] enabled = true` either, except for `/logs` which would
  always 401 from the CLI.
- Operators rely on **network reachability** as the auth boundary
  (`docs/deployment.md:139` instructs binding `tcp:0.0.0.0:9080` —
  i.e., all interfaces).

---

## 7. Client-side cryptography concerns

### 7.1 The flow

`useVmManager.ts:916-958`, called from `createVm` and `updateVM`:

```ts
async function encryptEnv(envs, kmsEnabled, appId) {
  if (!kmsEnabled || envs.length === 0) return undefined;
  let appIdToUse = appId;
  if (!appIdToUse) {
    const appCompose = await makeAppComposeFile();
    appIdToUse = await calcAppId(appCompose);                      // sha256(compose).slice(0, 40)
  }
  const keyBytes = hexToBytes(appIdToUse);
  const response = await vmmRpc.getAppEnvEncryptPubKey({ app_id: keyBytes });
  return encryptEnvWithKey(envs, response.public_key);
}

async function encryptEnvWithKey(envs, publicKeyBytes) {
  const envsJson = JSON.stringify({ env: envs });
  const remotePubkey = publicKeyBytes && publicKeyBytes.length ? publicKeyBytes : new Uint8Array();

  const seed = crypto.getRandomValues(new Uint8Array(32));
  const keyPair = x25519.generateKeyPair(seed);                    // ephemeral key in browser
  const shared = x25519.sharedKey(keyPair.private, remotePubkey);  // ECDH

  const importedShared = await crypto.subtle.importKey('raw', shared,
                              { name: 'AES-GCM', length: 256 }, true, ['encrypt']);
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encrypted = await crypto.subtle.encrypt({ name: 'AES-GCM', iv },
                              importedShared, new TextEncoder().encode(envsJson));
  // layout: ephemeral_pub(32) || iv(12) || ciphertext+tag
  …
}
```

The format matches `docs/encrypted-env-spec.md:84-93`.

### 7.2 What's correct

- The ephemeral X25519 key is generated with `crypto.getRandomValues(32)` and
  is fresh per call (`useVmManager.ts:934-935`).
- The shared secret is fed straight into AES-256-GCM with a fresh 12-byte IV.
- Plaintext is the entire `{env: [...]}` JSON object, so once encrypted only
  the in-TEE recipient can read it.

### 7.3 What's not verified

The `PublicKeyResponse` returned by the VMM proxy includes a `signature_v1`
field with a `secp256k1` signature over `Keccak256("dstack-env-encrypt-pubkey"
+ ":" + app_id + timestamp_be_bytes + public_key)` from the KMS root key
(`docs/encrypted-env-spec.md:56-81`,
`vmm/rpc/proto/vmm_rpc.proto` PublicKeyResponse fields, returned by the
proxy at `vmm/src/main_service.rs:455-468`).

The UI **does not verify it**:

```
Grep "signature|signature_v1|verify|secp256k1" vmm/ui/src/composables/useVmManager.ts
→ No matches found
```

This is the critical gap. The UI does:

```
GetAppEnvEncryptPubKey → trust the bytes → encrypt with them
```

There is no verification that the public key actually came from a known KMS
root, nor that the response is recent (the `timestamp` field exists for
replay-resistance and is also ignored).

### 7.4 Concrete attack scenarios (multi-tenant relevance)

A reachable VMM is a key-exchange oracle for the deployer's secrets:

1. **In-flight tampering.** Anyone able to MITM the unauthenticated `/prpc`
   path (recall §6: the typical deployment is plaintext HTTP on
   `tcp:0.0.0.0:9080`, see `docs/deployment.md:139`) can substitute their own
   X25519 public key. The browser will encrypt env vars to the attacker's
   key. The CVM will fail to decrypt, but the attacker has a copy of the
   plaintext.
2. **Compromised VMM.** Even under TLS, a compromised or malicious VMM can
   substitute a key. RA-TLS pinning between VMM and KMS is also missing
   (`vmm/src/app.rs:1007-1014`, `tls_no_check = true`; analysis at
   `docs/vmm-rewrite-plan/06-vmm-auth-today-and-gaps.md:141-155`), so the VMM
   itself receives an unverified key from KMS.
3. **Stale key replay.** If a KMS rotates the root, the UI has no way to
   notice — `timestamp` is unused.

For a multi-tenant rewrite this matters even more: an org's secrets must not
be sealable against another org's KMS, and the UI is the right place to do
the verification. Today the UI is permissive by design; a rewrite should
treat the public key as untrusted-until-verified.

---

## 8. Multi-tenant blockers (numbered, with file:line evidence)

Each item is something the new system has to solve before this UI (or its
successor) can be shown to more than one principal.

1. **No authentication on any UI request.** The browser sends no auth header
   (`vmm/ui/src/lib/vmmRpcClient.ts:60-91`, `useVmManager.ts:499-513`). The
   server enforces `Authorized` only on `/logs`
   (`vmm/src/main_routes.rs:107-110`) and even that is disabled by default
   (`vmm/vmm.toml:126-128`). All `/prpc/*`, `/guest/*`, `/api/*`, and
   `/api-docs/*` are open (`vmm/src/main.rs:81-100`,
   `ra-rpc/src/rocket_helper.rs:212-269`).

2. **Single global VM list, no owner field.** `Vmm.Status` returns *all* VMs
   in process state without filtering by caller (`vmm/src/app.rs:826-876`,
   particularly the `iter_vms()` call at line 838). The UI paginates this
   single list (`useVmManager.ts:517-550`). The persisted manifest
   (`vmm/src/app.rs:48-73`) has `id`, `name`, `app_id`, `created_at_ms`,
   `image`, etc., but **no `owner_id` / `org_id` / `created_by` field**. There
   is nothing for a server-side filter to key on.

3. **Any reachable client can stop or remove any VM.** `removeVm` and
   `stopVm` only check Dev Mode locally (`useVmManager.ts:1362-1396`); the
   server-side handlers `RpcHandler::stop_vm` / `remove_vm`
   (`vmm/src/main_service.rs:337-351`) accept any caller and forward to
   `App::stop_vm` / `remove_vm` (`vmm/src/app.rs:284-291`). No identity is
   carried into these methods (`Result<...>` signatures take just the VM
   ID).

4. **Supervisor-process modal exposes the entire host's supervised
   processes.** The Process Manager overlay calls `Vmm.SvList` /
   `SvStop` / `SvRemove` (`useVmManager.ts:1574-1628`,
   `vmm/src/main_service.rs:584-617`). The supervisor is host-wide
   (`SupervisorClient::start_and_connect_uds` at `vmm/src/main.rs:209-222`),
   so this view leaks every process — including, e.g., the gateway and KMS
   if they run under the same supervisor — to anyone who opens the page.

5. **Image registry is global.** `Vmm.ListImages` /
   `Vmm.ListRegistryImages` / `PullRegistryImage` / `DeleteImage`
   (`vmm/src/main_service.rs:357-371, 619-695, 697-740`) all operate on the
   shared `config.image.path` directory. The UI surfaces every image
   (`useVmManager.ts:1490-1503`). One tenant deleting an image
   (`useVmManager.ts:1524-1533`) breaks every VM that uses it across all
   tenants. The deletion handler does check that no VM uses the image
   (`vmm/src/main_service.rs:670-682`), but it cannot scope by caller.

6. **GPU pool is global.** `Vmm.ListGpus` (`vmm/src/main_service.rs:556-563`)
   exposes the entire host's GPU inventory and `is_free` flags
   (`useVmManager.ts:608-623`). A multi-tenant scheduler needs per-org GPU
   quotas and visibility filtering; today the UI lets anyone pick any free
   slot, including `attach_mode: 'all'` which the server rejects only via
   `cvm.gpu.allow_attach_all` (`vmm/src/main_service.rs:78-83`).

7. **Compose hash is computed and trusted client-side for the app_id.** The
   browser computes `app_id = sha256(compose).slice(0, 40)`
   (`useVmManager.ts:911-914`), then asks the VMM for the corresponding
   encryption pubkey (`useVmManager.ts:921-926`). A multi-tenant model needs
   this binding to be enforced server-side per org (today there is no `org`
   to bind to).

8. **No verification of the env-encryption pubkey signature.** §7. The UI
   does not check `signature_v1` or `timestamp`
   (`useVmManager.ts:916-958`, no `signature` reference anywhere in the UI).
   In a multi-tenant world this is exploitable by anyone in the trust path
   (operator, network position, compromised VMM).

9. **Logs route is global by VM ID.** `/logs?id=<vmId>`
   (`vmm/src/main_routes.rs:107-127`) reads `app.work_dir(&id)`. Anyone with
   a VM ID can stream that VM's serial / stdout / stderr. The Status RPC
   already hands out every VM's ID (point 2). Coupled with the default
   `auth.enabled = false`, log streams are world-readable.

10. **`Reload VMs` is a host-wide button.** The header dropdown's "Reload
    VMs" calls `Vmm.ReloadVms` (`useVmManager.ts:1283-1320`,
    `vmm/src/main_service.rs:574-577`) which re-scans the entire VM
    directory. There is no scope.

11. **Dev Mode toggle silently bypasses confirmations.** A `localStorage`
    flag (`useVmManager.ts:47, 1273-1281`) gates the only safety prompt
    before destructive actions (`useVmManager.ts:1360, 1376-1379`). It also
    *silently force-stops a running VM before removing it* if Dev Mode is on
    (`useVmManager.ts:1382-1388`). For a shared UI, this is a per-browser
    policy override that no one else can see.

12. **No CSRF / origin / SameSite protection.** No `cors|csrf|rate.?limit`
    matches in `vmm/src/` (per `06-vmm-auth-today-and-gaps.md:124`); the UI
    doesn't read or set any cookies; the proto `Content-Type:
    application/octet-stream` is browser-sendable cross-origin without
    preflight under classic SOP rules. If auth ever lands as a cookie
    session, CSRF protection has to be designed in from scratch.

13. **The legacy `/v0` console is also unguarded.** `console_v0.html` is
    served at `/v0` (`vmm/src/main_routes.rs:60-63`); `/res/x25519.js`
    backs its env-encryption flow (`vmm/src/main_routes.rs:65-74`,
    `vmm/src/x25519.js`). Until removed, it's another door to the same
    open backend.

---

## 9. Notes for the rewrite

The current UI is shaped as "one reactive composable wrapping a single global
fleet, with no auth, served as a self-contained HTML embed." A multi-tenant
rewrite has to invert most of those assumptions. Concretely:

### 9.1 Auth flow

- **Login screen.** A real one. The UI must support at least one of:
  email+password, OAuth (GitHub/Google), or SIWE (sign-in-with-Ethereum). The
  rewrite plan's open framing question §4 of `00-overview.md:59-62` lists
  these explicitly.
- **Bearer token attached to every request.** The protobuf client at
  `vmmRpcClient.ts:60-91` and the JSON helper at `useVmManager.ts:499-513`
  must thread an `Authorization: Bearer …` (or
  short-lived JWT cookie) through every fetch.
- **Server-side enforcement on every route.** Replace the
  `prpc_routes!`-generated raw handlers
  (`ra-rpc/src/rocket_helper.rs:234-269`) with versions that take an auth
  guard analogous to `Authorized` but tied to a session / token / org. The
  current single-route guard pattern in `vmm/src/main_routes.rs:107-110` is
  the model; it just needs to be applied universally.
- **Session / refresh model.** None today; pick one
  (cookie+CSRF, stateless JWT, OIDC) and design with revocation.

### 9.2 Org / workspace switcher

- **Header surface.** Replace the title-only `app-header` block
  (`vmm/ui/src/templates/app.html:6-17`) with an org switcher dropdown +
  current-user widget on the right of the same bar. The system menu
  (`app.html:25-75`) becomes a per-user/per-org submenu.
- **Org context in URL.** The current SPA has no routing; everything is one
  page at `/`. Add at minimum `/orgs/:orgId/...` so links are shareable and
  the active org is bookmarkable.
- **Org-scoped data.** Every RPC needs an `org_id` request scope. `Vmm.Status`
  (`vmm/src/main_service.rs:353-355`) must filter by membership; image
  registry (`Vmm.ListImages` + `ListRegistryImages`,
  `vmm/src/main_service.rs:357-371, 619-662`) must be either per-org or
  shared-read with per-org policy on pulls/deletes.

### 9.3 Scoped views

- **VM list:** filter `Status` server-side by org membership (point §8.2).
  Pagination is already client-driven (`useVmManager.ts:517-550`); switch to
  cursor-based pagination keyed off the persistence layer once VMs carry
  owner ids.
- **Detail panel:** drop fields that leak host context where they don't
  belong (e.g., supervisor process IDs and host gateway settings should not
  necessarily be visible to a non-admin org member).
- **Logs:** require server-side authorization that the caller is an org
  member with read access to the VM ID. Today's `vm_logs`
  (`vmm/src/main_routes.rs:107-127`) needs an org-aware extension of
  `Authorized`.
- **Supervisor / image registry / GPU pool:** these are *host-admin* surfaces,
  not org surfaces. Move them out of the per-org UI and into a
  separate "host operator" console that requires a stronger role.

### 9.4 Crypto-correctness in the UI

- **Verify the env-encryption pubkey before encrypting.** §7. The UI must
  fetch the KMS root pubkey out-of-band (a config endpoint or a pinned
  baked-in value, not via the same untrusted VMM proxy) and verify
  `signature_v1` over `(app_id || timestamp || public_key)` per
  `docs/encrypted-env-spec.md:78-81`.
- **Reject stale timestamps.** Define a TTL for `PublicKeyResponse.timestamp`
  and surface a clear error if it's expired.
- **Move the X25519 + AES-GCM helper out of `useVmManager`.** Today
  `encryptEnvWithKey` lives at `useVmManager.ts:930-958` mixed with view
  state. The rewrite should isolate the crypto into a tested module (the
  vendored `x25519.js` at `vmm/ui/src/lib/x25519.js` is fine; a typed wrapper
  is the boundary).

### 9.5 State / framework choice

- **Replace the 1700-line composable.** The single `useVmManager` shape
  (`useVmManager.ts:250-1722`) is too big to test or reason about. Per-page
  composables, plus a Pinia (or equivalent) store, would map cleanly to the
  pages identified in §4. Each polling loop (VMs, registry, supervisor)
  belongs in its own composable with an explicit start/stop lifecycle, not
  three parallel `setInterval`s in one function.
- **Adopt SFCs and a real bundler.** The custom `build.mjs`
  (`vmm/ui/build.mjs:1-262`) ships nothing the rewrite needs that Vite
  can't. Switching to `.vue` SFCs unlocks proper TS in templates,
  hot-reload, and tree-shaking. Vue 3 Composition API moves from
  `declare const Vue: any` (`useVmManager.ts:4`) to ESM imports.
- **Decouple from the binary.** The current `include_str!` shape
  (`vmm/src/main_routes.rs:19`) ties UI iteration to Rust rebuilds. A
  separate UI artifact served by a small static-file route (or by a
  reverse proxy in front) would let the UI version independently and unblock
  multi-tenant features that do not require backend changes.

### 9.6 Audit & "who did what"

- **No audit trail today.** §1.7 of
  `docs/vmm-rewrite-plan/06-vmm-auth-today-and-gaps.md:207-219`. The UI must
  emit a stable `actor` identifier on every mutating call so the new server
  can write `(actor, org, action, resource, timestamp)` rows. The current
  `tracing::instrument` in `ra-rpc/src/rocket_helper.rs:235, 254` only
  records `(method, request_id)`; this is unrelated to audit.
- **Surface audit in the UI.** Per-org "Activity" view that reads the new
  audit log; particularly important for destructive surfaces (Stop/Kill/
  Remove, image delete, supervisor remove).

### 9.7 Smaller, but worth fixing in the rewrite

- The `<fork-vm-dialog>` mount (`app.html:107-113`) is dead UI; either remove
  or wire to the existing `cloneConfig()` (`useVmManager.ts:1143-1176`).
- `update_ports = true` is set unconditionally in
  `useVmManager.ts:1064`; intent should be derived from form state diffs.
- The `trim: "Teepod."` argument at
  `vmm/src/main.rs:87` is a leftover from the project's previous name; the
  proto package is `Vmm` (`vmm/rpc/proto/vmm_rpc.proto:300`) so the trim is
  effectively a no-op today, but leaves a stale reference in the wire
  contract.
- The legacy `/v0` console (`vmm/src/main_routes.rs:60-63`) and the
  `/res/x25519.js` route (`vmm/src/main_routes.rs:65-74`,
  `vmm/src/x25519.js`) should be retired as part of the rewrite, not
  carried forward.
