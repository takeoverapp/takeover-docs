# Table of contents

* [Takeover Developer Docs](README.md)

## Agent skills

* [Build with an agent — overview](skills/README.md)
* [Skill: launch a grid](skills/launch-grid.md)
* [Skill: build a yield adapter](skills/build-yield-adapter.md)
* [Skill: build a hook module](skills/build-hook-module.md)

## Concepts

* [What is Takeover](overview/what-is-takeover.md)
* [UGM v2.3](overview/ugm-v2.3.md)
* [The yield adapter model](overview/yield-adapter-model.md)
* [The hook model](overview/hook-model.md)

## Boards

* [Creating a grid](boards/creating-a-grid.md)
* [Sharding boards above 1024 seats](boards/sharding.md)
* [Wiring an adapter to a grid](boards/wiring-an-adapter.md)
* [Multi-asset grids](boards/multi-asset-grids.md)

## Yield adapters

* [Adapter overview](adapters/overview.md)
* [The IYieldAdapter interface](adapters/interface.md)
* [Adapter lifecycle](adapters/lifecycle.md)
* [Building a yield adapter](adapters/building-an-adapter.md)
* [Yield token rules](adapters/yield-token-rules.md)
* [Asset hashes](adapters/asset-hash.md)
* [Testing your adapter](adapters/testing.md)
* [Security considerations](adapters/security.md)
* Examples
  * [Flaunch](adapters/examples/flaunch.md)
  * [Uniswap V3 LP NFTs](adapters/examples/uniswap-v3.md)
  * [Uniswap V4 LP NFTs](adapters/examples/uniswap-v4.md)
  * [Protocol fees](adapters/examples/protocol-fees.md)

## Hook modules

* [The IGridHooksV23 interface](hooks/interface.md)
* [Hook lifecycle and gas budgets](hooks/lifecycle.md)
* [Registration and the two-stage allowlist](hooks/registration.md)
* [Two-axis pause + per-module disable](hooks/pause-flags.md)
* [moduleTransferSeat: forced reassignment](hooks/module-transfer-seat.md)
* Examples
  * [Whitelist module](hooks/examples/whitelist-module.md)
  * [Anti-snipe module](hooks/examples/anti-snipe.md)

## Reference

* [UGM v2.3 API](reference/ugm-api.md)
* [IGrid](reference/igrid.md)
* [IFeeReceiver](reference/ifeereceiver.md)
* [Events](reference/events.md)
* [Deployments](reference/deployments.md)
* [Glossary](reference/glossary.md)

## Submit

* [Pre-submission checklist](submit/checklist.md)
* [Audit expectations](submit/audit-expectations.md)
* [Adapter approval process](submit/adapter-approval-process.md)

## Whitepaper

* [Whitepaper](whitepaper.md)
