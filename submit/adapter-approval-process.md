# 🛡 Adapter approval process

UGM v2.1 won't let an adapter touch a grid until the Takeover guardian flips
`approvedAdapters[adapter] = true`. This page is how you request that flip.

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

**DM [@takeoverfun on X](https://x.com/takeoverfun).** That's the channel.
No GitHub issue, no email form — direct message.

Open the DM with a single message that contains all of the following.
Reviewers will not chase missing info; an incomplete submission gets a
"please re-send with everything" reply and goes to the back of the queue.

### What to include in the DM

```text
Adapter:           <name, e.g. "AcmeYieldAdapter">
Source protocol:   <one line: "Acme V3 LP NFT fees on Base">
Source URL:        <github link, pinned to commit hash>
Commit hash:       <40-char hash>
Audit report:      <public URL or attached PDF>
Auditor:           <firm name>
Sepolia deploy:    0x…  (verified on Sepolia Basescan)
Mainnet deploy:    0x…  (or "not yet" if Sepolia-only)
Owner address:     0x…  (multisig preferred)
Yield token:       0x… symbol  (must match target grids)
Asset-hash rule:   keccak256(abi.encodePacked(<source>, <id>))
Pre-submission checklist: <link to a gist or repo file with each box ticked
                          and a per-line proof: test name, audit page, etc.>
Two-sentence pitch: what does the adapter do and why does it matter?
```

If your DM exceeds X's character limit, attach a single PDF or paste a gist.
Don't split across multiple DMs.

### What happens next

1. A reviewer responds within **3 business days** to acknowledge receipt.
2. Review takes **1–2 weeks** for mainnet, faster for Sepolia-only. The
   reviewer will reply in the same DM thread with either:
   - a list of concrete blockers, or
   - a "scheduled" message confirming the multisig is queueing
     `setApprovedAdapter(adapter, true)`.
3. The guardian transaction emits `ApprovedAdapterUpdated(adapter, true)`.
   You're live — register your first asset against any grid you have
   creator rights on.

### Why a DM and not a public issue?

Two reasons. (1) A lot of the audit context is sensitive until the adapter
is approved — funded exploits on a pre-approval adapter would be bad for
both sides. (2) Approval signal is onchain (`ApprovedAdapterUpdated`), so
the audit trail is already public where it matters.

If you want a public artifact for your own users, file a PR to this docs
repo adding an [example walkthrough](../adapters/examples/flaunch.md) once
you're approved.

## After approval

Once approved, a few things happen automatically:

- The Takeover indexer picks up `AssetRegistered` events from your adapter
  and starts surfacing per-asset accruals on `Grid.assets`.
- Your adapter shows up in the docs (Tier 2 polish: file a PR adding an
  [example walkthrough](../adapters/examples/flaunch.md)).
- Your adapter gets listed on [Deployments](../reference/deployments.md) —
  open a PR with the new addresses.

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

You'll get a written explanation in the DM thread. The most common
rejection reasons:

- "The audit didn't cover the swap path."
- "Withdraw can strand seat holders" (your unwind condition isn't safe).
- "`collectYield` reverts on a single bad asset" (no per-asset try/catch).
- "Yield can be re-routed by the adapter owner without seat-holder
  consent."

These are usually fixable; re-DM when the change lands and reference the
prior thread.

## Where to next

- [Pre-submission checklist](checklist.md).
- [Audit expectations](audit-expectations.md) — what scope auditors should
  cover.
