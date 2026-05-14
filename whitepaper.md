# 📜 Takeover: A Perpetually Contestable Market for Fee Rights

> **Note for developers.** This whitepaper describes the *protocol design*.
> For the current production implementation — UGM v2.3, the
> `IYieldAdapter` and `IGridHooksV23` interfaces, the live adapter set,
> and addresses — read alongside
> [What is Takeover](overview/what-is-takeover.md),
> [The yield adapter model](overview/yield-adapter-model.md),
> [The hook model](overview/hook-model.md),
> and [Deployments](reference/deployments.md). The "Implementation
> status" appendix at the bottom of this page tracks drift between the
> design and the deployed contracts.

**Abstract.** Takeover introduces a mechanism for allocating onchain trading fee distribution rights through continuously contestable ownership. Each liquidity pool's fee output is partitioned into a configurable number of positions ("seats"). Seat holders publicly declare a price for their seat and pay a continuous tax proportional to that price. Any participant may acquire any seat at its declared price — the holder cannot refuse. This creates a system where fee rights are always priced, always acquirable, and always contested. No order books, no market makers, no waiting for a counterparty. Just a price, a tax, and the constant threat that someone else wants your seat more than you do.

---

## 1. Introduction

Decentralized exchanges generate enormous trading fee revenue. Traditionally, participation in those fees is inseparable from providing liquidity — an LP must maintain a position to earn fees and must withdraw entirely to exit. There is no way to separately own, price, or contest the right to receive fees from a pool.

Takeover separates **fee distribution rights** from **liquidity provision**.

Liquidity provision continues to occur entirely within the underlying AMM. Takeover manages the allocation and contestation of the fee distribution rights associated with those pools. These rights are represented by **seats** — each entitling its holder to a fixed portion of the pool's fee output for as long as they can defend the position.

Seat ownership is governed by a **Harberger property system**: holders must publicly declare the value of their position, pay a continuous tax on that declared value, and accept that anyone can acquire their seat at the posted price. The holder cannot refuse a purchase. This is the core tension that makes the system work.

**Price too low?** Someone takes your seat and your fee stream with it.
**Price too high?** The tax drains your deposit and you lose the seat anyway.

The only sustainable strategy is honest pricing — declaring a value that accurately reflects what the fee stream is worth to you. Across all seats in a pool, these individual assessments aggregate into a continuously updated, mechanism-driven valuation of the pool's fee-generating capacity.

The result is not a passive financial instrument. It is a competitive, perpetually contested allocation system where participants must actively price, defend, and manage their positions — or lose them.

### 1.1 Contribution

Takeover combines three mechanisms into a single primitive:

1. **Fee Partitioning** — a pool's fee output is divided into a configurable number of independently-owned seats (currently bounded `[4, 4096]` per grid; see §6.1).
2. **Self-Assessed Ownership** — seat holders set their own price, and any participant may acquire the seat at that price.
3. **Continuous Taxation** — a tax proportional to the declared value discourages excessive valuation and maintains contestability.

Together, these create a system in which ownership of fee distribution rights is permanently open to challenge and reallocation.

---

## 2. Mechanism Design

### 2.1 Seats and Fee Partitioning

When a liquidity pool is registered with Takeover, its fee output is divided into a creator-chosen number of **seats**, bounded in `[MIN_TOTAL_SEATS, MAX_TOTAL_SEATS] = [4, 4096]`. Each seat entitles its holder to a fixed share of the pool's fee distributions while they maintain the position; under the default equal-share allocation that share is `1/totalSeats` per seat.

Seats are not tokens. They are non-transferable positions within the Takeover contract — a holder cannot list a seat on an exchange, wrap it in another protocol, or transfer it peer-to-peer. Ownership changes occur exclusively through the Harberger mechanism described below. This is a deliberate constraint: it ensures that all seat transfers happen under the same pricing and taxation rules, and that the contestability properties of the system cannot be circumvented.

Fees accumulate in the grid's configured `yieldToken` denomination (ETH, flETH/WETH, or any guardian-allowlisted ERC20 such as USDC) and are claimable by the seat holder at any time.

### 2.2 Harberger Pricing

Seat ownership follows a Harberger property regime defined by two rules:

1. **Self-assessment** — the seat holder declares a public price for the seat in the grid's tax token.
2. **Forced transfer** — any participant may acquire the seat by paying the declared price. The holder cannot refuse.

These two rules produce a tension that drives honest pricing:

- **Underprice** and a competitor buys the seat at a bargain, capturing the fee stream at below its true value. The loss is immediate and irreversible.
- **Overprice** and the continuous tax accelerates, draining the holder's deposit. The loss is gradual but relentless.

The rational response to this tension is to declare a price that reflects the holder's genuine assessment of the seat's value. The mechanism transforms pricing from a negotiation problem — where both parties posture — into an optimization problem with a clear equilibrium.

#### 2.2.1 Tax Mechanics

Each seat holder maintains a deposit in the grid's tax token from which tax is continuously deducted. The tax rate is expressed in basis points per week, applied to the self-assessed price:

```
tax_per_second = (price × tax_rate_bps) / (seconds_per_week × 10000)
tax_owed       = tax_per_second × elapsed_seconds
```

The protocol-wide tax rate is configurable (post-governance upgrade) within bounds of 0.1%–10% per week (10–1000 bps). The default rate is 5% per week (500 bps).

**Example.** A seat priced at 1,000 USDC at the default 5% weekly rate costs approximately 7 USDC per day in tax. The holder must maintain a sufficient deposit to cover this ongoing obligation or face forfeiture.

The tax rate calibrates the system's sensitivity. Higher rates penalize overpricing more aggressively, pushing declared values closer to true assessments. Lower rates give holders more runway but reduce contestability.

#### 2.2.2 Forfeiture

If a seat holder's deposit is exhausted — accumulated tax exceeds the remaining balance — the seat is **forfeited**:

1. All remaining deposit is collected as tax.
2. Any accrued but unclaimed fees are transferred to the holder.
3. The seat becomes vacant and claimable by any participant.

Forfeiture is the system's backstop. It guarantees that abandoned or neglected positions eventually return to open contestation. A holder who sets a price and walks away will, with certainty, lose the seat. The mechanism rewards active participation and punishes passivity.

#### 2.2.3 Forced Acquisition

Any participant can acquire any seat by paying its declared price in the grid's tax token:

1. The buyer pays the current price. Any amount above the price becomes the buyer's initial deposit.
2. A tile sale fee is deducted from the sale price as protocol revenue.
3. The previous holder receives the price minus the tile sale fee, plus their remaining deposit.
4. The buyer sets a new self-assessed price and begins paying tax on it.

A **20-minute price-change cooldown** prevents holders from front-running an incoming acquisition by briefly spiking their price.

#### 2.2.4 The Pricing Equilibrium

Consider a seat generating *f* in weekly fees. A rational holder prices the seat at approximately:

```
P* ≈ f / (r + τ)
```

Where *r* is the holder's opportunity cost of capital and *τ* is the tax rate. Setting the price above *P\** increases the tax burden faster than it increases acquisition protection. Setting it below *P\** invites acquisition at a price that doesn't compensate for the lost fee stream.

This equilibrium emerges naturally from the mechanism's incentives. No one needs to compute it explicitly — the twin pressures of acquisition risk and tax cost push participants toward honest valuation through trial, error, and competitive pressure.

---

## 3. Properties

### 3.1 Perpetual Contestability Without Market Infrastructure

Traditional markets for fee-generating rights require liquidity providers, order books, or AMMs to facilitate exchanges. If no one is willing to buy or sell at a given price, the market is illiquid and positions are stranded.

Takeover eliminates this dependency entirely. Every seat has a posted price. Every seat can be acquired at that price. There is no spread, no slippage, and no dependence on external liquidity. The system is, by construction, always contestable — any seat can always be challenged by a participant willing to pay the declared price.

Exit for existing holders works through repricing: a holder who wants to leave can lower their price to a level where acquisition becomes attractive to others, or simply abandon the seat and reclaim their remaining deposit. Exit does not require finding a counterparty. It requires only repricing.

### 3.2 The Cost of Mispricing

The Harberger mechanism penalizes mispricing asymmetrically:

| Direction | Consequence | Character |
|-----------|------------|-----------|
| **Price too low** | Seat is acquired; holder loses fee stream at below-value price | Immediate, irreversible |
| **Price too high** | Excessive tax drains deposit; eventual forfeiture if uncorrected | Gradual, correctable |

This asymmetry has a behavioral consequence: rational holders will tend to slightly overprice rather than underprice, because overpricing can be corrected (lower the price) while underpricing is catastrophic (you lose the seat). The tax rate calibrates this asymmetry — higher rates make the cost of overpricing steeper, compressing declared prices toward true assessments.

### 3.3 Aggregate Price Discovery

Across every seat in a pool, the aggregate of all declared prices represents the participants' collective assessment of the pool's fee-generating capacity. Each seat price encodes an individual expectation about the pool's future activity, and the sum of these prices produces a continuously updated, mechanism-driven signal.

This signal emerges without any explicit forecasting instrument. Participants are simply pricing their own positions under competitive pressure, and the aggregate reveals information that no individual participant is trying to produce.

### 3.4 Active Ownership

Unlike passive fee-receiving arrangements, Takeover seats require continuous engagement. A holder must:

- **Monitor** the pool's fee output relative to their tax obligation
- **Adjust** their declared price as conditions change
- **Maintain** a sufficient deposit to avoid forfeiture
- **Defend** against acquisition by pricing credibly

A participant who acquires a seat and ignores it will lose it — either through forfeiture (if fees don't justify the tax) or through acquisition (if the price is stale). This active management requirement is a feature, not a limitation. It ensures that seat ownership reflects current conviction, not historical accident.

---

## 4. The Competitive Dynamic

Takeover is a competitive system. The Harberger mechanism creates a persistent tension between seat holders and potential acquirers that produces emergent game dynamics.

### 4.1 Price It or Lose It

Every seat holder faces a continuous dilemma: price high enough to deter acquisition, but not so high that the tax becomes unsustainable. This creates an ongoing strategic calculation that is simple to understand but difficult to optimize — the hallmark of a compelling game mechanic.

When a pool's trading volume spikes, seat holders must decide: raise the price (increasing tax but protecting against acquisition) or hold steady (saving on tax but risking losing the seat to someone who values the higher fee stream). When volume drops, they face the inverse: lower the price (reducing tax but inviting opportunistic acquisition) or maintain conviction (paying above-market tax in expectation of recovery).

### 4.2 The Acquisition Game

Acquiring a seat from another participant is an aggressive, public act. The acquirer pays the declared price, displaces the previous holder, and sets a new price — broadcasting their own assessment of the seat's value. A sequence of acquisitions on a single seat reveals the market's evolving consensus about that pool's fee potential.

The tile sale fee imposes a cost on frequent flipping, discouraging pure arbitrage and rewarding participants who hold conviction.

### 4.3 Portfolio Strategy

Participants can hold seats across multiple pools, constructing a portfolio of fee rights with varying risk profiles. A high-volume established pool offers stable but competed-over fee streams. A newly launched token pool offers higher potential returns but greater uncertainty. The Harberger mechanism ensures that every position in this portfolio is independently contestable — there is no way to passively sit on a diversified basket without actively defending each position.

---

## 5. Comparison with Existing Fee Distribution Mechanisms

### 5.1 Traditional LP Positions

In standard AMMs, fee distribution rights are bundled with liquidity provision. To receive fees, you must provide liquidity. To stop receiving fees, you must withdraw liquidity. There is no way to separately own, price, or trade the fee component.

Takeover unbundles this. The LP position remains in the AMM. The fee distribution rights are separately allocated through the seat mechanism. This separation allows participants who have conviction about a pool's fee potential to express that conviction without providing liquidity themselves.

### 5.2 Pendle Finance

Pendle is the most prominent existing system for separating fee/yield rights from underlying positions. It splits yield-bearing tokens into Principal Tokens (PT) and Yield Tokens (YT), and operates an AMM for trading them.

Both Pendle and Takeover create systems for separately allocating fee distribution rights. The mechanisms and resulting properties differ substantially:

| Property | Pendle | Takeover |
|----------|--------|----------|
| **Underlying** | Yield-bearing tokens (stETH, GLP, etc.) | AMM trading fee streams |
| **Allocation mechanism** | AMM with bonding curve | Harberger self-assessed pricing |
| **Duration** | Fixed expiry (30 days, 6 months, etc.) | Perpetual — no expiry |
| **Contestability** | Depends on AMM liquidity depth | Intrinsic — every seat is always acquirable |
| **Exit mechanism** | Sell on Pendle AMM (requires buyer) | Reprice, get acquired, or abandon |
| **Participant experience** | Analytical, financial | Competitive, strategic |
| **Position representation** | Freely tradable ERC-20 tokens | Non-transferable contract positions |
| **Capital requirement** | Purchase price of YT | Seat price + ongoing tax deposit |

#### 5.2.1 The Contestability Difference

Pendle's primary scaling constraint is AMM liquidity. A Pendle market for a low-activity underlying often has wide spreads, shallow depth, and poor execution. Contestability depends on whether someone is willing to provide liquidity on the other side.

Takeover's contestability is structural, not liquidity-dependent. A pool with a single seat holder is just as contestable as a pool with 10,000 active participants. The Harberger mechanism guarantees that every seat can be acquired at its posted price, regardless of how many other participants are active. Contestability is a property of the mechanism, not of participation.

#### 5.2.2 Perpetual vs. Fixed Duration

Pendle's yield positions have fixed maturities. A 6-month YT expires after 6 months, requiring the holder to actively manage rollovers across maturities. This creates friction and fragmented liquidity across maturity tranches.

Takeover seats are perpetual. A holder receives their share of fees indefinitely, as long as they maintain their deposit and defend their price. There is no expiry, no rollover, and no maturity management. The position persists until the holder exits, is acquired, or is forfeited.

#### 5.2.3 The Engagement Difference

Pendle is designed as a financial protocol for sophisticated DeFi participants. Its core interaction — analyzing yield curves, splitting tokens, and trading on a specialized AMM — is abstract and analytical.

Takeover wraps comparable economic dynamics in competitive mechanics. Acquiring a seat from another participant, defending a position by maintaining a credible price, watching fee distributions accumulate — these interactions are strategic and social in a way that AMM-based systems are not. The Harberger mechanism ("price it or lose it") is inherently competitive, creating engagement loops accessible to participants who would never interact with a yield curve.

---

## 6. System Architecture

### 6.1 Unified Grid Manager

All grid types are managed by a single immutable contract: the **Unified Grid Manager (UGM)**. Rather than separate contracts for each pool type, the UGM handles every yield source through a unified engine, with source-specific logic isolated in adapter contracts behind a single standard interface.

Each grid has a single **yield token** denomination — `address(0)` for ETH, or an ERC20 address (e.g., USDC). A single grid can aggregate multiple yield-producing assets — several Flaunch tokens, multiple LP positions, or a mix — provided they share the same yield denomination.

```
Flaunch coin       → FlaunchYieldAdapter   ┐
Uniswap V3 LP NFT  → V3YieldAdapter        ├──→ UGM ──→ 100 seat holders
Uniswap V4 LP NFT  → V4YieldAdapter        │
Protocol fees      → ProtocolYieldAdapter  ┘
```

Every adapter implements the same `IYieldAdapter` interface (`collectYield`,
`pendingYield`); the UGM does not need to know anything source-specific. New
yield sources can be onboarded without modifying the UGM — see
[Building a yield adapter](adapters/building-an-adapter.md).

### 6.2 Supported Yield Sources

The current production adapters are:

- **Flaunch tokens** — `FlaunchYieldAdapter`
- **Uniswap V3 LP NFTs** — `V3YieldAdapter` (and V3-compatible forks: Aerodrome, PancakeSwap V3)
- **Uniswap V4 LP NFTs** — `V4YieldAdapter`
- **Protocol fees** (push-only) — `ProtocolYieldAdapter`, used by the Boardroom (hvTAKEOVER) grid

Adding a new yield source is a permissionless engineering task — implement
`IYieldAdapter`, get the contract audited, and submit it for guardian
approval. See [Adapter approval process](submit/adapter-approval-process.md).

### 6.3 Immutability and Versions

Each UGM is deployed as an immutable contract with no proxy upgrade mechanism. If a new version is needed, a new UGM is deployed and users migrate via migration zaps. The live integration target is **UGM v2.3** on Base mainnet and Base Sepolia; older versions remain readable on-chain for historical grids.

A single **guardian** role exists for emergency operations (pausing grids, rescuing stuck tokens), fee parameter tuning within bounds, and adapter approval (`setApprovedAdapter`). The guardian cannot modify core Harberger mechanics.

---

## 7. Grid Creator Economics

The grid creator — the address that launched the grid and registered the underlying assets — holds a distinct role in the system:

1. **Creator-owned seats.** At grid creation, the creator owns every seat (`totalSeats`, configurable in `[4, 4096]`) with independently set prices. Creator-held seats are **tax-exempt** and earn yield proportionally.
2. **Immediate Harberger.** When a user buys a seat from the creator, the buyer sets their own price and enters full Harberger mechanics immediately — earns yield, pays tax, can be bought out. There is no separate "pre-market" phase.
3. **Tax revenue.** The creator receives a share of all Harberger tax revenue from their grid, routed through the `GenericBuybackKeeper` — which either swaps tax tokens for a creator-specified target token or forwards them directly to the creator's address. Seat holders trigger these transfers as keepers, earning a small reward.
4. **Full withdrawal.** If the creator reacquires every seat on the grid, they may withdraw the underlying assets from the grid.

---

## 8. Revenue Structure

Seat activity generates two protocol revenue streams: **tile sale fees** deducted on each acquisition, and the protocol's share of **Harberger tax** from all grids.

Protocol revenue is periodically converted through a permissionless onchain path:

```
Tax Token → Keeper executes swap → TAKEOVER → Configurable receiver
```

Any seat holder can trigger this conversion as a keeper, earning a small reward. Creator tax revenue is handled separately through the `GenericBuybackKeeper` (see §7).

---

## 9. Conclusion

Takeover is a new primitive for allocating onchain fee distribution rights. By applying Harberger-style property mechanics to fractional fee positions, it achieves properties that are difficult or impossible to produce with traditional market structures:

- **Permanent contestability** without market makers, order books, or AMMs.
- **Incentive-aligned pricing** where misvaluation carries direct and asymmetric cost.
- **Perpetual positions** with no expiry, rollover, or duration management.
- **Aggregate price discovery** through the competitive pressure of self-assessed ownership.

The mechanism is general — any fee-generating onchain position can be partitioned into contestable seats (hint: this is not limited to AMM trading fees) — and the competitive dynamics make it accessible to participants far beyond the traditional DeFi audience.

Takeover does not ask participants to analyze curves, model term structures, or provide liquidity to an AMM. It asks a simpler and more visceral question: **what is this fee stream worth to you, and are you willing to defend that answer?**

---

## Appendix A. Implementation status (May 2026)

This section tracks divergence between the design described above and the
deployed contracts. It is the part of this document most likely to drift —
when in doubt, [Deployments](reference/deployments.md) wins.

| Concept (whitepaper section) | Implementation today |
|---|---|
| Unified Grid Manager (§6.1, §6.3) | **UGM v2.3** live on Base mainnet and Base Sepolia (`UnifiedGridManagerV23` + linked `UGMV23Linked` library) |
| Yield source onboarding (§6.1, §6.2) | Adapter contracts implementing [`IYieldAdapter`](adapters/interface.md), guardian-approved via `setApprovedAdapter`. Per-call gas budget: `ADAPTER_COLLECT_GAS_CAP = 600,000`. |
| Adapter set (§6.2) | `FlaunchYieldAdapter`, `V3YieldAdapter`, `V4YieldAdapter` (mainnet, on UGM v2.3); Sepolia adapter trio not yet redeployed against v2.3 |
| Hook modules (new) | UGM v2.3 ships [`IGridHooksV23`](hooks/interface.md), a strict superset of v2.2's `IGridGovernanceHooks`, used by games / DAO tooling / prediction markets to gate seat economics. Per-call gas budget: `GOVERNANCE_HOOK_GAS_CAP = 150,000`. |
| Tax token denomination (§2.2, §2.2.3) | Per-grid; configured at grid creation, must be guardian-allowlisted |
| Yield token denomination (§2.1) | Per-grid; ETH/flETH/WETH or any ERC20, set at grid creation and immutable |
| Seat counts (§2.1) | Configurable per grid in `[4, 4096]`. Atomic creation up to 1024; chunked creation (`createGridShell` + `appendInitialPricingChunk`) above that. See [boards/sharding.md](boards/sharding.md). |
| Guardian role (§6.3) | Multisig today; controls emergency ops, fee parameter bounds, two-axis pause (`pauseGrid` + `pauseGridModules`), and `setApprovedAdapter` / `setApprovedGovernanceModule` / `setApprovedModule` allowlists. |
| Migration between UGM versions | `MigrationZap` contracts (historical v1→v2; v2→v2.3 paths supported) |

---

*Takeover is live on Base. Protocol contracts are open source and verified onchain.*

*<https://takeover.fun>*
