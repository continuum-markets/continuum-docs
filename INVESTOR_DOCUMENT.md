<img src="https://blobs.continuum.markets/logo.webp" alt="Continuum Markets" width="240" />

# Continuum Markets: Technical Investor Document

**continuum.markets**

*24/7 Synthetic Exposure to Real World Assets on Solana*

***

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Problem](#2-the-problem)
3. [The Solution: Protocol Overview](#3-the-solution-protocol-overview)
4. [The Core Mechanism: Paired L/S Token Design](#4-the-core-mechanism-paired-ls-token-design)
5. [Trust & Control Surface: Why This Is a Category of One](#5-trust--control-surface-why-this-is-a-category-of-one)
6. [Composability](#6-composability)
7. [Tokenomics & Incentive Design](#7-tokenomics--incentive-design)
8. [Security & Risk](#8-security--risk)
9. [Go To Market Strategy](#9-go-to-market-strategy)
10. [Competitive Landscape](#10-competitive-landscape)
11. [Traction & Roadmap](#11-traction--roadmap)

***

## 1. Executive Summary

Continuum Markets is a Solana native protocol that enables 24/7 synthetic exposure to traditional financial assets (equities, indices, commodities, and forex) through a paired Long/Short (L/S) token primitive.

**The problem is structural.** Traditional markets close for 128+ hours per week. The world does not. Price relevant events (earnings, geopolitical shocks, macro data) do not wait for the opening bell. Existing on chain solutions for this exposure require trusting an issuer with freeze authority, delegate control, and pause mechanisms over the tokens they mint. Perpetual futures appear to solve the access problem but introduce a hidden cost. Funding rates erode long term directional exposure and often exceed 20% annualized in trending markets.

**Continuum solves this differently.** When a user deposits collateral, the protocol mints a paired Long token and Short token simultaneously. L tracks the oracle price of the underlying asset. S tracks the inverse. Together, L + S always equals the collateral value. This is the invariant. There is no counterparty on the other side of the trade. There are no funding rates, because there is no synthetic leverage mechanism that requires periodic rebalancing between longs and shorts. While arbitrageurs and liquidity providers interact directly with this mint/redeem engine, **everyday users do not need to.** To take directional exposure, users simply buy or sell the L or S tokens directly on any Solana DEX (like Jupiter), treating them like any other standard token.

**Why it matters:** The L and S tokens are standard tokens with no freeze authority, no delegate authority, no transfer restrictions, and no pause mechanism. The minting authority is controlled purely by immutable code, meaning no administrator or team member can interfere with your position. Once minted, these tokens are fully composable across Solana DeFi: tradeable on Jupiter, usable as collateral on lending protocols, and available for integration into vaults and structured products.

**Current stage:** Three on chain programs are built and tested. The core minting engine is deployed to Solana devnet. The frontend is complete. The off chain keeper system is implemented with six concurrent operational modules. A single anchor market will be seeded for launch to prove the primitive.

**The mechanism is the moat.** Paired L/S minting eliminates the trust surface that defines every other synthetic asset protocol. The result is not a better version of existing synthetics. It is a different category.

***

## 2. The Problem

### 2.1 Traditional Markets Close. The World Does Not.

US equity markets are open 6.5 hours per day, 5 days per week. That is 32.5 hours of trading out of a 168 hour week, approximately 19% of the time. The remaining 135.5 hours per week, traditional brokerage accounts cannot execute trades.

This is not merely an inconvenience. It is a structural exposure gap:

* **Weekend gaps.** Between Friday close and Monday open, any asset correlated with global macro events is exposed to unhedgeable risk. The S&P 500 has historically gapped open by more than 1% on approximately 15 to 20% of Mondays. In crisis periods (March 2020, October 2008), gap opens exceeded 5% in a single weekend.
* **After hours earnings.** Major companies report earnings after market close. NVIDIA, Apple, Tesla, and Meta regularly move 5 to 15% on earnings, price action that occurs entirely outside tradeable hours for most retail accounts.
* **Geopolitical events.** The Russia Ukraine conflict escalation in February 2022 occurred over a weekend. Gold moved $60/oz before any US trader could access their brokerage. Oil gapped 8% on the following Monday open.
* **Crypto correlation events.** Bitcoin flash crashes during weekends (May 2021 flash crash of 30% over a Saturday and Sunday) have second order effects on tech equities and risk assets that cannot be hedged until Monday.

For institutional and active traders, these gaps represent uncompensated risk. The price you see on Friday at 4 PM is not the price you can act on until Monday at 9:30 AM.

### 2.2 Existing On Chain Solutions Have an Issuer Problem

The DeFi ecosystem has attempted to solve the access problem with synthetic assets and tokenized securities. The solutions work until they don't.

**Centralized synthetic issuers and tokenized asset providers** (e.g. tokenized treasuries, wrapped equity tokens) typically retain:

* **Freeze authority:** The issuer can freeze any token holder's balance at any time. This is standard practice for compliance with securities regulations, but it means the holder does not have sovereign control over their position.
* **Pause mechanisms:** The issuer can halt all transfers, minting, and redemption globally.
* **Delegate control:** Administrative wallets can be granted authority over token operations.
* **Off chain custody:** The underlying collateral or reference asset is held by a custodian, not verifiable on chain.
* **Transfer restrictions:** Whitelists, blacklists, or KYC gated transfer hooks that limit where tokens can move.

Most users do not discover these constraints until they try to move their tokens and cannot, or until an issuer exercises their authority during a market stress event, precisely the moment when the ability to transact matters most.

**On chain perpetual futures protocols** solve the access problem differently but introduce their own costs:

* **Funding rates.** Perps maintain price parity with the underlying through a periodic funding mechanism: the dominant side (long or short) pays the minority side. In trending markets, funding rates can exceed 0.1% per 8 hour interval (roughly 36.5% annualized) for the popular side. This is a continuous drag on any long term directional position.
* **No composability.** A perpetual futures position is not a token. It is an entry in a protocol's internal ledger. You cannot transfer it, use it as collateral on another protocol, swap it on a DEX, or send it to another wallet. Your exposure is locked inside the platform where you opened it. If you want to do anything else with that value, you must close the position first, realize any gains or losses, and re enter elsewhere. Perps give you exposure but not ownership of a portable, programmable asset.
* **Counterparty assumptions.** Perp protocols rely on a pool of liquidity providers or an insurance fund to absorb the other side of trades. When these pools are depleted, the protocol socializes losses.
* **Liquidation cascades.** Leveraged positions are subject to forced liquidation, which can cascade in volatile markets and amplify downward moves.

### 2.3 The Gap

There is no existing product that simultaneously provides:

1. 24/7 exposure to traditional financial assets
2. Zero funding rate drag
3. No issuer control over minted tokens
4. Fully on chain, verifiable collateral
5. Standard token composability across DeFi

Continuum fills this gap.

***

## 3. The Solution: Protocol Overview

### 3.1 What Continuum Is

Continuum is a protocol for minting paired synthetic tokens that represent long and short exposure to real world assets. It is not an exchange, an orderbook, or a perpetual futures platform. It is a **primitive**: a base layer mechanism that generates composable financial building blocks.

The protocol accepts stablecoin collateral and mints two tokens: one Long (L) and one Short (S). These tokens are standard SPL tokens on Solana. They can be traded, transferred, used as collateral, or integrated into any protocol that accepts SPL tokens. The protocol does not operate a trading venue; it operates a mint and redeem engine based on **Net Asset Value (NAV)**.

**What is NAV?** In Continuum, NAV is the mathematically derived, intrinsic value of a token, determined by the on chain oracle price of the underlying asset rather than by trading supply and demand. Because the protocol always honors minting and redemption at this exact NAV price, the tokens are fundamentally backed by their mathematical value. If token prices on a DEX drift away from this NAV, arbitrageurs step in to correct the price. Trading happens wherever Solana liquidity exists, anchored by this protocol enforced NAV.

### 3.2 Chain and Stack

| Layer | Technology |
|---|---|
| Blockchain | Solana |
| Smart Contracts | Rust (Anchor framework) |
| Oracle | Pyth Network |
| Frontend | Next.js, TypeScript, TailwindCSS |
| Wallet Integration | Supports all standard Solana wallets |
| Off Chain Keepers | Rust, six concurrent operational modules |
| Deployment | Solana devnet/mainnet (programs) |

### 3.3 On Chain Architecture

The protocol consists of three Solana programs:

| Program | Responsibility |
|---|---|
| **Mint/Redeem** | Core L/S token minting and redemption with NAV pricing, risk based fees, multi collateral support |
| **CLP (Continuous Liquidity Provider)** | Core LP vault taking inventory risk, performance fee collection, yield deployment to lending protocols, and protocol buffer |
| **Oracle** | Price feed integration, dual TWAP calculation, circuit breakers, risk state management |

All critical state lives on chain: collateral balances, minted token supply, NAV calculations, CLP vault balances, and risk parameters.

### 3.4 Oracle System

**Pyth Network** is the primary oracle. Pyth provides sub second price updates for equities, indices, commodities, and forex pairs with confidence intervals that quantify pricing uncertainty.

The oracle system implements:

* **Dual TWAP (Time Weighted Average Price):** The protocol calculates two separate TWAPs simultaneously. A shorter Keeper TWAP (default 60 seconds) is used by keepers to execute arbitrage, while a longer User TWAP (default 5 minutes) is used for user mint/redeem quoting. This prevents frontrunning for users while giving the keepers responsive pricing for peg maintenance.
* **Ring buffer observations:** Up to 60 time weighted price observations stored on chain. The keeper updates observations at a configurable interval (typically every 5 seconds), representing approximately 5 minutes of pricing history.
* **Circuit breakers:** Four independent safety checks: confidence interval threshold, staleness threshold, maximum price movement cap (20% per observation), and emergency pause.
* **Risk state transitions:** The oracle program tracks a simplified Active/Passive market state based on feed health. The keeper's risk monitor translates this into the protocol's four risk states (Normal, Proxy Mode, Stress, Recovery) and propagates them to the mint/redeem program, adjusting protocol behavior based on oracle confidence, staleness, and volatility.

### 3.5 Off Chain Keeper System

An off chain Rust service runs six concurrent tasks to maintain protocol health:

| Task | Interval | Function |
|---|---|---|
| Oracle Updater | 5 seconds | Fetches Pyth prices, updates on chain TWAP observations |
| Risk Monitor | 10 seconds | Tracks oracle health, triggers risk state transitions |
| CLMM Manager | 30 seconds | Manages concentrated liquidity positions on Orca Whirlpools, adaptive rebalancing based on volatility |
| Arbitrage Execution | 5 seconds | Detects NAV to DEX price deviations, executes flash loan arbitrage to maintain peg |
| Inventory Monitor | 60 seconds | Tracks treasury volatile fraction, detects inventory breach conditions |
| Yield Harvester | 5 minutes | Deploys idle collateral to MarginFi for yield based on risk state |

**Why off chain?** Peg maintenance requires complex, multi step execution that cannot be autonomously triggered by a smart contract. The keeper system monitors market conditions continuously and executes arbitrage by chaining flash loans, DEX swaps, and protocol mint/redeems into single, atomic transactions. These are submitted as Jito bundles to guarantee execution efficiency, prevent sandwich attacks, and ensure the transaction only lands on chain if the entire arbitrage path is profitable.

**Centralized at Launch:** The protocol will run centralized keepers operated by the core team at launch. This is a pragmatic, standard approach in DeFi (used initially by MakerDAO, Synthetix, Drift, and Jupiter) for several critical reasons:

1. **Speed of Iteration:** If bugs arise under real market conditions, updating a centralized Docker container takes minutes. Coordinating updates with a decentralized node network is slow and dangerous during an emergency.
2. **Predictable Costs:** The core team pays the Solana transaction fees and Jito tips. This ensures continuous peg maintenance until the protocol reaches Product Market Fit.
3. **MEV Protection:** By running the keeper internally, the protocol maintains a tight grip on peg enforcement, preventing aggressive MEV searchers from draining liquidity during the fragile launch phase. Arbitrage profits are securely recycled back into the protocol's CLP vault.
4. **Simpler Smart Contracts:** Decentralizing keepers requires significant overhead (staking registries, whitelist checks, reward math). Centralization keeps the Rust contracts lean, focused purely on core business logic, easier to audit, and less prone to vulnerabilities.

These keepers maintain peg stability through arbitrage. When the DEX price of an L or S token deviates from its NAV:

* **DEX price below NAV:** The keeper borrows capital via flash loan, buys underpriced tokens on the DEX, redeems through the protocol at NAV, repays the flash loan, and keeps the profit.
* **DEX price above NAV:** The keeper borrows capital via flash loan, mints L+S through the protocol at NAV, sells the overpriced tokens on the DEX, repays the flash loan, and keeps the profit.

This arbitrage is economically incentivized (the keeper profits from the spread) and does not require any protocol subsidy. The mechanism is permissionless: any market participant can perform the same arbitrage.

### 3.6 Supported Assets

A single anchor market will be seeded for launch to prove the primitive in a live environment. Once validated, the protocol architecture supports expansion into any asset category with reliable oracle feeds. The following are purely examples of what the protocol could support in the future:

| Category | Potential Future Assets |
|---|---|
| **Indices** | SPX (S&P 500), NDX (Nasdaq 100) |
| **Equities** | NVDA, AAPL, TSLA |
| **Commodities** | XAU (Gold), WTI (Crude Oil) |
| **Forex** | EUR/USD, GBP/USD |

No crypto markets. Pure traditional finance exposure. This is the differentiator.

New markets are added through admin initialization. The process involves creating token mints for the L and S pair, configuring oracle feeds, and initializing the market with pricing parameters.

### 3.7 Full Lifecycle of a Synthetic Position

**Important Distinction: Traders vs. Minters**
For the vast majority of users, **trading Continuum synthetics does not require interacting with the protocol's mint/redeem engine at all.** Because L and S are standard SPL tokens, users simply buy or sell the specific exposure they want (e.g. buying AAPL L) directly on any Solana DEX (like Jupiter or Orca). 

The full mechanical lifecycle below describes how these tokens are created and destroyed at the base layer. These actions are primarily performed by arbitrageurs, liquidity providers, and the protocol's keepers to ensure the DEX price remains pegged to NAV.

1. **Deposit.** Minter deposits USDC into the protocol.
2. **Mint.** Protocol mints paired L + S tokens (equal value, 50/50 split) to the minter.
3. **Sell one side.** Minter sells the unwanted side on a DEX (e.g. sells S to go long the underlying asset).
4. **Hold.** Minter holds directional exposure. L tracks oracle price; S tracks inverse. No funding payments. No margin calls.
5. **Close.** Minter acquires the opposite token on the DEX (e.g. buys S back to pair with held L).
6. **Redeem.** Minter returns both L + S to the protocol. Both tokens are burned.
7. **Receive.** Protocol returns USDC collateral minus fees.

At no point does a counterparty, issuer, or administrator intervene. The entire lifecycle is permissionless and on chain.

***

## 4. The Core Mechanism: Paired L/S Token Design

This section describes the central innovation of Continuum. Everything else in the protocol exists to support this mechanism.

### 4.1 Paired Tokens & Minting Mechanics

For each supported asset (e.g. Nasdaq 100), Continuum creates two SPL tokens:
* **Long token (L):** Tracks the oracle price of the underlying asset.
* **Short token (S):** Tracks the inverse payoff. As the asset appreciates, S depreciates, but asymptotically approaches zero, never reaching it.

These tokens are always minted and redeemed in pairs. When a user deposits collateral (USDC), the protocol deducts a fee (default 5 bps), splits the net value equally between L and S, and calculates the token amounts using the current NAVs:
* **Long NAV** = current oracle price
* **Short NAV** = (initial Long price x initial Short price) / current oracle price

By construction, total L token supply always equals total S token supply.

### 4.2 The Invariant: L + S = Collateral Value

The Short NAV formula creates an inverse relationship where total value is conserved. If the underlying price doubles, L NAV doubles and S NAV halves, but the collateral backing remains unchanged.

Properties of this design:
* No liquidation cliff for short holders (S NAV approaches zero but never reaches it).
* Short holders can profit indefinitely as the underlying price falls toward zero.
* At any price, the product of L NAV and S NAV is constant.
* For equal quantities of L and S, value changes are perfectly zero sum.

### 4.3 Directional Exposure & Redemption

A user who wants long exposure to the Nasdaq 100 deposits 1,000 USDC, receives approximately 500 USDC worth of NDX L and 500 USDC worth of NDX S tokens, and sells the NDX S tokens on a Solana DEX. They now hold a pure long position. The proceeds from selling NDX S offset half the initial deposit, leading to a capital efficient entry. A user wanting short exposure does the reverse.

To redeem collateral, a user must provide both L and S tokens to the protocol. The protocol calculates the combined value, deducts the redeem fee, burns the tokens, and returns the USDC collateral. Because both tokens are needed for redemption, there is no way to extract more collateral than was deposited (adjusted for price movement). Every L token has a corresponding S token.

### 4.4 Zero Funding Rate & No Counterparty

Perpetual futures require funding rates because they create synthetic leverage. More capital is exposed directionally than the collateral pool can support. The funding mechanism transfers value from the overcrowded side to the undercrowded side.

Continuum has no such imbalance. By construction, every L token is minted alongside a corresponding S token. The total value of all L tokens plus all S tokens equals the total collateral at all times. When someone buys L on a DEX, someone else sells them L. The protocol's total exposure is unchanged. 

There is no long side and short side at the protocol level. There is no counterparty risk in the traditional sense, and the collateral vault does not bear directional risk. L and S holders are matched implicitly through the DEX market, not by the protocol. The funding rate is structurally zero.

### 4.5 Worst Case Quoting & Drift Correction

The protocol uses worst case quoting to protect against informed traders exploiting confidence interval edges:
* **Minting:** User pays the oracle price plus a confidence adjusted spread (the higher bound)
* **Redemption:** User receives the oracle price minus a confidence adjusted spread (the lower bound)

The spread scales with the protocol's risk state:
* **Normal:** 1.0x 
* **Proxy Mode:** 1.5x 
* **Stress:** Mints and redeems are disabled entirely. The market becomes free floating with no peg.
* **Recovery:** Starts at 3.0x and linearly decays to 1.0x over 1 hour (Dutch auction) to prevent MEV attacks on state transitions.

***

## 5. Trust & Control Surface: Why This Is a Category of One

### 5.1 Verified On Chain Properties

The following properties are verifiable by anyone inspecting the token mint accounts on chain:
* **No freeze authority on L or S tokens.** The freeze authority is explicitly set to null. This is an immutable property of the SPL token mint.
* **No delegate authority.** Mint authority belongs solely to a program derived address (PDA), a deterministic address controlled by protocol logic, not a human wallet or multisig.
* **No transfer restrictions or hooks.** L and S tokens use the standard SPL Token program. There are no transfer hooks, no whitelist/blacklist mechanisms, and no programmable restrictions on token movement.
* **No pause mechanism on tokens.** When protocol minting is paused, users cannot create new tokens, but they can continue to transfer, trade, and use their existing L and S tokens without restriction. The tokens are sovereign once minted.
* **On chain collateral.** All collateral is held in program owned vault accounts on Solana. There is no off chain custodian.
* **Multi collateral vault separation.** Each stablecoin (USDC, USDT, PYUSD, USDe) has its own physically separate vault. If one stablecoin issuer exercises freeze authority over their token, the blast radius is contained to that specific vault.

| Property | Continuum | Centralized Synth Issuers | On Chain Perp Protocols | Tokenized RWA Issuers |
|---|---|---|---|---|
| **Freeze authority on tokens** | None (null) | Yes (standard) | N/A (positions) | Yes (standard) |
| **Tokens tradeable if protocol paused** | Yes | No | No | No |
| **Off chain custody** | None | Yes (custodian) | None | Yes (primary) |

### 5.2 The Collateral Layer Risk (Honest Assessment)

Continuum's L/S tokens have no issuer control, but the collateral backing them (stablecoins) does. This is an honest limitation, addressed structurally:
1. **Vault separation:** A freeze on one stablecoin does not contaminate others.
2. **Provenance tracking:** Each user position records which collateral type was used.
3. **Weight based valuation:** Riskier stablecoins can be assigned lower weights (e.g. 0.95:1) to account for depeg or issuer risk.
4. **No cross issuer guarantees:** The protocol does not promise that one stablecoin's collateral will cover another's failure.
5. **Solana native only:** No bridges. Bridged assets add failure modes without adding safety.

***

## 6. Composability

L and S tokens are standard SPL tokens with 6 decimals. They are indistinguishable from any other SPL token at the protocol level.

### 6.1 Where L/S Tokens Can Be Used

* **DEX trading.** L and S tokens are tradeable on Jupiter (and all connected Solana DEXs). The keeper system maintains concentrated liquidity positions on Orca Whirlpools for primary L/USDC and S/USDC pairs.
* **Lending protocol collateral.** L and S tokens can be deposited as collateral on lending protocols like MarginFi, enabling leveraged strategies. The keeper already integrates with MarginFi for flash loans and yield deployment, demonstrating technical compatibility.
* **Vaults and structured products.** Third party vaults can hold L/S tokens, implement automated strategies (e.g. auto rebalancing, yield optimization), or construct structured products.
* **Spread and options strategies.** Holding both L and S tokens across different assets enables spread trading, pair trading, and synthetic options construction entirely on chain.
* **Payments and real world commerce.** Because L and S tokens are standard transferable tokens with real time oracle derived value, they can theoretically be used as a medium of exchange anywhere that accepts Solana tokens. A user holding NVDA L tokens could pay for dinner by transferring those tokens directly, or by routing through a DEX swap to stablecoins at the point of sale. This is not possible with perpetual futures positions, which are locked inside the protocol that created them. Continuum turns investment exposure into a portable, spendable asset.

### 6.2 Existing Integrations

The protocol has built in integration with the following Solana ecosystem protocols:
* **Orca Whirlpools** (Concentrated liquidity)
* **MarginFi** (Flash loans for arbitrage and yield deployment)
* **Pyth Network** (Primary oracle)
* **Jupiter** (DEX aggregation)

***

## 7. Tokenomics & Incentive Design

### 7.1 Protocol Fee Structure

Fees are applied to collateral on mint (0.05% default) and redeem (0.05% default). There are no liquidation penalties because the L/S token design has no liquidation mechanism by construction.

Fees are dynamically multiplied based on the protocol's risk state:
* **Normal:** 1.0x 
* **Proxy Mode:** 1.5x 
* **Stress:** Mints and redeems are disabled. The market becomes free floating.
* **Recovery:** Starts at 3.0x and linearly decays to 1.0x over 1 hour (Dutch auction).

Fee Revenue Distribution:
* **Dev Tax:** A continuous flow based tax directly sent to the developer wallet. On mint/redeem events, the tax is applied to the protocol fee portion. On Keeper arbitrage and spread profit events, the tax is applied to the full profit amount. The rate scales down automatically based on CLP TVL:

| CLP TVL | Dev Tax Rate |
|---|---|
| Below $5M | 40% |
| $5M to $25M | 30% |
| Above $25M | 15% |

This incentivizes early development and operational investment while progressively returning more value to LPs as the protocol scales.
* **CLP Vault:** The remainder stays in the Continuous Liquidity Provider vault to reward liquidity providers and absorb inventory risk.
* **Untaxed Yield:** Yield harvested from idle collateral deployed to MarginFi is left untaxed for simplicity.

### 7.2 The Continuous Liquidity Provider (CLP)

The CLP vault is the protocol's backbone. It aggregates USDC deposits from liquidity providers and uses them to take on inventory risk across supported markets. By seeding Orca Whirlpools with L and S tokens and backing the Keeper's arbitrage operations, the CLP earns yields from protocol fees, trading spreads, and lending protocols, socializing the profits and losses proportionally to all CLP depositors.

*There is no governance token required.* Continuum operates purely on clear, hard coded rules optimized for robust peg maintenance and continuous liquidity provision.

***

## 8. Security & Risk

### 8.1 Audit Status

No external audit has been completed. Getting the product to market and battle testing the core primitive on mainnet is the primary focus. A formal audit by a reputable Solana security firm will be planned for the future once the protocol achieves initial traction and Product Market Fit.

### 8.2 Smart Contract Risk Mitigations

**Framework level safety.** All programs use the Anchor framework, which provides automatic account validation, serialization safety, and discriminator checks. Account constraints enforce authorization at the instruction level.

**Program derived authority.** No human wallet is the mint authority for L or S tokens. The mint authority is a program derived address, a deterministic address controlled by protocol logic that cannot be changed after deployment.

**Checked arithmetic.** All mathematical operations use checked methods that return explicit errors on overflow rather than silently wrapping. This prevents a class of exploits common in financial smart contracts.

**Vault isolation.** Multi collateral mode uses physically separate vault accounts for each stablecoin. A compromise of one vault does not affect others.

### 8.3 Oracle Risk

**Risk:** Oracle price feeds could become stale, inaccurate, or manipulated.

**Mitigations:**

* **Confidence interval checks:** If Pyth confidence exceeds the configured threshold, the market transitions to a restricted risk state.
* **Staleness checks:** If the price feed has not updated within the configured window, operations are restricted.
* **20% price movement cap:** Any single price observation that moves more than 20% from the previous observation is rejected as potentially erroneous.
* **Emergency pause:** An emergency instruction can halt all oracle dependent operations immediately.
* **Dual TWAP buffering:** User facing prices use a 5 minute TWAP rather than spot, smoothing out short term oracle noise.
* **Worst case quoting:** Minting uses the upper bound of the confidence interval; redemption uses the lower bound. This creates a natural bid ask spread that protects the protocol from informed traders exploiting oracle edges.

**Residual risk:** During TradFi market closures (weekends, holidays), oracle feeds for equities and indices may not update. The protocol handles this by transitioning to Stress state, which disables all mints and redeems and releases the DEX liquidity pools from their peg. During Stress, the Keeper deploys wide, laddered liquidity with dead zones on Orca, allowing the synthetic tokens to trade as a free floating market based purely on supply and demand. This preserves 24/7 tradability at the cost of wider spreads. When the oracle comes back online, the protocol enters Recovery mode and the Keeper gradually re pegs the market over a 1 hour Dutch auction window.

### 8.4 Keeper Failure Scenarios

**Risk:** If all keepers go offline, the protocol loses its peg maintenance mechanism.

**Impact:** L and S token prices on DEXs could drift from NAV. Oracle observations stop being updated on chain. TWAP values become stale.

**Mitigations:**

* Keepers are stateless and can be restarted without data loss since all state is on chain.
* Multiple keepers can run simultaneously without conflict.
* The protocol remains solvent regardless of keeper status. Keepers affect peg accuracy, not collateral safety.
* Users can always redeem directly through the protocol (bypassing DEX prices) using on chain NAV, even without keeper operation.
* Risk state automatically transitions to Passive/Stress when oracle observations stop, restricting operations and widening protective spreads.

**Fallback:** Even without keepers running, any market participant can perform the arbitrage manually. The mechanism is permissionless. Keepers simply automate what any sophisticated user could do themselves.

### 8.5 Known Risks and Acceptance

| Risk | Severity | Mitigation | Residual |
|---|---|---|---|
| Stablecoin issuer freeze at collateral layer | High | Multi vault separation, provenance tracking | Cannot eliminate; fundamental to stablecoin design |
| Oracle staleness during TradFi closures | Medium | Risk state transitions, TWAP buffering, worst case quoting | Prices reflect last known value, not live |
| CLMM gamma risk at range boundaries | Medium | Dead zones, laddered ranges, adaptive width, inventory budgets | Large price jumps can still cause rapid inventory conversion |
| Smart contract bug | High | Anchor framework, checked math, planned audit | Pre audit risk remains until completion |
| Keeper centralization | Medium | Stateless design, permissionless arb | Single operator initially; plan to decentralize |
| Low liquidity at launch | Medium | Founding Cohort incentives, keeper LP management | Cold start problem for any new market |

***

## 9. Go To Market Strategy

Continuum's growth strategy focuses on high leverage community building, strategic partnerships, and a decentralized operational model. 

### 9.1 Bootstrapping and Single Market Start

Continuum plans to launch by seeding **a single anchor market** initially (e.g. NASDAQ 100 or NVDA). By isolating the launch to one highly liquid, recognizable asset, the protocol concentrates its initial total value locked (TVL) rather than spreading it thinly across multiple illiquid pools. 

**Capital Requirements:**
* **Strict Technical Minimum:** ~$10 USDC for base collateral logic, plus ~$100 USDC for the CLP Vault reserve.
* **Viable Minimum:** ~$10,000 to $25,000 USDC per market. Of this, roughly 30% to 40% goes to minting L and S token pairs (deposited as collateral into the protocol vault), 30% to 50% goes to seeding Orca Whirlpool liquidity pools (paired with the minted tokens), and the remainder stays in the CLP vault as an inventory buffer and is deployed to MarginFi to generate yield.
* **Ideal Bootstrapping:** ~$100,000 USDC per market. This allows the keeper to maintain deep, laddered liquidity bands that can handle institutional or whale flow without easily breaching the inventory risk budget or draining the flash loan arbitrage pools.

Once the first market establishes consistent volume, organic CLP deposit growth via the protocol's `/earn` platform will fund the launch of subsequent markets (like XAU or SPX).

### 9.2 Regulatory Strategy & Offshore Structuring

Operating synthetic derivatives that mirror real world assets necessitates a robust, proactive regulatory stance. Continuum approaches this structurally, drawing lessons from the trajectories of platforms like Uniswap, Polymarket, and Hyperliquid.

1. **Protocol Immutability:** Continuum's smart contracts have no freeze authorities on tokens, no delegate controls, and are designed to be explicitly immutable. The protocol is a base layer primitive, not an active trading venue.
2. **Geoblocking and Access Control:** While the smart contracts are permissionless, the official frontend interface will enforce strict IP based geoblocking for restricted jurisdictions (such as the US, UK, and others as legally advised) to prevent retail access from regions with hostile regulatory frameworks.
3. **Legal Entity:** Currently, Continuum operates purely as a decentralized application (dApp) with no formal legal entity. In the future, to mitigate regulatory problems, a robust company structure (such as a Delaware LLC or an entity in a crypto friendly jurisdiction like the Cayman Islands or Marshall Islands) will be established to shield the core developers and operations from domestic regulatory overreach. This mimics the successful defensive posturing used by decentralized perpetuals exchanges and prediction markets.

### 9.3 Marketing & Narrative Generation

Polymarket demonstrated that the most effective marketing for on chain markets is the market data itself. Continuum will focus on creating news artifacts out of extreme price movements during TradFi closures:
* Highlighting massive price gaps in synthetic NDX or NVDA over the weekend compared to Friday's close.
* Showcasing real time, on chain price discovery during Sunday night geopolitical events when traditional brokerages are inaccessible.
* Distributing visual content (like the CLMM laddered liquidity diagrams) that explains how Continuum maintains continuous, 24/7 liquidity mechanically when traditional markets shut off.

Web3 is fundamentally about communities. However, our immediate focus is entirely on seeding the first successful anchor market with deep liquidity. Once the core product is live and functioning, we will build organic, community driven growth:
* **Educational Content:** Creating high quality, mechanism focused content to explain the structural advantages of Continuum (e.g. zero funding rates, no freeze authority) to crypto native active traders and DeFi builders.
* **Borrowed Trust Partnerships:** Partnering with established voices in the Solana ecosystem, DeFi educators, and protocol teams to build credibility through association and co created content.
* **Shareable Artifacts:** Producing data driven tools like a "TradFi Hours Penalty Tracker" and "Synth Control Surface Scorecard" that earn distribution by being genuinely informative and useful to the broader ecosystem.

### 9.4 Target Audience & Investors

The initial outreach targets active crypto native traders, Solana DeFi protocol integrators, and institutional influencers who understand the limitations of existing synthetic and perpetual models.

**Capital Formation Strategy:**
* **Strategic Angels:** Initial capital formation will focus exclusively on strategic angel investors who can provide liquidity, integration opportunities, and ecosystem distribution. We are actively avoiding crypto VCs to ensure alignment with the protocol's ethos and to prevent early value extraction. This mirrors Hyperliquid's approach, which famously bootstrapped to over $1B in TVL with zero VC funding, proving that a protocol with strong product market fit does not need institutional capital to scale.
* **Future Token Launch and Points System:** We are not considering an Initial Coin Offering (ICO) or token launch yet until the core product is live and thoroughly battle tested. When the token (CNTM) is eventually launched, the distribution will be based on a points system determined by early participation. The points program will reward genuine value creation (like depositing liquidity, trading volume, or building integrations) rather than vanity farming.

### 9.5 Horizontal Company Structure

Continuum operates with a flat, web3 native team structure. Marketing, protocol development, and ecosystem growth run as parallel workstreams without traditional hierarchy. Decisions are made close to execution, empowering contributors to act swiftly. Contributors are incentivized by outcomes (protocol growth, TVL, volume) rather than tenure. The "Founding Cohort" of early participants is treated as an extension of the core team, earning recognition and rewards for meaningful actions that bootstrap the protocol's liquidity and security.

***

## 10. Competitive Landscape

### 10.1 Landscape Overview

The synthetic asset and tokenized finance space includes four categories of competitors, each with distinct strengths and structural limitations.

### 10.2 Structured Comparison

| Dimension | Continuum | Synthetix | GMX / GNS (Perps) | Drift / Jupiter Perps | Ondo / Backed / Hashnote (Tokenized RWA) | TradFi Brokerages |
|---|---|---|---|---|---|---|
| **24/7 availability** | Yes | Yes | Yes | Yes | Limited (business hours for some) | No (market hours only) |
| **Issuer freeze authority** | None (null) | Council controlled | N/A (positions) | N/A (positions) | Yes (standard) | N/A |
| **Funding rate exposure** | None (structural zero) | Yes (on synths) | Yes (perps) | Yes (perps) | None | None |
| **On chain collateral** | Yes (program owned vaults) | Yes (staking pool) | Yes (GLP/pool) | Yes (pool) | No (off chain custody) | No |
| **Collateral verifiable** | Yes (any explorer) | Yes | Yes | Yes | No (trust custodian) | No |
| **Token composability** | Full (standard SPL) | Limited (sUSD ecosystem) | None (positions) | None (positions) | Limited (restrictions) | None |
| **Tokens tradeable if protocol pauses** | Yes | No | No | No | No | N/A |
| **Solana native** | Yes | No (Ethereum/OP) | No (Arbitrum) | Yes | Partial | No |
| **Transfer restrictions** | None | None (but council control) | N/A | N/A | Yes (KYC gated) | Yes (brokerage account) |
| **TradFi asset coverage** | Yes (primary focus) | Limited | Limited | Limited | Yes (primary focus) | Yes |

### 10.3 Key Differentiators

**vs. Perpetual futures protocols (GMX, Drift, Jupiter Perps):** These protocols solve the 24/7 access problem but impose funding rates that erode long term positions. They create positions, not tokens. You cannot take your position and use it as collateral elsewhere. Continuum creates composable tokens with zero funding rate.

**vs. Synthetix:** Synthetix pioneered on chain synthetics but operates with a council controlled system on Ethereum/Optimism. The sUSD ecosystem is powerful but siloed: synthetic tokens have limited composability outside the Synthetix ecosystem. Continuum is Solana native with standard SPL token composability.

**vs. Tokenized RWA issuers (Ondo, Backed, Hashnote):** These provide legitimate TradFi exposure but with full issuer control: freeze authority, KYC gated transfers, and off chain custody. They serve institutional compliance needs but negate the sovereignty benefits of on chain finance. Continuum provides similar asset exposure without the control surface.

**vs. TradFi brokerages:** Traditional brokerages provide the deepest liquidity and regulatory clarity but cannot offer 24/7 access, on chain composability, or permissionless trading. Continuum is complementary for users who want exposure during hours when their brokerage is closed.

***

## 11. Traction & Roadmap

### 11.1 Current Stage

**Pre launch / Devnet.**

| Component | Status |
|---|---|
| Mint/Redeem program | Built, deployed to Solana devnet |
| CLP program | Built, tested on localnet |
| Oracle program | Built, tested on localnet |
| Frontend | Complete (all pages functional) |
| Off chain keeper | Complete (Rust, 6 concurrent modules) |
| Market initialization tooling | Complete |
| Liquidity seeding tooling | Complete |
| Token metadata | Complete for initial markets |

Three on chain programs are built and tested. The core minting engine is deployed to Solana devnet. Remaining programs are pending devnet deployment (operational blocker, not technical).

### 11.2 Roadmap

**2026 Goal:**

Our sole focus for 2026 is to successfully seed and launch our first anchor market with deep liquidity.

* Deploy all programs to devnet
* Initialize a single launch market (e.g. QQQ) to prove the primitive
* Seed concentrated liquidity for L/USDC and S/USDC pairs
* Begin keeper operation on devnet
* Mainnet deployment
* Phase 1 launch: single anchor market, conservative OI cap ($10M)

Once we achieve this goal, we will expand to 10 to 15 markets and decentralize the Keeper network.

### 11.3 KPI Ladder

| Stage | Metric | Target |
|---|---|---|
| Testnet | Partner sourced growth (protocol integrations in pipeline) | 3 to 5 active conversations |
| Mainnet Phase 1 | TVL (total collateral locked) | $1M to $10M |
| Mainnet Phase 2 | TVL | $10M to $50M |

***

## Appendix A: Architecture Overview

The system consists of three layers:

**Application layer (frontend).** A Next.js web application providing mint/redeem, trading, portfolio, and `/earn` (CLP) interfaces. Connects to Solana via standard wallet adapters.

**Protocol layer (on chain programs).** Three Solana programs handling minting/redemption, continuous liquidity provision (CLP), and oracle integration. All programs are written in Rust using the Anchor framework. The mint/redeem engine is the core: it enforces the L+S=collateral invariant, applies risk based fees, and manages multi collateral vaults.

**Infrastructure layer (keepers + external protocols).** A Rust based keeper runs six concurrent tasks: oracle updates, risk monitoring, concentrated liquidity management (Orca Whirlpools), arbitrage execution (using MarginFi flash loans), inventory monitoring, and yield harvesting. External protocol dependencies include Pyth (oracle), Orca (liquidity), and MarginFi (flash loans + yield).

## Appendix B: Risk State Machine

The protocol operates in four risk states, with automatic transitions based on oracle health:

**Normal.** Oracle healthy, confidence tight, feeds fresh. Tight spreads, standard fees (1.0x), full quoting available.

**Proxy Mode.** Oracle confidence widening (1 to 3%) or staleness increasing. Wider spreads, elevated fees (1.5x), TWAP based user quoting.

**Stress.** Oracle confidence high (3%+) or significantly stale (e.g. over the weekend). During Stress, the protocol disables mints and redeems entirely. The Keeper stops trying to peg the market and instead deploys extremely wide, laddered liquidity with dead zones on the DEX. This turns the synthetic asset into a **free floating market** with no peg, allowing price discovery to continue based purely on supply and demand until the oracle comes back online.

![CLMM Dead Zones Diagram](https://blobs.continuum.markets/clmm_dead_zones.webp)

**Recovery.** Confidence returning after a stress event. The fee multiplier starts at 3.0x and linearly decays to 1.0x over a 1 hour Dutch auction window to prevent MEV attacks on state transitions. Liquidity bands are progressively restored to their Normal state positions.

Transitions between states are triggered by the keeper's risk monitor based on oracle health metrics. The system degrades gracefully rather than failing abruptly. Fees increase and spreads widen before operations are restricted.

## Appendix C: NAV Mathematics

**Long NAV** equals the current oracle price of the underlying asset.

**Short NAV** equals the constant product (initial Long price multiplied by initial Short price) divided by the current oracle price.

This creates the following properties:

* As the underlying price rises indefinitely, Short NAV approaches zero asymptotically but never reaches it (no liquidation cliff)
* As the underlying price falls toward zero, Short NAV rises without bound
* At any price, Long NAV multiplied by Short NAV equals the constant product set at market initialization
* For equal quantities of L and S tokens, value changes are perfectly zero sum

**Worst case quoting** adjusts the NAV for minting and redemption by adding or subtracting the oracle confidence interval (scaled by the risk state multiplier), creating a natural bid ask spread that widens during uncertainty and narrows during stable conditions.

***

*This document was prepared based on direct inspection of the Continuum Markets protocol design and implementation.*

*continuum.markets*