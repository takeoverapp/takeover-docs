# The `IGridHooksV23` interface

`IGridHooksV23` is the surface UGM v2.3 grids call into when a creator
attaches a module. It is a strict superset of v2.2's `IGridGovernanceHooks`:

```solidity
interface IGridHooksV23 is IGridGovernanceHooks {
    function beforeClaim(uint256 gridId, uint256 seatId, address claimant) external;

    function beforeBuyout(
        uint256 gridId,
        uint256 seatId,
        address attacker,
        address defender,
        uint256 proposedPrice
    ) external;

    function beforePriceChange(
        uint256 gridId,
        uint256 seatId,
        uint256 oldPrice,
        uint256 newPrice
    ) external;

    function onForfeit(uint256 gridId, uint256 seatId, address lastHolder) external;

    function yieldWeight(uint256 gridId, uint256 seatId) external view returns (uint256);
}

// inherited from v2.2
interface IGridGovernanceHooks {
    function onSeatHolderChange(
        uint256 gridId,
        uint256 seatId,
        address oldHolder,
        address newHolder
    ) external;
}
```

A module that only implements the v2.2 surface (`onSeatHolderChange`) keeps
working on a v2.3 grid. UGM calls every additive callback inside a
`try { … } catch` and treats "function-selector-not-found" as
"didn't implement; treat as no-op". `WhitelistGovernanceModule` from v2.2
deploys onto v2.3 grids without a redeploy.

## Why the rename

UGM v2.2 called this `IGridGovernanceHooks` because the only known consumers
were DAO-style tooling. v2.3 broadens the surface for games, prediction
markets, custom registries, and other apps that want to inject policy at
state-changing seat events. The interface keeps `onSeatHolderChange` because
existing modules and indexers depend on its signature.

## Callback semantics

| Callback | Class | Fires on | Revert behavior |
|---|---|---|---|
| `onSeatHolderChange` | observer | every materialized seat-holder change | propagates |
| `beforeClaim` | gate | first-time acquisition of a vacant / lazy seat | propagates → revert blocks the buy |
| `beforeBuyout` | gate | a held seat is bought out | propagates → revert blocks the buyout |
| `beforePriceChange` | gate | `setPrice` | propagates → revert blocks the price change |
| `onForfeit` | observer | tax-underwater forfeit + Dutch auction entry | swallowed → emits `GovernanceHookSoftFail` |
| `yieldWeight` | view | claimFees per-seat allowance computation | swallowed → falls back to `1/totalSeats` |

Gating callbacks (`beforeClaim`, `beforeBuyout`, `beforePriceChange`) propagate
their revert reason. Observer callbacks (`onForfeit`, `yieldWeight`) absorb
the failure and emit `GovernanceHookSoftFail(gridId, module, selector)` so a
buggy module can't brick the forfeit path or freeze yield distribution.

`onSeatHolderChange` is the v2.2 holdover and **does** propagate reverts —
this is intentional, it's how `WhitelistGovernanceModule` blocks
non-whitelisted holders. New modules built specifically for v2.3 should
prefer `beforeClaim` / `beforeBuyout` / `beforePriceChange` over
`onSeatHolderChange`, which fires after the holder change is committed.

## Gas budgets

All state-changing callbacks (every method except `yieldWeight`) run inside
a UGM-supplied `GOVERNANCE_HOOK_GAS_CAP = 150_000` gas budget. `yieldWeight`
runs the same cap via `staticcall`.

If a gating callback runs out of gas with empty return data, UGM treats it
as "v2.2-only module didn't implement this hook" and silently no-ops
(see [hooks/lifecycle.md](lifecycle.md) for the exact dispatch semantics).
If it exceeds the cap with non-empty revert data, UGM bubbles the data so
modules can return readable revert reasons.

For modules with substantial onchain logic (graph traversal, weighted
computation, etc.) the practical implication is: **no SSTORE-heavy work in
hooks**. Cache reads, do the math offchain or in a separate transaction, and
keep the hook to a yes/no policy decision.

## `yieldWeight`: declared, not yet wired

`yieldWeight(gridId, seatId)` is declared in the interface but is **not
called** by the current UGM v2.3 deployment. It's reserved for a future
pro-rata yield distribution path. Today every materialised seat receives
`totalYield / totalSeats`. Modules can implement `yieldWeight` and ship it
defensively so they don't need a redeploy when UGM begins consuming it,
but it has no runtime effect right now.

## What hooks cannot do

Hooks are policy gates, not engines. UGM v2.3 deliberately does not give
hooks control over:

- **Forfeiture timing.** `onForfeit` is observer-only — by the time
  `_applyTax` decides to forfeit, the deposit is already exhausted and a
  revert would leave the seat in an invalid state.
- **Dutch auction pricing.** Modules wanting to absorb forfeited inventory
  should call `addBatch` with the current Dutch clearing price (often 0)
  rather than gate forfeit.
- **Phantom buyouts.** `moduleTransferSeat` supports `Deposit` refund or
  no refund (`None`); paying the prior holder their self-assessed price
  should always route through `addBatch` so the standard fee splits, hooks,
  and indexer events fire correctly.

For the forced-transfer entrypoint, see
[`moduleTransferSeat`](module-transfer-seat.md).

## Where to next

- [Hook lifecycle and gas budgets](lifecycle.md) — exact dispatch sites.
- [Registration and the two-stage allowlist](registration.md) — how a hook
  module gets approved and attached.
- [Two-axis pause + per-module disable](pause-flags.md) — the three
  orthogonal flags governing when hooks fire.
- [`moduleTransferSeat`](module-transfer-seat.md) — forced-transfer path.
- [Whitelist module](examples/whitelist-module.md) — annotated reference
  module, ports cleanly to v2.3.
- [Anti-snipe module](examples/anti-snipe.md) — net-new gating example.
