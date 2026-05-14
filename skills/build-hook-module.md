---
name: takeover-hook-module
description: >-
  Build a Takeover hook module — an `IGridHooksV23` implementation
  that adds gating, anti-snipe rules, weighted yield, or forced
  reassignment to a grid. Use when a user wants to plug game logic
  (military conquest, narrative loss), DAO tooling (council
  rotations), or prediction-market resolution into a UGM v2.3
  grid, or asks how `moduleTransferSeat` works.
---

# Skill: build a hook module

Hook modules are how external apps plug policy into a UGM v2.3
grid without forking the contract. The grid creator picks the
module at `setGridGovernanceModule(gridId, module)` time; UGM
calls it on every gateable seat event inside a gas-capped harness.

A module that ALSO appears on the guardian's
`approvedModules` allowlist gets a second power: the right to
call `moduleTransferSeat` (forcibly relocate seats, optionally
refunding the prior holder's deposit).

## Phase 0 — read before you write

1. [The hook model](../overview/hook-model.md) — what hooks can
   and can't do.
2. [The IGridHooksV23 interface](../hooks/interface.md) — every
   callback's exact signature + revert behavior.
3. [Hook lifecycle and gas budgets](../hooks/lifecycle.md) — when
   each callback fires and the
   `GOVERNANCE_HOOK_GAS_CAP = 150,000` budget.
4. [Registration and the two-stage allowlist](../hooks/registration.md) —
   guardian approval flow.
5. [Two-axis pause + per-module disable](../hooks/pause-flags.md) —
   `gridPaused`, `gridModulesPaused`, `moduleDisabled`.
6. [moduleTransferSeat: forced reassignment](../hooks/module-transfer-seat.md) —
   the v2.3-only privileged path.
7. Reference modules:
   [Whitelist module](../hooks/examples/whitelist-module.md),
   [Anti-snipe module](../hooks/examples/anti-snipe.md).

## Mental model in 60 seconds

UGM v2.3 calls FIVE callbacks plus a yield-weight view on the
attached module. All of them go through a gas-capped harness:

| Callback | Class | Gas cap | Revert behavior |
| --- | --- | --- | --- |
| `onSeatHolderChange(gridId, seatId, oldHolder, newHolder)` | required | 150,000 | propagates |
| `beforeClaim(gridId, seatId, claimant)` | required | 150,000 | propagates |
| `beforeBuyout(gridId, seatId, attacker, defender, proposedPrice)` | required | 150,000 | propagates |
| `beforePriceChange(gridId, seatId, oldPrice, newPrice)` | required | 150,000 | propagates |
| `onForfeit(gridId, seatId, lastHolder)` | observer | 150,000 | swallowed (`GovernanceHookSoftFail` event emitted) |
| `yieldWeight(gridId, seatId) returns (uint256)` | view | 150,000 (staticcall) | swallowed → falls back to `1/totalSeats` equal share |

A module that implements only the v2.2 surface
(`onSeatHolderChange` from `IGridGovernanceHooks`) keeps working —
UGM detects the missing-selector revert from the unimplemented
callbacks and treats it as "no policy".

## Pick a class

| Intent | Required callbacks | Approved-module needed? |
| --- | --- | --- |
| Whitelist / reservation gating before claim | `beforeClaim` | No |
| Anti-snipe (delay buyouts in the first N blocks) | `beforeBuyout` | No |
| Min/max self-assessed price corridors | `beforePriceChange` | No |
| Forfeit observer / off-chain telemetry | `onForfeit` | No |
| Weighted yield distribution (e.g. by building value) | `yieldWeight` view | No |
| Forced reassignment (military conquest, council shift, market resolution) | `moduleTransferSeat` ↔ entrypoint | **Yes** — guardian must `setApprovedModule(module, true)`. |

Combine as needed — a "Hex tile game" module typically
implements `beforeBuyout` (delay window), `yieldWeight` (building
value), and calls `moduleTransferSeat` (military conquest).

## Phase 1 — implement

Mandatory invariants:

1. **No reverts in observer paths.** `onForfeit` MUST NOT revert
   under any input; UGM logs `GovernanceHookSoftFail` if it does
   and the seat op continues. A reverting observer is a bug.
2. **Stay under the gas cap.** `GOVERNANCE_HOOK_GAS_CAP = 150,000`
   covers each callback. Cache reads, avoid storage writes
   inside `yieldWeight`, and don't call into other contracts that
   themselves have unknown gas profiles.
3. **Key state by `gridId`.** A single module instance can be
   attached to many grids. Keying by `seatId` alone leaks state
   between grids. The reference
   [WhitelistGovernanceModuleV2](../hooks/examples/whitelist-module.md)
   uses `mapping(uint256 gridId => mapping(uint256 seatId => …))`.
4. **Don't assume `onSeatHolderChange` fires before
   `beforeClaim`.** They fire in a defined order documented in
   [hooks/lifecycle.md](../hooks/lifecycle.md) — read it; the
   contract is the source of truth.
5. **`moduleTransferSeat` callers must:**
   - Be the same module the grid creator attached via
     `setGridGovernanceModule`.
   - Be on the guardian's `approvedModules` allowlist.
   - Pass `SeatTransferRefund.None` or `SeatTransferRefund.Deposit`.
     There is no `Price` mode — phantom buyouts must route
     through `addBatch` instead.
6. **Document your pause posture.** If your module has its own
   internal pause / kill switch, the existence of UGM-level
   `gridPaused`, `gridModulesPaused`, and per-module
   `moduleDisabled` MUST not surprise integrators. State
   explicitly which UGM flag your module reacts to.
7. **Implement `IHookDescriptor`** if the dashboard /
   `/builders/contracts` should pick up your module:
   `hookKind`, `hookVersion`, `supportedCallbacks`,
   `description`. Returning an accurate `supportedCallbacks`
   bitmask lets frontends surface "this module gates buyouts" /
   "this module sets yield weights" without reading source.

## Phase 2 — test

Write tests that map 1:1 to the `hook.*` items in
[`submit/checklist.yml`](../submit/checklist.yml):

- `hook.onseatholderchange_keys_by_gridid`
- `hook.beforeclaim_revert_blocks_acquisition`
- `hook.beforebuyout_revert_blocks_buyout`
- `hook.onforfeit_never_reverts`
- `hook.yield_weight_within_gas_cap`
- `hook.module_transfer_seat_only_attached_module` (if applicable)
- `hook.module_transfer_seat_refund_modes` (if applicable)
- `hook.no_state_writes_in_yield_weight`
- `hook.fork_test_e2e_v23`

Run:

```bash
forge test -vv
forge test --gas-report -vv
forge coverage --report summary
```

The gas report MUST show every callback under
`GOVERNANCE_HOOK_GAS_CAP`. The fork test pins a real v2.3
deployment (mainnet or Sepolia per
[Deployments](../reference/deployments.md)) and exercises a full
seat lifecycle with the module attached.

## Phase 3 — audit

Same bar as adapters. The audit must cover:

1. The module contract.
2. Every external call (e.g. into UGM for
   `moduleTransferSeat`).
3. The deployment script.
4. The `Ownable` / governance surface — owner-only functions
   must be safe even with a hostile owner.
5. The interaction surface with other modules / adapters
   attached to the same grid.

See [Audit expectations](../submit/audit-expectations.md).

## Phase 4 — verify against the checklist

```bash
curl -fsSL https://raw.githubusercontent.com/takeoverapp/takeover-docs/main/submit/checklist.yml -o checklist.yml
```

Iterate every `hook.*` item and produce a structured report
identical in format to the adapter skill's. Cite file:line for
every claim. `[FAIL]` items must be fixed or justified before
submission.

## Phase 5 — submit

Send the structured report and the audit PDF to
[@takeoverfun on X](https://x.com/takeoverfun). The guardian will
queue your module for `setApprovedGovernanceModule(module, true)`
and (if applicable) `setApprovedModule(module, true)`. Until both
calls land, attaching the module to a grid will revert with
`ModuleNotApproved()`.

## What you should not do

- **Do not call `moduleTransferSeat` without the
  `approvedModules` allowlist bit.** UGM reverts with
  `NotApprovedModule()`.
- **Do not implement `Price` refund mode.** It does not exist.
  Route economic takeovers through `addBatch` so all the standard
  fee splits + indexer events fire.
- **Do not write storage in `yieldWeight`.** It's a view; UGM
  invokes it via `staticcall` and the call reverts on any state
  write. The fallback equal-share path then kicks in, which is
  not what you want.
- **Do not silently soft-fail in `beforeClaim` /
  `beforeBuyout` / `beforePriceChange`.** A revert in those
  callbacks IS the gating mechanism. Use a clean custom error
  (`SeatReservedForAnotherAddress()`, `BuyoutDelayActive()`,
  …) so frontends can surface why an op failed.

## Pinned facts

- Interfaces live at
  [`takeover-contracts/src/interfaces/IGridHooksV23.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/interfaces/IGridHooksV23.sol)
  and
  [`takeover-contracts/src/interfaces/IGridGovernanceHooks.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/interfaces/IGridGovernanceHooks.sol).
- Reference module:
  [`takeover-contracts/src/WhitelistGovernanceModuleV2.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/WhitelistGovernanceModuleV2.sol).
- Gas cap is **`150,000`** per callback (`GOVERNANCE_HOOK_GAS_CAP`).
- Approval channel is **DM [@takeoverfun on X](https://x.com/takeoverfun)**.
- The two-stage allowlist:
  `setApprovedGovernanceModule` (lets the grid creator attach the
  module) AND `setApprovedModule` (lets the module call
  `moduleTransferSeat`). Most modules need only the first.
