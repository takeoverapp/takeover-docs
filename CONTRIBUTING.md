# Contributing

Thanks for opening a doc PR. A few conventions to keep this readable:

## Workflow

1. Branch from `main`.
2. **One topic per PR.** A new adapter walkthrough, a fix, a clarification —
   any one of those, but not all three at once.
3. Run the linters locally before pushing:
   ```bash
   npx markdownlint-cli2 "**/*.md"
   npx lychee --no-progress --offline "**/*.md"
   ```
4. Open a PR. Fill the template. Tag a reviewer from the Takeover protocol team.

## Style

- Write for a protocol engineer who has never used Takeover. Don't assume they
  know what a grid, a board, or `assetHash` is — link to the glossary the first
  time each shows up on a page.
- Keep sentences short. One idea per sentence.
- Code samples should compile. If a sample is incomplete on purpose, mark it
  `// pseudocode` so readers don't try to copy-paste it.
- Prefer linking to a real reference adapter in `takeover-contracts` over
  inline pseudocode. When you do inline a snippet, end it with the source path
  and a pinned commit:
  ```text
  > Source: takeover-contracts/src/V4YieldAdapter.sol @ commit abc1234
  ```
- Diagrams: mermaid wherever it works. GitBook renders mermaid natively.
- Headings: sentence case. `## Building a yield adapter`, not
  `## Building A Yield Adapter`.

## What lives here vs not

- **Here:** anything an external integrator needs to read to ship a working
  yield adapter, plus the surfaces they call (UGM, IGrid, IFeeReceiver).
- **Not here:** end-user product docs (UI flows, "how to claim a seat"),
  internal protocol design notes, audit reports, tokenomics. Those live in
  product surfaces or `takeover-contracts/`.

## Adding a new adapter walkthrough

1. Land the contract in [`takeoverapp/takeover-contracts`](https://github.com/takeoverapp/takeover-contracts)
   first. Walkthroughs always trail the source.
2. Copy `adapters/examples/flaunch.md` as your template.
3. Update `SUMMARY.md` so it shows up in the sidebar.
4. Cross-link from `adapters/building-an-adapter.md` if it illustrates a new
   pattern (push-only, swap-and-forward, multi-source, etc.).
