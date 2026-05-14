---
name: takeover-builder-skills
description: >-
  Landing card for Takeover's task-focused agent skills. Pick the
  skill that matches what the user is shipping: launching a grid
  (incl. sharded launches above 1024 seats), building a yield
  adapter, or building a hook module that plugs game / DAO / market
  policy into UGM v2.3. Use this file when an agent is asked to
  "build something on Takeover" without further context.
---

# Build with an agent

Takeover ships three task-focused skills. Each is a short, machine-
readable playbook for one job. Pick the one that matches the user's
intent and follow it end-to-end — do not improvise around the
checklist references.

| User intent | Skill |
| --- | --- |
| Stand up a board (atomic, chunked, or sharded). | [Launch a grid](launch-grid.md) |
| Plug an external yield source into a grid. | [Build a yield adapter](build-yield-adapter.md) |
| Add policy logic — gating, forced transfers, weighted yield, anti-snipe — to a grid. | [Build a hook module](build-hook-module.md) |

If you're not sure which skill applies:

- **Did the user say "create a board / grid / map / land"?** → Launch
  a grid.
- **Did the user say "yield / fees / LP / Flaunch / Uniswap"?** →
  Build a yield adapter.
- **Did the user say "module / hook / governance / whitelist /
  game / conquest / forced transfer"?** → Build a hook module.

You may need more than one. A "build a game on Takeover" request
typically pulls Launch a grid + Build a hook module; "ship our
token's revenue share" pulls Build a yield adapter only.

## Pinned facts (every skill assumes these)

- Target version is **UGM v2.3**. There is no longer a v2.2 surface
  in the docs — every example targets v2.3 unless explicitly noted.
- Mainnet UGM v2.3 lives at
  [`0xac02F47a3E4451f96c715313C8894c3041413F3A`](https://basescan.org/address/0xac02F47a3E4451f96c715313C8894c3041413F3A);
  Base Sepolia at
  [`0xf67159948Dde0dA920Ba465Fca9D63f0c6EFD10C`](https://sepolia.basescan.org/address/0xf67159948Dde0dA920Ba465Fca9D63f0c6EFD10C).
  Always cross-reference [Deployments](../reference/deployments.md)
  before a write.
- Grid sizes: `MIN_TOTAL_SEATS = 4`,
  `MAX_TOTAL_SEATS = 4096`,
  `MAX_SINGLE_TX_CREATE_TOTAL_SEATS = 1024`. Above 1024 use chunked
  creation; above 4096 use SDK-side multi-grid sharding (see
  [Sharding](../boards/sharding.md)).
- Gas budgets the contracts enforce on YOU:
  - `ADAPTER_COLLECT_GAS_CAP = 600_000` (yield adapters'
    `collectYield`).
  - `GOVERNANCE_HOOK_GAS_CAP = 150_000` (every hook callback).
- All approvals are guardian-gated. Get an adapter / module on
  mainnet via the [Adapter approval process](../submit/adapter-approval-process.md).
- The submission DM channel is
  [@takeoverfun on X](https://x.com/takeoverfun). GitHub issues
  and Discord pings do not get reviewed.

## How an agent should use these skills

1. Read this file first.
2. Pick the skill from the table above.
3. Follow the skill's phases in order. Each skill has a "phase 0
   — read before you write" pointer to the concept docs that ground
   the work; do not skip phase 0.
4. Verify against [`submit/checklist.yml`](../submit/checklist.yml)
   (or the new `hook.*` / `grid_launch.*` categories that ship
   alongside this rewrite) before submitting.
5. DM [@takeoverfun on X](https://x.com/takeoverfun) with the
   structured checklist report.
