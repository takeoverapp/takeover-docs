# Example: `FlaunchYieldAdapter`

The simplest reference adapter, and a good first read.

> Source: [`takeoverapp/takeover-contracts/src/FlaunchYieldAdapter.sol`](https://github.com/takeoverapp/takeover-contracts/blob/main/src/FlaunchYieldAdapter.sol).
>
> **Live v2.3 deployments:**
> mainnet [`0x1867650664c252186a214D5ed37CB1FD736b7bE6`](https://basescan.org/address/0x1867650664c252186a214D5ed37CB1FD736b7bE6),
> Sepolia [`0x444f97f0db32fa12e3c444b9f83b4b51f2767f59`](https://sepolia.basescan.org/address/0x444f97f0db32fa12e3c444b9f83b4b51f2767f59).
> See [Deployments](../../reference/deployments.md) for the full
> address book.

## What it does

Wraps a [Flaunch](https://flaunch.gg) memecoin NFT into a UGM grid. Each
Flaunch NFT corresponds to a memecoin pool that accrues fees in **flETH**.
The adapter:

- Holds the Flaunch NFT.
- Reads the pool's `totalFeesAllocated` from the FeeEscrow registry to
  compute pending yield.
- Pulls flETH from FeeEscrow on demand.
- Forwards flETH to UGM as ERC20 yield.

It's the canonical **single-token push** pattern: source pays in something
UGM already accepts (flETH, treated as ETH-equivalent), so no swap is needed.

## Why it's the easiest reference

- One source token (flETH).
- No price math — yield is denominated in the same units the grid wants.
- One asset = one NFT = one pool, no per-asset routing tables.
- Withdraw doesn't need to swap or close a position — just transfer the NFT
  back.

## Walking the contract

### Constructor

```solidity
constructor(address _owner, address _ugm, address _flETH, address _feeEscrowRegistry) {
    if (_owner == address(0) || _ugm == address(0) || _flETH == address(0) || _feeEscrowRegistry == address(0)) {
        revert ZeroAddress();
    }
    _initializeOwner(_owner);
    ugm = UnifiedGridManager(payable(_ugm));
    flETH = IFLETH(_flETH);
    feeEscrowRegistry = IFeeEscrowRegistry(_feeEscrowRegistry);
}
```

Three immutables: UGM, the flETH token, and Flaunch's FeeEscrow registry. The
registry is what lets one adapter cover every Flaunch deployment without
hardcoding addresses per memecoin.

### Deposit

```solidity
function deposit(address flaunch, uint256 tokenId, uint256 gridId) external nonReentrant {
    (address creator,,,,, address yieldToken) = ugm.gridConfig(gridId);
    if (msg.sender != creator) revert NotGridCreator();
    if (yieldToken != address(0) && yieldToken != address(flETH))
        revert IncompatibleYieldToken();

    bytes32 assetHash = keccak256(abi.encodePacked(flaunch, tokenId));
    if (assets[assetHash].active) revert AssetAlreadyDeposited();

    IERC721(flaunch).transferFrom(msg.sender, address(this), tokenId);

    PoolId pid = IFlaunchPoolId(flaunch).poolId(tokenId);

    assets[assetHash] = FlaunchAsset({
        flaunch: flaunch,
        tokenId: tokenId,
        gridId: gridId,
        assetHash: assetHash,
        poolId: pid,
        lastFeesAllocated: _totalPoolFees(pid),
        active: true
    });
    assetHashes.push(assetHash);

    ugm.registerAsset(gridId, assetHash);

    emit AssetDeposited(assetHash, flaunch, tokenId, gridId);
}
```

Things worth noting:

- **`yieldToken == address(0) || yieldToken == flETH`** — UGM accepts flETH
  on ETH-yield grids, so we allow either. See [Yield token rules](../yield-token-rules.md).
- **`lastFeesAllocated` baseline.** Recorded at deposit time so the adapter
  doesn't credit historical fees as new yield. Without this, the first
  `collectYield` would over-pay.
- **`assetHash = keccak256(flaunch, tokenId)`** — both pieces of the preimage
  are needed because Flaunch may run multiple deployments.

### `collectYield`

```solidity
function collectYield(bytes32[] calldata _assetHashes) external override nonReentrant {
    if (msg.sender != address(ugm)) revert NotUGM();

    _withdrawAllFees();                          // pull from FeeEscrow

    uint256 count = _assetHashes.length;
    if (count == 0) return;

    uint256 totalToForward;
    uint256[] memory deltas = new uint256[](count);

    for (uint256 i; i < count; ++i) {
        FlaunchAsset storage asset = assets[_assetHashes[i]];
        if (!asset.active) continue;

        uint256 current = _totalPoolFees(asset.poolId);
        uint256 delta = current - asset.lastFeesAllocated;

        deltas[i] = delta;
        totalToForward += delta;
    }
    if (totalToForward == 0) return;

    // The adapter may not have pulled enough flETH for the full delta
    // (e.g. legacy escrows). Cap by actual balance.
    uint256 bal = flETH.balanceOf(address(this));
    if (totalToForward > bal) totalToForward = bal;

    IERC20(address(flETH)).forceApprove(address(ugm), totalToForward);

    uint256 forwarded;
    for (uint256 i; i < count; ++i) {
        if (deltas[i] == 0) continue;

        uint256 amount = deltas[i];
        if (forwarded + amount > totalToForward) amount = totalToForward - forwarded;
        if (amount == 0) break;

        assets[_assetHashes[i]].lastFeesAllocated += amount;
        forwarded += amount;
        ugm.receiveYieldERC20(_assetHashes[i], address(flETH), amount);
        emit YieldClaimed(_assetHashes[i], amount);
    }
}
```

Pattern highlights:

- **Pre-pull, then attribute.** `_withdrawAllFees()` runs once for the whole
  batch instead of per-asset. Keeps the gas profile flat regardless of how
  many assets UGM passes in.
- **Two-pass deltas.** First loop computes how much each asset should move.
  Second loop actually moves it. This decouples "what's owed" from "what's
  in the contract" so a partial pull doesn't desync individual asset
  baselines.
- **Idempotent.** No new fees → `totalToForward == 0` → early return. Two
  `claimFees` calls back-to-back move nothing on the second.

### `pendingYield`

```solidity
function pendingYield(bytes32 assetHash) external view override returns (uint256) {
    FlaunchAsset storage asset = assets[assetHash];
    if (!asset.active) return 0;
    return _totalPoolFees(asset.poolId) - asset.lastFeesAllocated;
}
```

That's it. Yield is already in flETH units; no conversion needed.

### Withdraw

```solidity
function withdraw(bytes32 assetHash, address recipient) external nonReentrant {
    FlaunchAsset storage asset = assets[assetHash];
    if (!asset.active) revert AssetNotActive();

    (address creator,,uint16 totalSeats,,,) = ugm.gridConfig(asset.gridId);
    if (msg.sender != creator) revert NotGridCreator();
    if (ugm.holderSeatCount(asset.gridId, creator) != totalSeats) revert NotAllSeatsHeld();

    _collectSingle(assetHash);                  // sweep last yield first
    ugm.withdrawAsset(asset.gridId, assetHash);

    asset.active = false;
    _removeAssetHash(assetHash);

    IERC721(asset.flaunch).transferFrom(address(this), recipient, asset.tokenId);

    emit AssetWithdrawn(assetHash, recipient);
}
```

The standard "creator holds all seats" gate. If you want to permit unwinds
under different conditions, document them on your own example page.

## What this adapter doesn't do

- **No swaps.** flETH in, flETH out.
- **No multi-grid routing per NFT.** One NFT = one grid for the lifetime of
  the registration.
- **No partial collect.** It's all-or-nothing per asset per call.

If your source needs any of those, look at [`V4YieldAdapter`](uniswap-v4.md)
(swap-and-forward) or [`ProtocolYieldAdapter`](protocol-fees.md) (push-only).

## Pattern checklist for forking this adapter

You can mostly s/Flaunch/yours/ if your source matches this shape:

- [ ] Source produces yield in a single token (flETH equivalent in your case).
- [ ] You can read pending yield from the source as a monotonically-
      increasing counter.
- [ ] You can pull pending yield with one call (or batch them yourself).
- [ ] You hold one ERC721 (or equivalent) per asset.
- [ ] Grid creators and zaps are the only entities depositing.

If any of those don't hold, peek at the other examples first.
