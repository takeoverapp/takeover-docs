# Example: whitelist module

A whitelist module gates first-time seat acquisition: only specific
addresses can claim specific seats before a deadline; after the
deadline, every seat opens to the public.

The reference contract on UGM v2.3 is
[`WhitelistGovernanceModuleV2`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/WhitelistGovernanceModuleV2.sol).
It still inherits the v2.2 `IGridHooksV2` interface (which UGM v2.3
treats as a strict subset — see
[hooks/interface.md](../interface.md)) and ships unchanged on v2.3
grids. This page walks you through the contract and points out the
v2.3-specific surfaces you'd touch when forking it.

## What it does

- `setReservations(gridId, seatIds[], claimants[])` — owner pins each
  seat to a specific claimant.
- `setClaimDeadline(gridId, deadline)` — owner picks the cutoff. Until
  `deadline` passes, reservations are enforced. After, the module
  becomes a no-op gate.
- `beforeClaim(gridId, seatId, claimant)` — UGM's first-time-acquire
  hook. The module reverts with `SeatReservedForAnotherAddress` if a
  reserved seat is being claimed by anyone other than its claimant.

Every other callback is a no-op: claimed seats (held by non-creators)
are always contestable; price changes, forfeits, and yield don't
participate in whitelist policy.

## Why this is the canonical first hook

- **One callback does all the work** (`beforeClaim`).
- **State is keyed by `(gridId, seatId)`** so a single module instance
  can serve multiple grids without reservation bleed-over.
- **Gas-bounded**: the gate is two SLOADs (`claimDeadline[gridId]` +
  `reservedSeat[gridId][seatId]`) plus three address comparisons,
  comfortably under the 150k cap.
- **No ambient state**: every check is local to the seat being
  acquired.

## Walking the contract

### Storage shape

```solidity
mapping(uint256 => mapping(uint256 => address)) public reservedSeat;
mapping(uint256 => uint256) public claimDeadline;
mapping(uint256 => uint256) public reservationCount;
```

Note the per-grid keying. Earlier whitelist modules used
`mapping(uint256 => address) reservedSeat` keyed only by `seatId`,
which meant attaching the same module to two grids leaked
reservations across them. v2 fixes that.

### `beforeClaim`

```solidity
function beforeClaim(uint256 gridId, uint256 seatId, address claimant) external view override {
    uint256 d = claimDeadline[gridId];
    if (d != 0 && block.timestamp > d) return;     // post-deadline → open

    address reserved = reservedSeat[gridId][seatId];
    if (reserved == address(0)) return;            // unreserved seat → open
    if (claimant == address(0)) return;            // not a real acquisition
    if (claimant == reserved) return;              // reservation holder
    revert SeatReservedForAnotherAddress();
}
```

Three short-circuit returns and one revert. The early returns are
deliberately permissive so the module never blocks something it
shouldn't.

The implicit "deadline never set" branch (`d == 0`) is critical:
without it, deploying the module against a grid would silently open
every reserved seat the moment the contract was attached. Default
state is "reservations enforced indefinitely until the owner sets a
non-zero deadline."

### Other callbacks

```solidity
function beforeBuyout(uint256, uint256, address, address, uint256) external pure override {}
function afterClaim(uint256, uint256, address) external override {}
function beforePriceChange(uint256, uint256, uint256, uint256) external override {}
function onForfeit(uint256, uint256, address) external override {}
function beforeTaxAccrual(uint256, uint256) external override {}
function beforeClaimFees(uint256, address) external override {}
function onYieldReceived(uint256, bytes32, address, uint256) external override {}
```

Empty bodies. UGM's safety harness invokes every callback
unconditionally; reverts here would propagate to seat ops, so empty
implementations are required (`pure` where possible to save the
SLOAD overhead).

The v2.3 callbacks (`yieldWeight`, the additional `beforeBuyout`
signature, etc.) aren't declared on this contract because it
inherits `IGridHooksV2`. UGM v2.3's safety harness handles the
"selector not found" path silently — see
[lifecycle.md](../lifecycle.md). The result: **this exact contract
deploys onto a v2.3 grid without modification**. You only need to
update it if you want the v2.3 callbacks to do non-trivial work.

### `IHookDescriptor`

```solidity
function hookKind() external pure override returns (string memory) { return "whitelist"; }
function hookVersion() external pure override returns (uint16) { return 2; }
function supportedCallbacks() external pure override returns (uint8) {
    return SUPPORTED_CALLBACKS;   // bit 0 only (beforeClaim)
}
function description() external pure override returns (string memory) {
    return "Reservation gating: reserved seats only to their claimants pre-deadline, fully open after.";
}
```

`IHookDescriptor` is informational; builder UIs use it to render which
callbacks the module participates in without having to invoke them.
Returning the bitmask (`bit 0 == beforeClaim`) keeps frontends accurate.

### Owner ops

```solidity
function setReservations(uint256 gridId, uint256[] calldata seatIds, address[] calldata claimants) external onlyOwner;
function setClaimDeadline(uint256 gridId, uint256 deadline) external onlyOwner;
```

Both fire indexer-watched events
(`ReservationsSet(gridId, count)`,
`ClaimDeadlineSet(gridId, deadline)`).

### `getUnclaimedReservations` view

The contract exposes a helper that walks `reservedSeat[gridId][...]`
against UGM's current seat holders and returns the unclaimed entries
so a creator can `abandonSeat` them after `claimDeadline`. Bound by
`totalSeats`, so practical limit is around 4096 (UGM's
`MAX_TOTAL_SEATS`); above that, paginate from the indexer.

## Forking it for v2.3

If you want to extend it (e.g. add a `beforePriceChange` ban during the
reservation window):

1. **Add a real implementation** for the additional callback.
2. **Update `IHookDescriptor.supportedCallbacks`** so the bitmask
   reflects the new participation.
3. **Update `hookVersion`** so frontends can detect the change.
4. **Profile under the gas cap.** Whitelist's two SLOADs leave plenty
   of headroom; a third SLOAD plus a comparison is still safe. SSTOREs
   from inside a hook are dangerous — leave them to the owner ops.
5. **Audit the per-grid key invariant.** Any new mapping needs to be
   keyed by `(gridId, ...)` to preserve the multi-grid safety
   guarantee.

## Submission

The whitelist module is already on the
`approvedGovernanceModules` allowlist on Base mainnet and Sepolia.
A fork submitted for approval should:

- Run [`submit/checklist.yml`](../../submit/checklist.yml) (the
  v2.3 schema with the `hook.*` category).
- Document any new callback's gas profile.
- Provide a minimal repro showing reservations don't leak across
  grids when the same instance is attached to multiple grids
  (the multi-grid keying invariant).

## What to read next

- [Anti-snipe module](anti-snipe.md) — a more aggressive `beforeBuyout`
  example.
- [Hook lifecycle](../lifecycle.md) — what each callback can and can't do.
- [Registration](../registration.md) — how to get a forked module
  approved.
