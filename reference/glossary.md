# Glossary

Every Takeover-specific term these docs use, plus the v2.3 additions.

| Term | Definition |
|---|---|
| **Adapter / yield adapter** | Contract that bridges an external yield source into UGM. Implements [`IYieldAdapter`](../adapters/interface.md). |
| **`ADAPTER_COLLECT_GAS_CAP`** | UGM v2.3 caps every `collectYield` invocation at 600,000 gas. Adapters that exceed it are silently skipped for that `claimFees` call. |
| **Asset hash** | `bytes32` handle UGM uses to identify a yield source attached to a grid. Globally unique. See [Asset hashes](../adapters/asset-hash.md). |
| **Atomic grid creation** | `createGrid(params)` — single tx, full initial pricing supplied inline. Limited to `totalSeats <= MAX_SINGLE_TX_CREATE_TOTAL_SEATS` (1024). |
| **Board** | Product term for a grid. Used interchangeably in user-facing surfaces. |
| **Boardroom** | The hvTAKEOVER grid. Receives protocol-wide tax revenue and redistributes it to its own seat holders. `HV_GRID_ID = 0` on UGM v2.1. |
| **Buyout** | Force-buying a held seat at the holder's self-assessed price. |
| **Chunked grid creation** | `createGridShell(params)` followed by N×`appendInitialPricingChunk(gridId, prices)`. Required for grids with `1024 < totalSeats <= MAX_TOTAL_SEATS`. Trades blocked by `InitialPricingIncomplete()` until pricing finishes. |
| **Forfeiture** | Seat is auto-released when its deposit is exhausted by tax. |
| **Forced transfer** | Module-driven seat reassignment via `moduleTransferSeat`. Distinct from buyouts (no payment to displaced holder beyond optional `Deposit` refund). |
| **Grid** | The onchain object backing a board. Identified by `gridId`. |
| **Grid creator** | Address that called `createGrid` (or `createGridShell`). Has special powers: register adapters, redirect tax/sale receivers, withdraw assets when they hold all seats, attach hook modules. |
| **`gridModulesPaused`** | Per-grid pause flag for `moduleTransferSeat`. Doesn't affect economic flows. Set by guardian's `pauseGridModules`. |
| **`gridPaused`** | Per-grid pause flag for **all** seat ops. The marketplace-level kill switch. Set by guardian's `pauseGrid`. |
| **`GOVERNANCE_HOOK_GAS_CAP`** | UGM v2.3 caps every `IGridHooksV23` callback at 150,000 gas. Reverts with empty data are treated as "didn't implement" (silent no-op); reverts with data propagate. |
| **Guardian** | Privileged role on UGM. Whitelists adapters/zaps/tax tokens/modules, tunes protocol fees, can pause grids and disable modules. Today: a Takeover protocol multisig. |
| **Harberger taxation** | Self-assessed price + continuous tax + open buyouts. The ownership model every Takeover seat uses. |
| **Hook module** | Contract implementing `IGridHooksV23` that a grid creator attaches to gate seat economics (claim, buyout, price change, forfeit) or drive forced transfers. |
| **`IFeeReceiver`** | Callback interface UGM uses to notify a contract that just received tax/sale revenue. |
| **`IGrid`** | Minimal interface for a Harberger seat ledger. UGM v2.3 implements it. |
| **`IGridHooksV23`** | The hook surface UGM v2.3 calls into. Strict superset of v2.2's `IGridGovernanceHooks` — old modules keep working. |
| **`IYieldAdapter`** | Two-function interface every yield adapter implements: `collectYield` and `pendingYield`. |
| **`MAX_TOTAL_SEATS`** | Hard cap on grid `totalSeats`. UGM v2.3: 4096. |
| **`MAX_SINGLE_TX_CREATE_TOTAL_SEATS`** | Atomic creation cap. UGM v2.3: 1024. Above this, use chunked creation. |
| **`MIN_TOTAL_SEATS`** | Floor on grid `totalSeats`. UGM v2.3: 4. |
| **`moduleDisabled`** | Per-module soft kill switch. Toggling disables `moduleTransferSeat` calls from the module across every grid. |
| **`moduleTransferSeat`** | v2.3-exclusive entrypoint for an authorised module to forcibly relocate a seat. Refund modes: `None` (looted) or `Deposit` (refund unspent deposit). See [hooks/module-transfer-seat.md](../hooks/module-transfer-seat.md). |
| **`pendingYield`** | View on an adapter that returns the best-effort estimate of yield owed to a single asset, denominated in the grid's `yieldToken`. |
| **Pre-buy / seat pre-buy** | Buying seats atomically with grid creation, usually via a launch zap. |
| **Seat** | Unit of ownership on a grid. One seat = 1/N of the grid's yield (where N = `totalSeats`) under the default equal-share allocation. |
| **`SeatTransferRefund`** | Enum picked by a module when calling `moduleTransferSeat`. `None` (looted, no refund) or `Deposit` (return unspent tax deposit). |
| **Sharded board** | An off-chain composition of multiple gridIds presented as one logical canvas. SDK-side only — there is no "board" entity onchain. Used for boards above `MAX_TOTAL_SEATS`. See [boards/sharding.md](../boards/sharding.md). |
| **Source protocol** | The external protocol whose yield an adapter wraps (Flaunch, Uniswap, etc.). |
| **Tax token** | ERC20 holders pay tax in. Configured per grid. Must be guardian-allowlisted. |
| **Two-axis pause** | UGM v2.3 separates `gridPaused` (full pause) from `gridModulesPaused` (module-only pause) so guardians can halt module-driven flows without freezing the marketplace. See [hooks/pause-flags.md](../hooks/pause-flags.md). |
| **TSFM** | Legacy `TakeoverFeeSplitManager` (v1). Predecessor to UGM. |
| **UGM** | UnifiedGridManager. The contract that hosts every grid. v2.3 today (`UnifiedGridManagerV23` + `UGMV23Linked` library). |
| **`UGMV23Linked`** | Linked Solidity library `DELEGATECALL`'d by `UnifiedGridManagerV23` to keep the main contract under EIP-170's 24,576-byte runtime cap. |
| **`yieldWeight`** | `IGridHooksV23` view callback returning per-seat yield weight. **Reserved** — UGM v2.3 doesn't currently consume it; falls back to equal share. |
| **Yield token** | ERC20 (or ETH if `address(0)`) that yield is paid in for a given grid. Set at `createGrid` time, immutable after. |
| **Zap** | Router contract that batches multiple actions into one tx. Adapter-facing zaps need `setApprovedZap` on UGM. |
