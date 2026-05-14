# Example: `V3YieldAdapter`

> **Tier 2 placeholder.** Full annotated walkthrough of the V3 adapter.
>
> **Live v2.3 mainnet deployment:**
> [`0x523c20e9c05Ec64F5d6Ad38c653FE3C90a88E5A9`](https://basescan.org/address/0x523c20e9c05Ec64F5d6Ad38c653FE3C90a88E5A9).
> See [Deployments](../../reference/deployments.md) for current
> per-network status (Sepolia v2.3 adapters not yet redeployed).

## Pattern: two-token swap-and-forward

V3-compatible LP NFTs (Uniswap V3, Aerodrome, PancakeSwap V3) accrue trading
fees in **both** pool tokens. Whichever side isn't the grid's `yieldToken`
gets swapped through the position's own pool.

> Source: [`takeoverapp/takeover-contracts/src/V3YieldAdapter.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/V3YieldAdapter.sol).

What's worth reading right now:

- One adapter handles **multiple position-manager deployments** by storing
  the `positionManager` address per asset and using it in the asset hash.
- The non-yield side is swapped via `IV3SwapRouter`, not an external
  aggregator — that's a deliberate "no extra trust assumptions" choice.
- WETH yield is unwrapped to ETH when the grid's `yieldToken == address(0)`.

A walkthrough mirroring [the Flaunch example](flaunch.md) will land in Tier 2.
