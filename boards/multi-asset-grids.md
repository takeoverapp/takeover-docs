# Multi-asset grids

> **Tier 2 placeholder.** Caveats for grids that aggregate yield from more
> than one source.

## Today's behaviour

UGM v2.1 stores yield as a single `totalYield[gridId]` counter. You can
register **multiple assets** to one grid — same adapter or different
adapters — and they'll all push into the same counter. From a seat
holder's perspective, it's still 1/N of `totalYield[gridId]` per seat.

A few things to know before you do this:

- All registered assets must produce the grid's **single** `yieldToken`.
  You can't mix flETH and USDC into one grid.
- `pendingYield` is **per asset**, not per grid. UI summing is the
  caller's job.
- `claimFees` triggers `collectYield` for **all** adapters that have
  registered assets on that grid. Bad collect logic on one adapter blocks
  the whole grid for that claimer (this is why every adapter must be
  best-effort and not revert on stale assets).

This page will be expanded with the recommended patterns for:

- Two LP positions feeding one grid.
- Mixing a primary yield source with a "top-up" `ProtocolYieldAdapter`.
- Migrating an asset from grid A to grid B (withdraw + re-register).
