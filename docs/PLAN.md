# PLAN: CIP-056 Simple Token Implementation

Status:
- **CIP-056 (V1):** on-ledger implementation complete (36/36 tests passing).
- **CIP-112 (V2):** experimental, non-conformant prototype landed across
  Holding, Transfer, Allocation, Settlement, AllocationRequest, iterated
  settlement, and provider-managed accounts — see
  [CIP-0112-EXTENSION-PLAN.md](CIP-0112-EXTENSION-PLAN.md).
- **Admin layer (AccessControl / Ownable / Pausable):** implemented, green, and
  **converged** (slices AL-0..AL-3) under `SimpleToken/Admin/` plus the Pausable
  chokepoints on `SimpleTokenRules`. Uses the hybrid **AccessControl-as-substrate**
  model: the `Admin` role is the Ownable owner; Pausable is `Pauser`-gated
  origination control; the `Admin` role is root-managed (one-level delegation) via
  the single-source `delegableRole` policy, which `simple-token` enforces at
  **compile time** (`-Werror=incomplete-patterns`). Hardened across five
  `/code-review` → fix rounds (0 correctness defects). Design:
  [ADMIN-LAYER-PLAN.md](ADMIN-LAYER-PLAN.md).
- **Supply management (Mint / Burn — AL-5/AL-6):** implemented, green, and
  **converged** — capability-gated mint and burn on top of the admin layer. Mint =
  A3 dispatch (preapproval-direct or `MintProposal`); Burn = B3 (owner redemption +
  `Burner`-gated forced clawback, incl. in-flight/locked funds); supply = D3
  enforced per-`Minter` allowance + D2 advisory `TotalSupply`. 13 tests in
  `Test/MintBurn.daml`; hardened across two `/code-review` → fix rounds. See
  [ADMIN-LAYER-PLAN.md §11](ADMIN-LAYER-PLAN.md#11-supply-management--mint--burn-al-5--al-6).
- **Test status:** full suite **141/141 passing** and `scripts/verify.sh` **3/3**
  (daml-lint clean on new source, daml-props, daml-verify 14/14) on SDK 3.4.11 /
  Java 21. Both AL-0..AL-3 and AL-5/AL-6 have reached a clean `/code-review` pass.
  Active/next work and sequencing: [§17](#17-active-and-future-slices-admin-layer--cip-112).

Scope: see [SCOPE.md](SCOPE.md) for authoritative scope boundaries, out-of-scope items, and post-MVP backlog.

## Goal

Deliver a minimal, secure, readable Canton token-standard implementation in DAML/Haskell that:
- Implements all 6 CIP-056 interface packages as-is (stable ABI)
- Is simpler than Splice's Amulet implementation while preserving required behavior and safety conventions

## Non-Goals

See [SCOPE.md §3](SCOPE.md#3-out-of-scope) for the full out-of-scope list.

---

## 1. CIP-056 Interface Specification

### 1.1 Dependency Graph

```
splice-api-token-metadata-v1          (AnyValue, ChoiceContext, Metadata, ExtraArgs)
    |
    v
splice-api-token-holding-v1           (InstrumentId, Lock, Holding, HoldingView)
    |
    +---> splice-api-token-transfer-instruction-v1   (Transfer, TransferFactory, TransferInstruction)
    |
    +---> splice-api-token-allocation-v1             (SettlementInfo, TransferLeg, Allocation)
              |
              +---> splice-api-token-allocation-instruction-v1  (AllocationFactory, AllocationInstruction)
              |
              +---> splice-api-token-allocation-request-v1      (AllocationRequest)
```

Our packages depend on all 6 interface DARs. We implement templates against them; we never modify the interfaces.

### 1.2 Data Types (MetadataV1)

| Type | Fields | Notes |
|---|---|---|
| `AnyValue` | Sum: `AV_Text`, `AV_Int`, `AV_Decimal`, `AV_Bool`, `AV_Date`, `AV_Time`, `AV_RelTime`, `AV_Party`, `AV_ContractId`, `AV_List`, `AV_Map` | Used in `ChoiceContext.values` |
| `AnyContract` | Interface, viewtype `AnyContractView` | Never implemented; use only as `ContractId AnyContract` via `coerceContractId` |
| `AnyContractId` | `= ContractId AnyContract` | Type alias for opaque contract references |
| `ChoiceContext` | `values : TextMap AnyValue` | App-backend-to-choice plumbing; keys are app-internal |
| `Metadata` | `values : TextMap Text` | DNS-prefix keys (k8s convention); keep small |
| `ExtraArgs` | `context : ChoiceContext`, `meta : Metadata` | Passed to every interface choice |
| `ChoiceExecutionMetadata` | `meta : Metadata` | Generic choice result wrapper |

### 1.3 Holding Interface (HoldingV1)

| Type | Fields |
|---|---|
| `InstrumentId` | `admin : Party`, `id : Text` |
| `Lock` | `holders : [Party]`, `expiresAt : Optional Time`, `expiresAfter : Optional RelTime`, `context : Optional Text` |
| `HoldingView` | `owner : Party`, `instrumentId : InstrumentId`, `amount : Decimal`, `lock : Optional Lock`, `meta : Metadata` |

**Interface**: `Holding` — viewtype `HoldingView`, no choices defined on the interface itself.

### 1.4 Transfer Interfaces (TransferInstructionV1)

| Type | Fields |
|---|---|
| `Transfer` | `sender : Party`, `receiver : Party`, `amount : Decimal`, `instrumentId : InstrumentId`, `requestedAt : Time`, `executeBefore : Time`, `inputHoldingCids : [ContractId Holding]`, `meta : Metadata` |
| `TransferInstructionResult` | `output : TransferInstructionResult_Output`, `senderChangeCids : [ContractId Holding]`, `meta : Metadata` |
| `TransferInstructionResult_Output` | `Pending {transferInstructionCid}` \| `Completed {receiverHoldingCids}` \| `Failed` |
| `TransferInstructionStatus` | `TransferPendingReceiverAcceptance` \| `TransferPendingInternalWorkflow {pendingActions : Map Party Text}` |
| `TransferInstructionView` | `originalInstructionCid : Optional (ContractId TransferInstruction)`, `transfer : Transfer`, `status : TransferInstructionStatus`, `meta : Metadata` |
| `TransferFactoryView` | `admin : Party`, `meta : Metadata` |

**Interface `TransferFactory`** (nonconsuming):

| Choice | Args | Controller | Returns |
|---|---|---|---|
| `TransferFactory_Transfer` | `expectedAdmin : Party`, `transfer : Transfer`, `extraArgs : ExtraArgs` | `transfer.sender` | `TransferInstructionResult` |
| `TransferFactory_PublicFetch` | `expectedAdmin : Party`, `actor : Party` | `actor` | `TransferFactoryView` |

**Interface `TransferInstruction`** (consuming):

| Choice | Args | Controller | Returns |
|---|---|---|---|
| `TransferInstruction_Accept` | `extraArgs : ExtraArgs` | `transfer.receiver` | `TransferInstructionResult` |
| `TransferInstruction_Reject` | `extraArgs : ExtraArgs` | `transfer.receiver` | `TransferInstructionResult` |
| `TransferInstruction_Withdraw` | `extraArgs : ExtraArgs` | `transfer.sender` | `TransferInstructionResult` |
| `TransferInstruction_Update` | `extraActors : [Party]`, `extraArgs : ExtraArgs` | `instrumentId.admin, extraActors` | `TransferInstructionResult` |

### 1.5 Allocation Interfaces (AllocationV1)

| Type | Fields |
|---|---|
| `Reference` | `id : Text`, `cid : Optional AnyContractId` |
| `SettlementInfo` | `executor : Party`, `settlementRef : Reference`, `requestedAt : Time`, `allocateBefore : Time`, `settleBefore : Time`, `meta : Metadata` |
| `TransferLeg` | `sender : Party`, `receiver : Party`, `amount : Decimal`, `instrumentId : InstrumentId`, `meta : Metadata` |
| `AllocationSpecification` | `settlement : SettlementInfo`, `transferLegId : Text`, `transferLeg : TransferLeg` |
| `AllocationView` | `allocation : AllocationSpecification`, `holdingCids : [ContractId Holding]`, `meta : Metadata` |
| `Allocation_ExecuteTransferResult` | `senderHoldingCids : [ContractId Holding]`, `receiverHoldingCids : [ContractId Holding]`, `meta : Metadata` |
| `Allocation_CancelResult` | `senderHoldingCids : [ContractId Holding]`, `meta : Metadata` |
| `Allocation_WithdrawResult` | `senderHoldingCids : [ContractId Holding]`, `meta : Metadata` |

**Interface `Allocation`**:

| Choice | Args | Controller | Returns |
|---|---|---|---|
| `Allocation_ExecuteTransfer` | `extraArgs : ExtraArgs` | `[executor, sender, receiver]` | `Allocation_ExecuteTransferResult` |
| `Allocation_Cancel` | `extraArgs : ExtraArgs` | `[executor, sender, receiver]` | `Allocation_CancelResult` |
| `Allocation_Withdraw` | `extraArgs : ExtraArgs` | `transferLeg.sender` | `Allocation_WithdrawResult` |

### 1.6 Allocation Instruction Interfaces (AllocationInstructionV1)

| Type | Fields |
|---|---|
| `AllocationInstructionView` | `originalInstructionCid : Optional (ContractId AllocationInstruction)`, `allocation : AllocationSpecification`, `pendingActions : Map Party Text`, `requestedAt : Time`, `inputHoldingCids : [ContractId Holding]`, `meta : Metadata` |
| `AllocationFactoryView` | `admin : Party`, `meta : Metadata` |
| `AllocationInstructionResult` | `output : AllocationInstructionResult_Output`, `senderChangeCids : [ContractId Holding]`, `meta : Metadata` |
| `AllocationInstructionResult_Output` | `Pending {allocationInstructionCid}` \| `Completed {allocationCid}` \| `Failed` |

**Interface `AllocationFactory`** (nonconsuming):

| Choice | Args | Controller | Returns |
|---|---|---|---|
| `AllocationFactory_Allocate` | `expectedAdmin : Party`, `allocation : AllocationSpecification`, `requestedAt : Time`, `inputHoldingCids : [ContractId Holding]`, `extraArgs : ExtraArgs` | `allocation.transferLeg.sender` | `AllocationInstructionResult` |
| `AllocationFactory_PublicFetch` | `expectedAdmin : Party`, `actor : Party` | `actor` | `AllocationFactoryView` |

**Interface `AllocationInstruction`**:

| Choice | Args | Controller | Returns |
|---|---|---|---|
| `AllocationInstruction_Withdraw` | `extraArgs : ExtraArgs` | `allocation.transferLeg.sender` | `AllocationInstructionResult` |
| `AllocationInstruction_Update` | `extraActors : [Party]`, `extraArgs : ExtraArgs` | `allocation.transferLeg.instrumentId.admin, extraActors` | `AllocationInstructionResult` |

### 1.7 Allocation Request Interface (AllocationRequestV1)

| Type | Fields |
|---|---|
| `AllocationRequestView` | `settlement : SettlementInfo`, `transferLegs : TextMap TransferLeg`, `meta : Metadata` |

**Interface `AllocationRequest`**:

| Choice | Args | Controller | Returns |
|---|---|---|---|
| `AllocationRequest_Reject` | `actor : Party`, `extraArgs : ExtraArgs` | `actor` | `ChoiceExecutionMetadata` |
| `AllocationRequest_Withdraw` | `extraArgs : ExtraArgs` | `settlement.executor` | `ChoiceExecutionMetadata` |

---

## 2. Template Designs

### 2.1 `SimpleHolding` (unlocked holding)

**Implementation:** `simple-token/daml/SimpleToken/Holding.daml`

```
template SimpleHolding
  with
    admin : Party
    owner : Party
    instrumentId : InstrumentId
    amount : Decimal
    meta : Metadata
  where
    signatory admin, owner

    interface instance Holding for SimpleHolding where
      view = HoldingView with
        owner
        instrumentId
        amount
        lock = None
        meta
```

- No choices on the template itself; holdings are consumed by factory logic.
- `admin` is the registry operator (equivalent to Splice's `dso`).

> `ensure amount > 0.0` enforced on the template as defense-in-depth (post-MVP hardening). Also enforced at the factory level (invariant #6).

### 2.2 `LockedSimpleHolding` (locked holding)

**Implementation:** `simple-token/daml/SimpleToken/Holding.daml`

```
template LockedSimpleHolding
  with
    admin : Party
    owner : Party
    instrumentId : InstrumentId
    amount : Decimal
    lock : Lock
    extraObservers : [Party]
    meta : Metadata
  where
    signatory admin, owner, lock.holders
    observer extraObservers

    interface instance Holding for LockedSimpleHolding where
      view = HoldingView with
        owner
        instrumentId
        amount
        lock = Some lock
        meta
```

- Lock holders are signatories (matches Splice's `LockedAmulet` pattern).
- Holdings with expired locks SHOULD be accepted as transfer inputs (spec requirement).

> **Gap 1 resolved:** Added `extraObservers : [Party]` field with `observer extraObservers`. Set to `[transfer.receiver]` during two-step transfers and `[settlement.executor, leg.receiver]` during allocations. This is a correctness requirement — without it, receivers/executors cannot see the locked holding.
>
> `ensure amount > 0.0` enforced on the template as defense-in-depth (post-MVP hardening). Also enforced at the factory level (invariant #6).
>
> `LockedSimpleHolding_Unlock` choice implemented (post-MVP hardening): owner can unlock holdings with expired locks. Unexpired locks cannot be unlocked (admin authority required). Transfer/allocation instruction choices handle the expire-lock context pattern for cases where the owner has already unlocked.

### 2.3 `SimpleTokenRules` (transfer factory + allocation factory)

**Implementation:** `simple-token/daml/SimpleToken/Rules.daml`

```
template SimpleTokenRules
  with
    admin : Party
    supportedInstruments : [Text]
  where
    signatory admin

    interface instance TransferFactory for SimpleTokenRules where
      view = TransferFactoryView with admin; meta = emptyMetadata
      transferFactory_transferImpl ...
      transferFactory_publicFetchImpl ...

    interface instance AllocationFactory for SimpleTokenRules where
      view = AllocationFactoryView with admin; meta = emptyMetadata
      allocationFactory_allocateImpl ...
      allocationFactory_publicFetchImpl ...
```

**`transferFactory_transferImpl` dispatches 3 transfer modes** (mirroring `amulet_transferFactory_transferImpl`):

1. **Self-transfer** (`sender == receiver`): Consume inputs, create new holding for sender. Returns `Completed`.
2. **Direct transfer** (preapproval present in `ChoiceContext`): Consume inputs, exercise nonconsuming `TransferPreapproval_Send` to create receiver holding. Returns `Completed`.
3. **Two-step transfer** (no preapproval, `sender != receiver`): Lock funds into `LockedSimpleHolding`, create `SimpleTransferInstruction`. Returns `Pending`.

**`allocationFactory_allocateImpl`**: Lock funds into `LockedSimpleHolding`, create `SimpleAllocation`. Returns `Completed`.

Both factory impls share the `archiveAndSumInputs` helper (see Gap 9) and validate all security invariants #1-#20 before any state mutation.

> **Gap 7 resolved:** Added `supportedInstruments : [Text]` field. Factory validates `transfer.instrumentId.id ∈ supportedInstruments`. One factory instance supports multiple instrument IDs, reducing operational overhead for multi-token registries.
>
> **Gap 9 resolved:** Consolidated input validation into `archiveAndSumInputs` helper function that validates owner (#10), instrumentId (#17), lock status (#19/#20), archives each holding, and returns the total sum. Shared between transfer and allocation paths.
>
> **Deviation from plan:** Plan had separate `simpleTransferImpl` / `simpleAllocateImpl` top-level functions called from the interface instances. Implementation inlines the logic directly in the interface instance methods, with `selfTransfer`, `directTransfer`, and `twoStepTransfer` as extracted helpers. Functionally equivalent, structurally different.

### 2.4 `SimpleTransferInstruction`

**Implementation:** `simple-token/daml/SimpleToken/TransferInstruction.daml`

```
template SimpleTransferInstruction
  with
    admin : Party
    transfer : Transfer
    lockedHoldingCid : ContractId LockedSimpleHolding
    originalInstructionCid : Optional (ContractId TransferInstruction)
  where
    signatory admin, transfer.sender
    observer transfer.receiver
```

- Mirrors `AmuletTransferInstruction`: signatories are `admin + sender`, observer is `receiver`.
- Accept: validates `executeBefore` deadline first (Gap 8), then unlocks the locked holding and creates new holding owned by receiver.
- Reject/Withdraw: unlock, return to sender via shared `returnLockedFundsToSender` helper. Both return `Failed`.
- Update: fails with `"TransferInstruction_Update is not supported"`.

> **Deviation from plan:** Added explicit `admin : Party` field (plan derived it from `transfer.instrumentId.admin`). This makes the signatory clause cleaner: `signatory admin, transfer.sender` instead of `signatory transfer.instrumentId.admin, transfer.sender`. Functionally equivalent since the factory always sets `admin = transfer.instrumentId.admin`.
>
> **Deviation from plan:** Added `originalInstructionCid : Optional (ContractId TransferInstruction)` field. This is required by the `TransferInstructionView` interface — the view must return it. Set to `None` on initial creation.
>
> **Gap 8 resolved:** Accept validates `now < transfer.executeBefore` before fetching/archiving the locked holding. Deadline check is the first operation in the choice body.

### 2.5 `SimpleAllocation`

**Implementation:** `simple-token/daml/SimpleToken/Allocation.daml`

```
template SimpleAllocation
  with
    admin : Party
    allocation : AllocationSpecification
    lockedHoldingCid : ContractId LockedSimpleHolding
  where
    signatory admin, allocation.transferLeg.sender
    observer allocation.settlement.executor, allocation.transferLeg.receiver
```

- Mirrors `AmuletAllocation`: signatories are `admin + sender`, observers are `executor` and `receiver`.
- `ExecuteTransfer`: validates `settleBefore` deadline first (Gap 8), then unlocks and transfers.
- `Cancel`: unlocks and returns to sender via shared `releaseAllocatedFunds` helper.
- `Withdraw`: validates `allocateBefore` deadline, then unlocks and returns to sender.

> **Deviation from plan:** Added explicit `admin : Party` field (same rationale as SimpleTransferInstruction).
>
> **Deviation from plan:** Added `allocation.transferLeg.receiver` as observer. Necessary for receiver visibility in DvP settlement — without it, the receiver cannot see the allocation contract when `Allocation_ExecuteTransfer` needs their authorization.
>
> **Gap 8 resolved:** `ExecuteTransfer` validates `now < settlement.settleBefore` before archival. `Withdraw` validates `now < settlement.allocateBefore` before archival. These are defensive checks beyond what the plan specified for allocation choices.

### 2.6 `SimpleAllocationRequest` (optional test helper)

**Implementation:** `simple-token/daml/SimpleToken/AllocationRequest.daml`

```
template SimpleAllocationRequest
  with
    settlement : SettlementInfo
    transferLegs : TextMap TransferLeg
    senders : [Party]
  where
    signatory settlement.executor
    observer senders

    interface instance AllocationRequest for SimpleAllocationRequest where
      view = AllocationRequestView with settlement; transferLegs; meta = emptyMetadata
      ...
```

- Utility template for testing DvP workflows; not part of core registry.

> **Deviation from plan:** Uses explicit `senders : [Party]` field instead of computed `observer (map (.sender) $ values transferLegs)`. This avoids a Daml compilation issue with complex expressions in `observer` clauses. Functionally equivalent — callers set `senders` to the list of leg senders.
>
> **Deviation from plan:** Removed `meta : Metadata` template field. The view returns `emptyMetadata` directly. Simplifies the template at no functional cost since metadata is not used in allocation request workflows.

### 2.7 `TransferPreapproval` (direct transfer authorization)

**Implementation:** `simple-token/daml/SimpleToken/Preapproval.daml`

```
template TransferPreapproval
  with
    admin : Party
    receiver : Party
    instrumentId : InstrumentId
    expiresAt : Optional Time
    meta : Metadata
  where
    signatory admin, receiver

    nonconsuming choice TransferPreapproval_Send : ([ContractId Holding], [ContractId Holding])
      with
        sender : Party
        transferInstrumentId : InstrumentId
        amount : Decimal
        totalInput : Decimal
        holdingMeta : Metadata
      controller admin, sender
      do
        -- Invariant #16: instrumentId match
        -- Check expiry if set
        -- Create receiver holding (receiver auth from preapproval signatories)
        -- Create sender change if needed
        ...
```

- Created by receiver ahead of time to authorize incoming direct transfers.
- The factory reads the preapproval contract ID from `ChoiceContext` under key `"transfer-preapproval"`.
- Signatories: `admin + receiver`. Controller: `admin, sender`.
- Lives in `Preapproval.daml`.

> **Gap 2 resolved:** Holding creation happens inside the preapproval choice body, which has receiver's signatory authority. This is an authorization model requirement — the factory alone cannot create `SimpleHolding with owner = receiver` because it lacks the receiver's signature.
>
> **Major deviation from plan:** Choice is **nonconsuming** (`TransferPreapproval_Send`) instead of consuming (`TransferPreapproval_Accept`). This matches both Splice and Standard2 behavior — a single preapproval can be reused for multiple incoming transfers. Consuming preapprovals would force the receiver to recreate one after every transfer, which is poor UX.
>
> **Deviation from plan:** Added `expiresAt : Optional Time` and `meta : Metadata` fields. Expiry is validated inside the choice body. These fields exist in Splice's `TransferPreapproval` and are needed for spec completeness.
>
> **Deviation from plan:** Controller changed from `sender` to `admin, sender`. The `admin` controller is required because the choice creates holdings with `signatory admin, owner` — the admin must authorize the `create` calls inside the choice body.
>
> **Deviation from plan:** Choice takes `totalInput : Decimal` (pre-computed by factory) instead of `inputHoldingCids`. Input archival and summing happens in the factory via `archiveAndSumInputs` before the preapproval is exercised. The preapproval only creates output holdings.

---

## 3. Module Structure

### Actual (implemented)

```
canton-network-token-standard/
  dars/                                  -- Symlinks to splice-api-token-* DARs
    splice-api-token-allocation-instruction-v1-1.0.0.dar -> ../../../splice/daml/dars/...
    splice-api-token-allocation-request-v1-1.0.0.dar     -> ...
    splice-api-token-allocation-v1-1.0.0.dar             -> ...
    splice-api-token-holding-v1-1.0.0.dar                -> ...
    splice-api-token-metadata-v1-1.0.0.dar               -> ...
    splice-api-token-transfer-instruction-v1-1.0.0.dar   -> ...

  simple-token/                          -- Production DAR (IMPLEMENTED)
    daml/
      SimpleToken/
        Holding.daml                     -- SimpleHolding, LockedSimpleHolding
        Rules.daml                       -- SimpleTokenRules + archiveAndSumInputs + dispatch helpers
        TransferInstruction.daml         -- SimpleTransferInstruction + returnLockedFundsToSender
        Allocation.daml                  -- SimpleAllocation + releaseAllocatedFunds
        AllocationRequest.daml           -- SimpleAllocationRequest (test helper, in production DAR)
        ContextUtils.daml                -- ToAnyValue/FromAnyValue, context lookup helpers
        Preapproval.daml                 -- TransferPreapproval (nonconsuming)
    daml.yaml                            -- SDK 3.4.11, target 2.1, depends on 6 splice DARs

  simple-token-test/                     -- Test DAR (IMPLEMENTED, 36/36 passing)
    daml/
      SimpleToken/
        Testing/
          SimpleRegistry.daml            -- SimpleRegistry, disclosure helpers, factory enrichment
          WalletClient.daml              -- listHoldings, checkBalance, listTransferOffers
        Test/
          Setup.daml                     -- TestParties, TestEnv, setupTestEnv, fundParty
          Transfer.daml                  -- 7 transfer lifecycle tests
          Allocation.daml                -- 5 allocation + DvP tests
          Defragmentation.daml           -- 2 tests: 10-holding merge, multi-instrument transfer
          Negative.daml                  -- 12 security/negative tests + 1 positive (expired lock)
    daml.yaml                            -- depends on simple-token DAR

```

> **Deviation from plan:** `Util.daml` became `ContextUtils.daml` — a 137-line module with `ToAnyValue`/`FromAnyValue` typeclasses and `lookupFromContext`/`getFromContext` helpers. This is more complex than the planned "requireExpectedAdminMatch, time validation, holding helpers" utility module, but the typeclasses are necessary for type-safe `ChoiceContext` serialization/deserialization. The pattern is borrowed from Splice's `TokenApiUtils`.
>
> **Deviation from plan:** `AllocationRequest.daml` lives in the production DAR (`simple-token/`) rather than the test DAR. This is because the `AllocationRequest` interface instance must be in the same package as the template definition for Daml's interface resolution to work correctly. The template is still only used in tests.
>
> **Deviation from plan:** Test structure uses `Testing/` subdirectory for infrastructure (`SimpleRegistry.daml`, `WalletClient.daml`) separate from `Test/` for actual tests. Plan had flat `Test/` structure.
>
> **Deviation from plan:** `Test/DvP.daml` was merged into `Test/Allocation.daml` (DvP is test_dvpTwoLegs). `Test/Defragmentation.daml` was added for merge and multi-instrument tests. `Test/AllocationRequest.daml` was not created as a separate file.

---

## 4. Transfer Flows

### 4.1 Self-Transfer (merge/split)

Sender and receiver are the same party. Completes immediately.

1. **Off-ledger**: Wallet fetches `TransferFactory` cid and `ChoiceContext` from registry API.
2. **On-ledger**: Wallet exercises `TransferFactory_Transfer` with `sender == receiver`.
3. **Impl**: `simpleTransferImpl` detects self-transfer:
   - Validates `expectedAdmin` matches `admin`.
   - Validates `amount > 0`, `requestedAt` in past, `executeBefore` in future.
   - Archives all `inputHoldingCids` (MUST archive to preserve contention guarantee).
   - Creates new `SimpleHolding` for sender with requested amount.
   - Creates change `SimpleHolding` if input total > requested amount.
4. **Result**: `TransferInstructionResult_Completed` with `receiverHoldingCids` = [new holding], `senderChangeCids` = [change if any].

### 4.2 Direct Transfer (with preapproval)

Receiver has pre-authorized receipt. Completes immediately.

1. **Off-ledger**: Wallet fetches `TransferFactory` cid. Registry includes preapproval contract id in `ChoiceContext` under a known key.
2. **On-ledger**: Wallet exercises `TransferFactory_Transfer` with preapproval in context.
3. **Impl**: `simpleTransferImpl` detects preapproval in `ChoiceContext`:
   - Validates `expectedAdmin`.
   - Validates transfer fields (amount, times, instrument).
   - Archives all `inputHoldingCids`.
   - Creates `SimpleHolding` for receiver with transfer amount.
   - Creates change `SimpleHolding` for sender.
4. **Result**: `TransferInstructionResult_Completed`.

**MVP simplification**: We implement a `TransferPreapproval` template (admin + receiver signatories) that the receiver creates ahead of time to authorize incoming transfers.

### 4.3 Two-Step Transfer (lock-then-accept)

No preapproval, sender != receiver. Requires receiver acceptance.

1. **Off-ledger**: Wallet fetches `TransferFactory` cid and `ChoiceContext`. No preapproval in context.
2. **On-ledger (step 1)**: Sender exercises `TransferFactory_Transfer`.
3. **Impl step 1**: `simpleTransferImpl` detects two-step path:
   - Validates all fields.
   - Archives all `inputHoldingCids`.
   - Creates `LockedSimpleHolding` with `lock.holders = [admin]`, `lock.expiresAt = Some executeBefore`, `lock.context = Some "transfer to <receiver>"`.
   - Creates `SimpleTransferInstruction` referencing the locked holding.
4. **Result**: `TransferInstructionResult_Pending` with `transferInstructionCid`.
5. **Off-ledger**: Receiver's wallet sees pending `TransferInstruction`. Fetches `ChoiceContext` for accept.
6. **On-ledger (step 2a — accept)**: Receiver exercises `TransferInstruction_Accept`.
   - Validates `executeBefore` not passed.
   - Archives `SimpleTransferInstruction` (consuming).
   - Exercises `LockedSimpleHolding_Unlock` (or directly archives locked holding with admin authority).
   - Creates `SimpleHolding` for receiver.
   - Returns `TransferInstructionResult_Completed`.
7. **On-ledger (step 2b — reject)**: Receiver exercises `TransferInstruction_Reject`.
   - Unlocks holding back to sender.
   - Returns `TransferInstructionResult_Failed`.
8. **On-ledger (step 2c — withdraw)**: Sender exercises `TransferInstruction_Withdraw`.
   - Unlocks holding back to sender.
   - Returns `TransferInstructionResult_Failed`.

**Lock semantics**: The admin is a lock holder so only admin+owner can unlock. This prevents the sender from unilaterally reclaiming funds during the pending window while the instruction exists. After `executeBefore`, expired locks SHOULD be accepted as inputs (spec: "Registries SHOULD allow holdings with expired locks as inputs to transfers").

---

## 5. Allocation / DvP Flow

### 5.1 Allocate

1. **Off-ledger**: App creates `SimpleAllocationRequest` (optional) and provides `AllocationFactory` cid + `ChoiceContext` to sender's wallet.
2. **On-ledger**: Sender exercises `AllocationFactory_Allocate`:
   - `simpleAllocateImpl` validates:
     - `expectedAdmin` matches.
     - `settlement.requestedAt` in past, `settlement.allocateBefore` in future.
     - `settlement.allocateBefore <= settlement.settleBefore`.
     - `transferLeg.amount > 0`.
     - `transferLeg.instrumentId` matches factory admin.
     - `requestedAt` in past.
     - At least one `inputHoldingCid`.
   - Archives all `inputHoldingCids`.
   - Creates `LockedSimpleHolding` (lock holders = [admin], expires at `settleBefore`, context = "allocation for transfer leg...").
   - Creates `SimpleAllocation` referencing the locked holding.
3. **Result**: `AllocationInstructionResult_Completed` with `allocationCid`.

### 5.2 Execute Settlement (DvP)

1. Once all legs have `Allocation` contracts, the executor (+ sender + receiver for each leg) exercises `Allocation_ExecuteTransfer` atomically in a single transaction.
2. Each `SimpleAllocation`:
   - Validates `settlement.settleBefore` not passed.
   - Unlocks locked holding.
   - Creates `SimpleHolding` for receiver.
   - Returns change to sender if applicable.
3. Atomicity: all legs in one transaction on the same synchronizer ensures DvP.

### 5.3 Cancel / Withdraw

- **Cancel** (executor + sender + receiver): Unlocks holdings, returns to sender. Used when settlement is aborted.
- **Withdraw** (sender only): Sender reclaims allocated holdings. SHOULD succeed if `allocateBefore` has not passed (sender can re-allocate).

---

## 6. Fee Model Decision

**Zero fees for MVP.**

Rationale:
- Eliminates fee reserve locking complexity (Splice locks `amount + fees * 4.0` to guard against fee changes between lock and execute).
- Eliminates mining round coupling (`OpenMiningRound`, `ClosedMiningRound`, `exercisePaymentTransfer`).
- Eliminates `ExpiringAmount` with `ratePerRound` and `createdAt` tracking.
- Transfer amount in = transfer amount out. No burn metadata to propagate.
- Change calculation becomes simple subtraction: `changAmount = sum(inputs) - requestedAmount`.

Post-MVP can introduce fees by:
1. Adding a `FeeSchedule` contract the factory fetches.
2. Deducting fees from transfer outputs.
3. Adding fee reserve multiplier to lock amounts.

---

## 7. Simplicity Decisions

| # | Splice Feature | Our Decision | Rationale |
|---|---|---|---|
| 1 | `ExpiringAmount` (holding fees via `ratePerRound`) | Flat `Decimal amount` | Zero fees; no holding decay |
| 2 | `OpenMiningRound` / `ClosedMiningRound` coupling | None | No fee calculation or round-based expiry |
| 3 | `feeReserveMultiplier = 4.0` over-lock | Lock exact amount | Zero fees; no fee drift between lock and execute |
| 4 | `PaymentTransferContext` / `exercisePaymentTransfer` | Direct create/archive | No fee engine indirection |
| 5 | `TransferPreapproval_Send` with `provider`/`beneficiaries` | Simple preapproval contract | No featured app rewards |
| 6 | `FeaturedAppRight` / `FeaturedAppActivityMarker` | Omitted | No reward system |
| 7 | `TransferCommand` with nonce deduplication | Omitted | No external-party delegation model |
| 8 | `TransferCommandCounter` | Omitted | No nonce tracking |
| 9 | `HasCheckedFetch` with `ForDso`/`ForOwner`/`ForRound` | Simple `fetch` + assert admin/owner | No group-id machinery; validate directly |
| 10 | `fetchReferenceData` for mining rounds | Omitted | No rounds to reference |
| 11 | `copyOnlyBurnMeta` propagation | `emptyMetadata` in results | No burns to propagate |

---

## 8. Contention Semantics

The CIP-056 spec mandates: "If [inputHoldingCids are] specified, then the transfer MUST archive all of these holdings, so that the execution of the transfer conflicts with any other transfers using these holdings."

**Implementation**:
- Every factory choice (`TransferFactory_Transfer`, `AllocationFactory_Allocate`) that receives non-empty `inputHoldingCids` MUST archive every listed holding contract.
- Archival is the first mutating action in the choice body, before creating new holdings.
- This means two concurrent transactions referencing the same holding will conflict at the ledger level — exactly one succeeds, the other aborts.
- Clients MUST retry on contention failures (this is normal Canton UTXO behavior).

**Change holdings**: When `sum(inputHoldings) > requestedAmount`, create a change `SimpleHolding` for the sender with the difference. Return it in `senderChangeCids`.

---

## 9. Security Invariants

| # | Invariant | Implementation Point | Status |
|---|---|---|---|
| 1 | `expectedAdmin` matches actual admin | `Rules.daml` — first check in both factory impls | ✅ |
| 2 | `requestedAt` must be in the past | `Rules.daml` — `requestedAt <= now` | ✅ |
| 3 | `executeBefore` must be in the future | `Rules.daml` — `executeBefore > now` | ✅ |
| 4 | `allocateBefore` must be in the future | `Rules.daml` — `allocateBefore > now` | ✅ |
| 5 | `allocateBefore <= settleBefore` | `Rules.daml` — `allocateBefore <= settleBefore` | ✅ |
| 6 | `amount > 0` | `Rules.daml` — both factory impls | ✅ |
| 7 | `instrumentId.admin` matches factory admin | `Rules.daml` — both factory impls | ✅ |
| 7b | `instrumentId.id ∈ supportedInstruments` | `Rules.daml` — both factory impls (Gap 7) | ✅ |
| 8 | At least one input holding | `Rules.daml` — `not (null inputHoldingCids)` | ✅ |
| 9 | All input holdings archived | `Rules.daml:archiveAndSumInputs` — archive before creating outputs | ✅ |
| 10 | Input holdings belong to sender | `Rules.daml:archiveAndSumInputs` — `hv.owner == sender` | ✅ |
| 11 | Lock holders include admin | `Rules.daml` — `lock.holders = [admin]` on all locked holding creation | ✅ |
| 12 | Lock expiry matches deadline | `Rules.daml` — `expiresAt = Some executeBefore` or `Some settleBefore` | ✅ |
| 13 | Transfer instruction signatories | `TransferInstruction.daml` — `signatory admin, transfer.sender` | ✅ |
| 14 | Allocation signatories | `Allocation.daml` — `signatory admin, allocation.transferLeg.sender` | ✅ |
| 15 | `TransferInstruction_Update` fails | `TransferInstruction.daml` — `fail "not supported"` | ✅ |
| 16 | Preapproval instrumentId matches transfer | `Preapproval.daml:TransferPreapproval_Send` — `instrumentId == transferInstrumentId` | ✅ |
| 17 | Per-input instrumentId validation | `Rules.daml:archiveAndSumInputs` — `hv.instrumentId == expectedInstrumentId` (Gap 4) | ✅ |
| 18 | `sum(inputs) >= amount` | `Rules.daml` — explicit check after `archiveAndSumInputs` (Gap 3) | ✅ |
| 19 | Expired locks accepted as inputs | `Rules.daml:archiveAndSumInputs` — `now >= expiresAt` passes (Gap 5) | ✅ |
| 20 | Unexpired locks rejected as inputs | `Rules.daml:archiveAndSumInputs` — `lockExpired` must be `True` (Gap 5) | ✅ |
| 21 | Accept validates executeBefore | `TransferInstruction.daml` — `now < transfer.executeBefore` (Gap 8) | ✅ |
| 22 | ExecuteTransfer validates settleBefore | `Allocation.daml` — `now < settlement.settleBefore` (Gap 8) | ✅ |
| 23 | Withdraw validates allocateBefore | `Allocation.daml` — `now < settlement.allocateBefore` (Gap 8) | ✅ |
| 24 | Preapproval expiry validated | `Preapproval.daml` — `now < deadline` if `expiresAt` is set | ✅ |

> **Status: 24 invariants implemented (plan originally had 15).** Invariants #16-#20 came from the gap analysis. Invariants #21-#24 are defensive deadline checks added during implementation.
>
> All 24 invariants implemented. `ensure amount > 0.0` on holding templates added as defense-in-depth (post-MVP hardening).

---

## 10. Acceptance Criteria

### Transfer Lifecycle (7 tests — `Test/Transfer.daml`)

| # | Criterion | Test Function | Status |
|---|---|---|---|
| 1 | Self-transfer: merge 2 holdings, verify balance | `test_selfTransfer` | ✅ |
| 1b | Self-transfer exact amount: no change holding created | `test_selfTransferExactAmount` | ✅ (added) |
| 2 | Direct transfer with preapproval: receiver gets holding, sender gets change | `test_directTransferWithPreapproval` | ✅ |
| 3 | Two-step transfer: factory returns Pending, creates locked holding + instruction | `test_twoStepTransferPending` | ✅ |
| 4 | Two-step accept: receiver accepts, gets holding, locked holding archived | `test_twoStepTransferAccept` | ✅ |
| 5 | Two-step reject: receiver rejects, sender gets holdings back, result is Failed | `test_twoStepTransferReject` | ✅ |
| 6 | Two-step withdraw: sender withdraws, gets holdings back | `test_twoStepTransferWithdraw` | ✅ |

### Allocation / DvP Lifecycle (5 tests — `Test/Allocation.daml`)

| # | Criterion | Test Function | Status |
|---|---|---|---|
| 8 | Allocate: creates allocation with locked holding, returns Completed | `test_allocate` | ✅ |
| 9 | ExecuteTransfer: receiver gets holding, sender gets change | `test_allocationExecuteTransfer` | ✅ |
| 10 | Cancel: sender gets holdings back | `test_allocationCancel` | ✅ |
| 11 | Withdraw: sender reclaims holdings | `test_allocationWithdraw` | ✅ |
| 12 | Multi-leg DvP: two allocations executed atomically, both receivers get holdings | `test_dvpTwoLegs` | ✅ |

### Defragmentation / Multi-Instrument (2 tests — `Test/Defragmentation.daml`)

| # | Criterion | Test Function | Status |
|---|---|---|---|
| 25 | Self-transfer merge 10 fragmented holdings into 1 | `test_selfTransferMerge10Holdings` | ✅ (added) |
| 26 | Multi-instrument transfer: USD and EUR from same factory | `test_multiInstrumentTransfer` | ✅ (added) |

### Security / Negative Tests (20 tests — `Test/Negative.daml`)

| # | Criterion | Test Function | Status |
|---|---|---|---|
| 13 | Wrong `expectedAdmin` fails | `test_wrongExpectedAdmin` | ✅ |
| 14 | `requestedAt` in future fails | `test_futureRequestedAt` | ✅ |
| 15 | `executeBefore` in past fails | `test_expiredExecuteBefore` | ✅ |
| 16 | Zero or negative amount fails | `test_nonPositiveAmount` | ✅ |
| 17 | Wrong `instrumentId` fails | `test_wrongInstrumentId` | ✅ |
| 18 | Empty `inputHoldingCids` fails | `test_emptyInputHoldings` | ✅ |
| 19 | Contention: two transfers using same holding — one fails | `test_holdingContention` | ✅ |
| 20 | Unauthorized accept (non-receiver) fails | `test_unauthorizedAccept` | ✅ |
| 21 | Accept after `executeBefore` fails | `test_expiredTransferAccept` | ✅ |
| 22 | Preapproval instrumentId mismatch (cross-instrument attack) | `test_preapprovalInstrumentIdMismatch` | ✅ (added) |
| 23 | Unexpired locked holding rejected as input | `test_unexpiredLockedHoldingInput` | ✅ (added) |
| 24 | Expired locked holding accepted as input (positive) | `test_expiredLockedHoldingInput` | ✅ (added) |
| 25 | Zero-amount `SimpleHolding` cannot be created | `test_zeroAmountHolding` | ✅ (hardening) |
| 26 | Negative-amount `SimpleHolding` cannot be created | `test_negativeAmountHolding` | ✅ (hardening) |
| 27 | Zero-amount `LockedSimpleHolding` cannot be created | `test_zeroAmountLockedHolding` | ✅ (hardening) |
| 28 | Preapproval rejects zero amount (defense-in-depth) | `test_preapprovalZeroAmount` | ✅ (hardening) |
| 29 | Owner can unlock expired locked holding | `test_ownerUnlockExpiredLock` | ✅ (hardening) |
| 30 | Owner cannot unlock unexpired locked holding | `test_ownerUnlockUnexpiredLock` | ✅ (hardening) |
| 31 | Reject succeeds after owner unlocks expired holding | `test_rejectAfterOwnerUnlock` | ✅ (hardening) |
| 32 | Withdraw succeeds after owner unlocks expired holding | `test_withdrawAfterOwnerUnlock` | ✅ (hardening) |

### Transfer Lifecycle Tests (9 tests — `Test/Transfer.daml`)

| # | Criterion | Test Function | Status |
|---|---|---|---|
| 7 | PublicFetch: returns correct factory view with admin | `test_publicFetch` | ✅ (hardening) |
| 33 | Transfer results include tx-kind metadata | `test_txKindMetadata` | ✅ (hardening) |

> **36/36 tests passing.** The plan originally specified 21 tests. Implementation added 6 tests beyond plan (1b, 22-26) for better coverage of edge cases and gap analysis. Post-MVP hardening added 9 more tests covering ensure clauses, unlock choice, expire-lock pattern, PublicFetch, and tx-kind metadata.

---

## 11. Execution Order

Sequenced by dependency. No time estimates.

### Step 1: Package Skeleton — ✅ COMPLETE
- Created `simple-token/daml.yaml` (SDK 3.4.10, target 2.1) with dependencies on all 6 `splice-api-token-*` DARs via `dars/` symlinks.
- Created `simple-token-test/daml.yaml` with dependency on `simple-token` DAR.
- Validated: `dpm build` succeeds for both packages.

### Step 2: Holding Templates — ✅ COMPLETE
- Implemented `SimpleHolding` and `LockedSimpleHolding` with `Holding` interface instances.
- Implemented `ContextUtils.daml` (replaces planned `Util.daml`) with `ToAnyValue`/`FromAnyValue` typeclasses and context lookup helpers.
- `ensure amount > 0.0` on both holding templates (post-MVP hardening).
- `LockedSimpleHolding_Unlock` choice: owner can unlock expired locks (post-MVP hardening).

### Step 3: Transfer Factory (self-transfer path) — ✅ COMPLETE
- Implemented `SimpleTokenRules` with `TransferFactory` and `AllocationFactory` interface instances.
- Implemented self-transfer path with `archiveAndSumInputs` helper.
- Validated: `test_selfTransfer`, `test_selfTransferExactAmount`.

### Step 4: Transfer Factory (two-step path) — ✅ COMPLETE
- Implemented `SimpleTransferInstruction` with accept/reject/withdraw/update.
- Implemented two-step path in transfer factory (lock + create instruction).
- Validated: `test_twoStepTransferPending`, `test_twoStepTransferAccept`, `test_twoStepTransferReject`, `test_twoStepTransferWithdraw`.

### Step 5: Transfer Factory (direct path) — ✅ COMPLETE
- Implemented `TransferPreapproval` with nonconsuming `TransferPreapproval_Send` choice.
- Implemented direct transfer path (read preapproval from `ChoiceContext`, exercise `TransferPreapproval_Send`).
- Validated: `test_directTransferWithPreapproval`.

### Step 6: Allocation Factory — ✅ COMPLETE
- Implemented `AllocationFactory` interface instance on `SimpleTokenRules`.
- Implemented `SimpleAllocation` with execute/cancel/withdraw including deadline checks.
- Validated: `test_allocate`, `test_allocationExecuteTransfer`, `test_allocationCancel`, `test_allocationWithdraw`.

### Step 7: DvP Scenario — ✅ COMPLETE
- Implemented `SimpleAllocationRequest`.
- Wrote multi-leg DvP test.
- Validated: `test_dvpTwoLegs`.

### Step 8: Security / Negative Tests — ✅ COMPLETE
- Wrote 12 negative test cases (criteria 13-24, exceeding plan's 13-21).
- Added defragmentation tests (10-holding merge, multi-instrument transfer).
- Post-MVP hardening added 10 more tests (amount invariants, unlock choice, expire-lock pattern, PublicFetch, tx-kind metadata).
- Validated: all 36 tests pass via `dpm test`.

---

## 12. Risks and Mitigations

| Risk | Mitigation | Status |
|---|---|---|
| Hidden complexity from synchronizer assignment | Document same-synchronizer prerequisite; test on LocalNet early | Open (no LocalNet testing yet) |
| UTXO fragmentation degrades UX | Implemented `test_selfTransferMerge10Holdings` proving merge works | ✅ Mitigated |
| Lock holder authorization complexity | Admin is sole lock holder; simplifies unlock authorization | ✅ Mitigated |
| Interface DAR version drift | Pinned `splice-api-token-*` 1.0.0 DARs in `dars/` symlinks; track CHANGELOG | ✅ Mitigated |
| Missing `Unlock` choice prevents manual lock release | `LockedSimpleHolding_Unlock` implemented for expired locks | ✅ Resolved |
| Gap 10 edge case: archived locked holding | Instruction choices fail if locked holding already archived | Open (see §13 Gap 10) |

---

## 13. Gap Analysis Resolution

| Gap | Description | Priority | Status | Implementation |
|---|---|---|---|---|
| 1 | `extraObservers` on `LockedSimpleHolding` | P0 | ✅ Resolved | `Holding.daml` — `extraObservers : [Party]` with `observer extraObservers` |
| 2 | Holding creation inside preapproval choice body | P0 | ✅ Resolved | `Preapproval.daml` — nonconsuming `TransferPreapproval_Send` creates holdings using receiver's signatory auth |
| 3 | Explicit `sum(inputs) >= amount` check | P1 | ✅ Resolved | `Rules.daml` — `assertMsg "Insufficient funds"` after `archiveAndSumInputs` |
| 4 | Per-input `instrumentId` validation | P1 | ✅ Resolved | `Rules.daml:archiveAndSumInputs` — `hv.instrumentId == expectedInstrumentId` per input |
| 5 | Expired lock handling for transfer inputs | P1 | ✅ Resolved | `Rules.daml:archiveAndSumInputs` — expired locks accepted (#19), unexpired rejected (#20) |
| 6 | `tx-kind` metadata annotations | P2 | ✅ Resolved | `ContextUtils.daml` — `txKindMetaKey` + `txKindMeta` helper. All result metadata annotated: `"transfer"` for transfers/allocations, `"merge-split"` for self-transfers. |
| 7 | Multi-instrument factory support | P2 | ✅ Resolved | `Rules.daml` — `supportedInstruments : [Text]` field, validated on every factory call |
| 8 | Deadline checks before input archival | P0 | ✅ Resolved | `TransferInstruction.daml`, `Allocation.daml` — deadline checks are first operation in choice bodies |
| 9 | `archiveAndSumInputs` helper pattern | P1 | ✅ Resolved | `Rules.daml` — single helper shared by transfer and allocation paths |
| 10 | `expireLockKey` pattern for withdraw/reject after lock expiry | P0 | ✅ Resolved | `TransferInstruction.daml`, `Allocation.daml` — `expireLockContextKey` context pattern. Off-ledger service passes `"expire-lock" = False` when locked holding already archived by owner. |

**Summary:** 10 of 10 gaps resolved.

---

## 13b. Open Questions Resolution

Resolved design questions matched to implementation evidence.

| Q# | Question | Resolution | Evidence |
|---|---|---|---|
| Q1 | Expired lock handling | Accept expired, reject unexpired | `Rules.daml:archiveAndSumInputs`, invariants #19/#20 |
| Q2 | Consuming vs nonconsuming preapproval | Nonconsuming | `Preapproval.daml:TransferPreapproval_Send` |
| Q3 | Expired fund cleanup | Sender self-serve unlock | `LockedSimpleHolding_Unlock` choice + expire-lock context pattern; expired locks accepted as factory inputs |
| Q4 | Per-input instrumentId | Per-input check | `Rules.daml:archiveAndSumInputs`, invariant #17 |
| Q5 | Multi-instrument | `supportedInstruments` list | `Rules.daml:SimpleTokenRules.supportedInstruments` |
| Q6 | Off-ledger auth | Out of scope | On-ledger contracts only |
| Q7 | Contention retries | Client-side | Standard Canton behavior |
| Q8 | DvP testing | `submitMulti` | `Test/Allocation.daml:test_dvpTwoLegs` |
| Q9 | Metadata DNS prefix | Splice convention | All result metadata uses `splice.lfdecentralizedtrust.org/tx-kind` for wallet interop |
| Q10 | Update choice | `fail` stub | `TransferInstruction.daml` — `fail "not supported"` |
| Q11 | Registry pause | Archive factory | No config contract needed |

## 14. Remaining TODO Items

All post-MVP hardening items (SCOPE.md §9, items 1-7) are now resolved:

| Item | Priority | Status | Notes |
|---|---|---|---|
| `expireLockKey` pattern | P0 | ✅ Done | `TransferInstruction.daml`, `Allocation.daml` — expire-lock context branching |
| `ensure amount > 0.0` on holdings | P0 | ✅ Done | `Holding.daml` — ensure clauses on both templates |
| `amount > 0.0` in `TransferPreapproval_Send` | P0 | ✅ Done | `Preapproval.daml` — defense-in-depth assertMsg |
| Contract keys | P0 | ❌ Not implementable | Daml LF 2.1 dropped contract key support; enforce at application level |
| `LockedSimpleHolding_Unlock` choice | P1 | ✅ Done | `Holding.daml` — owner unlocks expired locks |
| `test_publicFetch` | P1 | ✅ Done | `Test/Transfer.daml` — exercises both factory PublicFetch choices |
| `tx-kind` metadata annotations | P1 | ✅ Done | `ContextUtils.daml` — `txKindMeta` helper; all result metadata annotated |

## 15. Deferred (Post-MVP)

Items below are superseded where they now appear as active slices in [§17](#17-active-and-future-slices-admin-layer--cip-112).

- Burn/mint extension APIs → ✅ **done** as AL-5/AL-6 (capability-gated; [§17](#17-active-and-future-slices-admin-layer--cip-112))
- Delegation/operator model → partially active via provider-managed accounts (V2) + `BatchProcessor` role (**AL-3**)
- Multi-step allocation instructions (`AllocationInstruction` with `Update` workflow)
- Compliance policy contracts and richer settlement orchestration
- Fee schedule and holding fee decay
- Automatic holding selection (registry-side input picking)
- Merge/defragmentation utilities
- `TransferInstruction_Update` support for internal workflows
- Hold standard extension API

---

## 16. Canton Network Issues

Known Canton Network constraints relevant to this implementation.

**UTXO fragmentation:** Canton recommends keeping holdings below ~10 per user. Our self-transfer path serves as the merge mechanism (`test_selfTransferMerge10Holdings` proves 10-to-1 merge). In production, wallets that don't proactively merge will degrade performance.

**Same-synchronizer atomicity:** DvP only works when all contracts are on the same synchronizer. Cross-synchronizer transactions require the Global Synchronizer. The plan acknowledges this as risk #1 but doesn't specify which synchronizer topology to target. For institutional use cases (DTCC Treasury tokenization, Tradeweb repos), this is the critical infrastructure question.

**Contract reassignment:** When contracts move between synchronizers, they enter a "reassignment" state where they're temporarily unavailable. This is a real operational concern for multi-synchronizer deployments.

**Disclosure and privacy:** Canton's privacy model means wallets may not see contracts they need to exercise choices against. Our `extraObservers` field on `LockedSimpleHolding` addresses this for on-ledger flows by ensuring receivers and executors can see locked holdings.

**SDK and Canton version pinning:** The ecosystem is evolving rapidly (Polyglot Canton with EVM support announced late 2025, automated fee calculation via oracles proposed). Pinning SDK versions early and tracking the CHANGELOG is essential. Current pin: SDK 3.4.11, LF 2.1.

---

## 17. Active and Future Slices (Admin Layer + CIP-112)

This section is the authoritative slice backlog for everything beyond the
completed CIP-056 V1 core. It encompasses (a) the new Ownable / AccessControl /
Pausable administrative layer and (b) the remaining CIP-112 conformance work,
sequenced so the security perimeter lands and is verified **before** the V2
throughput surface depends on it.

Conventions: each slice is non-release and locally validated (`dpm build` +
`dpm test` + `scripts/verify.sh`) unless noted. Design authority for the admin
layer is [ADMIN-LAYER-PLAN.md](ADMIN-LAYER-PLAN.md); for V2 it is
[CIP-0112-EXTENSION-PLAN.md](CIP-0112-EXTENSION-PLAN.md). Status legend:
✅ done · 🔬 experimental prototype landed · 🎯 active/next · 📋 planned.

> **Hard constraint reminder (do not regress):** Daml LF 2.1 has **no contract
> keys**. No slice below may use `key` / `maintainer` / `lookupByKey` /
> `fetchByKey`. All admin/capability/pause references go by `ContractId` through
> `ChoiceContext` + explicit disclosure (the `transferPreapprovalContextKey`
> idiom). This is why the source research's keyed `TokenAdministrator` /
> `RoleCapability` / `GlobalPause` designs are **not** implementable as written —
> see [ADMIN-LAYER-PLAN.md §1](ADMIN-LAYER-PLAN.md#1-corrections-to-the-source-research-read-first).

### 17.1 Admin layer + supply slices (priority — landed first)

Status legend: ✅ implemented, tested & converged · 🎯 active/next · 📋 planned.

Architecture: the hybrid **AccessControl-as-substrate** model — `Admin` role = the
Ownable owner (`DEFAULT_ADMIN_ROLE`); Pausable = `Pauser`-gated origination control;
Mint/Burn (AL-5/AL-6) are the first capability-gated consumers.
See [ADMIN-LAYER-PLAN.md §0](ADMIN-LAYER-PLAN.md). **AL-0..AL-3 and AL-5/AL-6 are
converged** — each reached a clean `/code-review` pass (0 correctness defects),
across five fix rounds for the admin layer and two for mint/burn. **AL-4**
(formal-verification model) is the one remaining admin-layer item (🎯).

| Slice | Title | Status | Depends on | Deliverables (as built) |
|---|---|---|---|---|
| **AL-0** | Module skeleton | ✅ | — | `SimpleToken/Admin/Roles.daml` (closed `Role` sum type incl. `Admin`; `Minter`/`Burner`/`BatchProcessor` reserved for later slices), `SimpleToken/Admin/Errors.daml`. Capabilities are passed as explicit choice args (no `ChoiceContext` key needed). |
| **AL-1** | Pausable chokepoint | ✅ | AL-0 | `paused : Bool` on `SimpleTokenRules`; `assertNotPaused` injected at the five **origination** impls (transfer V1/V2, allocation V1/V2, settlement-via-factory); `Rules_SetPaused` (registry-wide `Pauser`-gated) + `Rules_GetPaused`. Pause is origination control — committed settlement/completion and recovery stay open ([ADMIN-LAYER-PLAN.md §4](ADMIN-LAYER-PLAN.md#4-pausable--origination-control-simpletokenrules)). Tests: `test_pauseBlocksTransfer`, `test_pauseBlocksAllocation`, `test_unpauseRestores`, `test_publicFetchWhilePaused`, `test_pauseAllowsRecoveryBlocksOrigination` (inv #33), `test_scopedCapabilityCannotPauseRegistry` (inv #30). |
| **AL-2** | AccessControl capabilities | ✅ | AL-0 | `SimpleToken/Admin/Capability.daml`: `RoleCapability` (with `scope : Optional InstrumentId` for least privilege) + `requireRole` (proof-by-fetch, scope-aware). Tests: `test_capabilityImpersonationFails`, `test_revokeRole`, `test_pauseRequiresPauserCap`. |
| **AL-3** | Ownable-as-Admin-role + delegated governance | ✅ | AL-2 | `SimpleToken/Admin/Authority.daml`: `TokenAdministrator` (fixed genesis root) with `Admin_IssueRole`/`Admin_RevokeRole` (root) and `Admin_DelegatedIssueRole`/`Admin_DelegatedRevokeRole` (any `Admin`-capability holder). The **`Admin` role is root-managed** — delegates issue/revoke only roles for which the single-source `delegableRole` policy is `True` (today just `Pauser`; `Admin` and reserved roles are root-only — no re-delegation, no reserved-role pre-minting), via single-sourced `mkRoleCapability`/`revokeIssuedCapability` helpers. Ownership handoff = grant/revoke the `Admin` role; **no party reassignment, no `OwnershipOffer`** (resolves the desync, C9). The `delegableRole` policy is enforced at **compile time** (`-Werror=incomplete-patterns` in `simple-token/daml.yaml`): a new `Role` without a delegation decision fails the build. Tests: `test_delegatedAdminGovernance`, `test_revokedAdminCannotDelegate`, `test_nonAdminCannotDelegateIssue`, `test_delegatedRevoke`, `test_delegatedCannotRevokeAdmin`. |
| **AL-4** | Verification model for the admin/supply layer | 🎯 | AL-1..AL-3, AL-5/6 | ✅ `ChoiceContext`/pause off-ledger semantics in [SCOPE.md §8](SCOPE.md#8-off-ledger-compatibility); ✅ `scripts/verify.sh` runs green (lint/props/verify). 🎯 **Remaining (the next admin-layer task):** grow `daml-props` rows (pause / role-authorization / mint-allowance conservation) and a `daml-verify` symbolic model with a capability relation (`admin-authorization`, `scopeAuthorizes`, capped-mint conservation) so invariants #25–#39 move from Daml-Script-covered to Z3-proved ([AUDIT.md](AUDIT.md)). These tools live in `$CANTON_TOOLS_HOME/tools/` and are extended via branches + PRs upstream — see [§17.6](#176-tooling-workspace-canton_tools_home--contribution-model). |
| **AL-5** | Capability-gated Mint | ✅ | AL-2 | `Rules_Mint` (instrument-scoped `Minter`, `whenNotPaused`) with **A3 dispatch**: direct via the recipient's `TransferPreapproval` (`TransferPreapproval_MintInto`) or a `MintProposal` (`SimpleToken/Supply.daml`) the recipient accepts. **D3** enforced cap: `RoleCapability.mintAllowance` checked + decremented (`consumeMintAllowance`, atomic on the direct path; the proposal path is unlimited-minters-only — `eCappedMinterRequiresPreapproval`); `Admin_IssueCappedMinter` for bounded minters. **D2** advisory `TotalSupply`, `(admin,instrumentId)`-anchored, service-reconciled via `TotalSupply_AdjustMinted` (not on the hot path). Tests: `test_mint*`, `test_cappedMinterAllowance`, `test_cappedMinterRequiresPreapproval`, `test_totalSupplyObservability`. See [ADMIN-LAYER-PLAN.md §11](ADMIN-LAYER-PLAN.md#11-supply-management--mint--burn-al-5--al-6). |
| **AL-6** | Capability-gated Burn | ✅ | AL-2 | **B3**: `SimpleHolding_Burn` (owner redemption, no cap, not pause-gated) + `SimpleHolding_ForcedBurn` and `LockedSimpleHolding_ForcedBurn` (instrument-scoped `Burner`-gated clawback, incl. **in-flight/locked** funds — the seized wrapper self-cleans after its deadline). `Minter`/`Burner` are enforced but root-managed (`delegableRole = False`). Tests: `test_ownerRedemptionBurn`, `test_forcedBurn*`, `test_forcedBurnSeizesInFlightTransfer`. |

> **Build/verify status:** validated on SDK 3.4.11 / DPM 1.0.17 / OpenJDK 21.
> `cd simple-token && dpm build` then `cd ../simple-token-test && dpm test` →
> **141/141 passing** (15 in `Test/Admin.daml` + 13 in `Test/MintBurn.daml`).
> `scripts/verify.sh` → **3/3** (daml-lint clean on the admin/supply source;
> daml-props; daml-verify 14/14). The `simple-token-test` package consumes the
> rebuilt `simple-token` DAR, so build the source package first. Standalone tools
> install via `scripts/setup.sh`.

### 17.2 CIP-112 V2 slices (gated by the admin layer where applicable)

Prototype slices already landed (historical evidence in CIP-0112-EXTENSION-PLAN.md):

| Slice | Title | Status |
|---|---|---|
| V2-1 | Account/Holding compile probe + V2 `Holding` interface | 🔬 |
| V2-2 | V2 `TransferInstruction` / `TransferFactory` (basic accounts) | 🔬 |
| V2-3 | V2 `AllocationInstruction` / `Allocation` | 🔬 |
| V2-4 | `SettlementFactory_SettleBatch` + `TransferEvents` (`EventLog`) | 🔬 |
| V2-5 | Compatibility matrix + negative tests (§5.4/§5.5 subset) | 🔬 |
| V2-6 | Provider-managed account authority | 🔬 |
| V2-7 | V2 `AllocationRequest` workflow | 🔬 |
| V2-8 | Iterated settlement + partial-progress (`numIterations`) | 🔬 |

Active / planned V2 slices:

| Slice | Title | Status | Depends on | Notes |
|---|---|---|---|---|
| **V2-9** | Pause the V2 origination chokepoints | ✅ | AL-1 | Done as part of AL-1: `assertNotPaused` sits on the V2 transfer/allocation factories and `SettlementFactory_SettleBatch`. By design, completion of committed flows (`Allocation_Settle`, `TransferInstruction_Accept`) is **not** gated — origination-control semantics ([ADMIN-LAYER-PLAN.md §4](ADMIN-LAYER-PLAN.md#4-pausable--origination-control-simpletokenrules), §6). |
| **V2-10** | `BatchProcessor`-gated `SettlementFactory_SettleBatch` | 📋 | AL-2 | Optional delegated matching-engine path: a capability holder drives the batch on behalf of executors. Replaces the research's invented `MatchingEngineRole`/`V2BatchTransferFactory` (corrections C5). |
| **V2-11** | V1/V2-compatible `Allocation` (`SimpleAllocationV1Compat`) | 📋 | V2-3 | CIP-0112 §5.1 dual-implementation. Open question TT-Q-002 (subset vs parallel template). |
| **V2-12** | Receipt allocations + `extraReceiptAuthorizers` | 📋 | V2-8 | **Blocked**: preview DAR omits the draft field; re-baseline with CIP authors (CIP-0112-EXTENSION-PLAN blocker evidence). |
| **V2-13** | Reduced-privacy / public-asset observer mode (§4.3.5.3) | 📋 | V2-4 | Opt-in observer expansion; default stays private-asset baseline. |
| **V2-14** | View-count baselines (§4.3.5: 28/21/25/4-view) + 3-trader batch | 📋 | V2-4 | Privacy-optimization evidence; test-only. |
| **V2-15** | Broader transfer-event reporting (direct/self/reject/withdraw) | 📋 | V2-4 | Mixed V1/V2 parsing + metadata fallback. |
| **V2-16** | CIP-0112 conformance matrix (full §5.4/§5.5) | 📋 | V2-9..V2-15 | Source-of-record gate: requires an **accepted** (non-preview) DAR set; currently pinned to `origin/token-standard-v2-daml-preview` `b91de5d4…`. |

### 17.3 Sequencing rationale

1. **AL-0 → AL-1 → AL-2 → AL-3** first: the research's central thesis is
   "establish the administrative perimeter before enabling V2 throughput." We
   honor it literally — Pausable and AccessControl land and are tested before
   any V2 slice takes a dependency on them.
2. **V2-9** is satisfied by AL-1: the V2 origination chokepoints already carry
   `assertNotPaused`. Pause is origination control; committed-settlement
   completion is intentionally ungated (the only robustly enforceable semantic in
   a keyless UTXO ledger — see [ADMIN-LAYER-PLAN.md §4](ADMIN-LAYER-PLAN.md#4-pausable--origination-control-simpletokenrules)).
3. Remaining V2 slices (V2-10..V2-16) proceed per CIP-0112-EXTENSION-PLAN, each
   gated by the now-present admin layer.
4. **Release gate (all slices):** the preview DAR source-of-record
   (M1-PF-001 / CIP-112-D-002) and AGPL→MIT extraction boundary (M1-PF-004) must
   be resolved before any conformance or public-API claim. Nothing in §17
   changes those gates.

### 17.4 Explicitly excluded (with rationale)

Per [ADMIN-LAYER-PLAN.md §10](ADMIN-LAYER-PLAN.md#10-out-of-scope-for-this-layer)
and the research corrections:

- **Keyed `GlobalPause` / `TokenAdministrator` / `RoleCapability`** — impossible on LF 2.1 (corrections C1/C2/C3).
- **On-ledger transfer of the instrument `admin` party (`OwnershipOffer` / a separate transferable `owner` field)** — the admin party is baked into every `InstrumentId`/holding, so it cannot move without re-issuing assets (correction C9). Ownership is the **`Admin` role**: handoff = grant/revoke the `Admin` capability. Operator key handoff is a Canton topology operation.
- **`V2BatchTransferFactory` (foldlA array dispatch)** — not the real CIP-112 surface (correction C5).
- **`V2OmnibusAccount`, Memo Pledge, CDP/`SecuritiesIntermediaryRole`/`LiquidatorRole`, `BridgeEscrowRole`** — stablecoin/custody/bridge use-cases; consume the `Role` primitive from sibling repos, not built here (correction C7).
- **Spoofable pause guards on committed-settlement choices** — pause is origination control; a per-asset emergency *freeze* is a separate future capability (correction: see [ADMIN-LAYER-PLAN.md §4](ADMIN-LAYER-PLAN.md#4-pausable--origination-control-simpletokenrules)).
- **Per-role admin hierarchy (`getRoleAdmin`) / two-step / timelocked issuance / on-ledger multisig** — named extension seams; the `Admin` role + Canton topology cover the MVP (see [ADMIN-LAYER-PLAN.md §5](ADMIN-LAYER-PLAN.md#5-ownable-as-admin-role-adminauthoritydaml)).

### 17.5 Current status & next steps (roadmap)

**Done & converged** (clean `/code-review`, 141/141, `verify.sh` 3/3): the full
admin layer **AL-0..AL-3**, the supply layer **AL-5/AL-6**, and V2 pause coverage
**V2-9**. The remaining `Minter`/`Burner` roles are enforced; only
`BatchProcessor` is still reserved.

Planned next steps, in recommended order — each is one self-contained slice that
keeps the suite green and is independently reviewable:

| Order | Slice | Why next / scope | Gating |
|---|---|---|---|
| 1 | **AL-4** — formal-verification model | Closes the only open admin-layer item: lift invariants #25–#39 from Daml-Script-covered to Z3-proved by giving `daml-verify` a capability relation + `daml-props` rows (pause / role-auth / capped-mint conservation). Pure verification depth, no new on-ledger surface — lowest risk, highest assurance-per-effort. **Lands in the `$CANTON_TOOLS_HOME` tool repos via branches + PRs** ([§17.6](#176-tooling-workspace-canton_tools_home--contribution-model)). | none |
| 2 | **V2-10** — `BatchProcessor`-gated `SettlementFactory_SettleBatch` | Graduates the last reserved role: a delegated matching engine drives batch settlement on behalf of executors via a `BatchProcessor` capability (the real CIP-112 surface, replacing the research's invented `V2BatchTransferFactory`). Decide whether `BatchProcessor` becomes `delegableRole`. | AL-2 |
| 3 | **V2 provider-managed clawback** (V2 hardening) | The documented V1-only gap: add forced-burn to `ProviderManagedSimpleHolding`/`ProviderManagedLockedHolding` for parity with AL-6, or keep deferred. | V2 source-of-record |
| 4 | **V2-11 … V2-16** — CIP-112 conformance | V1/V2-compatible allocations, receipt allocations, reduced-privacy observer mode, view-count baselines, broader transfer events, and the full §5.4/§5.5 conformance matrix (per [CIP-0112-EXTENSION-PLAN.md](CIP-0112-EXTENSION-PLAN.md)). | preview→accepted DAR gate |

**Release gates (unchanged, block conformance/public-API claims, not the slices
above):** the preview DAR source-of-record (M1-PF-001 / CIP-112-D-002) and the
AGPL→MIT extraction boundary (M1-PF-004) must be resolved before any CIP-112
conformance or public-API claim.

**Not yet committed:** AL-0..AL-6 + the supply layer live in the working tree
(2 new files: `SimpleToken/Supply.daml`, `Test/MintBurn.daml`; plus the new
`SimpleToken/Admin/` modules and `Test/Admin.daml`). A branch + commit is the
natural checkpoint before starting AL-4.

### 17.6 Tooling workspace (`CANTON_TOOLS_HOME`) & contribution model

The verification/library tooling that several upcoming slices **extend** lives in
a shared workspace at `~/canton-tools` (export
`CANTON_TOOLS_HOME=/Users/amar/canton-tools`; root guide:
`$CANTON_TOOLS_HOME/AGENTS.md`). Each entry below is an **independent git repo**,
not part of this repo's history.

| Path under `$CANTON_TOOLS_HOME` | Upstream | Role | Extended by |
|---|---|---|---|
| `tools/daml-lint` | `github.com/OpenZeppelin/daml-lint` | Rust static analyzer | AL-4 (optional CIP-112 dual-impl / capability-misuse detectors) |
| `tools/daml-props` | `github.com/OpenZeppelin/daml-props` | DAML property-testing library | **AL-4** (pause / role-authorization / capped-mint-conservation property rows) |
| `tools/daml-verify` | `github.com/OpenZeppelin/daml-verify` | Python/Z3 symbolic verifier | **AL-4** (add a capability relation → `admin-authorization`, `scopeAuthorizes`, capped-mint conservation proofs; the V2 proof rows in [AUDIT.md](AUDIT.md)) |
| `repos/oz-daml-contracts` | `github.com/peektism/oz-daml-contracts` | Canonical reusable DAML library scaffold | Public-API **extraction target** (gated by M1-PF-004 AGPL→MIT) |
| `canton-stablecoin` | `github.com/OpenZeppelin/canton-stablecoin` | CIP-056 + CDP/clawback reference | Pattern reference for the sibling stablecoin / forced-burn work |

**Contribution model (best practices, do not regress):**
- Treat each repo as independent — inspect/commit with `git -C <child>`; never
  assume the workspace root has git history, and never rewrite/reset/clean a
  child repo unless explicitly asked (per `$CANTON_TOOLS_HOME/AGENTS.md`).
- **Work off a feature branch and open a PR** for every tooling change — do not
  commit to a repo's default branch. Keep PRs small and single-purpose, scoped to
  one slice; land code + tests + docs together; reference the driving slice
  (e.g. "AL-4: daml-verify capability relation") in the PR; keep the repo's own
  CI/validation green (`cargo` for daml-lint, `python3 main.py`/`pytest` for
  daml-verify, `dpm build`/`dpm test` for daml-props).
- The repo-local `canton-token-template/tools/{daml-lint,daml-verify}` are
  **convenience clones** created by `scripts/setup.sh`; the **canonical source of
  truth and the target for extensions is `$CANTON_TOOLS_HOME/tools/`** → the
  upstream OZ repos. Land tool improvements upstream via PR, then re-pull locally.
- The active `daml.yaml` SDK/Java pins (3.4.11 / OpenJDK 21) and the DPM-native
  command conventions are shared workspace baselines — see
  `$CANTON_TOOLS_HOME/AGENTS.md`.
