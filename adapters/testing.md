# Testing your adapter

> **Tier 2 placeholder.** Patterns for testing yield adapters with Foundry.

The reference adapters in
[`takeoverapp/takeover-contracts/test/`](https://github.com/takeoverapp/takeover-contracts/tree/main/test)
cover everything you'll want to copy:

- `FlaunchYieldAdapter.t.sol` — single-token push, `MockUGM` driving
  `collectYield`.
- `V3YieldAdapter.t.sol`, `V4YieldAdapter.t.sol` — two-token swap-and-forward
  with mocked position managers.
- `ProtocolYieldAdapter.t.sol` — push-only via `notifyDeposit` and `flush`.
- `test/mocks/MockUGM.sol` — the canonical UGM mock for adapter unit tests.
- `test/mocks/RevertingAdapter.sol` — useful for testing UGM's tolerance of
  bad adapters in your own integration tests.

This page will be expanded with:

- A from-scratch fork-test template.
- The minimum invariants every adapter test suite should cover.
- Fuzz targets for swap-path adapters (`sqrtPriceX96` extremes, fee
  rounding).
- Coverage expectations.
