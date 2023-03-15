## 1. Use a more recent version of Solidity

- Use a solidity version of at least 0.8.10 to have external calls skip contract existence checks if the external call has a return value
- Use a solidity version of at least 0.8.12 to get `string.concat()` to be used instead of `abi.encodePacked(<str>,<str>)`
- Use a solidity version of at least 0.8.13 to get the ability to use `using for` with a list of free functions
- Use a solidity version of 0.8.15 where conditions necessary for inlining are relaxed. It shows significant dicrease in the bytecode size (and thus reduced deployment cost)
- Use a solidity version of 0.8.17 to prevent the incorrect removal of storage writes before calls to Yul functions that conditionally terminate the external EVM call; Simplify the starting offset of zero-length operations to zero; More efficient overflow checks for multiplication

```
All contracts
```

## 2. Add a timelock to critical functions

It is a good practice to give time for users to react and adjust to critical changes. A timelock provides more guarantees and reduces the level of trust required, thus decreasing risk for users. It also indicates that the project is legitimate (less risk of a malicious owner making a sandwich attack on a user).

```
216      function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {
223      function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol
```
634	function updateLeverage(uint256 _leverage) external requiresAuth {
657      function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
665      function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
672      function setDevFee(uint256 _devFee) external requiresAuth {
679      function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
685      function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol
```
476      function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
514      function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol


## 3. Initial value check is missing in Set Functions

Checking whether the current value and the new value are the same should be added.

```
216      function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {
223      function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol
```
657      function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
665      function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
672      function setDevFee(uint256 _devFee) external requiresAuth {
679      function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
685      function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol
```
476      function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
499      function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
506      function setMaxDepositAmount(uint256 _maxAmt) external requiresAuth {
514      function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol


## 4. Lack of event emission after critical `initialize()` function

To record the init parameters for off-chain monitoring and transparency reasons, please consider emitting an event after the `initialize()` function.

```
62	function init(
63		address _pool,
64		address _powerPerp,
65		address _exchange,
66		address _liquidityToken,
67		address _shortToken,
68		address _synthetixAdapter,
69		address _shortCollateral
70	) public {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol


## 5. Mark visibility of init(…)/initialize() functions as `external`

External instead of public would give more the sense of the `initialize(…)` functions to behave like a constructor (only called on deployment, so should only be called externally)

From a security point of view, it might be safer so that it cannot be called internally by accident in the child contract.

It might cost a bit less gas to use external over public.

It is possible to override a function from external to public, but it is not possible to override a function from public to external

For above reasons you can change `initialize(…)` to external

```
62	function init(
63		address _pool,
64		address _powerPerp,
65		address _exchange,
66		address _liquidityToken,
67		address _shortToken,
68		address _synthetixAdapter,
69		address _shortCollateral
70	) public {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol


## 6. The `nonReentrant modifier` should occur before all other modifiers

This is a best-practice to protect against reentrancy in other modifiers.

```
430	function openLong(uint256 amount, address user, bytes32 referralCode)
462	function closeLong(uint256 amount, address user, bytes32 referralCode)
494	function openShort(uint256 amount, address user, bytes32 referralCode)
526	function closeShort(uint256 amount, address user, bytes32 referralCode)
557	function liquidate(uint256 amount) external override onlyExchange nonReentrant {
568	function hedgePositions() external override requiresAuth nonReentrant {
591	function rebalanceMargin(int256 marginDelta) external requiresAuth nonReentrant {
613	function increaseMargin(uint256 additionalMargin) external requiresAuth nonReentrant {
692	function placeQueuedOrder() external requiresAuth nonReentrant {
704	function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol
```
376      function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {
383      function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {
389      function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {
395      function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {
401      function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
424      function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {
436      function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
450      function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol
```
85	function collectCollateral(address collateral, uint256 positionId, uint256 amount)
106	function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {
121	function liquidate(uint256 positionId, uint256 debt, address user)
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol


## 7. `require()`/`revert()` statements should have descriptive reason strings

```
240      require(totalCost <= maxCost);
257      require(params.collateral == shortPosition.collateral);
277      require(totalCost >= minCost);
294      require(totalCost >= minCost);
304      require(shortToken.ownerOf(params.positionId) == msg.sender);
308      require(shortPosition.collateral == params.collateral);
322      require(totalCost <= maxCost);
335      require(maxDebtRepayment > 0);
357      require(positionId != 0);
360      require(!isInvalid);
362      require(shortToken.ownerOf(positionId) == msg.sender);
383      require(positionId != 0);
386      require(!isInvalid);
388      require(shortToken.ownerOf(positionId) == msg.sender);
394      require(shortPosition.collateralAmount - amount >= minCollateral);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol
```
184      require(user != address(0x0));
185      require(amount >= minDepositAmount);
216      require(user != address(0x0));
354      require(!isInvalid);
356      require(!isInvalid);
406      require(!isInvalid && baseAssetPrice != 0);
409      require(positionData.totalMargin >= minMargin + marginWithdrawing);
441      require(positionData.totalCollateral >= minColl + collateralToRemove);
456      require(success);
466      require(_feeReceipient != address(0x0));
477      require(_performanceFee <= 1e17 && _withdrawalFee <= 1e16);
530      require(_lev <= 5e18 && _lev >= 1e18);
538      require(_ratio >= 1.5e18);
548      require(token != address(SUSD));
557      require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);
560      require(delayedOrder.sizeDelta == 0);
563      require(position.size >= 0);
566      require(!isInvalid && baseAssetPrice != 0);
575      require(usedFunds + marginRequired + collateralRequired < totalFunds);
614      require(positionData.pendingLongPerp > 0 && positionData.pendingShortPerp == 0);
617      require(delayedOrder.sizeDelta == 0);
624      require(lastTradeData.isOpen);
634      require(totalFunds + positionData.premiumCollected > usedFunds + maxCost);
671      require(positionData.positionId != 0 && positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);
674      require(delayedOrder.sizeDelta == 0);
677      require(position.size >= 0);
731      require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp > 0);
734      require(delayedOrder.sizeDelta == 0);
741      require(!lastTradeData.isOpen);
752      require(usedFunds + lastTradeData.collateralAmount < totalFunds);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol
```
173      require(msg.sender == address(exchange));
248      require(liquidityToken.balanceOf(msg.sender) >= tokens);
270      require(liquidityToken.balanceOf(msg.sender) >= tokens);
353      require(!isInvalid);
386      require(!isInvalid);
438      require(!isInvalid);
470      require(!isInvalid);
502      require(!isInvalid);
517      require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
534      require(!isInvalid);
559      require(!isInvalid);
616      require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
635      require(_leverage >= 1e18);
643      require(_standardSize >= 1e18);
695      require(order.sizeDelta == 0);
710      require(success);
731      require(!isInvalid || spotPrice > 0);
757      int256 requiredPosition = wadMul(skew, int256(delta));
767      require(!isInvalid || spotPrice > 0);
805      require(!isInvalid);
809      require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol
```
24       require(msg.sender == liquidityPool);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol
```
11       require(systemManager.isPaused(key) == false);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/utils/PauseModifier.sol
```
28       require(msg.sender == exchange);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol
```
54       require(susdRatio > susdLiqRatio);
73       require(msg.sender == address(exchange));
160      require(currencyKey != "");
162      require(coll.isApproved);
164      require(!isInvalid);
167      require(!isInvalid);
183      require(currencyKey != "");
185      require(coll.isApproved);
194      require(!isInvalid);
197      require(position.shortAmount > 0);
201      require(collateral.isApproved);
205      require(!isInvalid);
217      require(!isInvalid);
220      require(position.shortAmount > 0);
224      require(collateral.isApproved);
228      require(!isInvalid);
253      requiresAuth
257      require(synth != address(0x0));
258      require(liqRatio < collateralRatio);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol
```
39       require(msg.sender == exchange);
69       require(trader == ownerOf(positionId));
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol
```
71       require(!isInitialized);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol


## 8. `require()` should be used instead of `assert()`

Prior to solidity version 0.8.0, hitting an assert consumes the remainder of the transaction’s available gas rather than returning it, as `require()`/`revert()` do. `assert()` should be avoided even past solidity version 0.8.0 as its [documentation](https://docs.soliditylang.org/en/v0.8.14/control-structures.html#panic-via-assert-and-error-via-require) states that “The assert function creates an error of type Panic(uint256). … Properly functioning code should never create a Panic, not even on invalid external input. If this happens, then there is a bug in your contract which you should fix”.

```
220      assert(queuedDepositHead + count - 1 < nextQueuedDepositId);
285      assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol


## 9. Avoid nested `if` statements

For better readability and analysis it is better to avoid nested `if` blocks if possible.

```
234	if (params.isLong) {
        ...
246	} else {
                ...       
255		if (params.positionId != 0) {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol
```
55	if (positionId == 0) {
        ...
68	} else {
                ...
73		if (shortAmount >= position.shortAmount) {
                      ....
82		if (position.shortAmount == 0) {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol
```
289	if (susdToReturn > availableFunds) {
                ...
298		if (withdrawalFee > 0) {
...
623	if (pendingOrderCancelled) {
                ...
639		if (totalCost > positionData.premiumCollected) {
                        ...
658		if (positionData.shortAmount == 0) {
...
740	if (pendingOrderCancelled) {
                ...
774		if (positionData.shortAmount == 0) {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol


## 10. NatSpec comments should be increased in contracts

It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation.

https://docs.soliditylang.org/en/v0.8.19/natspec-format.html

In most contracts in scope, there is no NapSpec at all. And when there is NatSpec it is missing some tags like @author and @title

