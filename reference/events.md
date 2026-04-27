# Events

> **Tier 2 placeholder.** Catalogue of UGM and adapter events the Takeover
> indexer relies on.

## UGM v2.1 events (canonical)

These are stable across all UGM v2.1 deployments. The indexer pins to them.

| Event | Emitted by |
|---|---|
| `GridCreated` | `createGrid` |
| `GridPricesInitialized` | `createGrid` |
| `SeatBought` | seat acquisition / buyout |
| `PriceUpdated` | `setPrice` |
| `DepositAdded` / `DepositWithdrawn` | `addDeposit` / `withdrawDeposit` |
| `SeatAbandoned` | `abandonSeat` |
| `SeatForfeited` | tax-driven forfeiture |
| `FeesClaimed` | `claimFees` |
| `TaxCollected` | `pokeTax` and per-seat applications |
| `TaxRevenueWithdrawn` | `withdrawTaxRevenue` |
| `SeatTransferred` | seat ownership change |
| `AssetRegistered` / `AssetWithdrawn` | adapter registers/withdraws an asset |
| `YieldReceived` | adapter pushes yield via `receiveYield*` |
| `ApprovedAdapterUpdated` | guardian flips adapter approval |

## Adapter-side events

Reference adapters emit at minimum:

| Event | Emitted by |
|---|---|
| `AssetDeposited(assetHash, ..., gridId)` | adapter `deposit` |
| `AssetWithdrawn(assetHash, recipient)` | adapter `withdraw` |
| `YieldClaimed(assetHash, amount)` | inside `collectYield`, only on non-zero forward |

The indexer **does not** require these — UGM's events are the source of
truth — but they make per-adapter dashboards much easier.

This page will be expanded with full ABI signatures and indexer schema
mappings.
