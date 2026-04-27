# 📖 What is Takeover

Takeover is a protocol for **fractional, continuously contestable ownership of
onchain yield**. Each yield-producing asset (a Uniswap LP, a Flaunch memecoin,
a stream of protocol fees, anything else) gets wrapped into a **grid** — a
fixed set of seats, each entitled to an equal share of that grid's yield.

Seats are sold under [Harberger taxation](https://en.wikipedia.org/wiki/Harberger_Tax):

- The holder sets a self-assessed **price** in the grid's tax token.
- The holder pays a continuous **tax** on that price (5% per week by default,
  range 1–10%).
- Anyone can **buy out** the seat at the posted price at any time.

That tax + buyout pressure makes seats price themselves toward their true
yield value, and rotates ownership toward whoever values the seat most.

## Boards

In product language, a grid is a **board**. The two terms are used
interchangeably in these docs:

- **Grid** when we're talking about the onchain object (`gridId`,
  `gridConfig`, etc.).
- **Board** when we're talking about the user-facing concept ("the Boardroom",
  "the USDC board", etc.).

Every board today is hosted on a single contract: the **Unified Grid Manager
v2.1** (UGM v2.1). One contract, many grids — each grid is a row in UGM's
storage.

## Why this matters for integrators

If your protocol produces yield (LP fees, lending interest, MEV rebates,
anything that streams a token to a holder), you can wrap it as a Takeover
board. Doing so lets:

- Anyone buy a seat and earn 1/N of your protocol's yield without holding the
  underlying asset.
- The market discover what your yield stream is actually worth, with the
  Harberger tax going back to your protocol (or wherever you direct it).
- Your protocol's reach extend into the Takeover ecosystem: the Boardroom
  redistributes protocol-wide tax revenue to its own seat holders, so being
  on Takeover plugs you into a network of holders who already pay attention.

The bridge between **your protocol** and **a Takeover board** is a **yield
adapter**: a small, audited contract that knows how to pull yield out of your
protocol and push it into UGM. That adapter is what these docs are about.

## Where to next

- [The yield adapter model](yield-adapter-model.md) — the mental model for how
  yield flows from your protocol to seat holders.
- [Building a yield adapter](../adapters/building-an-adapter.md) — the
  step-by-step.
- [Glossary](../reference/glossary.md) — every Takeover-specific term in one
  place.
