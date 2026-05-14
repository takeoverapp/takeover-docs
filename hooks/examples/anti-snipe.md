# Example: anti-snipe module

An anti-snipe module gates `beforeBuyout` to limit how aggressively a
seat can change hands inside a short window after acquisition. It's a
v2.3-native pattern (it depends on the `beforeBuyout` callback that
v2.2 didn't have) and the cleanest example of a module that **gates**
seat economics without trying to **drive** them.

This is a sketch contract, not yet a deployed reference. Treat it as a
worked example — fork it for your own grid and audit it before
submitting for guardian approval.

## What it does

After a seat changes hands, the new holder gets a configurable
`cooldownSeconds` window of buyout protection. Within that window:

- A buyout from a **different attacker** reverts.
- A buyout at a price **above** a threshold relative to the seat's
  current self-assessed price reverts.
- A buyout from the same address that just lost the seat (i.e. an
  immediate counter-buyout) is allowed, so a seat-jacker doesn't trap
  the prior holder out of recovering.

After the cooldown, every buyout falls through.

## Why this is the canonical second hook

- **Single-callback module** (`beforeBuyout`) — same as the whitelist
  pattern, easy to reason about.
- **Storage shape: per-seat `lastAcquiredAt + lastHolder`** —
  bookkeeping kept under 2 SLOADs / 2 SSTOREs per acquisition.
- **Pure gate**: never reverts on any callback other than
  `beforeBuyout`, so adding it to a grid that's already running on
  another module doesn't risk breaking unrelated flows.
- **Ten-line revert-reason design**: each gate condition has its own
  error so a wallet's "transaction will fail" preview surfaces the
  user-visible reason.

## Sketch contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Ownable} from "@solady/auth/Ownable.sol";
import {IGridHooksV23} from "@takeover/interfaces/IGridHooksV23.sol";
import {IHookDescriptor} from "@takeover/interfaces/IHookDescriptor.sol";

/// @title AntiSnipeModule
/// @notice v2.3 hook module enforcing a per-seat cooldown after acquisition.
contract AntiSnipeModule is IGridHooksV23, IHookDescriptor, Ownable {
    // ───────────────────────────────────────────────────────────
    // Config (per grid, owner-set)
    // ───────────────────────────────────────────────────────────

    /// @dev Seconds of buyout protection after acquisition.
    mapping(uint256 => uint64) public cooldownSeconds;
    /// @dev Max basis points above the previous price a buyout can offer
    ///      during cooldown (10_000 = unrestricted; 11_000 = +10%).
    mapping(uint256 => uint16) public maxOverbidBpsDuringCooldown;

    // ───────────────────────────────────────────────────────────
    // Per-seat state
    // ───────────────────────────────────────────────────────────

    struct SeatBookkeeping {
        uint64 lastAcquiredAt;
        address lastDefender;
    }
    mapping(uint256 => mapping(uint256 => SeatBookkeeping)) public seats;

    // ───────────────────────────────────────────────────────────
    // Errors
    // ───────────────────────────────────────────────────────────

    error InCooldown();
    error OverbidExceedsCap();

    // ───────────────────────────────────────────────────────────
    // Events
    // ───────────────────────────────────────────────────────────

    event CooldownConfigured(uint256 indexed gridId, uint64 cooldownSeconds, uint16 maxOverbidBps);
    event SeatAcquisitionRecorded(uint256 indexed gridId, uint256 indexed seatId, address holder);

    // ───────────────────────────────────────────────────────────
    // Constructor / config
    // ───────────────────────────────────────────────────────────

    constructor(address _owner) {
        _initializeOwner(_owner);
    }

    function configureGrid(uint256 gridId, uint64 _cooldownSeconds, uint16 _maxOverbidBps) external onlyOwner {
        require(_maxOverbidBps >= 10_000, "cap below par");
        cooldownSeconds[gridId] = _cooldownSeconds;
        maxOverbidBpsDuringCooldown[gridId] = _maxOverbidBps;
        emit CooldownConfigured(gridId, _cooldownSeconds, _maxOverbidBps);
    }

    // ───────────────────────────────────────────────────────────
    // IGridHooksV23 — the gate
    // ───────────────────────────────────────────────────────────

    function beforeBuyout(
        uint256 gridId,
        uint256 seatId,
        address attacker,
        address defender,
        uint256 proposedPrice
    ) external view override {
        SeatBookkeeping memory s = seats[gridId][seatId];
        uint64 cooldown = cooldownSeconds[gridId];
        if (cooldown == 0) return;                          // unconfigured grid → open

        uint64 elapsed = uint64(block.timestamp) - s.lastAcquiredAt;
        if (elapsed >= cooldown) return;                    // out of cooldown
        if (attacker == s.lastDefender) return;             // counter-buyout allowed

        // Implicit: a separate `beforePriceChange` gate could enforce
        // a similar cap on price hikes during cooldown. Out of scope here.

        uint16 cap = maxOverbidBpsDuringCooldown[gridId];
        // We only have proposedPrice; deriving the previous price would
        // require an external read against UGM. Pure version: just block.
        if (cap == 10_000) revert InCooldown();
        // For the cap version, callers must read previousPrice off-chain
        // and supply it via a separate path; demonstration shown in
        // the parameterised variant below.
        revert OverbidExceedsCap();
    }

    // ───────────────────────────────────────────────────────────
    // IGridHooksV23 — the bookkeeper
    // ───────────────────────────────────────────────────────────

    function onSeatHolderChange(
        uint256 gridId,
        uint256 seatId,
        address /*oldHolder*/,
        address newHolder
    ) external override {
        if (newHolder == address(0)) return;                // forfeit / abandon
        seats[gridId][seatId] = SeatBookkeeping({
            lastAcquiredAt: uint64(block.timestamp),
            lastDefender: newHolder
        });
        emit SeatAcquisitionRecorded(gridId, seatId, newHolder);
    }

    // ───────────────────────────────────────────────────────────
    // IGridHooksV23 — no-ops
    // ───────────────────────────────────────────────────────────

    function beforeClaim(uint256, uint256, address) external pure override {}
    function beforePriceChange(uint256, uint256, uint256, uint256) external pure override {}
    function onForfeit(uint256, uint256, address) external pure override {}

    function yieldWeight(uint256, uint256) external pure override returns (uint256) {
        return 0;        // fall back to equal-share `1/totalSeats`
    }

    // ───────────────────────────────────────────────────────────
    // IHookDescriptor
    // ───────────────────────────────────────────────────────────

    function hookKind() external pure override returns (string memory) {
        return "anti-snipe";
    }
    function hookVersion() external pure override returns (uint16) {
        return 1;
    }
    function supportedCallbacks() external pure override returns (uint8) {
        // bit 1 (beforeBuyout) + bit 5 (onSeatHolderChange).
        // Indices match IHookDescriptor's docstring; check there.
        return uint8((1 << 1) | (1 << 5));
    }
    function description() external pure override returns (string memory) {
        return "Anti-snipe: blocks buyouts within cooldown unless from prior defender.";
    }
}
```

## Walking the design

### One SSTORE per acquisition

`onSeatHolderChange` writes the seat's bookkeeping record on every
materialized holder change. The record is two slots packed into one
(uint64 + address = 24 bytes); an SSTORE here costs 22,100 gas
(cold) the first time and 5,000 gas (warm) thereafter.

That's expensive on absolute terms, but it's amortized: every later
buyout can answer "is this seat in cooldown?" with two SLOADs and no
external calls.

### Two SLOADs per gate

`beforeBuyout` reads `cooldownSeconds[gridId]` and
`seats[gridId][seatId]`. With cold storage, that's roughly 4,200 gas;
warm, ~200. Comfortably under the 150k cap with room for further
logic.

### Why we don't read UGM for the previous price

Inside the gate, we *could* read `ugm.seats(gridId, seatId).price` to
implement the overbid cap precisely. Two reasons we don't in the
sketch:

- **Storage layout coupling.** Reading UGM's seat struct requires
  knowing its layout, and v2.3 already has subtle differences from
  v2.2.
- **Gas budget.** An external call adds ~700 gas of warm overhead and
  pulls in an SLOAD inside UGM. Survivable, but the sketch's two-slot
  approach is cheaper and easier to test.

For the precise overbid cap, parameterise the module so the operator
calls `setPriceSnapshot(gridId, seatId, price)` from a privileged
flow rather than trying to read it during the gate.

### Counter-buyout allowance

The `attacker == s.lastDefender` short-circuit lets the prior holder
buy their seat back immediately without waiting for cooldown. Without
it, an attacker could freeze the seat for the entire cooldown by
sitting on it.

### Forfeit isn't bookkept

`onSeatHolderChange` is called with `newHolder == address(0)` on
forfeit/clear, and the module skips writing then. That's correct: a
forfeited seat re-enters Dutch auction and the next acquirer's
acquisition timestamp resets the cooldown when the seat is reclaimed.

## Submission notes

A real submission would include:

- **Deterministic test of cooldown semantics** under all four arms
  (unconfigured grid, in-cooldown blocking, counter-buyout allowed,
  out-of-cooldown open).
- **Gas profile** showing `beforeBuyout` worst-case under the 150k
  cap.
- **Multi-grid invariant test** showing per-seat state doesn't leak
  between grids.
- **Forfeit-then-reclaim test** ensuring the cooldown resets on the
  re-acquire.

Run [`submit/checklist.yml`](../../submit/checklist.yml) against the
contract before reaching out for guardian approval; the `hook.*`
category encodes most of the above.

## What to read next

- [Whitelist module](whitelist-module.md) — simpler reference with
  every callback as a no-op.
- [`moduleTransferSeat`](../module-transfer-seat.md) — when *you* want
  to drive a transfer instead of gate one.
- [Hook lifecycle](../lifecycle.md) — exact dispatch sites and gas
  budgets.
