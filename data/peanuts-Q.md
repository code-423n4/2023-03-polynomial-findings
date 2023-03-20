## Table of Contents

- [L-01] KangarooVault#setDelays() should have a maximum time limit
- [L-02] KangarooVault#processDepositQueue() may fail if one transaction fails
- [L-03] KangarooVault#minDepositAmount() should be a fairly high number to prevent 0 deposits or dust deposits that may affect the deposit/withdrawal process
- [L-04] Deprecated safeApprove() function
- [N-01] Lock pragma version
- [N-02] Update pragma version




### [L-01] KangarooVault#setDelays() should have a maximum time limit

If withdrawals/deposits are queued up in KangarooVault, the withdrawals/deposits have to wait for a minimum delay before they can be processed

```
         if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
             return;
         }
```

If delays are too long, then there will be a backlog of deposits/withdrawals to be processed. Consider having a maximum time limit for delays

```
    function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
        emit UpdateDelays(minDepositDelay, _depositDelay, minWithdrawDelay, _withdrawDelay);
     	  require(minDepositDelay < 1 weeks, "Too long of a delay");
	  require(minDepositDelay < 1 weeks, "Too long of a delay");
        minDepositDelay = _depositDelay;
        minWithdrawDelay = _withdrawDelay;
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L514

```
    function processDepositQueue(uint256 idCount) external nonReentrant {
        uint256 tokenPrice = getTokenPrice();


        for (uint256 i = 0; i < idCount; i++) {
            QueuedDeposit storage current = depositQueue[queuedDepositHead];


            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
                return;
            }
```

### [L-02] KangarooVault#processDepositQueue may fail if one transaction fails

When deposits (same for withdrawal) are queued, processDepositQueue() will be called to process the deposits. The parameter takes in the idCount and loops over as many deposits as idCount takes (before hitting gas limits, which is not an issue because the value of idCount can be controlled). If it so happens that one transaction fails, then all the other transactions will fail too, which wastes time because the caller has to know which transaction is the cause for failure (probably due to the `minDepositDelay`. 
 
```
    function processDepositQueue(uint256 idCount) external nonReentrant {
        uint256 tokenPrice = getTokenPrice();


        for (uint256 i = 0; i < idCount; i++) {
            QueuedDeposit storage current = depositQueue[queuedDepositHead];


            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
                return;
            }
```

Consider implementing a try/check and loop over those that fails (make sure the queuedDepositHead is not incremented in this case), and it should be fine because if a certain id fails, the subsequent id will fail because of minDepositDelay.

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L243

### [L-03] KangarooVault#minDepositAmount() should be a fairly high number to prevent 0 deposits or dust deposits that may affect the deposit/withdrawal process

minDepositAmount can be set to 0:

```
    function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
        emit UpdateMinDeposit(minDepositAmount, _minAmt);
        minDepositAmount = _minAmt;
    }
```

or even 1 wei. If thats the case, then an attacker can just populate the deposit queue with dust deposits to prevent deposits from processing due to gas limit.

```
    function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
	  require(_minAmt > 1e17, "dusting not allowed")
        emit UpdateMinDeposit(minDepositAmount, _minAmt);
        minDepositAmount = _minAmt;
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L499-L502

### [L-04] Deprecated safeApprove() function

Using this deprecated function can lead to unintended reverts and potentially the locking of funds. A deeper discussion on the deprecation of this function is in OZ issue #2219 (OpenZeppelin/openzeppelin-contracts#2219). The OpenZeppelin ERC20 safeApprove() function has been deprecated, as seen in the comments of the OpenZeppelin code.

As recommended by the OpenZeppelin comment, I suggest replacing safeApprove() with safeIncreaseAllowance() or safeDecreaseAllowance() instead:

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L425
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L577
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L636
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L700
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L754

### [N-01] Lock pragma version

It is considered best practice to use a locked Solidity version, thereby only allowing compilation with a specific version. 
So the instances of `^0.8.9` should be changed to `0.8.9`.

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L2

### [N-02] Update pragma version


Use a solidity version of at least 0.8.10 to have external calls skip contract existence checks if the external call has a return value. Use a solidity version of at least 0.8.12 to get string.concat() instead of abi.encodePacked(<str>,<str>)

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L2