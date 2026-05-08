# Continuum.Markets — Project Summary

Solana-native protocol for 24/7 synthetic exposure to traditional assets via paired long/short tokens settled at oracle NAV. The position is a transferable SPL token (no margin, no funding rate, no liquidation), and the peg is held by an on-chain keeper running an arbitrage loop against Meteora DLMM pools — no off-chain hedging desk, no custodian who can freeze your wallet.

For full architecture see [`docs/ARCHITECTURE.md`](./ARCHITECTURE.md). For operations see [`docs/OPERATIONS.md`](./OPERATIONS.md). For per-instruction API see [`docs/API.md`](./API.md). For the keeper bot see [`keeper/README.md`](../keeper/README.md).

---

## What's actually built (MVP)

### Solana programs (Anchor / Rust)

| Program | Purpose | Status |
|---|---|---|
| `mint-redeem` | Paired mint/redeem at NAV (user-facing); keeper-only single-side mint/redeem (privileged) | Live on devnet |
| `oracle` | Pyth on-chain primary + Hermes HTTP fallback + TWAP + risk-state machine (`Normal`/`ProxyMode`/`Stress`/`Recovery`) | Live on devnet |
| `clp` | Capital & Liquidity Provider — CLP PDA owns Meteora DLMM positions via CPI proxy; per-market + global cUSDC vaults | Live on devnet |
| `governance` | CNTM staking, listing/parameter votes | Scaffolded, not yet wired |
| `faucet` | Devnet cUSDC drip | Live on devnet |
| `registry` | Market registry | Live on devnet |

### Frontend (Next.js 15)

- Pool views, mint/redeem UI, portfolio
- TradingView chart integration with NAV overlay (long pool projected via constant-product invariant for visual coherence check)
- Wallet adapters (Phantom, Backpack, Solflare)

### Keeper bot (Rust)

Off-chain async loop. Owns the market-making and peg-enforcement lifecycle: oracle updates, DLMM position management, single-side and paired arb, edge-weighted capital seeding from the global vault. Full architecture in [`keeper/README.md`](../keeper/README.md).

### External dependencies

- Meteora DLMM (pool infrastructure)
- Pyth + Hermes (oracle)
- Metaplex (token metadata)
- MarginFi / Kamino flash-loan adapters (optional, off by default)

---

## Core mechanics

### NAV

```
L_NAV = user_twap_price (or initial_l_price if TWAP == 0)
S_NAV = (initial_l_price × initial_s_price) / L_NAV
```

The constant-product invariant `L_NAV × S_NAV = initial_l × initial_s` holds at all times. Gains on one side mirror losses on the other; combined pair value `L_NAV + S_NAV` stays bounded.

### User mint / redeem

Users can only `mint_paired` and `redeem_paired`. Mint deposits cUSDC, returns matched L+S sized by NAV. Redeem burns L and S in independently-chosen amounts (NAV-weighted, not quantity-paired).

### Keeper arb

Two paths run each scan tick (15s default):

1. **Single-side** (primary): when one pool diverges from its NAV by more than `pool_fee_bps + slip/2`, the keeper either mints fee-free at NAV and sells on pool (pool over NAV) or buys from pool and redeems fee-free at NAV (pool under NAV). Uses `keeper_mint_single` / `keeper_redeem_single`, gated on signer matching `market.keeper_authority`.
2. **Paired**: when the *combined* L+S spread diverges from combined NAV. Mint pair → sell both legs, or buy both legs → redeem pair.

Both nets back to cUSDC. End-of-cycle and boot-time sweeps reclaim any residual synth.

### Pool fee tier

Default `binStep=10` (0.10% base fee) on the QQQ market — tightens the equilibrium NAV peg band to ±0.10%. Higher binStep recommended for more volatile underlyings (NVDA at 0.20-0.25%, etc.). Trade-off: lower fees → less LP revenue per swap but tighter peg and likely higher swap volume from arb activity.

---

## Asset roadmap

Targeted launch coverage is TradFi-only — no crypto markets:

- **Indices**: SPX, NDX (QQQ), FTSE 100, DAX
- **Equities**: NVDA, AAPL, TSLA, GME, AMC
- **Commodities**: XAU (Gold), XAG (Silver), WTI (Oil)
- **Forex**: EUR/USD, USD/JPY, GBP/USD

QQQ is the live devnet market. Other markets will roll on per the create-synthetic-market script flow described in [`docs/OPERATIONS.md`](./OPERATIONS.md).

---

## Things that explicitly do **not** exist

- No funding rate. Constant-product NAV invariant replaces it — no need to drain holders of the popular side.
- No liquidations. The position is a token, not a margin entry.
- No leverage. Mint cost = NAV × pair count; users prepay full collateral.
- No in-house AMM. Meteora DLMM is the trading venue; the protocol is the sole LP.
- No user-facing yield vaults. The CLP capital is operator-funded for the MVP. Idle-collateral yield modules (`yield_harvesting.rs`, `collateral_yield.rs`) compile but `YIELD_ENABLED=false`.
- No third-party LP shares. `GlobalLPAccount` was removed; admin funds the vault directly.

If you find references to "funding", "AMM", "margin & liquidations", or "insurance fund vaults" in older docs (`FINAL_STATUS.md`, `IMPLEMENTATION_STATUS.md`, etc.), those reflect an earlier design pivot that was never built. The historical snapshot files have `DEPRECATED` headers pointing to the canonical docs.

---

## Where to read next

- [`/README.md`](../README.md) — top-level project overview
- [`docs/ARCHITECTURE.md`](./ARCHITECTURE.md) — full system architecture, end-to-end flows, solvency invariants
- [`docs/OPERATIONS.md`](./OPERATIONS.md) — operator runbook (deploy market, fund vault, drain pools, recover from zombie positions)
- [`docs/API.md`](./API.md) — per-instruction reference
- [`docs/DEPLOYMENT.md`](./DEPLOYMENT.md) — devnet/mainnet deployment guide
- [`docs/DEVELOPMENT.md`](./DEVELOPMENT.md) — local dev setup
- [`keeper/README.md`](../keeper/README.md) — keeper bot architecture, arb paths, DLMM rebalancing, solvency
