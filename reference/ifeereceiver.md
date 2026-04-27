# `IFeeReceiver`

> **Tier 2 placeholder.** Reference for the callback UGM uses to notify a
> non-EOA tax/sale recipient.

## What it is

When `setGridFeeReceivers` points at a contract instead of an EOA, UGM
calls `notifyDeposit(gridId, amount)` on it after every `withdrawTaxRevenue`
and seat-sale fee transfer.

```solidity
interface IFeeReceiver {
    function notifyDeposit(uint256 gridId, uint256 amount) external;
}
```

> Source: [`takeoverapp/takeover-contracts/src/interfaces/IFeeReceiver.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/interfaces/IFeeReceiver.sol).

## When to implement it

- You want a `BoardroomFarm`-style contract that receives tax revenue and
  re-allocates it (see `PLAN_BOARDROOM_FARM.md` in `takeover-contracts`).
- You're a treasury contract that wants to record per-grid attribution.
- You're routing tax to another protocol's accounting system.

This page will be expanded with the v2.1 swallow-revert behaviour
(`_notifyFeeReceiver` swallows reverts but emits `NotifyDepositFailed`),
the `flushPool` recovery pattern, and worked examples.
