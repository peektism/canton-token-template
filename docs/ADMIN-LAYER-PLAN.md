# Admin Layer Plan: Ergonomic AccessControl / Ownable / Pausable for the Canton Token Template

Status: Implemented, tested, and **converged** — AL-0..AL-3 (admin layer) and
AL-5/AL-6 (mint/burn, §11) have each reached a clean `/code-review` pass.
Source under `simple-token/daml/SimpleToken/Admin/` (`Roles`, `Errors`,
`Capability`, `Authority`) + `Supply.daml`, the Pausable/Mint chokepoints on
`SimpleTokenRules`, and the burn choices on `SimpleHolding`/`LockedSimpleHolding`;
tests in `Test/Admin.daml` + `Test/MintBurn.daml`. Validated on SDK 3.4.11 / DPM
1.0.17 / OpenJDK 21: full suite **141/141 passing**, `scripts/verify.sh` **3/3**
(daml-lint clean on `Admin/*` + `Supply.daml`, daml-props, daml-verify). Remaining
admin-layer work and the forward roadmap are tracked in
[PLAN.md §17.5](PLAN.md#17-active-and-future-slices-admin-layer--cip-112) — next up:
**AL-4** (the `daml-verify` capability/allowance/scope proof model) then **V2-10**
(`BatchProcessor`-gated settlement, the last reserved role).

Scope owner: OpenZeppelin technical lead.
Reviewers: OZ security lead (authority/privacy), CIP-0112 authors (V2 gating
surface), Digital Asset technical contact (Daml LF 2.1 idioms).

Related docs:
[SCOPE.md](SCOPE.md) · [PLAN.md](PLAN.md) ·
[CIP-0112-EXTENSION-PLAN.md](CIP-0112-EXTENSION-PLAN.md).

---

## 0. The architecture in one paragraph

The three OpenZeppelin v5 administrative interfaces are **not** three mechanisms
here. **AccessControl is the substrate; Ownable and Pausable are its
specializations** — exactly how OZ v5 itself is layered (`Ownable` ≈
`DEFAULT_ADMIN_ROLE`). On Canton the instrument `admin` party is baked into every
`InstrumentId` and co-signs every holding, so it is the permanent cryptographic
root and cannot be reassigned without re-issuing assets. The admin layer embraces
that: a single fixed root party (the genesis `admin`) anchors a capability system;
the **`Admin` role is the "owner"**, delegable and revocable, so operational
governance can be handed to another party without a (impossible) party swap;
**Pausable is origination control** gated on a `Pauser` capability at the factory.
One mechanism, three faces, no second authority axis, no contract keys.

---

## 1. Corrections to the source research (still load-bearing)

The source research's *pattern* (registry + capabilities) is right; its
*mechanics* do not compile or do not hold here. Each is a hard constraint.

| # | Research claim | Why it fails here | What the implementation does instead |
|---|----------------|-------------------|---------------------------------------|
| C1 | `TokenAdministrator … key (owner, instrumentId); maintainer …` | **Daml LF 2.1 dropped contract keys** ([AUDIT.md G4](AUDIT.md), [SCOPE.md §9](SCOPE.md#9-post-mvp)). No `key`/`lookupByKey`. | Reference admin/capability contracts **by `ContractId`** — capabilities are passed as explicit choice arguments (the caller holds its own), with explicit disclosure where needed. |
| C2 | `GlobalPause … lookupByKey` for "O(1) pause" | Keys again; and a caller-supplied pause lookup is **spoofable** (the settler controls their own context). | Pause is a **non-spoofable `paused : Bool` field on `SimpleTokenRules`** — the contract being exercised — checked at the factory chokepoints. See §4. |
| C3 | `RoleCapability … key (assignee, instrumentId, roleType)` | Keys; and `roleType : Text` is unbounded/typo-prone. | Keyless `RoleCapability` template + closed `Role` sum type, validated by `requireRole` (proof-by-`fetch`). See §3. |
| C4 | `roleType : Text` (`"MINTER_ROLE"`) | Trips `daml-lint unbounded-fields`; admits typos. | Closed `data Role = Minter \| Burner \| Pauser \| BatchProcessor \| Admin`. |
| C5 | `V2BatchTransferFactory` (`foldlA` array dispatch) | Not the real CIP-112 surface. | Gate the **real** interfaces (`SettlementFactory_SettleBatch`, multi-leg `Allocation`). No invented template. |
| C6 | "Iterated settlement mutates one `ContractId`" | Daml `create this with …` yields a **new** CID. | Successor allocations already track `numIterations` / `originalAllocationCid`. Unchanged by the admin layer. |
| C7 | Memo Pledge / omnibus / CDP / bridge roles in the template | Stablecoin/custody/bridge use-cases, not token-standard interfaces. | Out of scope; those consume the generic `Role` primitive from sibling repos. |
| C8 | EVM error strings (`"ERC20Pausable: …"`) | Wrong mental model; breaks message conventions. | Canton-native namespaced messages in `Admin/Errors.daml`. |
| **C9** | **Ownable2Step transfers the owner *address/party*** | **The admin party is baked into `InstrumentId`; "transferring" it would orphan every holding.** | **Ownership is the `Admin` *role*, not a party.** Handoff = grant `Admin` to a new party and revoke the old; the immutable instrument party never moves. See §5. |

C9 is the correction that drove the final (hybrid) design — see [PLAN.md §17.3](PLAN.md#17-active-and-future-slices-admin-layer--cip-112) for how it superseded the earlier "OwnershipOffer two-step party transfer" sketch, which desynced from `SimpleTokenRules.admin`.

---

## 2. Architecture overview

```
   genesis root (FIXED)        AccessControl substrate            specializations
   ──────────────────          ───────────────────────            ───────────────
   admin : Party  ───issues──► RoleCapability {admin, assignee,    Admin   = Ownable owner
   (== InstrumentId.admin,                     role, scope}        Pauser  = Pausable authority
    signs every holding)       requireRole (proof-by-fetch)        Minter/Burner/BatchProcessor
        │                              │                                   = future role-gated ops
        │ Admin role can               │ gates
        ▼ re-delegate (handoff)        ▼
   TokenAdministrator           SimpleTokenRules.paused (origination chokepoint)
   (Admin_IssueRole,            transfer V1/V2 · allocation V1/V2 · settlement-via-factory
    Admin_DelegatedIssueRole)
```

Mapping to OpenZeppelin v5, corrected for Canton:

| OZ v5 | This design (Daml LF 2.1) | Mechanism |
|---|---|---|
| `AccessControl.hasRole` | `requireRole` over a co-signed `RoleCapability` | Proof-by-`fetch`, not a mapping read |
| `grantRole` / `revokeRole` | `Admin_IssueRole` / `Admin_RevokeRole` (root) and `Admin_DelegatedIssueRole` / `Admin_DelegatedRevokeRole` (delegate) | Capability create / archive |
| `DEFAULT_ADMIN_ROLE` | the **`Admin` role** | Holder may issue/revoke capabilities |
| `Ownable` owner | the holder of an `Admin` capability (initially the genesis `admin` party) | Owner == root role; no separate `owner` field |
| `Ownable2Step.transferOwnership` | grant `Admin` to the new party, revoke the old | Capability handoff; instrument party stays fixed (C9) |
| role scoping (per-resource) | `RoleCapability.scope : Optional InstrumentId` | `None` = registry-wide; `Some` = one instrument |
| `Pausable.whenNotPaused` | `assertNotPaused` on `SimpleTokenRules.paused` at origination chokepoints | Non-spoofable field on the exercised contract |

---

## 3. AccessControl — the capability substrate (`Admin/Roles.daml`, `Admin/Capability.daml`)

Closed role type (C4); `Admin` is the owner / `DEFAULT_ADMIN_ROLE`:

```haskell
data Role = Minter | Burner | Pauser | BatchProcessor | Admin
  deriving (Eq, Show)
```

Capability + validator:

```haskell
template RoleCapability with
    admin : Party                  -- registry root party (anchors the cap; unforgeable)
    assignee : Party
    role : Role
    scope : Optional InstrumentId  -- None = registry-wide; Some = least-privilege
  where signatory admin; observer assignee

requireRole : Party -> Role -> Party -> Optional InstrumentId -> ContractId RoleCapability -> Update RoleCapability
--           caller    role   admin    requiredScope            cap
-- asserts: cap.admin == admin ∧ cap.assignee == caller ∧ cap.role == role
--          ∧ scopeAuthorizes cap.scope requiredScope ; returns the validated cap
--          (so callers needing its fields, e.g. mintAllowance, don't re-fetch)
```

`scopeAuthorizes capScope requiredScope`: a registry-wide cap (`None`) authorizes
anything; an instrument-scoped cap authorizes only that exact instrument and
**not** registry-wide operations (so an instrument-scoped `Pauser` cannot halt the
whole registry). This is the least-privilege primitive (resolves the prior
"registry-wide only" limitation).

**Soundness without keys:** capabilities are admin-signed (unforgeable) and name a
specific `assignee`; the worst an adversary can do is present a capability they
don't hold and fail their own authorization. The caller passes their capability
CID as an **explicit choice argument** (the caller is the controller and holds its
own capability), with explicit disclosure where the caller is not already a
stakeholder — no `ChoiceContext` key is needed.

---

## 4. Pausable — origination control (`SimpleTokenRules`)

`SimpleTokenRules` carries `paused : Bool`. `assertNotPaused paused` is checked at
the five factory **origination** chokepoints before any state mutation:

- `transferFactory_transferImpl` (V1, V2)
- `allocationFactory_allocateImpl` (V1, V2)
- `settlementFactory_settleBatchImpl`

Pausing/unpausing is `Rules_SetPaused` (consuming recreate), gated on a
**registry-wide** `Pauser` capability; `Rules_GetPaused` is an ungated read.

**Why origination control is the correct (and only robustly enforceable)
semantic.** The earlier design doc — copying the research's "emergency halt aborts
in-flight DvP" — overstated coverage. In a keyless UTXO ledger:

1. Value-completing choices (`Allocation_Settle`, `Allocation_ExecuteTransfer`,
   `TransferInstruction_Accept`) live on the allocation/instruction contracts,
   which **cannot read the factory's pause flag** without a caller-supplied
   context hop that the settling party can omit — i.e. any guard there would be
   **spoofable**, not enforcement.
2. Those flows are **committed settlements**: funds are already locked (admin
   co-signs the lock) and a counterparty is relying on completion. Halting them
   would strand counterparties mid-DvP — the opposite of safe.

So pause stops **new exposure** (originations) and leaves committed flows and
recovery (withdraw/reject/cancel, INV-33) open. This matches how production
registries behave. A true per-asset **freeze** (e.g. the admin neutralizing a
specific suspect allocation via its lock-holder authority) is a *separate* future
capability — deliberately not conflated with pause. The docs (§6 below,
[AUDIT.md](AUDIT.md) INV-25) now state exactly this.

---

## 5. Ownable-as-Admin-role (`Admin/Authority.daml`)

There is **no transferable `owner` party and no `OwnershipOffer`**. `TokenAdministrator`
is the genesis issuance anchor, fixed to the root `admin` party:

```haskell
template TokenAdministrator with admin : Party where
  signatory admin
  nonconsuming choice Admin_IssueRole   ... controller admin            -- root issues
  nonconsuming choice Admin_RevokeRole  ... controller admin            -- root revokes
  nonconsuming choice Admin_DelegatedIssueRole  with caller, adminCap, role, … -- delegate issues
    controller caller do requireAdminDelegate caller admin adminCap
                          assertMsg eRoleNotDelegable (delegableRole role); mkRoleCapability …
  nonconsuming choice Admin_DelegatedRevokeRole with caller, adminCap, capCid  -- delegate revokes
    controller caller do requireAdminDelegate caller admin adminCap
                          revokeIssuedCapability admin delegableRole capCid   -- delegate: delegable roles only
```

Direct (root) and delegated paths share single-sourced helpers (`mkRoleCapability`,
`revokeIssuedCapability admin mayRevokeRole capCid`) so the create/revoke logic
cannot drift — the root passes `const True`, a delegate passes `delegableRole`.

**The delegation policy is a single function** — `Roles.delegableRole : Role -> Bool`
— used by *both* the delegated issue and revoke paths (not two ad-hoc checks), so
it cannot drift. A delegate may issue/revoke a role only when `delegableRole role`;
the genesis root (`Admin_IssueRole`/`Admin_RevokeRole`) may manage any role. Today
only `Pauser` is delegable; `Admin` is root-only, and the reserved roles
(`Minter`/`Burner`/`BatchProcessor`) are root-only **until their enforcing slice
lands** — so a delegate cannot pre-mint a reserved capability and retroactively
gain power when that slice starts gating on it. A reserved role graduates by
flipping its `delegableRole` case in the same change that adds its enforcement.

This caps the hierarchy at **root → Admin delegates → delegable roles** and closes
the escalation paths a flat model opens: re-delegation (a delegate minting `Admin`),
delegate-vs-delegate / delegate-vs-root `Admin` revocation, and reserved-role
pre-minting. A delegate may still manage *delegable operational* capabilities —
including ones the root issued (shared operational-role management) — and the
genesis root remains the sole governance authority, always recoverable via the
un-revocable `TokenAdministrator`.

**Ownership handoff** = the root grants an `Admin` capability to a new governance
party. That party administers operational roles via the `Admin_Delegated*` choices,
which run with the anchor's admin authority and so mint/archive admin-signed
capabilities **without holding the root key**. To hand control back, revoke the
delegate's `Admin` capability. Because every capability is anchored to the same
fixed `admin`, a delegate's capabilities work immediately — no factory rebind, no
desync (the failure mode of the earlier two-step sketch, C9).

This composes with the standard institutional pattern at zero extra code: a cold
root party (the genesis `admin`, backed by an offline multisig via Canton
topology) plus a hot governance party holding a live `Admin` capability.

Extension seams (intentionally not built, named for maintainability):
- **Per-role admins** (who may grant role X): generalize `Admin_DelegatedIssueRole`'s
  `requireRole … Admin …` check to a role→role admin map.
- **Two-step / timelocked issuance**: wrap issuance in a `RoleOffer` the assignee
  accepts, or a time-gated wrapper.
- **Multisig owner**: back the `Admin`-cap holder party with a multi-hosted Canton
  party — no Daml change.

---

## 6. CIP-112 integration — what the admin layer governs

| V2 surface | Existing actor check | Admin-layer gate |
|---|---|---|
| `TransferFactory_Transfer` (V1/V2) | `actors == [sender authority]` | `assertNotPaused` (origination) |
| `AllocationFactory_Allocate` (V1/V2) | `actors == [authorizer authority]` | `assertNotPaused` (origination) |
| `SettlementFactory_SettleBatch` | `actors == settlement.executors` | `assertNotPaused`; optional `BatchProcessor` capability for a delegated matching engine (V2-10) |
| `Allocation_Settle`, `Allocation_ExecuteTransfer`, `TransferInstruction_Accept` | controllers per CIP-112 | **Not pause-gated by design** — committed settlement / completion proceeds (see §4); robust enforcement there is impossible in a keyless model |
| future `Mint` / `Burn` | n/a | `Minter` / `Burner` capability via `requireRole` (scope-aware) |

This corrects the previous table, which incorrectly listed `Allocation_Settle →
whenNotPaused`.

---

## 7. Security invariants (extends [PLAN.md §9](PLAN.md#9-security-invariants))

| # | Invariant | Enforcement | Test |
|---|---|---|---|
| 25 | No factory **origination** choice succeeds while `paused` | `assertNotPaused` at 5 chokepoints | `test_pauseBlocksTransfer`, `test_pauseBlocksAllocation` |
| 26 | Pause/unpause requires a **registry-wide** `Pauser` capability | `Rules_SetPaused → requireRole … None` | `test_pauseRequiresPauserCap` |
| 27 | Capability honored only if `cap.admin == registry admin` | `requireRole` | `test_capabilityImpersonationFails`* |
| 28 | Capability honored only if `cap.assignee == caller` | `requireRole` | `test_capabilityImpersonationFails` |
| 29 | Capability honored only if `cap.role == required role` | `requireRole` | `test_pauseRequiresPauserCap`, `test_nonAdminCannotDelegateIssue` |
| 30 | Capability scope must authorize the operation (instrument cap ≠ registry-wide op) | `scopeAuthorizes` | `test_scopedCapabilityCannotPauseRegistry` |
| 31 | Capabilities issued/revoked only by the root `admin` or an `Admin`-capability holder; the **`Admin` role is root-managed** (delegates cannot issue/revoke `Admin`) | `TokenAdministrator` choices + `delegableRole` policy (`eRoleNotDelegable`) | `test_delegatedAdminGovernance`, `test_nonAdminCannotDelegateIssue`, `test_delegatedRevoke`, `test_delegatedCannotRevokeAdmin` |
| 32 | A revoked capability can no longer authorize | archival + `requireRole` `NotActive` | `test_revokeRole`, `test_revokedAdminCannotDelegate` |
| 33 | Pause does not freeze funds: recovery and committed-settlement completion stay open | recovery/settlement choices ungated | `test_pauseAllowsRecoveryBlocksOrigination` |
| 34 | Read-only `Rules_GetPaused` succeeds while paused | no guard on the read | `test_publicFetchWhilePaused` |
| 35 | Mint requires a `Minter` capability, a positive amount, and a supported instrument; mint is blocked while paused (origination) | `Rules_Mint` + `consumeMintAllowance` | `test_mintRequiresMinterCapability`, `test_mintBlockedWhilePaused`, `test_mintDirectViaPreapproval`, `test_mintProposalWhenNoPreapproval` |
| 36 | A capped `Minter` cannot mint beyond its remaining `mintAllowance` (D3); each mint decrements it | `consumeMintAllowance` | `test_cappedMinterAllowance` |
| 37 | The advisory `TotalSupply` (D2) reconciles up/down and never goes negative | `TotalSupply_AdjustMinted` | `test_totalSupplyObservability` |
| 38 | Forced burn (incl. of in-flight/locked funds) requires a `Burner` capability in scope; owner redemption needs none | `SimpleHolding_ForcedBurn`, `LockedSimpleHolding_ForcedBurn`, `SimpleHolding_Burn` | `test_forcedBurnRequiresBurnerCap`, `test_forcedBurnScopedToInstrument`, `test_forcedBurnLockedHolding`, `test_ownerRedemptionBurn` |
| 39 | Every `SimpleHolding`/`LockedSimpleHolding` has `instrumentId.admin == admin` (capability scoping stays keyed to a consistent admin) | template `ensure` clauses | covered by the holding `ensure` + factory paths |

\* INV-27's branch is exercised indirectly; a dedicated admin-mismatch test is a noted future addition.

---

## 8. Verification (extends [AUDIT.md](AUDIT.md))

- **daml-lint:** clean on `Admin/*` — `Role` is a closed sum type and no admin
  template has unbounded list fields (`scope` is `Optional`, not a list). The 7
  acknowledged `unbounded-fields` MEDIUMs are all pre-existing on other templates.
- **compiler-enforced policy:** `simple-token` builds with
  `--ghc-option=-Werror=incomplete-patterns`, so adding a `Role` constructor
  without a `delegableRole` clause is a **compile error**, not a runtime
  pattern-match crash — the delegation policy cannot silently regress.
- **daml-props / dpm test:** 141/141, incl. 15 `Test/Admin.daml` + 10
  `Test/MintBurn.daml` rows.
- **daml-verify:** 14/14 proved (transfer/allocation conservation + temporal). The
  `admin-authorization` capability relation, a `scopeAuthorizes` lemma, and a
  mint-allowance conservation property remain Z3 proof *targets* (the symbolic
  model has no capability relation yet); today they are covered by Daml Script.
  Closing these is **AL-4**, implemented by extending the `daml-verify` /
  `daml-props` repos under `$CANTON_TOOLS_HOME/tools/` via feature branches + PRs
  (the upstream OpenZeppelin tool repos are the source of truth, not the
  `scripts/setup.sh` convenience clones) — see
  [PLAN.md §17.6](PLAN.md#176-tooling-workspace-canton_tools_home--contribution-model).

---

## 9. Test plan (`Test/Admin.daml`, 14 tests)

Pausable: `test_pauseBlocksTransfer`, `test_pauseBlocksAllocation`,
`test_unpauseRestores`, `test_pauseRequiresPauserCap`,
`test_scopedCapabilityCannotPauseRegistry`, `test_publicFetchWhilePaused`,
`test_pauseAllowsRecoveryBlocksOrigination` (origination-blocked vs
recovery-allowed under one paused state; asserts the **unlocked** balance so the
recovery is load-bearing).
AccessControl: `test_capabilityImpersonationFails`, `test_revokeRole`.
Ownable-as-Admin-role: `test_delegatedAdminGovernance` (handoff; also asserts a
delegate cannot re-delegate `Admin`), `test_revokedAdminCannotDelegate`,
`test_nonAdminCannotDelegateIssue` (visible-but-unauthorized → fails the role
check, not visibility), `test_delegatedRevoke` (delegate revokes an operational
cap), `test_delegatedCannotRevokeAdmin` (delegate cannot revoke an `Admin` cap).

---

## 10. Out of scope (named seams, not gaps)

- Per-role admin hierarchy (`getRoleAdmin`) — seam in `Admin_DelegatedIssueRole`.
- Two-step / timelocked issuance — optional `RoleOffer` wrapper.
- Multisig / threshold owner — Canton topology (multi-hosted party), no code.
- Per-asset emergency **freeze** distinct from pause — future capability (§4).
- On-ledger reassignment of the instrument `admin` party — impossible by design
  (C9); operator key handoff is a Canton topology operation.
- CDP / custody / bridge roles — sibling stablecoin & bridge plans consume the
  generic `Role` primitive.

---

## 11. Supply management — Mint & Burn (AL-5 / AL-6)

The first consumers of the `Minter`/`Burner` roles. Source: `SimpleToken/Supply.daml`
(`TotalSupply`, `MintProposal`), `Rules_Mint` on `SimpleTokenRules`,
`TransferPreapproval_MintInto`, and `SimpleHolding_Burn` / `SimpleHolding_ForcedBurn`
/ `LockedSimpleHolding_ForcedBurn`; tests in `Test/MintBurn.daml`.

### 11.1 The core constraint (why this isn't EVM mint/burn)

EVM `_mint`/`_burn` edit a balance mapping unilaterally. On Canton a holding is
`signatory admin, owner`, and **you cannot create or archive a contract signed by
a party without that party's authority.** So minting *into* an account needs the
recipient's consent, and burning needs a choice the owner consented to at
creation. The design follows from that.

### 11.2 Chosen architecture (with the alternatives weighed)

| Dimension | Choice | Why (vs. alternatives) |
|---|---|---|
| **Mint recipient consent** | **A3 dispatch** — direct mint via the recipient's `TransferPreapproval` if present, else a `MintProposal` they accept | Mirrors the transfer factory's 3-way dispatch; reuses the tested preapproval mechanism for the warm path and needs no pre-setup for the cold path. (A1 alone forces a preapproval; A2 alone is always two-step.) **A capped minter (D3) may only use the direct path** — see §11.3. |
| **Burn trust model** | **B3** — owner `SimpleHolding_Burn` (redemption, no cap) + `Burner`-gated `SimpleHolding_ForcedBurn` (clawback) | Institutional RIs (RWA/stablecoin) need both. Forced burn is the deliberate trust shift — the registry can *destroy* a holding, not only move it with owner consent — so it is capability-gated, instrument-scoped, and called out here. (B1 can't do compliance clawback; B2 drops first-class redemption.) |
| **Supply cap (enforced)** | **D3** — `mintAllowance : Optional Decimal` on the `Minter` capability; checked and decremented per mint | A *sound, enforceable* bound with no global hot contract — contention is per-minter. Decrementing rotates the capability CID (the accepted D3 cost). `None` = unlimited. |
| **Supply total (observable)** | **D2** — `TotalSupply`, `(admin, instrumentId)`-anchored, reconciled off-ledger via `TotalSupply_AdjustMinted` | On-ledger auditability **without** putting a singleton on every op's hot path. Deliberately *not* updated inline by mint/burn — that would reintroduce the contention and disclosure-to-every-actor cost, and keyless LF 2.1 cannot enforce singleton uniqueness anyway (so an inline cap would be bypassable). D2 is therefore advisory/eventually-consistent observability; **D3 is the real cap.** |
| **Gating location** | `Rules_Mint` on the factory (admin authority, consistent with pause); burn choices on `SimpleHolding` (only place with the owner's at-creation consent for forced burn) | Mint can't live on the holding (the holding doesn't exist yet); forced burn can't live on the factory (no owner authority there). |
| **Scope** | Mint/burn gate on **instrument-scoped** capabilities (`requireRole … (Some instrumentId)`) | Finally exercises the `scope` field — a USD minter/burner can't touch EUR — giving `scopeAuthorizes` real per-instrument tests. |

### 11.3 Governance & pause interaction

- `Minter`/`Burner` are **enforced** but kept **root-managed** (`delegableRole = False`):
  appointing who may create or destroy supply is a root-level decision, not
  something an `Admin` delegate hands out. Flip the `delegableRole` case to make
  either delegable.
- **Mint is origination → `whenNotPaused`.** **Burn is not pause-gated** (reduction /
  redemption / compliance should remain available in an emergency, consistent with
  INV-33).
- **D3 enforcement is atomic, so capped minting is direct-path only.** The proposal
  (cold-recipient) path splits the mint across two transactions controlled by
  different parties; the allowance can't be enforced/decremented atomically there,
  and pre-debiting at proposal time would leak allowance on reject/withdraw. So a
  **capped** `Minter` must mint into a pre-authorized recipient (direct path) —
  `Rules_Mint` rejects the proposal path for capped minters
  (`eCappedMinterRequiresPreapproval`). **Unlimited** minters use the proposal path
  freely (no allowance to leak; the capability is validated but not consumed).
- **A `MintProposal` is a committed origination.** The minter, capability, pause,
  and supported-instrument were all validated at `Rules_Mint` time; the recipient's
  later `MintProposal_Accept` *completes* an already-authorized mint, so — like a
  committed transfer/allocation — it is not re-gated on pause. (Robust pause is
  only enforceable at the factory chokepoint in a keyless model; see §4.)

### 11.4 Accepted limitations (surfaced, not hidden)

- D2 `TotalSupply` is advisory and eventually-consistent (service-reconciled); it
  is **not** a tamper-proof cap, and there is no on-ledger uniqueness or
  anchor-match enforcement (keyless). The enforceable cap is D3 (`mintAllowance`).
- `TransferPreapproval_MintInto` is controlled by `admin`, so the registry root can
  exercise it directly as an **unmetered root mint** (bypassing the `Minter` cap,
  D3, pause, and the supported-instrument check). This is consistent with "admin is
  root" (it can equally self-issue an unlimited `Minter`); non-admins cannot reach
  it. `Rules_Mint` is the capability-gated/metered entry point.
- Owner `SimpleHolding_Burn` lets a holder self-destroy funds ahead of a clawback;
  seizure can't be made race-free against a consenting owner (they could also
  transfer the funds away). The off-ledger reconciler distinguishes redemption
  (owner actor) from forced burn (burner actor) by the archiving event.
- **In-flight (locked) clawback** IS supported via `LockedSimpleHolding_ForcedBurn`,
  which seizes the funds *immediately*. The cost is a **bounded dangling wrapper**:
  the `SimpleTransferInstruction` / `SimpleAllocation` that referenced the seized
  locked holding can no longer complete and lingers until its own deadline
  (`executeBefore`/`settleBefore`), after which it is cleaned up by the standard
  expire-lock recovery (returning nothing). We accept this rather than add a
  `Burner`-gated force-archive choice to every wrapper type (transfer instruction,
  V1/V2 allocation, allocation-instruction): that would add more surface/risk than
  the bounded, self-cleaning orphan it removes. Funds are never at risk — they are
  destroyed at seizure time. Verified by `test_forcedBurnSeizesInFlightTransfer`.
- **Pending `MintProposal`s are not cancelled** by revoking the minter's capability
  or by pausing — a proposal is a committed origination, and only
  `MintProposal_Withdraw` (admin) or recipient `MintProposal_Reject` stops it.
- **Clawback is V1-only.** `SimpleHolding_ForcedBurn` / `LockedSimpleHolding_ForcedBurn`
  cover the V1 holdings; the experimental V2 provider-managed holdings
  (`ProviderManagedSimpleHolding` / `ProviderManagedLockedHolding`) have no
  forced-burn yet. Closing that is a V2-hardening follow-up, gated on the V2
  source-of-record (it is not part of the AL-6 V1 clawback scope).
- Forced burn is a powerful capability: a `Burner` holder can destroy a holder's
  funds. It is instrument-scoped and capability-gated, and `Burner` is root-only,
  but operators must treat issuing a `Burner` capability as a high-trust action.
