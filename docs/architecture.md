# Princeps architecture

## Subsystems

Princeps is a single Rust binary composed of two cleanly-separated halves:

- **Consensus layer (CL)** — Malachite BFT, wired through `crates/consensus`. Owns leader election, voting, view changes, finality.
- **Execution layer (EL)** — Reth as a library, wired through `crates/evm`. Owns state, EVM execution, payload building, mempool.

Plus four pure state-machine subsystems that the EL composes:

- **CLOB** (`crates/clob`) — orderbook matching engine. Pure, deterministic, replayable.
- **Settlement** (`crates/funding`, `crates/oracle`, `crates/liquidation`) — funding rates, mark prices, liquidations. `funding` (Stage 8b), `liquidation` (10a margin math → 10b insurance fund → 10c multi-account scanner → 10d ADL), and `oracle` (11 aggregation → 11b signed observations) are all complete; each runs deterministically per block via the integration coordinator (Stages 14a–15e).
- **Vault** (`crates/vault`) — protocol-native vault primitive for strategy products. Shipped at Stage 12 (share-based collateral pooling); marked-to-market per block (Stage 14a).
- **Clearing** (`crates/clearing`) — per-account position bookkeeping. `apply_fill(account, price, qty, side)` updates `(position_size, avg_entry)` and returns realized PnL across the open/increase/partial-close/flip cases (Stage 16a). The bridge owns the `HashMap<AccountId, Account>` and routes every CLOB fill through `apply_fill` (Stage 16b); accounts are produced by real fills (Stage 17a) and persisted in the bridge snapshot.
- **Lending** (`crates/lending`) — per-market state (reserves, total borrowed/supplied, kinked IRM params, borrow/supply indices) and per-account position state (collateral + Aave-style scaled debt). Pure compute primitives for IRM, health factor, and per-block interest accrual (Stages 19a–19e). The bridge owns `BTreeMap<MarketId, Market>` + `BTreeMap<(AccountId, MarketId), Position>` and mutates them through `lending_*` methods + six EVM precompiles at `0x0c1f`–`0x0c24` (Stages 20–22b). The per-block `lending_tick` advances `borrow_index` and routes interest into reserves; `scan_lending_health` flags positions with HF < 1.0.
- **Integration coordinator** (`crates/node` — `PrincepsNode::tick`) — composes the pure subsystems above into one deterministic per-block routine: oracle refresh → liquidation scan → ADL absorption → vault mark-to-market → funding settlement. Driven from `LiveRethEvmBridge`'s commit path in `bin/princeps reth-devnet` (Stages 14a–15e); produces a `TickReport` whose fields the bridge applies back to per-account state. The lending tick is currently driven separately from the bridge — Stage 23 will unify them.

### Collateral flow

Collateral enters and leaves accounts through `deposit`/`withdraw`, exposed two ways (Stages 17b–17e):

- **Bridge methods** — `LiveRethEvmBridge::deposit(account, amount: i64)` (signed, no balance check) and `withdraw(account, amount: u64) -> Option<Notional>` (balance-checked). Used by `bin/princeps` to seed demo collateral.
- **EVM precompiles** — `princeps_deposit` at `0x…0c1d` and `princeps_withdraw` at `0x…0c1e`, alongside the two CLOB precompiles (`clob_read_best_bid` at `0x…0c1b`, `clob_place_order` at `0x…0c1c`). They mutate the same `Arc<Mutex<HashMap<AccountId, Account>>>` the bridge owns, shared via the precompile module's install globals — so an EVM-side deposit and a Rust-side bridge deposit are the same state change.

### Lending flow (Stages 19–22b)

Lending uses the same shared-Arc pattern: the bridge owns `Arc<Mutex<BTreeMap<MarketId, Market>>>` and `Arc<Mutex<BTreeMap<(AccountId, MarketId), Position>>>`. Six precompiles at `0x…0c1f`–`0x…0c24` (`deposit_collateral`, `borrow`, `repay`, `withdraw_collateral`, `health`, `liquidate`) mutate the same maps the bridge methods do — same equivalence as the perp deposit/withdraw precompiles. Borrow/withdraw/liquidate enforce post-operation health factor >= 1.0 via simulate-then-commit (clone position → simulate → check health → conditionally commit). The `princeps_lending_health` precompile is staticcall-safe (no mutation).

Known v0 limitations:

- Precompile mutations land regardless of whether the calling EVM transaction reverts (same caveat as `clob_place_order` / `princeps_deposit`); lending state is not yet covered by `snapshot_bridge_state` / `restore_bridge_state` revert-guard.
- `withdraw`'s balance check is against raw collateral rather than free-after-margin (the lending-side withdraw IS health-checked; the perp-side `princeps_withdraw` is the one that uses the avg-entry rule).
- Prices for the price-sensitive precompiles (`borrow` / `withdraw_collateral` / `health` / `liquidate`) are passed in calldata by the EVM caller. v1 will install an oracle global so precompiles read prices directly.

## The CL/EL contract

The boundary between consensus and execution is exactly four messages, defined as the `ConsensusBridge` trait in `crates/consensus/src/bridge.rs`:

| Direction | Message | Promise |
| :--- | :--- | :--- |
| CL → EL | `build_payload(parent, attrs)` | "Build me a candidate block on top of `parent`." |
| EL → CL | `payload_ready(block)` | "Here is the assembled block." |
| CL → EL | `validate_payload(block)` | "Would this block execute cleanly?" |
| CL → EL | `commit(block_hash)` | "Finalize this block. Update fork-choice." |

Every interaction between CL and EL flows through these four. Anything else is a contract leak.

## The pure / I/O split

| Crate group | I/O? | Tested how |
| :--- | :--- | :--- |
| `types`, `codec`, `clob`, `funding`, `liquidation`, `vault`, `oracle`, `clearing`, `lending` | No | Unit tests + proptest, microseconds per case |
| `evm`, `consensus`, `node` | Yes | Integration tests, devnet replay |

The pure crates do not depend on tokio, networking, disk, or system time. This is enforced by `unsafe_code = "forbid"` plus dependency-policy review.

## Determinism rules

State changes happen exclusively inside the pure crates. The I/O crates may only:

1. Receive an event from the network or disk.
2. Call into the pure crates with that event as input.
3. Persist or broadcast the result.

The pure crates never call `SystemTime::now`, `HashMap` iteration order, `rand`, or any operation whose output depends on host state. Determinism is the only reason multiple validators converge on the same state root; one violation forks the chain.

## ADRs

Significant design decisions are recorded as ADRs under `docs/adr/`. Each ADR is dated, stable, and never edited after acceptance — supersede with a new ADR instead.
