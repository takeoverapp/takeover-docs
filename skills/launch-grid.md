---
name: takeover-launch-grid
description: >-
  Launch a Takeover grid on UGM v2.3. Covers the three creation
  paths — atomic `createGrid` (≤1024 seats), chunked
  `createGridShell` + `appendInitialPricingChunk` (1025–4096 seats),
  and SDK-side multi-grid sharding (>4096 seats). Use when the user
  wants to spin up a board, deploy a map, mint a "set of tiles", or
  asks how many seats fit in one tx.
---

# Skill: launch a grid

Stand up a fresh grid on UGM v2.3. Each grid is a fixed-size set
of seats under Harberger taxation. The right creation path depends
purely on `totalSeats`.

## Phase 0 — read before you write

1. [What is Takeover](../overview/what-is-takeover.md) — vocabulary.
2. [UGM v2.3](../overview/ugm-v2.3.md) — what changed vs prior versions.
3. [Creating a grid](../boards/creating-a-grid.md) — full struct + validation tables.
4. [Sharding boards above 1024 seats](../boards/sharding.md) — chunked + multi-grid recipes.
5. [Deployments](../reference/deployments.md) — UGM addresses + library addresses per chain.

## Pick a path by `totalSeats`

| `totalSeats` | Path | Number of txs |
| --- | --- | --- |
| 4 – 1024 | Atomic `createGrid` | 1 |
| 1025 – 4096 | `createGridShell` then N × `appendInitialPricingChunk` | 1 + ⌈seats / chunkSize⌉ |
| 4097 + | SDK-side multi-grid sharding (`GridRef[]`) | N × atomic-or-chunked, one per shard |

Reverts you should know:

- `TotalSeatsRequiresChunkedCreation` → user passed >1024 to
  `createGrid`. Switch to `createGridShell`.
- `TotalSeatsOutOfBounds` → outside `[4, 4096]`. SDK-side shard.
- `InitialPricesLengthMismatch` → `initialPrices.length !=
  totalSeats` on the atomic path.
- `InitialPricingIncomplete()` → user tried to acquire a seat
  before all pricing chunks landed.

## Validation table (every path)

| Field | Rule | Source |
| --- | --- | --- |
| `taxRateBps` | `0` (inherit `DEFAULT_TAX_RATE_BPS = 500`) or in `[100, 1000]` (1%–10%) | `MIN_TAX_RATE` / `MAX_TAX_RATE` |
| `totalSeats` | `[4, 4096]` | `MIN_TOTAL_SEATS` / `MAX_TOTAL_SEATS` |
| `forfeitureDuration` | `0` (inherit 10-min default) or `≤ 30 days` | `MAX_FORFEITURE_DURATION` |
| `taxToken` | On guardian's `setAllowedTaxToken` allowlist (USDC by default) | `allowedTaxTokens` |
| `yieldToken` | `address(0)` for native ETH or any ERC20 the creator's adapter pushes | adapter contract |
| `initialPrices[i]` | `uint128`; below `floorPrice` for that seat reverts (per-seat creator-set listing) | `_packedInitialPrices` |

## Phase 1 — encode params

```ts
import { encodeFunctionData, parseUnits } from 'viem';
import { base } from '@takeover/sdk/networks';

const params = {
  taxRateBps: 500n,                          // 5%
  totalSeats: 256n,                          // atomic-tier
  forfeitureDuration: 0,                     // inherit default
  taxToken: base.usdcAddress,                // USDC
  yieldToken: '0x0000000000000000000000000000000000000000', // ETH
  initialPrices: Array.from({ length: 256 }, () =>
    parseUnits('5', 6) as unknown as bigint, // 5 USDC each
  ),
};

const data = encodeFunctionData({
  abi: ugmAbi,        // see `reference/ugm-api.md`
  functionName: 'createGrid',
  args: [params],
});
```

For >1024 seats, encode `createGridShell` with the same fields
minus `initialPrices`, then loop `appendInitialPricingChunk`. See
[boards/sharding.md](../boards/sharding.md) for the loop body.

## Phase 2 — simulate, then write

Use `publicClient.simulateContract({...})` first to surface revert
reasons cleanly; only then `walletClient.writeContract`.

`createGrid` returns `gridId`; the same value is emitted on the
`GridCreated` event. Decode it from the receipt's logs to keep
your local state in sync.

## Phase 3 — post-create checklist

Before declaring success:

- [ ] `GridCreated(gridId, creator, totalSeats, ...)` event present.
- [ ] All pricing chunks landed (chunked path) — listen for
      `GridInitialPricingComplete(gridId)`.
- [ ] `gridConfigV22(gridId)` returns the expected `taxToken`,
      `yieldToken`, `totalSeats`, `forfeitureDuration`, `taxRateBps`.
- [ ] If you intend to attach a hook module, the module has been
      `setApprovedGovernanceModule`-d by the guardian AND
      `setGridGovernanceModule(gridId, module)` has been called by
      the grid creator.
- [ ] If you intend to attach a yield adapter, the adapter has
      been `setApprovedAdapter`-d by the guardian. Then route
      `deposit(gridId, ...)` through the adapter.

## Phase 4 — verify against the launch checklist

The machine-readable checklist at
[`submit/checklist.yml`](../submit/checklist.yml) carries
`grid_launch.*` items. Iterate them and produce a structured
report (same format as the adapter / hook skills).

## What you should not do

- **Do not pre-mint NFTs for seats.** Seats are not ERC721. They
  are tracked in UGM's seat mappings. You can wrap a seat in an
  ERC721 via a custom hook module if your app needs it, but the
  on-chain seat itself is not a token.
- **Do not use `createGrid` above 1024 seats.** It reverts.
  Switch to `createGridShell` / `appendInitialPricingChunk`.
- **Do not silently fall back to `MAX_TOTAL_SEATS` shaping.** If
  the user wants 6,000 seats, that is multi-grid sharding (see
  [boards/sharding.md](../boards/sharding.md)) — not a clipped
  4,096-seat grid.
- **Do not mix tax tokens in one shard.** Multi-grid sharding
  must either replicate the same `taxToken` per shard (most
  builders) or document the per-shard variance explicitly.
