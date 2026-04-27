# Pre-submission checklist

Before requesting [guardian approval](adapter-approval-process.md) for a new
yield adapter, walk this list. Anything missing here is going to come back as
a review comment.

## Contract

- [ ] Implements [`IYieldAdapter`](../adapters/interface.md): `collectYield`
      and `pendingYield`.
- [ ] `collectYield` reverts if `msg.sender != address(ugm)`.
- [ ] `collectYield` is **idempotent**: two calls in a row with no source-
      side activity move zero tokens.
- [ ] `collectYield` is **best-effort per asset**: a problem with one asset
      doesn't revert the whole batch.
- [ ] `pendingYield` is a pure read; never reverts on missing/zero state.
- [ ] Yield is forwarded in the **grid's `yieldToken`** units. ETH-yield
      grids accept flETH, WETH, or raw ETH; ERC20-yield grids accept only
      that exact token.
- [ ] `forceApprove` is used for ERC20 approvals to UGM (USDT-style tokens
      will revert on plain `approve`).
- [ ] Deposit gates on the **grid creator**, read freshly from
      `ugm.gridConfig(gridId)`.
- [ ] Deposit verifies the source matches the grid's `yieldToken`.
- [ ] `assetHash` is unique per source-side object and includes the source
      contract address in its preimage.
- [ ] Withdraw is gated. Default reference gate: grid creator holds all
      seats. If yours is different, document it on the example page.
- [ ] Withdraw sweeps last yield (`_collectSingle`) before transferring the
      source-side asset back.
- [ ] No EOA-only entrypoints. UI, zaps, and contracts all need to integrate.
- [ ] Owner is set; emergency rescue functions are owner-only.
- [ ] No unbounded loops on a hot path. Cap any per-call work.
- [ ] Reentrancy: adapter-only entrypoints (deposit, withdraw, poke, flush)
      are `nonReentrant`. `collectYield` is **not** `nonReentrant` — UGM
      already holds its lock on that path.

## Tests

- [ ] Happy-path deposit + `claimFees` → seat balance moves.
- [ ] Idempotent collect: second `claimFees` in a row collects zero.
- [ ] `collectYield([fakeHash])` is a no-op, not a revert.
- [ ] `collectYield` from a non-UGM caller reverts.
- [ ] Deposit on a grid with incompatible `yieldToken` reverts.
- [ ] Withdraw without all seats reverts; with all seats succeeds.
- [ ] Approval revoke path: after `setApprovedAdapter(adapter, false)`,
      deposit reverts but residual yield still pushes and `withdrawAsset`
      still works for already-registered assets.
- [ ] If your adapter swaps, fuzz the swap path on edge prices (very low /
      very high `sqrtPriceX96`).
- [ ] Foundry coverage report runs cleanly. Branch coverage on
      `collectYield` and any conversion helpers should be near 100%.
- [ ] At least one **fork test** against the real source protocol on Base or
      Base Sepolia.

## Audit

- [ ] External audit complete. Auditor and report attached to the approval
      request.
- [ ] All critical/high findings resolved or accepted-with-justification.
- [ ] If you used [Slither](https://github.com/crytic/slither), the run is
      clean (or the suppressions are documented).

## Operational

- [ ] Adapter deployment script is reproducible (CREATE2 salt or
      forge-script with `--slow` and an env-var-driven config). See
      `script/13_DeployAdapters.s.sol` in `takeover-contracts` for the
      reference shape.
- [ ] Adapter address is the same on every chain you intend to support
      (CREATE2). Not strictly required, but expected.
- [ ] You can produce a 1-line "what does this adapter do" sentence and a
      5-line "how the source produces yield" paragraph for the
      [example page](../adapters/examples/flaunch.md).
- [ ] You have a wallet/key story for the adapter's `Ownable` role —
      multisig preferred for anything in production.

## Documentation

- [ ] Example walkthrough written: copy `adapters/examples/flaunch.md`,
      replace the contract-specific bits, add to `SUMMARY.md`.
- [ ] Any new pattern (push-only, multi-source, novel asset-hash convention)
      cross-linked from `adapters/building-an-adapter.md`.

When all of the above is green, file the [approval request](adapter-approval-process.md).
