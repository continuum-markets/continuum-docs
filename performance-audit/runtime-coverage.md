# Keeper and on-chain performance audit, 2026-10-09

## Evidence and limits

Inventory: 116 on-chain instruction entry points (including the new oracle account resize), 23 pre-existing keeper background tasks (24 with the new read-only subscription), and 13 keeper HTTP routes. Every current program entrypoint has a signed local SBF fixture; see the final per-instruction table for outcomes and timing. Public wallet actions also have live devnet receipts and three real React UI flows. Keeper task inventory is broader than measured cycle coverage: populated production handlers, complete background cycles and post-deployment improvements remain explicit gaps. No production admin keys or live keeper process were used.

Native math timings are not SBF compute units or transaction confirmation latency. Initial live observations below are baseline measurements. The authorized devnet rollout subsequently deployed a clean analytics-based CLP/Oracle release and completed 15 selected oracle resizes; see [rollout evidence](rollout/deployment.json). The deployed release excludes unrelated working-tree corporate-action changes: its separate 115-entrypoint, 835-transaction verification is in [release coverage](rollout/release-coverage.md). Its exact CLMM CU comparisons retain the same reported savings.

## Measured results

- Historical devnet XAU commit → terminal chain outcome: buy 9 seconds, sell 11 seconds, slippage refund 20 seconds (three samples, block-time resolution one second). Their eligible-order log → keeper-confirmation intervals were 1.874, 1.966 and 2.400 seconds. See `evidence/runtime-receipts.jsonl` and `evidence/keeper-live.json`.
- Keeper Railway resource samples: 61 over one hour, average 0.00183 CPU cores, average 27.9 MB memory and peak 41.0 MB. These samples do not indicate CPU saturation. Keeper has no public domain; these initial resource samples did not measure live HTTP timing. The later [rollout report](rollout/ROLLOUT.md) adds private-container HTTP reads, populated status responses and actual settlement-phase telemetry. All 13 routes were measured in a localhost empty-network startup fixture (20 warm samples each, medians 0.119–0.206 ms). Unknown-network handlers return JSON errors, not actual market data; telemetry routes return404. These timings do not cover populated state, upstream reads or loop contention.
- Controlled one-worker Tokio benchmark: a blocking 100 ms confirmation delayed an independent 10 ms timer. `block_in_place` restores timely timer progress; this is a synthetic scheduling test, not a deployed service measurement.
- Local websocket protocol fixture verifies both filtered subscriptions, a confirmed account event, immediate scheduler wakeup, monotonic observed-slot tracking, and cleanup of both subscriptions. Event request → wake measured 0.962 ms on localhost; network latency and the one-second scan coalescing cap remain in production.

### Authorized disposable-wallet devnet actions

These use the actual frontend prepare functions, signed-transaction validation and execute/confirm pipeline with an isolated devnet key. The RPC transport points directly at public devnet; timings exclude human wallet approval, the Next RPC proxy and React rendering. They are baseline measurements against deployed code, not after-deploy improvements. Each row is one sample; no percentile claims are made.

| Action | Total ms | Submitted → confirmed ms | Transaction CU |
|---|---:|---:|---:|
| Faucet cUSDC drip | 1,948.1 | 1,728.8 | 42,569 |
| Mint paired, $10 | 1,862.5 | 1,735.1 | 64,121 |
| Mint paired, $20 | 1,864.2 | 1,737.8 | 45,971 |
| Redeem paired | 1,865.8 | 1,756.6 | 41,001 |
| Pool buy long | 1,833.2 | 1,781.2 | 50,579 |
| Pool sell long | 11,864.4 | 1,770.3 | 50,555 |
| Pool buy short | 5,026.9 | 4,977.4 | 52,077 |
| Pool sell short | 1,795.2 | 1,743.4 | 52,305 |
| Donate $0.01 | 1,883.1 | 1,785.8 | 34,367 |
| Burn long tokens | 1,810.5 | 1,733.6 | 24,820 |
| Commit buy | 1,874.8 | 1,814.2 | 53,866 |

The 11.86-second long sell spent 10.06 seconds in preparation because public RPC returned 429 with a retry delay; pool preparation itself is pure PDA/instruction work, so this read is the pipeline's latest-blockhash request. The short buy spent 4.98 seconds confirming. Mint CU differs because the first operation creates token accounts. Preflight remains enabled, every submitted signature is reconciled, and uncertain writes are never automatically repeated.

Measured expected failures: faucet cooldown 54.5ms /16,125 simulated CU; deliberately impossible pool minOut 50.0ms /36,748CU; redeeming a just-minted $10 position122.5ms /25,106CU (fees leave output below the$10 minimum); early cancel 83.6ms /15,263CU (OrderNotExpired). A larger mint produced a successful redeem. The first CLI cancel attempt was blocked by browser-localStorage recovery protection before preparing or signing; its `unknown` row references the already-confirmed commit, not a new cancellation. The harness now durably saves public order metadata before releasing confirmed recovery rows. Evidence preserves that harness gap instead of treating it as a chain failure.

A new commit settled after 18  seconds by chain block times, with 41,057 CU. Keeper eligible log→confirmed log was 2.426 seconds; the earlier 15.6 seconds include discovery and oracle eligibility, which old telemetry cannot separate. The new per-stage telemetry makes that distinction observable after deployment. An expired cancellation success remains unmeasured because both fresh orders were settled by the live keeper; the audit did not disable a service to manufacture expiry. See `evidence/devnet-wallet-actions.jsonl`.

### Supporting services

- Faucet service health: cold first request 8,871.6ms while Railway had reported the service sleeping; warm requests 248.5 and 272.7ms. Historical service logs show two successful airdrops in 48,653 and 32,995ms (September 17, not new executions). The worker defaults to one concurrent request and a shared two-grant/eight-hour budget. Synchronous child processes can block its entire Node event loop for up to the 20-second subprocess timeout (`faucet-service/server.js:98`). No airdrop/probe or CAPTCHA automation was invoked. Health reads alone cannot benchmark that interactive workflow.
- Discord REST guild reads: 379.7, 231.9, 231.7ms, allHTTP 200, no messages or mutations. This repository contains management/snapshot scripts, not a continuously running Discord interaction bot. The snapshot script walks channel/message history serially and accumulates full per-channel history before writing. The helper has a 30-second request deadline and respects 429 retry_after; snapshot runtime scales with history and is not represented by a single guild read. Keeper alert posts have a 10-second HTTP timeout and retry on later monitoring cycles, with per-wallet cooldowns. Alert delivery itself was not exercised because this audit did not authorize messages to others.

Read-only supporting evidence: `evidence/supporting-services.json`. Faucet and Discord payloads/credentials were not written to the report.

### Native pool math, median of five rounds

Compiled with the production profile’s `opt-level=z` and overflow checks enabled (host ARM64, not SBF). Each round contains 25,000 calls. Exact frozen-reference comparison covers 53,750 identical inputs, four functions, and both mutated reserve arrays.

| Workload | Action | Before ns/call | After ns/call | Speedup |
|---|---|---:|---:|---:|
| single_bin | `quote_buy` | 17.3 | 15.1 | 1.15× |
| single_bin | `quote_sell` | 18.6 | 15.7 | 1.19× |
| single_bin | `execute_buy` | 21.9 | 24.6 | 0.89× |
| single_bin | `execute_sell` | 18.6 | 16.5 | 1.13× |
| full_16 | `quote_buy` | 556.1 | 108.5 | 5.13× |
| full_16 | `quote_sell` | 537.5 | 116.2 | 4.62× |
| full_16 | `execute_buy` | 587.0 | 118.1 | 4.97× |
| full_16 | `execute_sell` | 572.4 | 136.7 | 4.19× |
| sparse_16 | `quote_buy` | 224.2 | 88.8 | 2.53× |
| sparse_16 | `quote_sell` | 211.5 | 102.1 | 2.07× |
| sparse_16 | `execute_buy` | 227.0 | 88.5 | 2.57× |
| sparse_16 | `execute_sell` | 213.8 | 104.2 | 2.05× |

## Implemented changes

1. `programs/clp/src/clmm.rs`: carries the exact u128 price recurrence through bins. Full 16-bin walks perform 16 price steps instead of 136. Per-step integer rounding, saturation, cast placement, sparse-bin indexing, quotes and mutated reserves match the old implementation exactly. No float approximation or changed financial guards.
2. `keeper/src/commit_settler.rs` and `keeper/src/oracle.rs`: synchronous RPC/confirmation boundaries, including the market-readiness monitor in main.rs, yield the Tokio worker with `block_in_place`. Financial execution stays sequential; transaction journaling and unknown-outcome safeguards remain intact.
3. `keeper/src/settlement_subscription.rs` and `settlement_wakeup.rs`: confirmed Order and Market changes wake authoritative HTTP scans. Pending Order writes always wake; Market writes wake only while pending work exists. Notifications coalesce, scans start at most once per second, and websocket observed slots fence HTTP reads with minContextSlot. Existing 20-second idle / 5-second pending fallback stays active. Connect/subscribe/unsubscribe/shutdown have deadlines, transient disconnects back off with jitter, permanent rejection falls back to polling, and reconnect requests one recovery scan.
4. Settlement telemetry records scan trigger/duration, per-order oracle eligibility, commit/oracle timestamps, eligibility-to-submission and submission-to-confirmation durations, confirmation success and commit age. This separates first discovery, the wait for a newer oracle push, provisioning/queue time, and confirmation after deployment. `commit_age_ms` is based on the on-chain second-resolution creation timestamp and should not be treated as subsecond end-to-end accuracy. Notification payloads never supply prices, balances or instructions.
5. `keeper/src/data_collector.rs` / `jsonl_tail.rs`: R2 tails walk listed keys newest-first and stop once the selected suffix is complete, preserving ordering, whitespace, malformed-line counting and cross-object fragments. A linear chunk accumulator avoids repeated prefix copying. Missing requested batches produce a warned read gap. Local whole-file UTF-8 validation remains, with the blocking read moved off the async runtime. A 100,000-record fixture returns exactly the same 500 rows using 6 modeled GETs/34,140bytes instead of 1,000 GETs/5,679,000bytes; in-memory median selection is 1.884ms→0.040ms (network excluded).

## Remaining bottlenecks and next measurements

- `keeper/src/oracle.rs:476` / `config.rs:515`: 15-second oracle push cadence is a genuine eligibility bound for commit orders. Faster settlement must still require a TWAP strictly newer than the commit. The oracle also treats 60 samples × 15s as a 15-minute risk window; simply pushing more often changes financial inputs. This audit does not weaken that rule or freshness, confidence, slippage, window or journal safeguards.
- R2 tail/download listings now follow continuation tokens with bounded retries, deadlines and size limits. Malformed, incomplete, repeated or unavailable pages fail explicitly. Bucket-wide index preview remains intentionally capped. Actual R2 network before/after timing is unmeasured; the reported GET/byte reduction is a local fixture model. Local files still validate all UTF-8 bytes and raw downloads intentionally require all rows.
- Legacy Meteora, inventory, yield and rebalance paths contain further synchronous work. The changed readiness/oracle/settlement paths are not a claim that all keeper blocking I/O has been removed.
- `keeper/src/clmm_pool.rs:1634`: 30-second pool maintenance cadence includes serial financial operations. Historical startup logs span ~100–145 seconds across 14 markets. This is observation of startup duration, not a typical-cycle benchmark; do not parallelize sends behind the journal without a concurrency design.
- Frontend generated admin clients have separate IDL drift: `initializeOracleConfig` still encodes an extra `marketId` argument absent from current Rust. No call site outside generated exports was found in app/components/lib/scripts. Fresh isolated IDLs drive the fixtures; unrelated clients were not regenerated. The new resize entrypoint requires the reviewed fresh oracle IDL/operator transaction, not this stale initializer.
- `Cargo.toml:24`: program release uses size optimization. Switching to throughput optimization needs SBF binary size plus per-instruction CU evidence; native timing alone cannot justify that change.

## Controlled SBF execution and correctness regressions

Fresh source builds use platform-tools v1.45, SBFv2 and the unchanged production release profile inside `/tmp/continuum-performance-sbf`. LiteSVM 1.4.1 executes signed transactions with clock control, the real devnet SPL Token, Metaplex and Meteora executables, and fresh local account fixtures. No validator, deployment or network transaction submission is involved. Every measured row stores source/binary hashes, transaction input/state hashes, CU, host time, logs, outcomes and actual inner instruction bytes. Five identical-input replays are retained; builds verify the product source manifest before and after execution. Existing concurrent business-event edits were preserved in both CLMM variants.

The eight CLMM comparisons execute identical signed transactions against identical snapshots and compare every writable account byte and lamport exactly. Full 16-bin buy uses 108,575→75,346 CU (30.60% lower), sell 104,099→75,102 CU (27.86% lower). One-bin savings are negligible, as expected. The raw `clmm-sbf.jsonl` proves actual VM CU separately from host-only math timings.

The broader fixtures exposed two correctness failures that prevent responsiveness entirely:

- Wide internal Meteora helper calls corrupted trailing i32 values under the actual SBF compiler. An outer range of zero became an invalid negative inner bin index; after fixing position initialization the same issue surfaced in liquidity parameters. Immutable borrowed parameter structs preserve exact values across all four helpers. Actual external Meteora initialization/add/remove/claim/close CPIs now succeed, with negative lower-bound and full amount/range byte assertions. Public external instruction layouts, math and guards are unchanged. See `evidence/cpi-abi-regression.json`.
- `OracleConfig::space()` counted each u128 TWAP accumulator as eight bytes. Accounts were 16 bytes short, causing the 60th observation to fail serialization. Allocation now uses derived `INIT_SPACE`. The additive `realloc_oracle_config` instruction is PDA-bound, admin-only, rent-funded, byte-preserving and never shrinks. Regression scenarios cover wrong authority, current size, legacy size, the 60th/61st observations and a full ring. **Deployment alone does not resize existing accounts. The subsequent authorized devnet rollout upgraded Oracle and finalized all 15 reviewed admin resizes; four excluded accounts remain byte-identical.** All 60 observations and risk math are retained. See `evidence/oracle-ring-regression.json` and [the migration runbook](oracle-migration.md). The read-only devnet size audit found 18 configs with the known 1,765-byte size and one 1,733-byte older schema. A subsequent read-only check through the app’s existing configured devnet RPC verified current-layout decoding and PDA+bump for all 18 known-size configs at slot 509010528. Fifteen also match the owning Market authority (market verification slot 509010581); three reference missing Market accounts and are excluded pending intended-status review. The 1,733-byte schema is also excluded. These 15 were the verified candidates and are now migrated/finalized in the authorized devnet rollout; the four exclusions remain unchanged. See `evidence/oracle-readiness-proxy-verified.json` and `evidence/oracle-migration-candidates.csv`. Seventeen known-size accounts have zero observations and one has three, so no live ring-limit outage is claimed.

Reproduction: install pinned LiteSVM 1.4.1 in the isolated `/tmp/continuum-performance-svm` directory with `npm install --ignore-scripts`; obtain the read-only snapshots/executables using `frontend/scripts/performance-svm.ts`; then run `prepare-sbf-workspace.py`, sequential `build-sbf.py` / `build-fixture-idl.py` for each program (oracle/faucet use `--devnet`), and both fixture runners with `PERFORMANCE_SAMPLE=1..5`. `scripts/performance/summarize-svm.py` verifies replay completeness and emits the instruction table. Retained public build manifests are in `evidence/sbf-builds.json`; no private signer or browser wallet init data belongs in evidence.

## Validation and reproduction

```sh
bash scripts/performance/run-local.sh
python3 scripts/performance/runtime_reads.py
CARGO_BUILD_JOBS=1 cargo check --offline --manifest-path keeper/Cargo.toml
CARGO_BUILD_JOBS=1 cargo test --offline --manifest-path keeper/Cargo.toml --bin keeper --no-run
```

11 original pool tests pass. Nine R2 pagination/retry tests plus two standalone XML parsing tests pass; the live R2 roundtrip is skipped. 76 selected keeper financial/RPC/oracle/readiness/settlement/tail tests pass; the Bun-backed local websocket integration and13-route HTTP fixture pass separately. Test-build/check completed incrementally with one job. No financial transactions are created by those tests. All live receipt reads are cached by a hash of their actual request inputs, with fetch timestamps. The public RPC receipt query initially had one transient failure; the bounded retry/resume obtained all 22 sampled receipts.

To rerun the websocket integration: `cargo test --manifest-path keeper/Cargo.toml local_websocket_delivers_wakeups -- --ignored --nocapture`. Bun is needed only for this explicitly ignored localhost test.

## Keeper task coverage

Cadences below are source defaults, not measured runtime duration. Feature gates can disable tasks. The original main spawn function contains 23 task starts.

| Task | Default cadence | Benchmark status |
|---|---|---|
| Order analytics observer | 30s | source inspected |
| Financial journal reconciliation | 30s | financial safety tests |
| Oracle updater | 15s | production startup log + UpdateRiskState CU samples |
| Risk monitor | 10s | source inspected |
| Inventory monitor | 60s | source inspected |
| Market readiness | 60s | source inspected; worker blocking removed; readiness tests pass |
| Peer monitor | 30s | source inspected |
| Collateral balancer | 120s, minimum30s | source inspected |
| Swap inventory | 120s, minimum30s | source inspected |
| Native CLMM pool maintenance | 30s, minimum15s | SetPoolShape CU samples |
| Commit settler | 20s idle /5s pending | 5 real chain outcomes; new signed commit→settle18s;76 safety tests group |
| Devnet yield emulator | 3600s, minimum300s | source inspected |
| Legacy Meteora LP manager | 120s | legacy default disabled; no live benchmark |
| Legacy arb | 15s | legacy default disabled; no live benchmark |
| Legacy pool depth balancer | 60s, minimum15s | legacy default disabled; no live benchmark |
| SOL health | 300s, minimum15s | source inspected |
| Telemetry flusher | 30s, minimum5s | source inspected |
| Protocol snapshots | 300s | source inspected |
| Yield harvest | 300s | source inspected |
| Collateral yield | 300s | source inspected |
| Legacy liquidity seeder | 600s | legacy default disabled; no live benchmark |
| Devnet keeper top-up | 60s | gated; no live benchmark |
| Legacy LP fee claimer | 1800s | legacy default disabled; no live benchmark |
| New settlement subscriptions | event-driven; <=1 scan/s | local protocol +coalescing tests; live after-deploy comparison pending |

## Keeper HTTP coverage

| Route | Handler | Timing status |
|---|---|---|
| `/` | `index_handler` | localhost startup median 0.161ms /p95 0.215ms; HTTP 200 |
| `/api/networks` | `networks_handler` | localhost startup median 0.159ms /p95 0.235ms; HTTP 200 |
| `/api/status/:network` | `status_handler` | localhost startup median 0.145ms /p95 0.170ms; HTTP 200 unknown-network JSON |
| `/api/arb-scan/:network` | `arb_scan_handler` | localhost startup median 0.160ms /p95 0.220ms; HTTP 200 unknown-network JSON |
| `/api/pool-state/:network` | `pool_state_handler` | localhost startup median 0.151ms /p95 0.160ms; HTTP 200 unknown-network JSON |
| `/api/sol-health/:network` | `sol_health_handler` | localhost startup median 0.153ms /p95 0.317ms; HTTP 200 unknown-network JSON |
| `/api/data/:network` | `data_index_handler` | localhost startup median 0.206ms /p95 0.305ms; HTTP 404 no collector |
| `/api/data/:network/:stream` | `data_tail_handler` | localhost startup median 0.167ms /p95 0.212ms; HTTP 404 no collector |
| `/api/data/:network/:stream/download` | `data_download_handler` | localhost startup median 0.163ms /p95 0.199ms; HTTP 404 no collector |
| `/healthz` | `healthz_handler` | localhost startup median 0.119ms /p95 0.151ms; HTTP 200 |
| `/health` | `healthz_handler` | localhost startup median 0.127ms /p95 0.170ms; HTTP 200 |
| `/ping` | `ping_handler` | localhost startup median 0.135ms /p95 0.153ms; HTTP 200 |
| `/api/memory` | `memory_handler` | localhost startup median 0.192ms /p95 0.242ms; HTTP 200 |

## Every on-chain instruction

116 current entrypoints executed in 168 local scenarios, five identical-input replays each (840 signed transactions). All replays have stable input hashes and compute units. One intentionally deprecated getter rejects; `redeem_single_for_vault` is exercised through its required authorized CLP CPI. All other entries have successful executions.

CU ranges cover the successful local scenarios for each entry (or the expected deprecated rejection); they include transaction overhead and actual nested CPI work. The nested-only row reports its logged CPI CU and has no isolated host timing. Host medians are LiteSVM execution time on this Mac, excluding build, networking and confirmation. Local coverage does not establish live operation latency or exhaustive branch coverage.

| Program | Instruction | Source line | Result | CU range | Host median ms | Scenarios |
|---|---|---:|---|---:|---:|---:|
| clp | `absorb_loss` | 994 | success | 10013 | 0.132 | 1 |
| clp | `add_yield_to_buffer` | 961 | success | 12690 | 0.163 | 1 |
| clp | `admin_fund` | 114 | success | 11830 | 0.256 | 1 |
| clp | `admin_withdraw` | 147 | success | 21696 | 0.252 | 2 |
| clp | `allocate_to_market` | 235 | success | 25536 | 0.228 | 1 |
| clp | `cancel_swap` | 923 | success | 16055 | 0.187 | 2 |
| clp | `check_inventory_restrictions` | 1015 | success | 8239 | 0.110 | 1 |
| clp | `clear_meteora_positions` | 1756 | success | 14913 | 0.174 | 1 |
| clp | `clp_burn_excess` | 1511 | success | 53912 | 0.444 | 1 |
| clp | `clp_claim_lp_fees` | 2116 | success | 62056 | 0.610 | 1 |
| clp | `clp_close_meteora_position` | 2193 | success | 37362 | 0.277 | 1 |
| clp | `clp_fund_pool_synth` | 2611 | success | 24237 | 0.250 | 2 |
| clp | `clp_init_bin_array` | 1816 | success | 224501–269141 | 0.822 | 2 |
| clp | `clp_open_meteora_position` | 1853 | success | 119476 | 0.940 | 1 |
| clp | `clp_remove_meteora_liquidity` | 2046 | success | 85312 | 0.626 | 1 |
| clp | `clp_set_meteora_position` | 1776 | success | 18589 | 0.171 | 1 |
| clp | `clp_swap` | 517 | success | 34543–35203 | 0.343 | 2 |
| clp | `clp_topup_meteora_position` | 1977 | success | 101446 | 0.867 | 1 |
| clp | `collect_fees` | 1221 | success | 18694 | 0.185 | 1 |
| clp | `commit_swap` | 668 | success | 45401–48401 | 0.436 | 4 |
| clp | `configure_inventory_budget` | 444 | success | 13444 | 0.130 | 1 |
| clp | `configure_liquidity_seeder` | 1414 | success | 8177 | 0.101 | 1 |
| clp | `configure_meteora_pools` | 1742 | success | 20809 | 0.175 | 2 |
| clp | `configure_pool_geom` | 2314 | success | 12676 | 0.134 | 2 |
| clp | `configure_revenue_attribution` | 1320 | success | 11736 | 0.152 | 1 |
| clp | `configure_swap` | 472 | success | 14364 | 0.143 | 1 |
| clp | `configure_yield_strategy` | 1030 | success | 14068 | 0.132 | 1 |
| clp | `deposit_profit` | 1248 | success | 22378 | 0.200 | 1 |
| clp | `get_yield_status` | 1206 | success | 10317 | 0.179 | 1 |
| clp | `init_pool` | 2247 | success | 31195–31196 | 0.317 | 2 |
| clp | `init_revenue_attribution` | 1299 | success | 19826 | 0.197 | 1 |
| clp | `initialize_clp` | 329 | success | 29983 | 0.337 | 1 |
| clp | `initialize_global_clp` | 87 | success | 11944 | 0.199 | 1 |
| clp | `pool_swap` | 2365 | success | 44635–44650 | 0.378 | 4 |
| clp | `pool_sweep_synth` | 2565 | success | 25809 | 0.319 | 2 |
| clp | `pool_sweep_usdc` | 2518 | success | 25762 | 0.323 | 2 |
| clp | `realloc_clp` | 1371 | success | 7630–10433 | 0.152 | 2 |
| clp | `realloc_pool` | 2279 | success | 4178–7375 | 0.168 | 3 |
| clp | `record_deployment` | 1080 | success | 15632 | 0.158 | 1 |
| clp | `record_harvest` | 1163 | success | 20606 | 0.155 | 1 |
| clp | `record_withdrawal` | 1130 | success | 14814 | 0.142 | 1 |
| clp | `return_to_global` | 284 | success | 25859 | 0.209 | 1 |
| clp | `set_commit_spread` | 501 | success | 13009 | 0.131 | 1 |
| clp | `set_pool_shape` | 2329 | success | 20330 | 0.180 | 2 |
| clp | `settle_swap` | 757 | success | 38979–45491 | 0.390 | 6 |
| clp | `update_inventory` | 366 | success | 13853 | 0.132 | 1 |
| clp | `update_oi` | 1333 | success | 14326 | 0.122 | 1 |
| clp | `update_oi_cap` | 1352 | success | 13929 | 0.117 | 1 |
| clp | `vault_mint_pairs` | 1435 | success | 94271 | 0.786 | 1 |
| clp | `vault_record_liquidity_recall` | 1701 | success | 25142 | 0.231 | 1 |
| clp | `vault_redeem_pairs` | 1573 | success | 99624 | 0.824 | 1 |
| clp | `vault_redeem_single` | 1643 | success | 90082 | 0.733 | 1 |
| mint_redeem | `add_collateral_vault` | 837 | success | 17468 | 0.186 | 1 |
| mint_redeem | `apply_corporate_action` | 656 | success | 16548 | 0.138 | 1 |
| mint_redeem | `close_market_admin` | 2322 | success | 12626 | 0.125 | 2 |
| mint_redeem | `configure_risk_fees` | 778 | success | 13561 | 0.146 | 1 |
| mint_redeem | `configure_worst_case_quoting` | 751 | success | 13423 | 0.125 | 1 |
| mint_redeem | `configure_yield_ceiling` | 2143 | success | 14676 | 0.247 | 1 |
| mint_redeem | `create_token_metadata` | 1979 | success | 67407 | 0.537 | 1 |
| mint_redeem | `deploy_collateral_to_yield` | 2191 | success | 24357 | 0.264 | 1 |
| mint_redeem | `donate_to_vault` | 1823 | success | 30505–35371 | 0.292 | 2 |
| mint_redeem | `enable_multi_collateral` | 811 | success | 13555 | 0.122 | 1 |
| mint_redeem | `harvest_collateral_yield` | 2267 | success | 19702 | 0.192 | 1 |
| mint_redeem | `init_collateral_authority` | 2361 | success | 28641 | 0.369 | 1 |
| mint_redeem | `initialize_market` | 247 | success | 32957–37499 | 0.356 | 3 |
| mint_redeem | `keeper_burn_single` | 1465 | success | 21227 | 0.283 | 1 |
| mint_redeem | `keeper_mint_single` | 1048 | success | 41836 | 0.374 | 1 |
| mint_redeem | `keeper_redeem_single` | 1184 | success | 40838 | 0.443 | 1 |
| mint_redeem | `mark_yield_impairment` | 2305 | success | 14667 | 0.177 | 1 |
| mint_redeem | `migrate_to_unified_vault` | 2379 | success | 25157 | 0.242 | 1 |
| mint_redeem | `mint_paired` | 907 | success | 48562 | 0.479 | 1 |
| mint_redeem | `mint_paired_with_waiver` | 937 | success | 61904 | 0.462 | 1 |
| mint_redeem | `permissionless_burn` | 1537 | success | 25774 | 0.252 | 1 |
| mint_redeem | `realloc_market` | 2088 | success | 4070–6981 | 0.126 | 2 |
| mint_redeem | `reanchor_gamma` | 551 | success | 15703 | 0.147 | 1 |
| mint_redeem | `rebalance_collateral` | 1732 | success | 32540 | 0.249 | 1 |
| mint_redeem | `recall_collateral_from_yield` | 2238 | success | 20061 | 0.203 | 1 |
| mint_redeem | `redeem_paired` | 974 | success | 49853–56302 | 0.436 | 2 |
| mint_redeem | `redeem_paired_with_waiver` | 1001 | success | 63280 | 0.447 | 1 |
| mint_redeem | `redeem_single_for_vault` | 1323 | CPI success | 46197 | nested | 1 |
| mint_redeem | `set_buffer_target` | 448 | success | 13637 | 0.137 | 1 |
| mint_redeem | `set_s_gamma_index` | 488 | success | 14973 | 0.145 | 2 |
| mint_redeem | `update_collateral_vault` | 879 | success | 18666 | 0.253 | 1 |
| mint_redeem | `update_fee_recipient` | 2162 | success | 17100 | 0.170 | 1 |
| mint_redeem | `update_keeper_authority` | 1945 | success | 13787 | 0.124 | 1 |
| mint_redeem | `update_market` | 1913 | success | 14093–16041 | 0.141 | 3 |
| mint_redeem | `update_market_oracle` | 1954 | success | 21463 | 0.236 | 1 |
| mint_redeem | `update_risk_state` | 334 | success | 13180 | 0.194 | 6 |
| mint_redeem | `withdraw_excess_to_clp` | 1618 | success | 32619 | 0.246 | 1 |
| oracle | `configure_health_thresholds` | 281 | success | 10685 | 0.163 | 1 |
| oracle | `configure_twap_windows` | 258 | success | 10664 | 0.160 | 1 |
| oracle | `emergency_pause` | 473 | success | 11353 | 0.142 | 1 |
| oracle | `force_price` | 432 | success | 10913–12567 | 0.222 | 3 |
| oracle | `get_keeper_price` | 377 | success | 6662 | 0.160 | 1 |
| oracle | `get_keeper_twap` | 218 | expected rejection (deprecated) | 7292 | 0.154 | 1 |
| oracle | `get_market_state` | 239 | success | 6283 | 0.123 | 1 |
| oracle | `get_oracle_health` | 244 | success | 6695 | 0.181 | 1 |
| oracle | `get_price` | 314 | success | 9323 | 0.209 | 2 |
| oracle | `get_user_price` | 359 | success | 6617 | 0.171 | 1 |
| oracle | `get_user_twap` | 225 | success | 6288 | 0.122 | 1 |
| oracle | `initialize_oracle_config` | 22 | success | 24901 | 0.298 | 1 |
| oracle | `realloc_oracle_config` | 80 | success | 11702–41710 | 0.213 | 3 |
| oracle | `reset_volatility_counters` | 301 | success | 13511 | 0.231 | 1 |
| oracle | `update_oracle_config` | 389 | success | 12089 | 0.124 | 1 |
| oracle | `update_price_observation` | 106 | success | 16089–45435 | 0.250 | 5 |
| oracle | `upsert_oracle_feed` | 85 | success | 20387 | 0.295 | 1 |
| registry | `initialize` | 13 | success | 14501 | 1.568 | 1 |
| registry | `register_market` | 23 | success | 17602 | 0.390 | 1 |
| registry | `set_active` | 83 | success | 10073–10076 | 0.243 | 4 |
| registry | `transfer_authority` | 98 | success | 7538 | 0.201 | 1 |
| registry | `update_market` | 49 | success | 10704 | 0.248 | 1 |
| registry | `update_positions` | 69 | success | 10134 | 0.261 | 1 |
| faucet | `drip` | 47 | success | 32050 | 0.609 | 2 |
| faucet | `initialize` | 17 | success | 20331 | 1.570 | 1 |
| faucet | `transfer_authority` | 103 | success | 8097 | 0.181 | 1 |
| faucet | `update_params` | 90 | success | 5728 | 0.200 | 2 |
