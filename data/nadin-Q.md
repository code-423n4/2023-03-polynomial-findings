## Non-Critical Issues List
### Issue 
## [N-01] Emit an event for critical parameter changes.
## [N-02] CRITICAL ADDRESS CHANGES SHOULD USE TWO-STEP PROCEDURE
## [N-03] It's better to emit after all processing is done
## [N-04] Include return parameters in NatSpec comments
## [N-05] LOCK PRAGMAS TO SPECIFIC COMPILER VERSION
## [N-06] USE Custom Errors INSTEAD OF ASSERT
## [N-07] Function writing that does not comply with the Solidity Style Guide
## [N-08] Function order
## [N-09] Parameter is not in mixedCase
## [N-10] State variables that could be declared immutable
### Total: 10 issues
## [N-01] Emit an event for critical parameter changes.
```
File: src/KangarooVault.sol
401:     function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
...
411:             usedFunds -= marginWithdrawing;
...
415:            usedFunds += marginAdding;
```
- There should be an event to track changes in usedFunds in https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L401-L420

## [N-02] CRITICAL ADDRESS CHANGES SHOULD USE TWO-STEP PROCEDURE
Recommended Mitigation Steps
Lack of two-step procedure for critical operations leaves them error-prone. Consider adding a two- step procedure on the critical functions.
```
File: src/LiquidityPool.sol
651:        feeReceipient = _feeReceipient;
```
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L650-L652
```
File: src/VaultToken.sol
39:        vault = _vault;
```
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/VaultToken.sol#L35-L40

## [N-03] It's better to emit after all processing is done
```
File: src/Exchange.sol
217:        emit UpdateSkewNormalizationFactor(skewNormalizationFactor, _skewNormalizationFactor);
224:        emit UpdateMaxFundingRate(maxFundingRate, _maxFundingRate);
421:        emit UpdateFundingRate(fundingLastUpdated, normalizationFactor);
```

```
File: src/LiquidityPool.sol
237:            emit ProcessDeposit(current.id, current.user, current.depositedAmount, tokensToMint, current.requestedTime);
628:        emit UpdateSynthetixTrackingCode(synthetixTrackingCode, _trackingCode);
636:        emit UpdateLeverage(futuresLeverage, _leverage);
644:        emit UpdateStandardSize(standardSize, _standardSize);
658:        emit UpdateFees(depositFee, _depositFee, withdrawalFee, _withdrawalFee);
666:        emit UpdateBaseTradingFee(baseTradingFee, _baseTradingFee);
673:        emit UpdateDevFee(devFee, _devFee);
680:        emit UpdatePriceImpactDelta(perpPriceImpactDelta, _perpPriceImpactDelta);
686:        emit UpdateDelays(minDepositDelay, _minDepositDelay, minWithdrawDelay, _minWithdrawDelay);
699:        emit SubmitDelayedOrder(queuedPerpSize);
813:        emit TransferMargin(marginRequired);
```

```
File: src/KangarooVault.sol
260:            emit ProcessDeposit(current.id, current.user, current.depositedAmount, tokensToMint, current.requestedTime);
468:        emit UpdateFeeReceipient(feeReceipient, _feeReceipient);
479:        emit UpdateFees(performanceFee, withdrawalFee, _performanceFee, _withdrawalFee);
488:        emit UpdateSynthetixTrackingCode(synthetixTrackingCode, _code);
493:        emit UpdateReferralCode(referralCode, _code);
500:        emit UpdateMinDeposit(minDepositAmount, _minAmt);
507:        emit UpdateMaxDeposit(maxDepositAmount, _maxAmt);
515:        emit UpdateDelays(minDepositDelay, _depositDelay, minWithdrawDelay, _withdrawDelay);
523:        emit UpdatePriceImpactDelta(perpPriceImpact, _delta);
531:        emit UpdateLeverage(leverage, _lev);
539:        emit UpdateCollRatio(collRatio, _ratio);
```

## [N-04] Include return parameters in NatSpec comments
### Context: All contract
### Description
It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation. In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability.

https://docs.soliditylang.org/en/v0.8.15/natspec-format.html
### Recommendation
Include return parameters in NatSpec comments

## [N-05] LOCK PRAGMAS TO SPECIFIC COMPILER VERSION
### Context: All contract
### Description:
Pragma statements can be allowed to float when a contract is intended for consumption by other developers, as in the case with contracts in a library or EthPM package. Otherwise, the developer would need to manually update the pragma in order to compile locally.
https://swcregistry.io/docs/SWC-103
### Recommendation:
Ethereum Smart Contract Best Practices - Lock pragmas to specific compiler version.
solidity-specific/locking-pragmas

## [N-06] USE Custom Errors INSTEAD OF ASSERT
Assert should not be used except for tests, Custom Errors should be used
Prior to Solidity 0.8.0, pressing a confirm consumes the remainder of the process’s available gas instead of returning it, as request()/revert() did.
Assertion() should be avoided even after solidity version 0.8.0, because its documentation states “The Assert function generates an error of type Panic(uint256). Code that works properly should never Panic, even on invalid external input. If this happens, you need to fix it in your contract. there’s a mistake”.
```
File: src/LiquidityPool.sol
220:        assert(queuedDepositHead + count - 1 < nextQueuedDepositId);
285:        assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);
```

## [N-07] Function writing that does not comply with the Solidity Style Guide
### Context: All contract
### Description
Order of Functions; ordering helps readers identify which functions they can call and to find the constructor and fallback definitions easier. But there are contracts in the project that do not comply with this.

https://docs.soliditylang.org/en/v0.8.17/style-guide.html

Functions should be grouped according to their visibility and ordered:

constructor
receive function (if exists)
fallback function (if exists)
external
public
internal
private
within a grouping, place the view and pure functions last

## [N-08] Function order
Functions should be ordered following the Soldiity conventions ( https://docs.soliditylang.org/en/v0.8.15/style-guide.html#order-of-functions): receive() function should be placed after the constructor and before every other function.
```
File: src/LiquidityPool.sol
708:    receive() external payable {
```
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L708-L713
```
File: src/KangarooVault.sol
454:    receive() external payable {
```
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L454-L457

## [N-09] Parameter is not in mixedCase
```
1: Parameter Exchange.setSkewNormalizationFactor(uint256)._skewNormalizationFactor (src/Exchange.sol#216) is not in mixedCase
2: Parameter Exchange.setMaxFundingRate(uint256)._maxFundingRate (src/Exchange.sol#223) is not in mixedCase
3: Variable Exchange.PRICING_CONSTANT (src/Exchange.sol#34) is not in mixedCase
4: Variable Exchange.SUSD (src/Exchange.sol#50) is not in mixedCase
5: Parameter KangarooVault.setFeeReceipient(address)._feeReceipient (src/KangarooVault.sol#465) is not in mixedCase
6: Parameter KangarooVault.setFees(uint256,uint256)._performanceFee (src/KangarooVault.sol#476) is not in mixedCase
7: Parameter KangarooVault.setFees(uint256,uint256)._withdrawalFee (src/KangarooVault.sol#476) is not in mixedCase
8: Parameter KangarooVault.setSynthetixTracking(bytes32)._code (src/KangarooVault.sol#487) is not in mixedCase
9: Parameter KangarooVault.setReferralCode(bytes32)._code (src/KangarooVault.sol#492) is not in mixedCase
10: Parameter KangarooVault.setMinDepositAmount(uint256)._minAmt (src/KangarooVault.sol#499) is not in mixedCase
11: Parameter KangarooVault.setMaxDepositAmount(uint256)._maxAmt (src/KangarooVault.sol#506) is not in mixedCase
12: Parameter KangarooVault.setDelays(uint256,uint256)._depositDelay (src/KangarooVault.sol#514) is not in mixedCase
13: Parameter KangarooVault.setDelays(uint256,uint256)._withdrawDelay (src/KangarooVault.sol#514) is not in mixedCase
14: Parameter KangarooVault.setPriceImpactDelta(uint256)._delta (src/KangarooVault.sol#522) is not in mixedCase
15: Parameter KangarooVault.setLeverage(uint256)._lev (src/KangarooVault.sol#529) is not in mixedCase
16: Parameter KangarooVault.setCollRatio(uint256)._ratio (src/KangarooVault.sol#537) is not in mixedCase
17: Variable KangarooVault.UNDERLYING_SYNTH_KEY (src/KangarooVault.sol#63) is not in mixedCase
18: Variable KangarooVault.SUSD (src/KangarooVault.sol#66) is not in mixedCase
19: Variable KangarooVault.VAULT_TOKEN (src/KangarooVault.sol#69) is not in mixedCase
20: Variable KangarooVault.EXCHANGE (src/KangarooVault.sol#72) is not in mixedCase
21: Variable KangarooVault.LIQUIDITY_POOL (src/KangarooVault.sol#75) is not in mixedCase
22: Variable KangarooVault.PERP_MARKET (src/KangarooVault.sol#78) is not in mixedCase
23: Parameter LiquidityPool.updatedSynthetixTrackingCode(bytes32)._trackingCode (src/LiquidityPool.sol#627) is not in mixedCase
24: Parameter LiquidityPool.updateLeverage(uint256)._leverage (src/LiquidityPool.sol#634) is not in mixedCase
25: Parameter LiquidityPool.updateStandardSize(uint256)._standardSize (src/LiquidityPool.sol#642) is not in mixedCase
26: Parameter LiquidityPool.setFeeReceipient(address)._feeReceipient (src/LiquidityPool.sol#650) is not in mixedCase
27: Parameter LiquidityPool.setFees(uint256,uint256)._depositFee (src/LiquidityPool.sol#657) is not in mixedCase
28: Parameter LiquidityPool.setFees(uint256,uint256)._withdrawalFee (src/LiquidityPool.sol#657) is not in mixedCase
29: Parameter LiquidityPool.setBaseTradingFee(uint256)._baseTradingFee (src/LiquidityPool.sol#665) is not in mixedCase
30: Parameter LiquidityPool.setDevFee(uint256)._devFee (src/LiquidityPool.sol#672) is not in mixedCase
31: Parameter LiquidityPool.setPerpPriceImpactDelta(uint256)._perpPriceImpactDelta (src/LiquidityPool.sol#679) is not in mixedCase
32: Parameter LiquidityPool.setMinDelays(uint256,uint256)._minDepositDelay (src/LiquidityPool.sol#685) is not in mixedCase
33: Parameter LiquidityPool.setMinDelays(uint256,uint256)._minWithdrawDelay (src/LiquidityPool.sol#685) is not in mixedCase
34: Variable LiquidityPool.SUSD (src/LiquidityPool.sol#65) is not in mixedCase
35: Parameter LiquidityToken.mint(address,uint256)._user (src/LiquidityToken.sol#28) is not in mixedCase
36: Parameter LiquidityToken.mint(address,uint256)._amt (src/LiquidityToken.sol#28) is not in mixedCase
37: Parameter LiquidityToken.burn(address,uint256)._user (src/LiquidityToken.sol#32) is not in mixedCase
38: Parameter LiquidityToken.burn(address,uint256)._amt (src/LiquidityToken.sol#32) is not in mixedCase
39: Parameter PowerPerp.mint(address,uint256)._user (src/PowerPerp.sol#32) is not in mixedCase
40: Parameter PowerPerp.mint(address,uint256)._amt (src/PowerPerp.sol#32) is not in mixedCase
41: Parameter PowerPerp.burn(address,uint256)._user (src/PowerPerp.sol#36) is not in mixedCase
42: Parameter PowerPerp.burn(address,uint256)._amt (src/PowerPerp.sol#36) is not in mixedCase
43: Parameter PowerPerp.transfer(address,uint256)._to (src/PowerPerp.sol#40) is not in mixedCase
44: Parameter PowerPerp.transfer(address,uint256)._amount (src/PowerPerp.sol#40) is not in mixedCase
45: Parameter PowerPerp.transferFrom(address,address,uint256)._from (src/PowerPerp.sol#45) is not in mixedCase
46: Parameter PowerPerp.transferFrom(address,address,uint256)._to (src/PowerPerp.sol#45) is not in mixedCase
47: Parameter PowerPerp.transferFrom(address,address,uint256)._amount (src/PowerPerp.sol#45) is not in mixedCase
48: Parameter ShortToken.transferFrom(address,address,uint256)._from (src/ShortToken.sol#92) is not in mixedCase
49: Parameter ShortToken.transferFrom(address,address,uint256)._to (src/ShortToken.sol#92) is not in mixedCase
50: Parameter ShortToken.transferFrom(address,address,uint256)._id (src/ShortToken.sol#92) is not in mixedCase
51: Parameter ShortToken.safeTransferFrom(address,address,uint256)._from (src/ShortToken.sol#97) is not in mixedCase
52: Parameter ShortToken.safeTransferFrom(address,address,uint256)._to (src/ShortToken.sol#97) is not in mixedCase
53: Parameter ShortToken.safeTransferFrom(address,address,uint256)._id (src/ShortToken.sol#97) is not in mixedCase
54: Parameter ShortToken.safeTransferFrom(address,address,uint256,bytes)._from (src/ShortToken.sol#102) is not in mixedCase
55: Parameter ShortToken.safeTransferFrom(address,address,uint256,bytes)._to (src/ShortToken.sol#102) is not in mixedCase
56: Parameter ShortToken.safeTransferFrom(address,address,uint256,bytes)._id (src/ShortToken.sol#102) is not in mixedCase
57: Parameter SystemManager.init(address,address,address,address,address,address,address)._pool (src/SystemManager.sol#63) is not in mixedCase
58: Parameter SystemManager.init(address,address,address,address,address,address,address)._powerPerp (src/SystemManager.sol#64) is not in mixedCase
59: Parameter SystemManager.init(address,address,address,address,address,address,address)._exchange (src/SystemManager.sol#65) is not in mixedCase
60: Parameter SystemManager.init(address,address,address,address,address,address,address)._liquidityToken (src/SystemManager.sol#66) is not in mixedCase
61: Parameter SystemManager.init(address,address,address,address,address,address,address)._shortToken (src/SystemManager.sol#67) is not in mixedCase
62: Parameter SystemManager.init(address,address,address,address,address,address,address)._synthetixAdapter (src/SystemManager.sol#68) is not in mixedCase
63: Parameter SystemManager.init(address,address,address,address,address,address,address)._shortCollateral (src/SystemManager.sol#69) is not in mixedCase
64: Variable SystemManager.SUSD (src/SystemManager.sol#24) is not in mixedCase
65: Variable SystemManager.PERP_MARKET_CONTRACT (src/SystemManager.sol#27) is not in mixedCase
66: Parameter VaultToken.mint(address,uint256)._user (src/VaultToken.sol#27) is not in mixedCase
67: Parameter VaultToken.mint(address,uint256)._amt (src/VaultToken.sol#27) is not in mixedCase
68: Parameter VaultToken.burn(address,uint256)._user (src/VaultToken.sol#31) is not in mixedCase
69: Parameter VaultToken.burn(address,uint256)._amt (src/VaultToken.sol#31) is not in mixedCase
70: Parameter VaultToken.setVault(address)._vault (src/VaultToken.sol#35) is not in mixedCase
```

## [N-10] State variables that could be declared immutable
State variables that are not updated following deployment should be declared immutable to save gas.
```
File: src/utils/PauseModifier
8:    ISystemManager public systemManager;
```
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/utils/PauseModifier.sol#L8

```
File: src/SynthetixAdapter
10:    ISynthetix public synthetix;
11:    IExchangeRates public exchangeRates;
```
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/SynthetixAdapter.sol#L10-L11

```
File: src/SystemManager
30:    IAddressResolver public addressResolver;
31:    IFuturesMarketManager public futuresMarketManager;
```
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/SystemManager.sol#L30-L31 


## Suggestion List
### Issue 
## [S-01] GENERATE PERFECT CODE HEADERS EVERY TIME
## [S-02] Use nested if and, avoid multiple check combinations
## [S-03] Use descriptive names for Contracts and Libraries
### Total: 03  issues
## [S-01] GENERATE PERFECT CODE HEADERS EVERY TIME
### Description:
I recommend using header for Solidity code layout and readability
https://github.com/transmissions11/headers
```
/*//////////////////////////////////////////////////////////////
                           TESTING 123
//////////////////////////////////////////////////////////////*/
```

## [S-02] Use nested if and, avoid multiple check combinations
### Description:
Using nested is cheaper than using && multiple check combinations. There are more advantages, such as easier to read code and better coverage reports.
```
File: https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol
```

## [S-03] Use descriptive names for Contracts and Libraries
This codebase will be difficult to navigate, as there are no descriptive naming conventions that specify which files should contain meaningful logic.

Prefixes should be added like this by filing:

Interface I_
absctract contracts Abs_
Libraries Lib_
We recommend that you implement this or a similar agreement.