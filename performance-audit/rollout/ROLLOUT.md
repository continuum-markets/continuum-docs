# Performance rollout — 9 October 2026

The authorized devnet rollout upgraded CLP and Oracle, resized all 15 eligible Oracle configs, and deployed the scoped Keeper and frontend improvements. Live mint, redemption, pool buy, pool sell, commit and Keeper settlement succeeded and their six receipts are finalized. The entries below distinguish deployed source from the broader local audit build.

## Deployed source

| Component | Clean local commit | Deployment |
|---|---|---|
| Frontend, latest concurrent product release | `f97f949f6ffcd35d6831836cfaa8a89051c492be` (includes performance commit `9585167`) | Railway `1579d4dd-8d51-43c3-a326-a78717745a11`, SUCCESS; build `aNMyDW7W9BQ4QbUCLSkps` |
| Keeper | `b05154e` | Railway `4fabc48a-e122-4500-8116-8bc28d1b28f5`, SUCCESS |
| CLP and Oracle | `6d9d13d608aecb6350807a1d5c82a9dc6fbf523b` | Finalized devnet upgrades; exact deployed bytes verified |

The performance commits are retained on local `performance/frontend-20261009`, `performance/runtime-20261009` and `performance/programs-20261009` branches. The additional visual correction is retained on `analytics/visual-polish-20261009`, followed by the concurrent `product/retire-fee-waiver-offer-20261009` release. No push or dirty-main-worktree replacement was performed by this audit. Railway's environment is named `production`; its configured chain and both deployed programs are devnet. No mainnet action, authority change, or other program deployment occurred.

The frontend preserves posted-price commit `2a7c22e` and its preceding browser analytics. A concurrent posted-price deployment replaced the initial performance deployment during verification; the final release merges the performance changes onto that newer source. Its one merge conflict retained nullable idle-synth valuation. The SDK rewrite, address-derivation cache, exact-quote integration and durable transaction recovery remain outside this deployed frontend. The later shared HTTP confirmation fix works with the existing Anchor stack. Accordingly, the original audit's faster SDK wallet timings are not claims about this release.

The initial combined frontend cloud build failed in Next's Google-font resolver. One retry of the identical source succeeded, with no font, asset-quality, dependency or bundler change. The failed build never replaced the running site. The final container build ID matches the live T3-rendered document; the diagnostic harness now records and checks that ID.

A second concurrent deployment, `3024aab`, is a direct descendant of performance release `9585167` and preserves every fix. Its three changed lines across two files constrain the chart's grid children on narrow screens and suppress a redundant error when reference history is intentionally unavailable. All 332 benchmark source-manifest entries match that clean release. The build guard rejected samples taken across the deployment boundary; the final comparison reruns against combined build `IKi12OWWBM-jIGLF_59hR` rather than mixing versions. [Excluded samples](combined-browser-exclusions.json), [benchmark manifest](frontend-benchmark-source-manifest.json), [benchmark deployment](frontend-benchmark-deployment.json).

During the last checks, a separate product release `f97f949` retired the donation-for-waiver offer, clarified existing-waiver messaging and updated fee documentation. It descends from the benchmark build and changes only three files; all performance fixes, financial hooks and HTTP confirmation remain intact. Its 331-file source manifest matches exactly, and a live T3 check verifies its new build ID, working mint controls and no horizontal overflow. The timings below remain attributed to the benchmark build; this product change is not counted as an optimization or silently substituted into those measurements. [Latest-source preservation](latest-source-preservation.json).

Evidence: [frontend manifest](frontend-final-source-manifest.json), [frontend patch](frontend-final-release.patch), [deployment](frontend-final-deployment.json), [build attestation](frontend-final-build-attestation.json), [Keeper manifest](keeper-source-manifest.json), [Keeper deployment](keeper-deployment.json), [scope](release-scope.json).

## Programs and migration

- Oracle finalized at slot `509018501`; CLP at `509019969`. Reviewed binary hashes match the deployed prefix, unused allocation tails are zero, and upgrade authorities are unchanged.
- Oracle program storage first grew by 11,992 bytes. Fifteen selected config accounts then grew from 1,765 to 1,781 bytes, preserving every original byte and zeroing the new bytes. Total config rent top-up: zero; resize fees: 75,000 lamports.
- Three configs with missing Market accounts and one older 1,733-byte schema were excluded; their final bytes match their pre-rollout state.
- One public-RPC rate-limited submission was journaled and reconciled through finalized blockhash expiry. Only after proving it absent and the account unchanged was a new transaction signed. No blind replay occurred.
- Both tracked upload buffers are closed with zero balance. Rollback binaries and private submission journals remain locally available; raw CLI output and signing material were not copied into public evidence.

The clean program release passed 115 entrypoints, 167 scenarios and 835 signed local-VM fixture transactions, plus exact account parity for eight CLMM scenarios. Full 16-bin buys remain **108,575 → 75,346 CU (30.6% less)** and sells **104,099 → 75,102 CU (27.9% less)** with identical outputs. These are controlled SBF comparisons, not claims about elapsed network confirmation time.

Evidence: [final deployment verification](deployment.json), [per-account results](oracle-migration-results.csv), [program runbook](PROGRAM-PLAN.md), [coverage](release-coverage.md), [IDL compatibility](idl-compatibility.json).

## Keeper after deployment

All 370 enabled offline tests passed; 11 opt-in tests were ignored and three external roundtrip tests were filtered. The container is healthy and its confirmed Order/Market subscriptions are active, with the polling fallback retained. Analytics cursor conflicts were observed during container overlap at startup; the follow-up log query found no further coverage-gap entries after 01:51 UTC.

Five sequential HTTP samples per endpoint were measured inside the running container. This excludes SSH and public-network latency. Cached responses were already fast: medians of 0.15–0.24 ms before and 0.10–0.16 ms after. The status and wallet-health responses were populated; arb-scan and pool-state responses were empty, so those are not populated-workload benchmarks. The small absolute differences are not meaningful evidence of an HTTP optimization.

Three live R2-backed `protocol_snapshot?limit=500` reads returned 200 in 1.151, 0.968 and 1.015 seconds (3,578 response bytes). This measures a small available suffix, not 500 populated rows or a matched live before/after. The earlier deterministic 100,000-row fixture remains the evidence for equivalent rows with 1,000 → 6 object GETs. A pre-smoke settlement telemetry read returned 400 because the new stream had no pending-order records yet; the post-smoke read returned 23 records and includes the actual test order.

For order `3zHHqBR8nmsHQKbKRf1czb8pMJRi1qCc8Y3ySuCzkj4b`, Keeper first observed it at age 1.475 s, then waited for a newer oracle print. An account-change-triggered scan taking 125.6 ms observed the newer oracle print; eligibility was observed at order age 12.232 s. Eligibility-to-submit took **311.7 ms** and submit-to-confirm took **1,827.4 ms**, giving 14.372 s from the on-chain commit timestamp to Keeper's confirmed result. Chain block timestamps place commit and settlement 12 s apart (one-second precision). This one live observation validates the notification path and separates the required oracle wait from Keeper work; it does not establish p95 latency or a matched improvement over the old Keeper.

Evidence: [tests](keeper-tests.txt), [before HTTP samples](keeper-http-before.jsonl), [after HTTP samples](keeper-http-after.jsonl), [stable health](keeper-health-stable.json), [live tail samples](keeper-live-tail-after.jsonl), [live settlement phases](keeper-live-settlement.json).

## Final browser and wallet measurements

A signed mint exposed a deployed Anchor confirmation defect: the transaction finalized successfully at slot `509021414` using 51,839 CU, but the page waited 30 seconds and reported an unknown outcome. The exact signature was reconciled before further transactions. Installed Anchor calls `confirmTransaction(signature, 'confirmed')`; installed web3.js waits for its WebSocket subscription before starting the HTTP status read. The browser's derived WebSocket URL targets the site's POST-only RPC proxy. The scoped shared HTTP confirmation fix is now deployed; no SDK migration or automatic transaction replay is involved. Full frontend validation passes 187 tests across 39 files, TypeScript and scoped lint. Eighteen regression tests cover the installed Anchor provider, commitment levels, execution errors, expiry, cancellation, malformed responses, transient retry limits and no-resubmission behavior. [Finalized mint receipt](mint-timeout-reconciliation.json).

The successful real-wallet run used build `tGsBihMahyrh9iURsneoL` at commit `3dfb35c`. It used the actual React controls, 4× CPU throttling and Fast 4G, with a disposable devnet wallet automatically approving each signature. Human approval dwell and wallet connection setup are excluded. These are single observations per action, not percentiles.

| Live action | First visible status | Explicit success / committed status | Finalized CU |
|---|---:|---:|---:|
| Mint 20 cUSDC | 59.8 ms | 1,571.8 ms | 51,839 |
| Redeem 15 cUSDC | 40.0 ms | 2,346.2 ms | 46,870 |
| Pool buy, 1 cUSDC | 35.3 ms | 1,677.4 ms | 52,228 |
| Pool sell, 0.00001 XAUL | 63.0 ms | 1,686.5 ms | 52,186 |
| Commit buy, 1 cUSDC | 244.1 ms | 1,895.4 ms | 52,981 |

Commit cleared the previous status after 52.5 ms; its first nonempty progress message appeared at 244.1 ms. A pre-input “Swap confirmed” event belongs to the previous sell and is excluded. The page explicitly showed the order settled by **18.508 s** after input; this is an observed-by upper bound because the harness polls the page. The settlement used 43,481 CU. All six receipts have `err: null` and the expected instruction names.

The run also exposed two pool receipt links using the explorer's address path. Performance release `9585167` changes those two calls to request transaction URLs; the subsequent visual release adds the three layout/warning lines described above. Transaction execution and confirmation code are unchanged from the successful wallet build. Financial actions were not repeated solely for these presentation corrections. The final public-page diagnostics use combined build `IKi12OWWBM-jIGLF_59hR`.

Evidence: [wallet timings and finalized receipts](wallet-final-summary.json), [confirmation diagnosis and validation](confirmation-regression.md), [wallet-build source manifest](frontend-confirmation-source-manifest.json), [wallet-build attestation](frontend-confirmation-build-attestation.json). Full raw wallet records are in `frontend/docs/performance-audit/rollout-wallet-final/`.

### Matched public-page comparison

The posted-price baseline and combined final build were measured in new isolated contexts with 4× CPU throttling and Fast 4G. The market roster has three samples per build, using the same route, query and action order. These are small lab samples on a shared computer, not field INP, population percentiles or a load test. Action response is input to two animation frames; total long-task time covers the harness's observation window.

| Market roster measurement | Before, median | After, median | Change |
|---|---:|---:|---:|
| Stocks filter | 571.6 ms | 268.2 ms | 53.1% lower |
| Commodities filter | 296.9 ms | 98.5 ms | 66.8% lower |
| Forex filter | 231.5 ms | 123.6 ms | 46.6% lower |
| Observed long-task time | 2,562 ms | 1,887 ms | 26.3% lower |
| First contentful / largest contentful paint | 920 ms | 916 ms | Essentially unchanged |

The final mint page's median first paint is 884 ms and observed LCP is 2,360 ms, when its chain-backed reference paragraph paints. Its tested controls take 26.4–113.9 ms. The final AAPL market-detail page's median first paint and observed LCP are 940 ms. There is no matched posted-price baseline for those two routes, so these are current measurements only.

All nine final desktop route samples match the combined build ID and completed without a harness error. Of 138 control attempts, 105 produced timings and 33 were no longer present after earlier category-filter changes; those are marked unmeasured in the ledger. The 105 measured interactions range up to 331.6 ms. The earlier audit separately exercised all 14 market disclosures; a state-dependent miss here is not replaced with an invented timing.

The separate 375×900 check used the same 4× CPU / Fast 4G settings and verified the combined build on all three routes. One sample per route produced 35 timings from 46 attempts; the other 11 were state-dependent disclosure misses. Market controls took 30.8–221.7 ms, mint controls 26.9–74.5 ms and market-detail controls 28.4–77.1 ms. First paint was 908 / 892 / 932 ms respectively. All three screenshots were visually reviewed. The concurrent release's same-build EUR mint/detail width checks also record `scrollWidth: 375` at a 375 px viewport. These are responsive lab checks, not physical-phone or percentile claims. [Mobile evidence](mobile-summary.json).

The deployed optimizations share market reads and memoize roster rows, mount collateral details on disclosure while preserving prewarmed data, move yield selection to a compact server response, and stop the shared clock in hidden tabs. The original controlled yield fixture reduced the browser response from 2,306,433 compressed bytes to 76 bytes while selecting the same venue and rate. No image resolution, shader quality, typography or animation rate was reduced. The existing animation visibility and reduced-motion behavior were verified in the original audit.

Evidence: [per-route and action statistics](live-comparison.json), [raw action ledger](live-actions.csv), [final desktop rendering](live-combined-detail-desktop.png), [source checks](final-source-verification.json). Raw final public-page samples are in `frontend/docs/performance-audit/rollout-combined-final/`.

The final Keeper health read reports no issues, with both its latest oracle tick and latest task tick three seconds old. The latest frontend and Keeper deployment records are SUCCESS, the 331 latest frontend and 73 Keeper manifest entries match their retained release sources, and all owned diagnostic browser processes have exited. [Final Keeper health](keeper-final-health.json).

The first wallet-fixture attempt was rejected before signing because the existing Wallet Standard adapter omits its optional chain field. The fixture now verifies the app RPC's devnet genesis during connection, while retaining its exact-origin, connected-account and sole-signer guards. That setup time is outside action timing. [Fixture adjustment](wallet-fixture-adjustment.json).

The earlier local audit's broad coverage remains documented in [REPORT.md](../REPORT.md), with its original limitations. Intermediate live samples are retained; one first-deployment sample used the preceding bundle during edge propagation and is explicitly [excluded](browser-exclusions.json).

## Remaining coverage and latency limits

- The full local audit inventories 23 routes, HTTP endpoints, UI states and all program instruction entrypoints. A timed control, signed fixture or expected guard failure does not establish every business branch. The action and runtime ledgers retain misses and failures explicitly.
- The final live wallet proof covers five signed user actions and Keeper settlement. Wider donations, burns, both pool legs and failure branches have separate earlier live/fixture evidence; they were not all repeated after deployment. Private spending is disabled and has no successful end-to-end timing here. CAPTCHA, external message delivery and privileged browser flows remain separate gaps.
- Optional paid Pyth reference history is intentionally unconfigured on devnet. The interface states this and uses posted on-chain prices for trading. It is not counted as a fast successful history response. Mint's chain-backed content still takes about 2.2–2.5 seconds in this lab run.
- The required oracle-print wait remains in committed trades. Keeper notification latency is supported by one successful live order, not a before/after percentile distribution. Populated arb-scan/pool-state HTTP workloads and large live R2 tails remain unmeasured.
- The broader concurrent SDK, privacy and corporate-action work was excluded from this rollout. Mainnet performance, sustained load and real-user p95 latency are not established by this devnet audit.
