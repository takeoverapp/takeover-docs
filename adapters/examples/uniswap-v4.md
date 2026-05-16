# Example: `V4YieldAdapter`

> **Tier 2 placeholder.** Full annotated walkthrough of the V4 adapter.
>
> **Live v2.3 deployments:**
> mainnet [`0x06e31B4EfCCdb8450a2e29C590A4AF81843d3172`](https://basescan.org/address/0x06e31B4EfCCdb8450a2e29C590A4AF81843d3172),
> Sepolia [`0xAd97b449277de911d740026087547c582AA1FB59`](https://sepolia.basescan.org/address/0xAd97b449277de911d740026087547c582AA1FB59).
> See [Deployments](../../reference/deployments.md) for the full
> address book.

## Pattern: V4 modifyLiquidities + IPoolSwap

Uniswap V4 LP positions don't have a `collect()` like V3. Fees come out via
a 0-liquidity `modifyLiquidities` call with `DECREASE_LIQUIDITY` + `TAKE_PAIR`
actions. Non-yield-side tokens are swapped through the position's own pool
via `IPoolSwap`.

> Source: [`takeoverapp/takeover-contracts/src/V4YieldAdapter.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/V4YieldAdapter.sol).

Highlights to read directly:

- `_collectSingle` — the modifyLiquidities encoding for the fee-collect.
- `_convertToYield` — swap routing using the position's own `PoolKey`.
- `_toYieldUnits` — `pendingYield` math that walks `sqrtPriceX96` without
  overflow, using two-step `fullMulDiv`.
- `increaseLiquidity` — owner-controlled add-liquidity path that lets the
  grid creator top up the position without un-registering.

A walkthrough mirroring [the Flaunch example](flaunch.md) will land in Tier 2.
