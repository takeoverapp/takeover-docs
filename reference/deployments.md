# Deployments

Canonical contract addresses for Takeover on Base mainnet and Base Sepolia.

> **A note on UGM versions.** Two UGM contracts are live today:
>
> - **UGM v2.1** — what these docs target. Hosts the Boardroom (hvTAKEOVER)
>   today and is where new adapters should be registered.
> - **UGM v2** — the older deployment. Currently hosts the production grids
>   that use `FlaunchYieldAdapter`, `V3YieldAdapter`, and `V4YieldAdapter`.
>   Those adapters are great pattern references; a v2.1 fork follows the
>   same shape but points its constructor at the v2.1 UGM address.
>
> When in doubt: **new adapter ⇒ register on UGM v2.1**.

## Base mainnet (chain 8453)

### UGM v2.1 (primary)

| Contract | Address |
|---|---|
| `UnifiedGridManagerV21` | [`0x5455DB84A922eB6B29D8E7e3675e01a1436A735f`](https://basescan.org/address/0x5455DB84A922eB6B29D8E7e3675e01a1436A735f) |
| `ProtocolYieldAdapter` (HV adapter) | [`0x35caE134A6DdB3C9D0BB10ceBe404DdaD8b0Bb0A`](https://basescan.org/address/0x35caE134A6DdB3C9D0BB10ceBe404DdaD8b0Bb0A) |
| HV governance module | [`0x2e69A3dC3A8E2aA5fF380f16b23F8eCcCFDC0037`](https://basescan.org/address/0x2e69A3dC3A8E2aA5fF380f16b23F8eCcCFDC0037) |
| Boardroom grid ID | `0` |
| Boardroom tax token (`TAKEOVER`) | [`0x716F8E756f9277F8c9949926141C2666B86B5809`](https://basescan.org/address/0x716F8E756f9277F8c9949926141C2666B86B5809) |

### UGM v2 (legacy reference)

| Contract | Address |
|---|---|
| `UnifiedGridManager` (v2) | [`0x107Aba3406b3fcb93D1919BFE5c082A5f4d28A22`](https://basescan.org/address/0x107Aba3406b3fcb93D1919BFE5c082A5f4d28A22) |
| `FlaunchYieldAdapter` | [`0xd631b14727836824a87012B9E566Ebae159E0969`](https://basescan.org/address/0xd631b14727836824a87012B9E566Ebae159E0969) |
| `V3YieldAdapter` | [`0xeE1e0bcE34022d9a0c40614bC972F437504257bc`](https://basescan.org/address/0xeE1e0bcE34022d9a0c40614bC972F437504257bc) |
| `V4YieldAdapter` | [`0xA5B1128Baa4c4Bf5EE6Ccb7F7fc6BC7cE1961082`](https://basescan.org/address/0xA5B1128Baa4c4Bf5EE6Ccb7F7fc6BC7cE1961082) |
| `GridLaunchZap` | [`0x2d2d21236149e0bBF496bE3155D9F48b17616850`](https://basescan.org/address/0x2d2d21236149e0bBF496bE3155D9F48b17616850) |
| `V2_SWAP_EXECUTOR` | [`0x1f6e38610343301c2B3C328C271b2dd62DdfA1c8`](https://basescan.org/address/0x1f6e38610343301c2B3C328C271b2dd62DdfA1c8) |
| `V2_BUYBACK_KEEPER` | [`0x17a9AEa1Ddf95B6aDB627dCcd234278cae6c2Ee2`](https://basescan.org/address/0x17a9AEa1Ddf95B6aDB627dCcd234278cae6c2Ee2) |
| `V2_MIGRATION_ZAP` | [`0x430851df92db6831809da73dA4AB87748AeD3546`](https://basescan.org/address/0x430851df92db6831809da73dA4AB87748AeD3546) |

### Legacy v1

| Contract | Address |
|---|---|
| `TakeoverFeeSplitManager` (TSFM) factory | [`0xAbB70c40b74ec358220ef3FDe25563f58d37366C`](https://basescan.org/address/0xAbB70c40b74ec358220ef3FDe25563f58d37366C) |
| `TakeoverZap` | [`0x3d5EadF1585dC98eD306C81214574F75a99e8290`](https://basescan.org/address/0x3d5EadF1585dC98eD306C81214574F75a99e8290) |
| `TakeoverBuyback` | [`0x743e0d6C56D0fC701ce0a4bA167F5bb24CA41ed5`](https://basescan.org/address/0x743e0d6C56D0fC701ce0a4bA167F5bb24CA41ed5) |
| Graduation Zap | [`0x624c6353bD796Fd5134A1D1C76891B602c62A039`](https://basescan.org/address/0x624c6353bD796Fd5134A1D1C76891B602c62A039) |

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

### UGM v2.1 (primary)

| Contract | Address |
|---|---|
| `UnifiedGridManagerV21` | [`0x9ce222e1fe1F4301c4E36aFc9fbF7394952894b9`](https://sepolia.basescan.org/address/0x9ce222e1fe1F4301c4E36aFc9fbF7394952894b9) |
| `ProtocolYieldAdapter` (HV adapter) | [`0xBB58422B2D440857e40AF528677Ba059d910349B`](https://sepolia.basescan.org/address/0xBB58422B2D440857e40AF528677Ba059d910349B) |
| HV governance module | [`0x609c611Fb4D68F2Eab24e4F02c257919f98B9605`](https://sepolia.basescan.org/address/0x609c611Fb4D68F2Eab24e4F02c257919f98B9605) |
| Boardroom grid ID | `0` |
| Boardroom tax token (`TAKEOVER`) | [`0xe834c1a11fc37354e6a135d643f7969da4fc1561`](https://sepolia.basescan.org/address/0xe834c1a11fc37354e6a135d643f7969da4fc1561) |

### UGM v2 (legacy reference)

| Contract | Address |
|---|---|
| `UnifiedGridManager` (v2) | [`0x50aa815b0FCd2686CD61d1bDd61188611f8367bF`](https://sepolia.basescan.org/address/0x50aa815b0FCd2686CD61d1bDd61188611f8367bF) |
| `FlaunchYieldAdapter` | [`0xE80419EBD8aFEe8dcD86BaF0b971E25Ef0244890`](https://sepolia.basescan.org/address/0xE80419EBD8aFEe8dcD86BaF0b971E25Ef0244890) |
| `V3YieldAdapter` | [`0x4918ca4d0b1426649B87Fa40EaA22B28D36A594c`](https://sepolia.basescan.org/address/0x4918ca4d0b1426649B87Fa40EaA22B28D36A594c) |
| `V4YieldAdapter` | [`0x026E11F0220e22c7306e939EEaC682672053765A`](https://sepolia.basescan.org/address/0x026E11F0220e22c7306e939EEaC682672053765A) |
| `GridLaunchZap` | [`0x78F7B9912882E0d0d4A6b1551E7288bf004B5e65`](https://sepolia.basescan.org/address/0x78F7B9912882E0d0d4A6b1551E7288bf004B5e65) |
| `V2_SWAP_EXECUTOR` | [`0x2cc144Dc69EB27613cff6C9C6f9e61EEc2661966`](https://sepolia.basescan.org/address/0x2cc144Dc69EB27613cff6C9C6f9e61EEc2661966) |
| `V2_BUYBACK_KEEPER` | [`0xa47855b51dA1DCE5A1dEBD2998c4371Bc6810e66`](https://sepolia.basescan.org/address/0xa47855b51dA1DCE5A1dEBD2998c4371Bc6810e66) |
| `V2_MIGRATION_ZAP` | [`0x4e46a9C23aaA33283224479Ef9d3de96deE2fC7e`](https://sepolia.basescan.org/address/0x4e46a9C23aaA33283224479Ef9d3de96deE2fC7e) |

### Legacy v1

| Contract | Address |
|---|---|
| `TakeoverFeeSplitManager` (TSFM) factory | [`0xE911E8122948a2a6f25E415b1CE0De19fB130fB2`](https://sepolia.basescan.org/address/0xE911E8122948a2a6f25E415b1CE0De19fB130fB2) |
| `TakeoverZap` | [`0xA9545E24f94B8801D54dbD2D298017A8A67cF712`](https://sepolia.basescan.org/address/0xA9545E24f94B8801D54dbD2D298017A8A67cF712) |
| `TakeoverBuyback` | [`0x32Bc24A5CB7144aE260441f7344b55A08f588672`](https://sepolia.basescan.org/address/0x32Bc24A5CB7144aE260441f7344b55A08f588672) |
| Graduation Zap | [`0x9104e94a3b3560EcC4a34d364E0aa05A07272c71`](https://sepolia.basescan.org/address/0x9104e94a3b3560EcC4a34d364E0aa05A07272c71) |
| LP Graduation Zap | [`0xac481127bc1dcdf14677b3f9fbc382eb8d75ed0d`](https://sepolia.basescan.org/address/0xac481127bc1dcdf14677b3f9fbc382eb8d75ed0d) |
| UV4 yield interface | [`0x5c2b4dfd01e42df1ac5b1e11f759a8387f68b452`](https://sepolia.basescan.org/address/0x5c2b4dfd01e42df1ac5b1e11f759a8387f68b452) |

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

## Asset hash conventions

The reference adapters use these preimages — match them so UI and indexer
code can reconstruct hashes from on-chain identifiers.

| Source | Hash preimage |
|---|---|
| Flaunch | `keccak256(abi.encodePacked(flaunch, tokenId))` |
| Uniswap V3 / Aerodrome / Pancake V3 | `keccak256(abi.encodePacked(positionManager, tokenId))` |
| Uniswap V4 | `keccak256(abi.encodePacked(v4PositionManager, tokenId))` |
| `ProtocolYieldAdapter` | One hash per deployment, registered manually against the Boardroom (HV) grid |

See [Asset hashes](../adapters/asset-hash.md) for the rules new adapters
should follow.

## Keeping this page accurate

Sources of truth:

- App env files (`takeover-app/.env.mainnet`, `takeover-app/.env.sepolia`).
- Deploy broadcast logs in [`takeoverapp/takeover-contracts/broadcast/`](https://github.com/takeoverapp/takeover-contracts/tree/main/broadcast).

When a new deploy script lands:

1. Pull the new address from the broadcast JSON or the updated env file.
2. Update the table above.
3. Annotate any address being replaced as `revoked` (don't delete it —
   `withdrawAsset` from a revoked adapter is the documented unwind path).
4. PR with `deployments` in the title.
