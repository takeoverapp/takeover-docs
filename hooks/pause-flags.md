# Two-axis pause + per-module disable

UGM v2.3 has three orthogonal kill-switch flags that together govern when
hooks fire and when `moduleTransferSeat` succeeds. They're independent so
guardians can halt a misbehaving module without freezing the marketplace,
or freeze the marketplace without bricking integrated apps.

## The three flags

| Flag | Type | Set by | Purpose |
|---|---|---|---|
| `gridPaused[gridId]` | per-grid | guardian (`pauseGrid` / `unpauseGrid`) | Halts **all** seat ops on the grid. Economic + module flows. |
| `gridModulesPaused[gridId]` | per-grid | guardian (`pauseGridModules` / `unpauseGridModules`) | Halts only `moduleTransferSeat` on this grid. Economic flows continue. |
| `moduleDisabled[module]` | per-module | guardian (`setModuleDisabled`) | Soft kill switch for `moduleTransferSeat` from this module across every grid it's attached to. |

`gridPaused` is the v2.2 hammer. The two new flags (`gridModulesPaused` and
`moduleDisabled`) are v2.3 additions designed for surgical responses to
module incidents.

## Truth tables

### When governance hooks fire (`beforeClaim` / `beforeBuyout` / `beforePriceChange` / `onSeatHolderChange` / `onForfeit`)

| `gridPaused` | `module on approvedGovernanceModules` | Hook fires? |
|---|---|---|
| true | any | **No** — every seat-op revert path triggers before hooks |
| false | true | **Yes** |
| false | false | **No** — silent no-op (revoked approval since attach) |

`gridModulesPaused` and `moduleDisabled` do **not** affect governance
callbacks. Those flags only gate `moduleTransferSeat`.

### When `moduleTransferSeat` succeeds

| `gridPaused` | `gridModulesPaused` | `moduleDisabled[caller]` | `approvedModules[caller]` | `caller == gridGovernanceModule[gridId]` | Result |
|---|---|---|---|---|---|
| true | any | any | any | any | revert `GridPaused` |
| false | true | any | any | any | revert `GridModulesPaused` |
| false | false | true | any | any | revert `UnauthorizedModule` |
| false | false | false | false | any | revert `UnauthorizedModule` |
| false | false | false | true | false | revert `UnauthorizedModule` |
| false | false | false | true | true | **succeeds** |

`UnauthorizedModule` collapses three distinct checks (allowlist + attach +
disable) into one error so a misconfigured caller can't probe state
through revert messages. To diagnose which check failed, read the four
public mappings directly via RPC.

### When `claimFees` and routine seat ops succeed

The economic flows (`addBatch`, `setPrice`, `claimFees`, `abandonSeat`,
`abandonBatch`) only check `gridPaused`. The two module flags have no
effect on them.

## Why the split exists

A v2.3 grid often has a hook module performing real work (whitelist gating,
DAO seat rotation, game-driven forced transfers). Three failure modes that
shouldn't all collapse to "pause the grid":

1. **Module bug in `moduleTransferSeat`** — module's general gating
   (`beforeClaim`, etc.) is fine, but its forced-transfer logic is
   misbehaving.
   *Response:* `setModuleDisabled(module, true)`. Marketplace continues,
   modular forced-transfers stop. Reverse with `setModuleDisabled(module,
   false)` once patched.

2. **Per-grid module incident** — one specific grid has a module behaving
   weirdly, but the same module is fine on other grids.
   *Response:* `pauseGridModules(gridId)`. Just that grid's
   `moduleTransferSeat` calls revert; everything else (its claim/buy
   flows, the same module's behavior on other grids) continues.

3. **Marketplace-level incident** — economic flows are broken (e.g.
   misbehaving yield adapter, exploit attempt).
   *Response:* `pauseGrid(gridId)`. Everything halts.

## Operator playbook

| Symptom | First lever |
|---|---|
| `moduleTransferSeat` is being abused on one grid | `pauseGridModules(gridId)` |
| `moduleTransferSeat` from a specific module is buggy on multiple grids | `setModuleDisabled(module, true)` |
| Module is permanently retired | `setApprovedModule(module, false)` then later `setApprovedGovernanceModule(module, false)` |
| Marketplace-level incident on one grid | `pauseGrid(gridId)` |
| Grid creator wants out of a module mid-grid | grid creator calls `setGridGovernanceModule(gridId, address(0))` (no guardian involvement) |

`setModuleDisabled` is intentionally cheaper than removing the module from
both allowlists — toggling the flag is a single SSTORE per side, and
flipping back is symmetric. Use it as the first response and only escalate
to allowlist removal if the issue is irrecoverable.

## Indexer signals

| Event | Emitted by | Indexer use |
|---|---|---|
| `GridPausedEvent(gridId, paused)` | `pauseGrid` / `unpauseGrid` | mark grid as halted |
| `GridModulesPausedEvent(gridId, paused)` | `pauseGridModules` / `unpauseGridModules` | mark "modules halted" status independently |
| `ModuleDisabledEvent(module, disabled)` | `setModuleDisabled` | flag affected modules across all grids |
| `ApprovedModuleUpdated(module, approved)` | `setApprovedModule` | track module allowlist changes |
| `GovernanceHookSoftFail(gridId, module, selector)` | `_callOnForfeit`, future `yieldWeight` consumer | detect misbehaving modules without on-chain reverts |

Builders of integrated apps should subscribe to `GridModulesPausedEvent`
and `ModuleDisabledEvent` so their UIs can surface "this action is
temporarily disabled by the protocol guardian" messaging instead of
generic revert reasons.

## Module author checklist

Build your hook module assuming any of the three flags can flip mid-flow:

- **Don't store optimistic state.** If `moduleTransferSeat` fails on the
  hop from your module to UGM (because `gridModulesPaused` was just
  flipped), your in-module bookkeeping must still be consistent. Snapshot
  your state changes only after UGM accepts the call.
- **Watch `GovernanceHookSoftFail` for your address.** UGM swallows
  observer-callback reverts, so your module won't see them surface as
  user-facing errors. The event is your only signal.
- **No-op behavior under revoke.** A module that's been
  `setApprovedGovernanceModule(false)`'d will still see its public
  entrypoints called by users (e.g. an admin function the module owner
  exposes). Make sure those still behave correctly without UGM callbacks
  to backstop.

## What to read next

- [Hook lifecycle and gas budgets](lifecycle.md) — what each callback
  does.
- [`moduleTransferSeat`](module-transfer-seat.md) — full forced-transfer
  semantics, including the four-check authorization gate.
- [Registration and the two-stage allowlist](registration.md) — how flags
  combine with the approval flow.
