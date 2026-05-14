# Events

Catalogue of UGM v2.3 and adapter events the Takeover indexer relies on.
Every event below is stable across the v2.3 deployments on Base mainnet
and Base Sepolia.

## UGM v2.3 — canonical seat lifecycle

These signatures are inherited from `IGrid` and emitted by
`UnifiedGridManagerV23` (and its `UGMV23Linked` library).

| Event | Emitted by |
|---|---|
| `GridCreated(gridId, creator)` | `createGrid`, `createGridShell` |
| `GridPricesInitialized(gridId, prices)` | atomic `createGrid` (single-shot pricing) |
| `GridInitialPricesChunk(gridId, startSeat, prices)` | `appendInitialPricingChunk` (per chunk) |
| `GridInitialPricingComplete(gridId)` | `appendInitialPricingChunk` once `written == totalSeats` |
| `SeatBought(gridId, seatId, oldHolder, newHolder, price)` | first claim, buyout, and Dutch reclaim |
| `SeatReclaimedDutch(gridId, seatId, ...)` | Dutch-auction clearing (paired with `SeatBought`) |
| `PriceUpdated(gridId, seatId, oldPrice, newPrice)` | `setPrice` |
| `DepositAdded(gridId, seatId, amount)` / `DepositWithdrawn(...)` | `addDeposit` / `withdrawDeposit` |
| `SeatAbandoned(gridId, seatId, holder, refund)` | `abandonSeat`, `abandonBatch`, and tax-driven forfeit |
| `SeatForfeited(gridId, seatId, holder)` | tax-driven forfeit only (precedes `SeatAbandoned`) |
| `FeesClaimed(gridId, holder, amount)` | `claimFees` per holder |
| `TaxCollected(gridId, seatId, amount)` | `pokeTax` and per-seat applications |
| `TaxRevenueWithdrawn(gridId, receiver, amount)` | `withdrawTaxRevenue` |
| `SeatTransferred(gridId, seatId, from, to, newPrice)` | `transferSeat` (creator-driven) |
| `Graduated(gridId)` | last seat sold (graduation marker for indexer) |

## UGM v2.3 — adapter and yield

| Event | Emitted by |
|---|---|
| `AssetRegistered(gridId, assetHash)` | adapter calls `registerAsset` |
| `AssetWithdrawn(gridId, assetHash)` | adapter calls `withdrawAsset` |
| `YieldReceived(gridId, assetHash, amount)` | adapter pushes yield via `receiveYieldETH` / `receiveYieldERC20` |
| `ApprovedAdapterUpdated(adapter, approved)` | guardian flips adapter approval |
| `CreatorInventoryYieldClaimed(gridId, creator, amount)` | creator-side claim of yield on still-unsold inventory |
| `YieldBalanceCapped(gridId, holder, owed, paid)` | yield exceeded UGM balance (cap event) |
| `PayoutEscrowed(token, user, gridId, amount)` | failed transfer rerouted to pull-based escrow |
| `PayoutClaimed(token, user, to, amount)` | escrow drained via `claimPayout` |

## UGM v2.3 — modules and pause flags (new in v2.3)

These are the seven additional events v2.3 adds for hook modules and the
two-axis pause:

| Event | Emitted by | Indexer use |
|---|---|---|
| `GridGovernanceModuleSet(gridId, module)` | grid creator's `setGridGovernanceModule` | track which module is currently attached to a grid |
| `ApprovedModuleUpdated(module, approved)` | guardian's `setApprovedModule` | track the v2.3 `approvedModules` list (transfer-authorised subset) |
| `GridModulesPausedEvent(gridId, paused)` | guardian's `pauseGridModules` / `unpauseGridModules` | per-grid module pause flag |
| `ModuleDisabledEvent(module, disabled)` | guardian's `setModuleDisabled` | per-module kill switch (soft pause) |
| `SeatTransferredByModule(gridId, seatId, module, oldHolder, newHolder, refundMode, depositRefund)` | `moduleTransferSeat` | forced-transfer path (distinct from `SeatBought`) |
| `GovernanceHookSoftFail(gridId, module, selector)` | `_callOnForfeit` and other observer-only callbacks | detect modules silently misbehaving |
| `GridDeprecated(gridId)` | `creatorCloseGrid` (deprecation marker) | mark a grid as fully closed |

`SeatTransferredByModule` is intentionally **not** the same event as
`SeatBought`. Forced transfers are economically distinct from buyouts —
indexers should display them separately so users can tell whether a seat
moved via an attack/buyout or via a module's policy decision.

## UGM v2.3 — guardian admin

| Event | Emitted by |
|---|---|
| `GuardianUpdated(oldGuardian, newGuardian)` | `transferGuardian` |
| `GridPausedEvent(gridId, paused)` | `pauseGrid` / `unpauseGrid` |
| `GridCreationPausedEvent(paused)` | `pauseGridCreation` / `unpauseGridCreation` |
| `AllowedTaxTokenUpdated(token, allowed)` | `setAllowedTaxToken` |
| `ProtocolSeatSaleBpsUpdated(oldBps, newBps)` / `CreatorSeatSaleBpsUpdated(...)` / `ProtocolTaxBpsUpdated(...)` / `GlobalTaxRateBpsUpdated(...)` | guardian-controlled BPS setters |
| `ApprovedZapUpdated(zap, approved)` | `setApprovedZap` |
| `GridFeeReceiversUpdated(gridId, taxReceiver, saleReceiver)` | `setGridFeeReceivers` |
| `ProtocolFeeReceiverUpdated(oldReceiver, newReceiver)` | `setProtocolFeeReceiver` |
| `ProtocolTaxFeesWithdrawn(receiver, gridId, amount)` / `ProtocolSeatSaleFeesWithdrawn(...)` | guardian fee sweeps |

## Adapter-side events (reference adapters)

Reference adapters (`FlaunchYieldAdapter`, `V3YieldAdapter`,
`V4YieldAdapter`, `ProtocolYieldAdapter`) emit at minimum:

| Event | Emitted by |
|---|---|
| `AssetDeposited(assetHash, ..., gridId)` | adapter `deposit` |
| `AssetWithdrawn(assetHash, recipient)` | adapter `withdraw` |
| `YieldClaimed(assetHash, amount)` | inside `collectYield`, only on non-zero forward |

The Takeover indexer **does not** require adapter-side events — UGM's
`AssetRegistered` / `AssetWithdrawn` / `YieldReceived` triple is the
source of truth — but per-adapter events make protocol dashboards much
easier.

## Hook-module events (reference modules)

`WhitelistGovernanceModuleV2` and other reference hook modules emit:

| Event | Emitted by |
|---|---|
| `ReservationsSet(gridId, count)` | whitelist `setReservations` |
| `ClaimDeadlineSet(gridId, deadline)` | whitelist `setClaimDeadline` |

These are module-internal admin events, not part of the canonical UGM
indexer schema. New hook modules should emit similar admin events for
their own state changes; UGM's `GridGovernanceModuleSet` is what the
indexer uses to associate a module with a grid.

## What the chunked creation flow looks like

For grids with `1024 < totalSeats <= 4096`, creation emits:

```
GridCreated(gridId, creator)                      // from createGridShell
GridInitialPricesChunk(gridId, 0, prices[0..N])   // from chunk #0
GridInitialPricesChunk(gridId, N, prices[N..2N])  // from chunk #1
…
GridInitialPricingComplete(gridId)                // from the final chunk
```

Indexers should treat the grid as "incomplete" between `GridCreated` and
`GridInitialPricingComplete`. UGM enforces this on-chain by reverting
seat ops with `InitialPricingIncomplete()` until pricing finishes.

## What to read next

- [reference/ugm-api.md](ugm-api.md) — full UGM v2.3 method surface.
- [hooks/lifecycle.md](../hooks/lifecycle.md) — when each hook event
  fires.
- [boards/sharding.md](../boards/sharding.md) — chunked creation walk-
  through with a viem example.
