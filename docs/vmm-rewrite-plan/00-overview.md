# dstack-vmm rewrite — investigation & plan

This folder is **gitignored** (see root `.gitignore`). It is a working planning
space for a multi-tenant rewrite of the dstack-vmm management interface.

## Goal (as stated by the user)

Make the dstack-vmm management interface production-ready for a multi-tenant
deployment shaped like a typical cloud host (DigitalOcean / Hetzner / Render):

- Users sign up and get a personal **organization** (also commonly called a
  *workspace*).
- Owners can invite other users into their organization.
- An organization owns CVMs (the dstack analogue of "droplets" / "instances").
- The control plane needs the usual production niceties: auth, RBAC,
  audit logging, observability, multi-host scheduling primitives, sane
  failure handling, an upgrade story, and clearly separated concerns from
  KMS / Gateway / on-chain.

The user wants the rewrite to also **improve separation of concerns** — the
current vmm is tightly coupled to KMS / Gateway / on-chain / the host OS in
ways that may impede multi-tenant use.

## Investigation plan

Information-gathering is parallelised across 11 expert agents, each writing
its own report into this folder. Files `01-…` through `11-…` are individual
agent outputs. After all reports land, a synthesis pass produces the final
plan in `99-final-plan.md`.

| # | Track | Subject |
|---|-------|---------|
| 01 | Current state | VMM RPC/HTTP API surface |
| 02 | Current state | VMM web UI structure & flows |
| 03 | Current state | vmm-cli capabilities |
| 04 | Current state | VMM internal CVM lifecycle & state |
| 05 | Coupling | VMM ↔ KMS / Gateway / on-chain / guest-agent / host-OS coupling map |
| 06 | Auth | Auth & authz today + multi-tenant gaps |
| 07 | Persistence | Persistence model today + what multi-tenant needs |
| 08 | Production | Operations / observability / multi-host gaps |
| 09 | Domain | CVM domain model (states, invariants, relationships) |
| 10 | Design | Multi-tenant control-plane design space |
| 11 | Migration | Migration & compatibility strategy |

## Open framing questions for the user

(Surfaced in the chat alongside the agent kickoff — answers will narrow scope
in the synthesis pass.)

1. **Multi-host scope.** Does "production" mean a single TDX host with
   multi-tenant orgs, or a control plane that schedules CVMs across many
   bare-metal TDX hosts? This dramatically changes scheduler / state design.
2. **On-chain identity bridging.** dstack-kms today expects an EOA per app
   for upgrade authority. Should orgs map to a single (multisig?) wallet,
   should each user keep their own wallet and the org just be off-chain
   bookkeeping, or should the control plane custody keys on behalf of orgs?
3. **Terminology.** "Organization" or "workspace"? Default proposal:
   **organization** (matches AWS / GitHub / Hetzner-projects-internal).
4. **Auth providers.** Email + password / OAuth (GitHub/Google) /
   SIWE (sign-in-with-Ethereum) / all? dstack ethos leans crypto so SIWE
   is on the table.
5. **Self-hosted vs SaaS.** Is the control plane something an operator
   self-hosts for their own org+invitees, or is it a multi-org SaaS?
   (Affects abuse / quotas / billing primitives / multi-region.)
6. **Billing in scope?** Default proposal: out — primitives only (quotas,
   usage measurement) so a billing layer can be added later.
