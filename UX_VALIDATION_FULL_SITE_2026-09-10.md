# Full-site UX and SEO validation, 10 September 2026

## Verdict

**The earlier landing-page fix pass was not a full-site acceptance test. This broader pass is not a clean sign-off: it found task blockers, an asset-context routing bug, misleading failure states, and programmatic SEO defects.**

Validated a fresh local production build from frontend commit `fa2e44c` (`ui`). The frontend working tree was clean at the start. No application code, deployment, wallet connection, signature, trade, mint, redemption, or faucet transaction was changed or executed during this pass. A temporary checker and this report are audit artifacts only.

The checks used `ux-laws` and `ux-review`. Existing visual direction was retained as the reference: readable task-first forms, obvious actions, truthful data states, familiar navigation, and coherent editorial pages. No new gallery research or decorative component was needed for this review.

## Coverage: what was actually tested

“Rendered” below means inspected in Chrome, not inferred from a build. This is a representative route/state matrix, **not every URL at every breakpoint**.

| Surface and job | Browser coverage in this pass | Other checks / limits |
|---|---|---|
| Landing: explain the product and enter the app | 375 × 760; CTA navigation, browser Back, scroll-lock cleanup, keyboard skip link | Prior responsive fixes remain covered by the earlier report. Native iOS, physical momentum, and native reduced-motion preference changes were not certified. |
| `/pools`: choose a market | Screenshots at 375 × 760, 768 × 1024, 1440 × 900; populated list, keyboard-operated category filtering, card navigation | Fresh isolated context with delayed reads, then simulated 503 read failures. |
| `/pools/AAPL`: inspect and trade exposure | Screenshots at all three widths above; 100 cUSDC Short preview, 5,001 cUSDC auto-commit preview, width-only resize with draft retention | Disconnected only. Price-history error observed. No fills, fees, settlement, refunds, or portfolio accounting certified. |
| `/swap`, `/swap/AAPL` | HTTP redirects followed to `/pools` and `/pools/AAPL` | Trading is implemented in the market-detail page, not a separate swap screen. |
| `/mint`, `/mint?mode=redeem`, `/redeem` | Mobile and desktop renders; default QQQ and selected OIL; mode switch; enabled wallet-picker entry and Escape dismissal | Stale-price display, missing h1, input placement, and SEO-to-mint context tested. No connected transaction state. |
| `/faucet` | 375 × 760 and 1440 × 900; disconnected state | Loading/success/cooldown/error implementations inspected in source, not executed on-chain. |
| `/portfolio`, `/earn`, `/yield` | Portfolio disconnected DOM/controls inspected at desktop; redirects followed for Earn/Yield | Portfolio balances/positions not tested. |
| `/trade` asset directory | Desktop screenshot and rendered metadata/links | All index metadata checked by HTTP; no exhaustive directory interaction matrix. |
| `/trade/aapl` asset template | 375 and 1440; primary/secondary CTA destinations and actual secondary navigation | Other asset URLs covered by HTTP metadata/link checks, not all screenshot individually. |
| `/guide/hedge-portfolio-defi` guide template | 375 and 768; initial/mid-scroll content and reveal state | Broken related-market link traced to shared template. |
| `/compare/continuum-vs-synthetix` comparison template | 375 and 768; side-by-side/stacked feature panels | Third-party protocol claims not independently certified. |
| `/learn/synthetic-asset` glossary template | 375 and 1440; definition, supporting content, structured-data type, Lighthouse snapshot | Google rich-result eligibility is not implied by valid JSON. |
| Remaining SEO indexes, Terms, Privacy | HTTP, metadata and internal-link checks | Not a full visual or legal-content review. |
| Unknown routes | Invalid market rendered at desktop; invalid SEO route's rendered 404/recovery controls inspected | HTTP/robots checks across additional invalid route families. |
| Social images | One AAPL card visually inspected and decoded as 1200 × 630 | All 66 provided unique OG image URLs checked for HTTP 200 and image content type. Missing metadata images are counted separately. |

Surface classification matters: pools are a working interface, trade/mint/faucet are transactional, and SEO articles are explanatory. Decorative polish cannot compensate for an unusable action or an unknown value displayed as a known one.

## P1 findings: fix before calling the site validated

### 1. An AAPL mint CTA opens the QQQ form

**Browser-confirmed.** On `/trade/aapl`, “Or mint the AAPL pair directly” links to `/mint`. Clicking it opened Mint with **QQQ (Nasdaq-100 ETF)** selected, not AAPL.

The primary “Open the AAPL market” points correctly to `/pools/AAPL`. It is the asset-specific secondary action that drops intent. The template uses a generic mint URL, so this is a shared-template risk; AAPL → QQQ was the exercised case.

Source: `frontend/app/trade/[asset]/page.tsx:137–140`; the default selection in `components/MintRedeemView.tsx`.

**Recommendation:** carry a validated market symbol into the mint route and initialize the form from it. Test every asset-specific CTA against its resulting selection. Never silently fall back to another asset when the request names one.

### 2. Trading still has a disabled “Connect Wallet” action

**Browser- and source-confirmed.** `/pools/AAPL` renders a disabled “Connect Wallet” button in the trade form, including after entering an amount. Header “Select Wallet” remains the workaround. The Mint/Redeem fix did not reach `SwapPanel`.

Source: `components/SwapPanel.tsx:177, 463–478`: `canSubmit` requires connection, while the disabled transaction button is labeled “Connect Wallet” when disconnected.

**Recommendation:** use the existing wallet-picker trigger in the disconnected form state; keep transaction eligibility checks for the connected state. Do not simply enable the swap handler without its guards.

The faucet has a related onboarding gap: its “Get 1000 cUSDC” action is disabled while disconnected, with no in-card instruction or connect action. Disabling the request itself is correct; leaving the next usable action elsewhere is not. Source: `components/FaucetButton.tsx:27–40`.

### 3. Invalid markets render a plausible trading screen

**Browser- and HTTP-confirmed.** `/pools/NOTAREALMARKET` returns **HTTP 200**, displays that name as the h1, and renders a trade form, a **7.21% yield**, a “Pegged” badge, and a **0.30% spread**. It says native swaps are “not enabled … yet,” implying an existing/future market rather than an invalid address.

Metadata says “Market not found” and `noindex, follow`, but that does not fix the user-facing screen. No transaction was attempted, and no fund-loss claim is made.

Source: `app/pools/[symbol]/page.tsx:22–24, 37–42` handles metadata only; `pool-view.tsx:193–241` still renders the market interface and fallback metrics.

**Recommendation:** validate the symbol server-side and return the proper not-found boundary/status before rendering trading controls. Unknown data must not inherit plausible financial figures.

### 4. Core navigation is not reliably operable

**Market cards:** `/pools` cards are clickable divs with no link role, tabindex, or focusable child. The main market grid contains no anchors. Keyboard users can operate category filters but cannot use the primary card-selection path. “View →” also appears only on hover. Source: `app/pools/pools-view.tsx:374–429`.

**Tablet app header:** at **768px**, the Faucet link is covered by the Devnet control. Its link rectangle was x=462.58–518.16; hit-testing its center returned **Devnet**, not the link. This was observed on the shared app header, not the landing header. Source: `components/Header.tsx`, the absolutely centered desktop nav and right-hand network/wallet group.

**Recommendation:** use native market links and a header layout that adapts to available space instead of overlapping absolute groups. Changing contrast or increasing target size alone does not solve these access failures.

### 5. Data failures are confused with loading or idle operation

**Controlled browser test:** in a fresh isolated context, RPC, protocol snapshot, keeper-health and price/yield reads were first held pending, then returned simulated 503 responses. Initial skeletons were present. After failures, the screen still had no explicit data-failure/retry action; it showed dashes/placeholders and at one point **“Idle”** for collateral yield. Subsequent polling switched some values back to skeletons.

Source: `components/CollateralYieldMetric.tsx:32, 56` explicitly combines unavailable fallback data with “Idle.” That is a different state from confirmed idle collateral.

**Actual local history failure:** `/api/pyth-ohlcv?symbol=AAPL&resolution=1` returned 502 with:

```text
Pyth Benchmarks: all 4 feed(s) failed for AAPL (Error: Pyth Benchmarks request failed: 404)
```

The chart displayed a pool trace while reporting failed history below it. This is distinct from the separate Hermes CORS failures. These are local observations, not a diagnosis of production infrastructure or provider configuration.

**Recommendation:** separate loading, unavailable, stale and confirmed-idle states. Label surviving data by source/freshness, retain an actionable error near the affected content, and provide recovery. Do not fill failed reads with reassuring defaults.

## P2 findings: UX, content, and SEO follow-up

### 6. Stale mint data is too easy to encounter as an ordinary quote

The default QQQ mint view reported an on-chain price **916 minutes old** during the check. Selecting OIL displayed prices of $82.79 / $64.89 and a last update **21,319 minutes old** (roughly 15 days). The age warning was below the amount/preview area, not beside asset selection and the prominently displayed NAVs. Redeem did not show the same age warning.

The source's connected submit predicate is not tied to that displayed age condition. This is a **UI/source consistency concern**, not a certified statement about which transactions the on-chain program would accept. Verify operation-specific rules and then align state, labels, and gating; do not infer live execution from a displayed estimate.

### 7. Mobile task controls come after too much secondary information

At 375 × 760, the trade form began around document y=1,280, after the main chart and depth card. In the inspected redeem state, the amount field began at y=866.5 and the wallet action at y=1,092.5. Yield, volatility and supporting protocol details precede the primary input.

The desktop `/pools` directory also leaves half the first four-column category row empty because Indices contains only two cards, while Stocks starts below it. Category filtering works, but the default card wall is not efficient market scanning.

**Recommendation:** prioritize market selection and the amount/quote path; progressively disclose secondary yield/technical information. Consider a compact market-list treatment instead of large repeated metric cards. Keep required risk and execution details near the action rather than deleting them.

### 8. SEO pages do not inherit the landing-page messaging/accessibility fixes

`components/seo/SEOHeader.tsx` is a separate header: it still says “Launch app,” lacks the new devnet framing, and hides all its nav links below `md`. Its white-on-violet CTA still fails contrast. Glossary/guide endings can emphasize an early-access signup even though the devnet app is available.

The `/trade` intro still advertises crypto pairs, and the commodities description says oil, although its displayed live-market directory contains neither a crypto-pair category nor an oil market. `/trade/aapl` still instructs readers to post tokens as collateral or trade on “any Solana DEX.” Availability is more conditional than those words suggest.

**Recommendation:** propagate network/availability language through SEO headers, templates and CTA components. Separate trying devnet from joining a future-access list. Qualify integration claims and keep generated copy aligned with the live asset registry.

### 9. Long SEO sections remain dim while already readable on screen

On `/guide/hedge-portfolio-defi` at **768 × 1024**, the step section began around y=457 with several instructions in view, yet its settled opacity was **0.577444** before and after a 500ms wait. The animation is driven by `view()`, so waiting does not finish the reveal. Scrolling farther made the text clearer.

Source: `app/globals.css:677–682`, especially the entry 20% → entry 90% range applied to a long `.reveal-rise` section.

**Recommendation:** do not make readers scroll farther to obtain normal text contrast. Reveal a small heading/decorative group once, or keep body text opaque. The comparison page additionally stacks long per-protocol panels on phones, requiring substantial memory to compare the same feature; row-by-row mobile comparisons would be easier to use.

### 10. Landing-page Back navigation loses the reading position

**Browser-confirmed.** Starting with `.landing-shell.scrollTop = 1200`, clicking the visible “Try devnet” action opened `/pools` and correctly removed `landing-lock`. Browser Back returned to `/` with the inner scroller at **0**, not 1200.

The new keyboard skip link did work: it received focus, navigated to `#main-content`, and focused the `MAIN` element. The problem is restoration of the custom scroll container, not all landing navigation.

**Recommendation:** explicitly preserve/restore the route's scroll position or use document scrolling where appropriate. Verify back/forward alongside ordinary wheel/keyboard movement.

### 11. Faucet cooldown recovery needs a targeted fix and test

**Source-only finding; not an executed drip.** The button disables whenever `status.kind === 'cooldown'`. The hook does not schedule an expiry transition; its `reset` function is not consumed by `FaucetButton`. Even if a rerender makes the countdown say “now,” the state remains cooldown and the button remains disabled.

Source: `components/FaucetButton.tsx:9–14, 19–39`; `hooks/useFaucet.ts:72–75, 145–157`.

Also, the transaction uses the recipient as fee payer and may create a token account (`useFaucet.ts:81–118`), but the faucet page does not explain the need for devnet SOL for fees/rent. An actual request, insufficient-SOL error, cooldown, expiry and success state still need controlled wallet testing.

## Programmatic SEO results

The checker inspected **88 local HTML URLs**: the 73 sitemap URLs, additional app/redirect/error cases, and new internal-link targets discovered in those pages. It checked all 14 asset pages, 14 market pages, 8 guides, 7 comparisons, 21 glossary terms, and 9 sitemap static/index routes by HTTP. Rendered screenshots were representative, as specified above.

### Passed

- All **73 sitemap URLs are unique and return 200**.
- No exact duplicate title strings across those sitemap URLs.
- Titles, descriptions and canonical tags are present; some canonicals are wrong, listed below.
- No malformed JSON-LD was found. This is syntax validation, not schema eligibility certification.
- All **66 provided unique OG image URLs** returned 200 with an image content type. The AAPL card was visually checked at 1200 × 630.
- Invalid asset/guide/glossary/comparison routes return 404 and noindex. The rendered SEO 404 gives recovery destinations. The unknown-market exception is described above.
- `/swap`, `/swap/AAPL`, `/redeem`, `/earn` and `/yield` redirect to the intended consolidated app routes.

### Failed / incomplete

1. **Four guides link to two 404 assets:**
   - `/guide/arbitrage-synthetic-assets` and `/guide/portfolio-diversification-synthetics` → `/trade/eth-btc`.
   - `/guide/hedge-portfolio-defi` and `/guide/manage-liquidation-risk` → `/trade/vix`.
   - Cause: guide related assets are mapped and filtered for existence, but not live status (`app/guide/[slug]/page.tsx:75–78`). The asset-detail template already uses `isLiveAsset`; the guide template does not.
2. **Missing `og:image`:** `/pools`, `/trade`, `/guide`, `/compare`, `/learn`, `/mint`. Browser inspection confirmed absence after hydration as well as in HTTP markup. `pageMetadata` replaces the parent OG block without an image fallback (`lib/seo/metadata.ts:40–55`).
3. **Wrong canonical:** `/terms` and `/privacy` point to the homepage. Extra checked routes `/faucet` and `/portfolio` also inherit the homepage canonical and `index, follow`.
4. **Repeated brand suffix:** `/trade`, `/guide`, `/terms`, `/privacy` end in `| Continuum | Continuum`.
5. **Robots blocks render assets:** `app/robots.ts:9` disallows `/_next/`, which includes the JavaScript/CSS needed to render the site. Faucet is crawl-blocked but still emits index/follow; blocking crawl is not a reliable substitute for an intentional noindex policy.
6. **Missing page semantics:** `/mint` has no h1 in HTTP or rendered DOM; `/portfolio` has no main landmark. The SEO footer's h4 “Product” skips heading levels in the sampled glossary page.
7. **Context-routing concerns:** the AAPL mint CTA is wrong as demonstrated above. Asset category breadcrumbs encode `/trade?category=...`, but `TradePage` does not consume that parameter; this latter point was source-inspected rather than clicked through for every category.
8. **Rich results and search performance were not certified:** bare Product/Article/FAQPage JSON is not proof of Google eligibility, indexing, rankings, or correct third-party product claims. No Search Console or external rich-results submission was performed.

## Automated checks and what their scores do not mean

- Fresh `bun run type-check`, `bun run lint`, and `bun run build` passed. The build generated 152 static pages, with the same non-blocking bigint binding fallback and Node localStorage warnings observed previously.
- AAPL trading desktop Lighthouse snapshot: **96 accessibility**, with contrast failures in network-status/market-detail text; SEO category 100.
- Glossary desktop Lighthouse snapshot: **94 accessibility**, with Launch app contrast and SEO-footer heading-order failures; SEO category 100.
- These automated scores did **not** identify the asset-context bug, unusable market-card path, or tablet pointer obstruction. They are not acceptance criteria on their own.

The temporary checker and detailed results are under `/var/folders/6z/djx5y65d72g5sqz76384089w0000gn/T/opencode/`: `continuum_site_check.py`, `continuum-full-seo-results.json`, and `continuum-full-seo-summary.json`. Checks are sequential and cached per route with timestamps; application source was not modified.

## UX principles driving the recommended order

- **Jakob's Law:** make market selection and navigation behave like native links and predictable history, not div-only actions or covered controls.
- **Proximity:** put a usable wallet action and relevant status beside the form that needs it; keep labels, values and consequences together.
- **Postel's Law / recovery:** unknown, failed and stale data need honest states and a way forward, not plausible defaults or an unexplained disabled button.
- **Miller's Law:** reduce the information readers must hold while crossing long forms or comparing separate mobile panels.

Fix asset-context routing, primary actions, invalid-market handling and navigation obstruction first. Then address data-state recovery and the template/SEO defects. Broader visual refinements come after those task failures, not instead of them.

## Explicit limits

No wallet was connected and no transaction was submitted. Swap signing/settlement/cancellation/refunds, mint/redeem success and balance changes, faucet request/cooldown expiry, connected portfolio states, mainnet behavior and production provider configuration remain unverified. Quote previews are not a validation of execution math.

The simulated 503 test was limited to a fresh isolated browser context and read APIs; it did not change a backend or deployed configuration. The actual AAPL history 502/upstream 404 and Hermes CORS failures were observed locally and should be diagnosed separately from UI defects.

Native Safari/iOS, physical touch/trackpad momentum, native OS reduced-motion toggling, exhaustive zoom/text enlargement, every route × every viewport, all external forms/Docs workflows, semantic rich-result eligibility, indexing and search performance were not certified. The report deliberately distinguishes rendered observations, HTTP checks, source-only findings and recommendations.

