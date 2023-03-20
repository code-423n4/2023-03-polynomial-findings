### [L-01] `returnedAmount` is wrong for delayed withdrawal in `KangarooVault` and `LiquidityPool`

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L290

```solidity
    current.returnedAmount = availableFunds;
```
In `KangarooVault.processWithdrawalQueue`, when funds are not enough to return, `returnedAmount` is set to `availableFunds`. But the queued withdrawal will be processed in the future again, so the update is not correct.

The following is the correct implementation. 
```solidity
    current.returnedAmount += availableFunds;
```
`returnedAmount` is wrong, but it is not used in the codebase, the impact is low.

### [L-02] `depositFee` should be validated in `LiquidityPool`

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L657-L661
- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L184-L187

There is no validation about `depositFee` in `LiquidityPool.setFees`. But if depositFee > 1e18, `deposit` will revert and won't work.

```solidity
        uint256 fees = amount.mulWadDown(depositFee);
        uint256 amountForTokens = amount - fees;
```

### [L-03] `LiquidityPool.processDeposits` reverts when `count` = 0 and `queuedDepositHead` = 0

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L219-L220

```solidity
    assert(queuedDepositHead + count - 1 < nextQueuedDepositId);
```

`LiquidityPool.processDeposits` reverts when `count` = 0 and `queuedDepositHead` = 0 due to underflow, while it should work without reversion.


### [L-04] `usedFunds` is not validated in `LiquidityPool.hedgePositions`

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L581

```solidity
    usedFunds += marginDelta;
```

In `hedgePositions`, `marginDelta` can be positive so `usedFunds` can be increased. In the case, `usedFunds` should be validated that `usedFunds` <= `totalFunds`.

### [L-05] `VaultToken.setVault` can be front-run

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/VaultToken.sol#L35-L40

`VaultToken.setVault` don't have any permission to set vault and only accepts the first call. So if admin don't set the vault right after it is constructed in the same block, an attacker can front-run and call this function to cause a DOS attack.

### [L-06] `maxDepositAmount` is not used in `KangarooVault`

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L506-L509

The storage variable `maxDepositAmount` is only set and not used in `KangarooVault`.

### [L-07] Solmate's `safeApprove` don't check allowance.

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L636-L637

```solidity
        SUSD.safeApprove(address(LIQUIDITY_POOL), maxCost);
        uint256 totalCost = EXCHANGE.closeTrade(tradeParams);
```
In `KangarooVault._clearPendingOpenOrders`, it approves `maxCost`, but only `totalCost` is consumed in `LIQUIDITY_POOL`.
`totalCost` can be less than `maxCost`. So if `safeApprove` is from OpenZeppelin, `_clearPendingOpenOrders` will work only once because `OpenZeppelin`'s `safeApprove` checks allowance. But solmate's `safeApprove` doesn't check allowance and there is no problem in `_clearPendingOpenOrders` at the moment. If there is a plan to replace solmate's `SafeTransferLib` by OpenZeppelin's, this problem should be considered. 

