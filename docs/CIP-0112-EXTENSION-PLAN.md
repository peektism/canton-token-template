# CIP-0112 Extension Plan: canton-token-template

Date: 2026-05-19
Status: Active M1 implementation per
`/Users/x/canton/docs/decisions/2026-05-19-cip-112-m1-retarget.md` (Status:
accepted 2026-05-19). Daml source work in `simple-token/` and
`simple-token-test/` is active M1 scope against the accepted frozen-Draft
snapshot at `canton-foundation/cips` commit `a07d8db7…` and the accepted
Splice preview implementation reference at `b91de5d4…`.
Owner: OpenZeppelin technical lead
Reviewer: Digital Asset technical contact, CIP-0112 authors, OpenZeppelin
security lead for authority/privacy slices

## Scope

This is the per-tool extension plan for the comprehensive CIP-0112 (Token
Standard V2) implementation alongside the existing CIP-0056 V1
implementation in `tools/canton-token-template/simple-token/`. It is a
child of the workspace-level plan at
`/Users/x/canton/docs/architecture/cip-112-extension-plan.md`.

Active M1 implementation covers the V2 surface: basic-account and
provider-managed `Account` semantics, multi-leg `Allocation`,
`SettlementFactory_SettleBatch`, allocation-request workflows, iterated
settlement, receipt/reporting and event reconstruction, cross-admin
coordination, production-shaped disclosure/privacy policy, LocalNet/client
smoke paths, and compatibility-matrix expansion against CIP-0112 §5.4
transfer and §5.5 allocation rows. Public API extraction into
`repos/oz-daml-contracts/` follows the workspace-level slice 18
RI-backed extraction sequence; it is not part of this per-tool plan.

## Anchors in the existing codebase

`simple-token/daml/SimpleToken/` (per `tools/canton-token-template/docs/SCOPE.md`
§3):

- `Holding.daml` — `SimpleHolding`, `LockedSimpleHolding`
- `TransferInstruction.daml` — `SimpleTransferInstruction`
- `Allocation.daml` — `SimpleAllocation`, `SimpleAllocationRequest`
- `Rules.daml` — `SimpleTokenRules` (the factory; multi-instrument support
  via `supportedInstruments`)
- `Preapproval.daml` — `TransferPreapproval`
- `ContextUtils.daml` — shared utilities including `archiveAndSumInputs`,
  metadata helpers
- `AllocationRequest.daml` — `SimpleAllocationRequest`,
  `SimpleAllocationRequestV2`

Historical baseline tests in `simple-token-test/`: 9 transfer + 5 allocation
+ 2 defrag + 20 security = 36 total. The M1 CIP-0112 test impact table below
records the additional experimental V2 probes now present in this tool.

V1 DARs vendored in `dars/`:

- `splice-api-token-allocation-instruction-v1-1.0.0.dar`
- `splice-api-token-allocation-request-v1-1.0.0.dar`
- `splice-api-token-allocation-v1-1.0.0.dar`
- `splice-api-token-holding-v1-1.0.0.dar`
- `splice-api-token-metadata-v1-1.0.0.dar`
- `splice-api-token-transfer-instruction-v1-1.0.0.dar`

## Prototype evidence

Preview DAR provenance for the allocation/settlement/event files staged with
this prototype:

| Local DAR | Splice preview source | SHA-256 |
| :--- | :--- | :--- |
| `dars/splice-api-token-allocation-instruction-v2-1.0.0.dar` | `/Users/x/excanton/CN/splice` `origin/token-standard-v2-daml-preview` commit `b91de5d4b910ded598151981654dce2acc6f84ba`, path `daml/dars/splice-api-token-allocation-instruction-v2-1.0.0.dar` | `e1de9d448bd3c67d68bb50c73e51d4e0bbe53d6e8f5f9662c992f7c609cfd9aa` |
| `dars/splice-api-token-allocation-request-v2-1.0.0.dar` | `/Users/x/excanton/CN/splice` `origin/token-standard-v2-daml-preview` commit `b91de5d4b910ded598151981654dce2acc6f84ba`, path `daml/dars/splice-api-token-allocation-request-v2-1.0.0.dar` | `e350cc92ddb271c476d22ba54c7009ec14829734878b352a2a09937d13b0a99a` |
| `dars/splice-api-token-allocation-v2-1.0.0.dar` | `/Users/x/excanton/CN/splice` `origin/token-standard-v2-daml-preview` commit `b91de5d4b910ded598151981654dce2acc6f84ba`, path `daml/dars/splice-api-token-allocation-v2-1.0.0.dar` | `b023bd40cdf0ed9189e6819f94225fa05ed86b4276b647cb450aa6979b65b801` |
| `dars/splice-api-token-transfer-events-v2-1.0.0.dar` | `/Users/x/excanton/CN/splice` `origin/token-standard-v2-daml-preview` commit `b91de5d4b910ded598151981654dce2acc6f84ba`, path `daml/dars/splice-api-token-transfer-events-v2-1.0.0.dar` | `b865117fc61b1bdb46756dcaa2d2330822e62e7a925c9bec49805e71dd0bddda` |

Packaging handoff: use the staged Git index in `tools/canton-token-template`
as the commit/patch source for this reviewed prototype. Do not reconstruct this
slice from a staged/unstaged split.

### 2026-05-19 Account/Holding compile probe

Implemented the first experimental, non-conformant CIP-112 probe in
`simple-token/` using the real Splice preview V2 DARs from
`/Users/x/excanton/CN/splice` branch `origin/token-standard-v2-daml-preview`
at commit `b91de5d4b910ded598151981654dce2acc6f84ba`:

- Vendored only the V2 preview DARs directly needed by the current
  Account/Holding plus TransferInstruction/TransferFactory prototype:
  `splice-api-token-holding-v2-1.0.0.dar`,
  `splice-api-token-transfer-instruction-v2-1.0.0.dar`, and
  `splice-token-standard-utils-2.0.0.dar`.
- Added those DARs as `data-dependencies` in `simple-token/daml.yaml` and
  `simple-token-test/daml.yaml`.
- `simple-token/daml/SimpleToken/Experimental/Cip112/Account.daml`
  now wraps the real preview `Splice.Api.Token.HoldingV2.Account` and
  delegates `basicAccount` / `isBasicAccount` to
  `Splice.TokenStandard.Utils`.
- `simple-token/daml/SimpleToken/Experimental/Cip112/HoldingProbe.daml`
  defines `AccountHoldingProbe` and maps an existing V1 `HoldingView.owner :
  Party` into the preview V2 `account = basicAccount owner`, while upcasting
  `instrumentId` and `lock` via `Splice.TokenStandard.Utils`.
- `simple-token-test/daml/SimpleToken/Test/Cip112Probe.daml` adds
  `test_basicAccountHoldingProbe`, proving an existing V1 holding can be
  queried and projected into the preview V2 account model.

What this proved:

- The current V1 `owner : Party` holding surface has a straightforward
  compile-time mapping to preview CIP-112 basic accounts for providerless,
  default-account-id holdings.
- The mapping works on the existing SDK 3.4.11 package target when using
  Splice preview V2 DARs built for SDK 3.4.11 / LF 2.1.
- Basic-account observers are owner-only in this prototype, matching the
  CIP-0112 expectation that basic accounts preserve V1 visibility and
  authorization behavior.
- The draft spelling issue is resolved by the preview branch in favor of
  `splice-api-token-allocation-v2-1.0.0.dar`; no
  `splice-api-token-allocation-allocation-v2` DAR exists in the preview
  branch artifact set. Allocation V2 DARs are intentionally not vendored in
  this current transfer/basic-account slice and must only be reintroduced by a
  V2 Allocation slice that imports them directly.

What this did not attempt:

- This first probe only imported and exercised the real V2 data types and
  utility conversions; the follow-on V2 Holding interface prototype below is
  the first real V2 interface instance.
- No `SettlementFactory`, V2 `TransferInstruction`, V2 `Allocation`, or
  multi-leg allocation path was attempted in this first probe slice. Later
  sections below record the follow-on transfer, allocation, settlement, and
  event prototypes.
- The preview branch is still a branch-level source, not an accepted release
  source-of-record. Release/conformance work still needs an accepted source
  decision.

Decision rows informed:

- `CIP-112-D-002`: `origin/token-standard-v2-daml-preview` provides usable
  preview V2 DARs; stable source-of-record is still a release gate.
- `CIP-112-D-003`: the preview DARs compile with this repo's SDK 3.4.11
  target, so this probe did not prove a new SDK pin is needed for the
  Account/Holding slice.
- `CIP-112-D-005`: token-template default remains basic accounts only
  (`owner = Some owner`, `provider = None`, `id = ""`, owner authority).
- `TT-Q-003`: V2 locks were not changed; the probe preserves the existing V1
  `Lock` payload and therefore leaves lock-holder semantics for a later V2
  interface slice.

### 2026-05-19 V2 Holding interface prototype

Implemented the current experimental slice in `simple-token/` using the same
vendored Splice preview V2 DARs and `splice-token-standard-utils`:

- `SimpleHolding` now implements both V1 `Holding` and preview V2
  `Splice.Api.Token.HoldingV2.Holding`.
- `LockedSimpleHolding` now implements both V1 `Holding` and preview V2
  `Splice.Api.Token.HoldingV2.Holding`.
- Both V2 views preserve the existing template fields and project
  `owner : Party` into the preview basic-account shape:
  `owner = Some owner`, `provider = None`, and `id = ""`.
- V2 `instrumentId` and `lock` are upcast with `splice-token-standard-utils`;
  metadata is passed through unchanged.
- `simple-token-test/daml/SimpleToken/Test/Cip112Probe.daml` now queries the
  real V2 Holding interface for unlocked and locked holdings and asserts the
  account projection. Existing V1 tests still run unchanged.

Implementation assumptions recorded by the prototype:

| Surface | Assumption for this prototype |
| :--- | :--- |
| Signatories | `SimpleHolding` remains signed by `admin, owner`; `LockedSimpleHolding` remains signed by `admin, owner, lock.holders`. No V2 provider signatory is introduced because all projected accounts are providerless. |
| Observers | `SimpleHolding` keeps no explicit observers. `LockedSimpleHolding` keeps `extraObservers`. The V2 account projection does not add provider observers because `provider = None`. |
| Controllers and choices | The V2 Holding interface has no choices in the preview package, so controllers are unchanged. `LockedSimpleHolding_Unlock` remains controlled by `owner`. |
| Disclosed parties | No new disclosed parties are required for the V2 Holding view beyond parties already able to see the underlying contract. Basic-account observers are owner-only at the account layer. |
| Privacy | The prototype preserves V1 visibility. It does not implement provider visibility, account delegation visibility, or `SettlementFactory` choice-observer reductions. |
| Authorization | Movement authority remains the existing V1 transfer/allocation authority. The V2 Holding view is a read/projection surface only in this slice. |
| Archival behavior | Archival is unchanged: transfer/allocation flows archive or consume holdings through the existing V1 paths; V2 Holding exposes no archival choice. |
| Failure modes | Positive-amount checks remain on the templates. Invalid provider/id combinations are not representable through this slice because the V2 account is always `basicAccount owner`. |
| Lock handling | `LockedSimpleHolding.lock : V1.Lock` is upcast to `V2.Lock`. Lock holders remain `[Party]`; no Account-keyed lock model is introduced. Expired-lock behavior and `LockedSimpleHolding_Unlock` remain unchanged. |
| Upgrade assumptions | The implementation adds a preview V2 interface view while preserving the existing V1 fields and V1 interface. Future V2 source changes may require re-baselining because the DAR source-of-record is unresolved. |
| Non-conformance status | This is a historical status row for the Holding-only slice. Later sections add basic-account V2 `TransferInstruction`/`TransferFactory`, V2 Allocation, one narrow `SettlementFactory`, and explicit EventLog-context reporting. Provider-authorized accounts, metadata policy advertising, broader transfer-event reporting, and CIP-0112 conformance tests remain unimplemented. |

New blocker evidence:

- `dpm build` for `simple-token` and `dpm build && dpm test` for
  `simple-token-test` pass with SDK 3.4.11 / LF 2.1, so this Holding-only
  slice did not prove a new SDK/Canton pin is needed.
- The source-of-record blocker remains: the implementation depends on
  `origin/token-standard-v2-daml-preview` at
  `b91de5d4b910ded598151981654dce2acc6f84ba`, not a stable release source.
- This Holding-only slice produced no evidence for V2 `TransferInstruction`,
  V2 `Allocation`, `SettlementFactory`, stablecoin provider mapping, Wallet SDK
  authority, or ChainSafe alignment. The follow-on transfer slice below updates
  the `TransferInstruction`/`TransferFactory` evidence only.

### Unblocked follow-on prototypes

The V2 Holding prototype removed the local sequencing reason to wait before
starting the rest of the CIP-112 surface in this tool. After the transfer,
allocation, settlement, and event prototypes below, the following work remains active experimental
implementation scope, provided each change remains non-release, locally
validated, and documented with its authority/privacy assumptions:

1. A V1/V2-compatible allocation path required by CIP-0112 §5.1.
2. Receipt allocation auto-creation and `extraReceiptAuthorizers` once the
   preview/Draft shape stabilizes.
3. Broader V2 transfer-event reporting and metadata fallback sufficient for
   mixed V1/V2 parsing.
4. Targeted compatibility tests from CIP-0112 §5.4 and §5.5, starting with the
   smallest sender/receiver basic-account matrix before provider-managed
   accounts.
5. A privacy-preserving multi-party batch settlement scenario now that the
   single-asset sender/receiver batch path compiles and records its observer
   assumptions.

Use `/Users/x/excanton/CN/splice` `origin/token-standard-v2-daml-preview` at
`b91de5d4b910ded598151981654dce2acc6f84ba` as reference evidence, especially:

- `token-standard/V2_VALIDATION.md`
- `token-standard/examples/splice-test-token-v2/daml/Splice/Testing/Tokens/TestTokenV2.daml`
- `token-standard/examples/splice-test-token-v2/daml/Splice/Testing/Tokens/TestTokenV2/AccountConfig.daml`
- `token-standard/examples/splice-test-token-v2/daml/Splice/Testing/Tokens/TestTokenV2/Transfer.daml`
- `token-standard/examples/splice-test-token-v2/daml/Splice/Testing/Tokens/TestTokenV2/Allocation.daml`
- `token-standard/splice-token-standard-v2-test/daml/Splice/Tests/*.daml`

These Splice paths are blueprints and rough-edge evidence, not accepted OZ
scope. The token template should simplify them aggressively where the local
asset only supports basic accounts.

### 2026-05-19 V2 TransferInstruction and TransferFactory prototype

Implemented the next experimental slice in `simple-token/` against the same
Splice preview V2 DARs:

- `SimpleTransferInstruction` now implements preview
  `Splice.Api.Token.TransferInstructionV2.TransferInstruction` in addition to
  existing V1 `TransferInstruction`.
- `SimpleTokenRules` now implements preview
  `Splice.Api.Token.TransferInstructionV2.TransferFactory` in addition to the
  existing V1 factory.
- The V2 factory accepts only basic accounts:
  `owner = Some party`, `provider = None`, and `id = ""`. It rejects
  non-basic sender or receiver accounts before delegating to the V1 factory
  path.
- The V2 factory requires sender-owner authority through `actors = [sender]`
  and performs a local V2 actor guard with a unique failure marker before using
  the preview `splice-token-standard-utils` V2-to-V1 factory adapter. Existing
  V1 validation still enforces admin, amount, deadline, supported instrument,
  input holding, lock, and sufficiency checks.
- The V2 instruction view upcasts the existing V1 transfer into V2 account
  form and exposes an `availableActions : Map TransferInstructionAction
  [[Party]]` model matching the preview branch shape:
  `TIA_Withdraw -> [[sender]]`, `TIA_Accept -> [[receiver]]`, and
  `TIA_Reject -> [[receiver]]`.
- The V2 accept, reject, and withdraw choices delegate to the
  `splice-token-standard-utils` default implementations backed by the existing
  V1 accept/reject/withdraw choices after local V2 actor guards with unique
  failure markers. There is no separate preview
  `TransferInstruction_Update` choice in the inspected V2 DAR; the local V1
  `TransferInstruction_Update` stub remains unsupported.
- Added `simple-token-test/daml/SimpleToken/Test/Cip112Transfer.daml` with
  focused Daml Script probes for V2 factory initiation and action visibility,
  V2 accept/reject/withdraw execution plus instruction archival, wrong V2
  actors, and non-basic sender/receiver account rejection.

Implementation assumptions recorded by the prototype:

| Surface | Assumption for this prototype |
| :--- | :--- |
| Signatories | `SimpleTransferInstruction` remains signed by `admin, transfer.sender`. `SimpleTokenRules` remains signed by `admin`. No provider signatory is introduced because V2 sender and receiver accounts must be basic. |
| Observers | `SimpleTransferInstruction` keeps `transfer.receiver` as observer. V2 instruction choice observers use the preview utility default, which returns the instruction observers. V2 factory choice observers use the private-asset sender-account default. |
| Controllers and choices | V1 controllers are unchanged. V2 `TransferFactory_Transfer` is controlled by `actors`, locally checked to exactly `[sender]` for basic accounts before utility/V1 delegation. V2 accept/reject are controlled by `actors`, locally checked to exactly `[receiver]`; V2 withdraw is locally checked to exactly `[sender]`. |
| Disclosed parties | V2 factory initiation still requires disclosure of `SimpleTokenRules` to the sender in the local test harness. Pending transfer acceptance uses the instruction already visible to the receiver. No provider or account-config disclosures exist in this slice. |
| Privacy | The prototype preserves the V1 two-step privacy model. The sender initiates against the factory; the receiver observes the pending instruction and locked holding; non-participants do not receive new visibility. It does not implement provider visibility or reduced-privacy/public-asset observer modes. |
| Authorization | Authority is owner-only for basic accounts. The factory path rejects non-basic accounts and does not implement provider-call-on-behalf, joint authorization, or account-config delegation. |
| Archival behavior | V2 accept/reject/withdraw delegate to the consuming V1 choices, so the instruction is archived on successful completion or failure. `test_v2TransferInstructionAcceptArchivesAndCompletes` verifies the accepted instruction is no longer queryable through the V2 interface. |
| Failure modes | Non-basic accounts fail with `ensureBasicAccount`. Wrong V2 actors fail at local V2 actor guards before the preview utility adapter can delegate into V1; the Daml Script probes assert the guard-specific failure markers. Existing V1 failure modes are preserved for wrong admin, future `requestedAt`, expired `executeBefore`, non-positive amount, unsupported instrument, empty inputs, insufficient funds, locked inputs, and V1 unauthorized accept/reject/withdraw actors. |
| Upgrade assumptions | The implementation relies on the preview branch's current `availableActions : Map TransferInstructionAction [[Party]]` shape and utility adapters. Future preview or CIP changes may require re-baselining. |
| Non-conformance status | This remains experimental and non-conformant. It does not implement provider-managed accounts, V2 Allocation, SettlementFactory, transfer-event emission, stablecoin account semantics, Wallet SDK authority, ChainSafe pinning, or a CIP-0112 conformance matrix. |

New blocker evidence:

- `dpm build` for `simple-token` and `dpm build && dpm test` for
  `simple-token-test` pass with SDK 3.4.11 / LF 2.1, so this transfer slice
  did not prove a new SDK/Canton pin is needed for basic-account
  TransferInstruction/TransferFactory support.
- The direct preview DAR footprint is narrowed to V2 Holding, V2
  TransferInstruction, and `splice-token-standard-utils`. V2 Allocation,
  AllocationInstruction, AllocationRequest, and TransferEvents DARs are not
  direct data-dependencies and are not vendored as separate files for this
  slice. The Utils DAR remains justified because this prototype uses its
  `basicAccount`, `ensureBasicAccount`, V1/V2 upcasts, and V2-to-V1 transfer
  adapters; hardening should replace it with an accepted release package or a
  narrower utility source if the upstream package remains preview-only.
- The inspected draft text still describes a `TransferInstruction_Update`
  replacement shape, while the preview DAR exposes V2 accept/reject/withdraw
  choices with `actors`. The local implementation follows the compileable
  preview DAR and documents the mismatch as source-of-record churn.
- Transfer-event reporting remains out of scope for this slice even though the
  preview branch now expects event-based reporting for release-quality
  transaction parsing.

### 2026-05-19 V2 AllocationInstruction and Allocation prototype

Implemented the next experimental slice in `simple-token/` against the same
Splice preview V2 source, adding the two direct DARs this slice imports:

- `splice-api-token-allocation-v2-1.0.0.dar`
- `splice-api-token-allocation-instruction-v2-1.0.0.dar`

Implementation summary:

- `SimpleTokenRules` now implements preview
  `Splice.Api.Token.AllocationInstructionV2.AllocationFactory`.
- `SimpleAllocationInstructionV2` implements preview
  `Splice.Api.Token.AllocationInstructionV2.AllocationInstruction`.
- `SimpleAllocationV2` implements preview
  `Splice.Api.Token.AllocationV2.Allocation`.
- The factory accepts only providerless default/basic accounts for the
  allocation authorizer and all transfer-leg other-side accounts. It validates
  the preview V2 allocation shape with `splice-token-standard-utils`, checks
  supported instrument ids, and rejects wrong `actors` before creating a
  pending instruction.
- The instruction `Accept` choice locks the net debit amount needed for the
  authorizer's transfer-leg sides, returns change, and creates a V2
  allocation. The instruction `Withdraw` choice archives the instruction
  without locking funds and returns the unused input holding CIDs through
  `authorizerChangeCids`. The factory captures that CID recovery map when the
  pending instruction is created, so withdraw can clear the instruction even if
  one of the referenced input CIDs was consumed before withdraw.
- The allocation `Withdraw` and `Cancel` choices unlock and return the locked
  holdings to the authorizer. `Withdraw` uses the upstream
  `ensureWithdrawIsAllowed` helper for committed allocation deadline behavior.
- At the end of this allocation-only slice, `Allocation_Settle` was explicitly
  unsupported because `SettlementFactory_SettleBatch` was outside that boundary.
  The following settlement/event section supersedes this by adding one narrow
  basic-account settlement path and advertising `AA_Settle`.
- Added `simple-token-test/daml/SimpleToken/Test/Cip112Allocation.daml` with
  positive instruction-to-allocation flow coverage plus wrong actor,
  non-basic account, malformed allocation, instruction withdraw, allocation
  cancel, allocation withdraw, and committed-withdraw-after-deadline probes.

Implementation assumptions recorded by the prototype:

| Surface | Assumption for this prototype |
| :--- | :--- |
| Signatories | `SimpleAllocationInstructionV2` and `SimpleAllocationV2` are signed by `admin` and the basic authorizer owner. No provider signatory is introduced because non-basic accounts are rejected. |
| Observers | Both V2 allocation templates observe `allocation.settlement.executors`. Locked holdings created during instruction accept observe the executors. No counterparty, provider, or reduced-privacy public-asset observers are added. |
| Controllers and choices | V2 `AllocationFactory_Allocate`, `AllocationInstruction_Accept`, `AllocationInstruction_Withdraw`, and `Allocation_Withdraw` require `actors == [authorizer owner]`. V2 `Allocation_Cancel` requires the same party set as `settlement.executors` and is intentionally order-insensitive for multi-executor lists. In this allocation-only slice, `Allocation_Settle` failed with an explicit SettlementFactory boundary marker; the following settlement/event section replaces that with the narrow basic-account settle path. |
| Disclosed parties | The factory contract is disclosed to the authorizer in the local harness, matching the earlier V2 transfer tests. Wrong-actor negative tests use explicit disclosures so failures prove local actor guards rather than visibility failure. |
| Privacy | The prototype preserves a private-asset posture: authorizer, admin, and executors see the instruction/allocation state. Other trading accounts do not receive visibility in this slice. Batch-settlement privacy remained untested until the following SettlementFactory work. |
| Authorization | Authority is owner-only for basic accounts. Provider-call-on-behalf, joint authorization, account-config delegation, receipt authorizers, and wallet SDK authority are not implemented. |
| Archival behavior | Instruction `Accept` and `Withdraw` archive the instruction. Allocation `Withdraw` and `Cancel` archive the allocation and locked holdings before returning unlocked holdings to the authorizer. In this allocation-only slice, `Allocation_Settle` did not archive because it was unsupported; the following section records the current narrow settle/archive behavior. |
| Failure modes | Wrong V2 actors fail at local guard markers. Non-basic accounts, empty transfer-leg sides, unsupported instruments, future `requestedAt`, insufficient input funding, unexpired locked inputs, expired instructions, and premature committed-allocation withdrawal fail before funds are released or reallocated. Pending-instruction withdraw does not fetch input holdings, leaves live inputs active, and returns the originally captured CIDs to the authorizer result map even if an input CID has since become stale. |
| Lock handling | Instruction `Accept` creates `LockedSimpleHolding` contracts for net debit amounts only, with `holders = [admin]`, `expiresAt = allocation.settlement.settlementDeadline`, and executor observers. Existing party-keyed V1 lock semantics are reused. |
| Upgrade assumptions | The implementation relies on the preview branch's current `AllocationSpecification.transferLegSides`, `availableActions : Map AllocationAction [[Party]]`, and `AllocationInstructionResult` shapes. Source-of-record or spec changes may require re-baselining. |
| Non-conformance status | This remains experimental and non-conformant. After the following settlement/event slice, the remaining gaps are V1/V2-compatible Allocation implementations, receipt allocations, iterated settlement, provider-managed accounts, broader V2 transfer events, stablecoin account semantics, Wallet SDK authority, ChainSafe pinning, and a CIP-0112 conformance matrix. |

New blocker evidence:

- `dpm build` for `simple-token`, `dpm build` for `simple-token-test`, and
  `dpm test` for `simple-token-test` pass with SDK 3.4.11 / LF 2.1 after
  adding the preview V2 allocation DARs, so this slice did not prove a new
  SDK/Canton pin is needed for basic-account AllocationInstruction/Allocation
  support.
- The compileable preview interface differs from the Draft text inspected in
  `/Users/x/excanton/CF/cips/cip-0112/cip-0112.md`: the preview package uses
  `transferLegSides`, `FinalizedAllocation`, and no
  `extraReceiptAuthorizers` field on `SettlementFactory_SettleBatch`.
- At the end of the allocation slice, settlement was the next hard boundary.
  The following settlement/event prototype updates that evidence with a narrow
  `SettlementFactory_SettleBatch` path while preserving the non-conformance
  boundary.

### 2026-05-19 V2 SettlementFactory and TransferEvents prototype

Implemented the current experimental slice in `simple-token/` against the same
Splice preview V2 source, adding the direct transfer-events DAR this slice
imports:

- `splice-api-token-transfer-events-v2-1.0.0.dar`

Implementation summary:

- `SimpleTokenRules` now implements preview
  `Splice.Api.Token.AllocationV2.SettlementFactory` and preview
  `Splice.Api.Token.TransferEventsV2.EventLog`.
- `SettlementFactory_SettleBatch` is implemented on the existing rules
  contract, not a new settlement-rules template. It keeps the private-asset
  observer default (`[]`), requires `actors` to equal the settlement
  executors before delegating to the preview utility validator, and rejects
  non-basic transfer-leg accounts before allocation fetch/settle work.
- `SimpleAllocationV2.Allocation_Settle` now consumes the allocation, archives
  locked sender-side holdings, checks locked debit amounts against the net
  debit implied by `transferLegSides`, creates receiver-side holdings for net
  credits, rejects iterated-settlement arguments explicitly, and reports V2
  allocation-settlement transfer events through the rules `EventLog` context
  supplied by the settlement factory.
- `SimpleTransferInstruction.TransferInstruction_Accept` can report a V2
  transfer event when the caller supplies an explicitly disclosed EventLog
  context at accept time. Event logging is not stored from factory creation or
  auto-attached by the V2 factory because the receiver normally cannot see the
  rules contract under the private-asset observer model. For two-step transfer
  accept, the sender-side event reports the locked holding archived during
  accept, not the original input holdings archived when the pending instruction
  was created.
- Added focused Daml Script coverage for successful sender+receiver allocation
  batch settlement, wrong settlement actor rejection, non-basic transfer-leg
  account rejection, missing receiver authorization rollback, admin/executor
  overlap, explicit iterated-settlement rejection, explicit transfer accept
  event-log context, and factory-context non-persistence.

Implementation assumptions recorded by the prototype:

| Surface | Assumption for this prototype |
| :--- | :--- |
| Signatories | `SimpleTokenRules` remains signed by `admin`. `SimpleAllocationV2` remains signed by `admin` and the basic authorizer owner. Settlement creates or archives holdings signed by `admin` and the affected basic account owner; no provider signatory is introduced. |
| Observers | `SettlementFactory_SettleBatch` uses the private-asset observer default of no extra observers. `SimpleAllocationV2` and locked allocation holdings still observe `settlement.executors`. EventLog choices use account-party observers for affected basic accounts. |
| Controllers and choices | `SettlementFactory_SettleBatch` is controlled by `actors`, locally checked as a party set equal to `settlement.executors` before utility delegation. `Allocation_Settle` is controlled by `actors`, locally checked as a party set equal to `admin :: settlement.executors` before mutating locked funds, so duplicate admin/executor overlap does not fail spuriously. Transfer accept event logging remains receiver-controlled and only logs when an EventLog is explicitly supplied through accept context. |
| Disclosed parties | Settlement tests disclose the rules contract to the executor before exercising `SettlementFactory_SettleBatch`. Transfer event reporting requires the EventLog/rules contract to be visible or disclosed to the receiver at accept time; factory-time EventLog context is not persisted on the pending instruction. Without accept-time context, V2 accept falls back to the existing V1-compatible transfer behavior and does not emit a V2 event. |
| Privacy | The settlement factory follows the private-asset baseline: batch settlement is visible to admin and executors, while per-account holding changes are visible to the affected account parties through EventLog choice observers. The prototype does not opt into the reduced-privacy/public-asset observer mode. |
| Authorization | Basic-account owner authority remains the only account authority. The settlement factory validates executor actors before fetching allocations. Allocation settlement validates admin+executor actors before archiving locked holdings or creating credited holdings. Provider-call-on-behalf, joint-account authority, extra receipt authorizers, and Wallet SDK authority are not implemented. |
| Archival behavior | Successful batch settlement archives each settled `SimpleAllocationV2` and its locked holdings, creates credited receiver holdings, and returns `AllocationResult_Settled`. Failed settlement actor, non-basic account, iterated-settlement argument, or missing-authorization cases roll back before allocation or locked-holding archival. Transfer accept event reporting uses the locked holding archived in the accept transaction as the sender event input. |
| Failure modes | Wrong settlement actors fail at a local guard marker. Non-basic transfer-leg accounts fail before allocation fetch/settle delegation. Missing sender/receiver allocation authorizations fail in the preview utility validator and leave locked funds active. Unsupported `extraTransferLegSides` or `nextIterationFunding` fail explicitly inside `Allocation_Settle`. Locked-debit mismatches fail inside `Allocation_Settle` before credited holdings are created. |
| Stale state | Settlement fetches allocation and locked holding CIDs at execution time; consumed allocation CIDs, stale locked holdings, or mismatched allocation views fail the transaction and roll back atomically. The prior pending-instruction stale-input recovery behavior is unchanged. |
| Lock handling | Sender-side allocation accept still creates `LockedSimpleHolding` contracts with `holders = [admin]`, `expiresAt = settlementDeadline`, and executor observers. Settlement archives those locked holdings. Failed settlement keeps them locked for later valid settlement, cancel, withdraw, or expiry-path recovery. |
| Upgrade assumptions | The implementation relies on the preview branch's current `SettlementFactory_SettleBatch.allocations : [FinalizedAllocation]` shape and `TransferEventsV2.EventLog_HoldingsChange` surface. Draft/spec drift remains expected, especially around `extraReceiptAuthorizers` and package naming. |
| Non-conformance status | This remains experimental and non-conformant. It does not implement V1/V2-compatible allocation pairing, receipt allocation auto-creation, iterated settlement, extra receipt authorizers, reduced-privacy observer mode, provider-managed accounts, transfer events for direct/self transfers or reject/withdraw holding changes, stablecoin account semantics, Wallet SDK authority, ChainSafe pinning, or a CIP-0112 conformance matrix. |

New blocker evidence:

- `dpm build` for `simple-token`, `dpm build` for `simple-token-test`, and
  `dpm test` for `simple-token-test` pass with SDK 3.4.11 / LF 2.1 after
  adding the preview transfer-events DAR and the narrow settlement/event
  surface.
- The preview interface is sufficient for a basic-account
  `SettlementFactory_SettleBatch` compile/test path. It is not sufficient to
  close release questions because the compileable preview omits the draft
  `extraReceiptAuthorizers` field and still comes from the volatile
  `origin/token-standard-v2-daml-preview` branch.
- Transfer-event reporting exposed a real privacy/disclosure boundary:
  receiver-side V2 accept cannot exercise the rules `EventLog` unless that
  contract is explicitly disclosed. The prototype therefore makes transfer
  event logging accept-context-driven instead of silently widening rules
  visibility or carrying factory-supplied EventLog context onto pending
  instructions.
- This settlement/event slice originally left iterated settlement as an
  explicit blocker. The 2026-05-20 iterated-settlement slice below supersedes
  that status for the simple-token allocation path while preserving this row
  as historical evidence for the first settlement prototype boundary.
- The transfer-event regression tests use test-only `EventLog` templates inside
  the Daml Script test package. That is acceptable while the package remains a
  local non-uploadable test harness; if the package becomes uploadable, split
  those templates into a separate non-script test package to avoid uploading
  `daml-script` with ledger templates.

### 2026-05-19 CIP-0112 compatibility matrix and negative tests

Implemented the focused compatibility-matrix slice in
`simple-token-test/daml/SimpleToken/Test/Cip112CompatibilityMatrix.daml`.
This is test-only discovery coverage over the staged settlement/event
prototype; it does not add public API surface and does not change the
non-release, non-conformant status of the V2 code.

Coverage added:

- Transfer §5.4 rows:
  - V1 sender/factory on a V2 asset creates a dual-version pending instruction
    that a V2 receiver can inspect and accept with basic-account fallback.
  - V2 sender/factory on a V2 asset creates a pending instruction that a V1
    receiver can accept through the V1 choice.
  - Duplicate actor lists are rejected before V1 delegation, preserving the
    current exact singleton actor rule for basic accounts.
  - Provider-managed receiver accounts now enter the provider-managed
    authority path and advertise receiver-provider accept/reject actions.
  - The private rules/factory contract must be disclosed before ordinary
    holders can exercise V2 factory choices.
  - Explicit transfer EventLog context keeps non-participants out of test
    event visibility.
- Allocation §5.5 and §4.3 rows:
  - V2 settlement of a V1 allocation artifact is rejected without consuming
    the V1 locked funds.
  - Multi-leg V2 allocation acceptance locks only the authorizer's net debit
    amount and returns change, proving the current net-funding helper across
    more than one leg.
  - Pending allocation instructions are visible to the authorizer and
    executors but not the transfer-leg counterparty under the private-asset
    observer model.
  - Unexpired locked holdings cannot be reused as allocation funding; failed
    acceptance rolls back and leaves the pending instruction plus locked
    transfer holding live.
  - Stale receiver allocation CIDs fail batch settlement without consuming the
    sender allocation or its locked funds.
  - Settlement after `settlementDeadline` fails before allocation or
    locked-holding archival.

Implementation assumptions reinforced:

| Surface | Matrix-slice result |
| :--- | :--- |
| Signatories | No new templates were added outside the test harness. Existing V2 prototype signatories remain unchanged. |
| Observers | The compatibility tests assert private-asset observer expectations for allocation instructions, transfer EventLog entries, and the provider-managed receiver instruction row. They do not measure Canton view counts. |
| Controllers and choices | Basic-account V2 actors remain exact singleton lists for transfer/allocation owner choices and executor-set lists for settlement/cancel. Provider-managed transfer rows use singleton provider actors. Duplicate actor transfer initiation is rejected. |
| Disclosed parties | V2 factory calls require explicit rules disclosures in the local harness; the matrix asserts the missing-disclosure `NotVisible` failure marker instead of only rollback. Event logging requires explicit accept-time EventLog context. |
| Privacy | Non-participants do not observe the test EventLog entries or pending allocation instructions in the asserted private-asset cases. The settlement factory still uses the private-asset observer default, not reduced-privacy/public-asset observers. |
| Authorization | Basic-account owner authority, provider-managed singleton-provider authority, and settlement executor authority are the implemented V2 authorization models. |
| Archival behavior | Failed V2 settlement rows for V1 allocation artifacts, stale allocation CIDs, expired deadlines, and missing valid funding roll back before consuming live sender locks. |
| Failure modes | Negative tests now cover duplicate actors, missing disclosures, unsupported provider account shapes, V1 artifact settlement through V2, unexpired lock reuse, stale allocation CIDs, and deadline expiry. Disclosure, V1-as-V2, unexpired-lock, stale-CID, and deadline rows assert concrete failure substrings where the failure class matters. |
| Stale state | Stale allocation CIDs fail atomically with a `NotActive` ledger marker; pending-allocation stale input recovery from the earlier slice remains unchanged. |
| Lock handling | Multi-leg allocations lock net debit only. Unexpired transfer locks cannot be reused as allocation funding and assert the local unexpired-lock marker. Expired-lock recovery remains covered by the existing V1 negative tests and the prior V2 committed-withdraw deadline probe. |
| Upgrade assumptions | The tests follow the preview branch's current `Action -> [[Party]]`, `TransferLegSide`, `FinalizedAllocation`, and EventLog shapes. Spec/DAR churn may require re-baselining. |
| Non-conformance status | Still experimental and non-conformant. The matrix tests prove selected local behavior only; they are not CIP-0112 conformance tests. |

Remaining matrix gaps after this slice:

- Full §5.4 transfer matrix coverage for direct/self transfers, reject and
  withdraw EventLog reporting and internal-workflow authorization beyond this
  provider-managed authority slice.
- Full §5.5 allocation matrix coverage for V1/V2-compatible Allocation
  implementations, broader V2 allocation-request app/wallet variants,
  receipt-allocation auto-creation, V1-wallet-with-dApp-API flows, V1 asset
  contrast packages, and mixed multi-asset settlement across different admins.
- CIP-0112 §4.1.1 / §4.2 three-trader privacy-preserving batch settlement and
  view-count optimization evidence.
- Iterated-settlement coverage beyond the current simple-token successor
  allocation path, including request-originated iteration funding, stablecoin
  CDP repayment/liquidation use cases, and cross-admin settlement.
- Jointly controlled, special mint/burn, and explicit account-setup flows.
- Settlement EventLog reconstruction with a persistent custom log remains a
  harness gap because the current settlement factory supplies the rules
  EventLog context internally.

### 2026-05-19 Provider-managed authority design-space slice

Implemented the current experimental provider-managed account authority slice
in `simple-token/` and `simple-token-test/`. This remains non-release,
non-conformant discovery code. It does not implement custody-provider business
logic, Wallet SDK authority, production delegation, public API hardening, or a
CIP-0112 conformance claim.

Implementation summary:

- `SimpleToken.Experimental.Cip112.Account` now models two supported regular
  account shapes: `basicAccount owner` and
  `providerManagedAccount owner provider id`. Provider-managed accounts require
  `owner = Some owner`, `provider = Some provider`, and a non-empty `id`.
- `ProviderManagedSimpleHolding` and `ProviderManagedLockedHolding` are V2-only
  holding templates. They are signed by `admin` and the provider; the owner
  observes through `accountObservers`. They intentionally do not implement the
  V1 `Holding` interface.
- `SimpleTokenRules.TransferFactory_Transfer` keeps the basic-account V1
  delegation path when both accounts are basic. If either account is
  provider-managed, it follows a V2-only path that validates the regular
  account shape, requires `actors == [sender account authority]`, archives all
  V2 input holdings for contention, creates a locked V2 sender holding, and
  creates `ProviderManagedTransferInstruction`.
- `ProviderManagedTransferInstruction` exposes provider-driven
  `availableActions`: sender account authority for withdraw and receiver
  account authority for accept/reject. Successful accept archives the locked
  holding, creates receiver holdings in the receiver account shape, and returns
  sender change in the sender account shape.
- `SimpleAllocationInstructionV2` and `SimpleAllocationV2` now use
  `accountAuthority`: owner for basic accounts, provider for provider-managed
  accounts. Allocation locking, withdraw, cancel, settlement, and credited
  output creation now preserve the authorizer account shape instead of forcing
  basic accounts.
- Allocation locking has two explicit V2-only view-policy probes. The default
  policy keeps `Lock.holders = [admin]`; an opt-in choice-context policy
  `simple-token/cip-112-allocation-lock-view-policy = "account-parties"` uses
  `Lock.holders = admin + authorizer account parties`. Both policies keep
  movement authority on allocation choices rather than on a standalone lock
  choice.
- `SettlementFactory_SettleBatch` now accepts supported regular transfer-leg
  accounts, while retaining executor-set control on the factory and
  `admin + executors` control on each allocation settlement.

Authority matrix for this slice:

| Flow | Account principal | Required actors/controllers | `availableActions` | Observers and disclosures |
| :--- | :--- | :--- | :--- | :--- |
| Provider-managed holding | `owner` is the business principal; `provider` is the movement authority; `id` is the account id. | Template signatories are `admin, provider`; owner is not a controller. | No holding choices. | Owner and provider observe holdings; admin is a signatory. No third-party observers are added. |
| V2 transfer factory | `sender.owner` remains the principal; `sender.provider` acts when present. | `actors == [sender.provider]` for provider-managed senders; `actors == [sender.owner]` for basic senders. | Pending instruction advertises withdraw for sender authority and accept/reject for receiver authority. | Sender account parties observe the factory choice through the private-asset default. The pending instruction observes sender and receiver account parties. |
| V2 transfer accept/reject/withdraw | Receiver principal is `receiver.owner`; receiver provider acts for accept/reject. Sender provider acts for withdraw. | Accept/reject require `[receiver.provider]` for provider-managed receivers. Withdraw requires `[sender.provider]` for provider-managed senders. | `TIA_Accept -> [[receiver authority]]`, `TIA_Reject -> [[receiver authority]]`, `TIA_Withdraw -> [[sender authority]]`. | The instruction observers are reused as choice observers. Accept-time EventLog context is still explicit; no factory-time EventLog context is persisted. |
| V2 allocation factory and instruction accept/withdraw | `allocation.authorizer.owner` is the principal; provider acts when present. | Factory, instruction accept, and pending-instruction withdraw require `[authorizer provider]` for provider-managed accounts. | `AIA_Accept -> [[authorizer authority]]`, `AIA_Withdraw -> [[authorizer authority]]`. | Allocation instructions observe authorizer account parties and settlement executors; transfer-leg counterparties do not observe the instruction under the private-asset posture. |
| V2 allocation withdraw/cancel/settle | Authorizer provider can withdraw its own allocation. Executors cancel. Settlement remains admin/executor co-validated. | Withdraw requires `[authorizer provider]`; cancel requires executor set; settle requires `admin + executors` on each allocation. | `AA_Withdraw -> [[authorizer authority]]`, `AA_Cancel -> [executors]`, `AA_Settle -> [[admin, executors...]]`. | Allocations observe authorizer account parties and executors. Locked holdings observe authorizer account parties and executors. The default lock-holder view is admin-only; the opt-in view includes admin and authorizer account parties. |
| V2 settlement factory | Trading-account principals are the owners on each transfer leg; providers are visible but do not control batch execution after allocations are authorized. | `SettlementFactory_SettleBatch` still requires actors equal to settlement executors. | Settlement factory has no `availableActions`; allocation views advertise settle. | Factory choice uses the private-asset observer default. Per-allocation settlement creates account-shaped outputs visible to the credited account parties. |

Allocation lock-view policy probes:

| Policy | Signatories and observers | `Lock.holders` view | `availableActions` and disclosed parties | Stale, contention, and archival behavior |
| :--- | :--- | :--- | :--- | :--- |
| Admin-only allocation lock | Provider-managed locked holdings are signed by `admin, provider`; authorizer owner/provider and executors observe. | `[admin]`. | Allocation instruction/allocation actions are unchanged: authorizer provider accepts/withdraws, executors cancel, `admin + executors` settle. Daml Script still discloses the rules/factory contract before factory calls. | Accept archives input holdings and creates the lock. Withdraw/cancel archive the lock and return account-shaped holdings. Settlement archives allocations and locks. Stale CIDs fail with `NotActive`; unexpired locks remain rejected as later inputs. |
| Account-party allocation lock | Same template signatories and observers as admin-only, with authorizer account parties also repeated in the lock extra-observer policy for the probe. | `admin + accountAuthority + accountObservers allocation.authorizer`, de-duplicated. | `availableActions` and disclosed parties are intentionally identical to admin-only; only the lock-holder view changes. Unsupported policy values fail before input consumption. | The account-party tests cover visibility, withdraw, cancel, settlement, stale allocation reuse, post-archival residual visibility, and the same locked-input contention scenario as the admin-only path. |

Security and lifecycle assumptions:

- Signatories: provider-managed holdings are signed by `admin` and provider,
  not owner. This is the minimum shape that lets the provider move funds
  without importing a custody-provider delegation contract. It is not a
  production custody claim.
- Observers: account owners and providers observe provider-managed holdings,
  locked holdings, transfer instructions, allocation instructions, and
  allocations relevant to their account. Unrelated parties are excluded by
  transfer and allocation tests, including pending allocation instructions,
  accepted allocations, locked allocation holdings, and post-withdraw/cancel/
  settlement residual queries.
- Controllers: provider-managed account movement choices use singleton actor
  lists for the provider. Duplicate actors, owner-only actors, wrong providers,
  and spoofed provider/account combinations fail before successful state
  mutation.
- Disclosures: the local Daml Script harness still discloses the private
  rules/factory contract before V2 factory calls. Provider-spoofing tests
  deliberately disclose the referenced holding so the failure proves account
  validation rather than ordinary visibility failure.
- Privacy: the slice keeps the private-asset default. It does not implement
  reduced-privacy/public-asset settlement observers or view-count
  optimization. Owner/provider visibility is deliberate account visibility,
  not observer expansion to arbitrary third parties.
- Archival and contention: V2 provider-managed transfer and allocation inputs
  archive all supplied input holdings. Unexpired locked provider-managed
  holdings are rejected as later transfer/allocation inputs to preserve the
  contention guarantee.
- Stale state: stale transfer instructions and stale allocation/holding CIDs
  fail with ledger `NotActive` or the existing explicit stale/lock markers and
  roll back without consuming live funds.
- Malicious owner behavior: an owner without provider authority can observe
  provider-managed state but cannot accept, withdraw, allocate, or settle using
  owner-only actors in this prototype.
- Malicious provider behavior: a provider can move provider-managed holdings
  for accounts where it is the recorded provider. Attempts to spoof another
  provider or switch the owner/provider/id tuple fail on input account matching.
  The prototype does not model off-ledger legal authority, account-opening
  consent, provider revocation, or custody obligations.
- Failure modes: missing provider, wrong provider, wrong owner, unauthorized
  actor lists, owner-only instruction actions, unintended observer expansion,
  provider spoofing, stale instructions and allocations, lock-view policy
  choice, and locked-input contention have focused Daml Script coverage with
  exact local failure markers where the runtime exposes one.
- Upgrade assumptions: the implementation relies on the preview branch's
  current `Account`, `availableActions`, `actors`, `FinalizedAllocation`, and
  EventLog shapes. If CIP-0112 changes provider visibility or
  account-authority conventions, these helpers and tests must be re-baselined.
- Stablecoin CDP authority remains deferred. This token-template slice proved
  regular provider-managed account transfer/allocation/settlement authority
  without a CDP-specific vault need. CDP implementation changes would require a
  separate policy decision about whether a vault keeper, admin, or account
  provider controls CDP close/liquidation authority. The sibling stablecoin plan
  records that as a CDP-specific follow-up, not as provider-managed
  token-template scope.

### 2026-05-20 V2 AllocationRequest workflow slice

Implemented the current experimental allocation-request workflow slice in
`simple-token/` and `simple-token-test/` against the same Splice preview V2
source, adding the direct allocation-request DAR this slice imports:

- `splice-api-token-allocation-request-v2-1.0.0.dar`

Implementation summary:

- `SimpleAllocationRequestV2` implements preview
  `Splice.Api.Token.AllocationRequestV2.AllocationRequest` while preserving
  the existing V1 `SimpleAllocationRequest` behavior.
- Requests are created by settlement executors and observed by the authorizer
  account parties. Transfer-leg counterparties are not observers merely
  because they are referenced in requested allocation legs.
- `availableActions` advertises `ARA_Accept` and `ARA_Reject` for the
  authorizer account authority. `ARA_Withdraw` remains controlled by the
  settlement executors and is intentionally not advertised as an authorizer
  action.
- Accept, reject, and withdraw consume the request. Accept rejects expired
  `settleAt` values, iterated `nextIterationFunding`, empty transfer-leg
  shapes, non-positive leg amounts, and unsupported account shapes before
  archival.
- A local `SimpleAllocationRequestV2_AcceptAndCreateAllocations` choice lets
  the Daml Script wallet-style harness consume the request and exercise the
  existing V2 `AllocationFactory_Allocate` in one transaction. This is an
  experimental convenience for replay-safe app/wallet testing, not a Token
  Standard API addition or public API claim. The helper requires the supplied
  factory calls to cover each requested allocation exactly once, so zero,
  partial, duplicate, or extra non-requested calls fail before any allocation
  factory is exercised.
- Added
  `simple-token-test/daml/SimpleToken/Test/Cip112AllocationRequest.daml` with
  seven Daml Script probes covering create/view/disclosure, reject, withdraw,
  expiry, stale replay, unsupported iterated shape, hidden observer expansion,
  zero/partial/duplicate/extra accept-and-create factory-call coverage,
  missing factory disclosure, provider/basic-account mismatches, explicitly
  disclosed wallet-style submission, submitMulti submission, and
  request-to-instruction-to-allocation-to-settlement for app-proposed multi-leg
  requests.

Authority and lifecycle assumptions:

| Surface | Allocation-request slice result |
| :--- | :--- |
| Signatories | `SimpleAllocationRequestV2` is signed by `settlement.executors`, matching the app/executor request origin. No admin, provider, or wallet signatory is added by request creation. |
| Observers | The authorizer account parties observe the request: owner for basic accounts; owner and provider for provider-managed accounts. Executors see the request as signatories. Transfer-leg counterparties do not observe the request unless they are also authorizer account parties. |
| Controllers and choices | V2 accept/reject require `actors == [accountAuthority authorizer]`. V2 withdraw requires the same party set as `settlement.executors`. The local wallet-style accept-and-create choice is controlled by the same authorizer account authority and delegates actual allocation creation to the existing V2 allocation factory. |
| Disclosed parties | The private rules/allocation-factory contract must still be disclosed to the authorizer when a request acceptance also creates allocation instructions, unless the flow uses `submitMulti` with the rules admin as an authorizing party. A request alone does not disclose the factory or widen rules visibility. Negative tests assert missing rules disclosure separately from request visibility. |
| Privacy | Request visibility is limited to executors and authorizer account parties. Provider-managed requests are visible to both owner and provider for that account, but unrelated basic/provider accounts cannot query the request. Hidden observer expansion to transfer-leg counterparties is covered by tests. |
| Authorization | Basic accounts accept/reject through the owner. Provider-managed accounts accept/reject through the provider. Owner-only attempts against provider-managed requests fail at the local request actor guard before factory mutation. Provider/basic mismatches still fail in the allocation factory when the supplied funding account does not match the requested authorizer. |
| Archival behavior | Accept, reject, withdraw, and the local accept-and-create choice consume the request. Expired, unsupported, wrong-actor, stale-CID, missing-disclosure, and zero/partial/duplicate/extra factory-call failures leave live state unchanged or fail before any successful allocation creation. |
| Stale state | Replaying a consumed request CID fails with the ledger `NotActive` marker. Reusing stale allocation CIDs in the downstream settlement path remains covered by the existing V2 settlement rollback tests and the new request-driven end-to-end replay check. |
| Failure modes | Exact tests cover wrong actors, missing request visibility, missing rules/factory disclosure, expired `settleAt`, unsupported iterated preview shape, stale request CID replay, stale downstream settlement replay, provider-owner mismatch, provider/basic funding mismatch, and hidden observer expansion. |
| Unsupported paths | Iterated requests with `nextIterationFunding`, empty allocation shapes, non-regular account shapes, V1/V2-compatible allocation-pairing requests, receipt allocation auto-creation, reduced-privacy observer mode, Wallet SDK authority, and ChainSafe claims remain unsupported. |
| Upgrade assumptions | The implementation follows the preview branch's current `RequestedAllocation.transferLegSides`, `availableActions : Map AllocationRequestAction [[Party]]`, and `AllocationRequest_Accept/Reject/Withdraw` shapes. Any source-of-record change around request fields, iteration funding, or wallet/client authority requires re-baselining. |

New blocker evidence:

- `dpm build` for `simple-token`, `dpm build` for `simple-token-test`, and
  `dpm test` for `simple-token-test` pass with SDK 3.4.11 / LF 2.1 after
  adding the preview allocation-request DAR.
- The preview package is sufficient for request create/view/reject/withdraw
  and request-driven allocation creation in the local Daml Script harness via
  both explicit disclosure and `submitMulti`. It is not sufficient to claim
  CIP-0112 conformance because iterated request funding, V1/V2-compatible
  request flows, receipt allocation behavior, and wallet authority conventions
  remain unresolved.
- The plan asks for `extraSettlementAuthorizers` evidence for wallet/dApp UX.
  The inspected preview request and settlement packages expose
  `availableActions` and choice-observer defaults, but no
  `extraSettlementAuthorizers` field or choice argument on the request-driven
  path used here. Treat this as preview/Draft drift to re-baseline with the CIP
  authors before hardening wallet UX.
- `tools/canton-stablecoin` was not touched. No concrete CDP repayment or
  liquidation allocation-request flow was needed to prove this token-template
  request slice, so CDP-specific authority remains deferred to a separate
  stablecoin design decision.

### 2026-05-20 V2 iterated settlement and partial-progress slice

Implemented the current experimental iterated-settlement slice in
`simple-token/` and `simple-token-test/` against the same Splice preview V2
source. External references were refreshed before use:

- `/Users/x/excanton/CF/cips` was fetched and fast-forwarded to `main` commit
  `67986a1ff820521ffd0dea92e32d8a49da340756`; `cip-0112/cip-0112.md` had no
  diff from the accepted frozen snapshot
  `a07d8db7a1b86d6fd1109375263b8c56d82e90a0`.
- `/Users/x/excanton/CN/splice` was fetched and fast-forwarded on `main` to
  `5dc3a8aeab28eae09e421332392f1fd11d06f232`; the accepted preview evidence
  ref remains `origin/token-standard-v2-daml-preview` at
  `b91de5d4b910ded598151981654dce2acc6f84ba`.
- Inspected evidence paths:
  `token-standard/splice-api-token-allocation-v2/daml/Splice/Api/Token/AllocationV2.daml`,
  `token-standard/splice-token-standard-utils/daml/Splice/TokenStandard/Utils/Internal/Allocations.daml`,
  `token-standard/splice-token-standard-v2-test/daml/Splice/Tests/TestIteratedSettlement.daml`,
  `daml/splice-amulet/daml/Splice/AmuletAllocationV2.daml`, and
  `token-standard/examples/splice-test-token-v2/daml/Splice/Testing/Tokens/TestTokenV2/Allocation.daml`.

Implementation summary:

- `SimpleAllocationInstructionV2` and `SimpleAllocationV2` now carry
  `supportedInstruments` so next-iteration funding keys are validated against
  the local rules contract instead of silently minting unsupported instrument
  ids.
- `SimpleAllocationV2` now stores `numIterations`. The initial accepted
  allocation starts at `0`; each settlement that supplies
  `nextIterationFunding = Some ...` creates a successor allocation with
  `numIterations + 1`, `originalAllocationCid` pointing at the first
  allocation, empty base `transferLegSides`, and locked funding for the next
  iteration.
- Allocation accept reserves `nextIterationFunding` up front. The locked amount
  is `max 0 (nextIterationFunding - currentNetCredit)` per instrument; excess
  input is returned as authorizer change. Non-iterated allocations keep the
  previous net-debit lock behavior.
- `Allocation_Settle` now merges the original transfer-leg sides with
  finalized `extraTransferLegSides`, consumes the current allocation and locked
  holdings, pays out any amount above the requested next-iteration reserve, and
  creates the next locked allocation when `nextIterationFunding` is `Some`.
  `nextIterationFunding = None` finalizes the allocation and returns the
  remaining locked funding as unlocked authorizer holdings.
- `SettlementFactory_SettleBatch` now permits an empty `transferLegs` list when
  the batch still supplies allocations, which enables refresh/finalization of
  an iterated allocation with no new transfer legs. A fully empty no-op batch
  remains rejected.
- Added `simple-token-test/daml/SimpleToken/Test/Cip112IteratedSettlement.daml`
  and updated the prior allocation test that expected explicit iteration
  rejection. The focused coverage now exercises partial settlement, repeated
  settlement, finalization, cancellation after iteration, committed withdrawal
  recovery after the settlement deadline, receiver-side top-up funding,
  uncommitted successor withdrawal before deadline, invalid funding rejection,
  direct `Allocation_Settle` versus `SettlementFactory_SettleBatch` validation
  boundaries, stale previous-allocation replay, deadline rollback,
  no-double-settle/conservation checks, and the current receipt-allocation
  auto-creation blocker.
- `AllocationFactory_Allocate` now rejects invalid
  `allocation.nextIterationFunding` with the local funding marker before a
  pending instruction is created. `Allocation_Settle` now validates only its
  local settlement-time `extraTransferLegSides` arguments for positive amount,
  supported instrument id, and supported regular account shape before mutating
  the allocation. Full batch transfer-leg shape, duplicate leg IDs, and
  required sender/receiver authorization matching remain factory-owned through
  `SettlementFactory_SettleBatch` and the preview utility validator.
- Added two `daml-props` V2 accounting rows in
  `simple-token-test/daml/SimpleToken/Test/Props`: random iterated settlement
  sequences now model active allocation ids, consumed allocation ids,
  successor allocation ids, partial settlement, finalization, cancel, withdraw,
  and stale/repeated settlement attempts with an explicit target allocation id
  while checking reserve conservation and no-double-settle invariants.
  Deterministic model regressions assert that a replay targeting consumed
  allocation id `1` records a rejected repeat, while a replay targeting the
  live successor allocation id `2` remains an invalid no-op without changing
  accounting fields. This is a lightweight model of the local simple-token
  accounting, not a full V2 account/wallet conformance generator.

Authority, privacy, and lifecycle assumptions:

| Surface | Iterated-settlement slice result |
| :--- | :--- |
| Signatories | `SimpleAllocationV2` remains signed by `admin` and the account authority: owner for basic accounts, provider for provider-managed accounts. Successor allocations preserve the same authorizer, account shape, and signatory model. |
| Observers | Allocation and locked-holding observers remain authorizer account parties plus settlement executors according to the selected allocation lock-view policy. No reduced-privacy/public-asset observer mode is introduced. |
| Controllers and choices | `SettlementFactory_SettleBatch` remains executor-controlled. Each `Allocation_Settle` remains controlled by `admin + executors`. `Allocation_Cancel` remains executor-controlled and `Allocation_Withdraw` remains authorizer-authority controlled. |
| Disclosed parties | The private rules/factory contract still needs explicit disclosure or admin authorization in the Daml Script harness. Iteration does not add a new disclosure carrier; successor allocations are visible only to their signatories/observers. |
| Privacy | Extra transfer-leg sides are visible through the allocation settlement transaction to the same admin/executor/authorizer audiences as existing V2 settlement. The slice does not optimize Canton view counts or implement reduced-privacy observer modes. |
| Authorization | Iterated settlement is only available when the authorizer enabled it by setting `allocation.nextIterationFunding = Some ...` at allocation creation. Non-iterated allocations still reject `extraTransferLegSides` and settlement-time `nextIterationFunding`. |
| Archival behavior | Each settlement consumes the current allocation and its locked holdings. A settlement with `nextIterationFunding = Some ...` creates a successor allocation; a settlement with `nextIterationFunding = None` finalizes and creates no successor. Cancellation/withdrawal of a successor allocation archives it and returns remaining locked funding. |
| Failure modes | Unsupported funding instruments, non-positive funding entries, invalid direct extra transfer-leg-side arguments, unauthorized actors, settlement after deadline, stale allocation CIDs, missing receipt allocations, insufficient reserved funding, and non-iterated settlement arguments fail before successful state mutation. Full batch transfer-leg validation remains factory-owned. |
| Stale state | Reusing a consumed allocation CID fails with the ledger `NotActive` marker and leaves the successor allocation and locked funding active. This is the local stale-state proxy; cross-DAR rebuild/migration stale-state probes remain a future upgrade-test target. |
| Conservation | The repeated-settlement script checks a 100-token prefund settling 10 and 15 token partial bills, then finalizing with 75 returned to the authorizer and 25 paid to the receiver. The `daml-props` model adds random V2 iterated reserve conservation plus consumed-allocation replay/no-double-settle rows. These are Script/property regressions, not completed `daml-verify` proofs. |
| Upgrade assumptions | The implementation relies on the preview branch's current `FinalizedAllocation.extraTransferLegSides`, `FinalizedAllocation.nextIterationFunding`, `AllocationView.originalAllocationCid`, and `AllocationView.numIterations` shapes. Source-of-record changes require re-baselining. |
| Non-conformance status | This remains experimental and non-conformant. It does not implement V1/V2-compatible allocation pairing, automatic receipt-allocation creation, `extraReceiptAuthorizers`, reduced-privacy observer mode, cross-admin coordination, production reporting, Wallet SDK authority, ChainSafe alignment, production relayers, public APIs, or CIP-112 conformance. |

New blocker evidence:

- The accepted preview `SettlementFactory_SettleBatch` shape has
  `allocations : [FinalizedAllocation]` but no `extraReceiptAuthorizers` field.
  The local blocker test therefore proves that a batch missing the receiver
  allocation still fails with `missing authorizations`; auto-created receipt
  allocations must be re-baselined with CIP authors or a later preview shape
  before this tool can implement them.
- The Splice `TestTokenV2` example still marks iterated settlement as TODO,
  while the Amulet preview implementation supports the successor-allocation
  pattern. This token-template slice follows the Amulet-shaped funding and
  successor-allocation model, simplified to the local no-fee simple-token
  accounting model.
- `tools/canton-stablecoin` was not touched. Partial repayment, partial
  liquidation, bad-debt, change-output, and vault archival timing implications
  remain a stablecoin-specific design slice because this token-template work
  only proves regular account settlement semantics.

## Module-by-module V2 mapping

### Holding.daml

CIP-0112 §4.3.2 changes `HoldingView.owner : Party` → `HoldingView.account :
Account`.

Prototype decision:

- `SimpleHolding` and `LockedSimpleHolding` do not gain a new `account` field
  in this slice. V1 `owner : Party` stays canonical on the templates, and the
  V2 view projects it as `basicAccount owner`.
- Both templates implement `Api.Token.HoldingV1.Holding` and preview
  `Api.Token.HoldingV2.Holding`; the V1 implementation uses the existing
  fields, and the V2 implementation upcasts V1-compatible fields via
  `splice-token-standard-utils`.
- `ProviderManagedSimpleHolding` and `ProviderManagedLockedHolding` add a
  V2-only holding surface for regular provider-managed accounts. They preserve
  the exact `Account` record in the V2 view, are signed by `admin` and
  provider, and observe owner/provider account parties.
- The `ensure amount > 0.0` invariant (P0 item #2 in
  `docs/SCOPE.md` §9) is unchanged.
- `LockedSimpleHolding` admin-only lock holder semantics (item Q3 in
  `docs/SCOPE.md` §7) are preserved for this slice. Preview V2 keeps
  `Lock.holders : [Party]`, so no Account-keyed lock model was introduced.

### TransferInstruction.daml

CIP-0112 §4.3.2 replaces `TransferInstruction_Accept` controller from
`receiver` to a configurable `actors : [Party]` field. The `status :
TransferInstructionStatus` view field is replaced by `availableActions :
Map [Party] [TransferInstructionAction]`.

Implementation status:

- Implemented the preview V2 `TransferInstruction` interface on
  `SimpleTransferInstruction` for basic accounts and
  `ProviderManagedTransferInstruction` for the V2-only provider-managed path.
- The inspected preview DAR uses `availableActions :
  Map TransferInstructionAction [[Party]]`, not the older draft
  `Map [Party] [TransferInstructionAction]` shape. The prototype follows the
  compileable preview shape.
- Owner/sender authorization is modelled for basic-account withdrawal and
  factory initiation; provider authorization is modelled for provider-managed
  transfer initiation, accept/reject, and withdraw. Wrong V2 actors are
  rejected by local guards before mutation, and tests assert those
  guard-specific failure markers.
- V2 accept/reject/withdraw choices delegate to the existing consuming V1
  accept/reject/withdraw choices through `splice-token-standard-utils` for
  basic accounts, so V1 archival and result behavior is preserved. The
  provider-managed instruction is V2-only and archives itself explicitly.
- Account-configuration contracts, owner/provider joint approval, provider
  revocation, and Wallet SDK authority remain later slices.
- The current preview DAR has no `TransferInstruction_Update` choice. OZ's V1
  `TransferInstruction_Update` remains a `fail` stub (Q10 in `docs/SCOPE.md`
  §7), and this slice maps the real V2 update path onto preview
  accept/reject/withdraw choices instead.

### Allocation.daml + AllocationRequest.daml

CIP-0112 §4.3.1 introduces multi-leg `Allocation` with
`AllocationSpecification.transferLegs : [TransferLeg]` and an
`authorizer : Account` field. §5.1 requires two `Allocation` implementations
on V2-compatible assets: a V2-only flow and a V1/V2-compat flow.

Implementation status:

- Implemented a V2-only `SimpleAllocationInstructionV2` and
  `SimpleAllocationV2` for basic accounts and regular provider-managed
  accounts.
- The current V2 factory creates a pending instruction, and instruction accept
  creates a locked allocation. This is intentionally less optimized than the
  Splice preview state machine, but it proves the instruction surface directly.
- `SimpleAllocationInstructionV2` carries an internal
  `AllocationLockViewPolicy` field selected from the factory choice context.
  Missing context defaults to the admin-only policy, so existing V2 allocation
  behavior is preserved. The account-party policy is opt-in and V2-only.
- `SimpleAllocationV2.Allocation_Settle` now supports the narrow
  SettlementFactory-driven regular-account path: it archives the allocation
  and locked holdings, verifies available locked funding against net
  transfer-leg debits and requested next-iteration reserves, creates credited
  holdings in the authorizer account shape, and logs a V2
  allocation-settlement event when the settlement factory supplies the rules
  EventLog context.
- Iterated settlement is implemented for `SimpleAllocationV2`: allocations
  that carry `nextIterationFunding = Some ...` may settle additional
  `extraTransferLegSides`, create successor allocations with incremented
  `numIterations`, cancel or withdraw successor allocations, and finalize with
  `nextIterationFunding = None`.
- `SimpleAllocationRequestV2` now implements the preview V2
  `AllocationRequest` interface as the app/wallet-facing request entry point
  alongside the direct authorizer-side V2 allocation factory path. It preserves
  V1 request behavior through the existing `SimpleAllocationRequest` template
  and a V1 downcast view on the V2 request template.
- V1 `SimpleAllocation` and the existing V1 allocation tests are unchanged.
- `SimpleAllocationV1Compat`, automatic receipt allocations,
  `extraReceiptAuthorizers`, and reduced-privacy settlement observer modes
  remain later slices.

### Rules.daml (factory)

CIP-0112 §4.3.1 introduces `SettlementFactory` as a separate interface,
with `nonconsuming choice SettlementFactory_SettleBatch`. The spec keeps
it "separate from `AllocationFactory` to give implementations the
flexibility to keep the choices on one or two contracts."

Implementation actions:

- Current transfer decision: `SimpleTokenRules` carries V1 `TransferFactory`,
  V1 `AllocationFactory`, preview V2 `TransferFactory`, and preview V2
  `AllocationFactory`, preview V2 `SettlementFactory`, and preview V2
  `EventLog` in one template, matching the Splice `TestTokenV2.TokenRules`
  default. A split settlement-rules template remains unproven and should not
  be introduced without local code pressure.
- The existing `SimpleTokenRules.supportedInstruments` is the multi-instrument
  list. CIP-0112 §4.3.4 requires "Allocations MUST NOT contain transfer
  legs for instruments with different admin parties." The OZ design
  already satisfies this implicitly (one admin per
  `SimpleTokenRules`), but the V2 invariants must enforce it explicitly.
- Configurable `actors : [Party]` (§4.3.1) on `SettlementFactory_SettleBatch`
  replaces V1's fixed `[executor, sender, receiver]` controllers. OZ must
  implement the `actors` validation rules ("Implementations MUST check
  this value to avoid unauthorized settlement execution").
- `TransferFactory_Transfer` dispatches provider-managed regular accounts to
  a V2-only path instead of V1 delegation. `AllocationFactory_Allocate` and
  `SettlementFactory_SettleBatch` now validate supported regular accounts
  rather than basic accounts only.
- `settlementFactory_settleBatchExtraObservers` now uses the private-asset
  default (`[]`) and `allocationFactory_allocateExtraObservers` keeps the
  preview utility private-asset default. The default for the OZ token template
  remains the §4.3.5.1 baseline (privacy preserved across traders) unless
  §4.3.5.3 (reduced privacy) is explicitly opted in by an asset variant.
- `SettlementFactory_SettleBatch` accepts empty `transferLegs` only when
  allocations are supplied, so iterated allocations can be refreshed or
  finalized without a new transfer leg while a no-op empty batch remains
  rejected.

### Preapproval.daml

OZ's `TransferPreapproval_Send` is a nonconsuming choice (Q2 in
`docs/SCOPE.md` §7). CIP-0112 does not directly change preapproval; the
V2 changes propagate via the `Account` type substitution.

Implementation actions:

- Surface whether V2 introduces `Account`-aware preapproval semantics
  (e.g., preapprove a specific `Account.provider` rather than a `Party`).
  Currently CIP-0112 does not specify; treat as an asset-implementation
  choice.

### ContextUtils.daml

CIP-0112 §4.3.5 introduces choice observers; observer-derivation
helpers belong here.

Implementation actions:

- Implemented helper `accountObservers : Account -> [Party]` for owner+provider
  and `accountAuthority : Party -> Account -> Party` for owner/basic versus
  provider-managed actor resolution.
- Plan helper `settleBatchObservers : SettlementInfo -> [TransferLeg] ->
  [Party]` for the §4.3.5 baseline.
- Plan helper `validateAuthorizers : [Allocation] -> [Party] -> Either
  Text ()` for the per-§5.1 V1/V2 dual-impl validation.

## Test impact

| Existing test class | Count | V2 impact |
| :--- | :--- | :--- |
| Experimental CIP-112 probe | 67 | The first 3 tests cover basic-account Holding projection. The V2 transfer/compatibility probes cover basic-account initiation, pending action visibility, V1/V2 cross-choice accept, accept/reject/withdraw execution, instruction archival, explicit EventLog-context reporting with locked-holding input semantics, factory EventLog-context non-persistence, non-participant EventLog privacy, wrong or duplicate V2 actors, missing rules disclosure, and provider-managed receiver support. The V2 allocation/settlement/compatibility probes cover basic-account instruction creation, accept-to-allocation locking, multi-leg net-debit funding, pending withdraw CID recovery for live and stale inputs, private allocation-instruction observers, allocation withdraw/cancel unlocks, wrong actors, non-basic accounts, malformed allocation rejection, committed-withdraw deadline behavior, unexpired locked-input rejection, order-insensitive multi-executor cancel, matching batch settlement, admin/executor overlap, iterated settlement with successor allocations, wrong settlement actors, non-basic settlement transfer-leg accounts, missing receiver allocation rollback, V1 allocation artifact rejection by V2 settlement, stale allocation CID rollback, and settlement-deadline rollback. The 12 provider-managed probes cover transfer authority matrix, allocation settlement, allocation withdraw/cancel, admin-only and account-party allocation lock-view policies, unsupported lock-view policy rejection, missing provider, wrong provider, wrong owner, unauthorized actor lists, allocation observer minimization, provider spoofing, stale instruction/allocation behavior, and locked-input contention under both allocation lock policies. The 7 allocation-request probes cover request create/view/disclosure, reject, withdraw, expiry, stale replay, unsupported iterated shape, hidden observer expansion, zero/partial/duplicate/extra factory-call coverage, missing factory disclosure, provider/basic-account mismatches, explicit-disclosure and submitMulti request-driven paths, and request-driven multi-leg settlement. The 8 iterated-settlement probes cover partial repeated settlement, receiver-side top-up, finalization, cancellation after iteration, uncommitted and committed withdrawal recovery, invalid funding, direct-allocation versus factory validation boundaries, stale replay/no-double-settle, deadline rollback, conservation checks, and the `extraReceiptAuthorizers`/receipt-allocation preview blocker. These are not V2 conformance tests. |
| Transfer tests | 9 | Existing V1 transfer tests remain unchanged and passing. The V2 tests cover the smallest basic-account transfer/factory paths plus targeted cross-version, disclosure, observer, actor, provider-managed receiver, and non-basic unsupported account cases. Wrong V2 actors are asserted to fail before V1 delegation or V2-only provider-managed mutation. Explicit accept-time EventLog-context reporting is covered for the basic path; provider-managed transfer EventLog remains accept-context driven and not a production reporting claim. Direct/self transfer EventLog reporting and reject/withdraw holding-change events remain unimplemented. Full §5.4 coverage still requires more sender×receiver×asset rows and account-configuration variants. |
| Allocation tests | 5 | Existing V1 allocation tests remain unchanged and passing. The V2 allocation probes cover the smallest basic-account instruction/allocation path, provider-managed allocation/settlement/withdraw/cancel paths, targeted negative cases, matching sender/receiver `SettlementFactory_SettleBatch`, multi-leg net funding, private observer checks, stale-state and deadline rollback, admin/executor overlap, and iterated-settlement successor allocations. The allocation-request probes now cover a request-driven path into the existing V2 allocation factory and settlement flow. Iterated-settlement probes add partial repeated settlement, receiver-side top-up, finalization, cancellation, uncommitted and committed withdrawal recovery, invalid funding, direct/factory validation-boundary coverage, stale replay/no-double-settle, and receipt-auto-creation blocker coverage. V1/V2-compatible Allocation implementations, automatic receipt allocations, reduced-privacy observer mode, V1 asset contrast packages, and full §5.5 app×settlement×wallet×asset variants remain required. |
| Defragmentation tests | 2 | Likely unchanged; defrag is a sender-only self-transfer that does not interact with V2 settlement. Verify. |
| Security tests | 20 | Invariants tied to `owner : Party` (notably #17 per-input instrumentId check, and authority invariants from `docs/PLAN.md` §9) need V2-equivalent restatements over `Account`. |

New test scenarios from CIP-0112 to add:

- §4.1.1 / §4.2 three-trader privacy-preserving batch settlement.
- §4.3.5 view-count baselines: 28-view baseline, 21-view UI-driven, 25-view
  reduced-privacy, 25-view eliding, 4-view fully optimised.
- Joint-account and account-configuration support must add explicit actor
  set/order/dedup tests before it is considered hardened. The current
  provider-managed slice deliberately uses singleton provider actor lists;
  multi-party account support must prove whether actor validation is
  order-insensitive, duplicate-rejecting, and complete for every allowed actor
  set before any production-style delegation claim.

## Verification pipeline impact

`scripts/verify.sh` orchestrates `daml-lint` → `daml-props` → `daml-verify`
today. Discovery findings:

- `daml-lint`: no V1-only assumptions. Confirm during the implementation
  slice; add a CIP-112 dual-implementation detector if surfacing is
  desirable.
- `daml-props`: the property catalog now includes two V2 iterated-settlement
  rows: reserve conservation across random partial/finalize/cancel/withdraw
  sequences, and consumed-allocation replay/no-double-settle conservation
  across longer sequences. The model tracks allocation lifecycle ids, explicit
  stale-replay target ids, and a rejected repeated-settlement counter rather
  than treating stale replay as a bare no-op. Full generators still need to
  learn the preview `Account` shape; the ownership-by-party invariants need
  restatement before this becomes a V2 conformance property suite.
- `daml-verify`: conservation symbolic model becomes per-account; add
  `cip-112-conservation` for iterated funding and `cip-112-authorization` for
  `actors` validation. Those proof rows are re-baselining targets, not
  completed proof claims from this slice.

## Open questions for the stakeholder packet

Inherits CIP-112-D-001 through CIP-112-D-012 from
`/Users/x/canton/docs/architecture/cip-112-extension-plan.md` §7. Tool-specific
additions:

- TT-Q-001: Should `SimpleTokenRules` carry all V1+V2 interfaces, or split
  to a new `SimpleSettlementRules`? (Affects backward-compat surface and
  on-ledger contract count.)
- TT-Q-002: Should `SimpleAllocationV1Compat` be a strict subset of
  `SimpleAllocationV2` (struct sub-typing pattern) or a parallel template
  with explicit duplication? (Affects audit reviewability.)
- TT-Q-003: For `LockedSimpleHolding`, the current prototype keeps
  Party-keyed locks because the preview V2 `Lock` still has
  `holders : [Party]`. Release-quality work should confirm with the CIP
  authors that this is intentional.

## License note

`canton-token-template` is AGPL-3.0. Extending it with V2 interfaces in
this repo is compatible with the AGPL license. **Importing the extended
source into `repos/oz-daml-contracts/` (target license: MIT per
`SCOPE.md` §10) requires resolving M1-PF-004 first.** Discovery work in
this repo does not by itself bind any future license decision.

## Non-goals

- No public API hardening in `repos/oz-daml-contracts/` ahead of the
  workspace slice 18 RI-backed extraction sequence and the M1-PF-004
  license-boundary gate.
- No broad preview DAR vendoring. Add a preview DAR (e.g.
  `splice-api-token-allocation-request-v2`) only in the slice that
  directly imports and validates it; record provenance and SHA-256 in the
  prototype evidence section. The current direct V2 DAR footprint is
  Holding, TransferInstruction, Allocation, AllocationInstruction,
  AllocationRequest, TransferEvents, and Utils.
- No new SDK/Canton/Splice pins unless an active slice proves the exact
  need; observations feed back into M1-PF-001 rather than landing a pin
  here.
- No hosted CI; local manual workflow remains the evidence path.
