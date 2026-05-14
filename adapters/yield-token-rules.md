# Yield token rules

> **Tier 2 placeholder.** Detailed accept matrix for `receiveYieldETH` and
> `receiveYieldERC20` on UGM v2.3.

## Today's quick reference

| Grid `yieldToken` | What `receiveYieldETH` accepts | What `receiveYieldERC20` accepts |
|---|---|---|
| `address(0)` (ETH) | raw ETH (`msg.value == amount`) | flETH or WETH only |
| any ERC20 | reverts (`WrongYieldToken`) | exactly that ERC20 |

Reference (UGM v2.3):

- `receiveYieldETH` requires `_grids[gridId].yieldToken == address(0)` and
  `msg.value == amount`.
- `receiveYieldERC20` requires the token to be either flETH/WETH (when
  `yieldToken == 0`) or an exact match (otherwise).

This page will be expanded with examples for each row, including the
WETH ↔ ETH unwrap pattern used in `V3YieldAdapter` / `V4YieldAdapter` and the
flETH passthrough used in `FlaunchYieldAdapter`.
