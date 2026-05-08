# Operations Runbook

This document is the canonical guide for running Continuum — deploying new markets, funding the protocol vault, controlling supply, and keeping the keeper fed. Everything here is performed by the **admin authority** (the keypair stored in `$WALLET_PATH`, defaulting to `~/.config/solana/id.json`). There are no user-facing vault deposits — the protocol itself is the sole LP, funded by the operator.

## Mental model

```
Admin wallet
    │  admin_fund(amount)                    ← you
    ▼
Global CLP vault  (protocol treasury)
    │  allocate_to_market(symbol, amount)    ← keeper, every 600s
    ▼
Per-market vault
    │  vault_mint_pairs  →  L+S synth tokens
    │  clp_open_meteora_position
    ▼
Meteora DLMM  (L pool + S pool)
    │  rebalance, arb, harvest fees          ← keeper, every 120s
    ▼
Fees + profit  →  deposit_profit  →  Global vault
```

**Capital flow:** You top up the global vault. The keeper does everything from there — allocating to markets, minting L+S pairs, deploying Meteora positions, rebalancing, arbitraging NAV divergence, recycling fees back to the global vault.

**Supply control:** L/S token total supply is bounded per-market by `oi_cap`. Mint/redeem fees discourage runaway minting. The pause switch (via governance) kills a market's mint/redeem if something's wrong.

---

## Environment setup

Every script reads:

```bash
RPC_URL=https://api.devnet.solana.com          # default
WALLET_PATH=~/.config/solana/id.json           # admin keypair
```

All scripts live in `scripts/` and are invoked via `bun run scripts/<name>.ts`.

---

## Deploying a new market

Example: launching NVDA.

### 1. Mint the synthetic tokens and initialise on-chain state

```bash
SYMBOL=NVDA \
INITIAL_PRICE=120 \
bun run scripts/create-synthetic-market.ts
```

This creates the `Market` PDA (mint-redeem program), the L and S SPL mints, and the collateral vault. Initial price should match the current Pyth oracle for the asset.

### 2. Initialise the per-market CLP account

```bash
SYMBOL=NVDA bun run scripts/initialize-clp.ts
```

Creates the per-market `Clp` account that holds risk config, OI cap, and is the PDA authority for the per-market USDC vault.

### 3. Set the OI cap

The OI cap bounds how many L/S synth tokens can exist for this market. Start conservative; widen as liquidity grows.

```bash
SYMBOL=NVDA \
OI_CAP=250000 \
bun run scripts/update-clp-oi-cap.ts
```

Rule of thumb: `oi_cap ≈ 2 × expected_LP_depth`. If you're seeding $200k of liquidity, set cap around $400k–$500k. Tighter is safer — cap can be raised later.

### 4. Create CLP PDA token ATAs

The CLP PDA needs ATAs for L, S, and USDC so it can hold tokens before deploying them to Meteora.

```bash
SYMBOL=NVDA bun run scripts/create-clp-token-atas.ts
```

### 5. Seed the Meteora DLMM pools

Creates the long and short DLMM pools and seeds initial liquidity around the current Pyth price.

```bash
MARKET=NVDA \
TOKEN_AMOUNT=5000 \
USDC_AMOUNT=5000 \
BIN_STEP=10 \
BASE_FACTOR=10000 \
bun run scripts/seed-meteora-liquidity.ts
```

This writes the pool addresses into `frontend/lib/market-addresses.json` automatically. After this runs, the frontend sees the new market.

**Pool fee tier guidance:**

`base_fee_pct = bin_step × base_factor / 1e6`. Recommended starting point per asset class:

| Asset profile | binStep | baseFactor | Base fee | Position range (radius=34) |
|---|---|---|---|---|
| Index ETF (QQQ, SPY) | 10 | 10000 | 0.10% | ±3.4% |
| Liquid equity (NVDA, AAPL) | 20 | 10000 | 0.20% | ±6.8% |
| Volatile equity (TSLA, GME) | 25 | 10000 | 0.25% | ±8.6% |

Lower fees → tighter NAV peg band but less LP revenue per swap. For QQQ-class assets `binStep=10` is the current default. Going below `binStep=5` is not recommended — the position range collapses too tight and intraday spikes blow active_id out of range, triggering close+reopen cycles.

### 6. Fund the global vault (if not already funded)

See [Funding the global vault](#funding-the-global-vault) below.

### 7. Verify

```bash
bun run scripts/protocol-status.ts
```

Shows the new market, its OI cap, keeper allocation, and pool depth. The keeper will start allocating from the global vault on its next 600s tick.

---

## Funding the global vault

There are two paths. Pick based on context.

### Clean path (mainnet-grade)

Calls the on-chain `admin_fund` instruction. Updates the cumulative `total_deposits` counter on `GlobalClp` so stats are accurate. Requires the admin wallet to already hold the USDC.

```bash
AMOUNT=50000 bun run scripts/admin-fund.ts
```

### Devnet shortcut

On devnet, the admin typically doesn't hold USDC yet. This script cranks the cUSDC faucet drip to the requested amount, mints in a single call, restores faucet params, and transfers straight to the vault via a plain SPL transfer.

```bash
AMOUNT=200000 bun run scripts/admin-fund-global-vault.ts
```

Note: the shortcut bypasses `admin_fund`, so the on-chain `total_deposits` stat doesn't advance. Funds still reach the vault and the keeper still sees them; only the cumulative counter is affected. Prefer the clean path once you have real USDC.

---

## Withdrawing from the global vault

Only **idle** balance can be withdrawn. Capital already allocated to per-market vaults or deployed to Meteora positions is not reachable via `admin_withdraw` — recall it first.

### Withdraw idle balance

```bash
AMOUNT=10000 bun run scripts/admin-withdraw.ts
```

### Recall capital from a market (before withdrawing)

1. Close Meteora positions for the market:

   ```bash
   MARKET=NVDA bun run scripts/withdraw-meteora-liquidity.ts
   MARKET=NVDA bun run scripts/clear-meteora-positions.ts
   ```

2. Redeem L+S balances on the CLP PDA back to USDC in the per-market vault:

   ```bash
   MARKET=NVDA bun run scripts/redeem-clp-synths.ts
   ```

   This calls `vault_redeem_pairs(L_balance, S_balance)` with whatever sits in the CLP PDA's L and S ATAs. `redeem_paired` accepts asymmetric amounts so a balanced pair isn't required.

3. The keeper's next tick will `return_to_global` the now-idle per-market USDC. (Or trigger manually if the keeper is off.)

4. Once that USDC lands in the global vault, `admin-withdraw.ts` can pull it out.

---

## Controlling supply

### Adjust OI cap

Tighten or widen the cap for a market as liquidity changes.

```bash
SYMBOL=NVDA OI_CAP=500000 bun run scripts/update-clp-oi-cap.ts
```

The cap is hard — once `current_oi == oi_cap`, further mints are rejected by the mint-redeem program. Redeems remain allowed.

### Recommend a cap based on pool depth

```bash
SYMBOL=NVDA bun run scripts/recommend-oi-cap.ts
```

Reads current pool depth and suggests an OI cap that leaves enough liquidity for orderly redemption under stress.

### Pause a market

If a market needs to be killed (oracle compromised, bad config, etc.) use the governance pause path. See `docs/DEPLOYMENT.md` for the governance flow. Existing positions can still redeem, but new mints are blocked.

---

## Reading protocol state

### Single-shot status dump

```bash
bun run scripts/protocol-status.ts
```

Prints: global vault balance, cumulative funding, fees accumulated, per-market allocations, OI utilisation, risk state, per-market vault idle balance, DLMM pool depth for L and S.

Use this before and after major ops to confirm things went as expected.

---

## Keeper checklist

The keeper doesn't need user deposits to do its job. It does need:

- [x] Admin keypair at `WALLET_PATH` (used to sign `allocate_to_market`, `vault_mint_pairs`, `clp_open_meteora_position`, `deposit_profit`)
- [x] Some SOL in the admin keypair for tx fees
- [x] USDC in the global vault above the `liquidity_seed_threshold` (default $1k)
- [x] Pyth Hermes reachable
- [x] Solana RPC reachable (prefer a private RPC for production)

See `keeper/README.md` for startup flags.

---

## Day-zero checklist (cold start)

1. Deploy programs: `anchor deploy --provider.cluster devnet` (or use the script in `docs/DEPLOYMENT.md`).
2. Initialise the global CLP: `bun run scripts/initialize-global-clp.ts`.
3. Launch first market (see [Deploying a new market](#deploying-a-new-market) above).
4. Fund the global vault.
5. Start the keeper.
6. Watch `protocol-status.ts` for the first 10 minutes to confirm allocation happens.

---

## Recovery playbook

Things that go wrong and how to fix them. All of these have happened on devnet at least once.

### Position has zombied (Meteora won't close it)

Symptom: keeper logs `close_position2` failing with `NonEmptyPosition (6030)` after repeated `remove_liquidity` attempts. Meteora's position has non-zero `liq_share` slots but all corresponding bin arrays show `liq_supply == 0` — the tokens have already been extracted.

Fix: the keeper's rebalancer detects zombies via `is_position_zombie` and skips the close step automatically. The position account's rent (~0.05 SOL) is sacrificed; all tokens are already in the CLP vault.

If the keeper's CLP-state pointer is lost (position pubkey set to default while the Meteora account still exists), the keeper scans for orphans via `find_positions_in_pool`. To recover manually:

```bash
MARKET=QQQ SIDE=1 bun run scripts/recover-orphaned-position.ts
```

This calls `clp_set_meteora_position` to restore the stored pubkey; the keeper drains it on the next tick.

### Pool is wedged, zombie positions trapping all liquidity

Symptom: a DLMM pool's active price is far from NAV (50%+ divergence), small swaps fail with `AccountNotEnoughKeys` or `ExceededAmountSlippageTolerance`, and you can see many CLP-owned positions in the pool via `getProgramAccounts`.

Fix: drain everything, then optionally replace the pool.

```bash
# Sweep every CLP-owned position (handles zombies best-effort)
MARKET=QQQ bun run scripts/drain-all-orphaned-positions.ts
```

If the pool itself is trapped (stuck zombies plus permanent bin misconfiguration) or you want to migrate to a different fee tier, drain everything first then create replacements:

```bash
# Drain all CLP-owned positions
MARKET=QQQ bun run scripts/drain-all-orphaned-positions.ts

# Redeem the synth tokens left in CLP PDA ATAs back to cUSDC
MARKET=QQQ bun run scripts/redeem-clp-synths.ts

# Create replacement pools at the desired fee tier (one per side)
MARKET=QQQ SIDE=long  BIN_STEP=10 BASE_FACTOR=10000 POOL_PRICE=<L_NAV> \
    bun run scripts/create-meteora-pool.ts
MARKET=QQQ SIDE=short BIN_STEP=10 BASE_FACTOR=10000 POOL_PRICE=<S_NAV> \
    bun run scripts/create-meteora-pool.ts

# Wire the new pools into CLP state (auto-clears stale position pointers
# via clear_meteora_positions on next keeper boot)
MARKET=QQQ LONG_POOL=<new_long> SHORT_POOL=<new_short> \
    bun run scripts/configure-meteora-pools.ts

# Optional: explicit clear of stored position pubkeys
MARKET=QQQ bun run scripts/clear-meteora-positions.ts

# Restart keeper — the seeder picks up the new pools and dlmm.rs's
# recovery path opens fresh positions on the next 600s tick.
```

`scripts/create-new-short-pool.ts` is a thin shim that pre-dates the unified script; both still work but `create-meteora-pool.ts` is the canonical entry.

### Keeper wallet has stranded synth tokens

Symptom: keeper wallet shows non-zero L or S balances after a crash or bad arb.

Fix: the keeper auto-sweeps on boot via `sweep_residual_pairs` — handles both paired and asymmetric residue (calls `keeper_redeem_single` for one-sided leftovers, `redeem_paired` when both sides have stock). If the keeper can't run, the same logic exists for the CLP PDA's ATAs in `redeem-clp-synths.ts`:

```bash
MARKET=QQQ bun run scripts/redeem-clp-synths.ts
```

For stranded synths in the keeper's own wallet (rather than CLP PDA), the simplest path is to start the keeper — boot sweep handles it. Manual recovery requires a small script that calls `redeem_paired` (paired) or `keeper_redeem_single` (asymmetric) signed with the keeper authority.

### Pool prices drifted far from NAV with no convergence

Symptom: NAV is at X, pools sit at Y for hours or days, the keeper's paired arb never triggers because `profit_fraction` is below fees.

Diagnosis:
1. **Per-side depegs diverge but combined spread is tight?** Paired arb can only close the combined spread. On mainnet external flow corrects per-side; on devnet with zero external traders, this is expected. The DLMM rebalancer's fee-weighted centroid helps passively correct over many cycles but it's slow.
2. **Pool depth exhausted?** Check `spl-token balance <reserve_y_account>`. If the Y-side (cUSDC) depth is near zero, the keeper's sell path has nowhere to unload. Seed more capital: `LIQUIDITY_SEED_FRACTION_BPS=8000` and restart keeper.
3. **Zombie position starvation?** See previous playbook.

Manual push toward NAV (operator intervention, does not route through CLP):

```bash
MARKET=QQQ \
MINT_AMOUNT=5000 \
BUY_QQQL_AMOUNT=5000 \
SELL_QQQS_AMOUNT=10 \
bun run scripts/fix-pool-prices.ts
```

Use sparingly — this is a manual override, not a maintenance tool.

### Fresh seed at specific bins

Sometimes you want to rebuild from scratch with specific position anchors (e.g. to test fee-weighted centering with a known fee history).

```bash
MARKET=QQQ \
DEPLOY_USDC=50000 \
BIN_RADIUS=34 \
L_TARGET_BIN=2587 \
S_TARGET_BIN=2357 \
bun run scripts/seed-at-nav.ts
```

---

## Scripts reference

Most scripts are self-documenting at the top of the file. Grouped by purpose:

### Lifecycle / setup
- `initialize-global-clp.ts` — one-shot create of the global CLP PDA
- `initialize-clp.ts` — per-market CLP init
- `initialize-markets.ts` — create a synthetic market + mints + oracle config
- `create-clp-token-atas.ts` — create L/S/cUSDC ATAs for the CLP PDA
- `configure-meteora-pools.ts` — wire Meteora pool addresses into CLP state
- `create-synthetic-market.ts` — full market init flow

### Capital / funding
- `admin-fund.ts` — clean path (`admin_fund` ix, advances `total_deposits`)
- `admin-fund-global-vault.ts` — devnet shortcut (cranks faucet)
- `admin-withdraw.ts` — withdraw idle global vault balance
- `mint-cusdc-to-keeper.ts` — devnet helper to fund the keeper wallet directly
- `faucet-drip.ts` — one-off cUSDC drip (respects cooldown)

### Meteora / LP
- `seed-meteora-liquidity.ts` — create pools + seed initial liquidity
- `seed-meteora-exact.ts` — seed without creating (reuse existing pools)
- `seed-at-nav.ts` — seed at precise bin targets
- `reseed-two-sided.ts` — close and re-seed with two-sided balance
- `create-meteora-pool.ts` — canonical pool creator; `SIDE=long|short BIN_STEP=N BASE_FACTOR=N POOL_PRICE=N`. Updates market-addresses.json automatically
- `create-new-short-pool.ts` — legacy shim, predates the unified `create-meteora-pool.ts`; still works for short side only
- `withdraw-meteora-liquidity.ts` — close positions
- `reclaim-pool-liquidity.ts` — reclaim tokens from a specific position
- `open-position-manual.ts` — open a position with explicit bin range
- `clear-meteora-positions.ts` — clear position refs from CLP state

### Recovery / admin
- `drain-all-orphaned-positions.ts` — drain every CLP-owned position in both pools (set+remove+close + return cUSDC to global)
- `recover-orphaned-position.ts` — point CLP state at an existing orphan
- `redeem-clp-synths.ts` — redeem CLP PDA's L/S balances back to cUSDC and return to global vault. Run after `drain-all-orphaned-positions.ts` to recover the synth tokens that drain leaves stranded in CLP ATAs
- `migrate-keeper-positions.ts` — one-shot migration helper for V1 → V2a
- `fix-pool-prices.ts` — manual push toward NAV (direct swaps, not CLP)

### Status / ops
- `protocol-status.ts` — single-shot state dump
- `find-pools.ts` — locate Meteora pools by token pair
- `recommend-oi-cap.ts` — suggest an OI cap from current pool depth
- `force-price.ts` — devnet-only oracle override for testing
- `update-clp-oi-cap.ts` — adjust OI cap
- `update-keeper-authority.ts` — rotate the keeper authority pubkey
- `update-fee-recipient.ts` — rotate fee recipient
- `update-market-oracle.ts` — swap oracle feed for a market
- `configure-inventory-budget.ts` — adjust risk hard/soft bounds
- `configure-yield-strategy.ts` — idle-capital yield params (yield module is scaffolded, off by default)
- `configure-liquidity-seeder.ts` — adjust seeder threshold/cadence

### Token metadata / one-offs
- `add-token-metadata.ts`, `attach-token-metadata.ts`, `update-token-metadata.ts`, `create-cusdc-metadata.ts`, `upload-token-icons.ts`
- `burn-tokens.ts` — burn from admin wallet (emergency supply adjustment)
- `inject-cusdc.ts` — devnet-only cUSDC mint helper

---

## Things that do **not** exist anymore

- **No user vault deposits.** The `/yield` and `/earn` routes redirect to `/pools`. The `global_deposit`/`global_withdraw` instructions have been removed from the CLP program.
- **No LP shares.** `GlobalLPAccount` was deleted. The protocol is the sole LP; there is no share accounting.
- **No 3-day lockup.** Admin can fund/withdraw at will (withdraw is still gated by idle balance only).

If you see references to these in old code or docs, they're stale.

---

## Future: opening the vault back up

The removal was intentional — it matched the MVP scope of "keeper as LP seeder". If you later want third-party LPs, the git history has the full share/deposit/lockup implementation. The keeper-authorised instructions (`allocate_to_market`, `vault_mint_pairs`, etc.) were preserved exactly, so re-enabling users mostly means restoring the three instructions and the `GlobalLPAccount` state.
