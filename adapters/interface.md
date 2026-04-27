# The `IYieldAdapter` interface

Every adapter implements two functions. That's it.

```solidity
interface IYieldAdapter {
    function collectYield(bytes32[] calldata assetHashes) external;
    function pendingYield(bytes32 assetHash) external view returns (uint256);
}
```

> Source: [`takeoverapp/takeover-contracts/src/interfaces/IYieldAdapter.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/interfaces/IYieldAdapter.sol).

The rest of the adapter — deposit/withdraw flows, swap logic, holder gating —
is up to you. UGM only ever calls the two functions above.

## `collectYield(bytes32[] assetHashes)`

> "For each of these assets you registered with me, pull the pending yield
> from the source and push it back to me. I'll be calling this from inside my
> own reentrancy lock."

**Caller:** UGM v2.1 only. Implementations must revert (or no-op) if `msg.sender`
isn't UGM.

```solidity
if (msg.sender != address(ugm)) revert NotUGM();
```

**Semantics:**

- For each `assetHash` in the input, the adapter must pull whatever yield is
  newly available from the source, convert it to the grid's `yieldToken`, and
  forward it to UGM via `receiveYieldETH` (for ETH-yield grids) or
  `receiveYieldERC20` (for ERC20-yield grids).
- **Idempotent.** Calling twice in a row with no new source-side activity is a
  no-op — no transfers, no reverts, just an early return.
- **Best-effort, not all-or-nothing.** If asset 3 of 5 has nothing to collect,
  skip it; don't revert the whole call. Seat holders rely on `collectYield`
  not failing for an unrelated reason.
- **No external user money.** The function takes no value, no token approvals.
  Yield it pulls is yield the adapter already has the right to.
- **Bounded gas.** UGM calls `collectYield` inside `claimFees`; if it OOGs,
  no holder on this grid can claim. Cap loops, batch sensibly, and prefer
  `try/catch` over revert when interacting with the source.

**Inside-the-lock note.** `collectYield` runs while UGM holds its reentrancy
guard. The adapter therefore **must not** be `nonReentrant` itself in a way
that conflicts (or it must use a separate lock). Reference adapters use
Solady's `ReentrancyGuard` on adapter-only entrypoints (deposit, withdraw,
poke, flush) and leave `collectYield` unguarded — UGM's lock is enough.

## `pendingYield(bytes32 assetHash) view returns (uint256)`

> "Without doing any work or mutating state, give me your best estimate of
> how much yield is sitting in this asset right now, denominated in the
> grid's yieldToken."

**Caller:** anyone. UI, indexer, keepers, off-chain APIs. Pure read.

**Semantics:**

- Returns 0 if the `assetHash` isn't active on this adapter.
- Returns a value denominated in the grid's `yieldToken`. If the source
  produces fees in two tokens (e.g. a Uniswap LP earns fees in both pool
  tokens), the adapter must convert to `yieldToken` units using a price
  source it trusts.
- **Best-effort.** UI and indexers display this number; small slippage and
  pricing rounding are expected.
- **No reverts on missing source data.** If the source doesn't expose a price
  cleanly, return the raw fee-token amount or 0 — never revert. Reverting
  here would brick UI list views.

A few patterns from the reference adapters:

- **Single-token push** (`FlaunchYieldAdapter`): yield is already in the
  grid's `yieldToken`, so `pendingYield = currentTotalFees - lastFeesAllocated`.
  No conversion.
- **Two-token swap-and-forward** (`V3YieldAdapter`, `V4YieldAdapter`): query
  the position's fee growth, decompose into `owed0` / `owed1`, then convert
  the non-yield side using the position's own pool price (`sqrtPriceX96`).
- **Push-only** (`ProtocolYieldAdapter`): the adapter holds a USDC balance
  waiting to be flushed; `pendingYield = usdc.balanceOf(this)`.

## What's *not* in `IYieldAdapter`

The interface deliberately doesn't include:

- **Deposit / withdraw.** Those are adapter-specific. An adapter that holds
  Uniswap V3 NFTs needs `deposit(positionManager, tokenId, gridId)`; an
  adapter that holds nothing (push-only) needs no deposit at all.
- **Asset registration.** That's a UGM call (`registerAsset`), not an adapter
  call. The adapter calls UGM during deposit; UGM doesn't call back.
- **Owner / guardian admin.** Adapters are owned (Solady `Ownable`) for
  emergency rescue, but UGM doesn't enforce that. Use whatever access-control
  pattern fits your operational model.

## Where to next

- [Adapter lifecycle](lifecycle.md) — when each function gets called.
- [Building a yield adapter](building-an-adapter.md) — fill in the rest of
  the contract.
- [Yield token rules](yield-token-rules.md) — what `receiveYieldETH` and
  `receiveYieldERC20` actually accept.
