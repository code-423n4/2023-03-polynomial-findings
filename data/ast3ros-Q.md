# [L-01] Set maximize deposit logic is not implemented in deposit function in Kangaroo Vault

There is a `maxDepositAmount` variable declaration and a function to set the amount: `setMaxDepositAmount`. However, in the deposit logic, the checking is missing.

For the impact, there are no risks for the fund in the vault. However, errors could happen when the admin calls `setMaxDepositAmount` from the admin view and assumes the maximum deposit amount is set, but in reality, nothing happens.

https://github.com/code-423n4/2023-03-polynomial/blob/30be22e965b0fdaa64fb30a6fc6d358e40baff8b/src/KangarooVault.sol#L88
https://github.com/code-423n4/2023-03-polynomial/blob/30be22e965b0fdaa64fb30a6fc6d358e40baff8b/src/KangarooVault.sol#L506-L509

Recommended Mitigation Steps: 
- Remove the setMaxDepositAmount function from external interface for avoiding confusion.
- Add logic to check max deposit amount in `initiateDeposit` function:
https://github.com/code-423n4/2023-03-polynomial/blob/30be22e965b0fdaa64fb30a6fc6d358e40baff8b/src/KangarooVault.sol#L183-L208

```diff
require(amount >= minDepositAmount); 
+require(amount <= maxDepositAmount); 
```

# [L-02] Float pragam

Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

Instance:
https://github.com/code-423n4/2023-03-polynomial/blob/30be22e965b0fdaa64fb30a6fc6d358e40baff8b/src/LiquidityToken.sol#L2

Recommended Mitigation Steps: 
- Use fixed solidity version

# [L-03] canCall function in Auth library is not implement

In Auth library, `Authority` contract is required with `cancall` fuction implementation. 
https://github.com/transmissions11/solmate/blob/main/src/auth/Auth.sol

Required implementation:
https://github.com/transmissions11/solmate/blob/1b3adf677e7e383cc684b5d5bd441da86bf4bf1c/src/auth/Auth.sol#L59-L63

Recommended Mitigation Steps: 
- Add `cancall` implementation to alway return false to follow library instruction.

# [NC-01] Unused variable

Variable that are not used in function should be removed

Instance:
https://github.com/code-423n4/2023-03-polynomial/blob/30be22e965b0fdaa64fb30a6fc6d358e40baff8b/src/ShortToken.sol#L9
https://github.com/code-423n4/2023-03-polynomial/blob/30be22e965b0fdaa64fb30a6fc6d358e40baff8b/src/PowerPerp.sol#L9
https://github.com/code-423n4/2023-03-polynomial/blob/30be22e965b0fdaa64fb30a6fc6d358e40baff8b/src/LiquidityToken.sol#L8