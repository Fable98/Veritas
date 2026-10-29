# Veritas: Build Plan for Cursor

Protocol linking bounded agent spending authority (mandate), task-typed delivery evidence (evidence policy), and deterministic settlement. Research-grade, testnet only.

Place this file at the repo root. Place `veritas.mdc` in `.cursor/rules/`. Work phase by phase; each phase has copy-paste Cursor prompts at the end.

---

## 0. Decisions locked for v0 (change these and regenerate affected tasks)

These are my defaults for the three open reservation questions. Override any of them before Phase 1 starts.

| # | Question | Default decision |
|---|----------|------------------|
| D1 | Where is budget enforced? | Execution time, in an ERC-7579 hook (`preCheck`/`reserve`). Never in `validateUserOp`, because bundle validation sees stale state. SDK pre-simulates; each mandate gets its own ERC-4337 nonce key so ops from one mandate are ordered. A failed reservation reverts at execution; gas is borne by the account (paymaster optional later). |
| D2 | What counts as `reserved`? | Escrow price plus the buyer's pre-reserved challenger bond allowance. Released budget returns immediately at terminal state (`Settled`/`Refunded`/`Split`). `reserved` carries across window rollover until finalized; `committed` resets at rollover. |
| D3 | Delegation budget model | Carve-out: the parent reserves the child's budget at `delegate()`. Maximum depth 3. Shared-pool is a v1 research option. |

---

## 1. Invariant registry (the spine of the project)

Every invariant gets: an ID, a Foundry invariant test, a formal spec (Halmos or Certora), and a row in `docs/invariants.md`. No feature merges without its invariants.

| ID | Statement |
|----|-----------|
| INV-1 | For every account and mandate: `committed + reserved <= budget` at all times |
| INV-2 | A child mandate is never wider than its parent (budget, perTxCap, validity window, targets, assurance floor) |
| INV-3 | Loosening any mandate parameter requires a matured timelock; tightening is immediate |
| INV-4 | `finalizeReservation` never reverts because of Frozen/Revoked status |
| INV-5 | Escrow funds leave to the seller only via acceptance, an unchallenged elapsed window, or a resolved dispute in the seller's favour |
| INV-6 | Every non-terminal escrow state has a deadline and a defined exit; total duration <= `maxDisputeDuration` |
| INV-7 | Value conservation: sum of payouts + slashes == price + sellerBond + challengerBond for each escrow |
| INV-8 | Each reservation is finalized exactly once; escrow terminal state implies reservation closed |
| INV-9 | Recovery or upgrade paths cannot loosen a mandate without the timelock |
| INV-10 | Replay: a signed seller offer or agent intent executes at most once |

---

## 2. Repository layout

```
veritas/
├── docs/
│   ├── spec/                  # formal model: mandate, evidence policy, settlement fn
│   ├── threat-model.md
│   ├── gap-matrix.md
│   ├── invariants.md
│   └── decisions/             # ADRs, one per locked decision (D1..D3 + later)
├── packages/
│   ├── contracts/             # Foundry
│   │   ├── src/
│   │   │   ├── interfaces/    # IMandateModule, IEscrowSettlement, IVerifier, VeritasTypes
│   │   │   ├── mandate/       # MandateHook, DelegationLib, TimelockLib, RecoveryExecutor
│   │   │   ├── settlement/    # EscrowSettlement, BondManager, SettlementMath
│   │   │   ├── verification/  # SchemaSigVerifier, CommitteeVerifier
│   │   │   ├── adapters/      # erc8004/, x402/, ap2/
│   │   │   └── libraries/
│   │   ├── test/{unit,fuzz,invariant,fork,attacks}/
│   │   ├── formal/
│   │   ├── script/
│   │   └── foundry.toml
│   ├── shared/                # ABIs, zod schemas, EIP-712 types, enums
│   ├── verification/          # strategy pattern, registry, assurance lattice
│   ├── sdk/                   # policy-gate, mandate-client, escrow-client, negotiation, userop
│   ├── agent-runtime/         # LLM adapter; proposes only, never signs
│   ├── adapters/              # TS clients for x402, AP2, ERC-8004
│   └── benchmark/             # scenarios/, baseline/, harness/
├── apps/
│   ├── dashboard/
│   ├── ref-seller-api/
│   └── ref-seller-compute/
├── infra/                     # anvil, bundler, docker-compose
├── pnpm-workspace.yaml
└── .github/workflows/
```

Package scope: `@veritas/*`. EIP-712 domain name: `"Veritas"`, version `"1"`. Fix this before any signing code is written.

---

## 3. Phase plan (26 weeks, matches the synopsis)

Each task lists acceptance criteria. A task is not done until its tests and its invariant entries exist.

### Phase 1: Specification (weeks 1-3)

| Task | Deliverable | Acceptance |
|------|-------------|------------|
| 1.1 | Monorepo scaffold: pnpm workspace, Foundry, TS config, CI skeleton | `pnpm install`, `forge build`, `pnpm -r build` all pass in CI |
| 1.2 | `docs/threat-model.md` | Adversaries, assets, trust boundaries, out-of-scope list; one owner assigned |
| 1.3 | `docs/gap-matrix.md` | Rows: concurrent-budget safety, evidence-typed settlement, delegation attenuation, bounded dispute liveness. Columns: x402, AP2, ERC-8004, ERC-4337/7579, escrow protocols, agent registries, verification systems. Each cell cites a source and a verified status date |
| 1.4 | `docs/spec/` formal model | Mandate, EvidencePolicy, SettlementFunction defined as math plus pseudo-code; INV-1..10 stated formally |
| 1.5 | Freeze interfaces v0 | `IMandateModule.sol`, `IEscrowSettlement.sol`, TS `VerificationStrategy` compile; ADRs for D1-D3 written |
| 1.6 | Standards review | Pin EntryPoint version and the ERC-7579 reference implementation commit; record current status of x402, AP2, ERC-8004 |

Exit gate: only gap-matrix cells that are empty after review get claimed as contributions. Narrow the claim if needed.

### Phase 2: Core (weeks 4-10)

| Task | Deliverable | Acceptance |
|------|-------------|------------|
| 2.1 | `VeritasTypes.sol`, `SettlementMath.sol` | `computePayout` is pure; fuzz test proves INV-7 for random inputs |
| 2.2 | `MandateHook` core: createMandate, ledger, reserve, finalizeReservation, preCheck for direct payments | INV-1, INV-4, INV-8 invariant suites green at 256 runs x depth 64 minimum |
| 2.3 | `TimelockLib` plus tighten/loosen split | INV-3 suite; fuzz: random parameter diffs classified correctly as tightening or loosening |
| 2.4 | `DelegationLib` (carve-out, depth 3) | INV-2 suite; `isAttenuation` is pure and also exported to the SDK |
| 2.5 | Revocation modes Complete / Freeze / Unwind | Each mode has tests for in-flight escrows; INV-4 holds in all |
| 2.6 | `RecoveryExecutor` with timelock and guardian cancel | INV-9; test that recovery cannot swap the hook or loosen limits early |
| 2.7 | `EscrowSettlement` state machine | State-transition table test covers every (state, action) pair; invalid ones revert |
| 2.8 | `BondManager` | Challenger bond >= expected griefing gain (parameterised); INV-7 |
| 2.9 | Reservation/escrow coupling | `openEscrow` reserves in the same tx; every terminal path calls `finalizeReservation` once (INV-8) |
| 2.10 | Deterministic verifier: `SchemaSigVerifier` | Schema hash plus signature check; unit and fuzz tests |
| 2.11 | TS: `@veritas/shared`, `verification/core`, structured-api strategy | Strategy interface implemented end to end against a local seller |
| 2.12 | SDK skeleton: policy-gate, userop builder with per-mandate nonce key | Policy gate rejects out-of-mandate offers before signing |
| 2.13 | Local devnet in `infra/` | One command starts anvil, bundler, deploys contracts, seeds accounts |

### Phase 3: Verification and attacks (weeks 11-18)

| Task | Deliverable | Acceptance |
|------|-------------|------------|
| 3.1 | Deterministic-compute strategy with re-execution | Independent re-run, output hash comparison, cost per run recorded |
| 3.2 | Sampled verification | Sampling rate tied to assurance level; statistical bound documented |
| 3.3 | Escalation ladder: deterministic -> re-execution -> committee -> arbiter | Each rung invoked only on challenge; `maxDisputeDuration` = sum of rung timeouts, proven in INV-6 |
| 3.4 | `CommitteeVerifier` (quorum of signed attestations) | Disagreement handling and verifier-disappearance default (`defaultOnTimeout`) tested per verification class |
| 3.5 | Adversarial benchmark: all scenarios as Foundry attack tests | See scenario list below |
| 3.6 | Baseline implementation (plain wallet limit plus simple escrow) | Same scenarios run against it |
| 3.7 | Fix cycle | Every failed attack becomes a regression test |

Scenarios (one file each in `packages/benchmark/scenarios/` and `packages/contracts/test/attacks/`):
1. Prompt-injecting seller descriptions and offers
2. Concurrent transactions that each pass but collectively exceed budget
3. Replay and budget-drain attempts
4. Revocation during pending escrows
5. Seller-verifier collusion; griefing through false disputes
6. Verifier disappearance and indefinite lockup
7. Policy bypass via upgrade or recovery paths

### Phase 4: Integration and hardening (weeks 19-26)

| Task | Deliverable | Acceptance |
|------|-------------|------------|
| 4.1 | Adapters: x402, AP2, ERC-8004 | Each isolated behind an adapter interface; status rechecked before release |
| 4.2 | Reference app: registry, negotiation, dashboard, audit trail | Buyer agent selects a provider and reaches an agreement without manual selection |
| 4.3 | Pay-per-call API demo and compute-job demo on testnet | End-to-end: negotiate, reserve, deliver, verify, settle |
| 4.4 | Static analysis, long fuzz and invariant campaigns, formal specs | Slither clean or triaged; Halmos/Certora specs for INV-1, 5, 6, 7 |
| 4.5 | Benchmark results | Published, reproducible: attacks blocked vs baseline, cost, latency and assurance per verification class |
| 4.6 | Technical report | Falsifiable result per research area A-C, residual-trust statements explicit |

Stretch (only if all of the above is done): TEE attestation and zk-based verification classes.

---

## 4. Contract design rules (non-negotiable)

1. Stateful budget checks live in the hook at execution time, not in validation.
2. Only trusted settlement contracts may call `reserve` / `finalizeReservation`; trust changes are timelocked (loosening).
3. `openEscrow` does reserve, lock price, and pull seller bond atomically. No path creates one without the others.
4. Settlement outcome is computed by one pure function (`computePayout`); the state machine only chooses its inputs.
5. Every escrow state has a deadline; permissionless `finalize`, `claimDeliveryTimeout`, `claimResolutionTimeout` guarantee termination.
6. Checks-effects-interactions; no external calls before ledger updates; reentrancy guards on every fund-moving entry point.
7. Uint128 amounts, explicit casts with checks; no unchecked math in ledger code.
8. Events for every state change; the dashboard rebuilds history from events alone.
9. `AssuranceLevel` ordinal is a placeholder. Revisit as a partial order before Phase 3.

---

## 5. Testing strategy

- **Unit:** every external function, every revert path.
- **Fuzz:** `SettlementMath`, mandate diff classification, attenuation check.
- **Invariant (Foundry):** handlers with bounded actors (agent, buyer, seller, challenger, resolver, bundler). Actions: reserve, finalize, delegate, revoke, tighten, propose/execute loosening, open, submit, challenge, escalate, resolve, warp time. Ghost variables track expected totals.
- **Fork:** real EntryPoint plus ERC-7579 reference account with the hook installed.
- **Attacks:** one test file per benchmark scenario.
- **Formal:** Halmos or Certora specs mirroring `docs/invariants.md`.
- **CI gates:** `forge test`, invariant runs, `slither`, TS typecheck and tests, benchmark smoke run.

---

## 6. SDK and runtime rules

- The LLM proposes; deterministic code enforces. Nothing from model output reaches the signer without passing the policy gate.
- Policy gate checks, in order: offer signature and `termsHash`; mandate active; amount <= perTxCap and availableBudget; target allowlist; assurance >= `minAssurance` via `assessFeasibility`; simulation of the UserOp succeeds.
- Seller-supplied text (descriptions, offers) is untrusted data and never concatenated into instructions the signer path depends on.
- Per-mandate nonce key; SDK serializes submissions per mandate and pre-simulates against current chain state.

---

## 7. Team split

| Area | Owner focus | Primary packages |
|------|-------------|------------------|
| A | Mandate semantics, wallet contracts | `contracts/mandate`, `sdk/policy-gate` |
| B | Verification framework | `verification`, `contracts/verification` |
| C | Settlement and disputes | `contracts/settlement`, escalation ladder |
| D | Integration, benchmark, reference app | `adapters`, `benchmark`, `apps/*` |

Areas A and C co-own the formal model and finish it first. One member owns the threat model. Each member writes tests and docs for their own area.

---

## 8. Cursor prompts (copy-paste, in order)

Always attach `@docs/invariants.md`, `@docs/spec`, and the relevant interface files with `@`.

**P1: Scaffold**
> Create the Veritas monorepo from `@VERITAS_PLAN.md` section 2. pnpm workspace with `@veritas/*` packages, Foundry in `packages/contracts` (solc 0.8.28, via_ir off, fuzz runs 1000, invariant runs 256 depth 64), strict TS, GitHub Actions running forge test, slither, and pnpm -r test. Create empty `docs/` files with headings only. Do not implement any logic.

**P2: Interfaces**
> Add `IMandateModule.sol`, `IEscrowSettlement.sol`, `IVerifier.sol`, `VeritasTypes.sol` to `packages/contracts/src/interfaces` exactly as in the attached drafts. Ensure they compile against the pinned ERC-7579 reference. Generate ABIs into `@veritas/shared`. No implementations.

**P3: Mandate ledger**
> Implement `MandateHook` ledger core per `@IMandateModule.sol`: createMandate, reserve, finalizeReservation, availableBudget. Follow decisions D1-D3 in `@VERITAS_PLAN.md`. Write unit tests, then a Foundry invariant suite for INV-1, INV-4, INV-8 with a handler covering reserve/finalize/revoke/warp. Do not touch delegation or recovery yet.

**P4: Timelock and delegation**
> Implement `TimelockLib`, tighten vs loosen classification, and `DelegationLib` (carve-out, depth 3). Add fuzz tests for diff classification and `isAttenuation`, and invariant suites for INV-2, INV-3. Explain any parameter you treat as ambiguous instead of guessing.

**P5: Escrow**
> Implement `EscrowSettlement` and `SettlementMath` per `@IEscrowSettlement.sol`. Include a full (state, action) transition table test. `openEscrow` must call `reserve` and pull funds atomically; every terminal path must call `finalizeReservation` exactly once. Add invariant suites for INV-5, 6, 7, 8.

**P6: Verification layer**
> Implement `@veritas/verification` core (registry, assurance, escalation ladder) and the structured-api strategy against `@VerificationStrategy` types. Include `assessFeasibility` negotiation tests that refuse offers below `minAssurance`.

**P7: SDK policy gate**
> Implement the SDK policy gate and UserOp builder with per-mandate nonce keys. Gate order is in `@VERITAS_PLAN.md` section 6. Add tests where a malicious seller description attempts to alter the transaction; the gate must reject.

**P8: Attacks and benchmark**
> For each scenario in section 3 Phase 3, write a Foundry attack test and a benchmark harness entry running the same scenario against `benchmark/baseline`. Output results as JSON plus a markdown table. Failing attacks against Veritas must be reported, not hidden.

**P9: Hardening**
> Run slither, triage findings in `docs/security/slither-triage.md`, add Halmos specs for INV-1, 5, 6, 7, and raise invariant runs to 10k in a nightly workflow.

---

## 9. Definition of done (from the synopsis)

- Each research area has at least one falsifiable result: a proved or exhaustively checked invariant, a published attack Veritas blocks and the baseline does not, or a measured cost per verification class.
- A buyer agent selects a provider and reaches a valid agreement without manual selection.
- Out-of-mandate offers and transactions are rejected, including at contract level.
- Successful delivery settles verifiably; failed delivery follows the refund or dispute path.
- Every escrow terminates within its bound.
- A user can inspect the full transaction history.
- Veritas measurably outperforms the baseline in the adversarial benchmark. If not, the added complexity is not justified.

Document as research-grade, not audited, not for real funds.
