## Low and Non-Critical Issues Summary
| Number |Issues Details |
|:--:|:-------|
|[L-01]| MISSING EVENT FOR CRITICAL PARAMETER CHANGE
|[NC-01]| Use latest Solidity version
|[NC-02]| Use stable pragma statement
|[NC-03]| Missing or incomplete NatSpec documentation
|[NC-04]| INITIAL VALUE CHECK IS MISSING IN SETTER FUNCTIONS
|[NC-05]| Code Structure Deviates From Best-Practice
|[NC-06]| The nonReentrant modifier should occur before all other modifiers
|[NC-07]| You can use named parameters in mapping types
***

## [L-01] MISSING EVENT FOR CRITICAL PARAMETER CHANGE

When changing state variables events are not emitted. Emitting events allows monitoring activities with off-chain monitoring tools.

```solidity
LiquidityPool.sol

650: function setFeeReceipient(address _feeReceipient) external requiresAuth {
```
***


## [NC-01] Use latest Solidity version

Solidity pragma versioning should be upgraded to latest available version. 
***

## [NC-02] Use stable pragma statement

Using a floating pragma statement `^0.8.9` is discouraged as code can compile to different bytecodes with different compiler versions. Use a stable pragma statement to get a deterministic bytecode.
***

## [NC-03] Missing or incomplete NatSpec documentation

NatSpec documentation to all public/external functions and variables is essential for better understanding of the code by developers and auditors and is strongly recommended.
***

## [NC-04] INITIAL VALUE CHECK IS MISSING IN SETTER FUNCTIONS
Setter functions are missing checks to validate if the new value being set is the same as the current value already set in the contract. 
```solidity
Exchange.sol

216: function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {
223: function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {

LiquidityPool.sol

650: function setFeeReceipient(address _feeReceipient) external requiresAuth {
657: function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
665: function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
672: function setDevFee(uint256 _devFee) external requiresAuth {
679: function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
685: function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {

SystemManager.sol
89: function setStatusFunction(bytes32 key, bool status) public requiresAuth {

KangarooVault.sol
487: function setSynthetixTracking(bytes32 _code) external requiresAuth {
499: function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
506: function setMaxDepositAmount(uint256 _maxAmt) external requiresAuth {
522: function setPriceImpactDelta(uint256 _delta) external requiresAuth {
```
Checking whether the current value and the new value are the same should be added.
***
## [NC-05] Code Structure Deviates From Best-Practice

The best-practice layout for a contract should follow the following order: state variables, events, modifiers, constructor and functions. Function ordering helps readers identify which functions they can call and find constructor and fallback functions easier. Functions should be grouped according to their visibility and ordered as: constructor, receive function (if exists), fallback function (if exists), external, public, internal, private. Functions should then further be ordered with view functions coming after the non-view labeled ones.

The most obvious thing in the contracts is that the events are at the end of the code.

**Recommendation**: Consider adopting recommended best-practice for code structure and layout.
***
## [NC-06] The nonReentrant modifier should occur before all other modifiers
This is a best-practice to protect against re-entrancy in other modifiers.
```solidity
LiquidityPool.sol

692: function placeQueuedOrder() external requiresAuth nonReentrant {
430: function openLong(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant

ShortCollateral.sol

85: function collectCollateral(address collateral, uint256 positionId, uint256 amount)
        external
        onlyExchange
        nonReentrant

KangarooVault.sol

376: function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {
383: function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {
389: function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {
395: function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {
401: function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
424: function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {
436: function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
450: function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```
***
## [NC-07] You can use named parameters in mapping types

From Solidity [0.8.18](https://blog.soliditylang.org/2023/02/01/solidity-0.8.18-release-announcement/) you can use named parameters in mapping types. This will make the code much more readable.