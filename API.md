# Continuum.Markets API Reference

> **Status note:** This doc reflects the current MVP programs (`mint-redeem`, `oracle`, `clp`, `governance` scaffolded). Older drafts referenced Funding / AMM / Margin / Insurance programs that were never shipped. For authoritative per-instruction accounts and parameters, see IDLs in `target/idl/` after `anchor build`.

## Program instructions

### mint-redeem program

Devnet ID: `5MBjhNUUguLTPNR5WG6YBUUw7vUcxQ14ARw3NsS3rKu4`

#### `initialize_market(asset_symbol, oracle_type, initial_l_price, initial_s_price, mint_fee_bps, redeem_fee_bps)`
Initialize a new synthetic asset market. Creates the Market PDA, long/short SPL mints (authority = market PDA), and collateral vault.

#### `mint_paired(amount)`
**Paired** mint at NAV (open to anyone). Deposit `amount` cUSDC, receive matched L+S tokens.

- Transfers `amount × mint_fee_bps / 10_000` cUSDC → fee recipient
- Net `N = amount - fee`; half value to each side
- User receives `N / (2 × L_NAV)` L-tokens + `N / (2 × S_NAV)` S-tokens

Accounts: `market`, `user` (signer), `user_collateral`, `user_long`, `user_short`, `long_mint`, `short_mint`, `collateral_vault`, `dev_token_account`, `oracle_address`, `token_program`.

For users, there is **no single-sided mint** — see `keeper_mint_single` below for the privileged path.

#### `redeem_paired(l_amount, s_amount)`
**Paired** redeem (open to anyone). Burn L and S in independently-chosen amounts; receive `l_amount × L_NAV + s_amount × S_NAV - redeem_fee` cUSDC.

Amounts need not match in quantity — L and S have different prices, so the pair is weighted by NAV, not count.

Accounts: same as `mint_paired`.

For users, there is **no single-sided redeem** — see `keeper_redeem_single` below.

#### `keeper_mint_single(is_long, collateral_amount)`
**Keeper-only, fee-free**, single-side. Deposit `collateral_amount` cUSDC, receive only the chosen side's tokens at NAV. No mint fee, no worst-case quoting markup, no q-imbalance gate.

- Signer must match `market.keeper_authority`
- `is_long = true` → receive `collateral_amount / L_NAV` long tokens
- `is_long = false` → receive `collateral_amount / S_NAV` short tokens

Used in two-phase keeper arb cycles to close per-side pool divergence above NAV. The keeper signs both this mint and the offsetting pool swap atomically per cycle; mid-flight crashes are caught by the boot-time `sweep_residual_pairs`.

#### `keeper_redeem_single(is_long, token_amount)`
**Keeper-only, fee-free**, single-side. Burn `token_amount` of the chosen side's tokens, receive `token_amount × NAV` cUSDC.

- Signer must match `market.keeper_authority`
- Used after the keeper has bought the underpriced side from the pool, or to clear asymmetric residue at end of arb cycles

#### `update_risk_state(new_state)`
Keeper-only. Mirrors oracle risk state (`Normal` | `ProxyMode` | `Stress` | `Recovery`) into the market. Signer must match `market.keeper_authority`.

---

### oracle program

Devnet ID: `5vxiCrDpFnQ2W5QtgZBC66K2XTC19bjVBjinGYYBsadC`

Maintains price observations + TWAP + risk-state machine. Keeper pushes observations every ~15s (Pyth on-chain, Hermes HTTP fallback on devnet).

Primary instructions: `initialize_oracle_config`, `update_observation`, `configure_oracle_feeds`, `set_risk_fee_config`, `update_twap_config`.

Price helpers: the oracle program is read-only for mint-redeem/CLP consumers — they read `user_twap_price`, `last_update_time`, and `state` directly from the on-chain account.

---

### clp (Capital & Liquidity Provider) program

Devnet ID: `8xauDRjw9XRyk4FE3hW1JKjD8nC87gfr59Xig1dJqLES`

Holds per-market and global cUSDC vaults plus all Meteora DLMM positions on the protocol's behalf. The CLP PDA (seed `[b"clp", market_pda]`) signs Meteora CPI calls.

**Admin lifecycle:**
- `initialize_global_clp()` — one-shot global state init
- `initialize_clp(oi_cap, oracle_config)` — per-market init
- `configure_meteora_pools(long_pool, short_pool)` — wire pool addresses
- `update_oi_cap(new_cap)`, `configure_inventory_budget(...)`, `configure_yield_strategy(...)`, `configure_liquidity_seeder(threshold)`

**Capital flow:**
- `admin_fund(amount)` — operator deposits cUSDC into global vault
- `admin_withdraw(amount)` — operator withdraws idle global vault balance
- `allocate_to_market(amount)` — keeper moves cUSDC from global → per-market vault
- `return_to_global(amount)` — keeper returns idle per-market cUSDC
- `deposit_profit(amount)` — keeper routes arb surplus into global vault

**Paired mint/redeem via CLP (keeper-called):**
- `vault_mint_pairs(amount)` — CPI into mint-redeem with per-market cUSDC
- `vault_redeem_pairs(l_amount, s_amount)` — CPI into mint-redeem with CLP-held L+S

**Meteora DLMM proxy (keeper-called):**
- `clp_init_bin_array(index)` — init a bin array PDA if missing. Signs with CLP authority (the program enforces `has_one = authority`)
- `clp_open_meteora_position(side, amount_x, amount_y, active_id, target_bin, bin_radius)` — create + fund position
- `clp_topup_meteora_position(side, amount_x, amount_y, active_id, lower_bin, upper_bin)` — add liquidity to an existing position. Used by the seeder to grow depth on positions still in range without paying close+reopen rent. `lower_bin`/`upper_bin` must match the position's stored range
- `clp_remove_meteora_liquidity(side, lower_bin, upper_bin)` — drain position
- `clp_close_meteora_position(side, lower_bin, upper_bin)` — close + reclaim rent
- `clp_set_meteora_position(side, position_pk)` — admin helper: rebind an orphaned CLP-owned position so it can be drained
- `clear_meteora_positions()` — reset stored position pubkeys to default

---

### governance program (scaffolded)

Devnet ID: `6es5KcjMWKGhVWrttUytYiZ3YELXe4HQyrmsMVdbVawT`

Planned: CNTM staking, proposals for listings / parameter changes, time-locked execution. Not yet wired into program execution paths.

---

### faucet program (devnet-only)

Devnet ID: `9tUeQAPEtVSB68NSfvFAqfwaB74GuVxm6Zbp1hrMiNKY`

Provides cUSDC drips for devnet testing. Key instructions: `initialize_faucet`, `drip`, `update_params`.

Admin helpers in `scripts/`: `admin-fund-global-vault.ts` (cranks faucet to fund the global vault in one call), `faucet-drip.ts` (one-off drip, respects cooldown).

---

### Not shipped

These programs were designed but never implemented — no on-chain deployment, no code in `programs/`:
- **Funding** — not needed; constant-product NAV invariant bounds pair value
- **AMM** — replaced by external Meteora DLMM pools (CLP is the LP)
- **Margin** / liquidation — not applicable, paired mint is pre-paid
- **Insurance** — folded into CLP profit-share and over-collateralisation

---

## REST API Endpoints

### Market Data

#### `GET /api/markets`
Get all available markets.

**Response**:
```json
[
  {
    "id": "spx",
    "symbol": "SPX",
    "name": "S&P 500",
    "category": "Index",
    "oraclePrice": 5234.56,
    "longPrice": 5234.56,
    "shortPrice": 1.02,
    "fundingRate": 0.0123,
    "volume24h": 1234567,
    "openInterest": 5678901,
    "oiCap": 10000000,
    "isActive": true
  }
]
```

#### `GET /api/markets/:id`
Get specific market details.

#### `GET /api/markets/:id/orderbook`
Get orderbook (from Redis cache).

**Response**:
```json
{
  "bids": [
    { "price": 5234.50, "size": 1.5, "total": 7851.75 }
  ],
  "asks": [
    { "price": 5235.00, "size": 1.2, "total": 6282.00 }
  ]
}
```

### Portfolio

#### `GET /api/portfolio/:wallet`
Get user portfolio.

**Response**:
```json
{
  "totalValue": 10000.00,
  "totalPnL": 150.50,
  "positions": [
    {
      "market": "SPX",
      "side": "long",
      "size": 1.5,
      "entryPrice": 5200.00,
      "currentPrice": 5234.56,
      "pnl": 51.84
    }
  ]
}
```

### Governance

#### `GET /api/governance/proposals`
Get all proposals.

#### `GET /api/governance/proposals/:id`
Get proposal details + IPFS metadata.

#### `GET /api/governance/stats`
Get governance stats (total staked, active proposals, etc.)

---

## WebSocket Feeds

### Price Updates
```typescript
ws://api.continuum.markets/ws/price/:market
```

**Message Format**:
```json
{
  "market": "SPX",
  "price": 5234.56,
  "change24h": 1.23,
  "timestamp": 1735689600
}
```

### Liquidations
```typescript
ws://api.continuum.markets/ws/liquidations
```

**Message Format**:
```json
{
  "account": "7xK...abc",
  "market": "NVDA",
  "collateralLiquidated": 1000.00,
  "bounty": 50.00,
  "timestamp": 1735689600
}
```

---

## Rate Limits

- Public endpoints: 100 req/min
- Authenticated endpoints: 1000 req/min
- WebSocket connections: 10 per IP

## Error Codes

| Code | Meaning |
|------|---------|
| 6000 | Market not active |
| 6001 | Invalid amount |
| 6002 | Insufficient collateral |
| 6003 | Oracle price unavailable |
| 6004 | Slippage exceeded |
| 6005 | Position healthy (can't liquidate) |
| 6006 | OI cap exceeded |
| 6007 | Voting closed |

---

For more examples, see [DEVELOPMENT.md](./DEVELOPMENT.md).
