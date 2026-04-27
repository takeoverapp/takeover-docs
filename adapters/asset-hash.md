# Asset hashes

> **Tier 2 placeholder.** Conventions and uniqueness invariants for
> `assetHash`, the only handle UGM has on whatever your adapter wraps.

## Quick rules

- **Globally unique.** UGM reverts `AssetAlreadyRegistered` if the same hash
  is registered twice — across grids, across adapters, ever.
- **Reproducible offchain.** UI and indexer code reconstruct the hash from
  source-side identifiers. Pick a preimage that's available without
  read-only state.
- **Includes the source contract address.** Two distinct position-manager
  deployments must hash differently even for the same `tokenId`.

## Reference patterns

| Adapter | Preimage |
|---|---|
| `FlaunchYieldAdapter` | `keccak256(abi.encodePacked(flaunch, tokenId))` |
| `V3YieldAdapter` | `keccak256(abi.encodePacked(positionManager, tokenId))` |
| `V4YieldAdapter` | `keccak256(abi.encodePacked(v4PositionManager, tokenId))` |
| `ProtocolYieldAdapter` | One hash per deployment; registered manually |

This page will be expanded with anti-patterns (don't hash the grid id, don't
hash a holder address, etc.) and the rationale for the address-in-preimage
rule.
