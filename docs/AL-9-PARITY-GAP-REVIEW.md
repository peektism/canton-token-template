# AL-9 — OZ-Parity Gap Review (US-FI-Prioritized)

> The AL-9 deliverable: a structured pass over every planning doc to find each
> place we pre-assume *"no motivation / MVP / out of scope"* for a feature where
> we actually want to match OpenZeppelin's Solidity contract-library **breadth**,
> re-weighted by what **US Financial Institutions (US-FIs)** actually require of
> regulated on-chain assets.
>
> Per the AL-9 charter ([PLAN.md §17.7, TT-Q-009a](PLAN.md#177-resolved-decisions--al-8--al-9)),
> the **first artifact is the US-FI usage-expectations research summary** (§1);
> that summary then **weights the parity-gap register** (§2), which **reclassifies
> the [§17.4](PLAN.md#174-explicitly-excluded-with-rationale) "[AL-9: re-examine]"
> items into prioritized slices** (§3). AL-9 is a **review/planning slice**: it
> emits slices and open questions, it adds **no on-ledger surface itself**.
>
> AL-8 (the role-admin hierarchy) is the exemplar this review generalizes: a
> "named extension seam" that was really an OZ-parity gap. AL-9 finds the rest.
>
> Assumes [PLAN.md §17](PLAN.md#17-active-and-future-slices-admin-layer--cip-112)
> for sequencing, [ADMIN-LAYER-PLAN.md](ADMIN-LAYER-PLAN.md) for per-slice detail,
> and [ARCHITECTURE.md](ARCHITECTURE.md) for the substrate-vs-decoupled shape.

Scope owner: OpenZeppelin technical lead.
Inputs: two independent US-FI usage-expectations research briefs (regulatory +
architectural), synthesized and reconciled here against this repo's Daml LF 2.1
constraints (the [C1–C9 corrections](ADMIN-LAYER-PLAN.md#1-corrections-to-the-source-research-still-load-bearing)).

---

## 0. TL;DR

1. **The "code is law" / permissionless posture is legally invalid for US-FIs.**
   For a regulated issuer, the inability to freeze, seize, or recover an asset is
   not immutability — it is a **compliance failure**. The architecture is best read
   not as a set of token features but as an **implementation of financial-institution
   control objectives** (segregation of duties, sanctions enforcement, recoverability,
   auditability, business continuity).
2. **Three gaps reclassify from "excluded" to *Must-Have* prioritized slices**, all
   high US-FI relevance and all feasible on Canton:
   - **AL-10 — account-level freeze / compliance hold** (OFAC/AML; GENIUS-Act
     "seize, freeze, burn, or prevent the transfer" capability).
   - **AL-11 — forced transfer / seize-and-reallocate** (court orders, lost-key
     recovery, SEC Rule 17Ad-12 theft recovery) — extends AL-6 forced-*burn* to
     forced-*reallocation*.
   - **AL-12 — modular transfer-restriction hook** (ERC-1404 / ERC-3643
     `TransferValidator` + allow/blocklist; BSA/AML KYC, SEC Reg D/S jurisdiction).
3. **One *Strong* slice emits: AL-13 — reference-contract global halt** (full
   `Pausable` parity beyond the existing AL-1 origination pause; NYDFS redemption-pause
   authority, incident response).
4. **Three items stay classified-out (confirmed, not reopened):**
   `AccessControlEnumerable` (off-ledger by necessity — no keyless global lookup),
   on-chain `Governor`/`Votes` (US-FIs govern off-ledger; *Nice-to-Have*), and
   single-key `Ownable` (an **anti-pattern** for regulated issuers).
5. **Custody/CDP/bridge roles stay in sibling repos** *except* the **compliance &
   legal admin roles** (`ComplianceAdmin`, `LegalAdmin`, `EmergencyOps`) that the
   freeze/seize slices require here — these fold into the AL-8 `roleAdmin` graph.
6. **All six open questions are resolved (§4, 2026-06-07).** OQ-2/4/5/6 **closed**;
   OQ-1/3 **decided with a defaulted posture, flagged for compliance+infra validation
   before AL-12.** Net posture: *safe by default; manual for deprivation actions;
   dual-control for irreversible legal actions (seize/forced-transfer); no unnecessary
   hot-path contracts (AL-13 → runbook-only); no global-enumerability hacks (off-ledger
   index); single-domain v1 (cross-domain identity deferred); hybrid compliance
   validation (cached routine + live high-risk).* **AL-10/AL-11 are unblocked.**

---

## 1. US-FI usage-expectations research summary (TT-Q-009a artifact)

Tokenization does not eliminate regulatory obligations; it changes the technology
used to satisfy them. Whether ownership lives in a mainframe, a securities
registry, or a distributed ledger, regulators expect the same outcomes: accurate
ownership records, controlled issuance, secure administration, auditable
operations, and **enforceable** compliance controls. The control requirements are
constant; only the implementation changes.

### 1.1 Regulatory drivers

| Driver | Source | What it mandates on-ledger |
|---|---|---|
| **OCC — technology-neutral risk management** | IL 1170 (custody, 2020), IL 1172 (stablecoin reserves, 2020), IL 1174 (INVN nodes, 2021), IL 1179 (non-objection, 2021), **IL 1183 (rescinds 1179; restores tech-neutrality, Mar 2025)** | Apply the **same ITGCs** to smart contracts and keys as to legacy systems: segregation of duties, dual control, change management, auditability, recovery. A single key that can mint or alter rules violates "safe and sound" banking. |
| **NYDFS — stablecoin reserve & redemption** | June 2022 USD-stablecoin guidance; VOLT initiative | 1:1 segregated reserves (T-bills ≤3mo / reverse repo / approved MMFs), par redemption (T+2), and an explicit DFS authority to **require/allow redemption pauses** in extraordinary circumstances → mandates system-wide / account-level **pause**. Mint must be dual-control, treasury-restricted (no over-issuance). |
| **OFAC / BSA / AML / FinCEN Travel Rule** | SDN list (strict liability); GENIUS Act language | Capability to **"seize, freeze, burn, or prevent the transfer"** of assets linked to sanctioned parties; pre-transfer identity interrogation of both sender and receiver → **allow/blocklists + transfer hooks**. Permissionless ERC-20 is largely incompatible. |
| **SEC — transfer-agent safeguarding** | §17A(d) Exchange Act; **Rule 17Ad-12**; *Equiniti Trust Co.* ($850k fine, 2024, $6.6M theft) | Zero tolerance for single-point cyber compromise → **multi-step / time-delayed admin** (the `AccessControlDefaultAdminRules` / `TimelockController` posture) **and forced-transfer asset recovery** when a custody architecture is breached. |
| **SOC-2 / IT audit** | AICPA TSC; routine FI audits | Prove exactly which key holds which role at any instant, with an immutable grant/revoke audit trail → **role enumeration & admin-action auditability** (`AccessControlEnumerable` analogue). |

The throughline: **IL 1183 reduces entry friction but raises the control bar** —
the burden shifts to proving banking-grade ITGCs are *hardcoded into the admin
suite*. The cypherpunk paradigm is not a defense.

### 1.2 Institutional token standards as the breadth target

The market has already converged on compliance-aware standards that treat admin
controls as **first-class**, not optional add-ons:

| Dimension | ERC-20 (permissionless DeFi) | **ERC-3643 / ERC-1400 (regulated)** |
|---|---|---|
| Transfer control | Unrestricted | Blocked unless both parties verified |
| Identity | None | Mandatory (e.g. ONCHAINID identity registry) |
| KYC/AML | Off-ledger only | **Enforced at the contract level** |
| Jurisdiction gating | None | Per-token compliance modules |
| Asset recovery | None ("lost keys = lost assets") | **Forced transfer with on-chain audit trail** |

ERC-3643 (T-REX) is the de-facto institutional RWA standard (ISO TC 307 / TC 68
standardization underway). Its modular split — token logic / identity registry /
**pluggable compliance validators** — is the architectural model AL-12 mirrors in
Daml. This is the breadth ceiling decided in
[TT-Q-009b](PLAN.md#177-resolved-decisions--al-8--al-9): OZ `access/` + `governance`
**plus** the token transfer-restriction extensions OZ ships adjacently.

### 1.3 The DeFi-divergence inversion (why this is not "centralized tyranny")

For each Must-Have below, the DeFi instinct is the **opposite** of the regulated
requirement. This inversion is the crux of AL-9 and worth stating plainly:

| Capability | DeFi view | US-FI reality |
|---|---|---|
| Address freeze | Censorship / tyranny | **Strict legal requirement** (OFAC) |
| Forced transfer / clawback | "Rug-pull" vector / anti-pattern | **Non-negotiable** (court orders, recovery) |
| Restricted transfer | Defeats the point | **Compliance baseline** (KYC/AML/jurisdiction) |
| Single-key owner | Convenient | **ITGC violation** (no separation of duties) |
| On-chain DAO voting | The ideal | **Rare** — FIs govern off-ledger, execute via multi-step admin |

The objective is not *discretionary* control — it is **legally enforceable,
auditable** control, exercised by humans through governed workflows.

### 1.4 Operational role taxonomy US-FIs expect

The research maps cleanly onto four separated administrative responsibilities,
which is exactly the segregation-of-duties the OCC expects and which AL-8's
`roleAdmin` graph already accommodates:

- **Treasury Administration** — controlled issuance/redemption (today: `Minter`/`Burner`).
- **Compliance Administration** — sanctions controls, transfer restrictions, policy updates (**new: `ComplianceAdmin`**, AL-10/AL-12).
- **Legal Administration** — court-ordered seizure, asset recovery (**new: `LegalAdmin`**, AL-11).
- **Emergency Operations** — incident/market-disruption response, global halt (**new: `EmergencyOps`**, AL-13).

---

## 2. Parity-gap & prioritization register

Each capability is scored on three axes, then prioritized:
**(OZ-parity coverage) × (US-FI demand) × (DAML LF 2.1 feasibility)**.
Demand tiers: **Must-Have** > **Strong** > **Nice-to-Have** > **Anti-Pattern**.

| # | Capability / admin feature | Regulatory driver | US-FI demand | OZ EVM analogue | DAML LF 2.1 feasibility | Status here | AL-9 disposition |
|---|---|---|---|---|---|---|---|
| R1 | **RBAC + role-admin hierarchy** | OCC ITGC; SOC-2 | Must-Have | `AccessControl`, `getRoleAdmin`/`setRoleAdmin` | ✅ via party/capability + compiled `roleAdmin` | ✅ **Done (AL-7/AL-8)** | — |
| R2 | **Two-step + timelocked admin handoff** | SEC 17Ad-12; *Equiniti* | Must-Have | `Ownable2Step`, `AccessControlDefaultAdminRules`, `TimelockController` | ✅ propose-accept + ledger-time gate | ✅ **Done (AL-8)** | — |
| R3 | **Account-level freeze / compliance hold** | OFAC; BSA/AML; GENIUS Act | **Must-Have** | `Pausable` (per-account), ERC-3643 freeze | ✅ issuer-signatory hold contract / withheld signature | ❌ **Gap** (pause is origination-only) | → **AL-10** |
| R4 | **Forced transfer / seize-and-reallocate** | SEC 17Ad-12; court orders; lost-key recovery | **Must-Have** | ERC-3643 forced transfer | ✅ issuer is signatory → `ForceTransfer` choice | ⚠️ **Partial** (forced-*burn* AL-6; no reallocate) | → **AL-11** |
| R5 | **Modular transfer restriction (allow/blocklist, KYC, jurisdiction)** | OFAC; FinCEN; SEC Reg D/S | **Must-Have / Strong** | ERC-1404 hooks, ERC-3643 identity+compliance registries | ✅ `TransferValidator` interface + per-tx claim validation | ❌ **Gap** | → **AL-12** |
| R6 | **Global contract pause (full breadth)** | NYDFS redemption pause; incident response | **Strong** | `Pausable` (global) | ⚠️ no global state — reference-contract or sync-domain suspension | ⚠️ **Partial** (AL-1 origination pause) | → **AL-13** |
| R7 | **On-ledger role-member enumeration** | SOC-2 audit | Strong | `AccessControlEnumerable` | ❌ **not expressible keyless** (no global lookup) | ❌ off-ledger only | **Classified-out — off-ledger (OQ-6)** |
| R8 | **On-chain governance / voting** | DeFi DAO patterns | Nice-to-Have | `Governor`, `ERC20Votes` | ✅ propose-accept could model it | ❌ not built | **Classified-out — off-ledger governance** |
| R9 | **Single-key ownership** | rapid prototyping | **Anti-Pattern** | `Ownable` | n/a | ❌ never adopted (Admin = role) | **Classified-out — ITGC violation** |
| R10 | **Custody / CDP / bridge roles** (omnibus, `SecuritiesIntermediaryRole`, `LiquidatorRole`, `BridgeEscrowRole`) | use-case-specific | Strong (in-context) | n/a (app-level) | ✅ | ❌ deferred to siblings (C7) | **Stays in siblings** (compliance/legal roles excepted → AL-10/11) |

Feasibility note: every Must-Have is feasible on Canton — Daml's signatory model
makes freeze (R3) and seize (R4) *more* explicit and auditable than EVM, because
contract archiving notifies stakeholders with cryptographic proof. The hard ones
are the **global-state** features (R6, R7) where Canton's deliberate absence of a
public global registry forces a different shape (see §3 mechanics).

---

## 3. Reclassification → prioritized slices

The following replaces the "[AL-9: re-examine]" tags in
[PLAN.md §17.4](PLAN.md#174-explicitly-excluded-with-rationale) and
[ADMIN-LAYER-PLAN.md §10](ADMIN-LAYER-PLAN.md#10-out-of-scope-named-seams-not-gaps).
Each emitted slice is one self-contained, independently-reviewable unit that keeps
the suite green and consumes the AL-7/AL-8 library; **none should build until its
gating open questions (§4) are resolved.**

### AL-10 — Account-level freeze / compliance hold *(Must-Have)* — ✅ DELIVERED

- **Driver:** OFAC strict liability; BSA/AML; GENIUS Act "freeze … or prevent the
  transfer." Distinct from AL-1 *origination* pause (registry-wide) — this is a
  **per-account** compliance hold.
- **Daml mechanics (as built):** a **`frozenAccounts : [Party]` field on
  `SimpleTokenRules`** — the same non-spoofable shape as `paused` (a field on the
  contract being exercised, not a caller-supplied lookup; keyless LF 2.1 has no global
  registry — correction C2). `assertAccountsNotFrozen` is checked at **all five
  origination chokepoints** (V1 + V2 transfer, V1 + V2 allocation, batch settlement),
  at **direct mint**, and at **cold-path mint acceptance** — the latter via
  `Rules_AcceptMintProposal`, a factory-mediated accept that re-checks the live freeze
  at holding-creation, so a recipient frozen in the propose→accept window cannot be
  credited new supply. V2 chokepoints check **`accountParties`** (the account owner
  **and** its provider), so freezing either the owner or the acting custody provider
  blocks a provider-managed flow. The hold is realized by the registry admin
  **withholding its required signature** at the factory — no public on-ledger blocklist
  (privacy-preserving, GDPR-compatible). Maintained by `ComplianceAdmin`-gated
  `Rules_SetAccountFrozen` (a `Bool`-parameterised freeze/unfreeze mirroring
  `Rules_SetPaused`; redundant-change rejected, admin never freezable),
  with a read-only `Rules_GetFrozenAccounts` for wallet/audit observability. The
  `reason` choice argument is recorded immutably in the exercise node (audit trail).
- **New role:** `ComplianceAdmin` (`roleAdmin = Admin`, delegable through the AL-8
  path). `LegalAdmin`/`EmergencyOps` were added **reserved** in the same enum edit
  (OQ-4), graduated by AL-11/AL-13.
- **OQs honored:** OQ-2 (manual-only, single officer — freeze is reversible),
  OQ-4 (roles live here). **Invariants:** INV-40 (refused at every origination
  chokepoint incl. mint-proposal accept), INV-41 (registry-wide `ComplianceAdmin`
  gate), INV-42 (non-idempotent; admin unfreezable). **Tests:** `Test/Freeze.daml`,
  12 scripts incl. V2 basic, provider-managed (provider frozen), and mint-proposal
  accept — **suite 158 green**.
- **Documented coverage boundary:** freeze is *origination* control like pause —
  it gates new transfer legs in a settlement batch but does **not** reach into
  already-committed `allocations` (seizing committed funds is the distinct
  forced-transfer capability, AL-11).

### AL-11 — Forced transfer / seize-and-reallocate *(Must-Have)* — ✅ DELIVERED (code+tests; verify/props pending)

- **Driver:** court-ordered seizure, lost-key recovery, SEC 17Ad-12 theft recovery,
  erroneous-transfer correction. We had forced-*burn* (AL-6); seize-to-a-new-party
  (reallocation) was the missing half.
- **Daml mechanics (as built):** the issuer co-signs every holding, so a consuming
  **`SimpleHolding_ForcedSeize`** (controller `admin`) archives it without the owner's
  per-action consent — the consuming archive is the owner's **immutable cryptographic
  proof** of the seizure. The value is reallocated to `newOwner` via `newOwner`'s
  `TransferPreapproval` (creating a holding needs the recipient's authority — the
  recovery destination must be onboarded, as any transfer recipient), so net supply is
  unchanged. The whole thing is wrapped in **two-person control** (OQ-2): one
  `LegalAdmin` calls `Admin_ProposeSeizure` (→ a `SeizureProposal`) and a **different**
  `LegalAdmin` calls `SeizureProposal_Approve` (`approver /= proposer`) — no single
  officer can effect an irreversible deprivation of property. Manual-only; the `reason`
  is carried into the (consuming) approve transaction as an on-ledger audit trail.
- **New role:** `LegalAdmin` (graduated from reserved; `roleAdmin = Admin`, delegable).
- **OQs honored:** OQ-2 (manual-only + two-person/separation-of-duties — the irreversible
  counterpart to AL-10's single-officer freeze; **neither officer may be the beneficiary**),
  OQ-4 (role lives here). **Invariants:** INV-43 (two-person + LegalAdmin + beneficiary≠officer
  + cross-registry rejected), INV-44 (value preserved, no redirect, stale rejected).
  **Tests:** `Test/Seize.daml`, 11 scripts (incl. beneficiary-is-officer, cross-registry,
  foreign-preapproval, instrument-scoped), **suite 158→170 green**. Review-hardened (`/code-review`).
- **Pending (next, branch-pinned like AL-8/AL-10):** daml-verify two-person-control proof
  + a daml-props seizure property model.
- **Documented scope boundary:** v1 seizes unlocked `SimpleHolding` (settled funds — the
  primary court-order target); the type system scopes the choice to unlocked holdings.
  Seizing in-flight/`LockedSimpleHolding` funds (mid-transfer/allocation) reuses the AL-6
  `LockedSimpleHolding_ForcedBurn` destruction pattern; reallocation of locked funds is a
  follow-up.

### AL-12 — Modular transfer-restriction hook *(Must-Have / Strong)*

- **Driver:** FinCEN Travel Rule, KYC/AML, SEC Reg D/S jurisdiction limits, holder
  caps — the ERC-1404 / ERC-3643 surface.
- **Daml mechanics:** a `TransferValidator` **interface** with pluggable
  implementations (`BlocklistValidator`, `ClaimsValidator`, jurisdiction modules)
  via Daml Finance interface polymorphism, so compliance logic **evolves by package
  upgrade without redeploying core token logic**. Identity validated per-transfer
  against issuer-held off-ledger KYC claims rather than a global on-ledger registry
  (preserves sub-transaction privacy).
- **New role:** `ComplianceAdmin` (shared with AL-10) maintains the rule sets.
- **Gating OQs:** **OQ-1 (issuer-node availability vs NYDFS timely redemption)**,
  OQ-3 (cross-domain identity).

### AL-13 — Reference-contract global halt *(Strong)*

- **Driver:** NYDFS redemption-pause authority during reserve stress; cyber incident
  response — full `Pausable` breadth beyond AL-1's origination chokepoints.
- **Daml mechanics:** no global boolean is possible. Two candidate mechanisms —
  **(a)** app-level: every transfer choice requires a `ReferenceContract` the issuer
  can archive to halt the ecosystem; **(b)** infra-level: suspend the private Canton
  sync domain. Choice is **OQ-5**.
- **New role:** `EmergencyOps`.
- **Gating OQs:** **OQ-5 (mechanism choice)**.

### Confirmed classified-out (not reopened)

- **`AccessControlEnumerable` (R7)** — keyless ledger has no global member set;
  enumeration is **off-ledger indexing** of active role-grant contracts (auditors
  granted observer rights query them). AL-9 **confirms** this satisfies SOC-2,
  pending OQ-6 sign-off that the off-ledger index is the agreed audit contract.
- **On-chain `Governor`/`ERC20Votes` (R8)** — US-FIs govern off-ledger (board
  resolutions) and execute via the AL-8 multi-step admin. *Nice-to-Have*, not slated.
- **Single-key `Ownable` (R9)** — anti-pattern; `Admin` is a delegable **role**, not
  a key. Permanent.
- **Custody/CDP/bridge roles (R10)** — `V2OmnibusAccount`, Memo Pledge,
  `SecuritiesIntermediaryRole`/`LiquidatorRole`, `BridgeEscrowRole` stay in the
  sibling stablecoin/bridge repos (C7). AL-9's narrowing: only the **compliance and
  legal admin roles** needed by AL-10/AL-11 belong *here*; the rest are use-case
  apps downstream of this template.

---

## 4. Open questions — resolved (2026-06-07)

All six are resolved (also recorded as TT-Q-009c…009h in
[PLAN.md §17.7](PLAN.md#177-resolved-decisions--al-8--al-9)). **OQ-2/4/5/6 are
closed; OQ-1/3 are decided with a defaulted posture but flagged for compliance +
infra validation before AL-12 builds.** The resulting RI posture: *safe by default,
manual for deprivation actions, dual-control for irreversible legal actions, no
unnecessary hot-path contracts, no global-enumerability hacks, single-domain v1,
hybrid compliance validation, cross-domain identity deferred.*

- **OQ-1 (TT-Q-009c) — Compliance-validation topology → ✅ Hybrid (validate w/ stakeholders).**
  *Gates AL-12.* **Cached allowlist/rule-set for routine transfers; live issuer
  validation for high-risk** (large value, new counterparty, restricted jurisdiction,
  suspicious activity, policy override). Rationale: do not make all liquidity depend on
  live issuer-node uptime (operationally fragile), but pure-cached validation carries
  OFAC staleness risk — hybrid is the US-FI compromise. **The redemption/burn path must
  stay available even when ordinary transfers are halted**; SLA / failover assumptions
  documented explicitly. ⚠️ Confirm the topology + SLA with compliance + infra before AL-12.
- **OQ-2 (TT-Q-009d) — Freeze/seize automation → ✅ Manual-only. CLOSED.**
  *Gated AL-11, constrained AL-10.* Freeze, seize, forced transfer, and clawback
  **never fire automatically** from oracle events, expired claims, timers, or rule
  failures. **Freeze/unfreeze: single `ComplianceAdmin`** is acceptable (reversible, may
  need speed). **Seize / forced transfer / clawback: two-person control (propose +
  approve)** because deprivation of property is irreversible. Every action emits an audit
  trail with reason/reference metadata. (Mirrors the existing AL-6 `SimpleHolding_ForcedBurn`
  `controller + requireRole` shape — human-exercised by design.)
- **OQ-3 (TT-Q-009e) — Cross-sync-domain identity → ✅ Single-domain v1 (cross-domain deferred).**
  *Scopes AL-12.* **AL-12 v1 supports issuer-known parties and issuer-held compliance
  claims only.** Cross-domain identity (trust frameworks, claim portability, ZK proofs,
  attestations, cross-domain oracle/router) is **real architecture, not a small extension**
  — explicitly **deferred to a future slice**, not a v1 requirement.
- **OQ-4 (TT-Q-009f) — Role placement → ✅ Add the three roles here. CLOSED.**
  *Gated AL-10/AL-11.* A US-FI-ready RI ships with **generic** regulated-token roles
  (unlike product-specific intermediary/bridge-escrow/liquidator roles, which stay in
  siblings). Add to the template `Role` enum:
  - **`ComplianceAdmin`** — freeze/unfreeze + transfer-rule maintenance (AL-10/AL-12).
  - **`LegalAdmin`** — seize / forced transfer / court-order actions (AL-11); **distinct from
    `ComplianceAdmin`** for separation of duties.
  - **`EmergencyOps`** — pause / emergency-halt authority (AL-13).

  **Keep the flat `roleAdmin _ = Admin` default** (matches the AL-8 TT-Q-008c decision;
  intermediate admins remain opt-in by package upgrade). Each new constructor adds a
  `roleAdmin`/`roleId` line under `-Werror=incomplete-patterns`.
- **OQ-5 (TT-Q-009g) — Global halt / AL-13 → ✅ Documentation/runbook-first. CLOSED.**
  The existing primitives suffice: **AL-1** pause (new originations / registry-level
  operational pause) + **AL-10** freeze (specific accounts/holdings) + **sync-domain
  suspension** as the true break-glass infra halt. **Do not add a reference-contract
  check to the transfer hot path** unless an auditable *per-instrument on-ledger* halt is
  explicitly required. **AL-13 v1 = a documented incident-response runbook**, no new
  hot-path dependency. Revisit only if per-instrument on-ledger halt is later requested.
- **OQ-6 (TT-Q-009h) — Off-ledger enumeration → ✅ Acceptable, backed by on-ledger events. CLOSED.**
  Canton should not mimic `AccessControlEnumerable` where it harms privacy / forces a
  global-lookup assumption. **Grant/revoke events are the authoritative audit trail; an
  off-ledger index materializes current role membership; auditors get observer/query
  access where appropriate.** Document the index as the SOC/audit control.

---

## 5. AL-9 completion checklist

- [x] **US-FI usage-expectations research summary** produced (§1) — TT-Q-009a artifact.
- [x] **Parity-gap register** scored on OZ-coverage × US-FI-demand × DAML-feasibility (§2) — TT-Q-009b breadth applied.
- [x] **Every "[AL-9: re-examine]" item reclassified** (§3): 3 → Must-Have slices (AL-10/11/12), 1 → Strong slice (AL-13), 4 confirmed classified-out.
- [x] **Open questions surfaced and resolved** (§4): OQ-1…OQ-6 — OQ-2/4/5/6 closed, OQ-1/3 decided-with-posture (validate OQ-1 w/ compliance+infra; cross-domain deferred).
- [x] **PLAN.md §17.4 / §17.5 / §17.7 + ADMIN-LAYER-PLAN §10 updated** to point here and carry the new slice IDs.
- [x] **AL-10 ✅ delivered + review-hardened + formally verified** (§3 above) — `frozenAccounts` freeze, `ComplianceAdmin` role, unified `assertCanOriginate` gate, INV-40..42, suite 146→158. Code-review fixes #1–#8 (`c2e61f3`/`f062f16`); formal rows **daml-verify A14–A17 ([#6](https://github.com/OpenZeppelin/daml-verify/pull/6); gate logic, non-vacuous)** + **daml-props `Freeze` ([#4](https://github.com/OpenZeppelin/daml-props/pull/4); owner+provider + cold-path accept)** — the proofs verify the gate logic, the per-chokepoint wiring / V2 expansion / cold-path are script-tested. `LegalAdmin`/`EmergencyOps` added reserved.
- [x] **AL-11 ✅ delivered (code+tests, review-hardened; verify/props pending)** (§3 above) — two-person + SoD `SeizureProposal` (`approver /= proposer`, neither is beneficiary), `SimpleHolding_ForcedSeize` + reallocation via preapproval, `LegalAdmin` graduated, INV-43/44, suite 158→170 (11 tests). 3 design calls open ([PLAN §17.7](PLAN.md#177-resolved-decisions--al-8--al-9)). daml-verify/props rows are the next branch-pinned sub-PRs.
- [ ] **AL-13 → runbook/doc-only** unless per-instrument on-ledger halt is later requested. **AL-12** proceeds on single-domain v1 once the OQ-1 hybrid SLA is confirmed with stakeholders.

AL-9 is a review/planning slice — it adds **no on-ledger surface**. Its output is
this register and the four emitted slices, gated on the six open questions above.
