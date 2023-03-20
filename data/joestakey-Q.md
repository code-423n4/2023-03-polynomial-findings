# Report


## Low Issues


| |Issue|
|-|:-|
| [L-1](#L-1) | Use `require` instead of `assert` |
| [L-2](#L-2) | Setters should ensure contract address implement appropriate interfaces |
| [L-3](#L-3) | Minor loss of precision when multiplying after dividing  |
| [L-4](#L-4) | Use `_safeMint` instead of `_mint` for ERC721 tokens |
| [L-5](#L-5) | Check sender in `KangarooVault.receive()` |


### L-1  Use `require` instead of `assert`
  As per Solidity’s [documentation](https://docs.soliditylang.org/en/v0.8.17/control-structures.html?highlight=assert#panic-via-assert-and-error-via-require):
 "The assert function creates an error of type Panic(uint256). The same error is created by the compiler in certain situations as listed below. 
Assert should only be used to test for internal errors, and to check invariants. Properly functioning code should never create a Panic, not even on invalid external input. If this happens, then there is a bug in your contract which you should fix"

*Instances (2)*:
```solidity
File: src/LiquidityPool.sol

220:         assert(queuedDepositHead + count - 1 < nextQueuedDepositId);

285:         assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);

```

### L-2  Setters should ensure contract address implement appropriate interfaces

Because the function can only be used once, ensure correctness of the `vault_` by checking it implement a `KangarooVault` interface (which should be defined as it does not exist currently)

```solidity
File: src/VaultToken.sol
35: function setVault(address _vault) external {
36:         if (vault != address(0x0)) {
37:             revert();
38:         }
39:         vault = _vault;
40:     }
```


### L-3 Minor loss of precision when multiplying after dividing
Because Solidity truncates divisions, multiplication after division should be avoided

```solidity
File: src/ShortCollateral.sol
169:        collateralAmt = markPrice.mulDivDown(shortAmount, collateralPrice);
170:        collateralAmt = collateralAmt.mulWadDown(coll.collateralRatio);
```

There can be some truncation in `shortAmount * markPrice / collateralPrice`, which is multiplied afterwards to `coll.collateralRatio`


### L-4 Use `_safeMint` instead of `_mint` for ERC721 tokens
This will ensure that the recipient is either an EOA or implements `IERC721Receiver`

*Instances (1)*:


```solidity
File: src/ShortToken.sol

67:             _mint(trader, positionId);

```

### L-5 Check sender in `KangarooVault.receive()`
`KangarooVault` currently transfers `ETH` received to `feeReceipient`.
This means a user that mistakenly transfers `ETH` to it will lose that amount.
On the other hand, there is a function `saveToken` for ERC20 tokens that would get stuck in the contract.

*Instances (1)*:
```solidity
File: src/KangarooVault.sol

455:         (bool success,) = feeReceipient.call{value: msg.value}("");

```

For consistency with the behavior of `saveToken`:

- refactor `receive()` so that it reverts
- create a function `fundFeeRecipient` to transfer `ETH` to `feeReceipient`





## Non Critical Issues


| |Issue|
|-|:-|
| [NC-1](#NC-1) | Remove unused variables |
| [NC-2](#NC-2) | Constants defined more than once |
| [NC-3](#NC-3) | Constants on the left are better |
| [NC-4](#NC-4) | Typos |
| [NC-5](#NC-5) | Order of functions not following Solidity standard practice | 
| [NC-6](#NC-6) | Address receiving ETH should be payable |
| [NC-7](#NC-7) | `nonReentrant` should be the first modifier |

### NC-1 Remove unused variables

*Instances (1)*:
```solidity
File: src/Exchange.sol

50:     ERC20 public SUSD;

```

### NC-2 Constants defined more than once
Some constants are defined twice or more in different contracts. Define them in only one contract, for instance using a constants library, so that values cannot become out of sync when only one location is updated

*Instances (3)*:

`SUSD`, `marketKey` and `systemManager`
```solidity
File: src/KangarooVault.sol

66:     ERC20 public immutable SUSD;

```

```solidity
File: src/LiquidityPool.sol

65:     ERC20 public immutable SUSD;

```

```solidity
File: src/PowerPerp.sol

9:     bytes32 public immutable marketKey;

10:     ISystemManager public immutable systemManager;

```

```solidity
File: src/ShortCollateral.sol

46:     ISystemManager public immutable systemManager;

```

```solidity
File: src/ShortToken.sol

9:     bytes32 public immutable marketKey;

11:     ISystemManager public immutable systemManager;

```

```solidity
File: src/SystemManager.sol


24:     ERC20 public immutable SUSD;


```

```solidity
File: src/utils/PauseModifier.sol

8:     ISystemManager public systemManager;


```

### NC-3 Constants on the left are better
This is [safer in case you forget to put a double equals sign](https://www.moserware.com/2008/01/constants-on-left-are-better-but-this.html), as the compiler wil throw an error instead of assigning the value to the variable.

*Instances (42)*:
```solidity
File: src/Exchange.sol

236:             require(shortPositions == 0, "Short position must be closed before opening");

248:             require(holdings == 0, "Long position must be closed before opening");

257:                 require(params.collateral == shortPosition.collateral);

291:             require(shortPositions == 0, "Shouldn't have short positions to close long postions");

302:             require(holdings == 0, "Shouldn't have long positions to close short postions");

308:             require(shortPosition.collateral == params.collateral);

```

```solidity
File: src/KangarooVault.sol

188:         if (positionData.positionId == 0) {

219:         if (positionData.positionId == 0) {

249:             if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {

275:             if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minWithdrawDelay) {

281:             if (availableFunds == 0) {

342:         if (totalFunds == 0) {

347:         if (positionData.positionId == 0) {

557:         require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);

557:         require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);

560:         require(delayedOrder.sizeDelta == 0);

590:         if (positionData.positionId == 0) {

614:         require(positionData.pendingLongPerp > 0 && positionData.pendingShortPerp == 0);

617:         require(delayedOrder.sizeDelta == 0);

658:             if (positionData.shortAmount == 0) {

671:         require(positionData.positionId != 0 && positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);

671:         require(positionData.positionId != 0 && positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);

674:         require(delayedOrder.sizeDelta == 0);

731:         require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp > 0);

734:         require(delayedOrder.sizeDelta == 0);

774:             if (positionData.shortAmount == 0) {

```

```solidity
File: src/LiquidityPool.sol

173:         require(msg.sender == address(exchange));

226:             if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {

291:             if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minWithdrawDelay) {

297:             if (availableFunds == 0) {

341:         if (totalFunds == 0) {

348:         if (skew == 0) {

695:         require(order.sizeDelta == 0);

```

```solidity
File: src/LiquidityToken.sol

24:         require(msg.sender == liquidityPool);

```

```solidity
File: src/PowerPerp.sol

28:         require(msg.sender == exchange);

```

```solidity
File: src/ShortCollateral.sol

73:         require(msg.sender == address(exchange));

94:         if (userCollateral.collateral == address(0x0)) {

210:         return position.collateralAmount < minCollateral;

```

```solidity
File: src/ShortToken.sol

39:         require(msg.sender == exchange);

55:         if (positionId == 0) {

69:             require(trader == ownerOf(positionId));

82:             if (position.shortAmount == 0) {

```



### NC-4 Typos

```solidity
File: src/KangarooVault.sol
182: /// @param amount Amount of sUSD being depositted//@audit-info deposited
```


### NC-5 Order of functions not following Solidity standard practice
Follow Solidity's [style guide](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-functions) for function ordering: constructor, receive/fallback, external, public, internal and private.

*Instances (4)*:
```solidity
File: src/Exchange.sol

206:     function orderFee(int256 sizeDelta) public view returns (uint256 fees) {

216:     function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {

223:     function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {


```


```solidity
File: src/LiquidityPool.sol

153:     constructor(ERC20 _susd, bytes32 _baseAsset, bytes32 _trackingCode, ISystemManager _systemManager)

163:     function refresh() public { //@audit-info external functions defined after

184:     function deposit(uint256 amount, address user) external override nonReentrant whenNotPaused("POOL_DEPOSIT") {

200:     function queueDeposit(uint256 amount, address user)

219:     function processDeposits(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_DEPOSITS") {

247:     function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {

264:     function queueWithdraw(uint256 tokens, address user)

284:     function processWithdraws(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_WITHDRAWS") {

340:     function getTokenPrice() public view override returns (uint256) {

367:     function getSlippageFee(int256 sizeDelta) public view returns (uint256) {

379:     function orderFee(int256 sizeDelta) public view override returns (uint256) {

399:     function baseAssetPrice() public view override returns (uint256 spotPrice, bool isInvalid) {

404:     function getMarkPrice() public view override returns (uint256, bool) { //@audit-info external functions defined after

409:     function getSkew() external view override returns (int256) {

414:     function getDelta() external view override returns (uint256) {

419:     function getExposure() external view returns (int256) {

```



### NC-6 Address receiving ETH should be payable
Low level calls can be performed on non-payable addresses, even when sending value. But to follow best practice (see OpenZeppelin's [Address library](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/dependencies/openzeppelin/contracts/Address.sol#L60-L65)) and add an extra layer of safety during compilation, consider casting these addresses as payable

*Instances (2)*:
```solidity
File: src/KangarooVault.sol

455:         (bool success,) = feeReceipient.call{value: msg.value}("");

```

```solidity
File: src/LiquidityPool.sol

709:         (bool success,) = feeReceipient.call{value: msg.value}("");

```


### NC-7 `nonReentrant` should be the first modifier
This ensures there is no reentrancy happening in other modifiers (even access control ones)

*Instances (19)*:
```solidity
File: src/KangarooVault.sol

376:     function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {

383:     function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {

389:     function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {

395:     function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {

401:     function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {

424:     function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {

436:     function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {

450:     function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {

```

```solidity
File: src/LiquidityPool.sol

184:     function deposit(uint256 amount, address user) external override nonReentrant whenNotPaused("POOL_DEPOSIT") {

219:     function processDeposits(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_DEPOSITS") {

247:     function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {

284:     function processWithdraws(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_WITHDRAWS") {

557:     function liquidate(uint256 amount) external override onlyExchange nonReentrant {

568:     function hedgePositions() external override requiresAuth nonReentrant {

591:     function rebalanceMargin(int256 marginDelta) external requiresAuth nonReentrant {

613:     function increaseMargin(uint256 additionalMargin) external requiresAuth nonReentrant {

692:     function placeQueuedOrder() external requiresAuth nonReentrant {

704:     function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {

```

```solidity
File: src/ShortCollateral.sol

106:     function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {

```
