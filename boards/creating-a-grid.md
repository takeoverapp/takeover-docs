# Creating a grid

A grid is a fixed-size set of seats under Harberger taxation. UGM
v2.3 supports three grid sizes:

- **4 – 1024 seats** → atomic [`createGrid`](#atomic-creategrid-≤1024-seats).
- **1025 – 4096 seats** → chunked [`createGridShell` +
  `appendInitialPricingChunk`](#chunked-creategridshell--appendinitialpricingchunk-1025-4096-seats).
- **4097+ seats** → SDK-side multi-grid sharding. See
  [Sharding boards above 1024 seats](sharding.md).

## The `CreateGridParams` struct

```solidity
struct CreateGridParams {
    uint16 taxRateBps;          // 0 (inherit DEFAULT_TAX_RATE_BPS = 500), or [100, 1000]
    uint16 totalSeats;          // [4, 4096]
    uint32 forfeitureDuration;  // 0 (inherit 10-min default), or ≤ 30 days
    address taxToken;           // guardian-allowlisted ERC20 (USDC by default)
    address yieldToken;         // address(0) for ETH, or any ERC20 the adapter pushes
    uint128[] initialPrices;    // length == totalSeats — per-seat creator-set listing
}

uint256 gridId = ugm.createGrid(params);
```

For chunked launches the shell is a struct without `initialPrices`:

```solidity
struct CreateGridShellParams {
    uint16 taxRateBps;
    uint16 totalSeats;
    uint32 forfeitureDuration;
    address taxToken;
    address yieldToken;
}

uint256 gridId = ugm.createGridShell(shellParams);
ugm.appendInitialPricingChunk(gridId, prices0);
ugm.appendInitialPricingChunk(gridId, prices1);
// …until _initialPricesSeatsWritten[gridId] == totalSeats.
```

Both paths emit the same `GridCreated` event with `gridId`,
`creator`, and the resolved field set. The chunked path additionally
emits `GridInitialPricesChunk(gridId, fromSeat, toSeat)` per chunk
and one `GridInitialPricingComplete(gridId)` when the last chunk
lands.

## Validation table

| Field | Rule | Source / revert |
| --- | --- | --- |
| `taxRateBps` | `0` or in `[100, 1000]` (1%–10% per week). `0` resolves to `DEFAULT_TAX_RATE_BPS = 500`. | `MIN_TAX_RATE` / `MAX_TAX_RATE` (`TaxRateOutOfBounds`) |
| `totalSeats` | `[4, 4096]`. Above 1024 reverts in `createGrid` with `TotalSeatsRequiresChunkedCreation`. | `MIN_TOTAL_SEATS` / `MAX_SINGLE_TX_CREATE_TOTAL_SEATS` / `MAX_TOTAL_SEATS` |
| `forfeitureDuration` | `0` or `≤ 30 days`. `0` resolves to `DEFAULT_FORFEITURE_DURATION = 10 minutes`. | `MAX_FORFEITURE_DURATION` (`ForfeitureDurationTooLong`) |
| `taxToken` | Must be on the guardian's `setAllowedTaxToken` allowlist. USDC is allowlisted by default. | `TaxTokenNotAllowed` |
| `yieldToken` | `address(0)` for native ETH, or any ERC20 the adapter you intend to attach pushes via `receiveYieldERC20`. | adapter-side gate |
| `initialPrices[i]` | `uint128`. Length MUST equal `totalSeats` on the atomic path. | `InitialPricesLengthMismatch` |

The atomic path packs every `initialPrices[i]` into
`_packedInitialPrices` two-per-slot; the chunked path expects the
same density, accumulated by `appendInitialPricingChunk` until
`_initialPricesSeatsWritten[gridId] == totalSeats`.

## Atomic `createGrid` (≤1024 seats)

This is the default path and what you want for almost every grid.

```ts
import { encodeFunctionData, parseUnits } from 'viem';
import { base } from '@takeover/sdk/networks';
import { walletClient, publicClient } from './clients';

const ugm = base.ugmV23;

const params = {
  taxRateBps: 500n,
  totalSeats: 256n,
  forfeitureDuration: 0,
  taxToken: base.usdcAddress,
  yieldToken: '0x0000000000000000000000000000000000000000',
  initialPrices: Array.from({ length: 256 }, () =>
    parseUnits('5', 6) as unknown as bigint, // 5 USDC each
  ),
};

const { request } = await publicClient.simulateContract({
  abi: ugmAbi,
  address: ugm,
  functionName: 'createGrid',
  args: [params],
});

const hash = await walletClient.writeContract(request);
const receipt = await publicClient.waitForTransactionReceipt({ hash });

const gridCreated = receipt.logs.find(
  (l) => l.address.toLowerCase() === ugm.toLowerCase(),
);
// Decode `GridCreated(gridId, creator, totalSeats, ...)` to recover
// the new gridId. The same value is also returned by createGrid().
```

Why `simulate` first: viem decodes UGM's custom errors
(`TotalSeatsRequiresChunkedCreation`, `InitialPricesLengthMismatch`,
…) into readable strings on the simulate path, so a config bug is
caught locally instead of producing an unsolicited on-chain revert.

## Chunked `createGridShell` + `appendInitialPricingChunk` (1025–4096 seats)

Use when `totalSeats > 1024`. A single `createGrid` tx exceeds the
chain-wide per-tx gas maximum (~50M measured at 4096 seats); the
chunked path lets each tx pack a sub-cap slice of `initialPrices`
into `_packedInitialPrices` two-per-slot until the grid is fully
priced.

Decision tree:

```
                 +--------------------+
                 |   totalSeats <= 4? |--no--+
                 +--------------------+      |
                            |                |
                           yes               |
                            |                v
                  REJECT (MIN_TOTAL_SEATS)   +--------------------+
                                              | totalSeats <= 1024? |--yes--> createGrid (atomic)
                                              +--------------------+
                                                          |
                                                         no
                                                          v
                                              +--------------------+
                                              | totalSeats <= 4096? |--yes--> createGridShell + N×appendInitialPricingChunk
                                              +--------------------+
                                                          |
                                                         no
                                                          v
                                              +--------------------+
                                              | SDK-side sharding   |
                                              | (boards/sharding.md)|
                                              +--------------------+
```

Until the last chunk lands, the grid is pricing-incomplete.
Seat-economic ops (`addBatch`, `claim`, `setPrice`,
`abandonBatch`, …) revert with `InitialPricingIncomplete()`.

See [boards/sharding.md](sharding.md) for the full chunked-launch
recipe.

## Picking the parameters

### `taxRateBps`

The continuous tax rate, in basis points per week. `500` = 5% per
week.

- Lower rates (1–3%) reward sticky holders. Good for
  collectible-style grids where most seats settle after the first
  rotation.
- Higher rates (5–10%) maximize Harberger churn. Good for
  campaign-style grids where you want the seat to constantly cycle.
- `0` inherits the protocol default of 5%. Fine for most cases.

### `totalSeats`

Pick by gameplay / asset density, not by gas.

- ≤ 100 → cosmetic boards, founder rosters, "100 spots"
  campaigns.
- 100 – 1024 → most grids. Atomic launch, single tx.
- 1024 – 4096 → larger maps (game tile sets, NFT mosaics).
  Chunked launch, multiple txs.
- 4096+ → multi-grid sharding (see
  [Sharding](sharding.md)). The SDK stitches them into one canvas
  off-chain.

### `forfeitureDuration`

The Dutch decay window after a tax-underwater forfeit. `0`
inherits the 10-minute default.

- 5 minutes – 1 hour: aggressive — a forfeited seat clears
  quickly, keeping the grid liquid.
- 1 hour – 1 day: moderate — gives the prior holder a chance to
  re-acquire at a discount.
- > 1 day: only for special cases. Cap is 30 days.

### `yieldToken`

Pick the token your yield adapter will push:

- `address(0)` (ETH) — if your adapter calls `receiveYieldETH`.
  `flETH` / `WETH` are accepted variants and unwrapped on receive.
- A specific ERC20 — if your adapter calls
  `receiveYieldERC20(token, …)`. The token MUST exactly match the
  grid's `yieldToken`.

### `initialPrices[]`

The starting self-assessed price for each seat in `taxToken`
units. Common patterns:

- **Flat** — every seat starts at the same price. Simplest.
- **Tiered** — center / corner / edge seats priced differently.
  Encodes "premium real estate" without the contract knowing
  about it.
- **Auction-bias** — start every seat at zero. The first acquirer
  pays nothing; Harberger taxation kicks in once they self-assess.

## Post-create checklist

- [ ] `GridCreated(gridId, creator, totalSeats, ...)` event present
      in the receipt.
- [ ] If chunked: `GridInitialPricingComplete(gridId)` emitted
      after the final chunk.
- [ ] `gridConfigV22(gridId)` returns the expected `taxToken`,
      `yieldToken`, `totalSeats`, `forfeitureDuration`,
      `taxRateBps`.
- [ ] If you intend to attach a hook module: guardian has
      `setApprovedGovernanceModule(module, true)`-d it AND the
      grid creator has called
      `setGridGovernanceModule(gridId, module)`. (Optional second
      stage: `setApprovedModule(module, true)` if the module
      will call `moduleTransferSeat`.)
- [ ] If you intend to attach a yield adapter: guardian has
      `setApprovedAdapter(adapter, true)`-d it. Then route
      `deposit(gridId, ...)` through the adapter.

## See also

- [Sharding boards above 1024 seats](sharding.md) — chunked +
  multi-grid recipes.
- [Wiring an adapter to a grid](wiring-an-adapter.md) — adapter
  attach flow.
- [Multi-asset grids](multi-asset-grids.md) — grids with multiple
  registered assets per yield-token bucket.
- [The hook model](../overview/hook-model.md) — when to attach a
  hook module to a fresh grid.
- [UGM v2.3 API](../reference/ugm-api.md) — full ABI for the
  creation surface.
