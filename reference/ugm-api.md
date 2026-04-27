# UGM API reference

> **Tier 2 placeholder.** Reference of the UGM v2.1 functions an adapter or
> integrator calls.

## What an adapter calls on UGM

| Function | When | Reverts if |
|---|---|---|
| `registerAsset(gridId, assetHash)` | Inside `deposit`, after taking ownership of the source asset | adapter not approved; asset already registered; grid doesn't exist |
| `withdrawAsset(gridId, assetHash)` | Inside `withdraw`, after sweeping last yield | caller isn't the registering adapter; asset not in grid |
| `receiveYieldETH(assetHash, amount)` (payable) | Inside `collectYield` for ETH-yield grids | caller isn't the registering adapter; `msg.value != amount`; grid's yieldToken isn't `address(0)` |
| `receiveYieldERC20(assetHash, token, amount)` | Inside `collectYield` for ERC20-yield grids (or flETH/WETH on ETH grids) | caller isn't the registering adapter; wrong token for grid |

## What an integrator reads from UGM

| View | Returns |
|---|---|
| `gridConfig(gridId)` | `(creator, createdAt, totalSeats, taxRateBps, taxToken, yieldToken)` |
| `seats(gridId, seatId)` | `SeatInfo` (holder, price, deposit, timestamps, etc.) |
| `holderSeats(gridId, holder)` | List of seat IDs held by an address |
| `holderSeatCount(gridId, holder)` | Number of seats held by an address |
| `totalYield(gridId)` | Cumulative yield deposited to this grid |
| `assetToGrid(assetHash)` | Grid an asset is registered to |
| `assetAdapter(assetHash)` | Adapter responsible for an asset |
| `approvedAdapters(adapter)` | Whether an adapter is whitelisted |

## What a grid creator calls

| Function | Effect |
|---|---|
| `createGrid(params)` | Mints a new grid with the given config |
| `setGridFeeReceivers(gridId, taxReceiver, saleReceiver)` | Redirect tax/sale revenue away from the creator |
| `setSeatPrice(gridId, seatId, newPrice)` | Update a held seat's self-assessed price (subject to cooldown) |

This page will be expanded with full signatures, gas notes, and
cross-links to source.

> Source: [`takeoverapp/takeover-contracts/src/interfaces/IGrid.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/interfaces/IGrid.sol)
> and [`UnifiedGridManagerV21.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/archive/legacy-unified-grid/UnifiedGridManagerV21.sol).
