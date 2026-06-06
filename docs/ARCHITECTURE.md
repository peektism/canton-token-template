# Access Control Architecture — Hybrid Substrate vs. Decoupled Primitives

> Why the admin layer is shaped the way it is, where it should and should not
> stay coupled, and how we explore the decoupled (Solidity-mirroring) shape in
> parallel without throwing away the tested baseline.
>
> This document answers the review questions raised on the WIP access-control
> work (placement, decoupling, package size) and weighs the alternatives
> honestly. It assumes [ADMIN-LAYER-PLAN.md](ADMIN-LAYER-PLAN.md) for the
> per-slice detail and [PLAN.md §17](PLAN.md) for sequencing.

---

## 0. TL;DR

1. **The review is right about the library layer.** The *reusable* access-control
   primitives (RBAC, Ownable, Pausable) belong in `oz-daml-contracts` as
   **independent, token-agnostic packages**, importable one at a time. They should
   not live inside `canton-token-template`, and they should not be welded into one
   bundle. The current in-token modules are an **instantiation / test harness** that
   proved the behaviour and the invariants — *the foundations will change* on the way
   to the library.
2. **The convergence ("hybrid") is right only at the *token* layer.** A specific
   token needs *one coherent authority model* — a single immutable admin party
   threaded through every `InstrumentId` — and at that layer "Admin = owner" and
   "pause is a `Pauser`-gated chokepoint" are a deliberate **composition** of the
   primitives, not a property of the primitives themselves.
3. **DAML changes the *shape* of the decomposition, not the *intent*.** OpenZeppelin's
   Solidity value proposition — small, decoupled, independently importable modules —
   is exactly the target. But DAML has no inheritance, no storage mappings, no
   `msg.sender`, no contract keys (LF 2.1), and **monomorphic templates**. So
   "decoupled like the Solidity repo" maps to *independent DAML packages with minimal
   templates*, and "generic" forces a concrete, important tradeoff (§5) that Solidity
   sidesteps with `bytes32` role ids.
4. **Plan:** keep the tested in-token hybrid as the working baseline; build the
   decoupled library in `oz-daml-contracts` **on a branch**, in parallel, to explore
   the genericity tradeoff in isolation; have the token consume it via a
   branch-pinned `data-dependency`; compare empirically; converge on the layered
   design (Option 3 in §4). This is slice **AL-7** ([PLAN.md §17.5](PLAN.md)).

---

## 1. The review questions, answered directly

> **Q1. Is the current code the final form, or are the foundations going to change?**

**The foundations change.** What exists in `simple-token/daml/SimpleToken/Admin/`
(`Roles`, `Capability`, `Authority`, `Errors`) is a *correct, tested instantiation*
used to nail down the authorization semantics and lock in invariants #25–#39 — it is
not the shipped library. The generic substrate is extracted to `oz-daml-contracts`
(slice AL-7); the token keeps only its *composition* of that substrate.

> **Q2. Access control is a generic library, not tied to tokens — it should live in
> `canton-contracts`, not `canton-token`.**

**Agreed.** The capability/role mechanism (`RoleCapability`, `requireRole`,
`scopeAuthorizes`, the role-admin/delegation policy) has no token-specific content and
should be a standalone package in `oz-daml-contracts`. The token-template should
*depend on* it, exactly as a Solidity token `import`s `@openzeppelin/contracts/access/AccessControl.sol`.
The one token-specific binding — the `Optional InstrumentId` scope — is a *type
parameter of the consumer's instantiation*, not part of the generic core (§5).

> **Q3. RBAC, Ownable, and Pausable are grouped in one system; this deviates from the
> Solidity repo and inflates package size for projects that want only one. Each should
> be independent and minimal.**

**Agreed at the library layer, with one DAML nuance.** In the library they become
**three independent packages** (`OpenZeppelin.AccessControl`, `OpenZeppelin.Ownable`,
`OpenZeppelin.Pausable`); a consumer that wants only Pausable depends only on Pausable.
The nuance: in DAML, Ownable is most naturally expressed *as a one-role specialization*
of the capability mechanism, and the hybrid leaned into that to avoid duplicating the
authority-proof machinery. The decoupled design instead **mirrors Solidity**: `Ownable`
is its own minimal template (a single `owner` party + a transfer choice) that does **not**
depend on `AccessControl`, and a consumer who wants "owner == DEFAULT_ADMIN_ROLE" composes
the two itself. Independence wins for the library; the convergence is a *token* choice (§3, §4).

> **Q4. Do Canton contracts have a size / complexity limit we should be aware of?**

**Not an EIP-170-style per-contract code cap — the binding limits are at the
transaction/value layer.** See §6 for the numbers (4 MB default inbound message size,
configurable, 2 GB gRPC hard ceiling) and what "package size" actually costs in Canton
(dependency-graph vetting and DAR distribution, not a deploy-time byte limit). The
practical upshot *supports* decoupling: a consumer of one small package carries a smaller
dependency footprint.

---

## 2. Why DAML changes the shape, not the intent

The Solidity OZ modules are *abstract base contracts* you inherit; roles are storage
`mapping`s; gating is a `modifier` reading `msg.sender`; reuse is inheritance; and each
deployed contract has its own 24 KB EIP-170 code limit, so keeping modules small and
separate is partly a *deploy-size* discipline. **None of those mechanisms exist in DAML.**

| Concern | Solidity (EVM) | DAML (Canton, LF 2.1) | Consequence for decomposition |
|---|---|---|---|
| Code reuse | `contract Token is AccessControl` (inheritance) | `data-dependencies:` on another **package/DAR**; no inheritance | "Decoupled module" = **separate package**, not a base class |
| State | mutable storage `mapping(bytes32 => …)` | immutable **contracts** on the ledger (UTXO) | A "role grant" is a `RoleCapability` *contract*, not a map entry |
| Caller identity | `msg.sender` | **signatories / controllers**; authority is who signed | Authorization is a *capability you hold + co-sign*, not an address check |
| Role lookup | `hasRole(role, addr)` view over storage | `fetch` a capability by `ContractId` + disclosure; **no contract keys / no global lookup** | The caller *presents* its credential; the choice validates it |
| Polymorphism | generics-ish via `bytes32` ids | **templates are monomorphic** — a template field cannot be a type variable | Generic RBAC must pick a concrete role representation (§5) |
| Deploy unit / size | one contract, ≤24 KB bytecode (EIP-170) | upload a **DAR package**; templates aren't deployed individually | No per-contract byte cap; cost is dependency footprint (§6) |

So the *intent* the review states — small, independent, minimal-overhead, reusable —
is exactly correct and achievable. The *artifact* is a set of DAML packages with minimal
templates, validated by `requireRole`-style proof-by-fetch, rather than a tree of
inherited abstract contracts. Designing it to "look like the Solidity files" line-for-line
would be a category error; designing it to deliver the *same decoupling guarantees* is the
goal.

---

## 3. The two layers (the crux of the disagreement)

The single most useful distinction is **library primitive vs. application instantiation**:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Layer A — GENERIC PRIMITIVES   (oz-daml-contracts, decoupled)         │
│                                                                       │
│   OpenZeppelin.AccessControl   OpenZeppelin.Ownable   OpenZeppelin.   │
│   (roles, requireRole,         (owner + transfer)     Pausable        │
│    role-admin/delegation)      — independent —        (flag + guard)  │
│        ▲                            ▲                      ▲          │
└────────┼────────────────────────────┼──────────────────────┼─────────┘
         │  data-dependencies (a consumer picks only what it needs)
┌────────┼────────────────────────────┼──────────────────────┼─────────┐
│ Layer B — TOKEN INSTANTIATION   (canton-token-template)               │
│                                                                       │
│   The hybrid *composition*: Admin role := owner (DEFAULT_ADMIN_ROLE), │
│   pause := Pauser-gated origination control, one immutable genesis    │
│   admin baked into every InstrumentId, scope := Optional InstrumentId │
└───────────────────────────────────────────────────────────────────────┘
```

**Why the convergence is correct at Layer B and wrong at Layer A.**

- A **token** has exactly one immutable issuer party, baked into every `InstrumentId`
  and every holding. On a keyless ledger that party *cannot move* without re-issuing all
  assets (see [PLAN.md §17.4](PLAN.md), correction C9). So the token *needs* a single
  coherent authority root, and it is both natural and efficient to make the `Admin` role
  that root (the Ownable owner / `DEFAULT_ADMIN_ROLE`) and to gate pause on a `Pauser`
  capability drawn from the *same* substrate. This convergence buys: one authority-proof
  path, atomic "who-can-do-what" reasoning, and no duplicated machinery — for *this token*.
- A **library** has no such single-root requirement. A project that wants only a pause
  switch, or only an owner, should not inherit the token's coherence assumptions or its
  role set (`Minter`/`Burner`/`BatchProcessor` mean nothing to a non-token). Bundling
  there is exactly the overhead the review flags.

The bug in the current code is **not** that the hybrid exists — it is that the *generic
substrate lives at Layer B*, where it cannot be reused. AL-7 moves the substrate to Layer A
and leaves only the composition behind.

---

## 4. Options weighed

### Option 1 — Fully decoupled, independent packages (mirror the Solidity repo)

Three packages, each minimal and self-contained, no cross-dependency. `Ownable` is a bare
`owner`+transfer template; `AccessControl` is roles+`requireRole`; `Pausable` is flag+guard.

- **Pros:** maximal reuse; smallest dependency footprint per consumer; closest to OZ
  Solidity ergonomics; each module independently auditable; matches the agreed direction.
- **Cons:** to be *generic*, `AccessControl` must use concrete role ids (Text/`bytes32`-style),
  losing the closed-sum compile-time exhaustiveness the token enjoys (§5); some logic
  (signatory-based authority proof) is re-described per module; a consumer wanting
  "owner == admin" wires it by hand.

### Option 2 — Converged substrate (today's in-token hybrid)

One admin layer where Ownable and Pausable are roles over a shared capability substrate.

- **Pros:** one authority model, no duplicated proof machinery, type-safe closed `Role`
  sum (typo-proof, lintable, `-Werror=incomplete-patterns`), atomic reasoning; ideal *for a
  single token*.
- **Cons:** not reusable as-is; a non-token consumer pays for roles it never uses; lives in
  the wrong repo; deviates from the OZ decoupling promise; "package size for projects that
  want only one functionality" is exactly the cost the review names.

### Option 3 — Layered: decoupled primitives + thin token composition  ✅ recommended

`oz-daml-contracts` ships the three **independent** packages (Option 1's library). The
token-template depends on them and contributes a *thin composition module* that binds
`Admin → owner`, gates pause on `Pauser`, and supplies the token's concrete role set and
`InstrumentId` scope.

- **Pros:** delivers the decoupling/reuse the review wants (Layer A) **and** keeps the
  coherent, type-safe token authority model the hybrid proved valuable (Layer B);
  the convergence becomes an explicit, documented *choice of the consumer*, not a library
  constraint; other use-cases (stablecoin, custody, bridge — the C7 siblings) compose the
  same primitives with their *own* admins and role sets.
- **Cons:** two layers to maintain; the token must choose how to reconcile generic
  (Text-role) AccessControl with its type-safe closed `Role` (a thin typed wrapper, §5);
  slightly more upfront structure than Option 2.

Option 3 is the synthesis: **the review's decoupling at Layer A is non-negotiable; the
hybrid survives only as Layer B wiring.**

---

## 5. The genericity tradeoff DAML forces (the deepest point)

This is the part that has no Solidity analogue and most shapes the library design.

**DAML templates are monomorphic.** You cannot write a polymorphic template whose stored
field is a type variable:

```daml
-- ILLEGAL: a template field cannot be a type parameter.
template RoleCapability r
  with role : r
  ...
```

A serializable template field must be a concrete type. So a *generic* AccessControl
package — one that does not know `Minter`/`Burner`/`Pauser` ahead of time — has only two
honest options for representing "which role":

1. **Stringly-typed role ids** (`role : Text`, the direct analogue of Solidity's
   `bytes32 public constant MINTER_ROLE`). Fully generic and reusable across any contract.
   **Cost:** loses the compile-time exhaustiveness, typo-proofing, and lint coverage the
   token currently gets from a closed `Role` sum type — a role typo becomes a silent
   runtime mismatch, exactly as in Solidity.
2. **Per-consumer specialization** — the library ships the *pattern* (a function/typeclass
   surface and a documented template skeleton) and each consumer declares its own
   `RoleCapability` with its own closed `Role` sum. **Cost:** each consumer re-declares a
   small template; the "library" is partly a convention, not a single importable DAR.

Solidity never confronts this because `bytes32` ids + inheritance give it generic *and*
ergonomic. DAML makes us choose. The recommendation for `oz-daml-contracts`:

- Ship **generic `OpenZeppelin.AccessControl` with `Text` role ids** (option 1 above) as
  the importable, reusable default — this is what "a generic library, reusable by other
  contracts easily" actually requires in DAML, and it mirrors `bytes32`.
- Provide a **documented typed-wrapper pattern** so a consumer that wants exhaustiveness
  (like this token) maps its closed `Role` sum to/from the `Text` ids at its boundary —
  keeping type-safety in its own code while depending on the generic core.
- `Ownable` and `Pausable` carry no role type at all, so they are trivially generic and
  independent.

This tradeoff — *generic-and-reusable (Text ids)* vs *type-safe-and-exhaustive (closed sum)* —
is precisely what AL-7 builds **in parallel and in isolation** to evaluate empirically,
rather than asserting on paper. The token keeps its closed-sum baseline working while the
generic library is trialled beside it.

---

## 6. Canton size & complexity limits (Q4, with sources)

**There is no EIP-170 equivalent.** DAML templates are not individually deployed
bytecode; you upload a **DAR package** to a participant, and templates are instantiated as
ledger contracts. So "a contract that is too big to deploy" is not the failure mode.

The limits that *do* bind, in order of practical relevance:

1. **Transaction / message size.** Canton's default maximum inbound message size is **4 MB**
   (`max-inbound-message-size`, configurable on the participant ledger-API, admin-API, and
   domain). The synchronizer also enforces `maxRequestSize` (a dynamic protocol parameter,
   v4+) at the sequencer. gRPC imposes an absolute **2 GB** ceiling per message. Canton
   **rejects an over-large transaction with a low-level error**, so batch/fan-out choices —
   not module structure — are what risk hitting this. This bounds *how much one command
   creates/consumes/discloses*, including disclosed `RoleCapability` contracts.
2. **Daml-LF value limits.** The engine bounds value nesting depth and total serialized
   value size; deeply nested or very large records/lists in a contract argument are the
   concern, again at *runtime*, not at package build.
3. **Package / DAR footprint and the dependency graph.** Every package a consumer depends
   on must be **vetted across participants** and distributed; a larger dependency graph is
   topology and operational overhead, and a bigger DAR is slower to upload/recompile.
   This is where the review's "package size for projects that only want one functionality"
   lands in Canton: **not a deploy-time cap, but a real dependency-footprint cost** — and it
   is a direct argument *for* decoupling, since a Pausable-only consumer should vet and
   ship only Pausable.

Net: the size concern is valid but manifests as *transaction-size discipline* + *dependency
footprint*, not a per-contract byte limit. Decoupling helps the second; nothing about the
admin layer's logic stresses the first today (capabilities are tiny single-record contracts).

*Sources:* [Canton forum — Max payload size (4 MB default, 2 GB gRPC ceiling)](https://forum.canton.network/t/max-payload-size-on-canton/3751);
[Daml — Manage Synchronization Domains (`maxRequestSize` dynamic parameter)](https://docs.daml.com/canton/usermanual/manage_domains.html);
[Daml — Scaling and Performance](https://docs.daml.com/canton/usermanual/performance.html). *(Numbers are
version- and config-dependent — verify against the deployment's Canton/protocol version.)*

---

## 7. Multi-admin & extensibility — why this architecture earns its keep

The decoupled design is what makes "different use-cases involving different admins"
first-class:

- **Across deployments:** the admin party is a *parameter*, baked into each deployment's
  `InstrumentId`. Two tokens, two stablecoins, a custody vault — each instantiates the same
  primitives with its **own** admin root. The library carries zero hardcoded authority.
- **Within a deployment:** `scope : Optional InstrumentId` gives per-instrument least
  privilege (`None` = registry-wide, `Some iid` = one instrument), proven by daml-verify
  A4/A5. A generic `AccessControl` keeps `scope` as the consumer's instantiation type, so a
  non-token consumer scopes by *its* resource type instead.
- **Across the C7 siblings:** stablecoin / custody / bridge components (explicitly *not*
  built here — [PLAN.md §17.4](PLAN.md)) consume `AccessControl` / `Ownable` / `Pausable`
  independently, with their own role sets, rather than importing a token. That is only
  possible once the substrate is a decoupled Layer-A package — which is the whole point of
  the review and of AL-7.

So the hybrid is *not* a bet against extensibility; it is the Layer-B convergence that one
token needs. Extensibility comes from pushing the substrate **down** to a decoupled library,
not from bundling more into the token.

---

## 8. Recommendation & the parallel build (slice AL-7)

1. **Keep the in-token hybrid as the green, verified baseline.** It is tested
   (141/141 + A1–A8 proofs + AdminLayer property rows) and encodes the correct *token*
   authority semantics. Do not regress it while exploring.
2. **Build the decoupled library in `oz-daml-contracts` on a feature branch**, as three
   independent packages (`AccessControl` with Text role ids, `Ownable`, `Pausable`),
   following Daml package conventions and the §17.6 contribution model.
3. **Explore the genericity tradeoff in isolation** (§5): generic Text-role core +
   documented typed-wrapper pattern; measure ergonomics, footprint, and audit surface
   against the closed-sum baseline.
4. **Have `canton-token-template` consume the library via a branch-pinned
   `data-dependency`** and reduce its in-token modules to the thin Layer-B composition
   (Option 3). Compare side-by-side before committing to the cutover.
5. **Develop and test forward on branches — do not block on external merges.** The same
   discipline applies to the daml-verify / daml-props tool PRs: pin local verification to
   the PR branches now, evaluate, and only fold in once we are satisfied. Merging upstream
   is downstream of *our* validation, not a precondition for it.

The actionable form of this section lives in [PLAN.md §17.5 / slice AL-7](PLAN.md).
```
