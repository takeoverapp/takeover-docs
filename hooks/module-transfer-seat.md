# `moduleTransferSeat`: forced reassignment

`moduleTransferSeat` is the v2.3-exclusive entrypoint that lets an
authorised hook module forcibly relocate a seat from one holder to another
without payment. It exists for narrative flows that don't fit the
buyout path — military conquest in a game, DAO seat rotations,
prediction-market resolution, scripted faction shifts — and intentionally
omits the "phantom buyout" mode that would pay the prior holder their
self-assessed price.

If you want a buyout, use `addBatch`. If you want a forced move, use this.

## Signature

```solidity
function moduleTransferSeat(
    uint256 gridId,
    uint256 seatId,
    address newHolder,
    SeatTransferRefund refundMode
) external returns (uint256 depositRefund);

enum SeatTransferRefund { None, Deposit }
```

- `gridId` / `seatId` — the seat to move. Must exist; grid not paused
  (`gridPaused`); module flag not set (`gridModulesPaused`).
- `newHolder` — destination address. Cannot be `address(0)` and cannot
  equal the current holder. Both revert as
  `InvalidModuleTransferTarget`.
- `refundMode` — economic policy (see below).
- Returns `depositRefund` — the actual amount returned to the prior
  holder under `Deposit` mode (0 otherwise, or if the seat carried no
  deposit).

The function emits `SeatTransferredByModule(gridId, seatId, module,
oldHolder, newHolder, refundMode, depositRefund)`.

## Authorization gate (four checks, one error)

UGM v2.3 collapses four orthogonal checks into a single `UnauthorizedModule`
revert so a misconfigured caller can't probe state through error
messages:

```solidity
address attachedModule = gridGovernanceModule[gridId];
if (
    attachedModule == address(0)
    || msg.sender != attachedModule
    || !approvedModules[attachedModule]
    || moduleDisabled[attachedModule]
) {
    revert UnauthorizedModule();
}
```

To pass, the caller must:

1. **Be the grid's attached governance module.** A creator must have
   called `setGridGovernanceModule(gridId, msg.sender)`.
2. **Be on the `approvedModules` allowlist.** Guardian sets this with
   `setApprovedModule(module, true)`. This is a strict subset of
   `approvedGovernanceModules` — see
   [registration.md](registration.md).
3. **Not be currently disabled.** Guardian's
   `setModuleDisabled(module, true)` toggles a per-module kill switch
   independent of allowlist removal.

To diagnose which check failed, read the public mappings directly:

```solidity
ugm.gridGovernanceModule(gridId);
ugm.approvedModules(myModule);
ugm.moduleDisabled(myModule);
```

`approvedGovernanceModules` is `internal`; query its state via
`setGridGovernanceModule` revert behavior, or via the audit/operator
channel rather than RPC.

## Pause gates

Two pause flags also gate `moduleTransferSeat` (in addition to the
authorization checks above):

- `gridPaused[gridId]` → reverts via the `whenNotPaused` modifier with
  `GridPaused`.
- `gridModulesPaused[gridId]` → reverts with `GridModulesPaused`.

Full truth table is in [pause-flags.md](pause-flags.md).

## Refund modes

The module — not the displaced holder — picks the refund policy. UGM
intentionally exposes only two modes; the third ("phantom buyout") is
absent on purpose.

### `SeatTransferRefund.None` — looted

The prior holder loses both their unspent tax deposit and the
self-assessed price they had signalled they would sell at. The deposit
stays inside UGM (it has already been pulled and is part of the
contract's tax-token balance) and rolls into the standard tax/yield
accounting on the next sale.

Use this for narrative defeats where economic loss is part of the
story: military conquest, faction loss, DAO ousting.

### `SeatTransferRefund.Deposit` — refund unspent deposit

UGM routes the prior holder's unspent tax deposit back to them via the
same payout escrow path used by buyouts. The module provides nothing —
this is just routing the existing deposit.

Use this when the policy is "you lose the seat, but not your stake."
Common for prediction-market resolution and DAO seat rotations.

### Why no `Price` mode

The hypothetical third mode would pay the prior holder their
self-assessed price. UGM omits it because:

- Any flow paying the prior holder their listed price *is* an economic
  takeover.
- Routing through `addBatch` runs the standard split (10% buyout fees +
  forfeiture penalty if applicable), fires `beforeBuyout` and
  `onSeatHolderChange`, and emits the indexer-watched
  `SeatBoughtOut(...)` event.
- A separate `Price` refund path would mean two execution surfaces with
  near-identical economics, which is a recipe for indexer drift and
  user confusion.

If you need to pay the prior holder, your module should call `addBatch`
on the grid (with the buy price equal to the prior holder's
self-assessed price) rather than `moduleTransferSeat`.

## What `moduleTransferSeat` actually does

Internally, `moduleTransferSeat` is approximately a `_buyout` with the
fee splits replaced by the refund policy and `beforeBuyout` skipped:

1. Authorization gate (above).
2. `whenNotPaused` (`gridPaused`) and `gridModulesPaused` checks.
3. Lazy-seat materialization: if the seat was still creator-held at
   the packed initial price, materialize the slot, mark `seatEverSold`,
   bump `seatsSold`. (`_applyTax` is **skipped** on the lazy path
   because `lastTaxTime` was just set to 0.)
4. Held-seat path: `_collectAdapterYield`, `_applyTax`, then
   `_claimForSeat` to pay any unclaimed yield to the prior holder.
   If `_applyTax` forfeits the seat, abort with `SeatNotClaimed`.
5. Snapshot `seat.deposit` for refund.
6. Reassign holder to `newHolder`, clear deposit, emit
   `SeatTransferredByModule`, fire `onSeatHolderChange`.

`beforeBuyout` is **not** fired (this isn't a buyout). The module
calling `moduleTransferSeat` is the policy authority for this transfer
— there's no point gating it via callback.

## Reverts

| Error | Cause |
|---|---|
| `UnauthorizedModule()` | One of the four authorization checks failed. |
| `InvalidModuleTransferTarget()` | `newHolder == address(0)` or `newHolder == oldHolder`. |
| `GridPaused()` | `gridPaused[gridId] == true`. |
| `GridModulesPaused()` | `gridModulesPaused[gridId] == true`. |
| `GridDeprecated()` | Standard creator-close lifecycle gate. |
| `SeatDoesNotExist()` | Lazy-path: `seatId >= grid.totalSeats`. |
| `SeatNotClaimed()` | The seat is in Dutch-auction state (forfeited and not yet reclaimed) — modules cannot bypass the Dutch decay window. |

For the Dutch-auction case, modules wanting to absorb forfeited
inventory should call `addBatch` with the current Dutch clearing price
(often 0) and route the seat to the new holder via the buyout path.

## Worked example: claiming a seat as a forced transfer

A game module tracks military conquest. When attacker A captures the
seat at `(gridId, seatId)` from defender D, the module calls:

```solidity
function resolveConquest(uint256 gridId, uint256 seatId, address attacker) external {
    require(_resolutionAuthorisation(msg.sender, gridId, seatId, attacker), "no permission");

    // Looted: defender loses deposit; attacker gets the seat without paying.
    uint256 refunded = ugm.moduleTransferSeat(
        gridId,
        seatId,
        attacker,
        IUgm.SeatTransferRefund.None
    );
    require(refunded == 0, "expected no refund in None mode");

    emit ConquestResolved(gridId, seatId, attacker, msg.sender);
}
```

Counter-pattern, "honourable defeat": defender gets their unspent
deposit back even though they lose the seat:

```solidity
function resolveTreaty(uint256 gridId, uint256 seatId, address newHolder) external onlyDAO {
    uint256 refunded = ugm.moduleTransferSeat(
        gridId,
        seatId,
        newHolder,
        IUgm.SeatTransferRefund.Deposit
    );
    emit TreatyResolved(gridId, seatId, newHolder, refunded);
}
```

Counter-counter-pattern, paying the displaced holder for their seat:
**don't use `moduleTransferSeat`**. Use `addBatch`:

```solidity
IUgm.AddBatchParams memory params = _addBatchForBuyout(gridId, seatId, newHolder, currentSelfPrice);
ugm.addBatch(params);   // standard buyout path: fee splits, hooks, events
```

## Indexing forced transfers

`SeatTransferredByModule(gridId, seatId, module, oldHolder, newHolder,
refundMode, depositRefund)` is emitted on every successful call.
Indexers should treat it as a separate event from `SeatClaimed` /
`SeatBoughtOut` so the user-facing UI can surface "the conquest module
moved this seat" rather than a regular trade.

The accompanying `onSeatHolderChange` callback fires after the new
holder is written, with `(oldHolder, newHolder) == (defender,
attacker)`. Modules that also implement `onSeatHolderChange` can use it
as the on-chain hook for their own bookkeeping.

## What to read next

- [Pause flags](pause-flags.md) — exact halt conditions for forced
  transfers.
- [Registration](registration.md) — how to get on `approvedModules`.
- [Hook lifecycle](lifecycle.md) — the rest of the callback surface.
- [Examples: anti-snipe](examples/anti-snipe.md) — a module that gates
  `beforeBuyout` rather than reaching for `moduleTransferSeat`.
