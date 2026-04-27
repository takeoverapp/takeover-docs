# Deployments

Canonical contract addresses for UGM v2.1 and the reference yield adapters.

> **Maintenance note:** addresses below are pinned to the production
> deployment of UGM v2.1. When a new adapter version ships, the new address
> is added and the old one is annotated `revoked` rather than removed —
> existing assets keep working under a revoked adapter.

## Base mainnet

| Contract | Address | Notes |
|---|---|---|
| `UnifiedGridManagerV21` | <!-- TODO: V2_UGM_ADDRESS --> | UGM v2.1 |
| `FlaunchYieldAdapter` | <!-- TODO: V2_FLAUNCH_YIELD_ADAPTER_ADDRESS --> | flETH yield, single-token push |
| `V3YieldAdapter` | <!-- TODO: V2_V3_YIELD_ADAPTER_ADDRESS --> | Uniswap V3 / Aerodrome / Pancake V3 LP NFTs |
| `V4YieldAdapter` | <!-- TODO: V2_V4_YIELD_ADAPTER_ADDRESS --> | Uniswap V4 LP NFTs |
| `ProtocolYieldAdapter` | <!-- TODO: V2_PROTOCOL_YIELD_ADAPTER_ADDRESS --> | USDC protocol-fee push-only |
| `GridLaunchZap` | <!-- TODO: V2_GRID_LAUNCH_ZAP_ADDRESS --> | Atomic launch + deposit + seat pre-buys |

## Base Sepolia

| Contract | Address | Notes |
|---|---|---|
| `UnifiedGridManagerV21` | <!-- TODO --> | |
| `FlaunchYieldAdapter` | <!-- TODO --> | |
| `V3YieldAdapter` | <!-- TODO --> | |
| `V4YieldAdapter` | <!-- TODO --> | |
| `ProtocolYieldAdapter` | <!-- TODO --> | |
| `GridLaunchZap` | <!-- TODO --> | |

## Source-protocol references

Addresses your adapter likely needs in its constructor.

### Base mainnet

| Address | Used by |
|---|---|
| `flETH` | `FlaunchYieldAdapter` |
| `WETH` | `V3YieldAdapter`, `V4YieldAdapter` |
| Uniswap V3 `SwapRouter02` | `V3YieldAdapter` |
| Uniswap V4 `PoolManager` | `V4YieldAdapter` |
| Uniswap V4 `PositionManager` | `V4YieldAdapter` |
| Flaunch `FeeEscrowRegistry` | `FlaunchYieldAdapter` |
| Takeover `PoolSwap` | `V4YieldAdapter`, `TakeoverBuyback` |

> Concrete addresses for each row above are pulled from the same env
> manifest deploy scripts use. They will be filled in once the docs are
> wired into the deploy pipeline.

## Asset hashes (reference convention)

| Source | Hash preimage |
|---|---|
| Flaunch | `keccak256(abi.encodePacked(flaunch, tokenId))` |
| Uniswap V3 / Aerodrome / Pancake V3 | `keccak256(abi.encodePacked(positionManager, tokenId))` |
| Uniswap V4 | `keccak256(abi.encodePacked(v4PositionManager, tokenId))` |
| `ProtocolYieldAdapter` | One hash per deployment, registered manually via `registerAsset` against the hvTAKEOVER grid |

See [Asset hashes](../adapters/asset-hash.md) for the rules new adapters
follow.

## How to update this page

Source of truth: the deploy logs in
[`takeoverapp/takeover-contracts/broadcast/`](https://github.com/takeoverapp/takeover-contracts/tree/main/broadcast).
When a new deploy script lands:

1. Pull the new address from the broadcast JSON.
2. Update the table above.
3. Annotate any address being replaced as `revoked` (don't delete it —
   `withdrawAsset` from a revoked adapter is the documented unwind path).
4. PR with `deployments` in the title.
