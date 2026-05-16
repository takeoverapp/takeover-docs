# UGM v2.3

The **Unified Grid Manager v2.3** is the single immutable contract that
hosts every Takeover board. v2.3 is a strict superset of v2.2 — same
`IGrid` view surface, same yield adapter ABI, same Harberger mechanics —
plus four new capabilities purpose-built for builders integrating apps on
top:

1. **Hook modules** (`IGridHooksV23`) — gate seat economics
   (`beforeClaim` / `beforeBuyout` / `beforePriceChange`) and observe
   forfeit / yield events.
2. **Forced transfer entrypoint** (`moduleTransferSeat`) — let an
   approved module relocate a seat without the buyout payment path.
3. **Two-axis pause** (`gridPaused` + `gridModulesPaused`) — guardian
   can halt module flows without freezing the marketplace.
4. **Sharded grid creation** (`createGridShell` + `appendInitialPricingChunk`) —
   support boards up to 4,096 seats by chunking initial pricing across
   multiple transactions.

## Live deployments

| Network | Address |
|---|---|
| Base mainnet | [`0xF2DfBe1ef26AA82e9438dA95d5cC3007D3031F9e`](https://basescan.org/address/0xF2DfBe1ef26AA82e9438dA95d5cC3007D3031F9e) |
| Base Sepolia | [`0xC389D001C627C7fcE23405190d453906598c1607`](https://sepolia.basescan.org/address/0xC389D001C627C7fcE23405190d453906598c1607) |

The runtime image is split between `UnifiedGridManagerV23` and a linked
`UGMV23Linked` library (`DELEGATECALL`'d into) so the main contract stays
under EIP-170's 24,576-byte runtime cap. Storage lives entirely in the
main contract; the library is stateless. Full address book on
[Deployments](../reference/deployments.md).

## What stayed the same from v2.2

- **`IGrid` view surface.** `gridConfig(gridId)` still returns the same
  six-tuple `(creator, createdAt, totalSeats, taxRateBps, taxToken,
  yieldToken)`. Adapters and indexers don't need to change calldata.
- **`IYieldAdapter` interface.** Adapters built against v2.2 work
  unchanged on v2.3, modulo pointing the constructor at the new UGM
  address.
- **Harberger mechanics.** Self-assessed price, continuous tax, public
  buyouts, tax-deposit drain → forfeit + Dutch auction.
- **Per-grid yield/tax token configuration.** Set at create time,
  immutable thereafter.
- **Guardian role.** Same scope: pause grids, allowlist adapters /
  zaps / tax tokens, tune protocol fee BPS within hard caps.

## What's new in v2.3

### Hook modules

A grid creator can attach a hook module implementing
[`IGridHooksV23`](../hooks/interface.md). UGM calls into the module at
five lifecycle points:

| Callback | Class | Purpose |
|---|---|---|
| `onSeatHolderChange` | propagating observer (v2.2) | every materialized holder change |
| `beforeClaim` | gate | first-time acquisition of a vacant / lazy seat |
| `beforeBuyout` | gate | a held seat is bought out |
| `beforePriceChange` | gate | `setPrice` |
| `onForfeit` | swallowed observer | tax-underwater forfeit + Dutch entry |
| `yieldWeight` | swallowed view (reserved) | per-seat yield weight (not yet consumed) |

All callbacks run inside `GOVERNANCE_HOOK_GAS_CAP = 150,000` gas. Modules
that only implement the v2.2 surface (`onSeatHolderChange`) keep working
on v2.3 grids — the additional callbacks return empty revert data when
unimplemented, which UGM treats as silent no-op. See
[hooks/lifecycle.md](../hooks/lifecycle.md) for the full dispatch
semantics.

### `moduleTransferSeat`

A v2.3-exclusive entrypoint that lets an approved module forcibly
relocate a held seat to a new holder. Refund modes are explicit:

```solidity
enum SeatTransferRefund { None, Deposit }
```

`None` means the displaced holder loses both their unspent deposit and
their self-assessed price. `Deposit` returns the unspent deposit via
the standard payout escrow. The "phantom buyout" mode that would pay
the prior holder their listed price is intentionally absent — modules
needing that should call `addBatch` so the standard fee splits, hooks,
and indexer events fire.

The authorisation gate is four checks combined into one
`UnauthorizedModule()` revert: caller must be the attached governance
module, on the `approvedModules` allowlist, and not currently disabled.
Full breakdown in [hooks/module-transfer-seat.md](../hooks/module-transfer-seat.md).

### Two-axis pause

v2.2 had a single `gridPaused[gridId]` flag. v2.3 adds two more
orthogonal flags so guardians can respond to module incidents without
freezing the marketplace:

- `gridModulesPaused[gridId]` — halts only `moduleTransferSeat` on this
  grid.
- `moduleDisabled[module]` — soft kill switch for a module across every
  grid it's attached to.

Truth tables for what each flag halts in [hooks/pause-flags.md](../hooks/pause-flags.md).

### Sharded grid creation

`MAX_SINGLE_TX_CREATE_TOTAL_SEATS = 1024` is the atomic limit; above
that, `createGridShell` reserves the gridId without pricing, then
`appendInitialPricingChunk(gridId, prices)` is called repeatedly until
all seats are priced. Trades are blocked by `InitialPricingIncomplete()`
until the final chunk lands. Hard cap: `MAX_TOTAL_SEATS = 4096`.

For grids larger than 4096 seats, the SDK supports off-chain "sharded
boards" — multiple gridIds rendered as one logical canvas. Full
walkthrough in [boards/sharding.md](../boards/sharding.md).

## Where to next

- [The yield adapter model](yield-adapter-model.md) — building yield
  sources for v2.3 grids.
- [The hook model](hook-model.md) — module-side mental model.
- [UGM v2.3 API](../reference/ugm-api.md) — full method surface.
- [Events](../reference/events.md) — new v2.3 events the indexer
  consumes.
- [Agent skills](../skills/README.md) — task-focused guides for
  launching a grid, building an adapter, building a hook module.
