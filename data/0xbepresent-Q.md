1 - The ```UpdateFeeReceipient``` event is not used in ```setFeeReceipient``` function.
==

The [setFeeReceipient()](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L650) function does not emit the [UpdateFeeReceipient](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L944) event. The system monitor will not register those changes.  

2 - The ```KangarooVault.sol.maxDepositAmount``` is not used
==

The [maxDepositAmount](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L88) is not used in the ```KangarooVault.sol``` deposits. The deposits are not limited.


3 - The ```QueuedWithdraw.returnedAmount``` is not correct if the system process the same withdraw request multiple times
==

If the system doesn't have all the funds, it will [return the available funds](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L306) to the user. If the user process his withdraw again, the ```current.returnedAmount``` variable will be overwritten and the last returned value is lost.

The ```returnedAmount``` variable may not reflect the real returned amount.