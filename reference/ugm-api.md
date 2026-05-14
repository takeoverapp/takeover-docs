# UGM v2.3 API reference

Reference for `UnifiedGridManagerV23` — the functions an adapter, hook
module, integrator, or builder UI needs to call. Grouped by surface.

> Source: [`takeoverapp/takeover-contracts/src/UnifiedGridManagerV23.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/UnifiedGridManagerV23.sol)
> and the `UGMV23Linked` library it `DELEGATECALL`s into.

## Version detection

```solidity
bytes32 public constant ugmVersion = bytes32("v2.3");
```

Clients should read `ugmVersion()` instead of bytecode probing to pick
calldata builders.

## Grid creation

| Function | When to use | Notes |
|---|---|---|
| `createGrid(CreateGridParams params) returns (uint gridId)` | `totalSeats <= 1024` | Atomic. Initial pricing supplied in one shot. |
| `createGridShell(CreateGridShellParams params) returns (uint gridId)` | `totalSeats > 1024` | Reserves the gridId without pricing. Trades stay locked until pricing finishes. |
| `appendInitialPricingChunk(uint gridId, uint128[] prices)` | between `createGridShell` and trade unlock | Walks initial prices in chunks. Caller must equal grid creator. |

Validation bounds (enforced inside `UGMV23Linked.createGrid` /
`createGridShell`):

| Bound | Value |
|---|---|
| `MIN_TOTAL_SEATS` | 4 |
| `MAX_TOTAL_SEATS` | 4096 |
| `MAX_SINGLE_TX_CREATE_TOTAL_SEATS` | 1024 |
| `taxRateBps` | `[100, 1000]` (1%–10%) |
| `forfeitureDuration` | `[1 hours, 30 days]` |

See [boards/creating-a-grid.md](../boards/creating-a-grid.md) and
[boards/sharding.md](../boards/sharding.md) for the complete walkthrough
of each path.

Reverts with `TotalSeatsRequiresChunkedCreation()` if you call
`createGrid` with `totalSeats > 1024`.

## Seat operations

| Function | Caller | Effect |
|---|---|---|
| `addBatch(AddBatchParams params)` | anyone (with payment) | Atomic claim / buyout / abandon batch over multiple seats. The omnibus seat-write entrypoint. |
| `setPrice(uint gridId, uint seatId, uint newPrice)` | seat holder | Update self-assessed price. Fires `beforePriceChange` hook (if attached). |
| `addDeposit(uint gridId, uint seatId, uint amount)` | anyone | Top up tax deposit for a seat. |
| `withdrawDeposit(uint gridId, uint seatId, uint amount)` | seat holder | Remove unspent deposit (subject to safety floor). |
| `abandonSeat(uint gridId, uint seatId)` | seat holder | Voluntarily give up the seat. Refund delivered via standard payout escrow. |
| `abandonBatch(uint gridId, uint[] seatIds)` | seat holder | Abandon many seats in one tx. |
| `transferSeat(uint gridId, uint seatId, address to, uint newPrice)` | grid creator | Creator-driven move (e.g. zaps re-routing creator inventory). |

`addBatch` is the load-bearing entrypoint. It accepts arrays of
acquisition / buyout / abandon ops in a single struct and applies them
atomically — most builder integrations only need to know about
`addBatch`, `setPrice`, and the abandon family.

## Modules and hooks

| Function | Caller | Effect |
|---|---|---|
| `setGridGovernanceModule(uint gridId, address module)` | grid creator | Attach (or detach with `address(0)`) a hook module. Module must already be on `approvedGovernanceModules`. |
| `setApprovedGovernanceModule(address module, bool approved)` | guardian | Allowlist for hook attachment. Required for any `IGridHooksV23` callback. |
| `setApprovedModule(address module, bool approved)` | guardian | Strict subset of above — modules also authorised to call `moduleTransferSeat`. |
| `setModuleDisabled(address module, bool disabled)` | guardian | Soft kill switch for `moduleTransferSeat` from a module across every grid. |
| `pauseGridModules(uint gridId)` / `unpauseGridModules(uint gridId)` | guardian | Per-grid pause for `moduleTransferSeat`. |
| `moduleTransferSeat(uint gridId, uint seatId, address newHolder, SeatTransferRefund refundMode) returns (uint depositRefund)` | attached + approved + non-disabled module | Forced reassignment of a held seat. Emits `SeatTransferredByModule`. |

See [hooks/registration.md](../hooks/registration.md) and
[hooks/module-transfer-seat.md](../hooks/module-transfer-seat.md) for
the full flow and authorization gate.

## Pause flags (two-axis pause)

| Function | Caller | Effect |
|---|---|---|
| `pauseGrid(uint gridId)` / `unpauseGrid(uint gridId)` | guardian | Halts **all** seat ops on the grid. Economic + module flows. |
| `pauseGridModules(uint gridId)` / `unpauseGridModules(uint gridId)` | guardian | Halts only `moduleTransferSeat` on this grid. Economic flows continue. |
| `setModuleDisabled(address module, bool disabled)` | guardian | Per-module soft kill switch. |
| `pauseGridCreation()` / `unpauseGridCreation()` | guardian | Halts new `createGrid` / `createGridShell` calls protocol-wide. |

The full truth table is in [hooks/pause-flags.md](../hooks/pause-flags.md).

## Adapter-side surface

What an adapter contract calls on UGM:

| Function | When | Reverts if |
|---|---|---|
| `registerAsset(uint gridId, bytes32 assetHash)` | Inside `deposit`, after taking ownership of the source asset | adapter not approved; asset already registered; grid doesn't exist |
| `withdrawAsset(uint gridId, bytes32 assetHash)` | Inside `withdraw`, after sweeping last yield | caller isn't the registering adapter; asset not in grid |
| `receiveYieldETH(bytes32 assetHash, uint amount) payable` | Inside `collectYield` for ETH-yield grids | caller isn't the registering adapter; `msg.value != amount`; grid's yieldToken isn't `address(0)` (or flETH/WETH on the ETH path) |
| `receiveYieldERC20(bytes32 assetHash, address token, uint amount)` | Inside `collectYield` for ERC20-yield grids (or flETH/WETH on ETH grids) | caller isn't the registering adapter; wrong token for grid |

Adapter-side gas budget: `ADAPTER_COLLECT_GAS_CAP = 600_000`.
`collectYield` runs inside that budget; overrunning it makes UGM
silently skip the adapter for that `claimFees` invocation. See
[adapters/building-an-adapter.md](../adapters/building-an-adapter.md).

## Fees and yield (claim path)

| Function | Caller | Effect |
|---|---|---|
| `claimFees(uint[] gridIds)` | anyone (pulls for `msg.sender`) | Pulls adapter yield and pays out to the caller's seats across each gridId. |
| `pokeTax(uint gridId, uint[] seatIds)` | anyone | Applies pending tax to a batch of seats. Forfeits the tax-underwater ones. |
| `withdrawTaxRevenue(uint gridId, address receiver)` | grid creator (or `gridCreatorTaxReceiver`) | Sweep accumulated tax revenue. |
| `claimPayout(address token, address user)` | anyone | Drain the pull-based payout escrow when a push transfer failed. |

## Views (UGM v2.3)

| View | Returns |
|---|---|
| `ugmVersion()` | `bytes32("v2.3")` |
| `gridConfig(uint gridId)` | `(creator, createdAt, totalSeats, taxRateBps, taxToken, yieldToken)` — same 6-tuple as v2.2 |
| `gridConfigV22(uint gridId)` | `(forfeitureDuration, seatsMaterialized, seatsSold, graduated)` — auxiliary fields for indexers |
| `seats(uint gridId, uint seatId)` | `SeatInfo` (holder, price, deposit, timestamps, etc.) |
| `holderSeats(uint gridId, address holder)` | List of seat IDs held by an address |
| `holderSeatCount(uint gridId, address holder)` | Number of seats held |
| `totalYield(uint gridId)` | Cumulative yield deposited to this grid |
| `assetToGrid(bytes32 assetHash)` | Grid an asset is registered to |
| `assetAdapter(bytes32 assetHash)` | Adapter responsible for an asset |
| `approvedAdapters(address adapter)` | Whether an adapter is whitelisted |
| `gridGovernanceModule(uint gridId)` | Currently attached hook module (or `0x0`) |
| `approvedModules(address module)` | Whether a module is on the v2.3 transfer-authorised list |
| `moduleStatus(address module)` | `(approved, disabled)` — consolidated lookup, one RPC read |
| `isGridModulesPaused(uint gridId)` | Per-grid module pause flag |
| `gridPaused(uint gridId)` | Per-grid full pause flag |
| `gridDeprecated(uint gridId)` | Whether the grid has been creator-closed |
| `isGraduated(uint gridId)` | Whether all seats have been sold at least once |
| `creatorBaseline(uint gridId)` | Per-grid lazy-seat creator-yield baseline |

## Constants

| Constant | Value | Meaning |
|---|---|---|
| `ADAPTER_COLLECT_GAS_CAP` | 600,000 | Per-call gas budget for `collectYield` on adapters. |
| `GOVERNANCE_HOOK_GAS_CAP` | 150,000 | Per-call gas budget for `IGridHooksV23` callbacks. |
| `MAX_TOTAL_SEATS` | 4096 | Hard cap on `totalSeats`. |
| `MAX_SINGLE_TX_CREATE_TOTAL_SEATS` | 1024 | Above this, `createGridShell` + chunked pricing is required. |
| `MIN_TOTAL_SEATS` | 4 | Floor on `totalSeats`. |
| `BPS` | 10,000 | Basis-point denominator throughout. |

## Where to next

- [reference/events.md](events.md) — event catalogue indexers consume.
- [reference/deployments.md](deployments.md) — chain-specific addresses.
- [reference/glossary.md](glossary.md) — vocabulary.
- [hooks/lifecycle.md](../hooks/lifecycle.md) — hook callback dispatch
  details.
- [adapters/building-an-adapter.md](../adapters/building-an-adapter.md)
  — adapter lifecycle and gas budgeting.
