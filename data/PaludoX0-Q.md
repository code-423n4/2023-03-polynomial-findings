## `function tokenURI(uint256 tokenId)` function can be restricted to pure
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortToken.sol#L44
Please consider changing function to pure  `function tokenURI(uint256) public pure  override returns (string memory)`

## Function argument `tokenId` is not used
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortToken.sol#L44
Please consider to remove it leaving `function tokenURI(uint256) public view override returns (string memory)`

## `shortAmount` should be validated if > 0 
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortToken.sol#L63
It's important that `position.shortAmount` shall be != 0 otherwise it has no senset to open a short. 
In fact there's a check at line 82, that during adjusting position if shortAmount = 0 than the short is closed.

            if (position.shortAmount == 0) {
                _burn(positionId);
            }
If shortAmount is == 0 the positionId token is burnt.
When a new positionId is minted it's suggested to check in line 55 `if(!(shortAmount > 0)) revert();`


## Wrong comments
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L52
The following variables are marked as immutable but they are not.
Please consider to move them in another section.

    /// -----------------------------------------------------------------------
    /// Immutables
    /// -----------------------------------------------------------------------

    /// @notice Power Perp Instance
    IPowerPerp public powerPerp;

    /// @notice Power Perp Exchnage Instance
    IExchange public exchange;

    /// @notice Liquidity Token Instance
    ILiquidityToken public liquidityToken;

## Move variable caching outside for cycle
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L288
`uint256 tokenPrice = getTokenPrice();` can be move in line 286 outside for cycle

    function processWithdraws(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_WITHDRAWS") {
        assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);
        for (uint256 i = 0; i < count; i++) {
            uint256 tokenPrice = getTokenPrice();


## Update variables before token transfers outside the contract or after token transfer to the contract 
Even if `nonReentrant` modifier are applied to the following functions it is suggested to carry out token transfers when it is safer for the contract
src/LiquidityPool.sol#L247: in function `LiquidityPool.withdraw` first carry out `liquidityToken.burn(msg.sender, tokens);` after require statement and then transfer sUSD tokens to user and feeReceipient
src/KangarooVault.sol#L207: in function `KangarooVault.initiateDeposit` first move `SUSD.safeTransferFrom(msg.sender, address(this), amount);`  to line 186 before VAULT_TOKEN are minted.
src/KangarooVault.sol#L238: in function `KangarooVault.initiateWithdrawal`  first carry out `VAULT_TOKEN.burn(msg.sender, tokens);` after require statement and then transfer sUSD tokens to user 


## executePerpOrders function can revert if feeReceipient.call reverts.
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L710
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L456
Function `executePerpOrders` calls Synthetix function `PerpsV2MarketDelayedOrdersOffchain.executeOffchainDelayedOrder` to execute offchain orders by sending some eth to updated offchainOracle.
In case too much eth are sent, the remaining eth is sent back to protocol by PerpsV2ExchangeRate.offchainOracle:

    function updatePythPrice(address sender, bytes[] calldata priceUpdateData)
        external
        payable
        nonReentrant
        onlyAssociatedContracts
    {
        // Get fee amount to pay to Pyth
        uint fee = offchainOracle().getUpdateFee(priceUpdateData);
        require(msg.value >= fee, "Not enough eth for paying the fee");

        // Update the price data (and pay the fee)
        offchainOracle().updatePriceFeeds.value(fee)(priceUpdateData);

        if (msg.value - fee > 0) {
            // Need to refund caller. Try to return unused value, or revert if failed
            // solhint-disable-next-line  avoid-low-level-calls
            (bool success, ) = sender.call.value(msg.value - fee)("");
            require(success, "Failed to refund caller");
        }
    }

The above functions makes the complete call revert in case the calling contract doesn't receive the eth back, therefore there's already this check so that ETH will arrive in contract.

This is the function `receive()` implemented in the contract in KangarooVault#L456 and LiquidityPool.sol#L710
    
    receive() external payable {
        (bool success,) = feeReceipient.call{value: msg.value}("");
        require(success);

        emit ReceiveEther(msg.sender, msg.value);
    }

In my opinion it should be avoided to check `require(success)` because it would make reverting the complete call if feeReceipient reverts.
It's better to take accounting of fee received and sent, and implement a function (triggered by authority only) to send remaining fee to `feeReceipient`



## Missing check of queuedWithdrawalHead integrity with queuedWithdrawalHead
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.solL240 and 270
In `processDepositQueue` and `processWithdrawalQueue` there's no the following check as in the same functions of LiquidityPool #L220 and #L285
    `assert(queuedWithdrawalHead + idCount - 1 < nextQueuedWithdrawalId);`

It's not mandatory because the following if statement is enough to exit from the for cycle because `current.requestedTime == 0` in case the idCount is more than the `queuedWithdrawalHead` allowed

    if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
        return;
    }

But if for any reason this check is skipped  the `queuedWithdrawalHead` or `queuedDepositHead` will be messed up and the functions will be broken, therefore it is suggested to add the assert checks.

