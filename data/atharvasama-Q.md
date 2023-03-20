# Low and Non-critical issues

## [01] Solmate’s SafeTransferLib doesn’t check whether the ERC20 contract exists
Solmate’s SafeTransferLib, which is often used to interact with non-compliant/unsafe ERC20 tokens, does not check whether the ERC20 contract exists. The following code will not revert in case the token doesn’t exist (yet).

This is stated in the Solmate library: https://github.com/transmissions11/solmate/blob/main/src/utils/SafeTransferLib.sol#L9

[src/Exchange.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol)
```js
8: import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";
 
266:             ERC20(params.collateral).safeTransferFrom(msg.sender, address(this), params.collateralAmount);
 
366:         ERC20(shortPosition.collateral).safeTransferFrom(msg.sender, address(this), amount);
```

[src/KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol) 
```js
10: import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";
 
207:         SUSD.safeTransferFrom(msg.sender, address(this), amount);

222:             SUSD.safeTransfer(user, susdToReturn);

300:                     SUSD.safeTransfer(feeReceipient, withdrawFees);

304:                 SUSD.safeTransfer(current.user, availableFunds);

318:                     SUSD.safeTransfer(feeReceipient, withdrawFees);

322:                 SUSD.safeTransfer(current.user, susdToReturn);

789:         if (fees > 0) SUSD.safeTransfer(feeReceipient, fees);
```

[src/LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol) 
```js
9: import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";

191:         SUSD.safeTransferFrom(msg.sender, feeReceipient, fees);
192:         SUSD.safeTransferFrom(msg.sender, address(this), amountForTokens);

212:         SUSD.safeTransferFrom(msg.sender, address(this), amount);

253:         SUSD.safeTransfer(feeReceipient, fees);
254          SUSD.transfer(user, susdToReturn - fees);

311:                 SUSD.safeTransfer(current.user, availableFunds);

323:                 SUSD.safeTransfer(current.user, susdToReturn);

444:         SUSD.safeTransferFrom(user, address(this), totalCost);

450:         SUSD.safeTransfer(feeReceipient, externalFee);

476:         SUSD.safeTransfer(user, totalCost);

482:         SUSD.safeTransfer(feeReceipient, externalFee);

508:         SUSD.safeTransfer(user, totalCost);

514:         SUSD.safeTransfer(feeReceipient, externalFee);

540:         SUSD.safeTransferFrom(user, address(this), totalCost);

546:         SUSD.safeTransfer(feeReceipient, externalFee);
```

[src/ShortCollateral.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol)
```js
8: import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";

90:         ERC20(collateral).safeTransferFrom(address(exchange), address(this), amount);

113:         ERC20(userCollateral.collateral).safeTransfer(user, amount);

141:         ERC20(userCollateral.collateral).safeTransfer(user, totalCollateralReturned); 
```

## [02] Gas griefing/theft is possible on unsafe external call
return data (bool success,) has to be stored due to EVM architecture, if in a usage like below, ‘out’ and ‘outsize’ values are given (0,0) . Thus, this storage disappears and may come from external contracts a possible Gas griefing/theft problem is avoided

https://twitter.com/pashovkrum/status/1607024043718316032?t=xs30iD6ORWtE2bTTYsCFIQ&s=19

[src/KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol) 
```js
454      receive() external payable {
455:         (bool success,) = feeReceipient.call{value: msg.value}("");
456          require(success);
```

[src/LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol) 
```js
709      receive() external payable {
710:         (bool success,) = feeReceipient.call{value: msg.value}("");
711          require(success);
```

Using assembly instead, we can refactor the code as 

[src/KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol) 
```js
454:     receive() external payable {
455:         assembly {
456:             success := call(gas(), feeReceipient, msg.value, 0, 0)
457:         }
458:         require(success);
459:     }
```

[src/LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol) 
```js
709:     receive() external payable {
710:         assembly {
711:             success := call(gas(), feeReceipient, msg.value, 0, 0)
712:         }
713:         require(success);
```

## [03] init() function can be called by anybody
init() function can be called anybody when the contract is not initialized.

[src/SystemManager.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol) 
```js
62:     function init(
63:         address _pool,
64:         address _powerPerp,
65:         address _exchange,
66:         address _liquidityToken,
67:         address _shortToken,
68:         address _synthetixAdapter,
69:         address _shortCollateral
70:     ) public {
```

### Recommended Mitigation Steps
Add a control that makes init() only call the Deployer Contract or EOA;

```js
if (msg.sender != DEPLOYER_ADDRESS) {
    revert NotDeployer();
}
```

## [04] Unhandled return values of ```transfer``` and ```transferFrom``` 
ERC20/ERC721 implementations are not always consistent. Some implementations of transfer and transferFrom could return ‘false’ on failure instead of reverting. It is safer to wrap such calls into require() statements to these failures, or use OpenZeppelin’s SafeERC20 wrapper functions.

[src/KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol) 
```js
551:         ERC20(token).transfer(receiver, amt);
```

[src/LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol) 
```js
255:         SUSD.transfer(user, susdToReturn - fees);
```

## [05] Two steps verification before transferring ownership
Solmate's Auth.sol transfers ownership in a single step using the ```transferOwnership()``` function. Transferring of ownership should be a two-step process where the first transaction (from the old/current address) registers the new address (i.e. grants ownership) and the second transaction (from the new address) replaces the old address with the new one (i.e. claims ownership). This gives an opportunity to recover from incorrect addresses mistakenly used in the first step. If not, contract functionality might become inaccessible.

The contracts using Auth.sol include

[src/Exchange.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol)

[src/KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol) 

[src/LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol) 

[src/ShortCollateral.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol)

[src/SystemManager.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol) 

## [06] Unsafe Cast
It is more safer to use OpenZeppelin's SafeCast for casting integers. 

[src/KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol) 
```js
408:             uint256 marginWithdrawing = uint256(-marginDelta);

604:         positionData.lastSizeDelta = uint256(int256(position.size));

622:         uint256 currentSize = uint256(int256(position.size));

712:         positionData.lastSizeDelta = uint256(int256(position.size));

739:         uint256 currentSize = uint256(int256(position.size));
```

[src/LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol) 
```js
296:             uint256 availableFunds = uint256(int256(totalFunds) - usedFunds);

363:         totalValue -= uint256((int256(amountOwed) + usedFunds));

517          usedFunds += int256(totalCost + hedgingFees + externalFee);

616          usedFunds += int256(additionalMargin);

811          usedFunds += int256(marginRequired);
```

## [07] SafeApprove of OpenZeppelin is Deprecated
SafeApprove should be changed to 'safeIncreaseAllowance' and 'safeDecreaseAllowance' as recommended by OpenZeppelin.

[src/Exchange.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol)
```js
272:             ERC20(params.collateral).safeApprove(address(shortCollateral), params.collateralAmount);

367:         ERC20(shortPosition.collateral).safeApprove(address(shortCollateral), amount);
```

[src/KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol) 
```js
425:         SUSD.safeApprove(address(EXCHANGE), additionalCollateral);

579:         SUSD.safeApprove(address(EXCHANGE), collateralRequired);

638:             SUSD.safeApprove(address(LIQUIDITY_POOL), maxCost);

702:         SUSD.safeApprove(address(LIQUIDITY_POOL), maxCost);

756:             SUSD.safeApprove(address(EXCHANGE), lastTradeData.collateralAmount);
```