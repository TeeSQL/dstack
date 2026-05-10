# 10 — Multi-tenant control-plane design space

This is the design-space report. The brief is to figure out what a
multi-tenant control plane on top of (or in place of) `dstack-vmm` should
look like, with the user-facing shape of a typical cloud host
(DigitalOcean / Hetzner / Render): a person signs up, gets a personal
organization, can be invited into other organizations, and an
organization owns CVMs.

The goal here is **explicit option-mapping**, not a single forced
recommendation. Each section presents the realistic alternatives, the
tradeoffs, and a concrete recommendation that the synthesis pass in
`99-final-plan.md` can either take or override.

---

## Anchor: what the codebase currently is

Before getting to design, the relevant facts about today's vmm so the
"new vs keep" tradeoffs are concrete:

- **Single-host design.** `dstack-vmm` is one process per TDX host. It
  binds either to a Unix socket (`vmm.sock`, the default) or a TCP/HTTP
  port (e.g. `0.0.0.0:9080`). It owns a CID pool
  (`config.cvm.cid_start..cid_start + cid_pool_size`), a host port
  pool, a workdir per VM under `run_path/<vm_id>/`, and a list of guest
  images under `image.path`. It is *the* host-level operator process.
  See `vmm/src/main.rs` and `vmm/src/config.rs`.
- **No identity, no tenancy.** The auth model is a static API-token
  list (`[auth] tokens = [...]`) that can be turned off entirely
  (`auth.enabled = false`) — there is no concept of users, orgs,
  per-resource ACLs, or per-caller audit attribution. See
  `vmm/src/main_routes.rs::vm_logs` (just `_auth: Authorized`) and
  `vmm/vmm.toml`.
- **The RPC surface is global and flat.** `dstack_vmm_rpc::Vmm` exposes
  `CreateVm`, `StartVm`, `StopVm`, `RemoveVm`, `Status`, `ListImages`,
  `ListGpus`, `SvList`, `ReloadVms`, etc. No `org`/`workspace`
  parameter anywhere; every authenticated caller can manipulate every
  CVM on the host (`vmm/rpc/proto/vmm_rpc.proto`).
- **Per-VM identity is a UUID + an `app_id`.** The `app_id` is either
  `truncate40(sha256(compose_file))` for "register-only" mode or an
  on-chain Ethereum address (a deployed `DstackApp` proxy) when KMS is
  used (`vmm/src/main_service.rs::app_id_of`,
  `kms/auth-eth/contracts/DstackApp.sol`).
- **On-chain authority is a single EOA per app.** `DstackApp` inherits
  `OwnableUpgradeable`; the `owner()` is the only caller authorised to
  add/remove compose hashes, add/remove device IDs, toggle
  `allowAnyDevice` / `requireTcbUpToDate`, or upgrade the contract
  (`DstackApp.sol`). `DstackKms` ships a factory
  (`deployAndRegisterApp`) that takes an `initialOwner` — that's the
  hand-off to a human or wallet. There is no built-in role split,
  no co-signers, no per-action allowlist.
- **VM discovery is per-host and via a runtime-dir registry file.**
  `vmm/src/discovery.rs` writes `${XDG_RUNTIME_DIR}/dstack-vmm/<uuid>.json`
  on startup. There is no cluster-wide registry today.
- **The "host API" on vsock is for guests, not users.** `HostApiServer`
  on `vsock:2` serves CVM-side requests (`get_sealing_key`, `notify`).
  This is not the user-facing surface (`vmm/src/host_api_service.rs`).

These constraints define the problem: every multi-tenant feature below
has to be added in a layer that does *not* exist today. The interesting
choice is whether that layer goes in front of vmm, replaces vmm, or
absorbs vmm as a library.

---

## 1. Terminology — "organization" or "workspace"?

### Survey of comparable products

| Product | Top-level tenant noun | Personal default |
|---|---|---|
| GitHub | "Organization" | The user's own namespace acts as personal org |
| AWS | "Account", grouped under "Organization" | Root account is the personal scope |
| GCP | "Organization" + "Project" | "My First Project" |
| Azure | "Tenant" + "Subscription" | Default directory |
| DigitalOcean | "Team" | "My Team" auto-created on signup |
| Hetzner Cloud | "Project" | First project on signup |
| Render | "Workspace" | Personal workspace |
| Vercel | "Team", with personal "Hobby" account | Hobby account |
| Linear | "Workspace" | First workspace on signup |
| Slack | "Workspace" | — |
| Notion | "Workspace" | Personal workspace |
| Stripe | "Account" | Account at signup |

### Tradeoffs

**"Organization"**

Pros:
- Most-used in developer-tooling space (GitHub, GCP).
- Implies multi-user-by-default: people *expect* an org has members,
  roles, billing, audit log.
- Plays nicely with on-chain framing: an organization can map to a
  multisig in a way that "workspace" does not.

Cons:
- Heavier-feeling word; some products avoid it for personal use because
  it sounds like "the thing I joined for work".
- Requires the personal-org concept to be slightly hidden (auto-named
  `<username>` and not strictly "personal", as Vercel handles).

**"Workspace"**

Pros:
- Lighter, more SaaS-y. Render / Linear / Notion / Slack all use it.
- Doesn't carry the "billing entity / legal org" weight; people sign
  up without thinking about creating a company.

Cons:
- Less precise about role/membership semantics; "workspace member" is
  vaguer than "organization member".
- Awkward fit when on-chain identity comes in — "workspace's wallet"
  reads oddly compared with "organization's wallet".

**"Team"**

Pros:
- DigitalOcean's term. Friendly. Implies multi-user.

Cons:
- Confuses with "team within an org" (GitHub's nested teams).
- Doesn't extend cleanly when one user is in several of them.

### Recommendation: **organization**, slug `org` in URLs.

Reasoning:
1. dstack already lives in a developer-tooling / GitHub-shaped world,
   and "organization" is the default expectation there.
2. Organizations map cleanly to on-chain entities (a multisig, a DAO).
   The "workspace" framing makes that bridge awkward.
3. The personal-org concept is well-precedented (GitHub: your username
   *is* your personal org). dstack should auto-create `<user-slug>` as
   the personal org on signup, marked `is_personal = true`.
4. The slug `org` is short, unambiguous in URLs, and matches GitHub's
   `/orgs/<slug>` convention.

Stick with one word everywhere. No "team" UI label that secretly means
"organization". No "workspace" alias.

---

## 2. Entity model

### Core entities (proposed)

**`User`**
- `id` — UUID (control plane's internal identifier; never exposed in
  URLs).
- `email` — unique, verified.
- `display_name`, `avatar_url`.
- Auth provider data (next section): one or more linked identities
  (passkey credentials, GitHub OAuth ID, Ethereum address).
- `created_at`, `last_login_at`.
- `personal_org_id` — FK to the auto-created personal org.

**`Organization`**
- `id` — UUID.
- `slug` — URL-safe, globally unique, mutable but rate-limited (the
  GitHub model). Reserved slugs: `admin`, `api`, `auth`, `orgs`,
  `users`, anything matching the public ingress base domain.
- `name` — human label.
- `is_personal` — boolean. Personal orgs cannot be deleted while the
  user exists, cannot have other members invited (debatable: see
  recommendation), and cannot be renamed past their owner's username.
- `billing_email` — optional.
- `created_at`, `created_by_user_id`.
- On-chain identity refs: `wallet_address` (the canonical EOA or
  multisig), and possibly `wallet_type` ("eoa" | "safe" | "managed");
  see §4.

**`Membership`**
- `id` — UUID.
- `org_id`, `user_id`.
- `role` — enum `owner | admin | member | viewer`.
- `invited_by` — FK to user.
- `joined_at`.
- Constraint: every org has at least one `owner`; deleting the last
  owner is rejected. Personal orgs always have exactly one membership
  (their user, role=`owner`).

**`Invitation`**
- `id` — UUID.
- `org_id`.
- `email` — invitee.
- `role` — proposed role.
- `invited_by_user_id`.
- `token` — opaque random string, 30-day expiry, single-use.
- `accepted_at` — nullable; `revoked_at` — nullable.

**`ApiToken`**
- `id` — UUID.
- `org_id` — every token is org-scoped (not user-scoped). This matches
  GitHub's "fine-grained PAT" direction and avoids the
  user-leaves-org-token-still-works hazard.
- `created_by_user_id` — for audit.
- `name` — human label.
- `prefix` + `hash` — store only the hash; show the secret once.
- `scopes` — bitfield or comma-list (e.g. `cvm:read`, `cvm:write`,
  `members:read`); roughly orthogonal to the role matrix below.
- `expires_at` — required, max 1 year.
- `last_used_at`, `revoked_at`.

**`AuditLogEntry`**
- `id` — ULID (sortable).
- `org_id`, `actor_user_id` (nullable for system actions),
  `actor_token_id` (nullable).
- `action` — e.g. `cvm.create`, `member.invite`, `org.wallet.update`.
- `target_type`, `target_id` — what was acted on.
- `request_id` — correlation ID; matches HTTP X-Request-ID.
- `payload_redacted` — JSON; secrets stripped.
- `result` — `ok | denied | error`, with reason.
- `created_at`, `client_ip`, `user_agent`.

### Cardinalities

- User : Organization — many-to-many *through* Membership.
  Plus a single 1:1 personal-org link (`user.personal_org_id`).
- User : ApiToken — many-to-many *through* an `ApiToken` whose owner is
  the org. The user is the *creator*, not the owner.
- Organization : CVM — 1:N. CVM IDs are globally unique within the
  control plane, and an `org_id` FK is added to every CVM record.
- Organization : Wallet — 1:1 in the recommended design (§4); 1:N if
  the operator wants a "production wallet" + "staging wallet" split,
  but recommend keeping that out of v1.
- Invitation : Membership — 1:1 once accepted.
- AuditLogEntry — append-only, partitioned by `org_id`.

### The personal-org concept

Every signup creates exactly one personal organization, owned by the
new user, role=`owner`, `is_personal = true`, slug derived from the
username (with disambiguation if taken). This:
- Lets the user "just get started" — they don't need to think about
  orgs to deploy their first CVM.
- Makes "default deploy target" trivial: it's their personal org.
- Mirrors GitHub semantics that developers already know.

Decisions to make for personal orgs:
- **Can other people be invited into them?** Recommendation: **yes**.
  GitHub doesn't allow it; Vercel does. Allowing it means there's no
  hard "you must create another org if you want collaborators" cliff —
  a user upgrades the social shape of their personal org without
  migrating CVMs. The cost is people occasionally inviting friends to
  their personal namespace; mitigated by clear UI labelling.
- **Can they be deleted?** Recommendation: only when the user is
  deleted, and only after CVMs/wallets have been migrated or torn down.
- **Can they be renamed?** Recommendation: yes (just like the
  username), with the same slug-rename rate limit.

---

## 3. RBAC matrix

Four roles is the GitHub-shaped sweet spot; three is too coarse, five
is over-engineered. Concrete responsibilities:

- **owner** — can do anything in the org, including delete it,
  transfer ownership, and update the on-chain wallet. Multiple owners
  allowed (recommended for any non-personal org).
- **admin** — operational god mode short of destroying the org. Can
  manage CVMs, members (except other owners), tokens, settings.
- **member** — the default for most humans. Can deploy, manage, and
  view CVMs, but cannot manage org-level settings or other members.
- **viewer** — read-only. Useful for support, audit, paying clients
  watching their service.

| Action | owner | admin | member | viewer |
|---|---|---|---|---|
| **CVM lifecycle** | | | | |
| Create CVM | yes | yes | yes | no |
| Start CVM | yes | yes | yes | no |
| Stop CVM | yes | yes | yes | no |
| Restart CVM | yes | yes | yes | no |
| Update / re-deploy CVM | yes | yes | yes | no |
| Resize CVM (vCPU / RAM / disk) | yes | yes | yes¹ | no |
| Delete CVM | yes | yes | yes | no |
| View CVM list / details | yes | yes | yes | yes |
| View CVM logs | yes | yes | yes | yes |
| Exec into CVM (if exposed) | yes | yes | yes² | no |
| **Image management** | | | | |
| List available images | yes | yes | yes | yes |
| Pull image from registry | yes | yes | yes¹ | no |
| Delete local image | yes | yes | no | no |
| **Members** | | | | |
| Invite member | yes | yes | no | no |
| Change another member's role | yes | yes³ | no | no |
| Remove member | yes | yes³ | no | no |
| Leave org (self) | self only⁴ | yes | yes | yes |
| **API tokens** | | | | |
| Create org-scoped API token | yes | yes | yes (for self) | no |
| List all org tokens | yes | yes | own only | no |
| Revoke token | yes | yes | own only | no |
| **Settings & on-chain** | | | | |
| Edit org name / slug | yes | yes | no | no |
| Update billing email | yes | yes | no | no |
| Update on-chain wallet binding | yes | no | no | no |
| Sign on-chain `addComposeHash` etc. | yes⁵ | yes⁵ | no | no |
| Delete organization | yes | no | no | no |
| Transfer ownership | yes | no | no | no |
| **Billing / usage** | | | | |
| View usage / quota | yes | yes | yes | yes |
| View invoices / billing history | yes | yes | no | no |
| Update payment method | yes | no | no | no |
| **Audit log** | | | | |
| View audit log | yes | yes | own actions | own actions |

¹ Resize/pull may consume quota. The control plane checks quota
regardless of role (see §7).
² Exec is a high-risk operation; some operators may want to gate it
to admins-only via a per-org policy flag. Default: members yes.
³ Admins can manage members below them in the role hierarchy. They
cannot demote, remove, or change owners.
⁴ A non-owner can leave; the last owner cannot leave without first
promoting another member.
⁵ "Sign" here means *propose* the on-chain action through the control
plane. Whether it actually goes through depends on the wallet
custody model in §4 — for a multisig, "sign" means "sign one share".

API tokens carry **scopes** that further constrain what the token can
do, but a token is at most as privileged as the role it inherits at
creation time (and re-validated per request to catch role downgrades).

---

## 4. On-chain identity bridging

This is the hardest design choice and the one most likely to leak into
every other surface. dstack-kms today relies on `DstackApp.owner()`
being the EOA the user controls; everything else is consequence.

### Option A — One wallet per organization (recommended a multisig)

Every organization has a single canonical wallet address. All
`DstackApp` contracts owned by the org are deployed with that address
as `initialOwner`. On-chain mutations (add compose hash, add device,
flip flags, upgrade) are constructed by the control plane and submitted
to the wallet for signing.

Sub-options:
- **A1** — wallet is an EOA the org imports / connects (the "Etherscan
  experience"; user signs in the browser via WalletConnect or a similar
  flow when an action requires it).
- **A2** — wallet is a Safe (Gnosis Safe) multisig with the org's
  owners as signers. Signature threshold configurable, default 2-of-N.
- **A3** — wallet is whatever contract address the org configures.
  Control plane builds the calldata and emits a "you have a pending
  action" — the org signs it however its wallet does.

Pros:
- Aligns on-chain authority with off-chain authority — a single org has
  a single thing it owns. Auditing is simple.
- Multisig (A2) gives orgs meaningful security: a single compromised
  user account cannot push a bad compose hash to production.
- Doesn't require the control plane to custody any keys.
- Recoverability is a property of the wallet (Safe recovery, etc.),
  not of the control plane.
- Onboarding has a clear, well-known UX: "connect wallet" buttons.

Cons:
- "Connect wallet" is a real ask for non-crypto users. The first time
  someone deploys a CVM is the moment they hit a Metamask popup.
- Onboarding multisig (A2) is non-trivial for a first-time team.
- Per-action signing latency: every compose-hash change is an L2 tx
  with confirmation latency. The control plane has to model "pending
  on-chain action" as a first-class state.

### Option B — Per-user wallet, off-chain bookkeeping

Each user keeps their own wallet. Org membership is *purely* an
off-chain control-plane fact. CVMs in an org continue to be owned
on-chain by whichever user originally registered them.

Pros:
- Closest to the current dstack model — almost no on-chain change.
- Decentralization story is clean: every user is the sovereign owner
  of their things.

Cons:
- "I'm in the org but I can't change anything" — the membership
  abstraction lies. An admin who can manage members but can't push a
  compose-hash update because the original deployer holds the EOA is
  exactly the friction multi-tenancy is supposed to remove.
- When a member leaves, on-chain ownership remains with them. Either
  the org has to do an on-chain ownership transfer for every CVM (UX
  nightmare), or it accepts that the leaver retains lifetime authority
  over CVMs they deployed (security nightmare).
- Audit log can show "alice deployed CVM X" but cannot show "the org
  owns CVM X" — it doesn't, alice does.

### Option C — Custodial signer per org (control plane holds the key)

Control plane derives or generates a key per org, holds it (HSM,
ideally TDX-protected), and signs every on-chain mutation on the org's
behalf. The user just clicks "deploy" and the control plane does the
rest.

Pros:
- Best UX, by far. No wallet popups, no gas concerns surfaced to the
  user, no signing flow.
- Compatible with offering a SaaS that hides on-chain entirely from
  customers who don't want to know.

Cons:
- Control plane is now a single point of compromise for every org.
  TDX-protecting the signing key reduces that, but does not eliminate
  it (and means the control plane itself becomes a CVM with all the
  attendant complexity).
- Decentralization story breaks. People deploying to dstack *because*
  it's TEE-rooted may be surprised that an off-chain admin can push a
  compose-hash change to their app.
- Recoverability is the operator's problem. Lose the key, lose the
  org's apps.
- Regulatory exposure: holding keys with substantial economic effect
  starts to look like custody for legal purposes in some jurisdictions.

### Recommendation: **A2 (multisig per org) as the default, with B and C as opt-in modes**

Concrete shape:
- On org creation, the control plane offers the user three paths:
  1. **Connect a wallet** — single EOA owns the org's contracts.
     Simple, classic, the user is the "root" admin on-chain.
  2. **Create a Safe** — control plane assists with deploying a Safe
     using the org's owners as signers. Default threshold 1 (single
     owner) on personal orgs, 2 on multi-owner orgs.
  3. **Use managed signing (custodial)** — control plane signs.
     Available only when explicitly opted into; surfaced in UI as
     "managed mode (control plane signs for you)" with a clear
     explanation of trust implications.
- The control plane treats all three uniformly via an "OnChainSigner"
  abstraction:
    - propose action → render calldata
    - request signature → pop wallet / route to Safe queue / sign
      internally
    - submit tx → track receipt → reconcile with off-chain state
- On membership change (owner removed, owner added), the control plane
  *suggests* a Safe membership update but does not force one. Mismatch
  between off-chain owners and on-chain Safe signers is a yellow
  warning surfaced in org settings.
- Personal orgs default to A1 (single EOA) because the multisig is
  overkill and the trust model collapses to "the user owns it" anyway.

Why this default:
- It matches the "DigitalOcean experience" for users who are happy with
  C, the "Phala experience" for users who want A1, and the "real
  production" experience for orgs that want A2 — without picking one
  ideology.
- It does not force the control plane to hold keys (the most risky
  choice) on by default.
- It preserves dstack's value proposition: the on-chain root of trust
  for an org's apps remains in the org's hands.

The thing to *avoid*: option B silently. Some product surfaces will be
tempted to ship "off-chain only" because it's simpler — that lies to
users about who controls what. Either you are bridging on-chain
authority (A or C) or you are explicit that this org's CVMs are owned
by a single person.

---

## 5. Auth providers

### Options

**Email + password**
- Pros: universal floor, no third-party dependency.
- Cons: terrible-by-default; users reuse passwords. Forces the control
  plane to handle password hashing, reset flows, breach checks, MFA
  enrollment. Lots of surface area for very little value if other
  options are present.

**Email + magic link (no password)**
- Pros: dramatically simpler; no password storage; lower phishing
  exposure than passwords.
- Cons: dependent on email deliverability; slightly slower login
  (check email, click). Not great as a *sole* method but excellent as
  a fallback / floor.

**OAuth — GitHub**
- Pros: target audience overlap is huge (developers). Good UX.
- Cons: introduces a hard dependency on GitHub being up and on each
  user having a GitHub account.

**OAuth — Google**
- Pros: even broader reach than GitHub.
- Cons: same hard-dependency concern.

**SIWE (Sign-In-With-Ethereum, EIP-4361)**
- Pros: the user's wallet *is* their identity, which is a
  philosophical fit for a TEE platform. Naturally aligns the auth
  identity with the on-chain identity (§4).
- Cons: still niche outside crypto-native users. Requires a wallet
  install on first login. UX worse than OAuth for non-crypto users.
  Wallet rotation is non-trivial.

**Passkeys (WebAuthn)**
- Pros: phishing-resistant, no shared secrets, increasingly
  cross-platform. Great primary credential.
- Cons: still requires a fallback (lost device). User support burden
  rises if it's the *only* method.

### v1 set, recommended

- **Magic link (email)** — primary "low-friction" path. Required as the
  contact channel anyway.
- **GitHub OAuth** — primary developer path. Deep audience overlap.
- **SIWE** — first-class for crypto-native users; this is the same
  signature flow they already use to sign on-chain mutations (§4), so
  there's strong synergy. Notably, a SIWE login can be used to derive
  the org wallet binding automatically.
- **Passkeys** — *not* v1 if it costs significant time, but I'd ship
  it as a secondary credential ("add a passkey") in v1.1. Once added,
  it's the best primary credential.

What I'd *skip* for v1:
- **Password login.** It earns nothing on top of magic link and adds a
  pile of liability (password store, leak monitoring, MFA UX). If a
  user wants password login later because some compliance regime
  insists, add it in v1.x.
- **Google OAuth.** A second OAuth provider doubles the abuse-mitigation
  work (account-takeover detection per provider) for marginal new
  reach over magic link + GitHub. Add in v1.x.
- **SAML / SSO.** Enterprise-only; explicit out-of-scope until v2.

Identity model implication: a User can have N linked credentials
(`magic_link_email`, `github_oauth_id`, `siwe_address`, `passkey`).
Login resolves any of them to the same User. This means a user can
sign in via SIWE with the wallet that owns their org's contracts, and
the control plane *knows* the link without the user re-claiming it.

---

## 6. API shape

### Options

**REST/JSON**
- Pros: industry standard; works with curl, Postman, any HTTP client;
  trivial to document with OpenAPI; familiar to every developer.
- Cons: more handwritten plumbing than a code-gen approach.

**gRPC / prpc**
- Pros: dstack ecosystem already uses prpc heavily
  (`vmm/rpc`, `kms/rpc`, `gateway/rpc`, all the SDKs). Reusing it
  internally avoids duplicate client codegen.
- Cons: not browser-native. The web UI cannot speak prpc directly,
  meaning a translation layer is needed at the edge anyway.

**GraphQL**
- Pros: flexible queries from a JS frontend. Single endpoint.
- Cons: significant complexity (schema, resolvers, N+1, caching) for a
  control-plane that mostly needs a small, well-defined set of
  endpoints. Rarely the right call here.

### Recommendation: **REST/JSON for the user-facing API; preserve prpc for internal control-plane↔node calls**

Layout:
- **External (user-facing)** — REST/JSON, OpenAPI-documented. Versioned
  with `/v1`. JSON bodies, idempotency-key support on POSTs that
  create resources. Standard 2xx/4xx/5xx semantics.
- **Internal (control-plane → vmm-node)** — keep prpc. The vmm crate
  already speaks it, the security model (RA-TLS) already exists for
  it, and the control plane is the only thing that needs to talk to
  it. It's the right tool when both ends are owned by us.

Resource sketch (REST):

```
# Auth
POST   /v1/auth/magic-link/request      { email }
POST   /v1/auth/magic-link/verify       { token }
POST   /v1/auth/oauth/github/callback   { code, state }
POST   /v1/auth/siwe/nonce              -> { nonce }
POST   /v1/auth/siwe/verify             { message, signature }
POST   /v1/auth/logout
GET    /v1/auth/session                 -> current session
POST   /v1/auth/sessions/:id/revoke

# Users
GET    /v1/users/me
PATCH  /v1/users/me                     { display_name, ... }
GET    /v1/users/me/orgs                # orgs the user belongs to
DELETE /v1/users/me                     # soft-delete

# Organizations
POST   /v1/orgs                         { slug, name }       # non-personal
GET    /v1/orgs/:slug
PATCH  /v1/orgs/:slug                   { name, slug, billing_email, wallet }
DELETE /v1/orgs/:slug

# Members & invitations
GET    /v1/orgs/:slug/members
PATCH  /v1/orgs/:slug/members/:user_id  { role }
DELETE /v1/orgs/:slug/members/:user_id
POST   /v1/orgs/:slug/invitations       { email, role }
GET    /v1/orgs/:slug/invitations
DELETE /v1/orgs/:slug/invitations/:id
POST   /v1/invitations/:token/accept    # invitee path

# API tokens
POST   /v1/orgs/:slug/tokens            { name, scopes, expires_at }
GET    /v1/orgs/:slug/tokens
DELETE /v1/orgs/:slug/tokens/:id

# CVMs (the meat)
GET    /v1/orgs/:slug/cvms              # paginated, filterable
POST   /v1/orgs/:slug/cvms              { name, image, compose, ... }
GET    /v1/orgs/:slug/cvms/:id
PATCH  /v1/orgs/:slug/cvms/:id          # resize, update compose, env
DELETE /v1/orgs/:slug/cvms/:id
POST   /v1/orgs/:slug/cvms/:id/start
POST   /v1/orgs/:slug/cvms/:id/stop
POST   /v1/orgs/:slug/cvms/:id/restart
GET    /v1/orgs/:slug/cvms/:id/logs     # streaming
GET    /v1/orgs/:slug/cvms/:id/events
GET    /v1/orgs/:slug/cvms/:id/attestation

# Images
GET    /v1/orgs/:slug/images
POST   /v1/orgs/:slug/images/pull       { tag }

# On-chain
GET    /v1/orgs/:slug/onchain/wallet
POST   /v1/orgs/:slug/onchain/actions   # propose action
GET    /v1/orgs/:slug/onchain/actions   # list pending
POST   /v1/orgs/:slug/onchain/actions/:id/sign
POST   /v1/orgs/:slug/onchain/actions/:id/submit

# Quotas, usage, billing
GET    /v1/orgs/:slug/usage
GET    /v1/orgs/:slug/quota

# Audit log
GET    /v1/orgs/:slug/audit             # paginated, filterable

# Webhooks (see §10)
GET    /v1/orgs/:slug/webhooks
POST   /v1/orgs/:slug/webhooks
PATCH  /v1/orgs/:slug/webhooks/:id
DELETE /v1/orgs/:slug/webhooks/:id
POST   /v1/orgs/:slug/webhooks/:id/test
```

Notes:
- The path uses `:slug` (human-readable, mutable but rate-limited) for
  the user-facing UI, but every endpoint also accepts `:org_id`
  (UUID) for stability — programmatic clients should pin to the UUID
  to survive renames.
- Streaming (logs, events) uses Server-Sent Events. The vmm already has
  a streaming `/logs` endpoint over Rocket; the control plane proxies
  it.
- Long-running operations (CVM create, on-chain action) return
  `202 Accepted` with an operation ID and a `Location` header pointing
  to a status endpoint, GCP-style. This avoids "request hung for
  40 seconds while QEMU boots" UX.

---

## 7. Quotas and resource accounting

### Why this matters

The vmm today has *host-level* limits — `max_allocable_vcpu`,
`max_allocable_memory_in_mb`, `cid_pool_size` — but no per-org
limits, no per-user limits within an org, and no clear admission
control: it's possible (in principle) to fill the host with one org's
CVMs and starve everyone else.

### Levels of caps

**Per-organization (always required)**
- `max_cvms` — number of CVMs.
- `max_total_vcpu` — sum of vCPUs across running CVMs.
- `max_total_memory_mb`.
- `max_total_disk_gb`.
- `max_gpus`.
- `max_concurrent_deploys` — parallel deploy actions in flight.
- `max_api_tokens`, `max_webhooks` — long-tail.

**Per-organization, per-resource-class (optional v1)**
- The same caps but split by image (e.g. only N GPU CVMs even though
  the org has more vCPU headroom).

**Per-member-within-org (optional v1)**
- Most products don't ship this on day one. Useful for "members can
  deploy at most M CVMs each" so that a single member can't burn the
  org's quota before others. Recommend: model the data so it's
  possible, but ship a single per-org quota in v1.

### Soft vs hard limits

- **Hard** — the request is rejected. Used for safety-critical caps
  (vCPU, memory, GPU, CID pool). Rejected with `HTTP 402` if the cap
  is "you need to upgrade plan", `HTTP 403` if it's "you've hit your
  policy limit", `HTTP 409` if it's a transient resource-exhaustion
  with retry-after.
- **Soft** — the request goes through; the control plane writes a
  warning to the audit log and surfaces a banner in the UI. Used for
  things like "you've used 80% of your monthly quota" or "your CVM
  has been running unusually long".

Recommendation: in v1, every cap is **hard**, but the response
distinguishes "billing/plan" rejection from "policy" rejection. Soft
warnings ship later as a dashboard concern.

### Where enforcement lives

**Control plane (recommended for tenancy caps).** The deploy admission
check is a single SQL query: sum running CVM resource usage for the
org, add the requested amount, compare to quota. The control plane
holds the per-org state, has the consistent view, and has the
transaction boundary to make the check atomic with the
"insert pending CVM" operation.

**Vmm node (host caps).** The CID pool and `max_allocable_*` checks in
`vmm/src/main_service.rs` *also* apply, but they exist to defend the
host, not the tenant. Both layers stay; they have different roles.

The interaction is straightforward: the control plane scheduler picks
a node that has *host* capacity, then admits the deploy if the *org*
has tenancy capacity. Failures at the node level are reported back so
the scheduler can pick another node.

Edge cases to design for:
- Resize that *increases* resources is a quota check just like a new
  deploy.
- An org going over quota due to admin reducing the quota (rather than
  a user adding usage) — the existing CVMs do not get killed; new
  deploys are rejected. Eventually, if the org persistently exceeds,
  alert via webhook.
- Quota changes by an admin are themselves audit-logged.

---

## 8. Audit logging

### What to record

Every action that *changes* state, plus authentication outcomes:

**Auth**
- `auth.login.success` / `auth.login.failure` (with provider)
- `auth.logout`
- `auth.session.revoked`
- `auth.api_token.created` / `auth.api_token.revoked`
- `auth.passkey.added` / `auth.passkey.removed`

**Membership / org**
- `member.invited` / `invitation.accepted` / `invitation.revoked`
- `member.role_changed` / `member.removed` / `member.left`
- `org.created` / `org.renamed` / `org.deleted`
- `org.transferred` (ownership change)

**On-chain settings**
- `onchain.wallet.bound` / `onchain.wallet.changed`
- `onchain.action.proposed` / `onchain.action.signed` /
  `onchain.action.submitted` / `onchain.action.confirmed` /
  `onchain.action.failed`
- (rich `target_id` = action ID; payload includes calldata,
  contract, network, chain ID)

**CVMs**
- `cvm.created`, `cvm.started`, `cvm.stopped`, `cvm.restarted`,
  `cvm.updated`, `cvm.resized`, `cvm.deleted`
- `cvm.boot.progress`, `cvm.boot.error`, `cvm.shutdown`,
  `cvm.crashed`
- `cvm.exec.started` / `cvm.exec.ended` (if exec is exposed)
- `cvm.image.pulled`, `cvm.image.deleted`

**Quotas / billing primitives**
- `quota.exceeded.deploy` (denial with reason)
- `quota.changed` (admin updated the org's quota)

### What *not* to record

- Read calls in v1. Read-call auditing bloats the log without
  much-of-anyone wanting it. Add as a per-org opt-in setting later.

### Retention

- Hot retention: 90 days, queryable from the API.
- Cold retention: 1 year, exported to object storage if the operator
  configures it. Beyond that, the operator's policy.
- Personal orgs and free-tier orgs may have shorter retention; that's a
  product decision, not a security one.

### Redaction

- Never store secrets in audit payloads — encrypted env blobs, API
  token values, passkey credentials. The payload should record
  *what changed* (e.g. "encrypted_env updated, new hash 0xabc...") not
  the value.
- IP addresses and user agents *are* recorded (for security
  investigations) — flag this in the privacy notice.
- Webhook payloads strip headers that look like auth (Authorization,
  X-Api-Key) before logging.

### Append-only

The audit log is append-only at the API level: there is no edit/delete
endpoint. At the storage level, recommendation: a Postgres table with
no UPDATE/DELETE grants for the application role. This won't stop a
DBA, but it stops accidental tampering by application code.

---

## 9. Multi-host scheduling

Right now `dstack-vmm` is one process per TDX host, and the only
"discovery" is per-host via `XDG_RUNTIME_DIR/dstack-vmm/<uuid>.json`
(`vmm/src/discovery.rs`). Going multi-host means a new
control-plane↔node protocol.

### Options for the control-plane↔node split

**Option α — vmm becomes a "node" service that registers with the
control plane on startup; control plane is the only authoritative
source for tenancy state.**
- Each TDX host runs `dstack-vmm` (or its successor, a "node
  agent"). On startup, it dials the control plane (HTTPS + RA-TLS
  recommended), authenticates with a node-registration token, and
  reports its capacity (vCPU, RAM, GPUs, image cache, network
  capabilities).
- The control plane stores a `Node` row per host, periodically
  health-checks (heartbeat or pull), and uses the live data for
  scheduling decisions.
- All user-facing API calls go to the control plane; the control
  plane fans out to specific nodes via prpc.

**Option β — the control plane *is* a vmm. Each "node" is just an
extension of the same process via remote workers.**
- One vmm "leader" process; other hosts run thin "executor" agents
  that the leader RPCs into. Functionally similar to α but blurs the
  separation between control plane and node.
- Tradeoff: simpler short-term, much harder to scale or operate
  long-term (the leader becomes a SPOF).

**Option γ — Kubernetes-style declarative model.**
- The control plane stores desired CVM state. Each node runs a
  controller loop that reconciles "what should be running here"
  against "what is running here". No imperative "deploy" RPCs; the
  control plane just edits the desired state and the node converges.
- Nice for resilience (a flapping node still converges when it's back).
  Heavyweight to implement; mostly worth it if you expect *thousands*
  of nodes.

### Recommendation: **option α**

Reasoning:
- It's the smallest delta from today's vmm. The current single-host
  `Vmm` RPC service stays — the only addition is an outbound
  registration / heartbeat to the control plane.
- It cleanly separates concerns: the control plane is responsible for
  org/user/quota/scheduling/audit, the node is responsible for
  running QEMU.
- It scales fine for the foreseeable future. K8s-style reconciliation
  becomes worth it somewhere around hundreds of nodes; until then,
  imperative RPCs are simpler to debug.

### Scheduler v1 (concrete proposal)

- **Inputs**: list of healthy nodes, their reported capacity, their
  current usage (kept fresh via heartbeat), the deploy request
  (vCPU/RAM/disk/GPU spec, image, networking constraints).
- **Filters** (eliminate ineligible nodes):
  - Has the requested image (or can pull it).
  - Has GPU passthrough enabled if requested.
  - Has free capacity for vCPU/RAM/disk.
  - Has free CIDs.
  - Matches any operator-set constraints (e.g. "this node is for
    paying-tier orgs only").
- **Scoring** (rank eligible nodes): start with the simplest possible
  thing — least-loaded by vCPU%. Don't optimise prematurely.
- **Sticky placement**: once a CVM is on a node, updates and resizes
  go to the same node. Re-balancing across nodes is out of scope for
  v1.
- **Failure handling**: if the picked node fails the deploy, the
  scheduler tries the next-best node, with a fixed retry budget.
  Audit log records each attempt.

### Control-plane ↔ vmm-node protocol sketch

Reuse prpc. Two surfaces:

```proto
service VmmNode {
  // The existing CRUD calls, scoped to a single node.
  rpc CreateCvm(CreateCvmRequest) returns (CreateCvmResponse);
  rpc StartCvm(CvmId) returns (Empty);
  rpc StopCvm(CvmId) returns (Empty);
  rpc RemoveCvm(CvmId) returns (Empty);
  rpc UpdateCvm(UpdateCvmRequest) returns (CvmId);
  rpc StreamLogs(LogsRequest) returns (stream LogChunk);
  rpc GetCvm(CvmId) returns (CvmInfo);
  rpc ListCvms(ListRequest) returns (ListResponse);
}

service NodeRegistration {
  // Outbound from node to control plane.
  rpc Register(NodeRegistrationRequest) returns (NodeRegistrationResponse);
  rpc Heartbeat(NodeHeartbeat) returns (Empty);
  rpc ReportEvent(NodeEvent) returns (Empty);   # CVM lifecycle event
}
```

The first surface is an evolution of the current `Vmm` service with
the ID-only-by-CVM form (no implicit "all VMs on host"). The second
is new and lives outbound from each node.

Authentication: each node has a long-lived registration token issued
when the operator adds it to the control plane. Node-→control RPCs
present the token; control-→node RPCs are mutually authenticated via
RA-TLS, where the node's cert chain attests it's running an approved
vmm image. This piggybacks on dstack's existing RA-TLS infrastructure
(`ra-tls/`).

---

## 10. Webhooks / events for users

### What users actually want

The questions a deployed-app developer asks of a webhook:
- "Did my CVM start?" / "Did it crash?"
- "Did my deploy succeed?"
- "Did it finish booting?"
- "Did my ingress / domain assignment go live?"
- "Did my on-chain action confirm?"

### Options

**Webhooks (HTTP POST to user-supplied URL)**
- Pros: simple, integrates with PagerDuty / Slack / Discord / OpsGenie /
  custom infra. Industry-standard.
- Cons: users have to host a receiver. Delivery semantics
  (retries, signatures, idempotency) need careful design.

**Server-Sent Events on the API**
- Pros: nothing to host on the user's side.
- Cons: only useful while a client is connected; not great for paging
  on real failures.

**Email notifications**
- Pros: zero-config for users.
- Cons: not actionable enough for production teams; *also* useful in
  parallel.

### Recommendation: **v1 ships webhooks + email; SSE is the live UI feed (not a notification system)**

Webhook design:
- Per-org webhook endpoints, each with a name, URL, and a
  *subscription* (event type filter).
- Each delivery includes:
  - `X-Dstack-Event` header (event type, e.g. `cvm.crashed`).
  - `X-Dstack-Delivery` header (unique ULID for the delivery, used
    for idempotency on the receiver).
  - `X-Dstack-Signature` header (HMAC-SHA256 of body using a
    per-endpoint secret; receivers verify to ensure authenticity).
  - JSON body with `event`, `org_id`, `org_slug`, `target` (typed
    object), `occurred_at`, and `data` (event-specific payload).
- Retry policy: exponential backoff up to 24 hours; after that, the
  endpoint is auto-disabled and the org is notified.
- A test endpoint (`POST .../test`) sends a synthetic event to verify
  the receiver.

V1 minimum event set:
- `cvm.created`, `cvm.deleted`
- `cvm.started`, `cvm.stopped`, `cvm.crashed`
- `cvm.boot.failed`, `cvm.boot.succeeded`
- `cvm.ingress.assigned` (when gateway gives it a domain)
- `org.member.added`, `org.member.removed`
- `onchain.action.confirmed`, `onchain.action.failed`

Out of v1:
- Per-CVM webhook subscriptions (everything is org-level v1).
- Filtering by tag.
- Custom event payloads.

Email notifications: a much shorter list, mostly sec-ops:
- New API token created.
- Member added/removed.
- Quota nearly exceeded.
- On-chain wallet changed (with a "if this wasn't you, click here"
  link, fielding lost-cookie attacks).

---

## 11. Integration with the existing components

### What stays as-is

- **`dstack-kms`** — no changes required for tenancy. It is identity-
  blind: it cares about app IDs and on-chain authority, not who
  deployed what. The new control plane just calls KMS using the
  org-bound app contracts. (This is a feature: it preserves the
  existing trust model.)
- **`dstack-gateway`** — no tenancy-aware changes required for v1. It
  routes by app ID via the ingress mapping
  (`<id>.<base_domain>`). The control plane records ownership; the
  gateway routes traffic. Long-term, it might consume an org slug for
  per-org subdomain branding, but that's not required.
- **`dstack-guest-agent`** — no changes. It's per-CVM and identity-
  blind in the same sense as KMS.
- **`certbot` / `ct_monitor` / `verifier`** — unchanged.
- **The `tdx-attest`, `dstack-attest`, RA-TLS plumbing** — unchanged
  and reused for control-plane↔node mutual auth.

### What changes

- **`dstack-vmm`** changes shape (see architectural shapes below).
- **A new `dstack-control-plane` (or whatever name it gets)** appears,
  hosting all org/user/membership/quota/billing/audit/scheduling
  logic.
- **The web UI moves out of vmm.** Today the UI lives in `vmm/ui/` and
  is bundled into the binary (`CONSOLE_V1` in `main_routes.rs`). The
  new UI lives in the control plane and talks REST/JSON to itself.

### Where the control plane sits

> Diagrammatically:
>
> ```
>   ┌──────── browser / CLI / SDK ────────┐
>   │                                     │
>   │  REST/JSON over HTTPS              │
>   ▼                                     │
> ┌──────────────────────────────┐       │
> │       dstack-control-plane     │       │
> │   ─ users / orgs / RBAC        │       │
> │   ─ scheduler                  │       │
> │   ─ quota / audit              │       │
> │   ─ on-chain action queue      │       │
> └─────┬───────────────┬─────────┘       │
>       │               │                 │
>       │  prpc + RA-TLS                  │
>       ▼               ▼                 │
> ┌──────────┐    ┌──────────┐            │
> │ vmm-node │    │ vmm-node │ ...        │
> │  (host A)│    │  (host B)│            │
> └────┬─────┘    └────┬─────┘            │
>      │               │                  │
>      ▼               ▼                  │
>  CVMs (KMS, gateway, guest-agent are CVMs themselves)
> ```

The control plane:
- terminates the user's HTTPS,
- looks up the org via slug,
- enforces RBAC (role + scopes),
- enforces tenancy quotas,
- decides which node to schedule on,
- forwards the deploy via prpc to that node,
- writes the audit entry,
- updates its database,
- and on success, optionally proposes the on-chain action and waits
  on the org's signer.

Each vmm node:
- only knows about CVMs it's running,
- enforces *host-level* limits (CID pool, host capacity),
- speaks its existing CVM lifecycle RPCs to KMS / gateway /
  guest-agent / supervisor,
- never directly talks to a user.

---

## Architectural shapes — the "macro" choice

Three concrete shapes for how the control plane relates to vmm:

### Shape 1 — Monolithic control plane that subsumes vmm

The new control plane *replaces* `dstack-vmm`. There is one process
type with both control-plane and node responsibilities. The single-
host case is "everything runs in one process"; the multi-host case is
"each host runs the same binary, with one elected leader for control-
plane duties".

Pros:
- Simplest to operate at single-host scale.
- No internal RPC; everything's an in-process function call.
- Less to build.

Cons:
- Leader election is an entire distributed-systems problem to ship
  before you have multi-host.
- Surface area of a single process explodes (UI, REST API, scheduler,
  QEMU lifecycle).
- Failure-isolation suffers: a bug in the scheduler crashes the host
  vmm.

### Shape 2 — Thin control plane that orchestrates many vmm-nodes

A new process (control plane). Each TDX host runs `vmm-node`,
which is the existing `dstack-vmm` cleaned up to be node-scoped. The
control plane and the node speak prpc. The control plane has a
database (Postgres), the node has its workdir on disk.

Pros:
- Clean separation of concerns. The control plane is a stateless
  REST API + DB; the node is the QEMU operator. Each has a small,
  understandable role.
- Naturally scales to multi-host.
- Failure isolation: a node crash takes down its CVMs, not the
  control plane or other nodes' CVMs.
- The existing `dstack-vmm` is *almost* the right thing already; the
  delta is "remove the UI, accept node-registration tokens, scope
  things to a node identity".
- Matches the deployed reality of every comparable cloud (DO, Hetzner,
  Linode, Vultr).

Cons:
- More machinery: HTTPS hop between control plane and node, two
  binaries, two configs.
- Single-host ops is now "two processes" instead of one.
- The control plane is a critical service that needs its own ops
  story (HA, DB backups, etc.).

### Shape 3 — vmm becomes a library inside the control plane

The control plane *links* the existing vmm code as a Rust library. No
RPC between control plane and node; in single-host mode it's all one
process. In multi-host mode, the control plane runs on the leader and
remote-control of other hosts is its own problem (e.g. via SSH to
spawn a vmm-lib process).

Pros:
- Single binary in single-host setups.
- Maximum code reuse — vmm becomes "an SDK for managing a TDX host".

Cons:
- Multi-host is an afterthought: how do you remote-control another
  host? This shape pushes the answer toward "ssh + spawn", which is
  hard to operate.
- Library API surface is unstable today; turning vmm into a library
  is a meaningful refactor.
- Conflates the deployment unit (control plane) with the per-host
  unit (vmm) and you eventually unconflate them under pressure.

### Recommendation: **Shape 2 — thin control plane, vmm-node per host**

Concrete plan:
- Rename `dstack-vmm` to `dstack-vmm-node` (or equivalent), strip out
  the embedded HTML UI (`main_routes.rs`), and add an outbound
  `Register` / `Heartbeat` to the control plane.
- New crate `dstack-control-plane`:
  - REST/JSON external API.
  - Postgres for state.
  - Scheduler.
  - On-chain signer abstraction (§4) with adapters for EOA-via-
    WalletConnect, Safe, and managed.
  - Audit log writer.
  - Webhook dispatcher.
- The web UI is a separate React (or similar) SPA served by the
  control plane.
- Internal protocol: prpc (existing), RA-TLS-mutual-authed.
- Single-host deployment: one host runs both processes locally
  (control plane + one vmm-node) — operationally still simple,
  conceptually correct.
- Migration: existing single-host operators run the new control plane
  and a single registered vmm-node; their CVMs migrate as part of
  the migration plan in track 11.

Why Shape 2 over the others:
- It's the only shape that handles multi-host correctly without a
  later rewrite.
- It matches the org/multi-tenant brief — tenancy is a property of
  the control plane, not of any individual host. Putting the auth/
  tenancy code into a per-host vmm pollutes a binary that should stay
  focused on QEMU.
- It plays best with the existing prpc + RA-TLS plumbing dstack
  already has — this isn't speculative architecture, the wires
  already exist.
- It survives the operator stories that show up later: "hot-add a
  node", "drain a node for kernel update", "regional pools",
  "different node classes (GPU vs not)". All of those are natural
  extensions of "control plane has a list of nodes"; none of them are
  natural extensions of "vmm is the world".

---

## Summary of recommendations

| # | Decision | Recommendation |
|---|---|---|
| 1 | Terminology | **Organization** (slug `org`); personal org auto-created on signup. |
| 2 | Entity model | User × Org through Membership; org-scoped ApiTokens; append-only AuditLog. |
| 3 | RBAC | 4 roles (owner / admin / member / viewer) with the matrix above. |
| 4 | On-chain identity | Per-org wallet, default Safe multisig (option A2), with EOA (A1) and managed (C) as opt-ins. Avoid the "no on-chain bridge" option B. |
| 5 | Auth providers (v1) | Magic link + GitHub OAuth + SIWE. Add passkeys in v1.1, Google later, password never. |
| 6 | API shape | REST/JSON externally, prpc internally. `/v1/orgs/:slug/cvms` etc. |
| 7 | Quotas | Per-org hard caps in v1; control-plane-side enforcement; node still enforces host caps. |
| 8 | Audit log | Mutations + auth outcomes; redact secrets; 90d hot, 1y cold. |
| 9 | Multi-host | Control-plane scheduler (option α). Filter / score, sticky placement, prpc + RA-TLS to nodes. |
| 10 | Webhooks | Per-org webhook endpoints with HMAC signing; SSE for live UI; email for security events. |
| 11 | Architecture | **Shape 2** — thin control plane, vmm becomes a node service. KMS / gateway / guest-agent unchanged. |

These choices form a self-consistent design: an org is a tenancy
boundary in the control plane, has an on-chain identity bound to a
real wallet, owns CVMs scheduled across vmm-nodes, and is governed by
a small but realistic RBAC matrix. None of the choices force any of
the others, but together they avoid the failure modes that show up
when a multi-tenant control plane is bolted on top of a single-tenant
host operator.

---

## Open questions for the synthesis pass

These are the things this report intentionally *doesn't* settle —
they need other tracks' input or a product call:

1. **Billing in v1?** The brief says "primitives only". This report
   leaves room for a billing layer (quotas, usage measurement) but
   does not spec invoices, plans, or payment-method handling. Confirm
   that's still the intent in the final plan.
2. **Self-hosted vs SaaS distribution.** A self-hosted operator may
   want a much simpler default (single org, single user, single host
   = "skip the control plane, point at vmm directly"). The shape-2
   recommendation handles this fine (control plane + 1 node, both
   local), but it's worth a UX call — should we ship a "single-tenant
   shortcut" mode or insist on the same shape everywhere?
3. **Node-installation flow.** Adding a node to the control plane
   needs a real onboarding story (registration token? mTLS bootstrap?
   RA-TLS quote-on-first-connect with the operator confirming?). This
   report assumes "long-lived registration token" but the security
   review is part of track 06 / track 11.
4. **Org-deletion semantics.** Soft delete with grace period vs. hard
   delete vs. archive — there are real consequences for on-chain
   state (the contracts persist forever). Probably soft delete + a
   "purge" action that the owner explicitly confirms.
5. **Cross-org transfers.** Can a CVM (and its underlying
   `DstackApp`) be transferred between orgs? On-chain ownership
   transfer is straightforward; the off-chain bookkeeping is too;
   but the user expectation needs to be set. Recommend deferring to
   v1.x.
6. **Secrets / encrypted env management.** The control plane proxies
   the existing per-app pubkey + encrypted env flow; a deeper
   integration ("rotate org secrets across all CVMs") is interesting
   but out of scope here.
