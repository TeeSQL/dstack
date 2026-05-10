# 99 — Skeptic pass on the dstack-vmm rewrite plan

**Prepared:** 2026-05-05
**Stance:** loyal opposition. Read the locked decisions in `00-decisions.md` and
the 11 investigation reports, then push back on every place where reasonable
engineers could disagree. The point is not to be contrarian; the point is to
stress-test the plan before it ossifies into code.

Format per item:
- **Consensus** — the decision in `00-decisions.md`, restated in one sentence.
- **Steelman against** — concrete reasoning, code-grounded where possible.
- **Alternative** — what I would do instead.
- **Cost of being wrong** — in either direction.
- **Verdict** — do I actually recommend the alternative, or does the consensus
  hold up.

The "Recommended changes" section at the end is the executive summary.

---

## 1. Single-host scope (F1)

**Consensus.** `dstack-control-plane` manages exactly one TDX bare-metal box;
multi-host orchestration is out of scope for v1, deferred to v2
(`00-decisions.md:30`, `:289`).

**Steelman against.** The strongest case here is not "multi-host is needed for
hyperscaler reasons", but a humbler one: even modest TDX operators end up with
3–5 boxes within the first year of operating, and the consensus design
*forces* them into 3–5 separate control planes — meaning 3–5 user databases,
3–5 Privy apps (or 3–5 customer accounts of the same Privy app, which has its
own pricing implications), 3–5 audit logs to subpoena from, 3–5 quota
configurations, 3–5 places to provision the master KMK. The "operator runs a
SaaS for their customers" framing in the decisions doc breaks the moment one
of those customers wants their CVMs to span boxes — which is the *normal* case
for any operator whose customers are running anything bigger than a single
hobby project (an active/passive web app, a primary + replica DB, anything
GPU-heavy that wants two boxes).

The investigation reports themselves are not unanimous on this. Report 10
(multi-tenant design space) explicitly lands on **Shape 2 — thin control plane,
vmm becomes a node service** as its top-level architecture recommendation
(`10-multitenant-design-space.md:1140-1180`), with the reasoning "It's the
only shape that handles multi-host correctly without a later rewrite". Report
11 similarly recommends strategy **(b) coexistence, old vmm becomes a node**
because "It is the only strategy that makes the multi-host story honest from
day one" (`11-migration-strategy.md:478-487`). The locked decision in 00
overrides both of those recommendations, in the same direction, with no
written explanation of which arguments from 10 and 11 lost.

The hidden cost of "single host v1" is not the missing v1 feature. It is that
a multi-host control plane in v2 has to undo decisions taken in v1:

- The `users` and `organizations` tables get duplicated per-host. Merging them
  later requires either a global directory of orgs or accepting that "org X on
  host A" and "org X on host B" are unrelated entities. Both are bad.
- Audit log lineage is per-host. The hash chain in §6 of 00-decisions.md is a
  hash chain of *that host's* events; tying them together in v2 requires a
  v2-only "umbrella" hash chain, which means the v1 chain is provable only for
  v1-era events.
- The on-chain Safe in §3 of 00-decisions.md is per-org, but org-id is a v1
  control-plane concept. Two control planes that both think they own the same
  on-chain Safe is a recipe for racy proposals.
- The OCI registry in §8 is per-org and lives at `registry.<base>.tld` of one
  host. Operators with 3 hosts end up with 3 registries, with the same images
  duplicated on each. There is no obvious migration to a single shared
  registry without breaking image refs.

**Alternative.** Design the data model and the auth model for multi-host from
day one, but build only the single-host implementation. Concretely:

- The `cvms` table has an `host_id uuid REFERENCES hosts(id)` from day one.
  Single-host installs have one row in `hosts`. Adding a second host in v2 is
  an INSERT, not a schema migration.
- Org-id is global (uuid7), not per-host.
- The control plane and the host-side worker are *the same binary* in v1, but
  they communicate through a process-internal trait that has the same shape as
  the v2 RPC. The v1 binary just shorts the trait to in-process function
  calls; v2 swaps in a prpc-over-RA-TLS implementation.
- The Safe in §3 is per-org globally, not per-host-per-org.
- The OCI registry serves a single namespace; in v1 it stores blobs on the
  one host's filesystem; in v2 it grows a "blob distributor" that knows how
  to fetch from peer hosts.

This is what report 10 actually recommends (`10-...:1140`), packaged as
"Shape 2 with the multi-host wires stubbed out for v1". The marginal v1 cost
is small: an extra `host_id` foreign key and one trait boundary.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (single host *and* single-host data model): if
  multi-host comes back in v2, you eat a forklift. The v1 audit log can't be
  joined; the v1 orgs become "imported orgs" in v2 with no clean hand-off; the
  v1 OCI registry has duplicated blobs that have to be deduplicated at v2
  cutover.
- Wrong toward the alternative (multi-host shape, single-host implementation):
  you carry an extra `host_id` column and a trait boundary you don't use in
  v1. The cost is real but small. The trait boundary actually *helps* testing
  — you can mock the worker side without spinning up QEMU.

**Verdict. Recommend the alternative.** Specifically: keep the single-host
*implementation* commitment (don't build a scheduler, don't build node
heartbeats, don't build live migration), but adopt the multi-host *data model*
and the trait boundary between control plane and worker. The skeptic position
holds up here — the consensus has already accepted the cost (a redesign in v2)
without writing it down. Make that cost explicit, and pay a fraction of it now
so the v2 redesign is a feature add, not a forklift.

---

## 2. Safe + Base mainnet for on-chain authority

**Consensus.** Per-org Safe multisig on Base mainnet (chain ID 8453), with
admins auto-synced as 1-of-N signers, used for compose-hash whitelist updates
and admin add/remove ceremonies (`00-decisions.md:79-100`).

**Steelman against.** This is the place where the locked decisions are most
overcooked relative to v1's actual use cases.

What does v1 *actually* need to do on-chain?

The vmm has zero on-chain code today, by design — `05-vmm-coupling-map.md:31`
explicitly rates on-chain coupling as "0/5, no direct coupling". The only
on-chain dependency is *transitive*: KMS reads `DstackKms` to authorize
compose hashes, and `DstackApp.owner()` is the EOA that signs
`addComposeHash`. Today that EOA is the *operator*, and `vmm-cli.py` plus
Hardhat scripts are how it gets used. The control plane doesn't have to be
on-chain at all.

The "Safe per org" decision adds the following work to v1:

- A "deploy a Safe lazily on first on-chain action" flow that the user hits
  during *deployment*, which is the worst possible UX moment.
- A signer-state cache (`safes` table per `00-decisions.md:135`) that has to
  be reconciled with on-chain state to detect drift — meaning a periodic
  background poll of Base RPC, with all the failure modes that implies.
- A 1-of-N threshold model that *isn't actually multisig* — it's just
  per-admin signing power with a multisig wrapper. From a security standpoint,
  this provides almost nothing over a single-EOA model: any current admin can
  sign anything, by definition. The only thing the Safe buys you is "we can
  rotate signers on admin add/remove without rotating the underlying app
  ownership", which is a real but small benefit.
- Auto-sync of admins to Safe signers on every role change. Now adding an
  admin is two transactions: an off-chain DB insert (after the on-chain
  confirm, per `00-decisions.md:84-89`), and a Safe `addOwnerWithThreshold`
  proposal that admin #1 signs. The admin-add ceremony went from "INSERT INTO
  memberships" to "user signs a transaction, waits for confirmation, retries
  on failure, deals with gas". And this happens on Base mainnet — not free,
  not fast, and with a real chain to debug when something breaks.
- A 24-hour on-chain timelock for the recovery signer (`:103-107`), which is
  a serious piece of infrastructure for a feature that ships off-by-default.

What does v1 actually *use* this for? The decisions doc says "Safe quorum"
gates admin removal (`:88`), but a 1-of-N Safe with admin #1 still in
the signer set means admin #1 can remove admin #2 single-handedly. The
"quorum" is a single signature. The only real on-chain action that v1 needs
is `addComposeHash` when an org deploys a new compose, and there is no
governance use case for that until the org has multiple admins disagreeing
about whether to deploy version X — which is exactly the case M-of-N
threshold solves, and which is *deferred* (`:289`).

The other complications:

- Privy's per-user embedded wallet means the "current admin" who signs the
  Safe proposal is signing with a key that Privy holds. So the on-chain
  ceremony's security model is "Privy custodies the keys, we trust Privy".
  That's the same trust model as the off-chain DB; the on-chain wrapper is
  decorative.
- Gas costs on Base are low but not zero; an operator running a SaaS for a
  thousand customers, half of whom add an admin once, is paying real money
  to Privy + Base for a feature they're using as off-chain RBAC.
- Recovery edge cases: Privy down + admin #1 unreachable + admin #2 needs
  to remove admin #1 = stuck. The 24-hour timelock recovery signer in
  `:103-107` exists for this, but it's opt-in, off by default, and (per the
  decision) "every signed-batch lineage and recovery-signer capability
  resets" if the operator loses their KMK. That's a lot of moving parts for
  v1.

**Alternative.** Drop on-chain authority from v1 entirely. Use operator-managed
admin signatures (Privy-issued JWT or the session cookie HMAC) for *all*
control-plane actions, including admin add/remove. When a deploy needs an
on-chain action (`addComposeHash`), surface it as "you need to sign this
transaction in your wallet" — exactly the path `vmm-cli.py` already takes
today (`03-vmm-cli.md:91-92` shows `compose` builds an app-compose, and
`docs/vmm-cli-user-guide.md` documents the Hardhat `app:add-hash` flow). The
control plane shows the user the calldata; the user signs in their connected
wallet (Privy embedded or BYO); the control plane submits.

This means:

- No `safes` table, no Safe SDK dependency, no signer-state reconciliation
  loop.
- No "lazy Safe deploy on first on-chain action" UX trap.
- No 1-of-N theater.
- The on-chain identity is whatever EOA the org's admins have configured —
  which can be a Safe if they want one, with the threshold and signer set
  they actually chose, controlled outside the control plane.

The control plane's job becomes: "build the calldata, show it to a user who
has authority to sign it, submit when signed". That's exactly the
"OnChainSigner abstraction" report 10 describes (`10-...:431-440`), minus the
obligation to deploy and sync the Safe per-org.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (Safe per org, auto-synced): you carry the
  complexity for a year of v1, you find out customers wanted M-of-N anyway
  and your 1-of-N Safe is now an awkward middle ground, you have to deal
  with Base mainnet outages affecting deploys, and you have a harder
  time pivoting if you want to support a different chain later (Optimism,
  Polygon, whatever).
- Wrong toward the alternative (no on-chain authority in v1): you ship without
  a "this org's compose-hash whitelist is governed on-chain" story, which is a
  philosophical mismatch with dstack's whole pitch. Some customers may bounce
  on this. You also have to add the Safe-per-org work later when M-of-N
  thresholds become a real customer ask, and that work happens *after* you
  have customers, when retrofitting auth changes is much harder.

**Verdict. Lean toward the alternative, but qualified.** I think the consensus
is over-engineered for v1's actual use case. The case I'd accept the consensus
on is: *if the v1 customer profile genuinely includes orgs with multi-admin
disagreement about deploys, then the on-chain governance has a real use case
even at 1-of-N (the audit trail of who signed what is on-chain, not just in a
DB the operator can rewrite).* But the decisions doc explicitly defers
M-of-N (`:289`), explicitly defers public publication of signed batches
(`:286`), and explicitly deletes "Privy account export / 'claim my wallet'"
(`:296`). With all three of those out, the Safe is providing operator-manageable
RBAC, dressed up. **I would defer the Safe to v2** when M-of-N thresholds and
batch publication land together. v1 ships with operator-managed admin
signatures and a "show me the calldata, I'll sign with my wallet" flow for
on-chain actions that genuinely need the chain (`addComposeHash`,
`addDevice`).

---

## 3. Privy lock-in

**Consensus.** Privy is the identity provider — embedded wallet plus BYO
external wallet, OAuth/email/SMS through Privy
(`00-decisions.md:48-50`).

**Steelman against.** Privy is a vendor with pricing, account portability,
SLA, and operational concerns that the rewrite plan doesn't address. The
default implications:

- **Pricing.** Privy charges per Monthly Active User. For a SaaS with
  thousands of seats this is a recurring opex item. The rewrite plan mentions
  it nowhere and treats Privy as "just the auth layer".
- **Account portability.** The decisions doc explicitly defers "Privy account
  export / 'claim my wallet'" to v2 (`:296`). That means v1 users are
  custodied by Privy — they can't take their wallet (or their identity)
  somewhere else. For a privacy-first / TEE-rooted product, that's an
  uncomfortable framing: "your trust root is in a TEE; your auth is
  custodied by a third party".
- **Operational coupling.** Privy's webhooks, the app secret, and the
  webhook signing secret are all secrets the control plane has to rotate
  and store (`:155-156`). A Privy outage is a control-plane outage.
- **Security model.** Privy's embedded wallet is "Privy holds the key, we
  give you a JWT". For a TEE platform whose value prop is "no one custodies
  your secrets but the TEE", that's a pretty fundamental tension. The
  recovery signer feature in `:103-107` partially addresses this, but
  off-by-default.
- **Lock-in switching cost.** If Privy sunsets the embedded wallet feature,
  triples the price, or has an extended outage, the migration story is
  essentially "rebuild the auth layer". Every user record in the
  `users` table has a `privy_user_id` foreign key (`:120`); detaching
  from Privy is non-trivial.

What does Privy buy us? Two things:

1. Embedded wallets for users who don't want to install Metamask. This is
   real value, especially for the "cloud host" UX framing in 00-overview.md.
2. Multi-provider auth (email magic link, SMS, OAuth) wrapped in one SDK.

The latter is solvable in-house. SIWE + WalletConnect is a 2-week
implementation for someone who has done it before. The first one is harder —
embedded wallets are a real engineering effort if you build them yourself.

**Alternative.** SIWE + WalletConnect for v1, no Privy. Users bring their own
wallet; if they don't have one, they get a "you need a wallet" page with a
link to install Metamask or WalletConnect. The cost: you lose the
"DigitalOcean experience" (sign up with email, no wallet needed) for v1, and
you gain a dependency-free auth surface.

The hybrid alternative: keep Privy for the embedded wallet and OAuth, but
*also* support direct SIWE/WalletConnect from day one, so users can detach
from Privy. The `users` table keeps a `wallets` set; one of them might be
Privy-issued, others might be BYO.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (Privy as the auth layer): you carry vendor
  risk for the life of the product. If the price triples or the SLA breaks,
  you have a forklift on your hands. You also lose the cleanest "TEE-native"
  story for users whose threat model includes "I don't trust an off-chain
  custodian".
- Wrong toward the alternative (no Privy): you ship without embedded wallets,
  meaning every signup needs Metamask or WalletConnect. The conversion
  funnel for non-crypto users is worse. Building embedded wallets in-house
  later is real engineering effort.

**Verdict. Recommend the hybrid.** Use Privy for the embedded wallet and
OAuth, but design the data model so a user can have a BYO wallet *as their
primary signer* without Privy involved. The decisions doc is already
half-way there (`:51-52`: "Bring-your-own external wallet supported via
Privy connectors from day one") — the change is "and bypass Privy entirely
if you want to". This costs little to design correctly v1 and is much
cheaper than "all Privy or all not-Privy" later.

If forced to pick a side: I'd ship Privy for v1 because the embedded-wallet
UX is genuinely valuable and SIWE-only is a hard sell for non-crypto users.
But I'd flag the lock-in as a risk in the decisions doc, not bury it.

---

## 4. Three roles only (Admin / Billing / Viewer)

**Consensus.** Three org-scoped roles only — Admin, Billing, Viewer — plus
a global Platform Admin for the operator. No per-resource ACLs, no per-user
limits inside an org (`00-decisions.md:53-65`).

**Steelman against.** Three is too few because the obvious fourth role is a
**Deploy** role for CI: can push images to the registry, can deploy/start/stop
CVMs, *cannot* invite users, *cannot* change org settings, *cannot* change
billing. This is the role a customer's GitHub Actions runner needs. Today's
"Admin" gives the runner full powers including changing billing and
inviting users; "Viewer" can't deploy. Neither fits.

The locked decision in 00 sidesteps this with API tokens
(`:68-77`): "fields: ... `role` (Admin/Billing/Viewer)". So the runner gets
an Admin token. That's the right answer if and only if "an Admin token
that can do anything" is acceptable for CI — which most security-conscious
customers will say it is not.

The reversal in §9 of this report (API tokens lower than creator's role)
addresses some of this, but it's still solving the wrong problem: the
runner's role is *different* from a human admin's role, not just a
restricted version of it. A Deploy role would say "can mutate CVMs, cannot
mutate org membership, cannot mutate billing, cannot read other users'
secrets even if they happen to be in compose env".

This matters more than it sounds because role-model retrofits are painful.
Once you ship Admin/Billing/Viewer and a thousand customers have configured
their CI with Admin tokens, *introducing* a Deploy role means either:

- Migrating Admin-token-using-CI to Deploy automatically (risky, behavior
  change).
- Leaving Admin-token-using-CI alone forever, which means the Deploy role
  is opt-in and most customers never adopt it.

Compare with GitHub's evolution: they started with Owner/Admin/Member,
realized they needed more granularity, and ended up with the much more
complex permissions matrix they have now. The earlier you add the
distinctions you actually need, the cheaper.

**Alternative.** Ship four roles: Admin, **Deploy**, Billing, Viewer. Deploy
is "Admin minus member management minus billing minus settings". The
permissions matrix in `10-multitenant-design-space.md:268-307` is
already roughly this shape with the names "owner / admin / member / viewer"
— *report 10 explicitly recommends 4 roles*, and the locked decision in 00
collapses that to 3 with no explanation in the doc.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (3 roles): customers who want least-privilege CI
  hit a cliff at "Admin token has too much power"; you bolt on Deploy in
  v1.x with the migration pain above; until then, customers who care about
  security either run their CI with Admin tokens (footgun) or build their
  own restricted-token shim outside the control plane.
- Wrong toward the alternative (4 roles): you maintain one extra row in
  every permissions check. Trivial cost.

**Verdict. Recommend the alternative.** The fourth role is essentially free
to add now and expensive to add later. The locked decision is wrong, and
report 10 (which the synthesis pass had access to) recommended four roles.
There is no upside to dropping Deploy — even if no customer immediately
needs it, having it in the role enum costs nothing and means the eventual
"we need Deploy" conversation doesn't require a migration.

The §9 reversal (API tokens with lower role than creator) doesn't fix this
because Admin-with-restricted-scope is *not* the same shape as Deploy.
Admin-restricted-to-cvm:write *plus* the ambient Admin permissions on
member management is a different role than "this token can do CVM things
and nothing else". Don't conflate them.

---

## 5. Full OCI registry in v1

**Consensus.** Ship a full OCI-compliant per-org private container registry
in v1, on `registry.<base>.tld` with bearer-token auth, mark-and-sweep GC,
and per-org repo path scoping (`00-decisions.md:204-216`).

**Steelman against.** The investigation reports do not advocate for this.
Report 10 has nothing on a private registry — the closest mention is
"images are global, operator-managed" (which is image *catalog*, not
container images). Report 11 is silent on it. The decisions doc treats it
as obvious that v1 needs one, but the obvious-from-the-user-persona
alternative is option (c): support pulling from external private registries
(Docker Hub, GHCR, ECR, GCR) via per-org credentials, and skip the
registry server.

Why this is plausible:

- **The user persona in the decisions doc is "operators running SaaS for
  their customers".** Those customers already have container infrastructure.
  They have GHCR or ECR or a private Docker Hub plan. They are not
  going to migrate their container build pipeline to your registry just
  because you happen to host one — they're going to ask their CI to push
  to *their existing* registry and pull from there.
- **A per-org registry is real engineering effort.** OCI distribution is
  not a small spec. A correct implementation has to handle: blob upload
  (chunked), manifest write (with content-addressing), manifest list
  (multi-arch), garbage collection (mark-and-sweep with running CVMs as
  GC roots — which is exactly what `00-decisions.md:209` says, but the
  "running CVM" reachability check is not trivial when CVMs reference
  images by digest), token auth (bearer-token flow per OCI spec is its
  own protocol), CDN-friendly cache headers, retry-on-resumable-upload,
  and a security audit because anything that runs as root on the host
  with the ability to write blobs the host then runs is a juicy target.
- **The library options for hosting a registry from scratch are limited.**
  The decisions doc lists the three real options
  (`00-decisions.md:316-318`): "build on `distribution/distribution`'s Go
  code via FFI? roll our own with `oci-spec-rs`? embed Zot?". All three
  are non-trivial. Zot is the realistic choice (it's a Go binary you'd
  run as a sidecar), but then you've added a second service to the
  deployment topology, and the "single binary, one host" theme of
  the decisions doc starts to feel like a fiction.
- **The one user-experience cost of dropping the registry** is
  "customers have to manage their own registry credentials". For the
  given user persona, that is *not* a problem — they are already doing
  this. The control plane just stores a per-org bag of registry
  credentials (`docker_username`, `docker_password`, `ecr_role_arn`,
  whatever) and passes them to the CVM at deploy time, the same way
  Kubernetes `imagePullSecrets` works.

**Alternative.** Ship option (c) for v1. Per-org registry credentials,
stored in the secrets table, used at compose-time when the guest agent
pulls. No registry server. The control plane never holds OCI blobs.

Build effort estimate: ~1 week to design the credential storage and the
guest-agent push, vs ~6-10 weeks for a correct private registry server
including GC and security review. This is the single biggest scope-cut in
the plan if you take it.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (full registry in v1): you eat 6-10 weeks of
  engineering, you ship more code with more bugs and a bigger attack
  surface, and your "single binary, one host" deployment story now has
  an opaque-blob-storage subdirectory under `/var/lib/dstack/registry`
  that operators have to back up separately. If customers don't actually
  use the registry (they keep using GHCR), all that work is dead weight.
- Wrong toward the alternative (external registries only): customers who
  *want* an integrated registry have to use a different product or wait
  for v2. Some friction at signup ("set up your registry credentials")
  that the SaaS UX could have absorbed. Marginal.

**Verdict. Strongly recommend the alternative.** This is the place where
the consensus ships the most code for the smallest user value. The decisions
doc went big on the registry without an investigation report backing it up.
Cut to option (c). If a customer asks for a private registry later, it's a
real customer signal and you build it as a v2 feature; until then, it's
speculative scope.

---

## 6. Tamper-evident signed-batch audit in v1

**Consensus.** Always-on hash-chained audit log plus operator-side hourly
signed batches with a dedicated audit signing key
(`00-decisions.md:166-187`).

**Steelman against.** The hash chain is essentially free — every audit row
gets a `prev_hash` and a `row_hash`, and that's it. That part should stay.
But the *signed batches* part is where the YAGNI alarm goes off.

What is the signed batch defending against?

The threat model is: "the operator (or someone with database access)
rewrites audit history retroactively". The signed batch lets an external
party (a customer, a regulator, anyone holding a copy of the operator's
public audit signing key) verify that no row inside a 1-hour batch has
been tampered with after the batch was signed.

Now ask: who is the "external party" in v1?

- The customer? They don't have the operator's public key in any
  channel that is harder to tamper with than the database. The decisions
  doc explicitly defers public publication (`:178`) — there is no S3, no L2
  anchor, no Git repo where the public key + batch signatures live. The
  signed batches are signed and *stored locally* in `signed_batches`. An
  operator who can rewrite `audit_log` can also rewrite `signed_batches`.
- A regulator? In a v1 SaaS run by a single operator for their own
  customers, there is no regulator yet. The "compliance theater" framing
  is honest — this is the SOC 2 / ISO 27001 box-tick for the future, not a
  v1 user ask.
- The internal-malice case? Real, but the `prev_hash` chain alone is enough
  to detect tampering of *individual rows*, and the hot/cold retention
  plus operator-managed cold storage in `:179` is enough for the actual
  forensics use case. You don't need batch signatures to detect "someone
  changed the row" — you need them to *prove* tampering, and proving it
  requires an external chain of custody for the public key, which v1
  doesn't have.

The cost of the signed batches:

- A dedicated `audit_signing_key` separate from the recovery signer
  (`:172`), stored in the secrets table, with all the rotation /
  recovery / loss complexity that implies.
- An hourly job that reads the unsigned rows since last batch, computes
  a Merkle root, signs it, and writes a `signed_batches` row. Cron-job
  reliability is a known operational headache (what if the job is missed
  for 6 hours due to a DB outage? does the next batch cover 7 hours?
  what if the operator rolls back the DB? what if the audit signing key
  is rotated mid-batch?).
- An admin-only export endpoint that produces JSON. JSON of what,
  exactly? Per `:177`, it's the `signed_batches` rows, but those rows
  alone are useless without the `audit_log` rows they sign. So really
  the export is "audit_log + signed_batches together", which is just
  "the whole audit log in a JSON file". The signed-batch part adds
  verification utility *if and only if* the importer has the public key,
  which is in the secrets table in the same operator's box.

**Alternative.** Hash chain on, signed batches off. Defer signed batches
to v2 alongside public publication (which the decisions doc already
defers). The `signed_batches` table exists in the schema but stays
empty in v1. When public publication lands in v2 with an external
chain of custody for the public key, the signed-batch loop turns on.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (signed batches in v1): you build and operate
  an hourly cron job with no current external verifier. If something goes
  wrong (key loss, missed batch, schema change), debugging is real work
  for a feature nobody is using yet. There is some marginal benefit if
  v2 lands soon and the v1 signed batches become retroactively useful.
- Wrong toward the alternative (defer signed batches): you ship without
  the "tamper-evident audit log" marketing line. If a customer asks
  "do you have tamper-evident audit", you say "the hash chain is on,
  signed batches with public publication land in v2". Most customers
  will accept that; the few who won't are also the few who would have
  caught the "but you don't publish the public key anywhere" problem
  with the v1 design anyway.

**Verdict. Recommend the alternative.** Keep the hash chain (cheap, real
tamper detection). Drop signed batches until v2 lands with public
publication. The current shape is ceremony without substance — the v1
implementation defends against an attacker who can't already rewrite
the `signed_batches` table, which is a small and weird threat model.

---

## 7. Per-org VLAN tagging for network isolation

**Consensus.** Per-org VLAN tag in bridge mode, *subject to F3 investigation*
(`00-decisions.md:228-229`). The decision is conditional but if the
investigation says yes, VLAN tagging ships in v1.

**Steelman against.** The F3 investigation has not yet landed (no file
`12-f3-vlan-isolation.md` or similar exists in `/home/fbx/dstack/docs/vmm-rewrite-plan/`).
Until it does, this section is pre-arguing the case. Here is the case for
"don't bother":

What isolation primitives already exist in v1?

1. **TEE memory isolation.** Each CVM runs in a TDX guest. A compromised CVM
   cannot read another CVM's memory; a compromised host cannot read either
   CVM's memory unless TDX is broken. This is the strongest hard boundary
   dstack has (`06-vmm-auth-today-and-gaps.md:303-305`).
2. **Per-org Linux UID for QEMU.** The decisions doc adds this in
   `:222-223`. Per-org UID means org A's QEMU process cannot ptrace org B's
   QEMU process; cannot read org B's workdir if filesystem permissions are
   set right; cannot signal org B's QEMU process. This is meaningful
   defense-in-depth.
3. **Private bridge per host.** The current code has a single bridge
   (`vmm/src/config.rs:402-404`, default `virbr0`) — but per-host the
   bridge is a private network. Anything reaching it from outside the host
   has to come through the gateway (which terminates TLS and routes to
   the WireGuard mesh, with each CVM as a separate WireGuard peer per
   `gateway/rpc/proto/gateway_rpc.proto:62-77`).
4. **Gateway WireGuard mesh.** Each CVM has a unique WireGuard public key
   and IP in the mesh (`gateway_rpc.proto:14, 73-77`). External traffic
   to a CVM is routed via WireGuard; cross-CVM traffic in the mesh is
   constrained by WireGuard's allowed-ips, not by the bridge.

What does VLAN tagging buy on top of this?

The remaining attack surface is "CVM A on the bridge can talk to CVM B on
the same bridge over L2 without going through the gateway". This is real
— a malicious CVM can scan the bridge for other CVMs' DHCP-assigned IPs
and try to talk to them. But:

- All inter-CVM traffic that *matters* goes through the gateway (because
  the gateway issues the WireGuard config; CVMs don't know each other's
  internal IPs by default).
- A malicious CVM that *does* find another CVM's IP is talking to a TEE.
  The TEE is the security boundary. The "can talk on L2" capability is
  a noise floor, not a privilege escalation.
- The "private bridge per org" approach via VLAN tagging requires
  per-org bridge plumbing on the host — for the current code that means
  programmatically creating bridges, attaching tap interfaces with VLAN
  tags, configuring DHCP per-bridge, configuring the gateway WireGuard
  mesh per-bridge, and managing the ID space for VLAN tags (which are
  12 bits, max 4094 distinct tags). All of which is real Linux
  networking complexity that is famously fiddly to debug, and which adds
  per-org host configuration that has to be in sync with the DB.

**Alternative.** Drop VLAN tagging. The combination of TEE isolation +
per-org Unix UID + the WireGuard mesh is the actual security boundary; L2
adjacency on the bridge is a tiny residual that does not move the needle.
Document this honestly in the threat model: "two CVMs on the same host
can L2-ping each other; their TEE memory is still isolated". Customers who
need stronger network isolation than this are running on dedicated hosts
already.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (VLAN tagging in v1): you build the per-org
  bridge plumbing, you debug it for months, and the security benefit is
  defense-in-depth over an already-strong TEE isolation. Some sophisticated
  customers will appreciate it; most won't notice.
- Wrong toward the alternative (no VLAN): a malicious CVM can L2-ping its
  neighbors. If TEE isolation is broken (a hypothetical TDX vulnerability),
  L2 adjacency makes lateral movement easier — but if TEE isolation is
  broken, you have bigger problems and VLAN tagging won't save you.

**Verdict. Lean toward the alternative.** I would wait for the F3 report
before locking this. If F3 confirms that the gateway's WireGuard mesh + the
per-CVM IP space already provides equivalent isolation for the actual
attack surface, drop VLAN. If F3 says "no, there's a real residual risk we
can't close any other way", reconsider. My prior, given what's in the code
today, is "the F3 report will say drop VLAN".

The decisions doc already hedges with "subject to F3 investigation",
which is the right call. My only complaint is that the synthesis pass
locked the rest of the plan without F3 first; the right ordering is "run
F3 *before* finalizing v1 scope" so this section either lands as a
required implementation surface or doesn't exist.

---

## 8. Greenfield-only (N — hard cut, fresh installs)

**Consensus.** Hard cut, fresh installs only. No `vmm-cli.py` compat, no
prpc-v1 compat, no on-chain re-registration, no data migration
(`00-decisions.md:32-35, 263-269`).

**Steelman against.** Report 11 (migration strategy) explicitly recommends
**strategy (b) — coexistence with old vmm as a node**
(`11-migration-strategy.md:478-487`). Report 11 is the single report
specifically about migration. The locked decision in 00 picks strategy
(a) (hard cut) instead, in direct contradiction to its own track 11 expert.

The case for the consensus position is: "small operator population,
hard cut is cleaner internally, the on-disk layout is self-describing
enough that an import tool is a separate v2 problem"
(`11-migration-strategy.md:60-83`). That's a real argument — a hard cut
genuinely *is* simpler internally.

The case against:

- **Existing operators have running CVMs.** The decisions doc treats
  "fresh installs only" as a non-issue, but every existing operator
  has running infrastructure. "Throw it all away and redeploy" is a
  big ask, and the realistic alternative for those operators is "stay
  on the old vmm forever". You will end up with two product lines.
- **The fresh install is for greenfield deployments.** What about the
  customers of existing operators? If the operator stays on the old vmm,
  their customers are stuck on the old vmm too. Now the new control
  plane is a parallel product, not an evolution. That's a marketing
  problem (which version do new customers buy?) and a maintenance
  problem (you're maintaining two codebases).
- **Report 11 specifically calls out that the on-disk layout is
  self-describing.** `11-...:67-72`: "The on-disk layout in
  `~/.dstack-vmm/vm/<uuid>/` is already self-describing
  (`vm-manifest.json` carries app-id, image, ports, gpu, kms_urls,
  gateway_urls, networking — `vmm/src/app.rs:48-73`), so an *import*
  tool can re-adopt CVMs without a real 'compatibility shim'". The
  cost of a read-only import is small. The cost of forcing every
  existing CVM through a redeploy is larger.
- **The on-chain registrations don't require re-registration.**
  `11-...:281-323` explicitly says "nothing on-chain has to change. The
  control plane just needs an 'import on-chain app' flow: user pastes
  `(chain_id, app_id_address)`, signs a message proving control of the
  current `owner()`, and the org adds the app to its registry". This
  is a half-day of work, not a forklift.

**Alternative.** Ship a small "import" path for v1: read-only adoption of
existing CVMs (visible in the new UI as "imported CVMs"), with on-chain
registrations imported by signing a control-message. Existing CVMs can be
viewed, started, stopped, and have logs streamed; *new* operations
(updating compose, resizing, deleting) require explicit migration to a
new-style record (probably backed by a re-deploy, but the user gets to
choose the timing).

This is much smaller than a full migration and dramatically reduces the
adoption friction for existing operators. It's exactly what the
phase-0 + phase-1 plan in `11-...:411-441` describes.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (fresh installs only): existing operators stay
  on the old vmm forever. New customers either run two products or bypass
  the new product. The "greenfield" framing is honest but commercially
  costly. Marketing has to explain "we have a new thing, here's how to
  start over".
- Wrong toward the alternative (import path): you carry a small piece of
  legacy reading code (parse `vm-manifest.json`, populate `cvms`
  table, mark as imported), and you have to make sure imported CVMs
  display correctly in the UI. Maintenance burden ~1 file.

**Verdict. Recommend a minimal alternative.** The consensus is too
absolutist. A read-only import path costs almost nothing and dramatically
reduces friction for existing operators. The decisions doc went hard on
"greenfield only" without absorbing track 11's recommendation, and there's
no upside to leaving existing operators stranded.

The minimal viable import: a one-shot `vmm-cli.py vmm import` (or
equivalent) that registers an existing host with the new control plane,
reads the workdirs, populates the `cvms` table, and tags every CVM with
`imported = true`. Imported CVMs are visible-and-startable but not
mutable until "promoted" to a native record (which is a re-deploy, but
the timing is the user's choice). This is what report 11's strategy (b)
actually describes; it's not the full forklift the consensus made it
sound like.

---

## 9. API tokens with role lower than creator's role

**Consensus.** API tokens are org-bound, GitHub-PAT shape, with `role`
field in (Admin/Billing/Viewer). Token role is whatever the creator
sets within the three-role enum (`00-decisions.md:68-77`).

**Steelman against.** This is fine *if and only if* the role enum is
the right shape. As argued in §4 above, three roles is too few; the
right shape is four (Admin/Deploy/Billing/Viewer). With four roles,
the question becomes: should a token's role be at most the creator's
role?

This is the standard "least privilege for automation" concern. Two
sub-cases:

1. **An Admin creates a Deploy token for CI.** Admin > Deploy in the
   role hierarchy, so this is allowed under "token role ≤ creator's
   role". This is the obvious case the §9 reversal is asking about,
   and the answer is yes.
2. **A Deploy creates a Viewer token for a dashboard.** Deploy > Viewer,
   so this is allowed. Slightly more useful — a CI runner that creates
   read-only tokens for a status page or whatever.

The case against allowing this:

- **Audit clarity.** "Who can do what" gets harder to reason about when
  any role can mint a lower-role token. Today's model is "Admin can mint
  any token"; with the relaxation, you have to walk the creator chain
  to know what trust path each token represents.
- **Security regressions.** A compromised Deploy account can mint Viewer
  tokens that the security team has to track. Without the relaxation,
  the security team can clean up by revoking the Deploy account; with
  the relaxation, they have to find every token that account ever
  minted.

The case for allowing this:

- **Least privilege is a security property, not a UX one.** Forcing CI
  to use Admin tokens because there's no Deploy role (or because Deploy
  can't mint Viewer tokens) is the *opposite* of least privilege. The
  relaxation is the security-correct choice.
- **The audit clarity concern is solved by the audit log.** Every
  token-creation event records the creator. Walking the creator chain
  is a query, not a manual process.
- **The security regression concern is solved by token revocation
  cascades.** When you revoke a parent token, you revoke its
  descendants — same pattern as Linux's `setpgid` cascade or
  GitHub's PAT inheritance.

**Alternative.** Allow tokens to have any role at most the creator's
role, with a parent-token reference for cascading revocation. The
`api_tokens` schema gets a `parent_token_id uuid REFERENCES
api_tokens(id)` column. When a token is revoked, descendants are
revoked too. Audit log shows the creator chain.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (token role = creator's role exactly,
  which is what the locked decision says — see "Used as
  `Authorization: Bearer dst_<token>`" with no scope-down field):
  customers can't do least-privilege automation. Every CI runner is
  Admin-class. Eventually you bolt on the relaxation, with all the
  retrofitting pain that implies.
- Wrong toward the alternative (token role ≤ creator's role with cascading
  revocation): you carry a `parent_token_id` column and a cascade
  query. Audit clarity is slightly harder. Real but small.

**Verdict. Recommend the alternative.** This is a small change with
real security upside. The §9 reversal in the prompt asks the right
question; the answer is yes, allow the token role to be lower than the
creator's. This pairs with the §4 alternative (add a Deploy role) for
maximum benefit; even without the Deploy role, allowing Admin to mint
Viewer tokens is useful for status pages and similar.

The cost of designing it correctly v1 is one schema column and one
cascade query. The cost of bolting it on later is a security audit and
a migration. Pay it now.

---

## 10. Postgres in Docker on the same host with Unix-socket peer auth

**Consensus.** Postgres in a sibling Docker container on the same host,
connected over Unix-socket peer auth, WAL replay enabled by default,
no scheduled backups (`00-decisions.md:111-118`).

**Steelman against.** The consensus picks Postgres for what is
fundamentally a single-process, single-host workload. Look at the
actual concurrency story:

- **Single control-plane process.** "No HA, no active-active. Operators
  who want HA can run two stateless API replicas pointing at the same
  Postgres in v2; not designed in for v1" (`00-decisions.md:42-43`).
- **Single host.** F1 says one box per control plane.
- **The data.** Per `07-persistence-today-and-multitenant.md:216-457`,
  the schema has ~12 tables. None of them have giant row counts: orgs
  in low hundreds, users in low thousands, CVMs in hundreds, audit log
  entries growing at perhaps 100s/day per active org. Total DB size at
  steady state for a typical operator: probably under 10GB even with
  generous audit retention.
- **The query patterns.** Mostly single-row lookups (`get user by id`,
  `list cvms by org`). No reporting, no analytics, no full-text search.
  No JSON aggregation queries. No window functions. No GIS. The
  workload is "read a row, write a row, occasionally write 5 rows in a
  transaction".

This workload is the textbook fit for SQLite. Specifically:

- **No separate process to run.** No Docker container. No Unix-socket
  configuration. No "is Postgres up?" health check. No "did Postgres
  fsync?" question after a power loss. The DB is a file.
- **No backup story to invent.** "Copy the DB file" is the backup
  story. SQLite's backup API supports online backup if you want
  hot backups; the simplest case is "stop the control plane, copy
  the file, start the control plane" — which is fine for v1 because
  the decisions doc already says "no scheduled backups in v1"
  (`:117`).
- **No connection pool.** SQLite is in-process. The control plane
  imports a Rust crate (`rusqlite` or `sqlx` in SQLite mode), opens
  the file, and that's it. No connection lifecycle, no pool
  saturation, no PgBouncer.
- **No operational learning curve.** The operator doesn't need to
  know what `vacuum analyze` does, what `pg_dump` does, what
  `pg_restore` does, what `wal_level` does, what `synchronous_commit`
  does. The DB is a file.
- **No Docker.** The decisions doc puts Postgres in Docker, which means
  the control plane's deployment story is now "two containers,
  ordered". For a "single binary, one host" pitch (which is the
  recurring theme of the decisions doc), having a Docker dependency
  for the DB is incongruous.

What does Postgres buy us for v1?

- **JSONB.** Useful for the audit log payload column. SQLite has
  JSON1 (well-supported), which is enough for what the audit log
  needs (write JSON, occasional read).
- **Concurrent writers.** SQLite serializes writes through a single
  global lock by default. With WAL mode (which everyone enables), you
  get concurrent readers + one writer. For this workload, that's
  plenty. The control plane is *one process* — there isn't a
  contended writer.
- **Replication / HA.** Out of v1 scope. If v2 wants HA, you migrate
  to Postgres then.
- **`citext` and other extension types.** Real but minor — the
  decisions doc uses `citext` for emails and slugs
  (`07-...:225, 238`), but case-insensitive comparison in SQLite is
  one collation away.
- **Row-level security.** Useful for multi-tenant enforcement, but the
  decisions doc doesn't lean on RLS — every query is `WHERE org_id =
  $1` at the application layer.

The strongest argument for Postgres is "we know we're going to need it
in v2 for HA, so why bother with SQLite first?". Counterargument: the
v1 → v2 migration is a one-time cost, and SQLite → Postgres is well-
travelled (every Rails / Django / Node app has done it). The amount
of v1 operational simplicity you gain by using SQLite is real and
recurring; the v2 migration cost is one-time.

**Alternative.** SQLite in WAL mode with a single connection (or a
small connection pool sharing one writer). The schema in
`07-...:216-457` translates almost verbatim — drop `bytea` for `BLOB`,
drop `citext` for collation, drop `inet` for `TEXT`, drop `bigserial`
for `INTEGER PRIMARY KEY AUTOINCREMENT`, drop `jsonb` for `TEXT`
(SQLite's JSON1 is a function library, not a column type, but the
behavior is the same). Backups are `cp /var/lib/dstack/db.sqlite
/backup/`. Done.

**Cost of being wrong, in either direction.**

- Wrong toward the consensus (Postgres in Docker): you ship with a
  separate process, separate fsync semantics, separate health probe,
  separate failure mode (Docker daemon dead = control plane dead).
  Operators have to learn `pg_dump`. The "single binary, one host"
  story has an asterisk.
- Wrong toward the alternative (SQLite): you have to migrate to
  Postgres in v2 if you want HA. Migration tooling is plentiful but
  real (a `sqlite-to-postgres` migration is at least a few days of
  testing). If your data outgrows SQLite (~~250TB~~ in practice with
  WAL — you won't), you have to migrate sooner. With this workload,
  you won't.

**Verdict. Recommend the alternative.** This is the second-biggest
scope-cut available after the registry (§5). SQLite in v1, Postgres in
v2 if HA actually becomes a real ask. The decisions doc commits to
Postgres because that's the "production" answer, but the v1 product
explicitly disowns HA, and the rest of the decisions ("no scheduled
backups", "no replication", "single process") all point at "you don't
need a separate DB process".

If forced to pick a side, I'd accept Postgres but only if it's run as
a sibling *binary*, not in Docker. The Docker dependency is the part
that grates against the "single binary, one host" framing. Run
Postgres directly via systemd, peer-auth via Unix socket, no Docker
daemon involved. But SQLite is the more honest answer for the
workload described.

---

## Recommended changes

Of the ten push-backs above, here is what I'd actually change in
`00-decisions.md`:

1. **Multi-host data model, single-host implementation (decision 1).**
   Add a `host_id` column to every host-scoped table from day one.
   Use a worker trait between control-plane and host-side that's
   in-process for v1 but shaped like the v2 RPC. This costs little in
   v1 and prevents a forklift in v2. The investigation reports already
   recommend this; the locked decision overrode them without
   explanation.

2. **Add a Deploy role (decision 4).** Four roles:
   Admin/Deploy/Billing/Viewer. The fourth role is essentially free to
   add now and expensive to add later. Report 10 already recommends
   four roles.

3. **Drop the in-house OCI registry from v1 (decision 5).** Use option
   (c): per-org credentials for external registries (Docker Hub, GHCR,
   ECR, GCR), pulled at deploy time. Saves ~6-10 weeks. The user
   persona doesn't need an integrated registry to exist; their CI
   already pushes to one.

4. **Defer the on-chain Safe per org to v2 (decision 2).** v1 uses
   operator-managed admin signatures (Privy JWT or session HMAC) for
   all control-plane actions. On-chain actions (`addComposeHash`,
   `addDevice`) are surfaced as "sign this calldata in your wallet"
   prompts, which is what `vmm-cli.py` already does. The Safe wrapper
   adds little at 1-of-N and is overkill until M-of-N + public batch
   publication land together in v2.

5. **Allow API tokens with role ≤ creator's role with cascading
   revocation (decision 9).** Add `parent_token_id` to the
   `api_tokens` schema. Revocation cascades. This is small now and
   expensive later.

6. **Defer signed audit batches to v2 (decision 6).** Keep the hash
   chain on (cheap, real). Drop the signed batches until v2 ships
   public publication. A signed batch with no external chain of
   custody is ceremony.

7. **Add a minimal read-only import path for existing CVMs (decision
   8).** Don't full-migrate, but adopt existing CVMs in read-only mode
   so they're visible in the new UI. This is what report 11 actually
   recommends; the locked decision overrode it.

8. **Defer VLAN tagging until F3 lands (decision 7).** The decisions
   doc already hedges "subject to F3 investigation"; the synthesis
   pass should not have locked the rest of the plan before F3 ran.
   My prior is that F3 will say "drop VLAN", but I'd run F3 before
   committing.

The two consensus positions I'd actually keep:

- **Privy as the auth layer (decision 3).** Steelman against was real
  but the embedded-wallet UX value is also real. Hybrid (Privy + BYO
  bypass) is the right shape; the decisions doc is already most of
  the way there. Flag the vendor lock-in as a risk in the decisions
  doc, but ship Privy.
- **Postgres-in-Docker (decision 10).** Steelman against was the
  strongest of any decision here, but the v2 HA story is real and
  the migration cost is real, and Postgres-in-Docker is what most
  teams know how to operate. Drop the Docker part if possible (run
  it as a systemd unit instead) but keep Postgres.

The pattern across these recommendations: the locked decisions in
`00-decisions.md` consistently picked the more-ambitious-for-v1
choice over the more-conservative choice, and consistently overrode
the investigation reports' recommendations toward "ship less, defer
more". The synthesis pass appears to have aimed at "ship a
production-grade SaaS in v1" without weighting "but how much can we
cut and still ship?". The recommendations above pull back toward
that question.
