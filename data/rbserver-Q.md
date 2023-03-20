## QA REPORT

| |Issue|
|-|:-|
| [01] | LACK OF LIMITS FOR SETTING FEES |
| [02] | `VaultToken.setVault` FUNCTION IS CALLABLE BY ANYONE, AND DEV TEAM'S `VaultToken.setVault` TRANSACTION CAN BE FRONTRUN BY MALICIOUS ACTOR |
| [03] | ALLOWING `ShortCollateral.refresh` FUNCTION TO BE CALLABLE BY ANYONE CAN BE DANGEROUS |
| [04] | SOME FUNCTIONS DO NOT FOLLOW CHECKS-EFFECTS-INTERACTIONS PATTERN |
| [05] | MISSING `address(0)` CHECKS FOR CRITICAL ADDRESS INPUTS |
| [06] | `nonReentrant` MODIFIER CAN BE PLACED AND EXECUTED BEFORE OTHER MODIFIERS IN FUNCTIONS |
| [07] | MISSING REASON STRING IN `revert` STATEMENT |
| [08] | REDUNDANT `return` KEYWORDS IN `ShortToken.transferFrom` and `ShortToken.safeTransferFrom` FUNCTIONS |
| [09] | CONSTANTS CAN BE USED INSTEAD OF MAGIC NUMBERS |
| [10] | HARDCODED STRING THAT IS REPEATEDLY USED CAN BE REPLACED WITH A CONSTANT |
| [11] | `ShortToken.adjustPosition` FUNCTION DOES NOT NEED TO UPDATE `totalShorts` and `position.shortAmount` IN CERTAIN CASE |
| [12] | `LiquidityPool.withdraw` FUNCTION CALLS BOTH `SUSD.safeTransfer` AND `SUSD.transfer` |
| [13] | `LiquidityPool.orderFee` FUNCTION CAN CALL `getMarkPrice()` INSTEAD OF `exchange.getMarkPrice()` |
| [14] | IMMUTABLES CAN BE NAMED USING SAME CONVENTION |
| [15] | UNUSED IMPORT |
| [16] | FLOATING PRAGMAS |
| [17] | SOLIDITY VERSION `0.8.19` CAN BE USED |
| [18] | ORDERS OF LAYOUT DO NOT FOLLOW OFFICIAL STYLE GUIDE |
| [19] | INCOMPLETE NATSPEC COMMENTS |
| [20] | MISSING NATSPEC COMMENTS |

## [01] LACK OF LIMITS FOR SETTING FEES
When calling the following `LiquidityPool.setFees` function, there are no limits for setting `depositFee` and `withdrawalFee`. If these fees are set to 1e18, calling the `LiquidityPool.deposit` function can cause all of the `amount` to become the deposit fees and zero liquidity tokens to be minted to the user, and calling the `LiquidityPool.withdraw` function can cause all of the `susdToReturn` to become the withdrawal fees and zero `SUSD` tokens to be transferred to the user. If these fees are set to be more than 1e18, calling the `LiquidityPool.deposit` function can revert because `amount - fees` underflows, and calling the `LiquidityPool.withdraw` function can also revert because `susdToReturn - fees` underflows.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L657-L661
```solidity
    function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
        emit UpdateFees(depositFee, _depositFee, withdrawalFee, _withdrawalFee);
        depositFee = _depositFee;
        withdrawalFee = _withdrawalFee;
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L184-L195
```solidity
    function deposit(uint256 amount, address user) external override nonReentrant whenNotPaused("POOL_DEPOSIT") {
        uint256 tokenPrice = getTokenPrice();
        uint256 fees = amount.mulWadDown(depositFee);
        uint256 amountForTokens = amount - fees;
        uint256 tokensToMint = amountForTokens.divWadDown(tokenPrice);
        liquidityToken.mint(user, tokensToMint);
        totalFunds += amountForTokens;
        SUSD.safeTransferFrom(msg.sender, feeReceipient, fees);
        SUSD.safeTransferFrom(msg.sender, address(this), amountForTokens);
        ...
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L247-L259
```solidity
    function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {
        ...
        uint256 tokenPrice = getTokenPrice();
        uint256 susdToReturn = tokens.mulWadDown(tokenPrice);
        uint256 fees = susdToReturn.mulWadDown(withdrawalFee);
        SUSD.safeTransfer(feeReceipient, fees);
        SUSD.transfer(user, susdToReturn - fees);
        totalFunds -= susdToReturn;
        liquidityToken.burn(msg.sender, tokens);
        ...
    }
```

Moreover, the `LiquidityPool.setDevFee` function has no limit for setting `devFee`, and similar issues can occur. For example, if `devFee` is set to more than 1e18, calling the `LiquidityPool.openLong` function will revert because `externalFee` is more than `feesCollected` and executing `feesCollected - externalFee` reverts.

As a mitigation, to prevent the `LiquidityPool.deposit`, `LiquidityPool.withdraw`, and `LiquidityPool.openLong` functions from behaving unexpectedly, the `LiquidityPool.setFees` and `LiquidityPool.setDevFee` functions can be updated to only allow the corresponding fees to be set to values that cannot exceed certain limits, which are reasonable values that are less than 1e18.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L672-L675
```solidity
    function setDevFee(uint256 _devFee) external requiresAuth {
        emit UpdateDevFee(devFee, _devFee);
        devFee = _devFee;
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L430-L457
```solidity
    function openLong(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCost)
    {
        ...
        uint256 tradeCost = amount.mulWadDown(markPrice);
        uint256 fees = orderFee(int256(amount));
        totalCost = tradeCost + fees;

        SUSD.safeTransferFrom(user, address(this), totalCost);

        uint256 hedgingFees = _hedge(int256(amount), false);
        uint256 feesCollected = fees - hedgingFees;
        uint256 externalFee = feesCollected.mulWadDown(devFee);

        SUSD.safeTransfer(feeReceipient, externalFee);

        usedFunds -= int256(tradeCost);
        totalFunds += feesCollected - externalFee;
        ...
    }
```

## [02] `VaultToken.setVault` FUNCTION IS CALLABLE BY ANYONE, AND DEV TEAM'S `VaultToken.setVault` TRANSACTION CAN BE FRONTRUN BY MALICIOUS ACTOR
The following `VaultToken.setVault` function is callable by anyone, and the dev team's `VaultToken.setVault` transaction can be frontrun by a malicious actor, which can cause `vault` to be set to an address that is not the deployed `KangarooVault` contract. When this occurs, calling the `KangarooVault` contract's functions that further call the `VaultToken.mint` or `VaultToken.burn` function would revert due to the `onlyVault` modifier. Hence, such functionalities of the `KangarooVault` contract are DOS'ed.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L35-L40
```solidity
    function setVault(address _vault) external {
        if (vault != address(0x0)) {
            revert();
        }
        vault = _vault;
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L27-L33
```solidity
    function mint(address _user, uint256 _amt) external onlyVault {
        _mint(_user, _amt);
    }

    function burn(address _user, uint256 _amt) external onlyVault {
        _burn(_user, _amt);
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L18-L23
```solidity
    modifier onlyVault() {
        if (msg.sender != vault) {
            revert OnlyVault(address(this), msg.sender, address(vault));
        }
        _;
    }
```

If users are not promptly notified about this situation, they can interact with the deployed `KangarooVault` contract, which can cause users' transactions to revert or lose funds unexpectedly. For example, after a user calls the `KangarooVault._openPosition` function to open a position, another user can call the `KangarooVault.initiateDeposit` function to queue a deposit request, which transfers `SUSD` tokens from such user to the deployed `KangarooVault` contract. Later, calling the `KangarooVault.processDepositQueue` function will revert because executing `VAULT_TOKEN.mint(current.user, tokensToMint)` reverts. As a result, such user loses the transferred `SUSD` tokens, which are locked in the deployed `KangarooVault` contract.

As a mitigation, the `VaultToken.setVault` function can be updated to be only callable by trusted admin.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L556-L611
```solidity
    function _openPosition(uint256 amt, uint256 minCost) internal {
        ...
        if (positionData.positionId == 0) {
            positionData.positionId = positionId;
        }
        ...
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L183-L208
```solidity
    function initiateDeposit(address user, uint256 amount) external nonReentrant {
        ...
        // Instant processing
        if (positionData.positionId == 0) {
            ...
        } else {
            // Queueing the deposit request
            QueuedDeposit storage newDeposit = depositQueue[nextQueuedDepositId];

            newDeposit.id = nextQueuedDepositId++;
            newDeposit.user = user;
            newDeposit.depositedAmount = amount;
            newDeposit.requestedTime = block.timestamp;

            totalQueuedDeposits += amount;
            emit InitiateDeposit(newDeposit.id, msg.sender, user, amount);
        }

        SUSD.safeTransferFrom(msg.sender, address(this), amount);
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L243-L265
```solidity
    function processDepositQueue(uint256 idCount) external nonReentrant {
        uint256 tokenPrice = getTokenPrice();

        for (uint256 i = 0; i < idCount; i++) {
            QueuedDeposit storage current = depositQueue[queuedDepositHead];

            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
                return;
            }

            uint256 tokensToMint = current.depositedAmount.divWadDown(tokenPrice);

            current.mintedTokens = tokensToMint;
            totalQueuedDeposits -= current.depositedAmount;
            totalFunds += current.depositedAmount;
            VAULT_TOKEN.mint(current.user, tokensToMint);
            ...
        }
    }
```

## [03] ALLOWING `ShortCollateral.refresh` FUNCTION TO BE CALLABLE BY ANYONE CAN BE DANGEROUS
The following `ShortCollateral.refresh` function is callable by anyone. If the external `synthetix` contract is hacked or malfunctions, calling `synthetix.synths(key)` can return an untrusted address. When this happens, anyone can call `ShortCollateral.refresh` function to set `susd.synth` to such untrusted address. As a result, all functionalities that rely on `susd.synth` will become unreliable and insecure.

As a mitigation, the `ShortCollateral.refresh` function can be updated to be only callable by trusted admin.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L64-L70
```solidity
    function refresh() public {
        ...
        Collateral storage susd = collaterals["sUSD"];
        susd.synth = synthetixAdapter.getSynth("sUSD");
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SynthetixAdapter.sol#L18-L20
```solidity
    function getSynth(bytes32 key) public view override returns (address synth) {
        synth = synthetix.synths(key);
    }
```

## [04] SOME FUNCTIONS DO NOT FOLLOW CHECKS-EFFECTS-INTERACTIONS PATTERN
Functions like `LiquidityPool.withdraw` and `ShortCollateral.collectCollateral` below transfer the corresponding tokens before updating the relevant states, which do not follow the checks-effects-interactions pattern. In contrast, functions like `LiquidityPool.deposit` and `ShortCollateral.sendCollateral` below transfer the corresponding tokens after updating the relevant states. To reduce the potential attack surface and increase the level of security, please consider updating the functions that do not follow the checks-effects-interactions pattern to follow such pattern.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L247-L259
```solidity
    function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {
        ...
        SUSD.safeTransfer(feeReceipient, fees);
        SUSD.transfer(user, susdToReturn - fees);
        totalFunds -= susdToReturn;
        liquidityToken.burn(msg.sender, tokens);
        ...
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L85-L101
```solidity
    function collectCollateral(address collateral, uint256 positionId, uint256 amount)
        external
        onlyExchange
        nonReentrant
    {
        ERC20(collateral).safeTransferFrom(address(exchange), address(this), amount);

        UserCollateral storage userCollateral = userCollaterals[positionId];

        if (userCollateral.collateral == address(0x0)) {
            userCollateral.collateral = collateral;
        }

        userCollateral.amount += amount;
        ...
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L184-L195
```solidity
    function deposit(uint256 amount, address user) external override nonReentrant whenNotPaused("POOL_DEPOSIT") {
        uint256 tokenPrice = getTokenPrice();
        uint256 fees = amount.mulWadDown(depositFee);
        uint256 amountForTokens = amount - fees;
        uint256 tokensToMint = amountForTokens.divWadDown(tokenPrice);
        liquidityToken.mint(user, tokensToMint);
        totalFunds += amountForTokens;
        SUSD.safeTransferFrom(msg.sender, feeReceipient, fees);
        SUSD.safeTransferFrom(msg.sender, address(this), amountForTokens);

        emit Deposit(user, amount, fees, tokensToMint);
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L106-L116
```solidity
    function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {
        UserCollateral storage userCollateral = userCollaterals[positionId];

        userCollateral.amount -= amount;

        address user = shortToken.ownerOf(positionId);

        ERC20(userCollateral.collateral).safeTransfer(user, amount);

        emit SendCollateral(positionId, userCollateral.collateral, amount);
    }
```

## [05] MISSING `address(0)` CHECKS FOR CRITICAL ADDRESS INPUTS
( Please note that the following instances are not found in https://gist.github.com/Picodes/c9a016e3df62a3f634ba5d47b49bd1c0#nc-1-missing-checks-for-address0-when-assigning-values-to-address-state-variables. )

To prevent unintended behaviors, critical address inputs should be checked against `address(0)`. `address(0)` checks are missing for the `address` input variables in the following constructor and `init` function. Please consider checking them.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L47-L82
```solidity
    constructor(
        address _addressResolver,
        address _futureMarketManager,
        address _susd,
        bytes32 _baseAsset,
        bytes32 _perpMarketName
    ) Auth(msg.sender, Authority(address(0x0))) {
        addressResolver = IAddressResolver(_addressResolver);
        futuresMarketManager = IFuturesMarketManager(_futureMarketManager);
        ...
        SUSD = ERC20(_susd);
        ...
    }

    function init(
        address _pool,
        address _powerPerp,
        address _exchange,
        address _liquidityToken,
        address _shortToken,
        address _synthetixAdapter,
        address _shortCollateral
    ) public {
        ...
        pool = ILiquidityPool(_pool);
        powerPerp = IPowerPerp(_powerPerp);
        exchange = IExchange(_exchange);
        liquidityToken = ILiquidityToken(_liquidityToken);
        shortToken = IShortToken(_shortToken);
        synthetixAdapter = ISynthetixAdapter(_synthetixAdapter);
        shortCollateral = IShortCollateral(_shortCollateral);
        ...
    }
```

## [06] `nonReentrant` MODIFIER CAN BE PLACED AND EXECUTED BEFORE OTHER MODIFIERS IN FUNCTIONS
As a best practice, the `nonReentrant` modifier could be placed and executed before other modifiers in functions to prevent reentrancies through other modifiers and make code more efficient. To follow the best practice, please consider placing the `nonReentrant` modifier before the `requiresAuth` modifier in the following functions.

```solidity
src\KangarooVault.sol
  376: function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {    
  383: function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {    
  389: function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {    
  395: function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {    
  401: function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {    
  424: function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {    
  436: function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {    
  450: function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```

## [07] MISSING REASON STRING IN `revert` STATEMENT
( Please note that the following instance is not found in https://gist.github.com/Picodes/c9a016e3df62a3f634ba5d47b49bd1c0#nc-2--requirerevertstatements-should-have-descriptive-reason-strings. )

When the reason string is missing in the `revert` statement, it is unclear about why certain condition reverts. Please add a descriptive reason string for the following `revert` statement.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L35-L40
```solidity
    function setVault(address _vault) external {
        if (vault != address(0x0)) {
            revert();
        }
        vault = _vault;
    }
```

## [08] REDUNDANT `return` KEYWORDS IN `ShortToken.transferFrom` and `ShortToken.safeTransferFrom` FUNCTIONS
The following `ShortToken.transferFrom` and `ShortToken.safeTransferFrom` functions do not have `returns` but have `return` statements. Moreover, Solmate's corresponding `ERC721.transferFrom` and `ERC721.safeTransferFrom` functions do not return anything. Thus, these `ShortToken.transferFrom` and `ShortToken.safeTransferFrom` functions' `return` keywords are redundant. To improve the code quality, please consider removing the `return` keywords from these functions.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L92-L105
```solidity
    function transferFrom(address _from, address _to, uint256 _id) public override {
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
        return ERC721.transferFrom(_from, _to, _id);
    }

    function safeTransferFrom(address _from, address _to, uint256 _id) public override {
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
        return ERC721.safeTransferFrom(_from, _to, _id);
    }

    function safeTransferFrom(address _from, address _to, uint256 _id, bytes calldata data) public override {
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
        return ERC721.safeTransferFrom(_from, _to, _id, data);
    }
```

## [09] CONSTANTS CAN BE USED INSTEAD OF MAGIC NUMBERS
To improve readability and maintainability, a constant can be used instead of the magic number. Please consider replacing the magic numbers, such as `1e18`, used in the following code with constants.

```solidity
src\Exchange.sol
  191: fundingRate = fundingRate / 1 days; 
  197: int256 normalizationUpdate = 1e18 - totalFunding;   

src\ShortCollateral.sol
  235: if (safetyRatio > 1e18) return maxDebt; 
  237: maxDebt = position.shortAmount / 2; 
```

## [10] HARDCODED STRING THAT IS REPEATEDLY USED CAN BE REPLACED WITH A CONSTANT
`sUSD` is repeatedly used in the `ShortCollateral` contract. For better maintainability, please consider replacing it with a constant.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L51-L70
```solidity
    constructor(uint256 susdRatio, uint256 susdLiqRatio, uint256 susdLiqBonus, ISystemManager _systemManager)
        Auth(msg.sender, Authority(address(0x0)))
    {
        ...
        Collateral storage susd = collaterals["sUSD"];
        susd.currencyKey = "sUSD";
        ...
    }

    function refresh() public {
        ...
        Collateral storage susd = collaterals["sUSD"];
        susd.synth = synthetixAdapter.getSynth("sUSD");
    }
```

## [11] `ShortToken.adjustPosition` FUNCTION DOES NOT NEED TO UPDATE `totalShorts` and `position.shortAmount` IN CERTAIN CASE
When calling the following `ShortToken.adjustPosition` function, if `positionId == 0` is false and `shortAmount` equals `position.shortAmount`, `totalShorts` and `position.shortAmount` will be unchanged. Hence, to increase the code's efficiency, this function can be updated to not update `totalShorts` and `position.shortAmount` in this case.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L48-L90
```solidity
    function adjustPosition(
        uint256 positionId,
        address trader,
        address collateral,
        uint256 shortAmount,
        uint256 collateralAmount
    ) external onlyExchange returns (uint256) {
        if (positionId == 0) {
            ...
        } else {
            require(trader == ownerOf(positionId));

            ShortPosition storage position = shortPositions[positionId];

            if (shortAmount >= position.shortAmount) {
                totalShorts += shortAmount - position.shortAmount;
            } else {
                totalShorts -= position.shortAmount - shortAmount;
            }

            position.collateralAmount = collateralAmount;
            position.shortAmount = shortAmount;

            if (position.shortAmount == 0) {
                _burn(positionId);
            }
        }
        ...
    }
```

## [12] `LiquidityPool.withdraw` FUNCTION CALLS BOTH `SUSD.safeTransfer` AND `SUSD.transfer`
The `LiquidityPool.withdraw` function calls `SUSD.safeTransfer(feeReceipient, fees)` and `SUSD.transfer(user, susdToReturn - fees)`. For consistency and a higher level of security, the `LiquidityPool.withdraw` function can be updated to call `SUSD.safeTransfer(user, susdToReturn - fees)` instead of `SUSD.transfer(user, susdToReturn - fees)`.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L247-L259
```solidity
    function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {
        ...
        SUSD.safeTransfer(feeReceipient, fees);
        SUSD.transfer(user, susdToReturn - fees);
        ...
    }
```

## [13] `LiquidityPool.orderFee` FUNCTION CAN CALL `getMarkPrice()` INSTEAD OF `exchange.getMarkPrice()`
The following `LiquidityPool.orderFee` function calls `exchange.getMarkPrice()` while all other functions in the same contract that need to call `exchange.getMarkPrice()`, such as the `LiquidityPool.getTokenPrice` function below, call `getMarkPrice()` instead. To make code more consistent and better, please consider updating the `LiquidityPool.orderFee` function to call `getMarkPrice()` instead of `exchange.getMarkPrice()`.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L379-L396
```solidity
    function orderFee(int256 sizeDelta) public view override returns (uint256) {
        ...
        (uint256 markPrice,) = exchange.getMarkPrice();
        ...
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L340-L365
```solidity
    function getTokenPrice() public view override returns (uint256) {
        ...
        (uint256 markPrice, bool isInvalid) = getMarkPrice();
        ...
    }
```

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L404-L406
```solidity
    function getMarkPrice() public view override returns (uint256, bool) {
        return exchange.getMarkPrice();
    }
```

## [14] IMMUTABLES CAN BE NAMED USING SAME CONVENTION
As shown below, some immutables are named using capital letters and underscores while some are not. For a better code quality, please consider naming these immutables using the same naming convention.

```solidity
src\Exchange.sol
  34: uint256 public immutable override PRICING_CONSTANT;

src\KangarooVault.sol
  60: bytes32 public immutable name;
  63: bytes32 public immutable UNDERLYING_SYNTH_KEY;
  66: ERC20 public immutable SUSD;
  69: IVaultToken public immutable VAULT_TOKEN;
  72: IExchange public immutable EXCHANGE;
  75: ILiquidityPool public immutable LIQUIDITY_POOL;
  78: IPerpsV2Market public immutable PERP_MARKET;

src\LiquidityPool.sol
  56: bytes32 public immutable baseAsset;
  65: ERC20 public immutable SUSD;

src\LiquidityToken.sol
  8: bytes32 public immutable marketKey;
  10: ISystemManager public immutable systemManager;

src\PowerPerp.sol
  9: bytes32 public immutable marketKey;
  10: ISystemManager public immutable systemManager;

src\ShortCollateral.sol
  46: ISystemManager public immutable systemManager;

src\ShortToken.sol
  9: bytes32 public immutable marketKey;
  11: ISystemManager public immutable systemManager;

src\SystemManager.sol
  21: bytes32 public immutable baseAsset;
  24: ERC20 public immutable SUSD;
  27: bytes32 public immutable PERP_MARKET_CONTRACT;
```

## [15] UNUSED IMPORT
The `IFuturesMarket` interface is not used in the `SystemManager` contract. Please consider removing the corresponding `import` statement for better readability and maintainability.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L14-L19
```solidity
import {IFuturesMarket} from "./interfaces/synthetix/IFuturesMarket.sol";
...
contract SystemManager is ISystemManager, Auth {
```

## [16] FLOATING PRAGMAS
It is a best practice to lock pragmas instead of using floating pragmas to ensure that contracts are tested and deployed with the intended compiler version. Accidentally deploying contracts with different compiler versions can lead to unexpected risks and undiscovered bugs. Please consider locking pragmas for the following files.

```solidity
src\Exchange.sol
  2: pragma solidity ^0.8.9; 

src\KangarooVault.sol
  2: pragma solidity ^0.8.9; 

src\LiquidityPool.sol
  2: pragma solidity ^0.8.9; 

src\LiquidityToken.sol
  2: pragma solidity ^0.8.9; 

src\PowerPerp.sol
  2: pragma solidity ^0.8.9; 

src\ShortCollateral.sol
  2: pragma solidity ^0.8.9; 

src\ShortToken.sol
  2: pragma solidity ^0.8.9; 

src\SynthetixAdapter.sol
  2: pragma solidity ^0.8.9; 

src\SystemManager.sol
  2: pragma solidity ^0.8.9; 

src\libraries\SignedMath.sol
  2: pragma solidity ^0.8.9; 

src\utils\PauseModifier.sol
  3: pragma solidity ^0.8.9; 
```

## [17] SOLIDITY VERSION `0.8.19` CAN BE USED
Using the more updated version of Solidity can enhance security. As described in https://github.com/ethereum/solidity/releases, Version `0.8.19` is the latest version of Solidity, which "contains a fix for a long-standing bug that can result in code that is only used in creation code to also be included in runtime bytecode". To be more secured and more future-proofed, please consider using Version `0.8.19` for all contracts, including the `VaultToken` contract that uses Version `0.8.9` currently.

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L2
```solidity
pragma solidity 0.8.9;
```

## [18] ORDERS OF LAYOUT DO NOT FOLLOW OFFICIAL STYLE GUIDE
https://docs.soliditylang.org/en/v0.8.19/style-guide.html#order-of-layout suggests that the following order should be used in a contract:
1. Type declarations
2. State variables
3. Events
4. Errors
5. Modifiers
6. Functions

Events or error are placed after functions in the following contracts. To follow the official style guide, please consider placing these events or error before all functions in these contracts.

```solidity
src\ShortCollateral.sol
  16: contract ShortCollateral is IShortCollateral, Auth, ReentrancyGuard {    

src\ShortToken.sol
  8: contract ShortToken is ERC721 {  

src\SystemManager.sol
  19: contract SystemManager is ISystemManager, Auth {  

src\VaultToken.sol
  6: contract VaultToken is ERC20 {  
```

## [19] INCOMPLETE NATSPEC COMMENTS
NatSpec comments provide rich code documentation. The following functions miss the `@param` and/or `@return` comments. Please consider completing the NatSpec comments for these functions.

```solidity
src\Exchange.sol
  87: function openTrade(TradeParams memory tradeParams)  
  155: function getIndexPrice() public view override returns (uint256 indexPrice, bool isInvalid) {    
  186: function getMarkPrice() public view override returns (uint256 markPrice, bool isInvalid) {  
  233: function _openTrade(TradeParams memory params) internal returns (uint256, uint256) {    

src\LiquidityPool.sol
  379: function orderFee(int256 sizeDelta) public view override returns (uint256) {    
  399: function baseAssetPrice() public view override returns (uint256 spotPrice, bool isInvalid) {    
  409: function getSkew() external view override returns (int256) {    
  430: function openLong(uint256 amount, address user, bytes32 referralCode)   
  720: function _getSkew() internal view returns (int256 skew) {   
  727: function _getDelta() internal view returns (uint256 delta) {   
  798: function _hedge(int256 size, bool isLiquidation) internal returns (uint256 hedgingFees) {   

src\ShortCollateral.sol
  121: function liquidate(uint256 positionId, uint256 debt, address user)  
  153: function getMinCollateral(uint256 shortAmount, address collateral)  
  176: function getLiquidationBonus(address collateral, uint256 collateralAmount)  
  192: function canLiquidate(uint256 positionId) public view override returns (bool) { 
  215: function maxLiquidatableDebt(uint256 positionId) public view override returns (uint256 maxDebt) {
```

## [20] MISSING NATSPEC COMMENTS
NatSpec comments provide rich code documentation. The following functions miss NatSpec comments. Please consider adding NatSpec comments for these functions.

```solidity
src\LiquidityToken.sol
  19: function refresh() public { 
  28: function mint(address _user, uint256 _amt) public onlyPool { 
  32: function burn(address _user, uint256 _amt) public onlyPool { 

src\PowerPerp.sol
  22: function refresh() public { 
  32: function mint(address _user, uint256 _amt) public onlyExchange { 
  36: function burn(address _user, uint256 _amt) public onlyExchange { 
  40: function transfer(address _to, uint256 _amount) public override returns (bool) { 
  45: function transferFrom(address _from, address _to, uint256 _amount) public override returns (bool) { 

src\ShortToken.sol
  33: function refresh() public {
  44: function tokenURI(uint256 tokenId) public view override returns (string memory) { 
  48: function adjustPosition(    
  92: function transferFrom(address _from, address _to, uint256 _id) public override { 
  97: function safeTransferFrom(address _from, address _to, uint256 _id) public override { 
  102: function safeTransferFrom(address _from, address _to, uint256 _id, bytes calldata data) public override { 

src\SynthetixAdapter.sol
  18: function getSynth(bytes32 key) public view override returns (address synth) {   
  22: function getCurrencyKey(address synth) public view override returns (bytes32 key) {   
  26: function getAssetPrice(bytes32 key) public view override returns (uint256, bool) {   

src\SystemManager.sol
  62: function init(  
  84: function refreshSynthetixAddresses() public {  
  89: function setStatusFunction(bytes32 key, bool status) public requiresAuth {

src\VaultToken.sol
  27: function mint(address _user, uint256 _amt) external onlyVault { 
  31: function burn(address _user, uint256 _amt) external onlyVault { 
  35: function setVault(address _vault) external {    

src\libraries\SignedMath.sol
  5: function signedAbs(int256 x) internal pure returns (int256) {   
  9: function abs(int256 x) internal pure returns (uint256) {   
  13: function max(int256 x, int256 y) internal pure returns (int256) {   
  17: function min(int256 x, int256 y) internal pure returns (int256) {   
```