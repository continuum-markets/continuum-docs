# Continuum Architecture

## System overview

Continuum is a Solana-native protocol for synthetic long/short exposure to real-world assets, priced off oracle NAV. Users mint paired L+S tokens against a cUSDC collateral deposit; the paired tokens trade on external DLMM venues (Meteora) where a keeper bot enforces the peg via arbitrage against NAV.

Unlike orderbook-perps, there is no funding rate, no liquidation, no margin. Position P&L = (L_NAV × L_holdings + S_NAV × S_holdings) - cost_basis; the two NAVs are linked by a constant-product invariant `L_NAV × S_NAV = initial_L × initial_S`, so gains on one side are mirrored by losses on the other while the pair value stays bounded.

## Core programs (Solana)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Next.js)                        │
│  Mint / Redeem UI · Pool views · Portfolio                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Anchor / web3.js
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Continuum programs                           │
│                                                                  │
│  ┌─────────────────────────┐                                    │
│  │ mint-redeem             │  Users deposit cUSDC → paired L+S  │
│  │ 5MBjhNU...rKu4          │  Burn paired L+S → cUSDC at NAV    │
│  └─────────────────────────┘                                    │
│                                                                  │
│  ┌─────────────────────────┐                                    │
│  │ oracle                  │  Pyth/Hermes → on-chain NAV state  │
│  │ 5vxiCrD...sadC          │  TWAP + risk-state machine         │
│  └─────────────────────────┘                                    │
│                                                                  │
│  ┌─────────────────────────┐                                    │
│  │ clp (Capital & Liquidity│  Per-market + global cUSDC vaults  │
│  │  Provider)              │  Holds Meteora positions as PDA    │
│  │ 8xauDRj...qLES          │  CPI proxy to Meteora DLMM         │
│  └─────────────────────────┘                                    │
│                                                                  │
│  ┌─────────────────────────┐                                    │
│  │ governance              │  Market listing + parameter votes  │
│  │ 6es5KcjM...avwT         │  (scaffolded, not yet active)      │
│  └─────────────────────────┘                                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ CPI
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                 External protocols                               │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Meteora DLMM    │  │     Pyth     │  │    Hermes        │  │
│  │  L/cUSDC pool    │  │  Price feed  │  │  HTTP fallback   │  │
│  │  S/cUSDC pool    │  │  on-chain    │  │  (devnet)        │  │
│  └──────────────────┘  └──────────────┘  └──────────────────┘  │
│  ┌──────────────────┐  ┌──────────────┐                        │
│  │  MarginFi/Kamino │  │  Metaplex    │                        │
│  │  (flash loans,   │  │  Token       │                        │
│  │   optional)      │  │  Metadata    │                        │
│  └──────────────────┘  └──────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
                           ▲
                           │ runs off-chain, signs as
                           │ keeper wallet + CLP authority
                           │
┌─────────────────────────────────────────────────────────────────┐
│                       Keeper bot (Rust)                          │
│  Oracle updates · DLMM rebalancing · Paired arbitrage            │
│  Liquidity seeding · Risk state propagation · Dashboard          │
└─────────────────────────────────────────────────────────────────┘
```

## Program details

### `mint-redeem`

Entry point for user capital. Creates and destroys paired synth tokens at NAV.

**Key instructions:**

User-facing (paired only — solvency invariant for general public):
- `mint_paired(amount_cusdc)` — deposit cUSDC, receive `(amount × (1 - mint_fee_bps) / 2) / L_NAV` L-tokens plus the analogous S-amount. Anyone can call.
- `redeem_paired(l_amount, s_amount)` — burn L and S in independent amounts; receive `l_amount × L_NAV + s_amount × S_NAV` cUSDC. Amounts are independent (not quantity-paired) because L and S have different prices. Anyone can call.

Keeper-only (signer must equal `market.keeper_authority`, fee-free):
- `keeper_mint_single(is_long, collateral_amount)` — deposit cUSDC, receive only the chosen side's tokens at NAV (no fee, no worst-case quoting markup, no q-imbalance check). Used by the keeper to push the pool price down on the underpriced-on-pool side by selling the freshly minted tokens.
- `keeper_redeem_single(is_long, token_amount)` — burn only the chosen side's tokens, receive `token_amount × NAV` cUSDC. Used after the keeper has bought the overpriced side from the pool. Also handles asymmetric residue at end of arb cycles.

Operator:
- `update_risk_state` — called by the keeper authority to mirror oracle risk state into the market account.

**Why is the keeper allowed unilateral single-side mint/redeem?** It would be unsafe for a user — minting only L without an offsetting S deposit creates an unbacked claim against the collateral vault. But the keeper-authority single-side path is part of an immediate two-phase arb cycle that nets back to cUSDC. The mint creates synth, the next pool swap returns cUSDC at the higher pool price, and the cUSDC delta lands in the keeper or CLP vault. The collateral side stays balanced because the synth gets burned (via subsequent redeem) or sits with the keeper as inventory backed by the pool position it bought. Net effect across both phases mirrors a paired round-trip but routes around the per-side fee + worst-case quoting drag that previously blocked sub-25-bps arbs.

**State:**
- `Market` PDA per asset (seed `[b"market", symbol]`). Holds `initial_l_price`, `initial_s_price`, `user_twap_price`, fees, mints, oracle address, and `keeper_authority` pubkey.
- `CollateralVault` token account holds all cUSDC backing outstanding L+S supply.

**NAV derivation:**
```
L_NAV = user_twap_price (from on-chain TWAP) or initial_l_price if TWAP==0
S_NAV = (initial_l_price × initial_s_price) / L_NAV
```
The constant-product relationship means `L_NAV × S_NAV` is invariant at the product of the initial prices.

### `oracle`

Maintains on-chain price observations for each market. Supports Pyth (primary), Hermes (HTTP fallback on devnet), and Switchboard (configurable per market).

**State machine:** `Active` ↔ `Passive` based on staleness; the mint-redeem program gates on this state plus a `RiskState` (`Normal`/`ProxyMode`/`Stress`/`Recovery`) to throttle mint sizing and quote conservatism.

**Keeper responsibility:** poll Hermes every 15s, push TWAP observations, and drive risk-state transitions.

### `clp` — Capital & Liquidity Provider

Owns all LP positions on external DEX venues. The CLP PDA (seed `[b"clp", market_pda]`) signs Meteora DLMM instructions via CPI, and holds:
- A per-market cUSDC vault (`clp_vault`)
- A global cUSDC vault (`global_clp`, deposit/withdraw venue for LP users)
- Per-market L and S token ATAs
- References to each market's long and short Meteora pool + current position

**Key instructions (keeper-called):**
- `allocate_to_market(amount)` — move cUSDC from global vault to per-market vault
- `vault_mint_pairs(amount)` — CPI mint_paired using per-market vault cUSDC
- `vault_redeem_pairs(l_amount, s_amount)` — CPI redeem_paired
- `clp_init_bin_array(index)` / `clp_open_meteora_position(...)` / `clp_topup_meteora_position(...)` / `clp_remove_meteora_liquidity(...)` / `clp_close_meteora_position(...)` — Meteora DLMM proxy. `topup` adds liquidity to an existing position; the seeder uses it to grow depth on positions still in range without paying close+reopen rent.
- `clp_set_meteora_position(side, position_pk)` — admin helper to recover an orphaned CLP-owned position
- `clear_meteora_positions()` — wipe stored position pointers (used during pool migration)
- `configure_meteora_pools(long_pool, short_pool)` — wire pool addresses into CLP state
- `return_to_global(amount)` — send cUSDC from per-market vault back to global
- `deposit_profit(amount)` — keeper calls this with any arb surplus; funds re-enter the global vault for redeployment

**Why a proxy?** Meteora positions have an `owner` field. By routing through a CLP PDA, no single keeper wallet has unilateral control over LP capital — position operations require the CLP program's logic to succeed (which validates each call against stored pool/position addresses, market PDA, etc.). This allows keeper rotation without re-creating positions.

### `governance` (scaffolded)

Planned: CNTM staking for market-listing votes and parameter changes. Not yet wired into program execution.

## Keeper bot

Off-chain Rust binary. See [`keeper/README.md`](../keeper/README.md) for a full architecture walkthrough. High-level responsibilities:

| Loop | Cadence | Purpose |
|---|---|---|
| Oracle | 15s | Push Hermes/Pyth observations to the oracle program |
| Risk | 15s | Derive `OracleHealth` state machine |
| Inventory | 60s | Track per-market q-imbalance and drawdown |
| DLMM | 120s | Open / rebalance / recall Meteora positions via CLP proxy |
| Arb | 15s | Scan pools, execute paired arbs on combined-price spread |
| Seeder | 600s | Edge-weighted capital allocation from global vault |

The keeper signs with two keypairs: a **payer** (SOL for fees, signs direct swap txs) and a **CLP authority** (matches `market.keeper_authority`, signs CLP proxy instructions).

## End-to-end flows

### User mint

1. Frontend calls `mint_redeem::mint_paired(amount)` with the user as signer
2. Program reads oracle to get `L_NAV`, derives `S_NAV` from constant-product formula
3. Program transfers cUSDC user → `CollateralVault`
4. Program mints `half_value / L_NAV` L-tokens and `half_value / S_NAV` S-tokens to the user's ATAs
5. `mint_fee_bps` of cUSDC transferred to fee recipient before the mint

### User redeem

1. User calls `mint_redeem::redeem_paired(l_amount, s_amount)`
2. Program burns L and S from the user's ATAs
3. Program transfers `l_amount × L_NAV + s_amount × S_NAV` cUSDC from vault to user (minus `redeem_fee_bps`)

### Keeper arb — single-side mint+sell (one side over NAV)

```
Phase 1  keeper_mint_single(is_long, X cUSDC)  → X / NAV synth tokens
Phase 2  swap synth → cUSDC on that side's DEX → cUSDC at the higher pool price
Profit:  X × (pool/NAV - 1) - pool_fee - slip
End      keeper wallet: +profit cUSDC, 0 synth on that side
```

### Keeper arb — single-side buy+redeem (one side under NAV)

```
Phase 1  swap cUSDC → synth on that side's DEX → cheaper synth at the lower pool price
Phase 2  keeper_redeem_single(is_long, Y synth) → Y × NAV cUSDC
Profit:  Y × (NAV/pool - 1) - pool_fee - slip
End      keeper wallet: +profit cUSDC, 0 synth on that side
```

### Keeper arb — paired sell (combined_dex > combined_nav)

```
Phase 0  mint_paired(trade_amount)         → N L + N S in keeper wallet
Phase 1  swap L → cUSDC on long pool       → push L price down toward NAV
Phase 2  swap S → cUSDC on short pool      → push S price down toward NAV
Net      spent trade_amount × combined_nav
         received ≈ N × combined_dex cUSDC
         profit = N × (combined_dex - combined_nav) - fees
End      keeper wallet: +profit cUSDC, 0 synth
```

### Keeper arb — paired buy (combined_dex < combined_nav)

```
Phase 0  swap cUSDC → L on long pool       → push L price up toward NAV
Phase 1  swap cUSDC → S on short pool      → push S price up toward NAV
Phase 2  redeem_paired(actual_L, actual_S) → receive cUSDC at NAV
         (amounts read at execution time; not pre-computed min_out,
          which would leak the slippage buffer as residual synth)
End      keeper wallet: +profit cUSDC, 0 synth
```

### Keeper DLMM rebalance

```
1. Tick fires every DLMM_INTERVAL_SECS
2. For each market × side:
     - Read on-chain active_id and current position range
     - Compute NAV target_bin from Hermes price
     - Compute vol-adaptive bin_radius from 15min realized vol
     - Compute fee-weighted centroid from per-bin fee accumulators
     - If existing range still covers active AND target → skip (no rebuild)
     - Else:
         clp_remove_meteora_liquidity
         clp_close_meteora_position
         [clp_init_bin_array if needed]
         clp_open_meteora_position(
             target = blend(centroid, naive_midpoint),
             radius = vol_adaptive_radius,
             strategy = CurveBalanced | CurveImBalanced,
         )
3. Any profit beyond the pre-rebalance allocated principal is
   return_to_global'd; CLP vault keeps a small liquid buffer
```

## Solvency invariants

1. **Paired-only mint/redeem for users.** The user-facing `mint_paired` / `redeem_paired` instructions always produce or burn matched L+S. The program rejects all other paths from non-keeper-authority callers.
2. **Single-side mint/redeem privileged to `market.keeper_authority`.** Used to close per-side NAV divergence in two-phase arb cycles. The keeper signs both the mint and the offsetting pool swap (or the offsetting pool buy and the redeem) atomically per cycle; mid-flight crashes are caught by the boot-time `sweep_residual_pairs`. Net effect across both phases mirrors a paired round-trip from the perspective of total supply and collateral.
3. **Keeper post-sweep solvency.** Every arb cycle ends in cUSDC at the keeper or CLP vault. End-of-cycle and boot-time sweeps reclaim any residual synth via `redeem_paired` (both sides present) or `keeper_redeem_single` (asymmetric residue). No accumulating synth inventory.
4. **All CLP-owned positions backed by on-chain cUSDC state.** The CLP PDA tracks `allocated_from_global` for each market; rebalance profit detection distinguishes principal from fees and returns fees to the global vault.
5. **Hard bound gating.** Per-market `q` (long/short imbalance) and `drawdown_bps` are tracked; user-facing operations at hard bounds are either restricted or routed through alternative paths (e.g. proxy pricing in ProxyMode). Keeper single-side ixns bypass the q-imbalance check by design — they exist precisely to push q back toward 50/50 by closing per-side NAV divergence.
6. **Constant-product L/S NAV.** `L_NAV × S_NAV = initial_l × initial_s` always holds, so the paired value `L_NAV + S_NAV` is bounded and the total collateral required for `N` pairs is deterministic.

## Data flow: full cycle

```
Hermes ────────── keeper::oracle ──────► mint_redeem::update_risk_state
                                                     │
                                                     ▼
                                         market.user_twap_price updated
                                                     │
  user mint_paired(X) ◄──────────────────────────────┤
                          cUSDC → vault              │
                          L+S   → user ATAs          │
                                                     │
  keeper::arb scan ──────────────────────────────────┤
                 │
                 ├── paired trade on Meteora pools ──► prices converge to NAV
                 │
                 └── arb surplus → clp::deposit_profit → global vault
                                                     │
                                                     ▼
                                     keeper::liquidity_seeder allocates
                                     global vault capital back into pools
                                     edge-weighted across markets
```

## Risk and failure modes

| Failure | Mitigation |
|---|---|
| Oracle stale/halted | `OracleHealth::Passive`, operations gated by `can_rebalance()` |
| Meteora position zombied (non-empty `liq_share`, drained bin arrays) | `is_position_zombie` detects; rebalancer skips close and opens fresh |
| RPC failure during position existence check | `position_exists` returning `Err` is treated as "alive" to prevent orphan duplicates |
| Keeper crash mid-arb | Startup sweep redeems residual paired L+S to cUSDC |
| Pool runs out of one side's depth | Per-leg depth gating in arb scan; trade capped at `min(long_depth, short_depth) × 2` |
| Fee revenue imbalance on specific bins | Fee-weighted centroid biases next position center toward high-volume bins |
| Price drifts outside position range | Vol-adaptive radius widens in high-vol regimes; overflow triggers immediate rebuild |

## Future work

- **Flash-loan integration** — MarginFi/Kamino adapters exist but are off by default; enabling bumps arb trade size without requiring keeper treasury
- **Governance activation** — `governance` program exists; listing/parameter votes not yet wired to program execution
- **Cross-market keeper coordination** — peers heartbeat exists; primary/standby logic is basic
- **Vol regime auto-tuning** — current thresholds (10/25/50% annualized) are VIX-inspired heuristics; asset-specific percentile calibration would be more optimal
- **Custom per-bin weights** — currently using Meteora's `CurveBalanced` default shape; custom weights from fee distribution history would be strictly better but requires a new CLP instruction
- **L/S direct pool** — earlier prototyped (triangular and LS-pool alignment arb paths in `arb.rs`); superseded by single-side keeper mint/redeem since the latter is fee-free at the protocol layer where LS arb paid both `mint_fee_bps + redeem_fee_bps + extra DEX leg`. Code remains gated on `LS_POOLS` env var for re-enable but should stay off in production
