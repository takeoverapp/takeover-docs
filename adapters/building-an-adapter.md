# Building a yield adapter

This is the load-bearing page. By the end of it you'll have the shape of a
contract that:

- Holds your protocol's yield source (LP NFT, escrow handle, fee-claim right,
  whatever).
- Implements [`IYieldAdapter`](interface.md).
- Plays correctly with the [adapter lifecycle](lifecycle.md).
- Is ready to submit for [guardian approval](../submit/adapter-approval-process.md).

Everything below assumes you've read [The yield adapter model](../overview/yield-adapter-model.md).
If you haven't, start there — the rest of this page won't make sense.

## Step 0: pick the simplest reference adapter that matches your shape

Don't start from scratch. Fork the reference adapter whose pattern is closest
to yours and replace its source-side calls.

| Pattern | Reference adapter | Use when |
|---|---|---|
| **Single-token push** | [`FlaunchYieldAdapter`](examples/flaunch.md) | Source already produces yield in the grid's yieldToken. No swap needed. |
| **Two-token swap-and-forward** | [`V3YieldAdapter`](examples/uniswap-v3.md) / [`V4YieldAdapter`](examples/uniswap-v4.md) | Source produces fees in two tokens; you swap one side via the source pool. |
| **Push-only / external trigger** | [`ProtocolYieldAdapter`](examples/protocol-fees.md) | The adapter holds nothing; it just receives token transfers and forwards them. `collectYield` is a no-op. |

## Step 1: pick the grid's `yieldToken`

The grid's `yieldToken` is set at `createGrid` time and is **immutable** after
that. Your adapter must produce that token (or accept that there's already a
fixed grid out there with a yieldToken your adapter has to match).

UGM accepts three flavours of `yieldToken`:

- `address(0)` — **raw ETH yield.** UGM also accepts `flETH` and `WETH`
  on this path, automatically.
- A specific ERC20 (e.g. `USDC`, `TAKEOVER`).
- Any other ERC20 a grid creator picks. UGM doesn't restrict this.

If your source pays in something the grid doesn't accept, your adapter has to
swap. The two-token-swap pattern in `V3YieldAdapter` shows how to use the
position's own pool to convert without trusting an external router.

> **Rule of thumb:** if your source pays in flETH, WETH, or USDC, pick a grid
> whose `yieldToken` matches one of those and skip the swap entirely.

See [Yield token rules](yield-token-rules.md) for the exact accepted matrix.

## Step 2: define your `assetHash`

`assetHash` is the only handle UGM has on whatever your adapter is wrapping.
Pick a hash function that:

- **Uniquely identifies the source-side object.** Two distinct LP NFTs must
  hash differently; two registrations of the *same* NFT must hash the same.
- **Is reproducible offchain.** UI and indexer code reconstruct asset hashes
  to display per-asset accruals.
- **Doesn't collide across protocols.** Including the source contract address
  in the preimage makes one adapter safely cover multiple position-manager
  versions (`V3YieldAdapter` covers Uniswap V3, Aerodrome, and PancakeSwap
  V3 NFTs from one contract using exactly this trick).

The reference convention is:

```solidity
bytes32 assetHash = keccak256(abi.encodePacked(positionManagerOrFlaunch, tokenId));
```

For non-NFT yield sources, hash whatever stable identifier you have:

```solidity
// Single-source, no NFT — adapter handles one logical position
bytes32 assetHash = keccak256(abi.encodePacked(address(this), "primary"));

// Per-pool, identified by Uniswap PoolId
bytes32 assetHash = keccak256(abi.encodePacked(address(positionManager), PoolId.unwrap(poolId)));
```

See [Asset hashes](asset-hash.md) for more patterns and the uniqueness
invariants UGM enforces.

## Step 3: scaffold the contract

> **Which UGM do I point at?** New adapters should target **UGM v2.1**.
> See [Deployments](../reference/deployments.md) for the chain-specific
> address. The reference adapters (`FlaunchYieldAdapter`, `V3YieldAdapter`,
> `V4YieldAdapter`) currently sit on the older v2 UGM — their patterns are
> still right, you just swap the constructor address.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@solady/auth/Ownable.sol";
import {ReentrancyGuard} from "@solady/utils/ReentrancyGuard.sol";

import {IYieldAdapter} from "@takeover/interfaces/IYieldAdapter.sol";

interface IUgm {
    function registerAsset(uint256 gridId, bytes32 assetHash) external;
    function withdrawAsset(uint256 gridId, bytes32 assetHash) external;
    function receiveYieldETH(bytes32 assetHash, uint256 amount) external payable;
    function receiveYieldERC20(bytes32 assetHash, address token, uint256 amount) external;
    function gridConfig(uint256 gridId)
        external
        view
        returns (
            address creator,
            uint64 createdAt,
            uint16 totalSeats,
            uint16 taxRateBps,
            address taxToken,
            address yieldToken
        );
    function holderSeatCount(uint256 gridId, address holder) external view returns (uint256);
}

contract MyYieldAdapter is IYieldAdapter, Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    IUgm public immutable ugm;

    struct Asset {
        uint256 gridId;
        // …source-specific state (NFT id, pool key, last-fees baseline)…
        bool active;
    }

    mapping(bytes32 => Asset) public assets;
    bytes32[] public assetHashes;

    error NotGridCreator();
    error NotUGM();
    error NotAllSeatsHeld();
    error AssetAlreadyDeposited();
    error AssetNotActive();
    error IncompatibleYieldToken();

    constructor(address _owner, address _ugm) {
        _initializeOwner(_owner);
        ugm = IUgm(_ugm);
    }
}
```

That's the skeleton every reference adapter starts from.

## Step 4: implement deposit

Deposit is where you take ownership of the source-side object and tell UGM
about it.

```solidity
function deposit(/* source-specific args */ uint256 gridId) external nonReentrant {
    (address creator,,,,, address yieldToken) = ugm.gridConfig(gridId);

    // (a) Gate: only the grid creator can attach yield to a grid.
    if (msg.sender != creator) revert NotGridCreator();

    // (b) Gate: source produces something the grid will accept.
    if (!_isYieldTokenCompatible(yieldToken)) revert IncompatibleYieldToken();

    // (c) Take ownership of the source-side object.
    //     (e.g. IERC721(positionManager).transferFrom(creator, address(this), tokenId);)

    // (d) Compute and store the asset hash.
    bytes32 assetHash = _computeAssetHash(/* source-specific */);
    if (assets[assetHash].active) revert AssetAlreadyDeposited();

    assets[assetHash] = Asset({gridId: gridId, /* … */ active: true});
    assetHashes.push(assetHash);

    // (e) Tell UGM. This is the call that needs guardian approval to succeed.
    ugm.registerAsset(gridId, assetHash);

    emit AssetDeposited(assetHash, /* source-specific */ gridId);
}
```

Notes:

- **Always read `creator` and `yieldToken` straight from UGM.** Don't take
  them as arguments — that's a forging vector.
- **Take ownership before registering.** If `transferFrom` reverts, you don't
  want a zombie registration.
- **Reentrancy.** `nonReentrant` on the public entrypoint is enough; UGM's
  `registerAsset` is internal-only state. Don't put `nonReentrant` on
  `collectYield` (see Step 5).

For zap-style flows where a router contract pre-funds the adapter with the
NFT and then calls `depositFor` on behalf of the creator, see
`V3YieldAdapter.depositFor` and the `approvedZaps` allowlist on UGM.

## Step 5: implement `collectYield`

This is the function UGM calls during `claimFees`. The reference loop:

```solidity
function collectYield(bytes32[] calldata _assetHashes) external override {
    if (msg.sender != address(ugm)) revert NotUGM();

    for (uint256 i; i < _assetHashes.length; ++i) {
        _collectSingle(_assetHashes[i]);
    }
}

function _collectSingle(bytes32 assetHash) internal {
    Asset storage asset = assets[assetHash];
    if (!asset.active) return;                       // soft-skip, not revert

    uint256 collected = _pullFromSource(asset);      // adapter-specific
    if (collected == 0) return;                      // idempotent

    (,,,,, address yieldToken) = ugm.gridConfig(asset.gridId);
    uint256 yieldAmount = _convertToYield(asset, collected, yieldToken);
    if (yieldAmount == 0) return;

    if (yieldToken == address(0)) {
        ugm.receiveYieldETH{value: yieldAmount}(assetHash, yieldAmount);
    } else {
        IERC20(yieldToken).forceApprove(address(ugm), yieldAmount);
        ugm.receiveYieldERC20(assetHash, yieldToken, yieldAmount);
    }

    emit YieldClaimed(assetHash, yieldAmount);
}
```

Five things to keep in mind:

1. **`msg.sender` check is mandatory.** Anyone calling this directly would
   either get an irrelevant pull, or — worse — drain pre-positioned token
   approvals. Always gate on UGM.
2. **No-op on `!active`.** UGM will sometimes pass hashes for assets that
   were just withdrawn. Don't revert; just skip.
3. **No-op on zero pending.** Calling twice in the same block must not move
   any tokens.
4. **Use `forceApprove`, not `approve`.** USDT-style tokens revert if you
   approve a non-zero allowance over a non-zero allowance. `forceApprove`
   resets first.
5. **Don't emit on no-op paths.** The indexer treats `YieldClaimed` as a
   real event. Empty events spam dashboards.

## Step 6: implement `pendingYield`

Pure read. Same shape as `collectYield`, minus the side effects, plus a
conversion to `yieldToken` units:

```solidity
function pendingYield(bytes32 assetHash) external view override returns (uint256) {
    Asset storage asset = assets[assetHash];
    if (!asset.active) return 0;

    uint256 raw = _previewPendingFromSource(asset);
    return _previewConvertToYield(asset, raw);
}
```

When converting, **do not revert** if the price source is missing. Return 0
or the raw amount; UI list views call this for every grid on the page.

For two-token sources, see how `V4YieldAdapter._toYieldUnits` walks
`sqrtPriceX96` to convert without overflow.

## Step 7: implement withdraw

Withdraw is the inverse of deposit. The reference contract requires the grid
creator to hold all seats first, so unwinding can't strand seat holders:

```solidity
function withdraw(bytes32 assetHash, address recipient) external nonReentrant {
    Asset storage asset = assets[assetHash];
    if (!asset.active) revert AssetNotActive();

    (address creator,, uint16 totalSeats,,,) = ugm.gridConfig(asset.gridId);
    if (msg.sender != creator) revert NotGridCreator();
    if (ugm.holderSeatCount(asset.gridId, creator) != totalSeats) revert NotAllSeatsHeld();

    _collectSingle(assetHash);                       // sweep last yield
    ugm.withdrawAsset(asset.gridId, assetHash);

    asset.active = false;
    _removeAssetHash(assetHash);

    // Return the source-side object.
    // e.g. IERC721(asset.positionManager).transferFrom(address(this), recipient, asset.tokenId);

    emit AssetWithdrawn(assetHash, recipient);
}
```

If your adapter pattern needs a different unwind condition (graduation, time
lock, governance vote), document it explicitly on the example page — it's the
single most common point of confusion for integrators.

## Step 8: tests

At minimum, your test suite should cover:

- **Happy path.** Deposit → `claimFees` from a holder → seat balance increases.
- **Idempotent collect.** Two `claimFees` in a row collect zero on the second
  call.
- **Inactive asset.** `collectYield([fakeHash])` is a no-op.
- **Wrong caller.** `collectYield` from a non-UGM address reverts.
- **Wrong yieldToken.** Deposit on a grid whose `yieldToken` your source
  can't match reverts `IncompatibleYieldToken`.
- **Withdraw gating.** Withdraw without all seats reverts; withdraw after
  buying all seats succeeds.
- **Revoked adapter.** After `setApprovedAdapter(adapter, false)`, deposit
  reverts but residual yield can still be pushed and `withdrawAsset` still
  works for already-registered assets.

See [Testing your adapter](testing.md) for fork-test patterns and the
`MockUGM` helper.

## Step 9: submit

When the contract is audited and tests are green:

1. Run through the [pre-submission checklist](../submit/checklist.md).
2. File a [guardian approval request](../submit/adapter-approval-process.md).
3. Once approved, deploy + call your own `deposit` once to register a real
   asset on a real grid.

## Where to next

- [Yield token rules](yield-token-rules.md) — exact `receiveYield*` accept
  matrix.
- [Asset hashes](asset-hash.md) — uniqueness, conventions, edge cases.
- [Security considerations](security.md) — common adapter footguns.
- [Examples](examples/flaunch.md) — annotated walkthroughs of every reference
  adapter.
