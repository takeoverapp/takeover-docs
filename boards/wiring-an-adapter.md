# Wiring an adapter to a grid

This page is for **grid creators** — once an adapter has been built, audited,
and approved by the Takeover guardian, what does *the protocol team launching
the board* actually have to do?

If you're an adapter author, see [Building a yield adapter](../adapters/building-an-adapter.md)
instead.

## Prerequisites

Before you wire anything:

- [ ] The adapter you want to use is **deployed** and **approved** on UGM
      v2.1. Check `ugm.approvedAdapters(adapter)` returns `true`.
- [ ] You know the **grid's `yieldToken`** — your adapter must produce that
      token. The reference adapters revert deposit if there's a mismatch.
- [ ] You hold the **source-side asset** the adapter expects (LP NFT,
      Flaunch NFT, etc.) in the same wallet that will create the grid.
- [ ] You're set as the grid's **creator** (you'll be by default if you call
      `createGrid`).

## Step 1: create the grid

Call `ugm.createGrid` with the right shape for your yield source:

```solidity
UnifiedGridManagerV21.CreateGridParams memory params = UnifiedGridManagerV21.CreateGridParams({
    taxRateBps: 500,                      // 5% per week. Range: 100..1000 (1%–10%), or 0 for default.
    totalSeats: 100,                      // 4..256, must be a perfect square.
    taxToken: USDC,                       // ERC20 holders pay tax in. Must be guardian-allowlisted.
    yieldToken: address(0),               // address(0) = ETH. Or: WETH, flETH, USDC, your token.
    seatPrices: initialSeatPrices         // length == totalSeats; per-seat starting price.
});
uint256 gridId = ugm.createGrid(params);
```

The two fields that lock you into a specific adapter:

- **`yieldToken`** — must match what your adapter forwards. flETH and WETH
  count as ETH for adapters that produce raw ETH yield; everything else is
  exact-match. See [Yield token rules](../adapters/yield-token-rules.md).
- **`taxToken`** — independent of the adapter. The guardian maintains an
  allowlist of accepted tax tokens; you pick from that list.

`totalSeats` and `taxRateBps` don't affect adapter wiring.

## Step 2: hand the source-side asset to the adapter

Each reference adapter exposes a `deposit(...)` flow. The exact arguments are
adapter-specific, but the shape is always:

> "Here's the source-side handle. Here's the `gridId`. Take it."

Examples:

```solidity
// Flaunch NFT
IERC721(flaunchAddress).approve(flaunchAdapter, tokenId);
flaunchAdapter.deposit(flaunchAddress, tokenId, gridId);

// Uniswap V3 / Aerodrome / Pancake V3 LP NFT
IERC721(positionManager).approve(v3Adapter, tokenId);
v3Adapter.deposit(positionManager, tokenId, gridId);

// Uniswap V4 LP NFT
IERC721(v4PositionManager).approve(v4Adapter, tokenId);
v4Adapter.deposit(tokenId, gridId);
```

Inside that single call the adapter:

1. Verifies you're the grid creator.
2. Verifies the grid's `yieldToken` is compatible with the source.
3. Pulls the source-side asset into itself.
4. Stores its bookkeeping baseline (so historical fees aren't credited as
   future yield).
5. Calls `ugm.registerAsset(gridId, assetHash)`.

If anything fails, the whole transaction reverts and you keep the asset.

## Step 3: verify

After deposit, sanity-check from any RPC:

```solidity
ugm.assetToGrid(assetHash) == gridId          // adapter is registered
adapter.assets(assetHash).active == true      // adapter has the asset
adapter.pendingYield(assetHash) >= 0          // returns without reverting
```

The indexer surfaces this same view via the `Grid.assets` GraphQL field once
it has crawled the `AssetRegistered` event.

## Step 4: pre-buy seats (optional)

The same wallet that just registered the asset can claim seats on the grid
in the same transaction via the `GridLaunchZap` if you want a full launch in
one call. See `script/13_DeployAdapters.s.sol` in `takeover-contracts` and
the per-protocol launch scripts for examples.

## Wiring to a tax-redirect (optional)

By default, tax revenue from the grid goes to the grid creator. To redirect
it (e.g. to a `BoardroomFarm` or your own treasury), call:

```solidity
ugm.setGridFeeReceivers(gridId, taxReceiver, saleReceiver);
```

`taxReceiver` is the contract that receives tax revenue when anyone calls
`ugm.withdrawTaxRevenue(gridId)`. If `taxReceiver` implements
[`IFeeReceiver`](../reference/ifeereceiver.md), UGM also calls
`notifyDeposit(gridId, amount)` on it after the transfer.

This is a different surface from the yield adapter — tax flows are separate
from yield flows. You can wire one, both, or neither.

## Multi-asset grids

You can register more than one asset to a single grid by calling deposit on
the same adapter multiple times (or on multiple adapters). All registered
assets contribute to that grid's `totalYield`. See
[Multi-asset grids](multi-asset-grids.md) for caveats.

## Where to next

- [Creating a grid](creating-a-grid.md) — full `CreateGridParams` reference.
- [Yield token rules](../adapters/yield-token-rules.md) — what `yieldToken`
  values pair with what adapters.
- [Deployments](../reference/deployments.md) — UGM and reference adapter
  addresses on each chain.
