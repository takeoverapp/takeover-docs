# Glossary

Every Takeover-specific term these docs use.

| Term | Definition |
|---|---|
| **Adapter / yield adapter** | Contract that bridges an external yield source into UGM. Implements [`IYieldAdapter`](../adapters/interface.md). |
| **Asset hash** | `bytes32` handle UGM uses to identify a yield source attached to a grid. Globally unique. See [Asset hashes](../adapters/asset-hash.md). |
| **Board** | Product term for a grid. Used interchangeably in user-facing surfaces. |
| **Boardroom** | The hvTAKEOVER grid. Receives protocol-wide tax revenue and redistributes it to its own seat holders. `HV_GRID_ID = 0`. |
| **Buyout** | Force-buying a held seat at the holder's self-assessed price. |
| **Forfeiture** | Seat is auto-released when its deposit is exhausted by tax. |
| **Grid** | The onchain object backing a board. Identified by `gridId`. |
| **Grid creator** | Address that called `createGrid`. Has special powers: register adapters, redirect tax/sale receivers, withdraw assets when they hold all seats. |
| **Guardian** | Privileged role on UGM. Whitelists adapters/zaps/tax tokens, tunes protocol fees, can pause grids. Today: a Takeover protocol multisig. |
| **Harberger taxation** | Self-assessed price + continuous tax + open buyouts. The ownership model every Takeover seat uses. |
| **`IFeeReceiver`** | Callback interface UGM uses to notify a contract that just received tax/sale revenue. |
| **`IGrid`** | Minimal interface for a Harberger seat ledger. UGM v2.1 implements it. |
| **`IYieldAdapter`** | Two-function interface every yield adapter implements: `collectYield` and `pendingYield`. |
| **`pendingYield`** | View on an adapter that returns the best-effort estimate of yield owed to a single asset, denominated in the grid's `yieldToken`. |
| **Pre-buy / seat pre-buy** | Buying seats atomically with grid creation, usually via `GridLaunchZap`. |
| **Seat** | Unit of ownership on a grid. One seat = 1/N of the grid's yield (where N = `totalSeats`). |
| **Source protocol** | The external protocol whose yield an adapter wraps (Flaunch, Uniswap, etc.). |
| **Tax token** | ERC20 holders pay tax in. Configured per grid. Must be guardian-allowlisted. |
| **TSFM** | Legacy `TakeoverFeeSplitManager` (v1). Predecessor to UGM. |
| **UGM** | UnifiedGridManager. The contract that hosts every grid. v2.1 today. |
| **vNext** | In-flight UGM successor. Out of scope for these docs until it ships. See `PLAN_UGM_VNEXT.md` in `takeover-contracts`. |
| **Yield token** | ERC20 (or ETH if `address(0)`) that yield is paid in for a given grid. Set at `createGrid` time, immutable after. |
| **Zap** | Router contract that batches multiple actions into one tx. Adapter-facing zaps need `setApprovedZap` on UGM. |
