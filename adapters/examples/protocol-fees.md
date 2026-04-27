# Example: `ProtocolYieldAdapter`

> **Tier 2 placeholder.** Full annotated walkthrough of the protocol-fee
> push-only adapter.
>
> **Status:** this is the only adapter currently registered on **UGM v2.1**
> (it feeds the Boardroom / hvTAKEOVER grid with USDC protocol fees pulled
> from v2 UGMs and v1 TSFM pools). See [Deployments](../../reference/deployments.md).

## Pattern: push-only, `collectYield` is a no-op

`ProtocolYieldAdapter` doesn't hold an NFT. It's the canonical
**push-only** pattern:

1. The adapter implements `IFeeReceiver.notifyDeposit(gridId, amount)` so
   that **other** UGM-deployed grids can route their protocol fees here.
2. Anyone can call `pokeUgmFees(...)` / `pokeTsfmFees(...)` / `pokeAll(...)`
   to pull USDC out of v2 UGM grids and v1 TSFM contracts in one tx.
3. After pulling, the adapter calls `_flushYield` which forwards 100% of
   its USDC balance to the hvTAKEOVER grid via `receiveYieldERC20`.
4. UGM's `collectYield` callback is a **no-op** — there's nothing to pull,
   the yield is already in the contract.

> Source: [`takeoverapp/takeover-contracts/src/ProtocolYieldAdapter.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/ProtocolYieldAdapter.sol).

Use this pattern when:

- The yield "source" is itself a contract that calls into your adapter.
- There's no per-asset object to hold (no NFT, no escrow handle).
- You want pulls to be permissionless (public goods style, like the
  Boardroom keeper).

A full walkthrough lands in Tier 2.
