# `IGrid`

> **Tier 2 placeholder.** Reference for the protocol-agnostic Harberger vault
> standard implemented by UGM v2.1.

## What it is

`IGrid` is the minimal interface a Harberger seat ledger has to expose.
UGM v2.1 implements it. Any future Takeover-compatible vault should too.

> Source: [`takeoverapp/takeover-contracts/src/interfaces/IGrid.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/interfaces/IGrid.sol).

## What's in it

- **Lifecycle:** `gridConfig`, `seats`, `holderSeats`, `holderSeatCount`.
- **Seat ops:** `addBatch`, `setPrice`, `addDeposit`, `withdrawDeposit`,
  `batchManageSeats`, `abandonSeat`.
- **Yield/tax:** `claimFees`, `pokeTax`, `withdrawTaxRevenue`.
- **Events** the indexer pins to: `GridCreated`, `SeatBought`,
  `PriceUpdated`, `DepositAdded`, `DepositWithdrawn`, `SeatAbandoned`,
  `SeatForfeited`, `FeesClaimed`, `TaxCollected`, `TaxRevenueWithdrawn`,
  `SeatTransferred`, `GridPricesInitialized`.

This page will be expanded with full signatures and behavioural notes.
