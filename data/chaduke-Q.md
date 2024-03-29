QA1. A range check is needed for both ``_depositFee`` and ``_withdrawalFee``. 

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L657-L661

QA2. setMaxDepositAmount() is used to enforce a maximum deposit amount. 

[https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L506-L509](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L506-L509)

However, ``initiateDeposit()`` fails to enforce the maximum deposit amount: 

[https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L183-L208](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L183-L208)

Mitigation: make sure ``initiateDeposit()`` enforces the maximum deposit amount.


QA3. When adjustPosition() burn a short position ``positionId``, it fails to delete the corresponding entry from mapping ``shortPositions``.

[https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortToken.sol#L48-L85](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortToken.sol#L48-L85)

Mitigation: When ``adjustPosition()`` burns a short position ``positionId``, we also delete the corresponding entry from mapping ``shortPositions``.

