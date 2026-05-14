# Pre-submission checklist

Before requesting [guardian approval](adapter-approval-process.md) for a
new yield adapter or hook module, walk this list. Anything missing here
is going to come back as a review comment.

> **Building with an AI agent?** Every item here also lives in
> [`submit/checklist.yml`](checklist.yml) with a structured `verify` hint
> per item (ripgrep patterns, named tests, manual questions). Point your
> agent at the YAML and ask it to produce a pass/fail row per `id`, then
> paste the result into your DM to [@takeoverfun](https://x.com/takeoverfun)
> alongside the prose responses. See the
> [Agent skills](../skills/README.md) section for the full workflow.

## If you're shipping a yield adapter

### Contract

- [ ] Implements [`IYieldAdapter`](../adapters/interface.md): `collectYield`
      and `pendingYield`.
- [ ] Constructor points at **UGM v2.3** for the target network.
- [ ] `collectYield` reverts if `msg.sender != address(ugm)`.
- [ ] `collectYield` is **idempotent**: two calls in a row with no source-
      side activity move zero tokens.
- [ ] `collectYield` is **best-effort per asset**: a problem with one asset
      doesn't revert the whole batch.
- [ ] `collectYield` stays under `ADAPTER_COLLECT_GAS_CAP = 600,000`.
      Profiled with a worst-case fork test.
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

### Tests

- [ ] Happy-path deposit + `claimFees` → seat balance moves.
- [ ] Idempotent collect: second `claimFees` in a row collects zero.
- [ ] `collectYield([fakeHash])` is a no-op, not a revert.
- [ ] `collectYield` from a non-UGM caller reverts.
- [ ] Deposit on a grid with incompatible `yieldToken` reverts.
- [ ] Withdraw without all seats reverts; with all seats succeeds.
- [ ] **Gas budget test:** `collectYield` over a worst-case batch runs
      under 600k gas.
- [ ] Approval revoke path: after `setApprovedAdapter(adapter, false)`,
      deposit reverts but residual yield still pushes and `withdrawAsset`
      still works for already-registered assets.
- [ ] If your adapter swaps, fuzz the swap path on edge prices (very low /
      very high `sqrtPriceX96`).
- [ ] Foundry coverage report runs cleanly. Branch coverage on
      `collectYield` and any conversion helpers should be near 100%.
- [ ] At least one **fork test** against the real source protocol on Base
      or Base Sepolia.

## If you're shipping a hook module

### Contract

- [ ] Implements [`IGridHooksV23`](../hooks/interface.md) — or, if you only
      need the v2.2 surface, `IGridGovernanceHooks` (UGM v2.3 will accept
      either).
- [ ] Every callback that participates in policy returns within
      `GOVERNANCE_HOOK_GAS_CAP = 150,000` gas, including the worst-case
      cold-cache path.
- [ ] Gating callbacks (`beforeClaim`, `beforeBuyout`, `beforePriceChange`)
      revert with a custom error or string. **No bare `revert()`** — UGM
      treats empty revert data as "didn't implement" and silently no-ops.
- [ ] Observer callbacks (`onForfeit`, `yieldWeight`) tolerate UGM
      swallowing reverts: any state change inside them must be either
      idempotent or skipped under failure.
- [ ] State is keyed by `(gridId, seatId)` (or `gridId` alone where
      appropriate) so a single module instance can serve multiple grids
      without bleed-over.
- [ ] If the module also calls `moduleTransferSeat`, it documents the
      authorisation flow it uses to determine when forced transfers are
      legitimate.
- [ ] Owner-only admin functions are gated; module bookkeeping events are
      emitted on every state change.
- [ ] No unbounded loops in callbacks. Iteration over seats / holders is
      a soft-fail risk under heavy grids.
- [ ] `IHookDescriptor` is implemented (`hookKind`, `hookVersion`,
      `supportedCallbacks`, `description`) so builder UIs can render
      accurate metadata.

### Tests

- [ ] Happy-path: gated buyout / claim / price change reverts when policy
      forbids; passes when allowed.
- [ ] Multi-grid invariant: same module instance attached to two grids,
      state changes on grid A don't bleed into grid B.
- [ ] Gas budget: each callback under 150k gas in worst-case cold-cache
      conditions.
- [ ] Empty-revert avoidance: `beforeClaim` etc. always revert with data
      when blocking. Confirm with a foundry test asserting `vm.expectRevert(SomeError.selector)`.
- [ ] Forfeit observer: `onForfeit` does not revert in any forge fork
      test of the forfeit path.
- [ ] If the module calls `moduleTransferSeat`: cooldown-pause fork tests
      asserting `pauseGridModules`, `setModuleDisabled`, and
      `setApprovedModule(false)` each individually block forced transfers.

## Audit (both paths)

- [ ] External audit complete. Auditor and report attached to the
      approval request.
- [ ] All critical/high findings resolved or accepted-with-justification.
- [ ] If you used [Slither](https://github.com/crytic/slither) and/or
      [Semgrep](https://semgrep.dev), the runs are clean (or the
      suppressions are documented).

## Operational (both paths)

- [ ] Deployment script is reproducible (CREATE2 salt or forge-script
      with `--slow` and an env-var-driven config). See
      `script/13_DeployAdapters.s.sol` in `takeover-contracts` for the
      adapter reference shape.
- [ ] You can produce a 1-line "what does this contract do" sentence and
      a 5-line description for the example page.
- [ ] You have a wallet/key story for the contract's `Ownable` role —
      multisig preferred for anything in production.

## Documentation (both paths)

- [ ] Example walkthrough written: for adapters, copy
      [`adapters/examples/flaunch.md`](../adapters/examples/flaunch.md);
      for hooks, copy
      [`hooks/examples/whitelist-module.md`](../hooks/examples/whitelist-module.md).
      Replace contract-specific bits and add to `SUMMARY.md`.
- [ ] Any new pattern (push-only, multi-source, novel asset-hash, novel
      hook surface) cross-linked from the relevant
      `adapters/building-an-adapter.md` or
      `hooks/build-hook-module.md`.

When all of the above is green, file the
[approval request](adapter-approval-process.md). For hook modules, the
same approval flow applies — DM the team with the audit report and the
filled-in checklist.yml output.
