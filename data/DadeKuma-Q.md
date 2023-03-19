
## Summary

---

### Low Risk Findings
|Id|Title|Instances|
|:--:|:-------|:--:|
|[L-01]| Missing valid `tokenURI` in `ShortToken` | 1 |
|[L-02]| There is no way to disapprove a collateral | 1 |
|[L-03]| Redundant checks should be removed to save gas | 1 |
|[L-04]| `Authority` is never used as it's always assigned to address zero | 5 |
|[L-05]| `PauseModifier` is not used in `KangarooVault` | 1 |
|[L-06]| Missing events in sensitive functions | 2 |

Total: 11 instances over 6 issues.

### Non Critical Findings
|Id|Title|Instances|
|:--:|:-------|:--:|
|[NC-01]| Use `require` instead of `assert` | 2 |
|[NC-02]| `require` without any message | 87 |
|[NC-03]| Parameter omission in events | 1 |
|[NC-04]| Use of floating pragma | 26 |
|[NC-05]| Missing or incomplete NatSpec | 47 |

Total: 163 instances over 5 issues.

## Low Risk Findings

---

### [L-01] Missing valid `tokenURI` in `ShortToken`


Some functionality is missing as the `tokenURI` is not implemented, remember to add it before deploying.


*There is 1 instance of this issue.*


```solidity
File: src/ShortToken.sol

44: 		    function tokenURI(uint256 tokenId) public view override returns (string memory) {
45: 		        return "";
46: 		    }

```

### [L-02] There is no way to disapprove a collateral


If a collateral is added by mistake, or a collateral shouldn't be supported in the future, there is no way to remove it.


*There is 1 instance of this issue.*


```solidity
File: src/ShortCollateral.sol

251: 		    function approveCollateral(bytes32 key, uint256 collateralRatio, uint256 liqRatio, uint256 liqBonus)
252: 		        external
253: 		        requiresAuth
254: 		    {
255: 		        Collateral storage collateral = collaterals[key];
256: 		        address synth = synthetixAdapter.getSynth(key);
257: 		        require(synth != address(0x0));
258: 		        require(liqRatio < collateralRatio);
259: 		
260: 		        collateral.currencyKey = key;
261: 		        collateral.synth = synth;
262: 		        collateral.isApproved = true;
263: 		        collateral.collateralRatio = collateralRatio;
264: 		        collateral.liqRatio = liqRatio;
265: 		        collateral.liqBonus = liqBonus;
266: 		
267: 		        emit ApproveCollateral(key, collateralRatio, liqRatio, liqBonus);
268: 		    }

```

### [L-03] Redundant checks should be removed to save gas


There is a check to ensure that the balance is high enough, but the burn would fail if the `amount` isn't enough. Remove these checks to save gas.


*There is 1 instance of this issue.*


```solidity
File: src/LiquidityPool.sol

248: 		        require(liquidityToken.balanceOf(msg.sender) >= tokens);
270: 		        require(liquidityToken.balanceOf(msg.sender) >= tokens);

```

### [L-04] `Authority` is never used as it's always assigned to address zero


Most contracts extends `Auth`, but the `Authority` is always void, as it's always assigned to the address zero. If this isn't a mistake, consider using `Owned` instead of `Auth`, as it's a lighter contract.


*There are 5 instances of this issue.*


```solidity
File: src/Exchange.sol

68: 		        Auth(msg.sender, Authority(address(0x0)))


File: src/KangarooVault.sol

164: 		    ) Auth(msg.sender, Authority(address(0x0))) {


File: src/LiquidityPool.sol

154: 		        Auth(msg.sender, Authority(address(0x0)))


File: src/ShortCollateral.sol

52: 		        Auth(msg.sender, Authority(address(0x0)))


File: src/SystemManager.sol

53: 		    ) Auth(msg.sender, Authority(address(0x0))) {

```

### [L-05] `PauseModifier` is not used in `KangarooVault`


`KangarooVault` extends `PauseModifier`, but it's not used anywhere inside the contract. If this isn't a mistake, remove the import and inheritance to save gas.


*There is 1 instance of this issue.*


```solidity
File: src/KangarooVault.sol

21: 		contract KangarooVault is Auth, ReentrancyGuard, PauseModifier {

```

### [L-06] Missing events in sensitive functions

Events should be emitted when sensitive changes are made to the contracts, but some functions lack them.

*There are 2 instances of this issue.*


```solidity
File: src/LiquidityPool.sol

650: 		    function setFeeReceipient(address _feeReceipient) external requiresAuth {
651: 		        feeReceipient = _feeReceipient;
652: 		    }


File: src/VaultToken.sol

35: 		    function setVault(address _vault) external {
36: 		        if (vault != address(0x0)) {
37: 		            revert();
38: 		        }
39: 		        vault = _vault;
40: 		    }

```

## Non Critical Findings

---

### [NC-01] Use `require` instead of `assert`

Prior to Solidity 0.8.0, the big difference between the `assert` and `require` is that the former uses up all the remaining gas and reverts all the changes made when the predicate is false.

It should be avoided even after Solidity version 0.8.0 as the documentation states:

> Properly functioning code should never create a Panic, not even on invalid external input. If this happens, then there is a bug in your contract which you should fix.

*There are 2 instances of this issue.*


```solidity
File: src/LiquidityPool.sol

220: 		        assert(queuedDepositHead + count - 1 < nextQueuedDepositId);

285: 		        assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);

```

### [NC-02] `require` without any message

If a transaction reverts, it can be confusing to debug if there aren't any messages. Add a descriptive message to all require statements.

*There are 87 instances of this issue.*

<details>
<summary>Expand findings</summary>


```solidity
File: src/Exchange.sol

240: 		            require(totalCost <= maxCost);

257: 		                require(params.collateral == shortPosition.collateral);

277: 		            require(totalCost >= minCost);

294: 		            require(totalCost >= minCost);

304: 		            require(shortToken.ownerOf(params.positionId) == msg.sender);

308: 		            require(shortPosition.collateral == params.collateral);

322: 		            require(totalCost <= maxCost);

335: 		        require(maxDebtRepayment > 0);

357: 		        require(positionId != 0);

360: 		        require(!isInvalid);

362: 		        require(shortToken.ownerOf(positionId) == msg.sender);

383: 		        require(positionId != 0);

386: 		        require(!isInvalid);

388: 		        require(shortToken.ownerOf(positionId) == msg.sender);

394: 		        require(shortPosition.collateralAmount - amount >= minCollateral);


File: src/KangarooVault.sol

184: 		        require(user != address(0x0));

185: 		        require(amount >= minDepositAmount);

216: 		        require(user != address(0x0));

354: 		        require(!isInvalid);

356: 		        require(!isInvalid);

406: 		            require(!isInvalid && baseAssetPrice != 0);

409: 		            require(positionData.totalMargin >= minMargin + marginWithdrawing);

441: 		        require(positionData.totalCollateral >= minColl + collateralToRemove);

456: 		        require(success);

466: 		        require(_feeReceipient != address(0x0));

477: 		        require(_performanceFee <= 1e17 && _withdrawalFee <= 1e16);

530: 		        require(_lev <= 5e18 && _lev >= 1e18);

538: 		        require(_ratio >= 1.5e18);

548: 		        require(token != address(SUSD));

557: 		        require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);

560: 		        require(delayedOrder.sizeDelta == 0);

563: 		        require(position.size >= 0);

566: 		        require(!isInvalid && baseAssetPrice != 0);

575: 		        require(usedFunds + marginRequired + collateralRequired < totalFunds);

614: 		        require(positionData.pendingLongPerp > 0 && positionData.pendingShortPerp == 0);

617: 		        require(delayedOrder.sizeDelta == 0);

624: 		            require(lastTradeData.isOpen);

634: 		            require(totalFunds + positionData.premiumCollected > usedFunds + maxCost);

671: 		        require(positionData.positionId != 0 && positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);

674: 		        require(delayedOrder.sizeDelta == 0);

677: 		        require(position.size >= 0);

731: 		        require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp > 0);

734: 		        require(delayedOrder.sizeDelta == 0);

741: 		            require(!lastTradeData.isOpen);

752: 		            require(usedFunds + lastTradeData.collateralAmount < totalFunds);


File: src/LiquidityPool.sol

173: 		        require(msg.sender == address(exchange));

248: 		        require(liquidityToken.balanceOf(msg.sender) >= tokens);

270: 		        require(liquidityToken.balanceOf(msg.sender) >= tokens);

353: 		        require(!isInvalid);

386: 		        require(!isInvalid);

438: 		        require(!isInvalid);

470: 		        require(!isInvalid);

502: 		        require(!isInvalid);

517: 		        require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));

534: 		        require(!isInvalid);

559: 		        require(!isInvalid);

616: 		        require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));

635: 		        require(_leverage >= 1e18);

643: 		        require(_standardSize >= 1e18);

695: 		        require(order.sizeDelta == 0);

710: 		        require(success);

731: 		        require(!isInvalid || spotPrice > 0);

767: 		        require(!isInvalid || spotPrice > 0);

805: 		        require(!isInvalid);

809: 		        require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));


File: src/LiquidityToken.sol

24: 		        require(msg.sender == liquidityPool);


File: src/PowerPerp.sol

28: 		        require(msg.sender == exchange);


File: src/ShortCollateral.sol

54: 		        require(susdRatio > susdLiqRatio);

73: 		        require(msg.sender == address(exchange));

162: 		        require(coll.isApproved);

164: 		        require(!isInvalid);

167: 		        require(!isInvalid);

185: 		        require(coll.isApproved);

194: 		        require(!isInvalid);

197: 		        require(position.shortAmount > 0);

201: 		        require(collateral.isApproved);

205: 		        require(!isInvalid);

217: 		        require(!isInvalid);

220: 		        require(position.shortAmount > 0);

224: 		        require(collateral.isApproved);

228: 		        require(!isInvalid);

257: 		        require(synth != address(0x0));

258: 		        require(liqRatio < collateralRatio);


File: src/ShortToken.sol

39: 		        require(msg.sender == exchange);

69: 		            require(trader == ownerOf(positionId));


File: src/SystemManager.sol

71: 		        require(!isInitialized);


File: src/utils/PauseModifier.sol

11: 		        require(systemManager.isPaused(key) == false);

```
</details>

---
### [NC-03] Parameter omission in events

Events are generally emitted when sensitive changes are made to the contracts.

However, some are missing important parameters, as they should include both the new value and old value where possible.

*There is 1 instance of this issue.*


```solidity
File: src/SystemManager.sol

92: 		        emit SetStatus(key, status);

```

### [NC-04] Use of floating pragma

Locking the pragma helps avoid accidental deploys with an outdated compiler version that may introduce bugs and unexpected vulnerabilities.

Floating pragma is meant to be used for libraries and contracts that have external users and need backward compatibility.

*There are 26 instances of this issue.*

<details>
<summary>Expand findings</summary>


```solidity
File: src/Exchange.sol

2: 		pragma solidity ^0.8.9;


File: src/KangarooVault.sol

2: 		pragma solidity ^0.8.9;


File: src/LiquidityPool.sol

2: 		pragma solidity ^0.8.9;


File: src/LiquidityToken.sol

2: 		pragma solidity ^0.8.9;


File: src/PowerPerp.sol

2: 		pragma solidity ^0.8.9;


File: src/ShortCollateral.sol

2: 		pragma solidity ^0.8.9;


File: src/ShortToken.sol

2: 		pragma solidity ^0.8.9;


File: src/SynthetixAdapter.sol

2: 		pragma solidity ^0.8.9;


File: src/SystemManager.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/IExchange.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/ILiquidityPool.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/ILiquidityToken.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/IPowerPerp.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/IShortCollateral.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/IShortToken.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/ISynthetixAdapter.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/ISystemManager.sol

2: 		pragma solidity ^0.8.9;


File: src/utils/PauseModifier.sol

3: 		pragma solidity ^0.8.9;


File: src/interfaces/synthetix/IAddressResolver.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/synthetix/IExchangeRates.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/synthetix/IFuturesMarket.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/synthetix/IFuturesMarketBaseTypes.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/synthetix/IFuturesMarketManager.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/synthetix/IPerpsV2Market.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/synthetix/IPerpsV2MarketBaseTypes.sol

2: 		pragma solidity ^0.8.9;


File: src/interfaces/synthetix/ISynthetix.sol

2: 		pragma solidity ^0.8.9;

```
</details>

---
### [NC-05] Missing or incomplete NatSpec

Some functions miss a NatSpec, or they have an incomplete one.
It is recommended that contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI).

Include either `@notice` and/or `@dev` to describe how the function is supposed to work, a `@param` to describe each parameter, and a `@return` for any return values.

*There are 47 instances of this issue.*

<details>
<summary>Expand findings</summary>


```solidity
File: src/Exchange.sol

74: 		    function refresh() public {

86: 		    /// @param tradeParams Check IExchange.TradeParams
87: 		    function openTrade(TradeParams memory tradeParams)

99: 		    /// @param tradeParams Check IExchange.TradeParams
100: 		    function closeTrade(TradeParams memory tradeParams)

113: 		    /// @param amount Amount of additional collateral being added
114: 		    function addCollateral(uint256 positionId, uint256 amount)

126: 		    /// @param amount Amount of collateral being removed
127: 		    function removeCollateral(uint256 positionId, uint256 amount)

139: 		    /// @param debtRepaying Amount of power perp being repaid by the liquidator
140: 		    function liquidate(uint256 positionId, uint256 debtRepaying)


File: src/KangarooVault.sol

182: 		    /// @param amount Amount of sUSD being depositted
183: 		    function initiateDeposit(address user, uint256 amount) external nonReentrant {

214: 		    /// @param tokens Amounts of VAULT_TOKENs being requested to burn / withdraw
215: 		    function initiateWithdrawal(address user, uint256 tokens) external nonReentrant {

375: 		    /// @param minCost Minimum premium to receive
376: 		    function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {

382: 		    /// @param maxCost Maximum premium to close the position
383: 		    function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {

449: 		    /// @notice Executed delayed orders
450: 		    function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {


File: src/LiquidityPool.sol

163: 		    function refresh() public {

183: 		    /// @param user Address of the user to receive liquidity tokens
184: 		    function deposit(uint256 amount, address user) external override nonReentrant whenNotPaused("POOL_DEPOSIT") {

199: 		    /// @param user Address of the user to receive liquidity tokens
200: 		    function queueDeposit(uint256 amount, address user)

246: 		    /// @param user Address of the user to receive sUSD
247: 		    function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {

263: 		    /// @param user Address of the user to receive sUSD
264: 		    function queueWithdraw(uint256 tokens, address user)

429: 		    /// @param user Address of the user
430: 		    function openLong(uint256 amount, address user, bytes32 referralCode)

461: 		    /// @param user Address of the user
462: 		    function closeLong(uint256 amount, address user, bytes32 referralCode)

493: 		    /// @param user Address of the user
494: 		    function openShort(uint256 amount, address user, bytes32 referralCode)

525: 		    /// @param user Address of the user
526: 		    function closeShort(uint256 amount, address user, bytes32 referralCode)

656: 		    /// @param _withdrawalFee New Withdrawal Fee
657: 		    function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {

684: 		    /// @notice Update delays for deposit and withdraw
685: 		    function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {

704: 		    function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {


File: src/LiquidityToken.sol

19: 		    function refresh() public {

28: 		    function mint(address _user, uint256 _amt) public onlyPool {

32: 		    function burn(address _user, uint256 _amt) public onlyPool {


File: src/PowerPerp.sol

22: 		    function refresh() public {

32: 		    function mint(address _user, uint256 _amt) public onlyExchange {

36: 		    function burn(address _user, uint256 _amt) public onlyExchange {

40: 		    function transfer(address _to, uint256 _amount) public override returns (bool) {

45: 		    function transferFrom(address _from, address _to, uint256 _amount) public override returns (bool) {


File: src/ShortCollateral.sol

64: 		    function refresh() public {

84: 		    /// @param amount Amount of collateral
85: 		    function collectCollateral(address collateral, uint256 positionId, uint256 amount)

105: 		    /// @param amount Amount of collateral being returned
106: 		    function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {

120: 		    /// @param user Address of the liquidator
121: 		    function liquidate(uint256 positionId, uint256 debt, address user)

250: 		    /// @param liqBonus Liquidation bonus
251: 		    function approveCollateral(bytes32 key, uint256 collateralRatio, uint256 liqRatio, uint256 liqBonus)


File: src/ShortToken.sol

33: 		    function refresh() public {

48: 		    function adjustPosition(

92: 		    function transferFrom(address _from, address _to, uint256 _id) public override {

97: 		    function safeTransferFrom(address _from, address _to, uint256 _id) public override {

102: 		    function safeTransferFrom(address _from, address _to, uint256 _id, bytes calldata data) public override {


File: src/SystemManager.sol

62: 		    function init(

84: 		    function refreshSynthetixAddresses() public {

89: 		    function setStatusFunction(bytes32 key, bool status) public requiresAuth {


File: src/VaultToken.sol

27: 		    function mint(address _user, uint256 _amt) external onlyVault {

31: 		    function burn(address _user, uint256 _amt) external onlyVault {

35: 		    function setVault(address _vault) external {

```
</details>

---