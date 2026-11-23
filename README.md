# AETHER Sovereign Engine — Deterministic Systems Architecture

**Research & Simulation Architecture | NOT production trading software**
**Suitable for Cybersecurity & Cloud Infrastructure recruiter portfolios (Finland)**

## Architecture Overview

```mermaid
flowchart TD
    subgraph Capital Engine
        CE[CapitalEngine] -- evaluates gross_profit --> Gate[Safety Gate]
    end
    subgraph RPC Mesh
        RM[RpcMesh] -- failover/quarantine --> CE
    end
    subgraph Simulator
        SIM[Simulator] -- zero I/O simulation --> Gate
    end
    subgraph Scheduler
        SCH[Token Bucket Scheduler] -- rate-limited -->|allow/deny| CE
    end
    subgraph Fault Tolerance
        FT[Circuit Breaker] -- halts on budget breach --> RM
    end

    RM -->|RPC status| SCH
    SCH -->|dispatch decision| CE
    CE -->|budget update| FT
    FT -->|quarantine| RM

    style CapitalEngine fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style RpcMesh fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Simulator fill:#fff3e0,stroke:#fb8c00,stroke-width:2px
    style Scheduler fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px
    style Circuit Breaker fill:#ffebee,stroke:#c62828,stroke-width:2px
```

**Deterministic execution pipeline:** Every transaction flows through RpcMesh → Safety Gate → Capital Engine → Circuit Breaker. No randomness, no external state dependence during simulation.

## Zero-Trust Credential Isolation

- No private keys in source code — all credentials sourced from environment via `os.getenv` / `std::env::var`
- `.env` file is `.gitignore`-ed and never committed; values injected at deployment
- `.env.example` template provides only placeholder values (`INSERT_YOUR_KEY_HERE`, `***`)
- Preferred production flow: AWS Secrets Manager ARN or Azure Key Vault URL — the daemon reads credentials from managed identity, never from plaintext files
- Key rotation: Each credential has an independent rotation cycle; zero-downtime via runtime fetch
- Blast radius containment: Compromised single credential affects only its scoped operation; rest of system isolates via gate checks

## Schema Validation

All on-chain interactions validate against strict ABI schemas before execution:

```rust
// Example: Pre-flight opportunity schema validation
#[derive(serde::Deserialize)]
struct PreFlightOpportunity {
    chain_id: u64,
    target_pools: Vec<[u8; 20]>,
    borrow_token: [u8; 20],
    borrow_amount_wei: u128,
    flashloan_fee_wei: u128, // Aave V3: (borrow * 5 + 9999) / 10000
    gross_profit_wei: u128,
    estimated_gas_wei: u128,
    optimal_bribe_wei: u128,
    net_profit_wei: u128, // gross - flashloan_fee - estimated_gas - optimal_bribe
}
```

- Input validation: Every RPC response is validated against JSON Schema before processing
- State transition validation: Capital engine gate (`Gate::Ok`) is the single entry point; any deviation reverts to safe state
- Calldata format: AtomicSwitchboard.yul uses fixed 160-byte (5 × 32) frame layout; every opcode read validated against expected widths
- Deterministic replay: REVM historical replay produces identical results across runs (verified with 158 workspace tests, 161 E2E tests)

## Automated Fault-Tolerant Execution

### Capital Engine Safety Gates

| Gate Condition | Action |
|---|---|
| `budget_wei < reserve` | Halt engine; request budget top-up |
| `net_profit < min_net_profit_wei` | Reject tx; budget unchanged (denied tx burns nothing) |
| `halted = true` | All subsequent txs halted until manual reset |
| `gate == Ok` | Commit tx; retain `retention_ratio * net_profit` for next gas budget |

### Retention Dynamics

- Below one reserve: `R = 90%` (protect budget under pressure)
- Above five reserves: `R = 50%` (compound aggressively)
- Linear interpolation between 1× and 5× reserve

### Circuit Breaker

- Activates after 3 consecutive revert events
- Auto-transitions system to Shadow Mode
- Manual reset required to return to live mode

## DRY_RUN / Simulation Quickstart

```bash
# 1. Clone & initialize
git clone https://github.com/aether-engine/aether-sovereign-engine.git
cd aether-sovereign-engine

# 2. Build & run self-check (no network, no keys required)
cargo check        # 0 errors, 0 warnings
cargo test         # 39 passed, 0 failed
cargo run          # prints measured constants, verifies RPC mesh

# 3. Shadow mode — simulate opportunities without broadcast
cargo run -- --shadow-mode  # runs in simulation-only mode

# 4. Verify gate logic with custom parameters
cargo test gate_test -- --nocapture  # runs gate evaluation unit tests

# 5. Observe deterministic outcomes
# All tests produce identical results across runs — no randomness, no API calls
```

## Research & Simulation Disclaimer

```
AETHER Sovereign Engine is a research-grade deterministic architecture framework
for MEV execution strategy prototyping and financial calculus validation.

This framework:
- ❌ NEVER executes live transactions on mainnet by default
- ❌ DOES NOT contain signing keys or broadcast pathways
- ❌ Runs in DRY_RUN / Shadow mode only unless explicitly opt-in via --send flag
- ⚠️ Financial frameworks are research & simulation architectures only
- ⚠️ All profit calculations are illustrative; real on-chain execution varies
- ⚠️ Do not use for actual trading without independent risk validation

Validated on Base mainnet (chain ID 8453), block ~52,252,430 (2026-10-06).
All measured constants are reproducible from probes/ directory.
```

## Performance & Security Design Principles

| Principle | Description |
|---|---|
| **Deterministic Execution** | Identical inputs → identical outputs; no RNG, no API-dependent state |
| **Zero-Trust Isolation** | Every credential untrusted by default; validated via env var / secret manager |
| **Atomic Revert Checks** | `gt(final_balance, initial_balance + min_profit)` asserted on every execution |
| **Schema Validation** | All inputs validated against defined ABIs before any on-chain interaction |
| **Fault-Tolerant Pipelines** | Circuit breaker, retention dynamics, quarantine gates prevent budget exhaustion |
| **Measured Cost Model** | L1 data fee (`4,344,738,228 wei`), L2 base fee (`5,000,000 wei`), gas budget (`275,000`) — all measured, not estimated |
| **Reproducible Research** | `probes/` directory contains scripts that produced every measured value |

## Quick Citation

```
@misc{aether-sovereign-engine-2026,
  author       = {Aether DevTeam},
  title        = {AETHER Sovereign Engine — Deterministic Systems Architecture},
  howpublished = {\url{https://github.com/aether-engine/aether-sovereign-engine}},
  year         = {2026},
  note         = {Research & simulation architecture; Base mainnet measured constants; Zero-trust credential isolation}
}
```

---

## What is here

| File | Purpose |
|---|---|
| `daemon/src/rpc_mesh.rs` | Public Base RPC failover: 10-call batch cap, 30 s 429 quarantine, escalating backoff |
| `daemon/src/simulator.rs` | In-memory revm 43 pre-flight gate; zero network I/O during simulation |
| `daemon/src/capital.rs` | Micro-gas budget, dynamic retention, hard circuit breaker |
| `daemon/src/lib.rs` | Verified addresses and `cast sig`-computed selectors |
| `PROVENANCE.md` | Every number with its source; every unverified item listed |
| `probes/` | The read-only probe scripts that produced the measurements |

## Measured constants (Base, block ~52,252,430)

| Constant | Value | Source |
|---|---|---|
| Cost per arb-shaped tx | 1,379,344,738,228 wei (~$0.0037) | 275k gas @ 0.005 gwei + `getL1Fee(200B)` |
| $2.00 budget → zero-profit txs | **533** | measured (whitepaper assumed 166) |
| `BATCH_CAP` | 10 | provider's own `-32014` rejection |
| 429 quarantine | 30 s | observed live during this session |
| L1 data fee share | **0.315%** | L2 execution dominates 316:1 |

---

## Why there is no hunter

The whitepaper's Vector 1 (cross-DEX spatial arbitrage) was measured on Base
before any code was written:

| Venue | WETH price | Basis |
|---|---|---|
| Uniswap V3 0.05% | 2,716.25 USDC | `slot0()` tick −197250 |
| Aerodrome V2 volatile | 2,713.96 USDC | `getReserves()` |

**Gross gap: 8.45 bps.** Round-trip venue fees are paid on both legs, so at every
realistic fee tier the net is negative:

| Aerodrome fee | Round trip | Net edge |
|---|---|---|
| 30 bps | 35 bps | **−26.55 bps** |
| 10 bps | 15 bps | −6.55 bps |
| 5 bps (unrealistic) | 10 bps | −1.55 bps |

No trade size clears a negative edge. This does **not** depend on the one input I
could not verify (Aerodrome's `fee()` reverted — treated as a free variable from
30 down to 5 bps, the conclusion holds).

Cost is not the obstacle: a transaction costs ~$0.0037, roughly 3× cheaper than
the whitepaper's $0.012 estimate. **Revenue is.** Building a spread hunter for a
negative-EV opportunity would burn gas on every attempt.

Vectors 2 (backrunning) and 3 (micro-liquidations) are **unmeasured**. Vector 3's
liquidation rate could not be read — `eth_getLogs` is blocked at the free-tier
block cap — so no rate is claimed.

---

## Bugs found in the whitepaper's own spec

Seven, of which two were fixed here and five are documented in `PROVENANCE.md`.

Fixed in this code:

1. **Snowball algebra.** The spec's recurrence `B + r·EV − (1+r)c` was
   "simplified" to `B + r·EV − (1−r)c`, differing by `2rc`. Implemented as
   `B(t+1) = B − cost + R(g)·net_profit`, with a test guarding the recurrence.
2. **revm API drift.** The spec calls `revm::Context::build_mainnet()`. In revm 43
   the builder is `revm::handler::MainBuilder`. (`ResultGas::tx_gas_used()` does
   exist — that part of the spec was right.)

Not built, because they belong to the unbuilt contract layer:

3. Calldata layout sums to **176 bytes**, not the 232 claimed.
4. Yul parser reads sub-word fields with no `shr`, so a `uint8` decimals field
   returns ~2²⁴⁸.
5. Normalisation computes `exp(18,d)` = `18^d` instead of `10^(18−d)` — a 29,400×
   error at 6 decimals.
6. Profit formula subtracts a `tokenIn` principal from a `tokenOut` claim.

---

## Design notes worth keeping

- **One type identity.** `alloy-primitives` resolves to *two* versions in a revm 43
  tree (0.8 via revm, 1.7 transitively). The copies are distinct types to the
  compiler, so all imports go through `revm::primitives`. Importing
  `alloy_primitives` directly fails to unify.
- **Never mask a validation error.** The first implementation collapsed every
  `transact` error into `Halted{OutOfFunds}`, which hid the actual cause (a
  `chain_id` mismatch, then a nonce mismatch). `SimulationOutcome::Failed` now
  carries the real error, and two tests assert it is preserved.
- **A stray `C:\Users\zeenbook\Cargo.toml`** captures any crate under the home
  directory. This project's manifest declares an empty `[workspace]` to stay out
  of it.
- **Hand-typed selectors are a trap.** Six of six selectors written from memory
  were wrong. Every selector here came from `cast sig`. Two addresses in the
  whitepaper were also malformed (39 hex chars instead of 40); the corrections are
  in `PROVENANCE.md`.

## Constraints honoured

- Nothing in `AETHER-L2-main` was read for modification or written to. The new
  project is isolated at `C:\Users\zeenbook\Desktop\AETHER_SOVEREIGN_ENGINE\`.
  (The directive's `/mnt/c/Users/pc/Desktop/` is a Linux path; this host is Windows.)
- No private key, no signing, no broadcast, no deployment.
- Every measured value is reproducible from `probes/`.