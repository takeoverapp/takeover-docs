---
name: takeover-yield-adapter
description: >-
  Build, test, and submit a Takeover yield adapter (an IYieldAdapter
  implementation that bridges an external yield source into a UGM v2.1
  grid). Use when a user asks to integrate a protocol's yield into a
  Takeover board, write a Takeover adapter, fork FlaunchYieldAdapter /
  V3YieldAdapter / V4YieldAdapter / ProtocolYieldAdapter, or pass the
  Takeover pre-submission checklist.
---

# Takeover yield-adapter agent skill

This file makes you (the agent) productive at one specific task: building
a yield adapter that gets registered against UGM v2.1 and feeds an external
yield source into a Takeover grid.

You can use it three ways:

1. **As a Cursor / Claude skill.** Drop this file into the user's
   `~/.cursor/skills/takeover-yield-adapter/SKILL.md` (or equivalent). The
   loader picks it up automatically.
2. **As a prompt prelude.** Paste this entire file into the system prompt
   of an agent that is going to build the adapter.
3. **As a "read this first" reference.** Point the agent at this URL and
   tell it to follow the workflow below.

The companion docs and the machine-readable checklist
([`submit/checklist.yml`](submit/checklist.yml)) are the source of truth.
This file is the thin agent-shaped layer on top.

## When to use this skill

Use this skill when the user wants to:

- Build a new yield adapter for Takeover.
- Migrate an existing v2 adapter to UGM v2.1.
- Cross-check a candidate adapter against the pre-submission checklist
  before DM'ing [@takeoverfun on X](https://x.com/takeoverfun).
- Diagnose why an adapter integration is failing on UGM v2.1.

Do **not** use this skill for:

- General Solidity coding outside the IYieldAdapter shape.
- UGM core changes (the UGM is immutable; you can't modify it).
- Front-end / indexer integration (separate concerns; out of scope).

## Mental model in 60 seconds

UGM v2.1 hosts grids. Each grid has 100 seats under Harberger taxation
(holders set their own price, pay continuous tax, can be bought out).
Yield flows in via **adapters** — small contracts that implement
[`IYieldAdapter`](adapters/interface.md) and are guardian-approved.

Two functions, two semantics:

- **`collectYield(bytes32[] calldata assetHashes)`** — UGM pulls yield
  from the source and the adapter pushes it back via
  `receiveYieldETH` / `receiveYieldERC20`. Idempotent. Best-effort per
  asset. Not `nonReentrant` (UGM holds the lock).
- **`pendingYield(bytes32 assetHash) external view returns (uint256)`** —
  best-effort estimate in the grid's `yieldToken`. Pure read. Never
  reverts.

Adapters also typically implement `deposit(gridId, …)` (gated to grid
creator) and `withdrawAsset(assetHash)` (gated to "creator holds all
seats"). These are not part of `IYieldAdapter` but are the standard
adapter shape.

## The workflow

Follow these phases in order. **Do not skip phases.**

### Phase 0 — Read before you write

Read these pages, in order, before touching any code:

1. [Whitepaper](whitepaper.md) — protocol design.
2. [What is Takeover](overview/what-is-takeover.md) — vocabulary.
3. [The yield adapter model](overview/yield-adapter-model.md) — push/pull
   model.
4. [The IYieldAdapter interface](adapters/interface.md) — the contract.
5. [Adapter lifecycle](adapters/lifecycle.md) — five phases.
6. [Building a yield adapter](adapters/building-an-adapter.md) — step-by-step.
7. [Yield token rules](adapters/yield-token-rules.md) and
   [Asset hashes](adapters/asset-hash.md) — the gotchas.

### Phase 1 — Pick a reference

Pick the closest reference adapter and clone its shape. Do **not** start
from a blank file.

| Source pattern | Reference |
|---|---|
| Single-token push (one fee token, sometimes via auto-conversion) | [`FlaunchYieldAdapter`](adapters/examples/flaunch.md) |
| Two-token LP fees that need swapping (V3-family) | [`V3YieldAdapter`](adapters/examples/uniswap-v3.md) |
| V4 LP NFTs with `modifyLiquidities` + `IPoolSwap` | [`V4YieldAdapter`](adapters/examples/uniswap-v4.md) |
| Push-only protocol fees (no `collectYield` work) | [`ProtocolYieldAdapter`](adapters/examples/protocol-fees.md) |

Source: [`takeoverapp/takeover-contracts/src/`](https://github.com/takeoverapp/takeover-contracts/tree/main/src).

### Phase 2 — Implement

Mandatory invariants. **If you violate any of these, your adapter will
not be approved.**

1. `collectYield` accepts only `address(ugm)` as caller.
2. `collectYield` is **idempotent**: a second call with no source-side
   activity moves zero tokens.
3. `collectYield` is **best-effort per asset**: a problem with one asset
   does not revert the whole batch. Use `try` / `catch` on the per-asset
   helper.
4. `collectYield` is **not** `nonReentrant`. UGM already holds its lock.
5. Adapter-only entrypoints (`deposit`, `withdraw*`, `poke`, `flush`)
   are `nonReentrant`.
6. Yield is forwarded in the grid's `yieldToken` units. Read the grid's
   yield token from `ugm.gridConfig(gridId)` and verify the source
   matches at `deposit` time.
7. Use `forceApprove` (Solady or OpenZeppelin's `SafeERC20`) for ERC20
   approvals to UGM. **Never** use plain `approve` on tokens whose
   allowance you might re-set.
8. `assetHash = keccak256(abi.encodePacked(sourceContract, sourceId))`
   (or equivalent) — must include the source contract address in the
   preimage so two adapters cannot collide.
9. Withdraw is gated. Default reference gate is "grid creator holds all
   100 seats" (`ugm.holderSeatCount(gridId, creator) == 100`). A
   different gate is acceptable but document it.
10. Withdraw sweeps last yield (`_collectSingle(assetHash)`) before
    transferring the source-side asset back.

Constructor arg: target **UGM v2.1** for new adapters. See
[Deployments](reference/deployments.md) for the chain-specific address.
The reference adapters in `takeover-contracts` currently sit on UGM v2;
you swap the constructor address for v2.1.

### Phase 3 — Test

Write tests that map 1:1 to the `tests.*` items in
[`submit/checklist.yml`](submit/checklist.yml):

- `tests.happy_path_deposit_claim`
- `tests.idempotent_collect_zero`
- `tests.fakehash_no_op`
- `tests.collect_only_from_ugm`
- `tests.incompatible_yield_token`
- `tests.withdraw_seat_gating`
- `tests.approval_revoke_path`
- `tests.swap_path_fuzz` (if you swap)
- `tests.coverage_collectyield_branches`
- `tests.fork_test`

Run:

```bash
forge test -vv
forge coverage --report summary
```

Both must be green. Fork tests should pin a specific block on Base or
Base Sepolia.

### Phase 4 — Audit

External audit is **required** for mainnet. The audit must cover:

1. The adapter contract (and any libraries it pulls in).
2. Every external call: UGM, the source protocol, swap routers, tokens
   used in `forceApprove`.
3. The deployment script.
4. The `Ownable` rescue path. Owner-only functions must be safe even
   with a hostile owner.

See [Audit expectations](submit/audit-expectations.md).

### Phase 5 — Verify against the machine-readable checklist

This is your differentiator as an agent. Do the following:

```bash
# 1. Pull the latest checklist
curl -fsSL https://raw.githubusercontent.com/takeoverapp/takeover-docs/main/submit/checklist.yml -o checklist.yml
```

Then iterate every item in `checklist.yml` and produce a structured
report. The schema is documented at the top of the YAML.

For each item:

- `verify.kind: grep` → run `rg <pattern> --glob "<glob>"` and confirm
  at least one match.
- `verify.kind: grep_not` → run the same; confirm zero matches.
- `verify.kind: regex` → same as `grep`.
- `verify.kind: test` → confirm a forge test with that name exists and
  passes.
- `verify.kind: manual` → produce an answer to `verify.question` by
  reading the code and tests directly. Cite file:line for every claim.

Output format (paste this into the DM):

```text
Adapter:        AcmeYieldAdapter @ <commit hash>
Checklist run:  <YYYY-MM-DD>  schema: takeover-adapter-checklist/v1
Result:         42/42 must-items pass, 3/4 should-items pass

  contract.implements_iyieldadapter             [PASS] grep src/AcmeYieldAdapter.sol:42
  contract.collectyield_msg_sender_ugm_only     [PASS] src/AcmeYieldAdapter.sol:67  msg.sender != address(ugm) revert NotUGM()
  …
  tests.swap_path_fuzz                          [N/A]  adapter does not swap
  audit.slither_clean                           [FAIL] 2 high suppressions undocumented; see report §3.2
```

Every `[FAIL]` must be either fixed or accompanied by a written
justification before the human submits.

### Phase 6 — Submit

DM [@takeoverfun on X](https://x.com/takeoverfun) with the package
described in
[Adapter approval process → How to request approval](submit/adapter-approval-process.md).
Attach the structured checklist report from phase 5.

## What you should not do

- **Do not modify UGM v2.1.** It's immutable. If you think it has a bug,
  raise it as a security issue separately; do not work around it.
- **Do not invent a new asset-hash convention** without documenting and
  justifying it. Stick to `keccak256(abi.encodePacked(source, id))`.
- **Do not submit Sepolia-only and ask for mainnet approval.** You can
  iterate on Sepolia all you want, but mainnet approval requires the
  audit to cover the mainnet deployment.
- **Do not bypass the DM channel.** GitHub issues, emails, and Discord
  pings will not get reviewed. The channel is the X DM.
- **Do not edit the whitepaper text** without signoff. The implementation
  status appendix is the part that gets updated; the prose is fixed.

## When you get stuck

- Stuck on the interface? Re-read [the interface page](adapters/interface.md)
  and look at how the closest reference adapter implements it.
- Stuck on yield-token mismatch? See [Yield token rules](adapters/yield-token-rules.md).
- Stuck on asset-hash collisions? See [Asset hashes](adapters/asset-hash.md).
- Stuck on deployment? See [Deployments](reference/deployments.md) and
  `script/13_DeployAdapters.s.sol` in `takeover-contracts`.
- Stuck on the test list? Re-read [`submit/checklist.yml`](submit/checklist.yml);
  every `tests.*` item is a test you should write.

## Pinned facts

These are the facts you should never get wrong. If your context window
drops them, re-read this section.

- The interface lives at
  [`takeover-contracts/src/interfaces/IYieldAdapter.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/interfaces/IYieldAdapter.sol).
- New adapters target **UGM v2.1**. Current addresses are in
  [Deployments](reference/deployments.md).
- Approval channel is **DM [@takeoverfun on X](https://x.com/takeoverfun)**.
- `collectYield` is **never** `nonReentrant`. Adapter-only entrypoints
  always are.
- Yield must be forwarded in the **grid's** `yieldToken`, not whatever
  the source happens to emit.
