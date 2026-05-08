# Local Staging — Full Validator Setup

A fully isolated copy of the Continuum stack running on
`solana-test-validator` for keeper development without touching devnet
or production. Hermes provides live prices; everything else (mints,
markets, CLP, Meteora pools) lives on a local validator.

This is **Tier 4** of the staging design from `keeper/.env.local.example` —
zero shared state with devnet, full RPC isolation.

## When to use it

- Iterating on `keeper/src/` without conflicting with the live keeper.
- Testing destructive flows (forced position closes, hard re-allocations,
  recovery from oracle failure) where you don't want to disrupt prod.
- Reproducing a bug where you need to control market state precisely.

For minor config tweaks that don't need isolation, just point a second
keeper instance at devnet with `INSTANCE_LABEL=staging-devnet` and a
distinct keeper signer wallet — much lighter.

## Prereqs

- `solana-test-validator` on PATH (ships with Solana CLI).
- `anchor build` already run so `target/deploy/*.so` exists.
- `~/.config/solana/id.json` set up with a funded admin wallet (script
  airdrops on the local cluster, but expects the keypair to exist).
- `bun`, `jq`, `solana-keygen`, `spl-token` on PATH.

## Quickstart

```bash
# Terminal 1 — start the validator (runs in foreground)
bash scripts/start-local-validator.sh

# Terminal 2 — bootstrap state (cUSDC, faucet, Global CLP, SPY+XAU markets)
bash scripts/setup-local-staging.sh

# Terminal 3 — run the keeper against this validator
cd keeper && KEEPER_ENV_FILE=.env.local ./target/release/keeper
```

Watch the dashboard at `http://127.0.0.1:8485` (port 8485 is the
local-staging convention; production uses 8484).

## What the scripts do

### `start-local-validator.sh`

1. Confirms `target/deploy/*.so` exists for mint_redeem, clp, oracle, faucet, registry.
2. Wipes `test-ledger/` (use `--keep` to preserve state across restarts).
3. Boots `solana-test-validator` with:
   - All five Continuum programs loaded at their canonical pubkeys (same as devnet/Anchor.toml).
   - Meteora DLMM program **cloned from devnet** (`LBUZK...`).
   - Three Meteora preset parameter PDAs cloned (binStep 10/20/25 with baseFactor 10000).
   - SPL Token Metadata program cloned.
   - Pyth Pull receiver program cloned (oracle program account-validates it on init).
4. Listens on `127.0.0.1:8899` (RPC) and `127.0.0.1:9900` (faucet).

### `setup-local-staging.sh`

Idempotent orchestrator. Reuses existing TS scripts via `MARKET_ADDRESSES_PATH`
override so the local market registry lives at
`frontend/lib/market-addresses.local.json` (gitignored).

Steps:
1. Wait for validator at `RPC_URL` (default localhost).
2. Generate or reuse a fresh keeper keypair at `scripts/.local-keeper-keypair.json` (gitignored).
3. Airdrop SOL to admin + keeper from the local faucet.
4. Create cUSDC mint (decimals=6) at `scripts/.local-cusdc-mint.json` (gitignored). Mint authority transferred to faucet PDA in step 5.
5. Run `scripts/initialize-faucet.ts` — same flow as devnet faucet bootstrap.
6. Run `scripts/initialize-global-clp.ts`.
7. For each market in `$MARKETS` (default `SPY,XAU`):
   - Fetch live price from Hermes.
   - Run `create-synthetic-market.ts` with `KEEPER_AUTHORITY` set to the
     local keeper pubkey (so the keeper this script writes can sign right
     out of the gate — no manual `update-keeper-authority` step).
   - Run `initialize-clp.ts`, `create-clp-token-atas.ts`,
     `mint-liquidity-inventory.ts`, `seed-meteora-liquidity.ts`,
     `configure-meteora-pools.ts`.
8. Top up Global CLP with `$GLOBAL_CLP_FUND_USD` cUSDC via `admin-fund-global-vault.ts`.
9. Generate `keeper/.env.local` with the local addresses + keeper keypair JSON inlined.

## Tuning knobs (env vars on `setup-local-staging.sh`)

| Var | Default | Purpose |
|---|---|---|
| `RPC_URL` | `http://127.0.0.1:8899` | Where the validator is listening. |
| `MARKETS` | `SPY,XAU` | Comma-separated symbols. Add `QQQ,NVDA` etc. to test multi-market interactions. |
| `PAIRS_PER_MARKET` | `50` | Paired-mint count → governs Meteora seed depth. |
| `GLOBAL_CLP_FUND_USD` | `100000` | Funds the seeder so it has something to deploy. |
| `KEEPER_WALLET` | `scripts/.local-keeper-keypair.json` | Override to test multi-keeper HA scenarios. |

## Gotchas

- **Hermes calls go out over the public internet** — the validator is
  isolated but the keeper's price source is not. Anywhere you'd add a
  mock-Pyth shim, you can simply unplug your wifi to simulate oracle
  outage instead.
- **Bin arrays do not get cloned.** They're created lazily on first
  liquidity. If you want to test "rebalance into new bin array territory"
  rent costs, you have to actually trade the local pool that far.
- **Pyth Pull receiver is cloned, but no real Pyth feed accounts are.**
  The oracle program's `upsert_oracle_feed` validates the pyth_oracle
  account at init; on local validator that account doesn't exist for
  arbitrary symbols. Setup script passes `ORACLE_FEEDS_JSON='[]'` to
  skip feed registration entirely. Keeper falls back to Hermes via
  ProxyMode (same as devnet).
- **Closing the validator wipes everything.** Use `--keep` on
  `start-local-validator.sh` if you want the ledger to persist between
  runs. Useful for debugging long-tail keeper state issues.
- **Port 8485 vs 8484** — the local keeper dashboards on a different
  port to coexist with a production keeper running locally.

## Tearing down

```bash
# Kill the keeper (Ctrl-C in its terminal)
# Kill the validator (Ctrl-C in its terminal)
# Optional — wipe generated state
rm scripts/.local-* keeper/.env.local frontend/lib/market-addresses.local.json
rm -rf test-ledger
```

## Related

- `keeper/.env.local.example` — annotated template for what the
  generated file contains.
- `keeper/src/main.rs` — `KEEPER_ENV_FILE` env var selects which dotenv
  file to load (default `.env`, override to `.env.local`).
- `scripts/lib/market-addresses.ts` — `MARKET_ADDRESSES_PATH` env var
  overrides which JSON the helper scripts read/write, so `setup-local-staging.sh`
  doesn't clobber the devnet registry.
