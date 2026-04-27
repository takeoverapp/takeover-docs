# Audit expectations

> **Tier 2 placeholder.** What auditors should cover when reviewing a yield
> adapter for Takeover guardian approval.

## Scope (minimum)

An audit accepted by the guardian covers, at minimum:

1. **The adapter contract itself**, including any libraries it pulls in.
2. **Every external call from the adapter:** UGM, the source protocol,
   any swap router, any token used in `forceApprove`.
3. **The deployment script** if it's the one used in production.
4. **The `Ownable` rescue path.** Owner-only functions need to be safe
   even with a hostile owner — at minimum, they shouldn't be able to
   re-route registered assets or steal pending yield.

## Out of scope (not your problem)

- UGM v2.1 itself — already audited.
- The source protocol — Takeover treats it as a black box.
- The Takeover indexer — separate review.

## What we read first in the report

- **All critical/high findings.** Every one has to be either fixed or
  accepted-with-justification in the approval DM thread.
- **Centralisation findings.** Anything that lets the adapter owner
  unilaterally change yield routing is a likely block.
- **Reentrancy + approval-race findings.** If the auditor flagged either,
  the fix needs to be visible in the diff.

This page will be expanded with the full reviewer checklist and a sample
acceptance template.
