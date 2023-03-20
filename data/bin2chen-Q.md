L-01:Exchange._addCollateral() Missing to determine whether the current collateral is still valid, _removeCollateral() and _openTrade() both have checks, it is recommended to also add checks to avoid the collateral is no longer valid
```solidity
   function _addCollateral(uint256 positionId, uint256 amount) internal {
..

        shortToken.adjustPosition(
            positionId,
            msg.sender,
            shortPosition.collateral,
            shortPosition.shortAmount,
            shortPosition.collateralAmount + amount
        );
+       //for check collateral valid
+       shortCollateral.getMinCollateral(shortPosition.shortAmount, shortPosition.collateralAmount + amount);
        shortCollateral.collectCollateral(shortPosition.collateral, positionId, amount);

        emit AddCollateral(msg.sender, positionId, shortPosition.collateral, amount);
    }
```
L-02:processDepositQueue() Each QueuedDeposit re-fetches the price more realistically, and each QueuedDeposit may modify the TokenPrice
like `processWithdrawalQueue()`:

```solidity
    function processDepositQueue(uint256 idCount) external nonReentrant {
-       uint256 tokenPrice = getTokenPrice();

        for (uint256 i = 0; i < idCount; i++) {
+           uint256 tokenPrice = getTokenPrice();        
            QueuedDeposit storage current = depositQueue[queuedDepositHead];

            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
                return;
            }

            uint256 tokensToMint = current.depositedAmount.divWadDown(tokenPrice);

            current.mintedTokens = tokensToMint;
            totalQueuedDeposits -= current.depositedAmount;
            totalFunds += current.depositedAmount;
            VAULT_TOKEN.mint(current.user, tokensToMint);

            emit ProcessDeposit(current.id, current.user, current.depositedAmount, tokensToMint, current.requestedTime);

            current.depositedAmount = 0;
            queuedDepositHead++;
        }
    }
```
L-03 LiquidityPool.minDepositDelay The limit must be greater than 0, to avoid QueuedDeposit and deposit() can be the same as real-time, but do not have to pay depositFee
```solidity
    function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
+       require(_minDepositDelay>0 && _minWithdrawDelay>0,">0");
        emit UpdateDelays(minDepositDelay, _minDepositDelay, minWithdrawDelay, _minWithdrawDelay);
        minDepositDelay = _minDepositDelay;
        minWithdrawDelay = _minWithdrawDelay;
    }
```    


L-04:EXCHANGE.sol#_addCollateral() Remove the restriction on determining whether the MarkPrice is valid or not, because to avoid liquidation, the user should be able to add collateral regardless of the current MarkPrice being temporarily invalid.
```solidity
    function _addCollateral(uint256 positionId, uint256 amount) internal {
        require(positionId != 0);
-       (, bool isInvalid) = getMarkPrice();
-       require(!isInvalid);

```