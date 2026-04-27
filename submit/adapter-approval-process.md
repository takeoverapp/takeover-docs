# Adapter approval process

UGM v2.1 won't let an adapter touch a grid until the Takeover guardian flips
`approvedAdapters[adapter] = true`. This page is how you request that flip.

> **Status:** the exact submission channel (Discord form vs. GitHub issue
> template vs. on-chain registry) is being finalised. Treat the URLs and
> handles below as canonical, but expect minor changes through the rest of
> 2026.

## Who approves

The **guardian** is the role on UGM v2.1 that calls
`setApprovedAdapter`. Today the guardian is a Takeover protocol multisig.

Approval is **global**: a flip means the adapter can register assets on any
grid. There is no per-grid approval.

## What we look at

In order of weight:

1. **External audit report.** Required for anything intended for mainnet.
   The reviewer will read it.
2. **Pre-submission checklist** ([here](checklist.md)) — every box ticked.
3. **Tests** — coverage and fork tests in `takeoverapp/takeover-contracts`
   (preferred) or your own repo with a clear pointer to the relevant
   commit.
4. **Source-side risk** — what happens to the adapter if the source protocol
   pauses, upgrades, or rugs? An adapter that can leave seat holders
   permanently un-claimable is a non-starter.
5. **Operational story** — who owns the adapter's `Ownable` role, what's
   the rescue plan, are deployments reproducible.

## How to request approval

> **Pending product sign-off:** confirm the channel below before publishing
> these docs externally.

The current expected flow is:

1. **Open an issue** in [`takeoverapp/takeover-contracts`](https://github.com/takeoverapp/takeover-contracts)
   with the `adapter-approval` label. Use the template (TBD).
2. Include:
   - Adapter source URL + commit hash.
   - Audit report URL.
   - Deployed addresses on Base Sepolia and (if applicable) Base.
   - The pre-submission checklist with each item linked to the proof
     (test name, audit page, etc.).
   - One-paragraph description of the source protocol and what the adapter
     does.
3. A protocol team reviewer is assigned within 3 business days.
4. Once reviewer approval is recorded on the issue, the guardian schedules
   a multisig transaction. Expect 1–2 weeks from filing to flip on mainnet,
   faster on Sepolia.
5. The guardian transaction emits `ApprovedAdapterUpdated(adapter, true)`.
   You're live.

## After approval

Once approved, a few things happen automatically:

- The Takeover indexer picks up `AssetRegistered` events from your adapter
  and starts surfacing per-asset accruals on `Grid.assets`.
- Your adapter shows up in the docs (Tier 2 polish: file a PR adding an
  [example walkthrough](../adapters/examples/flaunch.md)).
- Your adapter gets listed on [Deployments](../reference/deployments.md).

## Revocation

The guardian can revoke approval at any time with
`setApprovedAdapter(adapter, false)`. Reasons we'd actually do that:

- A new audit finding that you can't fix without a redeploy.
- The source protocol changes its fee mechanics in a way that breaks the
  adapter's accounting.
- Operational compromise of the adapter owner key.

Revocation only blocks **new** `registerAsset` calls. Existing assets keep
working — their yield can still be pushed and they can still be withdrawn.
This is intentional: revocation has to be safe to use.

## What happens if we say no

You'll get a written explanation on the issue. The most common rejection
reasons:

- "The audit didn't cover the swap path."
- "Withdraw can strand seat holders" (your unwind condition isn't safe).
- "`collectYield` reverts on a single bad asset" (no per-asset try/catch).
- "Yield can be re-routed by the adapter owner without seat-holder
  consent."

These are usually fixable; you re-submit when the change lands.

## Where to next

- [Pre-submission checklist](checklist.md).
- [Audit expectations](audit-expectations.md) — what scope auditors should
  cover.
