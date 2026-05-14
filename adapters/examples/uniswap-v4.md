# Example: `V4YieldAdapter`

> **Tier 2 placeholder.** Full annotated walkthrough of the V4 adapter.
>
> **Live v2.3 mainnet deployment:**
> [`0x50f30EE9543eC2e8905B29aC48180D5fC1Ee82Af`](https://basescan.org/address/0x50f30EE9543eC2e8905B29aC48180D5fC1Ee82Af).
> See [Deployments](../../reference/deployments.md) for current
> per-network status (Sepolia v2.3 adapters not yet redeployed).

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
