# Princeps

**The DeFi prime broker L1 on Reth.**

Lending → options → structured products → institutional rails, on a single shared risk engine.

> *Princeps* (Latin: "first, prime, principal") — pronounced **PRIN-seps**. The title Augustus adopted when founding the Roman Principate. Now: the prime broker stack for on-chain finance.

---

## What it is

Princeps is an open-source, Reth-based L1 designed to be the DeFi-native counterpart of a TradFi prime broker. Where Hyperliquid reinvented perps by embedding the matching engine into the L1, and Tempo reinvented payments by embedding FX and auto-swap, Princeps reinvents the primitives that perps don't cover:

- **Lending** — deterministic sub-second liquidations as state transitions (not keeper auctions), portfolio margin, native flash loans
- **Options** — Black-Scholes / IV surfaces / Greeks as precompiles, auto-exercise at expiry block, vol surface as canonical state
- **Structured products** — native vault/strategy composition over lending + options, continuous NAV, no keepers, native fee waterfalls
- **Institutional rails** — KYC overlay for compliance-required tenants, native fund-admin primitives (NAV strip, attribution)

All four layers share one risk engine — so a strategy can sell a covered call, post the underlying as collateral, borrow against it, and rebalance — atomically, in a single block. That's the prime broker thesis.

## Positioning

**The L1 that's everything HL isn't.** Princeps explicitly does not build perps; HL owns that. Princeps is the lending / options / structured-products / institutional-rails complement.

One-line pitch: *"If HL is the on-chain CME, Princeps is the on-chain prime broker."*

## Roadmap

| Version | Scope | Target |
|---|---|---|
| **v0** | Lending — single asset pair (USDC collateral, ETH borrow), deterministic liquidations, portfolio margin | Q3 2026 |
| **v1** | Options — orderbook, vol surface, Greeks precompiles, cross-margin with lending | Q4 2026 / Q1 2027 |
| **v2** | Structured products — native vault composition, continuous NAV, Tempo settlement integration | Q2 2027 |
| **v3** | Institutional rails — KYC overlay (per-tenant opt-in), fund-admin primitives, decentralized sequencer | H2 2027 |

## Architectural decisions (locked)

Seven foundational decisions, locked for the platform lifecycle:

| ADR | Decision | Note |
|---|---|---|
| [001](./docs/adr/001-consensus-malachite.md) | Consensus: **Malachite** (Informal Systems) BFT | Instant finality, no reorgs |
| [002](./docs/adr/002-sequencer-centralized-then-decentralize.md) | Sequencer: **centralized v0–v1**, decentralize at v3 | Ship-fast, decentralize-later pattern |
| [003](./docs/adr/003-oracle-validator-quorum-push.md) | Oracle: **validator-quorum push** in EL | Sub-second updates, no external oracle |
| [004](./docs/adr/004-base-unit-usd-stable.md) | Base unit: **USD-stable** (USDC default; USDT, Tempo-stable at v1+) | |
| [005](./docs/adr/005-settlement-standalone-then-tempo.md) | Settlement: **standalone L1** v0–v1, Tempo settlement at v2 | |
| [006](./docs/adr/006-identity-anon-default-kyc-overlay.md) | Identity: **anon-default**, KYC as Layer-3 per-tenant overlay at v3 | Not chain-wide KYC |
| [007](./docs/adr/007-token-none-until-revenue.md) | Token: **none until real revenue** | Earliest consideration: post-v1 |

## Status

Princeps inherits a working Reth + Malachite kernel from [openhl](https://github.com/psyto/openhl) — fully functional as of 2026-05-31. **286 tests pass across 11 crates.**

**Built (Stages 1–18a):**
- ✅ Consensus substrate (Reth + Malachite, 4-message bridge)
- ✅ Two-validator devnet with real follower replication via `ProposalAndParts`
- ✅ CLOB pure state machine
- ✅ Custom EVM precompiles (CLOB read + `place_order` + fill sink + `deposit` + `withdraw`)
- ✅ Funding state machine (per-block settlement applied to collateral)
- ✅ Liquidation (margin math, insurance fund, multi-account scanner, ADL — full safety-net loop closed)
- ✅ Oracle (median-of-medians + secp256k1-signed observations + publisher registry + cached aggregate persistence)
- ✅ Vault primitive (share-based, ERC-4626-style)
- ✅ Clearing (`apply_fill` state machine, bridge-owned account map, persistent across restarts)
- ✅ Node coordinator + `reth-devnet` boot ceremony (persistent MDBX, validator key persistence, chain-spec loading, per-block integration tick)
- ✅ Margin-aware withdraw with revert-safe precompile mutations (`OpenHlRevertGuard`)

**Next (v0 lending build):**
- 🚧 Lending markets crate (collateral types, IRM, position health)
- 🚧 Cross-margin engine (multi-asset collateral)
- 🚧 Single-asset-pair devnet (USDC / ETH)
- 🚧 Sub-second deterministic liquidation demo (productized over the existing scanner + ADL primitives)
- 🚧 Multi-validator network expansion (3+ validators) building on Stage 18a follower replication

## Architecture

```
bin/princeps
├── crates/
│   ├── types         # primitives: account/market/position IDs
│   ├── codec         # serialization (Reth ↔ Malachite)
│   ├── clob          # pure orderbook state machine
│   ├── funding       # funding rate state machine
│   ├── liquidation   # margin math + insurance fund + scanner + ADL
│   ├── oracle        # median-of-medians + signed observations
│   ├── vault         # share-based collateral pooling
│   ├── clearing      # settlement / clearing primitives
│   ├── lending       # 🚧 v0 lending markets
│   ├── evm           # Reth EVM bridge + custom precompiles
│   ├── consensus     # Malachite integration
│   └── node          # PrincepsNode coordinator (tick driver)
```

**Pure / I-O split:** state machines (`types`, `codec`, `clob`, `funding`, `liquidation`, `oracle`, `vault`, `lending`) have no I/O — deterministic, microsecond unit tests. I/O boundary (`evm`, `consensus`, `node`) talks to the outside world.

**Workspace defaults:** edition 2024, resolver 3, rust 1.95+, `unsafe_code = forbid`, release LTO + abort + strip.

**Pin policy:** Reth v2.2.0, Malachite v0.5.0, alloy v1.5 / v2.0 — release-tag SHAs only, dedicated PR per bump.

## Quickstart

```bash
# Build
cargo build --release

# Single-validator devnet (in-memory bridge)
cargo run --release -- devnet 1

# Single-validator devnet with real Reth EVM
cargo run --release -- reth-devnet 1 --moniker dev0

# Multi-validator scaffolding
cargo run --release -- reth-devnet 1 \
  --chain-spec ./chain.json \
  --validators ./validators.json \
  --listen-addr /ip4/0.0.0.0/tcp/26656 \
  --rpc-bind 0.0.0.0:8545
```

Data and validator key default to `$HOME/.princeps/data` and `$HOME/.princeps/validator_key.json` (perms 0o600).

## Related projects

By the same author, separate independent projects worth knowing about:

- [**openhl**](https://github.com/psyto/openhl) — open-source HL reference implementation. Princeps's code ancestor (Princeps is a fresh fork without shared git history; openhl continues as its own project).
- [**rethlab**](https://rethlab.xyz) — Reth and perp DEX learning lab. 19 courses on the Reth stack, REVM, and perp internals. Not Princeps-specific, but covers most of the foundations Princeps is built on.

## License

Apache 2.0 — see [LICENSE](./LICENSE).

## Contributing

Princeps is currently a solo build. External contribution model will be defined post-v0 ship.

## Contact

- GitHub: [github.com/psyto/princeps](https://github.com/psyto/princeps)
- X / Twitter: [@psyto](https://twitter.com/psyto)

---

*Princeps is bootstrapped solo by [Hiroyuki Saito](https://github.com/psyto). Open-source from day one. No token. No raise. Built on the [openhl](https://github.com/psyto/openhl) reference implementation.*
