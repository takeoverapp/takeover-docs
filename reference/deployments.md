# Deployments

Canonical contract addresses for Takeover on Base mainnet and Base Sepolia.

> **Target version: UGM v2.3** across both networks. The protocol guardian
> writes new approvals against v2.3; new adapter and hook submissions
> register against v2.3. Older versions remain readable on-chain for
> historical grids but are no longer the integration target.

The single source of truth for these addresses is the SDK preset:

```ts
import { base, sepolia } from '@takeover/sdk';

base.ugmV23;            // 0xF2DfBe1ef26AA82e9438dA95d5cC3007D3031F9e
base.library;           // 0x2b4530d49ccCBdF22005d84C175F552334a61445
base.yieldAdapters.v3;  // 0x0205dA82768Ecd4C6E66f94e5A97F63B8dE1EbCf
sepolia.ugmV23;         // 0xC389D001C627C7fcE23405190d453906598c1607
```

The Builder portal `/builders/contracts` page reads from the same preset
and is interactively switchable between mainnet and Sepolia. Builders
who do not pull `@takeover/sdk` can grab the raw ABIs from
[ABIs](#abis) below or directly from
`https://takeover.fun/abis/<contract>.json`.

## Base mainnet (chain 8453)

### UGM v2.3

| Contract | Address |
|---|---|
| `UnifiedGridManagerV23` | [`0xF2DfBe1ef26AA82e9438dA95d5cC3007D3031F9e`](https://basescan.org/address/0xF2DfBe1ef26AA82e9438dA95d5cC3007D3031F9e) |
| `UGMV23Linked` (linked library) | [`0x2b4530d49ccCBdF22005d84C175F552334a61445`](https://basescan.org/address/0x2b4530d49ccCBdF22005d84C175F552334a61445) |
| `FlaunchYieldAdapter` (v2.3) | [`0x1867650664c252186a214D5ed37CB1FD736b7bE6`](https://basescan.org/address/0x1867650664c252186a214D5ed37CB1FD736b7bE6) |
| `V3YieldAdapter` (v2.3) | [`0x0205dA82768Ecd4C6E66f94e5A97F63B8dE1EbCf`](https://basescan.org/address/0x0205dA82768Ecd4C6E66f94e5A97F63B8dE1EbCf) |
| `V4YieldAdapter` (v2.3) | [`0x06e31B4EfCCdb8450a2e29C590A4AF81843d3172`](https://basescan.org/address/0x06e31B4EfCCdb8450a2e29C590A4AF81843d3172) |
| Guardian | [`0x6b9Db1337B37426a7911e4108d27D44393B95eec`](https://basescan.org/address/0x6b9Db1337B37426a7911e4108d27D44393B95eec) |

The `UGMV23Linked` library is `DELEGATECALL`'d from
`UnifiedGridManagerV23` so the main contract image stays under the
EIP-170 24,576-byte cap. The library has no storage of its own; UGM is
the storage root.

### Source-protocol references

Useful when wiring an adapter constructor on Base mainnet.

| Token / contract | Address |
|---|---|
| USDC | [`0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`](https://basescan.org/address/0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913) |
| flETH | [`0x000000000d564d5be76f7f0d28fe52605afc7cf8`](https://basescan.org/address/0x000000000d564d5be76f7f0d28fe52605afc7cf8) |
| TAKEOVER | [`0x716F8E756f9277F8c9949926141C2666B86B5809`](https://basescan.org/address/0x716F8E756f9277F8c9949926141C2666B86B5809) |
| Uniswap V4 PoolManager | [`0x498581fF718922c3f8e6A244956aF099B2652b2b`](https://basescan.org/address/0x498581fF718922c3f8e6A244956aF099B2652b2b) |
| Takeover `PoolSwap` | [`0xdCF8e5E2a21e9B7e37B1B1a6612F1376723dd08e`](https://basescan.org/address/0xdCF8e5E2a21e9B7e37B1B1a6612F1376723dd08e) |
| Flaunch hooks | [`0x23321f11a6d44fd1ab790044fdfde5758c902fdc`](https://basescan.org/address/0x23321f11a6d44fd1ab790044fdfde5758c902fdc) |
| Flaunch NFT (primary) | [`0x516af52d0c629b5e378da4dc64ecb0744ce10109`](https://basescan.org/address/0x516af52d0c629b5e378da4dc64ecb0744ce10109) |
| Flaunch Zap | [`0x39112541720078c70164EA4Deb61F0A4811910F9`](https://basescan.org/address/0x39112541720078c70164EA4Deb61F0A4811910F9) |

Additional Flaunch NFT deployments the indexer tracks: `0xb4512bf57d50fbcb64a3adf8b17a79b2a204c18c`, `0x0cf6bdf0a85a9d6763361037985b76c8893553af`, `0x6a53f8b799be11a2a3264ef0bff183dcb12d9571`.

## Base Sepolia (chain 84532)

### UGM v2.3

| Contract | Address |
|---|---|
| `UnifiedGridManagerV23` | [`0xC389D001C627C7fcE23405190d453906598c1607`](https://sepolia.basescan.org/address/0xC389D001C627C7fcE23405190d453906598c1607) |
| `UGMV23Linked` (linked library) | [`0xb92EbB9fEbcf09F534B75bD312a66e1C8c918562`](https://sepolia.basescan.org/address/0xb92EbB9fEbcf09F534B75bD312a66e1C8c918562) |
| `FlaunchYieldAdapter` (v2.3) | [`0x444f97f0db32fa12e3c444b9f83b4b51f2767f59`](https://sepolia.basescan.org/address/0x444f97f0db32fa12e3c444b9f83b4b51f2767f59) |
| `V3YieldAdapter` (v2.3) | [`0x0f8387cd6ecad2618a3255baf8ec84a803b2efdb`](https://sepolia.basescan.org/address/0x0f8387cd6ecad2618a3255baf8ec84a803b2efdb) |
| `V4YieldAdapter` (v2.3) | [`0xAd97b449277de911d740026087547c582AA1FB59`](https://sepolia.basescan.org/address/0xAd97b449277de911d740026087547c582AA1FB59) |
| Guardian | [`0x6b9Db1337B37426a7911e4108d27D44393B95eec`](https://sepolia.basescan.org/address/0x6b9Db1337B37426a7911e4108d27D44393B95eec) |

> **Sepolia is now address-aligned with mainnet.** The May 2026 refactor
> redeploy (salt `-V3`) sits at a fresh CREATE2 address, the linked
> library is deployed alongside it, and all three yield adapters are
> approved on the v2.3 UGM. Sepolia and Base mainnet now share a single
> guardian (`0x6b9Db1337B…5eec`) so the same key wires both networks.

### Source-protocol references

| Token / contract | Address |
|---|---|
| USDC | [`0x036CbD53842c5426634e7929541eC2318f3dCF7e`](https://sepolia.basescan.org/address/0x036CbD53842c5426634e7929541eC2318f3dCF7e) |
| flETH | [`0x79FC52701cD4BE6f9Ba9aDC94c207DE37e3314eb`](https://sepolia.basescan.org/address/0x79FC52701cD4BE6f9Ba9aDC94c207DE37e3314eb) |
| TAKEOVER | [`0xe834c1a11fc37354e6a135d643f7969da4fc1561`](https://sepolia.basescan.org/address/0xe834c1a11fc37354e6a135d643f7969da4fc1561) |
| Uniswap V3 NonfungiblePositionManager | [`0x27F971cb582BF9E50F397e4d29a5C7A34f11faA2`](https://sepolia.basescan.org/address/0x27F971cb582BF9E50F397e4d29a5C7A34f11faA2) |
| Uniswap V3 Factory | [`0x4752ba5DBc23f44D87826276BF6Fd6b1C372aD24`](https://sepolia.basescan.org/address/0x4752ba5DBc23f44D87826276BF6Fd6b1C372aD24) |
| Uniswap V4 PositionManager | [`0x4B2C77d209D3405F41a037Ec6c77F7F5b8e2ca80`](https://sepolia.basescan.org/address/0x4B2C77d209D3405F41a037Ec6c77F7F5b8e2ca80) |
| Flaunch NFT (primary) | [`0xe2ef58a54ee79dac0d4a130ea58b340124df9438`](https://sepolia.basescan.org/address/0xe2ef58a54ee79dac0d4a130ea58b340124df9438) |
| Flaunch Zap | [`0x25B747AeCA2612b9804b5c3BB272a3DAeFdC6eaa`](https://sepolia.basescan.org/address/0x25B747AeCA2612b9804b5c3BB272a3DAeFdC6eaa) |

Additional Flaunch NFT deployment the indexer tracks: `0x7d375c9133721083df7b7e5cb0ed8fc78862dfe3`.

## ABIs

The same JSON files served by the Builder portal at
`/abis/<contract>.json` (or downloadable from `/builders/contracts`):

| Contract | ABI | Use it for |
|---|---|---|
| `UnifiedGridManagerV23` | [`UnifiedGridManagerV23.json`](https://takeover.fun/abis/UnifiedGridManagerV23.json) | Reading state, decoding logs, sending writes (addBatch, setPrice, claimFees, moduleTransferSeat) without the SDK |
| `IGridHooksV23` | [`IGridHooksV23.json`](https://takeover.fun/abis/IGridHooksV23.json) | **Hook authors.** Implement on your own contract → guardian `setApprovedModule(true)` → grid owner `setHook` |
| `IYieldAdapter` | [`IYieldAdapter.json`](https://takeover.fun/abis/IYieldAdapter.json) | Building a custom yield source. Two methods: `collectYield(bytes32[])`, `pendingYield(bytes32)` |
| `IFeeReceiver` | [`IFeeReceiver.json`](https://takeover.fun/abis/IFeeReceiver.json) | Implementing a grid-aware fee splitter / treasury that receives `notifyDeposit(gridId, amount)` |
| `FlaunchYieldAdapter` | [`FlaunchYieldAdapter.json`](https://takeover.fun/abis/FlaunchYieldAdapter.json) | Direct interaction with the live Flaunch adapter (deposits, queries) |
| `V3YieldAdapter` | [`V3YieldAdapter.json`](https://takeover.fun/abis/V3YieldAdapter.json) | Direct interaction with the live Uniswap V3 adapter |
| `V4YieldAdapter` | [`V4YieldAdapter.json`](https://takeover.fun/abis/V4YieldAdapter.json) | Direct interaction with the live Uniswap V4 adapter |

The JSON files contain only the `abi` field of the corresponding
Foundry artifact — no bytecode, no metadata. They are regenerated from
[`takeover-contracts/out/`](https://github.com/takeoverapp/takeover-contracts)
by `pnpm run abis:refresh` on a successful contract build, so they
follow the same Solidity surface as the live deployments.

> **Hook authors targeting v2.3 grids** should pair `IGridHooksV23.json`
> with the [Hooks guide](../hooks/) — the interface adds five callbacks
> on top of v2.2's `onSeatHolderChange`, plus an advisory `yieldWeight`
> view that the SDK reads for off-chain ranking. UGM never consumes
> `yieldWeight` itself: on-chain yield distribution stays uniform per
> seat. UGM gas-caps every callback at 150,000 gas via
> `try { ... } catch`, so a missing-selector revert in any callback is
> interpreted as "module didn't implement; no-op". A v2.2 module that
> only implements `onSeatHolderChange` keeps working unchanged.

## Asset hash conventions

The reference adapters use these preimages — match them so UI and indexer
code can reconstruct hashes from onchain identifiers.

| Source | Hash preimage |
|---|---|
| Flaunch | `keccak256(abi.encodePacked(flaunch, tokenId))` |
| Uniswap V3 / Aerodrome / Pancake V3 | `keccak256(abi.encodePacked(positionManager, tokenId))` |
| Uniswap V4 | `keccak256(abi.encodePacked(v4PositionManager, tokenId))` |
| Push-only protocol-fee adapters | One hash per deployment, registered manually against the receiving grid |

See [Asset hashes](../adapters/asset-hash.md) for the rules new adapters
should follow.

## Keeping this page accurate

Sources of truth, in priority order:

1. **SDK preset** ([`takeover-app/packages/sdk/src/networks/index.ts`](https://github.com/takeoverapp/takeover-app/blob/develop/packages/sdk/src/networks/index.ts)) — what the Builder portal and downstream apps read.
2. App env files (`takeover-app/.env.mainnet`, `takeover-app/.env.sepolia`).
3. Deploy broadcast logs in [`takeoverapp/takeover-contracts/broadcast/`](https://github.com/takeoverapp/takeover-contracts/tree/main/broadcast).

When a new deploy script lands:

1. Pull the new address from the broadcast JSON or the updated env file.
2. Update `packages/sdk/src/networks/index.ts` first.
3. Update the tables on this page to match.
4. Annotate any address being replaced as `revoked` (don't delete it —
   `withdrawAsset` from a revoked adapter is the documented unwind path).
5. PR with `deployments` in the title.
