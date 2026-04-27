# Security considerations

> **Tier 2 placeholder.** Common adapter footguns and how to avoid them.

## Quick checklist (full version coming)

- **Always read `creator` and `yieldToken` from UGM during deposit.** Don't
  accept them as arguments. Either is a forging vector.
- **Gate `collectYield` on `msg.sender == address(ugm)`.** Anything else
  lets random callers drain pre-positioned approvals.
- **Be idempotent.** No new source-side activity must mean no transfers.
- **Use `forceApprove`.** USDT-style tokens revert on plain `approve`.
- **`nonReentrant` on adapter entrypoints, not on `collectYield`.** UGM
  already holds its lock when it calls in.
- **Bound your loops.** A `claimFees` that OOGs because of a 200-asset
  collect loop bricks the grid for that holder.
- **Don't pull non-yield value into the adapter.** Fees only. No buyout
  proceeds, no tax revenue, nothing else seat holders are entitled to via
  other paths.
- **Withdraw must sweep first.** Otherwise the source-side asset leaves with
  uncredited yield still in flight.
- **Source-protocol upgradeability.** Document what happens to seat holders
  if the source pauses or changes its fee math.

This page will be expanded with annotated examples of each footgun and the
diff that fixes it.
