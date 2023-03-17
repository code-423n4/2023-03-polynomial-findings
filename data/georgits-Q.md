## Use the latest solidity version with a stable pragma statement
[PauseModifier.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/utils/PauseModifier.sol#L3), [LiquidityToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol#L2), [SynthetixAdapter.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SynthetixAdapter.sol#L2), [PowerPerp.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L2), [SystemManager.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L2), [ShortToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L2), [ShortCollateral.sol](), [Exchange.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L2), [LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L2), [KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L2)


## Missing zero address checks
VaultToken.sol - [28](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L28), [32](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L32)

LiquidityToken.sol - [29](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol#L29), [33](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol#L33)

PowerPerp.sol - [33](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L33), [37](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L37), [42](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L42), [47](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L47)

SystemManager.sol - [54](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L54-L57), [62](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L62-L81)

LiquidityPool.sol - [254](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L254), [508](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L508), [651](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L651)

KangarooVault.sol - [222](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L222), [549](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L549)

ShortCollateral.sol - [141](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L141)

## Remove unused imports
SystemManager.sol - [14](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L14)

LiquidityPool.sol - [18](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L18)

KangarooVault.sol - [4](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L4)

## Use a `require` statement instead of `assert`
LiquidityPool.sol - [220](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L220)

## `hedgePositions()`, `rebalanceMargin()` & `increaseMargin()` functions(which are in `Trading Methods` section) use `requiresAuth` instead of `onlyExchange` modifier
LiquidityPool.sol - [568](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L568), [591](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L591), [613](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L613)

## Anyone can call `init()` and set their own values
SystemManager.sol - [62](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L62-L70)

## Event is not emitted
LiquidityPool.sol - [651](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L651)
Add `emit UpdateFeeReceipient(feeReceipient, _feeReceipient);`

## Use a constant(for example `EMPTY_STRING`) for `“”`
ShortCollateral.sol - [160](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L160), [183](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L183)

## Use a constant for `1e18`
LiquidityPool.sol - [342](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L342), [635](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L635), [643](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L643)