## Summary

### Low Risk Issues

|  | Issue | Instances |
| --- | --- | --- |
| [L‑01] |  Initial value check is missing in Set Functions | 16 |
| [L‑02] | Add event emit for receive() function | 1 |
| [L‑03] |  Events not emitted for important state changes | 1 |
| [L‑04] | _safeMint() should be used rather than _mint() wherever possible | 4 |

Total: 22 instances over 4 issues

### Non-critical Issues

|  | Issue | Instances |
| --- | --- | --- |
| [N‑01] | Floating pragma | All Contracts |
| [N‑02] | Include return parameters in NatSpec comments | 5 |
| [N‑03] | Missing Event for initialize | 9 |
| [N‑04] | Consider using delete rather than assigning zero to clear values | 11 |
| [N‑05] | The nonReentrant modifier should occur before all other modifiers | 11 |
| [N‑06] | Duplicated require()/revert() checks should be refactored to a modifier or function | 4 |
| [N‑07] | Add NatSpec Mapping comment | 2 |
| [N‑08] | Function writing does not comply with the Solidity Style Guide | 6 |

## Low Risk Issues

### **[L-01] Initial value check is missing in Set Functions**

Checking whether the current value and the new value are the same should be added

*There is 16 instance of this issue:*

```solidity
File: /src/LiquidityPool.sol
650: function setFeeReceipient(address _feeReceipient) external requiresAuth {
657: function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
665: function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
672: function setDevFee(uint256 _devFee) external requiresAuth {
679: function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
685: function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
```

```solidity
File: /src/KangarooVault.sol
476: function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
487: function setSynthetixTracking(bytes32 _code) external requiresAuth {
492: function setReferralCode(bytes32 _code) external requiresAuth {
499: function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
506: function setMaxDepositAmount(uint256 _maxAmt) external requiresAuth {
514: function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
522: function setPriceImpactDelta(uint256 _delta) external requiresAuth {
529: function setLeverage(uint256 _lev) external requiresAuth {
537: function setCollRatio(uint256 _ratio) external requiresAuth {
```

```solidity
File: /src/SystemManager.sol
89: function setStatusFunction(bytes32 key, bool status) public requiresAuth {
```

### [L-02] Add event emit for `receive()` function

*There is 1 instance of this issue:*

```solidity
File: /src/KangarooVault.sol
454: receive() external payable {
455:        (bool success,) = feeReceipient.call{value: msg.value}("");
456:        require(success);
457:    }
```

**Mitigation**

```diff
	454: receive() external payable {
	455:        (bool success,) = feeReceipient.call{value: msg.value}("");
	456:        require(success);
+	457:        emit ReceiveEther(msg.sender, msg.value);
+	458:    }
```

**Add** `emit ReceiveEther(msg.sender, msg.value);`

### **[L-03] Events not emitted for important state changes**

When changing state variables events are not emitted. Emitting events allows monitoring activities with off-chain monitoring tools.

*There is 1 instance of this issue:*

```solidity
File: /src/LiquidityPool.sol
	650: function setFeeReceipient(address _feeReceipient) external requiresAuth {
	651:        feeReceipient = _feeReceipient;
	652:    }
```

- [LiquidityPool.sol#L650](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L650)

**Add** `emit UpdateFeeReceipient(feeReceipient, _feeReceipient);`

### [L‑04] `_safeMint()` should be used rather than `_mint()` wherever possible

`_mint()` is [discouraged](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d4d8d2ed9798cc3383912a23b5e8d5cb602f7d4b/contracts/token/ERC721/ERC721.sol#L271) in favor of `_safeMint()` which ensures that the recipient is either an EOA or implements `IERC721Receiver`. Both [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d4d8d2ed9798cc3383912a23b5e8d5cb602f7d4b/contracts/token/ERC721/ERC721.sol#L238-L250) and [solmate](https://github.com/Rari-Capital/solmate/blob/4eaf6b68202e36f67cab379768ac6be304c8ebde/src/tokens/ERC721.sol#L180) have versions of this function

*There are 4 instances of this issue:*

```solidity
File: /src/LiquidityToken.sol
29: _mint(_user, _amt);

File: /src/PowerPerp.sol
33: _mint(_user, _amt);

File: /src/ShortToken.sol
67: _mint(trader, positionId);

File: /src/VaultToken.sol
28: _mint(_user, _amt);
```

## Non-critical Issues

### [N-01] Floating pragma

**Description:** Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively. [https://swcregistry.io/docs/SWC-103](https://swcregistry.io/docs/SWC-103)

**All Contracts**

### [N-02] Include return parameters in NatSpec comments

 It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation. In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability.

[https://docs.soliditylang.org/en/v0.8.15/natspec-format.html](https://docs.soliditylang.org/en/v0.8.15/natspec-format.html)

Include return parameters in NatSpec comments

*There are 5 instances of this issue:*

- [ShortToken.sol#L92](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L92)
- [Exchange.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol)
- [LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol)
- [PowerPerp.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol)
- [KangarooVault.sol#L341](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L341)

```solidity
/// @notice Adds liquidity for the given recipient/tickLower/tickUpper position
/// @dev The caller of this method receives a callback in the form of IUniswapV3MintCallback#uniswapV3MintCallback
/// in which they must pay any token0 or token1 owed for the liquidity. The amount of token0/token1 due depends
/// on tickLower, tickUpper, the amount of liquidity, and the current price.
/// @param recipient The address for which the liquidity will be created
/// @param tickLower The lower tick of the position in which to add liquidity /// @param tickUpper The upper tick of the position in which to add liquidity
/// @param amount The amount of liquidity to mint
/// @param data Any data that should be passed through to the callback
/// @return amount0 The amount of token0 that was paid to mint the given amount of liquidity. Matches the value in the callback
/// @return amount1 The amount of token1 that was paid to mint the given amount of liquidity. Matches the value in the callback
    function mint(
        address recipient,
        int24 tickLower,
        int24 tickUpper,
        uint128 amount,
        bytes calldata data
    ) external returns (uint256 amount0, uint256 amount1);
```

### [N-03] Missing Event for initialize

Events help non-contract tools to track changes, and events prevent users from being surprised by changes Issuing event-emit during initialization is a detail that many projects skip

Add Event-Emit

*There are 9 instances of this issue:*

```solidity
File: /src/Exchange.sol
67: constructor(uint256 _priceNormalizationFactor, ISystemManager _systemManager)
```

- [Exchange.sol#L67](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L67)

```solidity
FIle: /src/LiquidityPool.sol
153: constructor(ERC20 _susd, bytes32 _baseAsset, bytes32 _trackingCode, ISystemManager _systemManager)
```

- [LiquidityPool.sol#L153](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L153)

```solidity
File: /src/LiquidityToken.sol
12: constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)
```

- [LiquidityToken.sol#L12](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol#L12)

```solidity
File: /src/PowerPerp.sol
15: constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)
```

- [PowerPerp.sol#L15](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L15)

```solidity
File: /src/ShortCollateral.sol
51: constructor(uint256 susdRatio, uint256 susdLiqRatio, uint256 susdLiqBonus, ISystemManager _systemManager)
```

- [ShortCollateral.sol#L51](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L51)

```solidity
File: /src/ShortToken.sol
26: constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)
```

- [ShortToken.sol#L26](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L26)

```solidity
File: /src/SynthetixAdapter.sol
13: constructor(ISynthetix _synthetix, IExchangeRates _exchangeRates) {
```

- [SynthetixAdapter.sol#L13](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SynthetixAdapter.sol#L13)

```solidity
File: /src/SystemManager.sol
47:  constructor(
48:        address _addressResolver,
49:        address _futureMarketManager,
50:        address _susd,
51:        bytes32 _baseAsset,
52:        bytes32 _perpMarketName
53:    ) Auth(msg.sender, Authority(address(0x0))) {
```

- [SystemManager.sol#L47](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L47)

```solidity
File: 
156: constructor(
157:        ERC20 _susd,
158:        IVaultToken _vaultToken,
159:        IExchange _exchange,
160:        ILiquidityPool _pool,
161:        IPerpsV2Market _perpMarket,
162:        bytes32 _underlyingKey,
163:        bytes32 _name
164:    ) Auth(msg.sender, Authority(address(0x0))) {
```

- [KangarooVault.sol#L156](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L156)

### [N‑04] Consider using `delete` rather than assigning zero to clear values

The `delete` keyword more closely matches the semantics of what is being done, and draws more attention to the changing of state, which may lead to a more thorough audit of its associated logic

*There are 11 instances of this issue:*

```solidity
File: /src/LiquidityPool.sol
	239: current.depositedAmount = 0;

	329: current.withdrawnTokens = 0;

	701: queuedPerpSize = 0;
```

- [LiquidityPool.sol#L239](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L239)

```solidity
File: /src/KangarooVault.sol
	262: current.depositedAmount = 0;

	328: current.withdrawnTokens = 0;

	641: positionData.premiumCollected = 0;

	667: positionData.pendingLongPerp = 0;

	779: positionData.pendingShortPerp = 0;

	794: positionData.premiumCollected = 0;
  795: positionData.totalMargin = 0;
	796: usedFunds = 0;
```

- [KangarooVault.sol#L262](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L262)

### [N-05] **The `nonReentrant` `modifier` should occur before all other modifiers**

This is a best-practice to protect against reentrancy in other modifiers

*There are 11 instances of this issue:*

```solidity
File: /src/ShortCollateral.sol	
85: function collectCollateral(address collateral, uint256 positionId, uint256 amount)

106: function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {

121: function liquidate(uint256 positionId, uint256 debt, address user)
122:        external
123:        override
124:        onlyExchange
125:        nonReentrant
126:        returns (uint256 totalCollateralReturned)
```

- [ShortCollateral.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol)

```solidity
File: /src/KangarooVault.sol
376: function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {

383: function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {

389: function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {

395: function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {

401: function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {

424: function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant

436: function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {

450: function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```

- [KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol)

### [N-06] Duplicated `require()`/`revert()` checks should be refactored to a modifier or function

The compiler will inline the function, which will avoid `JUMP` instructions usually associated with functions

*There is 4 instance of this issue:*

```solidity
File: /src/LiquidityPool.sol
require(!isInvalid);

File: /src/ShortCollateral.sol
require(!isInvalid);

File: /src/KangarooVault.sol
require(!isInvalid);

File: /src/ShortToken.sol
require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
```

### [N-07] Add NatSpec Mapping comment

 Add NatSpec comments describing mapping keys and values

*There is 2 instance of this issue:*

```solidity
File: /src/ShortToken.sol
 24: mapping(uint256 => ShortPosition) public shortPositions;

File: /src/SystemManager.sol
 45: mapping(bytes32 => bool) public isPaused;
```

### [N-08] Function writing does not comply with the `Solidity Style Guide`

Ordering helps readers identify which functions they can call and to find the constructor and fallback definitions easier. But there are contracts in the project that do not comply with this.

Functions should be grouped according to their visibility and ordered:

- `constructor()`
- `receive()`
- `fallback()`
- `external` / `public` / `internal` / `private`
- `view` / `pure`

*There is 6 file of this issue:*

- [SynthetixAdapter.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SynthetixAdapter.sol)
- [SystemManager.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol)
- [VaultToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol)
- [ShortToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol)
- [PowerPerp.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol)
- [LiquidityToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol)