# Performance audit checkpoint — 9 October 2026

The primary deliverable is [REPORT.md](REPORT.md), with runtime coverage in [runtime-coverage.md](runtime-coverage.md). Evidence is retained alongside those reports and in `frontend/docs/performance-audit/`.

## Rollout continuation

The user's later “proceed” authorized the devnet rollout. CLP and Oracle upgrades and 15 config resizes are finalized; scoped Keeper and frontend releases are live. The benchmark frontend preserves the intervening posted-price release and includes the later three-line visual correction; its source is `3024aab`, a descendant of performance commit `9585167`, deployed as Railway `4676f241-c577-4950-8f6d-91ef586f6921` / build `IKi12OWWBM-jIGLF_59hR`. The latest concurrent product release is descendant `f97f949`, Railway `1579d4dd-8d51-43c3-a326-a78717745a11` / build `aNMyDW7W9BQ4QbUCLSkps`; it retires the donation-for-waiver offer while preserving every performance fix. Its source and rendered mint controls were verified, and benchmark timings retain their original build IDs. Exact deployments, branch commits, verification and remaining SDK boundaries are recorded in [rollout/ROLLOUT.md](rollout/ROLLOUT.md). The old localhost server session 81269 has been stopped.

Live wallet verification found and fixed a 30-second Anchor confirmation failure caused by waiting for a WebSocket subscription on the HTTP-only RPC proxy. The originally submitted mint was finalized and reconciled without replay. On the fixed build, new mint/redeem/buy/sell actions reached explicit success in 1.57–2.35 s, and commit acknowledgment took 1.90 s. Keeper settled the order; all six receipts are finalized. Final combined-source desktop verification completed nine throttled samples and 105 timed interactions, with build IDs checked. Stocks-filter median improved 571.6→268.2 ms and Commodities 296.9→98.5 ms. The original checkpoint below is historical and includes a broader concurrent SDK implementation that this rollout intentionally excludes.

The final 375 px sweep completed three additional build-checked route samples and 35 timed interactions; all screenshots were reviewed. The final frontend and Keeper source manifests match, release worktrees are clean, and the owned diagnostic browsers have exited. Keeper's final health response has no issues. The rollout is complete; scope exclusions and untimed paths remain explicit in its report.

## Original local audit checkpoint

- User authorized and funded a disposable devnet wallet. Real signed frontend and CLI actions have succeeded. No deployments, public admin calls or commits were made.
- Final frontend production build passed, including TypeScript; 278 tests across 57 files pass. Build ID: `1BW5SxLZYUWO1X0klpgrK`. Source fingerprint: `d169d12bbb165366976ed3edf6feeb57e9d95fcbc6ff3177854fa7c033668e00`. All 564 manifest entries still match.
- The final localhost production server runs on port 3015, tool session 81269. The original baseline build remains at `/tmp/continuum-next-baseline-20261009`.
- Final three-sample throttled UI sweep completed; `performance-summary.py` regenerated `comparison-final.json` and the 608-attempt ledger. The larger 23-route sweep predates the final confirmation and concurrent analytics changes; reports distinguish them.
- Frontend improvements cover exact integer quotes, immutable address caching, shared roster reads, a compact server yield response, hidden-tab clock cancellation, transaction phase telemetry, wallet reconnection and bounded early confirmation reads. No visual-quality settings were reduced.
- Native visibility, reduced motion and offscreen cancellation passed. Longer heap runs and a JIT-disabled diagnostic found no growing DOM retention in the tested home/pools paths; the report preserves the finite-window limitation.
- Keeper changes cover runtime scheduling, account-notification wakeups, phase telemetry, newest-first JSONL tails and complete bounded stream/day pagination. Local tests pass; live after-deployment latency remains unmeasured.
- Fresh SBF fixtures cover the program instruction inventory. Identical-input CLMM comparisons show 108,575→75,346 CU for full-book buys and 104,099→75,102 for sells, with exact writable account parity.
- Runtime fixtures found and fixed corrupted trailing arguments in internal Meteora helpers. They also found an oracle allocation 16 bytes too small for the 60th observation. The latter now has a checked additive admin resize instruction; existing accounts require migration after deployment. Final fresh binaries, five stable fixture replays per scenario (840 transactions), regression assertions and the migration runbook are complete.

## Final verification

- Final wallet UI mint/redeem/commit reached visible results in 2.142 s / 2.150 s / 1.694 s. First feedback took 26–36 ms. Settlement was observed by 24.219 s; chain commit-to-settlement was 19 s. All four receipts verified. Public-RPC 429 failures remain in evidence; missing receipt reads succeeded through the app's configured, genesis-checked RPC.
- Final T3 rendered review completed and screenshot retained. The final frontend source manifest still matches; standalone TypeScript also passes with the additional oracle verifier.
- Read-only oracle readiness verification through the configured RPC found 15 candidates with matching current layout, PDA and owning-market admin. Three further current-layout configs reference missing Market accounts. One older 1,733-byte schema remains excluded. No deployment or resize occurred.
- Migration auditor cache/resume regression passes offline. Runtime documentation and migration exclusions are finalized. All linked artifacts exist. The reports are ready; remaining environment and deployment limits are explicit.

## Resource and scope constraints

This is a shared 24 GB Mac with roughly 14 GB of 15 GB swap in use. Use one compiler or one isolated diagnostic browser at a time. Do not kill the user's Chrome, T3 or shared MCP processes. Private disposable-wallet fixtures stay under `/tmp` with mode 0600 and must never be logged or copied into evidence.

The workspace root is not a Git repository; frontend, Keeper, programs and faucet service are separate repositories with substantial pre-existing and concurrent changes. Preserve them. The audit does not establish production post-deployment timing, every business branch, successful private spending, configured chart history, populated Keeper HTTP performance, CAPTCHA workflows or third-party message delivery. The coverage ledgers distinguish those gaps from executed measurements.
