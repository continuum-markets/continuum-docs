# Deployment Guide

Cold-start deployment of Continuum.Markets to Solana devnet (or mainnet-beta with the analogous cluster flag). For day-to-day market and capital operations on a deployed system, see [`docs/OPERATIONS.md`](./OPERATIONS.md).

## Prerequisites

- Solana CLI 1.18+
- Anchor CLI 0.32+ (matches `Anchor.toml`)
- Rust 1.75+
- Node.js 18+ and pnpm (or Bun for scripts — `bun run scripts/<name>.ts`)
- A funded Solana deployer wallet (~5 SOL on devnet, more on mainnet)

## 1. Configure the deployer wallet

```bash
# Set cluster
solana config set --url devnet  # or mainnet-beta

# Create or import the deployer keypair
solana-keygen new -o ~/.config/solana/id.json

# Fund (devnet)
solana airdrop 5

# Verify
solana balance
solana address
```

The same keypair signs program deploys, becomes the CLP authority, and signs admin scripts. For mainnet it should be a multisig (Squads recommended) — temporarily use a hot key for first deploy, then transfer upgrade authority to the multisig.

## 2. Build programs

```bash
# From repo root
anchor build
ls -la target/deploy/*.so
```

You should see five `.so` files:

- `mint_redeem.so` — paired mint/redeem + keeper-only single-side ixns
- `oracle.so` — Pyth/Hermes feed + TWAP + risk state
- `clp.so` — Capital & Liquidity Provider (Meteora DLMM proxy)
- `governance.so` — scaffolded; not yet wired to execution paths
- `faucet.so` — devnet-only cUSDC drip
- `registry.so` — market registry

## 3. Verify program IDs match `Anchor.toml`

The `[programs.devnet]` block in `Anchor.toml` pins the upgrade-authority pubkeys. Confirm:

```bash
anchor keys list
```

Output should match the `Anchor.toml` entries. If you generated fresh keypairs (e.g. for a new cluster), update both `Anchor.toml` and any references in `frontend/lib/idl/` + `scripts/lib/market-addresses.ts` (`CURRENT_PROGRAM_IDS`).

## 4. Deploy

```bash
# Devnet
anchor deploy --provider.cluster devnet

# Mainnet
anchor deploy --provider.cluster mainnet
```

This deploys all programs in one shot. For program-by-program deploys:

```bash
solana program deploy target/deploy/mint_redeem.so \
  --program-id <pubkey-from-Anchor.toml> \
  --url <cluster-url>
```

Subsequent upgrades reuse the same program-id and require the upgrade authority's signature.

## 5. Initialize global state

The CLP program needs a one-shot `initialize_global_clp` call before any per-market init:

```bash
bun run scripts/initialize-global-clp.ts
```

This creates the `global_clp` PDA and its cUSDC vault.

## 6. Devnet-only: create cUSDC mint + faucet

```bash
# Mint authority is the deployer wallet
bun run scripts/create-cusdc-metadata.ts
bun run scripts/initialize-faucet.ts
```

`B1c5xBYkp7AAemYhcu4VuH4CU4sPJDDuG2iuv6ts38uE` is the active devnet cUSDC mint; reuse it unless redeploying from scratch. Mainnet uses real USDC (`EPjFW...`).

## 7. Initialize the first market

Follow the [Deploying a new market](./OPERATIONS.md#deploying-a-new-market) flow in OPERATIONS.md. Briefly:

```bash
SYMBOL=QQQ INITIAL_PRICE=663 bun run scripts/create-synthetic-market.ts
SYMBOL=QQQ bun run scripts/initialize-clp.ts
SYMBOL=QQQ OI_CAP=250000 bun run scripts/update-clp-oi-cap.ts
SYMBOL=QQQ bun run scripts/create-clp-token-atas.ts

# Create both Meteora pools at the desired fee tier
MARKET=QQQ SIDE=long  BIN_STEP=10 BASE_FACTOR=10000 POOL_PRICE=663.95 \
  bun run scripts/create-meteora-pool.ts
MARKET=QQQ SIDE=short BIN_STEP=10 BASE_FACTOR=10000 POOL_PRICE=347.02 \
  bun run scripts/create-meteora-pool.ts

# Wire pool addresses into CLP state
MARKET=QQQ LONG_POOL=<long-pool> SHORT_POOL=<short-pool> \
  bun run scripts/configure-meteora-pools.ts
```

Pool fee tier is set per market by `bin_step × base_factor / 1e6`. Index ETFs run at `binStep=10` (0.10% fee); volatile equities at 20-25 (0.20-0.25%). See OPERATIONS.md fee tier table.

## 8. Fund the global vault

```bash
AMOUNT=200000 bun run scripts/admin-fund-global-vault.ts  # devnet shortcut
# or
AMOUNT=200000 bun run scripts/admin-fund.ts               # mainnet (admin holds USDC)
```

The keeper's seeder allocates from this vault into per-market positions on every 600s tick.

## 9. Start the keeper

```bash
cd keeper
cp .env.example .env

# Edit .env to set:
#   DEVNET_RPC_URL or MAINNET_RPC_URL
#   USDC_MINT (devnet: B1c5xBYkp7AAemYhcu4VuH4CU4sPJDDuG2iuv6ts38uE — required, falls back to mainnet USDC otherwise)
#   ACTIVE_MARKETS=QQQ,...
#   KEEPER_KEYPAIR_JSON or KEEPER_KEYPAIR
#   CLP_AUTHORITY_KEYPAIR_JSON

cargo build --release
./target/release/keeper
```

Dashboard at `http://localhost:8484`. Within ~10 minutes you should see the seeder allocate from global vault, mint pairs, open positions, and the arb bot start scanning.

## 10. Verify

```bash
bun run scripts/protocol-status.ts
```

Prints global vault balance, per-market allocations, OI utilization, pool depths. Useful before/after major ops.

## Troubleshooting

### Program deploy fails with "insufficient funds"

The deploy buffer requires ~1 SOL of rent during deploy. Add SOL to the deployer:

```bash
solana airdrop 5  # devnet only
```

### Program-id mismatch

If `anchor keys list` shows different IDs than `Anchor.toml`, re-sync:

```bash
anchor keys sync
anchor build
anchor deploy
```

Then update `frontend/lib/idl/*` and `scripts/lib/market-addresses.ts` to match.

### Oracle reads zero / stale

The oracle program is pull-based (the keeper pushes Hermes observations every 15s). On a fresh deploy, allow ~30s after starting the keeper before any user-facing mint/redeem will succeed — the market gates on `last_oracle_update` freshness.

### Keeper seeder skipping silently

Check `USDC_MINT` is set in `keeper/.env`. The default is mainnet USDC, which on devnet derives a non-existent ATA → balance reads as 0 → seeder skips below threshold. See the env-var fallback note in `keeper/README.md` config table.

## Mainnet checklist

1. Security audit (Sec3, OtterSec, Neodyme — at least one)
2. ≥4 weeks of devnet testing under realistic load
3. Bug bounty program live
4. Upgrade authority moved to a Squads multisig (≥3-of-5)
5. CLP authority moved to a separate Squads multisig (faster turnaround for keeper rotation than program upgrade)
6. Conservative initial OI caps per market
7. Pyth feeds confirmed on mainnet for every asset
8. Gradual rollout — invite-only first market, expand as load is observed
