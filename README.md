# Takeover Developer Docs

Source for [docs.takeover.fun](https://docs.takeover.fun) — the developer-facing
documentation for [Takeover](https://takeover.fun).

These docs target **UGM v2.3**, the contract that hosts every Takeover
board on Base mainnet and Base Sepolia today.

## Building with an AI agent? Start here

The fastest way to ship something on Takeover is to point an agent at
the [Agent skills](skills/README.md) section and pick the task that
matches your goal:

1. [Skill: launch a grid](skills/launch-grid.md) — atomic, chunked,
   or sharded grid creation.
2. [Skill: build a yield adapter](skills/build-yield-adapter.md) —
   bridge your protocol's yield into UGM v2.3.
3. [Skill: build a hook module](skills/build-hook-module.md) — gate
   seat economics and drive forced transfers via `IGridHooksV23`.

Each skill page is structured for an agent to follow end-to-end:
read-order links, code patterns, validation tables, gas budgets, and
the [machine-readable checklist](submit/checklist.yml).

## Cold start (humans)

If you are a protocol engineer and want your protocol's yield to flow
into a Takeover board (a UGM v2.3 grid), the canonical reading order is:

1. [Whitepaper](whitepaper.md) — protocol design and Harberger
   mechanics.
2. [What is Takeover](overview/what-is-takeover.md) — conceptual
   model.
3. [UGM v2.3](overview/ugm-v2.3.md) — what the live contract does.
4. [The yield adapter model](overview/yield-adapter-model.md) — how
   external yield sources connect.
5. [The hook model](overview/hook-model.md) — how policy modules
   layer on top.
6. [Building a yield adapter](adapters/building-an-adapter.md) — full
   walkthrough.
7. [Submission checklist](submit/checklist.md) — pre-flight before
   guardian approval.

## Repository

This repo is the source of truth for the docs. Markdown lands here;
GitBook syncs from `main` and renders the public site.

```
skills/     # Agent-facing skills (lead the navigation)
overview/   # Concepts and mental model
boards/     # Things grid creators do (atomic, chunked, sharded creation)
adapters/   # IYieldAdapter, lifecycle, how to build one, examples
hooks/      # IGridHooksV23, lifecycle, registration, examples
reference/  # API reference, events, deployments, glossary
submit/     # Checklist + adapter approval process
```

See [`SUMMARY.md`](SUMMARY.md) for the full table of contents.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Short version: branch from
`main`, one topic per PR, run `npx markdownlint-cli2 "**/*.md"` locally
before pushing, link to a reference adapter or hook module in
[`takeoverapp/takeover-contracts`](https://github.com/takeoverapp/takeover-contracts)
whenever you can.

## License

MIT — see [`LICENSE`](LICENSE).
