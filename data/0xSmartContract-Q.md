## Summary
### Issues List
| Number |Issues Details|Context|
|:--:|:-------|:--:|
|[L-01]|Prevent division by 0| 5 |
|[L-02]|Front running attacks by the `requiresAuth`  | 1 |
|[L-03]|Insufficient coverage  | All Contracts |
|[L-04]|Project Upgrade and Stop Scenario should be|All Contracts |
|[L-05]|Add to Event-Emit for critical functions| 2 |
|[L-06]|Update codes to avoid Compile Errors  | 2 |
|[L-07]|` processWithdrawalQueue()` event is missing parameters| 1 |
|[L-08]|In the `constructor` , there is no return of incorrect address identification| 1 |
|[L-09]|Critical Address Changes Should Use Two-step Procedure| 1 |
|[L-10]|The nonReentrant modifier should occur before all other modifiers| 21 |
|[N-11] |Tokens accidentally sent to the contract cannot be recovered|1|
|[N-12] |Floating pragma| 11 |
|[N-13] |Use SMTChecker| All Contracts |
|[N-14] |Constants on the left are better|9|
|[N-15] |`constants` should be defined rather than using magic numbers | 1 |
|[N-16] |Use the `delete` keyword instead of assigning a value of `0`| 13 |
|[N-17] |Function Calls in Loop Could Lead to Denial of Service due to Array length not being checked| 2 |
|[N-18] |Take advantage of Custom Error's return value property | 1 |
|[N-19] |Test environment comments and codes should not be in the main version| 1 |
|[N-20] |Starting with default values is confusing | 1 |
|[N-21] |Add NatSpec Mapping comment| 8 |
|[N-22] |Unnecessary Emission of block.timestamp | 1 |
|[N-23] |`Function writing` that does not comply with the `Solidity Style Guide` | All Contracts |

Total 23 issues

### [L-01] Prevent division by 0

On several locations in the code precautions are not being taken for not dividing by 0, this will revert the code.
These functions can be calledd with 0 value in the input, this value is not checked for being bigger than 0, that means in some scenarios this can potentially trigger a division by zero.

`PRICING_CONSTANT` value defined in constructor (it can be 0 and hasn’t any 0 value check)

```solidity

5 results - 2 files

src/Exchange.sol:
  66: 
  67:     constructor(uint256 _priceNormalizationFactor, ISystemManager _systemManager)
  68:         Auth(msg.sender, Authority(address(0x0)))
  69:     {
  70:         PRICING_CONSTANT = _priceNormalizationFactor;
  71:         systemManager = _systemManager;
  72:     }

```

`PRICING_CONSTANT` value can be 0 value;
```solidity

src/Exchange.sol:
  
  160:         indexPrice = baseAssetPrice.mulDivDown(baseAssetPrice, PRICING_CONSTANT);
  200:         uint256 squarePrice = baseAssetPrice.mulDivDown(baseAssetPrice, PRICING_CONSTANT);

src/LiquidityPool.sol:
  729:         uint256 pricingConstant = exchange.PRICING_CONSTANT();

```

Recommended Mitigation Steps:
```diff

src/Exchange.sol:
  66: 
  67:     constructor(uint256 _priceNormalizationFactor, ISystemManager _systemManager)
  68:         Auth(msg.sender, Authority(address(0x0)))
  69:     {
+               require( 0 < PRICING_CONSTANT, “A value of zero or less cannot be defined”
  70:         PRICING_CONSTANT = _priceNormalizationFactor;
  71:         systemManager = _systemManager;
  72:     }

```


### [L-02] Front running attacks by the `requiresAuth` 

` performanceFee ` and `withdrawalFee`value are not a constant values and can be changed with ` setFees() ` function, before a function using ` performanceFee ` and `withdrawalFee` state variable values in the project, `setSigner` function can be triggered by `requiresAuth` and and user may pay more commission than expected

If there are very high balances, the user's loss will be high, I mentioned it here because I am not sure whether it falls under the Medium or Low category since it falls under the user loss.

```js
1 result - 1 file

src/KangarooVault.sol:
  476:     function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
  477:         require(_performanceFee <= 1e17 && _withdrawalFee <= 1e16);
  478: 
  479:         emit UpdateFees(performanceFee, withdrawalFee, _performanceFee, _withdrawalFee);
  480: 
  481:         performanceFee = _performanceFee;
  482:         withdrawalFee = _withdrawalFee;
  483:     }
```


### Recommended Mitigation Steps

Use a timelock to avoid instant changes of the parameters.

### [L-03] Insufficient coverage

**Description:**
The test coverage rate of the project is ~85%. Testing all functions is best practice in terms of security criteria.
Due to its capacity, test coverage is expected to be 100%.


```js

| File                                          | % Lines           | % Statements      | % Branches       | % Funcs          |
|-----------------------------------------------|-------------------|-------------------|------------------|------------------|
| src/KangarooVault.sol                         | 63.82% (187/293)  | 64.33% (220/342)  | 42.86% (42/98)   | 46.67% (14/30)   |
| src/LiquidityPool.sol                         | 89.34% (243/272)  | 91.57% (326/356)  | 60.00% (36/60)   | 81.40% (35/43)   |
| src/ShortToken.sol                            | 96.55% (28/29)    | 96.55% (28/29)    | 92.86% (13/14)   | 83.33% (5/6)     |
| src/SystemManager.sol                         | 14.29% (2/14)     | 14.29% (2/14)     | 0.00% (0/2)      | 33.33% (1/3)     |
| src/VaultToken.sol                            | 0.00% (0/5)       | 0.00% (0/5)       | 0.00% (0/2)      | 0.00% (0/3)      |
```


### [L-04] Project Upgrade and Stop Scenario should be

At the start of the project, the system may need to be stopped or upgraded, I suggest you have a script beforehand and add it to the documentation.
This can also be called an " EMERGENCY STOP (CIRCUIT BREAKER) PATTERN ".

https://github.com/maxwoe/solidity_patterns/blob/master/security/EmergencyStop.sol


### [L-05] Add to Event-Emit for critical functions

Monitoring the amount of Ether coming to the contract outside the chain is an important data

```solidity
2 results - 2 files

src/KangarooVault.sol:
  453  
  454:     receive() external payable {
  455:         (bool success,) = feeReceipient.call{value: msg.value}("");
  456          require(success);

src/LiquidityPool.sol:
  707  
  708:     receive() external payable {
  709:         (bool success,) = feeReceipient.call{value: msg.value}("");
  710          require(success);



```


### [L-06] Update codes to avoid Compile Errors


```js

warning[5667]: Warning: Unused function parameter. Remove or comment out the variable name to silence this warning.
  --> src/ShortToken.sol:44:23:
   |
44 |     function tokenURI(uint256 tokenId) public view override returns (string memory) {
   |                       ^^^^^^^^^^^^^^^



warning[2018]: Warning: Function state mutability can be restricted to pure
  --> src/ShortToken.sol:44:5:
   |
44 |     function tokenURI(uint256 tokenId) public view override returns (string memory) {
   |     ^ (Relevant source part starts here and spans across multiple lines).

```

### [L-07] ` processWithdrawalQueue()` event is missing parameters


The `Kangaroo.sol` contract has very important function; `processWithdrawalQueue()` 


However, only some values are published in emits, whereas msg.sender must be specified in every transaction, a contract or web page listening to events cannot react to users, emit does not serve the purpose.
Basically, this event cannot be used


```diff
1 result - 1 file

src/KangarooVault.sol:
  267      /// @notice Process queued withdrawal requests
  268:     /// @param idCount Number of withdrawal queue items to process
  269:     function processWithdrawalQueue(uint256 idCount) external nonReentrant {
  270:         for (uint256 i = 0; i < idCount; i++) {
  271:             uint256 tokenPrice = getTokenPrice();
  272: 
  273:             QueuedWithdraw storage current = withdrawalQueue[queuedWithdrawalHead];
  274: 
  275:             if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minWithdrawDelay) {
  276:                 return;
  277:             }
  278: 
  279:             uint256 availableFunds = totalFunds - usedFunds;
  280: 
  281:             if (availableFunds == 0) {
  282:                 return;
  283:             }
  284: 
  285:             uint256 susdToReturn = current.withdrawnTokens.mulWadDown(tokenPrice);
  286: 
  287:             // Partial withdrawals if not enough available funds in the vault
  288:             // Queue head is not increased
  289:             if (susdToReturn > availableFunds) {
  290:                 current.returnedAmount = availableFunds;
  291:                 uint256 tokensBurned = availableFunds.divWadUp(tokenPrice);
  292: 
  293:                 totalQueuedWithdrawals -= tokensBurned;
  294:                 current.withdrawnTokens -= tokensBurned;
  295: 
  296:                 totalFunds -= availableFunds;
  297: 
  298:                 if (withdrawalFee > 0) {
  299:                     uint256 withdrawFees = availableFunds.mulWadDown(withdrawalFee);
  300:                     SUSD.safeTransfer(feeReceipient, withdrawFees);
  301:                     availableFunds -= withdrawFees;
  302:                 }
  303: 
  304:                 SUSD.safeTransfer(current.user, availableFunds);
  305: 
  306:                 emit ProcessWithdrawalPartially(
  307:                     current.id, current.user, tokensBurned, availableFunds, current.requestedTime
  308:                     );
  309:                 return;
  310:             } else {
  311:                 current.returnedAmount = susdToReturn;
  312:                 totalQueuedWithdrawals -= current.withdrawnTokens;
  313: 
  314:                 totalFunds -= susdToReturn;
  315: 
  316:                 if (withdrawalFee > 0) {
  317:                     uint256 withdrawFees = susdToReturn.mulWadDown(withdrawalFee);
  318:                     SUSD.safeTransfer(feeReceipient, withdrawFees);
  319:                     susdToReturn -= withdrawFees;
  320:                 }
  321: 
  322:                 SUSD.safeTransfer(current.user, susdToReturn);
  323: 
  324:                 emit ProcessWithdrawal(
- 325:                     current.id, current.user, current.withdrawnTokens, susdToReturn, current.requestedTime);
+ 325:                     msg.sender, current.id, current.user, current.withdrawnTokens, susdToReturn, current.requestedTime);
  327: 
  328:                 current.withdrawnTokens = 0;
  329:             }
  330: 
  331:             queuedWithdrawalHead++;
  332:         }
  333:     }

```



### Recommended Mitigation Steps
add `msg.sender` parameter in event-emit

### [L-08] In the `constructor` , there is no return of incorrect address identification

In case of incorrect address definition in the `constructor` , there is no way to fix it because of the variables are immutable

It is recommended to fix the architecture
Address definitions can be done changeable architecture

```solidity

src/KangarooVault.sol:
  155  
  156:     constructor(
  157:         ERC20 _susd,
  158:         IVaultToken _vaultToken,
  159:         IExchange _exchange,
  160:         ILiquidityPool _pool,
  161:         IPerpsV2Market _perpMarket,
  162:         bytes32 _underlyingKey,
  163:         bytes32 _name
  164:     ) Auth(msg.sender, Authority(address(0x0))) {
  165:         VAULT_TOKEN = _vaultToken;
  166:         EXCHANGE = _exchange;
  167:         LIQUIDITY_POOL = _pool;
  168:         PERP_MARKET = _perpMarket;
  169:         UNDERLYING_SYNTH_KEY = _underlyingKey;
  170:         name = _name;
  171:         SUSD = _susd;
  172:     }


```

### [L-09] Critical Address Changes Should Use Two-step Procedure

The critical procedures should be two step process.


```solidity

1 result - 1 file

src/LiquidityPool.sol:
  649      /// @param _feeReceipient Address of the new fee receipient
  650:     function setFeeReceipient(address _feeReceipient) external requiresAuth {
  651:         feeReceipient = _feeReceipient;
  652:     }

```

Recommended Mitigation Steps
Lack of two-step procedure for critical operations leaves them error-prone. Consider adding two step procedure on the critical functions.


### [L-10] The nonReentrant modifier should occur before all other modifiers

For multiple modifier functions, check that the order of modifiers is significant and correct. In terms of both safety and gas

Re-Entrancy modifier ı tüm modifierlardan önce gelmeli : The nonReentrant modifier should occur before all other modifiers
This is a best-practice to protect against reentrancy in other modifiers

```solidity

21 results - 3 files

src/KangarooVault.sol:
  376:     function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {
  383:     function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {
  389:     function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {
  395:     function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {
  401:     function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
  424:     function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {
  436:     function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
  450:     function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {

src/LiquidityPool.sol:
  430:     function openLong(uint256 amount, address user, bytes32 referralCode) external override onlyExchange nonReentrant
  458:     function closeLong(uint256 amount, address user, bytes32 referralCode) external override onlyExchange nonReentrant
  486:     function openShort(uint256 amount, address user, bytes32 referralCode) external override onlyExchange nonReentrant
  514:     function closeShort(uint256 amount, address user, bytes32 referralCode) external override onlyExchange nonReentrant
  541:     function liquidate(uint256 amount) external override onlyExchange nonReentrant {
  552:     function hedgePositions() external override requiresAuth nonReentrant {
  575:     function rebalanceMargin(int256 marginDelta) external requiresAuth nonReentrant {
  597:     function increaseMargin(uint256 additionalMargin) external requiresAuth nonReentrant {
  676:     function placeQueuedOrder() external requiresAuth nonReentrant {
  688:     function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {

src/ShortCollateral.sol:
   85:     function collectCollateral(address collateral, uint256 positionId, uint256 amount) external onlyExchange nonReentrant
  103:     function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {
  118:     function liquidate(uint256 positionId, uint256 debt, address user) external override onlyExchange nonReentrant

```


### [N-11] Tokens accidentally sent to the contract cannot be recovered


**Contex**
src/KangarooVault.sol:

It can't be recovered if the tokens accidentally arrive at the contract address, which has happened to many popular projects, so I recommend adding a recovery code to your critical contracts.

### Recommended Mitigation Steps

Add this code:

```solidity
 /**
  * @notice Sends ERC20 tokens trapped in contract to external address
  * @dev Onlyowner is allowed to make this function call
  * @param account is the receiving address
  * @param externalToken is the token being sent
  * @param amount is the quantity being sent
  * @return boolean value indicating whether the operation succeeded.
  *
 */
  function rescueERC20(address account, address externalToken, uint256 amount) public onlyOwner returns (bool) {
    IERC20(externalToken).transfer(account, amount);
    return true;
  }
}

```

### [N-12] Floating pragma

**Description:**
Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.
https://swcregistry.io/docs/SWC-103

Floating Pragma List: 

```solidity
11 results - 11 files

src/Exchange.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/KangarooVault.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/LiquidityPool.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/LiquidityToken.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/PowerPerp.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/ShortCollateral.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/ShortToken.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/SynthetixAdapter.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/SystemManager.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/libraries/SignedMath.sol:
  1  // SPDX-License-Identifier: MIT
  2: pragma solidity ^0.8.9;
  3  

src/utils/PauseModifier.sol:
  2  
  3: pragma solidity ^0.8.9;
  4 

```


**Recommendation:**
Lock the pragma version and also consider known bugs (https://github.com/ethereum/solidity/releases) for the compiler version that is chosen.

### [N-13] Use SMTChecker

The *highest* tier of smart contract behavior assurance is formal mathematical verification. All assertions that are made are guaranteed to be true across all inputs → The quality of your asserts is the quality of your verification.

https://twitter.com/0xOwenThurm/status/1614359896350425088?t=dbG9gHFigBX85Rv29lOjIQ&s=19

### [N-14] Constants on the left are better

If you use the constant first you support structures that veil programming errors. And one should only produce code either to add functionality, fix an programming error or trying to support structures to avoid programming errors (like design patterns). 

https://www.moserware.com/2008/01/constants-on-left-are-better-but-this.html

```solidity
9 instances

inputs[i].rewardWindows[j].startTime <= lastTime) {
src/KangarooVault.sol:
  671:         require(positionData.positionId != 0 && positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);

src/LiquidityPool.sol:
  787:         if (oldSize != 0 || isLiquidation || uint8(status) != 0) {


src/KangarooVault.sol:
  248  
  249:         if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
  275:         if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minWithdrawDelay) {
  557:         require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);
  671:         require(positionData.positionId != 0 && positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);
  731:         require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp > 0);

src/LiquidityPool.sol:
  226:         if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
  291:         if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minWithdrawDelay) {

```


### [N-15] `constants` should be defined rather than using magic numbers

A magic number is a numeric literal that is used in the code without any explanation of its meaning. The use of magic numbers makes programs less readable and hence more difficult to maintain and update.

```solidity

1 result - 1 file

src/ShortCollateral.sol:
  236  
  237:         maxDebt = position.shortAmount / 2;

```


### [N-16] Use the `delete` keyword instead of assigning a value of `0`


Using the 'delete' keyword instead of assigning a '0' value is a detailed optimization that increases code readability and audit quality, and clearly indicates the intent.

Other hand, if use `delete` instead `0` value assign , it will be gas saved

```solidity
13 results - 2 files

src/KangarooVault.sol:
  262:             current.depositedAmount = 0;
  328:                 current.withdrawnTokens = 0;
  641:                 positionData.premiumCollected = 0;
  667:         positionData.pendingLongPerp = 0;
  714:             positionData.premiumCollected = 0;
  779:         positionData.pendingShortPerp = 0;
  783:         positionData.positionId = 0;
  794:         positionData.premiumCollected = 0;
  795:         positionData.totalMargin = 0;
  796:         usedFunds = 0;

src/LiquidityPool.sol:
  239:             current.depositedAmount = 0;
  329:                 current.withdrawnTokens = 0;
  701:         queuedPerpSize = 0;


```


### [N-17] Function Calls in Loop Could Lead to Denial of Service due to Array length not being checked


```diff

1 result - 1 file

contracts/staking/NeoTokyoStaker.sol:
  1819: 	function configurePools (
  1820: 		PoolConfigurationInput[] memory _inputs
  1821: 	) hasValidPermit(UNIVERSAL, CONFIGURE_POOLS) external {
+                require(_inputs.length< max _inputesArrayLengt, "max length");
  1822: 		for (uint256 i; i < _inputs.length; ) {
  1823: 			uint256 poolRewardWindowCount = _inputs[i].rewardWindows.length;
  1824: 			_pools[_inputs[i].assetType].rewardCount = poolRewardWindowCount;
  1825: 			_pools[_inputs[i].assetType].daoTax = _inputs[i].daoTax;
  1826: 

2 results - 2 files

src/KangarooVault.sol:
  449      /// @notice Executed delayed orders
  450:     function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
+	  require(priceUpdateData.length< max _ priceUpdateDataArrayLengt, "max length");
  451          PERP_MARKET.executeOffchainDelayedOrder{value: msg.value}(address(this), priceUpdateData);

src/LiquidityPool.sol:
  703  
  704:     function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
+	  require(priceUpdateData.length< max _ priceUpdateDataArrayLengt, "max length");
  705          perpMarket.executeOffchainDelayedOrder{value: msg.value}(address(this), priceUpdateData);




```

**Recommendation:**
Function calls made in unbounded loop are error-prone with potential resource exhaustion as it can trap the contract due to gas limitations or failed transactions. Consider bounding the loop length if the array is expected to be growing and/or handling a huge list of elements to avoid unnecessary gas wastage and denial of service.

### [N-18] Take advantage of Custom Error's return value property

An important feature of Custom Error is that values such as address, tokenID, msg.value can 
be written inside the `()` sign, this kind of approach provides a serious advantage in 
debugging and examining the revert details of dapps such as tenderly.

For Example;

```solidity

src/VaultToken.sol:
  34  
  35:     function setVault(address _vault) external {
  36:         if (vault != address(0x0)) {
  37:             revert();
  38:         }
  39:         vault = _vault;
  40:     }
```


### [N-19] Test environment comments and codes should not be in the main version


```solidity

1 result - 1 file

src/KangarooVault.sol:
  4: import {console2} from "forge-std/console2.sol";
```
### [N-20] Starting with default values is confusing.


```diff
src/SystemManager.sol:

 42:     bool public isInitialized; //default valu is false

  46  
  47:     constructor(
  48:         address _addressResolver,
  49:         address _futureMarketManager,
  50:         address _susd,
  51:         bytes32 _baseAsset,
  52:         bytes32 _perpMarketName
  53:     ) Auth(msg.sender, Authority(address(0x0))) {
  54:         addressResolver = IAddressResolver(_addressResolver);
  55:         futuresMarketManager = IFuturesMarketManager(_futureMarketManager);
  56:         PERP_MARKET_CONTRACT = _perpMarketName;
  57:         SUSD = ERC20(_susd);
  58:         baseAsset = _baseAsset;
- 59:         isInitialized = false;
  60:     }
```

### [N-21]  Add NatSpec Mapping comment

**Description:**
Add NatSpec comments describing mapping keys and values


**Context:**

```solidity

8 results - 5 files

src/KangarooVault.sol:
  150      /// @notice Deposit Queue
  151:     mapping(uint256 => QueuedDeposit) public depositQueue;
  152  
  153      /// @notice Withdrawal Queue
  154:     mapping(uint256 => QueuedWithdraw) public withdrawalQueue;
  155  

src/LiquidityPool.sol:
  143      /// @notice Deposit Queue
  144:     mapping(uint256 => QueuedDeposit) public depositQueue;
  145  
  146      /// @notice Withdrawal Queue
  147:     mapping(uint256 => QueuedWithdraw) public withdrawalQueue;
  148  

src/ShortCollateral.sol:
  33      /// @notice Approved collaterals
  34:     mapping(bytes32 => Collateral) public collaterals;
  35  
  36      /// @notice User collateral information
  37:     mapping(uint256 => UserCollateral) public userCollaterals;
  38  

src/ShortToken.sol:
  23  
  24:     mapping(uint256 => ShortPosition) public shortPositions;
  25  

src/SystemManager.sol:
  44  
  45:     mapping(bytes32 => bool) public isPaused;
  46 
```

**Recommendation:**

```solidity

 /// @dev    address(user) -> address(ERC0 Contract Address) -> uint256(allowance amount from user)
    mapping(address => mapping(address => uint256)) public allowance;

```


### [N-22] Unnecessary Emission of block.timestamp



```solidity
1 result - 1 file

src/KangarooVault.sol:
  192              totalFunds += amount;
  193:             emit ProcessDeposit(0, user, amount, tokensToMint, block.timestamp);

```

**Recommendation:**
Any event emitted by Solidity includes the timestamp and block number implicitly. Adding it the second time just increases the cost of the transaction

### [N-23] `Function writing` that does not comply with the `Solidity Style Guide`

**Context:**
All Contracts

**Description:**
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
within a grouping, place the view and pure functions last
