# Takeover Developer Docs

Source for [docs.takeover.fun](https://docs.takeover.fun) — the developer-facing
documentation for [Takeover](https://takeover.fun).

If you are a protocol engineer and want your protocol's yield to flow into a
Takeover board (a UGM v2.1 grid), start here:

1. [📜 Whitepaper](whitepaper.md) — protocol design and Harberger mechanics
2. [📖 What is Takeover](overview/what-is-takeover.md)
3. [🔌 The yield adapter model](overview/yield-adapter-model.md)
4. [🛠 Building a yield adapter](adapters/building-an-adapter.md)
5. [✅ Submission checklist](submit/checklist.md)

Building with an AI agent? Start at [`SKILL.md`](SKILL.md) — it points the
agent at the right pages and the [machine-readable checklist](submit/checklist.yml).

## Repository

This repo is the source of truth for the docs. Markdown lands here; GitBook
syncs from `main` and renders the public site.

```
overview/   # Concepts and mental model
adapters/   # IYieldAdapter, lifecycle, how to build one, examples
boards/     # Things grid creators do (wire adapters, pick yield tokens)
reference/  # API reference, deployments, glossary
submit/     # Checklist + adapter approval process
```

See [`SUMMARY.md`](SUMMARY.md) for the full table of contents.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Short version: branch from `main`,
one topic per PR, run `npx markdownlint-cli2 "**/*.md"` locally before pushing,
link to a reference adapter in [`takeoverapp/takeover-contracts`](https://github.com/takeoverapp/takeover-contracts)
whenever you can.

## Status

These docs target **UGM v2.1**, the contract that hosts every Takeover board
today. UGM vNext is in flight; pages will be updated (or banner-marked) as it
ships.

## License

MIT — see [`LICENSE`](LICENSE).
