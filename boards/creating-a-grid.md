# Creating a grid

> **Tier 2 placeholder.** Full reference for `createGrid` parameters and
> grid-creation patterns.

## Quick reference

```solidity
struct CreateGridParams {
    uint16 taxRateBps;       // 0 (default 500), or 100..1000 (1%–10% per week)
    uint16 totalSeats;       // 4..256, must be a perfect square
    address taxToken;        // Guardian-allowlisted ERC20
    address yieldToken;      // address(0) = ETH (also accepts flETH/WETH), or any ERC20
    uint128[] seatPrices;    // length == totalSeats; per-seat starting price
}

uint256 gridId = ugm.createGrid(params);
```

For a step-by-step walkthrough including pre-buy zaps and the
`setGridFeeReceivers` redirect, see [Wiring an adapter to a grid](wiring-an-adapter.md).

This page will be expanded with:

- Picking `totalSeats` (auction dynamics vs gas cost).
- Picking `taxRateBps` (revenue vs participation tradeoff).
- Picking `yieldToken` and how it constrains adapter choice.
- `seatPrices` defaults and the "fair start" recommendation.
- The `taxToken` allowlist process.
