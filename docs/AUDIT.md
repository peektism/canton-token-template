# Verification Report: Canton Network Token Standard

Three open-source verification tools and manual code review by the author were used to analyze this codebase. All findings have been patched or acknowledged.

## Summary

| Metric | Value |
|--------|-------|
| Tools used | 3 (daml-lint, daml-props, daml-verify) |
| Findings identified | 5 |
| Patched | 2 (G1, G2) |
| Acknowledged | 1 (G4 — contract keys dropped in Daml LF 2.1) |
| Low severity (test only) | 1 (G5 — head-of-list in test harness) |
| Informational | 1 (unbounded list fields — admin-only creation) |
| Formal properties proved | 9/9 legacy rows; V2 rows are re-baselining targets |
| Property/model tests passed | 9/9 rows (7 randomized, 2 deterministic) |

---

## Tools Used

> **Tool provenance.** The three tools are independent OpenZeppelin repos that
> live in the shared workspace `$CANTON_TOOLS_HOME` (`/Users/amar/canton-tools`):
> `tools/daml-lint`, `tools/daml-props`, `tools/daml-verify` (see
> `$CANTON_TOOLS_HOME/AGENTS.md`). The `canton-token-template/tools/` copies are
> convenience clones produced by `scripts/setup.sh`; the **canonical source and
> the target for extensions (e.g. AL-4's `daml-verify` capability relation) is the
> upstream repo, contributed via a feature branch + PR** — see
> [PLAN.md §17.6](PLAN.md#176-tooling-workspace-canton_tools_home--contribution-model).

### daml-lint (Static Analysis)

Static analyzer for DAML that catches security anti-patterns through AST pattern matching. 6 detectors covering missing ensure clauses, unguarded division, missing positive-amount checks, archive-before-execute, head-of-list on queries, and unbounded fields.

**How it was run:**

```bash
cd daml-lint && cargo build --release

# Production code
daml-lint canton-network-token-standard/simple-token/daml/ --format markdown

# Test code
daml-lint canton-network-token-standard/simple-token-test/daml/ --format markdown
```

**Production code results (simple-token):**

| Detector | Severity | File | Finding |
|----------|----------|------|---------|
| `unbounded-fields` | MEDIUM | AllocationRequest.daml:12 | `senders` list without length bound |
| `unbounded-fields` | MEDIUM | Holding.daml:32 | `extraObservers` list without length bound |
| `unbounded-fields` | MEDIUM | Rules.daml:20 | `supportedInstruments` list without length bound |

No HIGH findings. All three MEDIUM findings are `unbounded-fields` on List parameters in templates where creation is admin-controlled.

**Test code results (simple-token-test):**

| Detector | Severity | File | Finding |
|----------|----------|------|---------|
| `head-of-list-query` | MEDIUM | Defragmentation.daml:80 | Head pattern on queryFilter |
| `head-of-list-query` | MEDIUM | Negative.daml:233 | Head pattern on queryFilter |
| `head-of-list-query` | MEDIUM | SimpleRegistry.daml:153 | Head pattern on queryFilter |

Test harness only. Non-deterministic query ordering does not affect test correctness in single-party sandbox.

### daml-verify (Formal Verification)

Lightweight formal verification using Z3 SMT solver. Proves critical invariants hold for **all possible inputs**, not just sampled test cases. 9 properties across conservation, division safety, and temporal ordering.

**How it was run:**

```bash
cd daml-verify
source .venv/bin/activate
python main.py
```

**Results:**

```
daml-verify: 9 properties, 9 proved, 0 disproved

  [PROVED] C1: conservation total
  [PROVED] C2: receiver amount
  [PROVED] C3: sender change
  [PROVED] D1: scaleFees safety
  [PROVED] D2: issuance safety
  [PROVED] D3: ensure sufficient
  [PROVED] T1: transfer temporal
  [PROVED] T2: allocation temporal
  [PROVED] T3: lock expiry
```

### daml-props (Property-Based Testing)

Pure DAML property-based testing library with shrinking. Generates random action sequences, checks invariants hold for all, and shrinks failing cases to minimal counterexamples.

**How it was run:**

```bash
cd /Users/x/canton/tools/daml-props && dpm build
export JAVA_HOME=/opt/homebrew/Cellar/openjdk@21/21.0.10/libexec/openjdk.jdk/Contents/Home
export PATH="$JAVA_HOME/bin:$PATH"
cd /Users/x/canton/tools/canton-token-template/simple-token-test && dpm test
```

**Results:** All 9 SimpleToken property/model rows passed. The original 5 V1
property tests and 2 experimental V2 iterated-settlement property rows run 200
random sequences each; the 2 stale-replay regressions are deterministic rows.

---

## Findings

### G1: Missing `ensure amount > 0.0` (Patched)

**Severity:** HIGH
**Detector:** daml-lint `missing-ensure-decimal`
**Location:** Holding.daml:19, Holding.daml:44
**Description:** `SimpleHolding` and `LockedSimpleHolding` lacked `ensure amount > 0.0` clauses, permitting zero-value holdings.
**Status:** Patched in commit `509e1f8` ("tdd for all hardening fixes").

### G2: Preapproval amount validation (Patched)

**Severity:** HIGH
**Detector:** daml-lint `missing-positive-amount`
**Location:** Preapproval.daml:39-40
**Description:** `TransferPreapproval_Send` accepted `amount` parameter without asserting `> 0`.
**Status:** Patched in commit `509e1f8`.

### G4: Missing contract keys (Acknowledged)

**Severity:** MEDIUM
**Source:** Manual review by the author
**Location:** Rules.daml:20-25, Preapproval.daml:14-22
**Description:** `SimpleTokenRules` and `TransferPreapproval` lack contract keys, permitting duplicate instances. Daml LF 2.1 dropped contract keys entirely. Single-admin architecture limits blast radius to admin-caused duplicates only.
**Status:** Acknowledged. Not patchable under Daml LF 2.1.

### G5: Head pattern on queryFilter (Low — test only)

**Severity:** LOW
**Detector:** daml-lint `head-of-list-query`
**Location:** SimpleRegistry.daml:153, Defragmentation.daml:80, Negative.daml:233
**Description:** `:: _` pattern on queryFilter results in test harness. Non-deterministic ordering is safe in single-party sandbox but would be unsafe in production code.
**Status:** Test harness only. No production impact.

### Unbounded list fields (Informational)

**Severity:** INFO
**Detector:** daml-lint `unbounded-fields`
**Location:** AllocationRequest.daml:12, Holding.daml:32, Rules.daml:20
**Description:** Three templates have `List` fields without ensure clauses bounding length. Template creation requires admin signatory, limiting abuse to admin-controlled operations.
**Status:** Acknowledged. Admin-only creation provides sufficient mitigation.

---

## Conservation Proofs (daml-verify)

These proofs establish that the transfer engine cannot create or destroy tokens. They hold for **all possible inputs** — not sampled test cases.

| Property | Statement | Result |
|----------|-----------|--------|
| C1: conservation total | `totalInput == receiverAmount + senderChange` for all transfer paths | PROVED |
| C2: receiver amount | Receiver gets exactly `transfer.amount` in self-transfer, direct, and two-step-accept | PROVED |
| C3: sender change | `senderChange == totalInput - transfer.amount` (non-negative by D3) | PROVED |

**Model fidelity:** Symbolic models in `daml_verify/model/transfer.py` were compared line-by-line to `Rules.daml`, `Preapproval.daml`, `TransferInstruction.daml`, and `Allocation.daml`. All arithmetic relationships are exact. Abstractions (per-input validation, lock expiry handling) do not affect conservation arithmetic.

27 guards were cataloged across all source files: 2 template ensure clauses, 8 transfer factory guards, 8 allocation factory guards, 3 per-input guards, and 6 choice-level guards.

### CIP-112 V2 Proof Rows

The V2 rows below are explicit proof targets for the preview-based
implementation. They are not formal proof claims until `daml-verify` grows a
per-account allocation model and is re-run against this code.

| Property | Statement | Result |
|----------|-----------|--------|
| CIP112-C1: iterated reserve conservation | For each iterated sender-side settlement, `lockedBefore + netCredit == authorizerPayout + nextReserve`; repeated settlement cannot create or destroy value. | TARGET; covered today by Daml Script and `daml-props` regressions |
| CIP112-C2: receiver-side top-up conservation | Receiver-side next reserve may be funded by `incomingCredit + authorizerTopUp`; the successor reserve must equal requested `nextIterationFunding`. | TARGET; covered today by focused Daml Script |
| CIP112-A1: settlement actor authorization | Direct `Allocation_Settle` requires `admin + executors`; factory settlement requires `executors` and supplies default allocation actors. | TARGET; covered today by focused negative tests |
| CIP112-A2: direct/factory validation boundary | Direct `Allocation_Settle` performs local actor and extra-side argument validation only; full batch transfer-leg shape and authorization matching remain factory-owned. | TARGET; covered today by focused negative tests |

### Admin Layer Invariants (AccessControl / Ownable-as-Admin-role / Pausable)

Introduced by slices AL-0..AL-3 (see [ADMIN-LAYER-PLAN.md §7](ADMIN-LAYER-PLAN.md#7-security-invariants-extends-planmd-9)).
Covered today by Daml Script tests in `Test/Admin.daml`; the `daml-verify` rows
are proof *targets*, not completed claims, until the symbolic model grows a
capability relation.

| Invariant | Statement | Status |
|-----------|-----------|--------|
| INV-25 | No factory **origination** choice (transfer V1/V2, allocation V1/V2, settlement-via-factory) succeeds while `paused` | Script: `test_pauseBlocksTransfer`, `test_pauseBlocksAllocation` |
| INV-26 | Pause/unpause requires a **registry-wide** `Pauser` capability | Script: `test_pauseRequiresPauserCap` |
| INV-27 | Capability honored only if `cap.admin == registry admin` | Script: `test_capabilityImpersonationFails` (indirect) |
| INV-28 | Capability honored only if `cap.assignee == caller` (anti-impersonation) | Script: `test_capabilityImpersonationFails` |
| INV-29 | Capability honored only if `cap.role == required role` | Script: `test_pauseRequiresPauserCap`, `test_nonAdminCannotDelegateIssue` |
| INV-30 | Capability scope must authorize the operation (instrument-scoped cap ≠ registry-wide op) | Script: `test_scopedCapabilityCannotPauseRegistry` |
| INV-31 | Capabilities issued/revoked only by the root `admin` or an `Admin`-capability holder; the **`Admin` role is root-managed** (delegates cannot issue/revoke `Admin`) | Script: `test_delegatedAdminGovernance`, `test_nonAdminCannotDelegateIssue`, `test_delegatedRevoke`, `test_delegatedCannotRevokeAdmin` |
| INV-32 | A revoked capability can no longer authorize | Script: `test_revokeRole`, `test_revokedAdminCannotDelegate` |
| INV-33 | Pause is origination control, not a fund freeze: recovery + committed-settlement completion stay open | Script: `test_pauseAllowsRecoveryBlocksOrigination` |
| INV-34 | Read-only `Rules_GetPaused` succeeds while paused | Script: `test_publicFetchWhilePaused` |
| INV-35 | Mint requires a `Minter` cap, positive amount, supported instrument; blocked while paused | Script: `test_mintRequiresMinterCapability`, `test_mintBlockedWhilePaused`, `test_mintDirectViaPreapproval`, `test_mintProposalWhenNoPreapproval` |
| INV-36 | A capped `Minter` cannot exceed its `mintAllowance` (D3); each mint decrements it | Script: `test_cappedMinterAllowance` |
| INV-37 | Advisory `TotalSupply` (D2) reconciles up/down, never negative | Script: `test_totalSupplyObservability` |
| INV-38 | Forced burn (incl. in-flight/locked funds) requires an in-scope `Burner` cap; owner redemption needs none | Script: `test_forcedBurnRequiresBurnerCap`, `test_forcedBurnScopedToInstrument`, `test_forcedBurnLockedHolding`, `test_ownerRedemptionBurn` |
| INV-39 | Every `SimpleHolding`/`LockedSimpleHolding` has `instrumentId.admin == admin` | template `ensure` clauses |
| INV-40 | A frozen account (AL-10 compliance hold) cannot ORIGINATE at any chokepoint — as sender, receiver, or mint recipient (V1/V2 transfer, V1/V2 allocation, batch settlement, direct mint, **and accepting a cold-path `MintProposal`** via `Rules_AcceptMintProposal`). V2 guards check `accountParties` (owner **and** provider), so freezing either blocks. Freeze is **origination control** like pause (INV-33): inbound committed flows still complete — seizing those is AL-11. | Script: `test_freezeBlocksTransferSender`/`Receiver`/`Allocation`/`Mint`, `test_unfreezeRestoresTransfer`, `test_freezeBlocksV2BasicTransfer`, `test_freezeProviderBlocksProviderManagedTransfer`, `test_mintProposalAcceptBlockedWhenRecipientFrozen` |
| INV-41 | `Rules_SetAccountFrozen` (the `Bool`-parameterised freeze/unfreeze, mirroring `Rules_SetPaused`) requires a **registry-wide** `ComplianceAdmin` capability (instrument-scoped ≠ registry-wide op; manual, capability-gated) | Script: `test_freezeRequiresComplianceAdmin`, `test_scopedComplianceAdminCannotFreeze`, `test_complianceAdminDelegableAndObservable` |
| INV-42 | A redundant freeze change is rejected (re-freezing a held / unfreezing an unheld account — OZ Pausable non-idempotency) and the registry administrator can never be frozen | Script: `test_freezeNonIdempotentAndAdminProtected` |
| admin-authorization | `requireRole` rejects every `(caller, role, admin, scope)` tuple not matching a co-signed capability; `assertNotPaused` is a total guard on the origination set; capped mint is conserved | `daml-verify` **PROVED** (A1–A8, [OpenZeppelin/daml-verify#4](https://github.com/OpenZeppelin/daml-verify/pull/4)) + `daml-props` **state-machine** (pause / authorization / conservation over random sequences, [OpenZeppelin/daml-props#2](https://github.com/OpenZeppelin/daml-props/pull/2)); both show locally after the PRs merge + `setup.sh` re-pull |
| role-admin (AL-8) | granting R requires `roleAdmin(R)`; root `Admin` never delegable; renounce self-only; timelock not bypassable | `daml-verify` **PROVED** (A9–A13, [#5](https://github.com/OpenZeppelin/daml-verify/pull/5)) + `daml-props` `RoleAdmin` state-machine ([#3](https://github.com/OpenZeppelin/daml-props/pull/3)) |
| freeze (AL-10) **gate logic** | the unified `assertCanOriginate` gate is exactly `pause ∧ freeze` (non-vacuous biconditional); the admin is never freezable; freeze change is non-idempotent | `daml-verify` **PROVED** (A14–A17, [OpenZeppelin/daml-verify#6](https://github.com/OpenZeppelin/daml-verify/pull/6); `isFrozen` uninterpreted) + `daml-props` `Freeze` state-machine ([OpenZeppelin/daml-props#4](https://github.com/OpenZeppelin/daml-props/pull/4)); both stacked on the AL-8 branches |
| freeze (AL-10) **wiring & V2/cold-path** | the gate is invoked at all 5 origination chokepoints + mint + cold-path accept; V2 checks owner **and** provider (`accountParties`); a recipient frozen in the propose→accept window cannot be minted into | **daml-script** (`Test/Freeze.daml`, incl. `test_freezeProviderBlocksProviderManagedTransfer`, `test_mintProposalAcceptBlockedWhenRecipientFrozen`) + **daml-props** `Freeze` (account owner+provider model + propose→accept). The abstract Z3 proofs verify the gate *logic*, not its per-chokepoint placement or the V2 account expansion — those are covered here. |

> daml-lint note: `Role` is a closed sum type and no admin template has unbounded
> list fields (`RoleCapability.scope` is `Optional`, not a list) — no new
> `unbounded-fields` findings.

**Verification run (2026-06-04, SDK 3.4.11 / DPM 1.0.17 / OpenJDK 21.0.11):**
`scripts/verify.sh` → **3 passed, 0 failed**.
- `daml-lint`: PASS. **No findings in `SimpleToken/Admin/*`, `Supply.daml`,** or the
  new `SimpleTokenRules`/`SimpleHolding`/`Preapproval` choices. The 7
  `unbounded-fields` MEDIUMs are all pre-existing on other templates
  (`extraObservers`, `supportedInstruments`, `senders`, `allocations`,
  `inputHoldingCids`) — admin-only creation mitigates.
- `daml-props` (`dpm test`): PASS — **141/141** (incl. 15 `Test/Admin.daml` + 10
  `Test/MintBurn.daml` rows). The admin-layer state-machine rows (pause /
  role-authorization / mint-allowance conservation, 5 property tests in
  `Examples/AdminLayer/`) are **implemented and passing upstream** in
  [OpenZeppelin/daml-props#2](https://github.com/OpenZeppelin/daml-props/pull/2);
  they join the local count once that PR merges and `scripts/setup.sh` re-pulls.
- `daml-verify`: PASS — 14/14 properties in the local `setup.sh` clone. The 9
  C/D/T rows model this repo's transfer/allocation logic; the V1–V5 rows are the
  tool's own stablecoin models. The AccessControl/Pausable/mint-cap proof rows
  (A1–A8: `admin-authorization`, `scopeAuthorizes`, capped-mint conservation,
  pause guard) are **implemented and proving upstream** in
  [OpenZeppelin/daml-verify#4](https://github.com/OpenZeppelin/daml-verify/pull/4)
  (AL-4 — 22/22 proved, pytest 22/22); they appear in the local count once that PR
  merges and `scripts/setup.sh` re-pulls the tool.

## Temporal Proofs (daml-verify)

These proofs establish that time-dependent logic is consistent.

| Property | Statement | Result |
|----------|-----------|--------|
| T1: transfer temporal | `requestedAt <= now` and `executeBefore > now` implies `requestedAt < executeBefore` | PROVED |
| T2: allocation temporal | `requestedAt <= now` and `allocateBefore > now` and `allocateBefore <= settleBefore` implies ordering | PROVED |
| T3: lock expiry | Lock `expiresAt == executeBefore` and `executeBefore > now` implies lock is still active | PROVED |

## Division Safety Proofs (daml-verify)

| Property | Statement | Result |
|----------|-----------|--------|
| D3: ensure sufficient | `totalInput >= requested > 0` implies `senderChange >= 0` (no negative balances from change) | PROVED |

D1 and D2 are Splice-specific properties (scaleFees and issuance tranche division safety). They prove that adding guards to Splice's `amuletPrice` and `capPerCoupon` fields would eliminate division-by-zero. CNTS has no unguarded division.

## Property-Based Testing Results (daml-props)

Pure state-machine model of the transfer engine plus an experimental V2
iterated-settlement accounting model. The V1 model uses 4 parties and 6 action
types (self-transfer, direct transfer, two-step initiate/accept/reject/withdraw).
The V2 model tracks one authorizer, one receiver, active successor reserve,
active allocation id, consumed allocation ids, successor allocation ids,
partial settlement, finalization, cancellation, withdrawal, explicit stale
replay target ids, and rejected repeated settlement attempts. Random property
rows run 200 sequences; deterministic stale-replay rows run one sequence each.

| Test | Property | Sequences | Max Length | Result |
|------|----------|-----------|------------|--------|
| `test_simpleTokenConservation` | Total supply (holdings + locked) is constant | 200 | 15 | PASS |
| `test_simpleTokenPositiveAmounts` | All holdings have amount > 0 | 200 | 15 | PASS |
| `test_simpleTokenNonNegativeBalances` | No party's balance goes negative | 200 | 15 | PASS |
| `test_simpleTokenLifecycle` | Full two-step lifecycle preserves invariants | 200 | 20 | PASS |
| `test_simpleTokenSelfTransferExact` | Self-transfer of exact balance produces single output | 200 | 1 | PASS |
| `test_v2IteratedSettlementConservation` | V2 iterated reserve + authorizer payout + receiver credit stays constant | 200 | 15 | PASS |
| `test_v2IteratedSettlementNoDoubleSettle` | Replays against consumed allocation ids do not alter modeled reserve or double-settle value | 200 | 25 | PASS |
| `test_v2IteratedSettlementStaleReplayModelTracksRejectedRepeat` | Replaying explicit consumed allocation id `1` records a rejected repeat without changing accounting fields | deterministic | 1 | PASS |
| `test_v2IteratedSettlementStaleReplayModelIgnoresLiveTarget` | Replaying explicit live allocation id `2` does not record a consumed-allocation repeat or change accounting fields | deterministic | 1 | PASS |

**Methodology:** Executors return `Right state` (no-op) for invalid preconditions (insufficient funds, bad amounts), matching Echidna/Foundry semantics. `Left` is reserved for true invariant violations. Generators bias toward edge cases using `genFrequency` for weighted selection.

---

## Not Applicable: Splice Issues

The following 22 MEDIUM-severity issues were identified through tool-based analysis (daml-lint, daml-verify) and manual review of the Splice codebase. They apply to Splice only, not to this project:

| ID | Splice Issue | Why N/A for CNTS |
|----|-------------|-----------------|
| M1 | Missing singleton enforcement on ValidatorLicense | No ValidatorLicense template |
| M2 | No queryFilter dedup on FeaturedAppRight | No FeaturedAppRight template |
| M3 | AppRewardCoupon minted without input validation | No reward coupon minting |
| M4 | ValidatorRewardCoupon round mismatch allowed | No reward coupons |
| M5 | Unchecked context map keys | ChoiceContext validated per-key |
| M6 | DsoRules_ConfirmAction majority threshold float | No DSO governance |
| M7 | Unbounded Text in vote reasons | No governance voting |
| M8 | AmuletPrice median edge cases | No price oracle |
| M9 | OpenMiningRound missing amuletPrice > 0 ensure | No mining rounds |
| M10 | SummarizingMiningRound unbounded arrays | No mining rounds |
| M11 | ExternalPartyAmuletRules missing input check | No external party rules |
| M12 | BuyTrafficRequest missing amount validation | No traffic purchases |
| M13 | WalletAppInstall batch empty-list | No wallet batching |
| M14 | MintingDelegation empty inputs | No minting delegation |
| M15 | AcceptedTransferOffer empty inputs | No transfer offers |
| M16 | TransferPreapprovalProposal empty inputs | Different preapproval design |
| M17 | Issuance capPerCoupon >= 0 allows zero | No issuance config |
| M18 | Issuance.daml:139 division by amuletPrice | No issuance logic |
| M19 | computeSynchronizerFees division by amuletPrice | No synchronizer fees |
| M20 | computeTransferPreapprovalFee division by amuletPrice | No preapproval fees |
| M21 | DsoRules fetchAndArchive before try/catch | No DSO governance |
| M22 | Unbounded Text fields in DsoRules/VoteRequest | No DSO governance |

---

## Methodology

### Model Fidelity

All symbolic models and pure state-machine models were validated against actual DAML source code through line-by-line comparison:

- **daml-verify models** (`transfer.py`, `allocation.py`, `fees.py`): 27 guards inventoried, all arithmetic relationships verified exact. Three discrepancies found and corrected during dogfooding (see `daml-verify/DOGFOOD.md`).
- **daml-props models** (`SimpleToken/Model.daml`): Pure executor faithfully reproduces all 6 V1 transfer action types from `Rules.daml` and adds an experimental V2 iterated-settlement accounting model with allocation lifecycle ids, explicit stale-replay target ids, and rejected repeated-settlement tracking. Invalid preconditions return `Right state` (no-op).

### Unmodeled Paths

Five conservation-preserving paths are not explicitly modeled (all are 1-holding-in, 1-holding-out of same amount):

- `transferInstruction_rejectImpl` (return locked funds to sender)
- `transferInstruction_withdrawImpl` (return locked funds to sender)
- `allocation_cancelImpl` (release allocated funds)
- `LockedSimpleHolding_Unlock` (unlock expired lock)
- Expire-lock pattern in `returnLockedFundsToSender`

### Tool Versions

- daml-lint: built from source (`cargo build --release`)
- daml-props: v0.1.0 (pure DAML, SDK 3.4.10, target 2.1)
- daml-verify: Python 3.10+, z3-solver >= 4.12
- DAML SDK: 3.4.10
