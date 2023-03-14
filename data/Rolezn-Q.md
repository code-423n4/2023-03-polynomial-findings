## Summary<a name="Summary">

### Low Risk Issues
| |Issue|Contexts|
|-|:-|:-:|
| [LOW&#x2011;1](#LOW&#x2011;1) | Add to `blacklist` function | 1 |
| [LOW&#x2011;2](#LOW&#x2011;2) | Do not allow fees to be set to `100%` | 4 |
| [LOW&#x2011;3](#LOW&#x2011;3) | Event is missing parameters | 2 |
| [LOW&#x2011;4](#LOW&#x2011;4) | Init functions are susceptible to front-running | 3 |
| [LOW&#x2011;5](#LOW&#x2011;5) | Minting tokens to the zero address should be avoided | 1 |
| [LOW&#x2011;6](#LOW&#x2011;6) | Use `_safeMint` instead of `_mint` | 1 |
| [LOW&#x2011;7](#LOW&#x2011;7) | Missing Contract-existence Checks Before Low-level Calls | 2 |
| [LOW&#x2011;8](#LOW&#x2011;8) | The `nonReentrant` modifier should occur before all other modifiers | 17 |
| [LOW&#x2011;9](#LOW&#x2011;9) | Protect your NFT from copying in POW forks | 1 |
| [LOW&#x2011;10](#LOW&#x2011;10) | `require()` should be used instead of `assert()` | 2 |
| [LOW&#x2011;11](#LOW&#x2011;11) | Solmate's SafeTransferLib doesn't check whether the ERC20 contract exists | 4 |
| [LOW&#x2011;12](#LOW&#x2011;12) | `tokenURI()` does not follow EIP-721 | 1 |

Total: 39 contexts over 12 issues

### Non-critical Issues
| |Issue|Contexts|
|-|:-|:-:|
| [NC&#x2011;1](#NC&#x2011;1) | Add a timelock to critical functions | 20 |
| [NC&#x2011;2](#NC&#x2011;2) | Avoid Floating Pragmas: The Version Should Be Locked | 11 |
| [NC&#x2011;3](#NC&#x2011;3) | Constants in comparisons should appear on the left side | 6 |
| [NC&#x2011;4](#NC&#x2011;4) | Duplicated `require()`/`revert()` Checks Should Be Refactored To A Modifier Or Function | 58 |
| [NC&#x2011;5](#NC&#x2011;5) | `block.timestamp` is already used when emitting events, no need to input timestamp | 2 |
| [NC&#x2011;6](#NC&#x2011;6) | Function writing that does not comply with the Solidity Style Guide  | 12 |
| [NC&#x2011;7](#NC&#x2011;7) | Large or complicated code bases should implement fuzzing tests | 1 |
| [NC&#x2011;8](#NC&#x2011;8) | Use `delete` to Clear Variables | 13 |
| [NC&#x2011;9](#NC&#x2011;9) | NatSpec return parameters should be included in contracts | 1 |
| [NC&#x2011;10](#NC&#x2011;10) | Initial value check is missing in Set Functions | 20 |
| [NC&#x2011;11](#NC&#x2011;11) | Contracts should have full test coverage | 1 |
| [NC&#x2011;12](#NC&#x2011;12) | Missing event for critical parameter change | 2 |
| [NC&#x2011;13](#NC&#x2011;13) | Implementation contract may not be initialized | 10 |
| [NC&#x2011;14](#NC&#x2011;14) | NatSpec comments should be increased in contracts | 1 |
| [NC&#x2011;15](#NC&#x2011;15) | Use a more recent version of Solidity | 12 |
| [NC&#x2011;16](#NC&#x2011;16) | Public Functions Not Called By The Contract Should Be Declared External Instead | 7 |
| [NC&#x2011;17](#NC&#x2011;17) | Remove `forge-std` import | 1 |

Total: 178 contexts over 17 issues

## Low Risk Issues

### <a href="#Summary">[LOW&#x2011;1]</a><a name="LOW&#x2011;1"> Add to `blacklist` function

NFT thefts have increased recently, so with the addition of hacked NFTs to the platform, NFTs can be converted into liquidity. To prevent this, I recommend adding the blacklist function.

Marketplaces such as Opensea have a blacklist feature that will not list NFTs that have been reported theft, NFT projects such as Manifold have blacklist functions in their smart contracts.

Here is the project example; Manifold

Manifold Contract https://etherscan.io/address/0xe4e4003afe3765aca8149a82fc064c0b125b9e5a#code

```solidity
     modifier nonBlacklistRequired(address extension) {
         require(!_blacklistedExtensions.contains(extension), "Extension blacklisted");
         _;
     }
```


#### <ins>Recommended Mitigation Steps</ins>
Add to Blacklist function and modifier.



### <a href="#Summary">[LOW&#x2011;2]</a><a name="LOW&#x2011;2"> Do not allow fees to be set to `100%`

It is recommended from a risk perspective to disallow setting 100% fees at all. A reasonable fee maximum should be checked for instead.

#### <ins>Proof Of Concept</ins>


```solidity

function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
        emit UpdateFees(depositFee, _depositFee, withdrawalFee, _withdrawalFee);
        depositFee = _depositFee;
        withdrawalFee = _withdrawalFee;
    }

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L657

```solidity

function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
        emit UpdateBaseTradingFee(baseTradingFee, _baseTradingFee);
        baseTradingFee = _baseTradingFee;
    }

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L665

```solidity

function setDevFee(uint256 _devFee) external requiresAuth {
        emit UpdateDevFee(devFee, _devFee);
        devFee = _devFee;
    }

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L672







### <a href="#Summary">[LOW&#x2011;3]</a><a name="LOW&#x2011;3"> Event is missing parameters

The following functions are missing critical parameters when emitting an event.
When dealing with source address which uses the value of `msg.sender`, the `msg.sender` value must be specified in every transaction, a contract or web page listening to events cannot react to users, emit does not serve the purpose. Basically, this event cannot be used.

#### <ins>Proof Of Concept</ins>


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

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L194

```solidity

function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {
        require(liquidityToken.balanceOf(msg.sender) >= tokens);

        uint256 tokenPrice = getTokenPrice();
        uint256 susdToReturn = tokens.mulWadDown(tokenPrice);
        uint256 fees = susdToReturn.mulWadDown(withdrawalFee);
        SUSD.safeTransfer(feeReceipient, fees);
        SUSD.transfer(user, susdToReturn - fees);
        totalFunds -= susdToReturn;
        liquidityToken.burn(msg.sender, tokens);

        emit Withdraw(user, tokens, fees, susdToReturn);
    }

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L258




#### <ins>Recommended Mitigation Steps</ins>
Add `msg.sender` parameter in event-emit



### <a href="#Summary">[LOW&#x2011;4]</a><a name="LOW&#x2011;4"> Init functions are susceptible to front-running

Most contracts use an init pattern (instead of a constructor) to initialize contract parameters. Unless these are enforced to be atomic with contact deployment via deployment script or factory contracts, they are susceptible to front-running race conditions where an attacker/griefer can front-run (cannot access control because admin roles are not initialized) to initially with their own (malicious) parameters upon detecting (if an event is emitted) which the contract deployer has to redeploy wasting gas and risking other transactions from interacting with the attacker-initialized contract.

Many init functions do not have an explicit event emission which makes monitoring such scenarios harder. All of them have re-init checks; while many are explicit some (those in auction contracts) have implicit reinit checks in initAccessControls() which is better if converted to an explicit check in the main init function itself.
(details credit to: code-423n4/2021-09-sushimiso-findings#64)

#### <ins>Proof Of Concept</ins>


```solidity
    function init(
        address _pool,
        address _powerPerp,
        address _exchange,
        address _liquidityToken,
        address _shortToken,
        address _synthetixAdapter,
        address _shortCollateral
    ) public {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol#L62



#### <ins>Recommended Mitigation Steps</ins>

Ensure atomic calls to init functions along with deployment via robust deployment scripts or factory contracts. Emit explicit events for initializations of contracts. Enforce prevention of re-initializations via explicit setting/checking of boolean initialized variables in the main init function instead of downstream function checks.





### <a href="#Summary">[LOW&#x2011;5]</a><a name="LOW&#x2011;5"> Minting tokens to the zero address should be avoided

The core function `mint` is used by users to mint an option position by providing token1 as collateral and borrowing the max amount of liquidity. `Address(0)` check is missing in both this function and the internal function `_mint`, which is triggered to mint the tokens to the to address. Consider applying a check in the function to ensure tokens aren't minted to the zero address.

XXX CHECK _MINT also doesn't contain zero address check XXX

#### <ins>Proof Of Concept</ins>


```solidity

function mint(address _user, uint256 _amt) external onlyVault {
        _mint(_user, _amt);
    }

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/VaultToken.sol#L27







### <a href="#Summary">[LOW&#x2011;6]</a><a name="LOW&#x2011;6"> Use `_safeMint` instead of `_mint`

According to openzepplin's ERC721, the use of `_mint` is discouraged, use _safeMint whenever possible.
https://docs.openzeppelin.com/contracts/3.x/api/token/erc721#ERC721-_mint-address-uint256-

#### <ins>Proof Of Concept</ins>


```solidity
67: _mint(trader, positionId);
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L67



#### <ins>Recommended Mitigation Steps</ins>

Use `_safeMint` whenever possible instead of `_mint`



### <a href="#Summary">[LOW&#x2011;7]</a><a name="LOW&#x2011;7"> Missing Contract-existence Checks Before Low-level Calls

Low-level calls return success if there is no code present at the specified address. 

#### <ins>Proof Of Concept</ins>


```solidity
455: (bool success,) = feeReceipient.call{value: msg.value}("");
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L455

```solidity
709: (bool success,) = feeReceipient.call{value: msg.value}("");
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L709




#### <ins>Recommended Mitigation Steps</ins>

In addition to the zero-address checks, add a check to verify that `<address>.code.length > 0`




### <a href="#Summary">[LOW&#x2011;8]</a><a name="LOW&#x2011;8"> The `nonReentrant` modifier should occur before all other modifiers

Currently the `nonReentrant` modifier is not the first to occur, it should occur before all other modifiers.

#### <ins>Proof Of Concept</ins>


```solidity
376: function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L376

```solidity
383: function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L383

```solidity
389: function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L389

```solidity
395: function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L395

```solidity
401: function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L401

```solidity
424: function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L424

```solidity
436: function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L436

```solidity
450: function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L450

```solidity
557: function liquidate(uint256 amount) external override onlyExchange nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L557

```solidity
568: function hedgePositions() external override requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L568

```solidity
591: function rebalanceMargin(int256 marginDelta) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L591

```solidity
613: function increaseMargin(uint256 additionalMargin) external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L613

```solidity
692: function placeQueuedOrder() external requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L692

```solidity
704: function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L704

```solidity
85: function collectCollateral(address collateral, uint256 positionId, uint256 amount)
        external
        onlyExchange
        nonReentrant
    {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L85

```solidity
106: function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L106

```solidity
121: function liquidate(uint256 positionId, uint256 debt, address user)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCollateralReturned)
    {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L121



#### <ins>Recommended Mitigation Steps</ins>

Re-sort modifiers so that the `nonReentrant` modifier occurs first.



### <a href="#Summary">[LOW&#x2011;9]</a><a name="LOW&#x2011;9"> Protect your NFT from copying in POW forks
Ethereum has performed the long-awaited "merge" that will dramatically reduce the environmental impact of the network

There may be forked versions of Ethereum, which could cause confusion and lead to scams as duplicated NFT assets enter the market.

If the Ethereum Merge, which took place in September 2022, results in the Blockchain splitting into two Blockchains due to the 'THE DAO' attack in 2016, this could result in duplication of immutable tokens (NFTs).

In any case, duplicate NFTs will exist due to the ETH proof-of-work chain and other potential forks, and there’s likely to be some level of confusion around which assets are 'official' or 'authentic.'

Even so, there could be a frenzy for these copies, as NFT owners attempt to flip the proof-of-work versions of their valuable tokens.

As ETHPOW and any other forks spin off of the Ethereum mainnet, they will yield duplicate versions of Ethereum’s NFTs. An NFT is simply a blockchain token, and it can work as a deed of ownership to digital items like artwork and collectibles. A forked Ethereum chain will thus have duplicated deeds that point to the same tokenURI

About Merge Replay Attack: https://twitter.com/elerium115/status/1558471934924431363?s=20&t=RRheaYJwo-GmSnePwofgag

#### <ins>Proof Of Concept</ins>


```solidity
44: function tokenURI(uint256 tokenId) public view override returns (string memory) {

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L44



#### <ins>Recommended Mitigation Steps</ins>

Add the following check:
```solidity
if(block.chainid != 1) { 
    revert(); 
}
```



### <a href="#Summary">[LOW&#x2011;10]</a><a name="LOW&#x2011;10"> `require()` Should Be Used Instead Of `assert()`

Prior to solidity version 0.8.0, hitting an assert consumes the remainder of the transaction's available gas rather than returning it, as `require()`/`revert()` do. `assert()` should be avoided even past solidity version 0.8.0 as its <a href="https://docs.soliditylang.org/en/v0.8.14/control-structures.html#panic-via-assert-and-error-via-require">documentation</a> states that "The assert function creates an error of type Panic(uint256). ... Properly functioning code should never create a Panic, not even on invalid external input. If this happens, then there is a bug in your contract which you should fix".

#### <ins>Proof Of Concept</ins>


```solidity
220: assert(queuedDepositHead + count - 1 < nextQueuedDepositId);

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L220

```solidity
285: assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L285







### <a href="#Summary">[LOW&#x2011;11]</a><a name="LOW&#x2011;11"> Solmate's SafeTransferLib doesn't check whether the ERC20 contract exists

Solmate's SafeTransferLib, which is often used to interact with non-compliant/unsafe ERC20 tokens, does not check whether the ERC20 contract exists. The following code will not revert in case the token doesn't exist (yet).

This is stated in the Solmate library: https://github.com/transmissions11/solmate/blob/main/src/utils/SafeTransferLib.sol#L9

#### <ins>Proof Of Concept</ins>


```solidity
8: import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L8

```solidity
10: import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L10

```solidity
9: import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L9

```solidity
8: import {SafeTransferLib} from "solmate/utils/SafeTransferLib.sol";

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L8




#### <ins>Recommended Mitigation Steps</ins>
Add a contract exist control in functions that use solmate's `SafeTransferLib`

```solidity
pragma solidity >=0.8.0;

function isContract(address _addr) private returns (bool isContract) {
    isContract = _addr.code.length > 0;
}
```



### <a href="#Summary">[LOW&#x2011;12]</a><a name="LOW&#x2011;12"> `tokenURI()` does not follow EIP-721

The <a href="https://eips.ethereum.org/EIPS/eip-721">EIP</a> states that `tokenURI()` "Throws if `_tokenId` is not a valid NFT", which the code below does not do. If the NFT has not yet been minted, `tokenURI()` should revert

#### <ins>Proof Of Concept</ins>


```solidity
44: function tokenURI(uint256 tokenId) public view override returns (string memory) {
        return "";
    }

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L44




## Non Critical Issues

### <a href="#Summary">[NC&#x2011;1]</a><a name="NC&#x2011;1"> Add a timelock to critical functions

It is a good practice to give time for users to react and adjust to critical changes. A timelock provides more guarantees and reduces the level of trust required, thus decreasing risk for users. It also indicates that the project is legitimate (less risk of a malicious owner making a sandwich attack on a user). Consider adding a timelock to the following functions:

#### <ins>Proof Of Concept</ins>


```solidity
216: function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L216

```solidity
223: function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L223

```solidity
465: function setFeeReceipient(address _feeReceipient) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L465

```solidity
476: function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L476

```solidity
487: function setSynthetixTracking(bytes32 _code) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L487

```solidity
492: function setReferralCode(bytes32 _code) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L492

```solidity
499: function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L499

```solidity
506: function setMaxDepositAmount(uint256 _maxAmt) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L506

```solidity
514: function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L514

```solidity
522: function setPriceImpactDelta(uint256 _delta) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L522

```solidity
529: function setLeverage(uint256 _lev) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L529

```solidity
537: function setCollRatio(uint256 _ratio) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L537

```solidity
650: function setFeeReceipient(address _feeReceipient) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L650

```solidity
657: function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L657

```solidity
665: function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L665

```solidity
672: function setDevFee(uint256 _devFee) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L672

```solidity
679: function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L679

```solidity
685: function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L685

```solidity
89: function setStatusFunction(bytes32 key, bool status) public requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol#L89

```solidity
35: function setVault(address _vault) external {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/VaultToken.sol#L35







### <a href="#Summary">[NC&#x2011;2]</a><a name="NC&#x2011;2"> Avoid Floating Pragmas: The Version Should Be Locked

Avoid floating pragmas for non-library contracts.

While floating pragmas make sense for libraries to allow them to be included with multiple different versions of applications, it may be a security risk for application implementations.

A known vulnerable compiler version may accidentally be selected or security tools might fall-back to an older compiler version ending up checking a different EVM compilation that is ultimately deployed on the blockchain.

It is recommended to pin to a concrete compiler version.

#### <ins>Proof Of Concept</ins>

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [Exchange.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [KangarooVault.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [LiquidityPool.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [LiquidityToken.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityToken.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [PowerPerp.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/PowerPerp.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [ShortCollateral.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [ShortToken.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [SynthetixAdapter.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SynthetixAdapter.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [SystemManager.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [SignedMath.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/libraries/SignedMath.sol#L2

```solidity
Found usage of floating pragmas ^0.8.9 of Solidity in [PauseModifier.sol]
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/utils/PauseModifier.sol#L3







### <a href="#Summary">[NC&#x2011;3]</a><a name="NC&#x2011;3"> Constants in comparisons should appear on the left side

Doing so will prevent <a href="https://www.moserware.com/2008/01/constants-on-left-are-better-but-this.html">typo bugs</a>

#### <ins>Proof Of Concept</ins>

```solidity
281: if (availableFunds == 0) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L281

```solidity
590: if (positionData.positionId == 0) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L590

```solidity
297: if (availableFunds == 0) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L297

```solidity
348: if (skew == 0) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L348

```solidity
55: if (positionId == 0) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L55

```solidity
82: if (position.shortAmount == 0) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L82


#### <a href="#Summary">[NC&#x2011;4]</a><a name="NC&#x2011;4"> Duplicated `require()`/`revert()` Checks Should Be Refactored To A Modifier Or Function

Saves deployment costs

#### <ins>Proof Of Concept</ins>

```solidity
240: require(totalCost <= maxCost);
322: require(totalCost <= maxCost);

264: require(totalCollateralAmount >= minCollateral, "Not enough collateral");
314: require(totalCollateralAmount >= minCollateral, "Not enough collateral");

277: require(totalCost >= minCost);
294: require(totalCost >= minCost);

357: require(positionId != 0);
383: require(positionId != 0);

360: require(!isInvalid);
386: require(!isInvalid);

362: require(shortToken.ownerOf(positionId) == msg.sender);
388: require(shortToken.ownerOf(positionId) == msg.sender);
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L240

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L322

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L264

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L314

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L277

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L294

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L357

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L383

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L360

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L386

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L362

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L388



```solidity
184: require(user != address(0x0));
216: require(user != address(0x0));

354: require(!isInvalid);
356: require(!isInvalid);

406: require(!isInvalid && baseAssetPrice != 0);
566: require(!isInvalid && baseAssetPrice != 0);

560: require(delayedOrder.sizeDelta == 0);
617: require(delayedOrder.sizeDelta == 0);
674: require(delayedOrder.sizeDelta == 0);
734: require(delayedOrder.sizeDelta == 0);

563: require(position.size >= 0);
677: require(position.size >= 0);
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L184

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L216

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L354

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L356

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L406

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L566

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L560

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L617

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L674

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L734

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L563

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L677



```solidity
248: require(liquidityToken.balanceOf(msg.sender) >= tokens);
270: require(liquidityToken.balanceOf(msg.sender) >= tokens);

353: require(!isInvalid);
386: require(!isInvalid);
438: require(!isInvalid);
470: require(!isInvalid);
502: require(!isInvalid);
534: require(!isInvalid);
559: require(!isInvalid);
805: require(!isInvalid);

517: require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
616: require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
809: require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));

731: require(!isInvalid || spotPrice > 0);
767: require(!isInvalid || spotPrice > 0);
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L248

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L270

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L353

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L386

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L438

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L470

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L502

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L534

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L559

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L805

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L517

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L616

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L809

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L731

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L767



```solidity
41: require(shortToken.balanceOf(_to) == 0, "Receiver has short positions");
46: require(shortToken.balanceOf(_to) == 0, "Receiver has short positions");
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/PowerPerp.sol#L41

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/PowerPerp.sol#L46



```solidity
160: require(currencyKey != "");
183: require(currencyKey != "");

162: require(coll.isApproved);
185: require(coll.isApproved);

164: require(!isInvalid);
167: require(!isInvalid);
194: require(!isInvalid);
205: require(!isInvalid);
217: require(!isInvalid);
228: require(!isInvalid);

197: require(position.shortAmount > 0);
220: require(position.shortAmount > 0);

201: require(collateral.isApproved);
224: require(collateral.isApproved);
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L160

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L183

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L162

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L185

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L164

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L167

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L194

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L205

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L217

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L228

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L197

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L220

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L201

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L224



```solidity
93: require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
98: require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
103: require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L93

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L98

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L103








### <a href="#Summary">[NC&#x2011;5]</a><a name="NC&#x2011;5"> block.timestamp is already used when emitting events, no need to input timestamp

#### <ins>Proof Of Concept</ins>

```solidity
193: emit ProcessDeposit(0, user, amount, tokensToMint, block.timestamp);

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L193

```solidity
224: emit ProcessWithdrawal(0, user, tokens, susdToReturn, block.timestamp);

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L224








### <a href="#Summary">[NC&#x2011;6]</a><a name="NC&#x2011;6"> Function writing that does not comply with the Solidity Style Guide

Order of Functions; ordering helps readers identify which functions they can call and to find the constructor and fallback definitions easier. But there are contracts in the project that do not comply with this.

https://docs.soliditylang.org/en/v0.8.17/style-guide.html

Functions should be grouped according to their visibility and ordered:
- constructor
- receive function (if exists)
- fallback function (if exists)
- external
- public
- internal
- private
- within a grouping, place the view and pure functions last

#### <ins>Proof Of Concept</ins>

Various in-scope contracts

See `KangarooVault.sol` for example



### <a href="#Summary">[NC&#x2011;7]</a><a name="NC&#x2011;7"> Large or complicated code bases should implement fuzzing tests

Large code bases, or code with lots of inline-assembly, complicated math, or complicated interactions between multiple contracts, should implement <a href="https://medium.com/coinmonks/smart-contract-fuzzing-d9b88e0b0a05">fuzzing tests</a>. Fuzzers such as Echidna require the test writer to come up with invariants which should not be violated under any circumstances, and the fuzzer tests various inputs and function calls to ensure that the invariants always hold. Even code with 100% code coverage can still have bugs due to the order of the operations a user performs, and fuzzers, with properly and extensively-written invariants, can close this testing gap significantly.

#### <ins>Proof Of Concept</ins>

Various in-scope contract files.




### <a href="#Summary">[NC&#x2011;8]</a><a name="NC&#x2011;8"> Use `delete` to Clear Variables

`delete a` assigns the initial value for the type to `a`. i.e. for integers it is equivalent to `a = 0`, but it can also be used on arrays, where it assigns a dynamic array of length zero or a static array of the same length with all elements reset. For structs, it assigns a struct with all members reset. Similarly, it can also be used to set an address to zero address. It has no effect on whole mappings though (as the keys of mappings may be arbitrary and are generally unknown). However, individual keys and what they map to can be deleted: If `a` is a mapping, then `delete a[x]` will delete the value stored at `x`.

The `delete` key better conveys the intention and is also more idiomatic. Consider replacing assignments of zero with `delete` statements.

#### <ins>Proof Of Concept</ins>

```solidity
262: current.depositedAmount = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L262

```solidity
328: current.withdrawnTokens = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L328

```solidity
641: positionData.premiumCollected = 0;
667: positionData.pendingLongPerp = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L641

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L667



```solidity
714: positionData.premiumCollected = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L714

```solidity
779: positionData.pendingShortPerp = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L779

```solidity
783: positionData.positionId = 0;
794: positionData.premiumCollected = 0;
795: positionData.totalMargin = 0;
796: usedFunds = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L783

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L794

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L795

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L796



```solidity
239: current.depositedAmount = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L239

```solidity
329: current.withdrawnTokens = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L329

```solidity
701: queuedPerpSize = 0;

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L701






### <a href="#Summary">[NC&#x2011;9]</a><a name="NC&#x2011;9"> NatSpec return parameters should be included in contracts

It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation. In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability.

https://docs.soliditylang.org/en/v0.8.15/natspec-format.html
#### <ins>Proof Of Concept</ins>


Various in-scope contracts

See `Exchange._closeTrade` for example.

#### <ins>Recommended Mitigation Steps</ins>

Include return parameters in NatSpec comments

Recommendation Code Style: (from Uniswap3)

```solidity
    /// @notice Adds liquidity for the given recipient/tickLower/tickUpper position
    /// @dev The caller of this method receives a callback in the form of IUniswapV3MintCallback#uniswapV3MintCallback
    /// in which they must pay any token0 or token1 owed for the liquidity. The amount of token0/token1 due depends
    /// on tickLower, tickUpper, the amount of liquidity, and the current price.
    /// @param recipient The address for which the liquidity will be created
    /// @param tickLower The lower tick of the position in which to add liquidity
    /// @param tickUpper The upper tick of the position in which to add liquidity
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



### <a href="#Summary">[NC&#x2011;10]</a><a name="NC&#x2011;10"> Initial value check is missing in Set Functions

A check regarding whether the current value and the new value are the same should be added

#### <ins>Proof Of Concept</ins>

```solidity
216: function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {
        emit UpdateSkewNormalizationFactor(skewNormalizationFactor, _skewNormalizationFactor);
        skewNormalizationFactor = _skewNormalizationFactor;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L216

```solidity
223: function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
        emit UpdateMaxFundingRate(maxFundingRate, _maxFundingRate);
        maxFundingRate = _maxFundingRate;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L223

```solidity
465: function setFeeReceipient(address _feeReceipient) external requiresAuth {
        require(_feeReceipient != address(0x0));

        emit UpdateFeeReceipient(feeReceipient, _feeReceipient);

        feeReceipient = _feeReceipient;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L465

```solidity
476: function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
        require(_performanceFee <= 1e17 && _withdrawalFee <= 1e16);

        emit UpdateFees(performanceFee, withdrawalFee, _performanceFee, _withdrawalFee);

        performanceFee = _performanceFee;
        withdrawalFee = _withdrawalFee;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L476

```solidity
487: function setSynthetixTracking(bytes32 _code) external requiresAuth {
        emit UpdateSynthetixTrackingCode(synthetixTrackingCode, _code);
        synthetixTrackingCode = _code;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L487

```solidity
492: function setReferralCode(bytes32 _code) external requiresAuth {
        emit UpdateReferralCode(referralCode, _code);
        referralCode = _code;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L492

```solidity
499: function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
        emit UpdateMinDeposit(minDepositAmount, _minAmt);
        minDepositAmount = _minAmt;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L499

```solidity
506: function setMaxDepositAmount(uint256 _maxAmt) external requiresAuth {
        emit UpdateMaxDeposit(maxDepositAmount, _maxAmt);
        maxDepositAmount = _maxAmt;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L506

```solidity
514: function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
        emit UpdateDelays(minDepositDelay, _depositDelay, minWithdrawDelay, _withdrawDelay);
        minDepositDelay = _depositDelay;
        minWithdrawDelay = _withdrawDelay;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L514

```solidity
522: function setPriceImpactDelta(uint256 _delta) external requiresAuth {
        emit UpdatePriceImpactDelta(perpPriceImpact, _delta);
        perpPriceImpact = _delta;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L522

```solidity
529: function setLeverage(uint256 _lev) external requiresAuth {
        require(_lev <= 5e18 && _lev >= 1e18);
        emit UpdateLeverage(leverage, _lev);
        leverage = _lev;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L529

```solidity
537: function setCollRatio(uint256 _ratio) external requiresAuth {
        require(_ratio >= 1.5e18);
        emit UpdateCollRatio(collRatio, _ratio);
        collRatio = _ratio;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L537

```solidity
650: function setFeeReceipient(address _feeReceipient) external requiresAuth {
        feeReceipient = _feeReceipient;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L650

```solidity
657: function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
        emit UpdateFees(depositFee, _depositFee, withdrawalFee, _withdrawalFee);
        depositFee = _depositFee;
        withdrawalFee = _withdrawalFee;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L657

```solidity
665: function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
        emit UpdateBaseTradingFee(baseTradingFee, _baseTradingFee);
        baseTradingFee = _baseTradingFee;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L665

```solidity
672: function setDevFee(uint256 _devFee) external requiresAuth {
        emit UpdateDevFee(devFee, _devFee);
        devFee = _devFee;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L672

```solidity
679: function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
        emit UpdatePriceImpactDelta(perpPriceImpactDelta, _perpPriceImpactDelta);
        perpPriceImpactDelta = _perpPriceImpactDelta;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L679

```solidity
685: function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
        emit UpdateDelays(minDepositDelay, _minDepositDelay, minWithdrawDelay, _minWithdrawDelay);
        minDepositDelay = _minDepositDelay;
        minWithdrawDelay = _minWithdrawDelay;
    }
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L685

```solidity
89: function setStatusFunction(bytes32 key, bool status) public requiresAuth {
        isPaused[key] = status;

        emit SetStatus(key, status);
    }

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol#L89

```solidity
35: function setVault(address _vault) external {
        if (vault != address(0x0)) {
            revert();
        }
        vault = _vault;
    }

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/VaultToken.sol#L35






### <a href="#Summary">[NC&#x2011;11]</a><a name="NC&#x2011;11"> Contracts should have full test coverage

The test coverage rate of the project is 85%. Testing all functions is best practice in terms of security criteria.
While 100% code coverage does not guarantee that there are no bugs, it often will catch easy-to-find bugs, and will ensure that there are fewer regressions when the code invariably has to be modified. Furthermore, in order to get full coverage, code authors will often have to re-organize their code so that it is more modular, so that each component can be tested separately, which reduces interdependencies between modules and layers, and makes for code that is easier to reason about and audit.

It is stated in the contest information:
```
- What is the overall line coverage percentage provided by your tests?: 85
```
Due to its capacity, test coverage is expected to be 100%.




### <a href="#Summary">[NC&#x2011;12]</a><a name="NC&#x2011;12"> Missing event for critical parameter change

When changing state variables events are not emitted. Emitting events allows monitoring activities with off-chain monitoring tools.

#### <ins>Proof Of Concept</ins>


```solidity
650: function setFeeReceipient(address _feeReceipient) external requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L650

```solidity
35: function setVault(address _vault) external {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/VaultToken.sol#L35





### <a href="#Summary">[NC&#x2011;13]</a><a name="NC&#x2011;13"> Implementation contract may not be initialized

OpenZeppelin recommends that the initializer modifier be applied to constructors. 
Per OZs Post implementation contract should be initialized to avoid potential griefs or exploits.
https://forum.openzeppelin.com/t/uupsupgradeable-vulnerability-post-mortem/15680/5

#### <ins>Proof Of Concept</ins>


```solidity
67: constructor(uint256 _priceNormalizationFactor, ISystemManager _systemManager)
        Auth(msg.sender, Authority(address(0x0)))
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L67

```solidity
156: constructor(
        ERC20 _susd,
        IVaultToken _vaultToken,
        IExchange _exchange,
        ILiquidityPool _pool,
        IPerpsV2Market _perpMarket,
        bytes32 _underlyingKey,
        bytes32 _name
    ) Auth(msg.sender, Authority(address(0x0)))
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L156

```solidity
153: constructor(ERC20 _susd, bytes32 _baseAsset, bytes32 _trackingCode, ISystemManager _systemManager)
        Auth(msg.sender, Authority(address(0x0)))
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L153

```solidity
12: constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)
        ERC20(_name, _symbol, 18)
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityToken.sol#L12

```solidity
15: constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)
        ERC20(_name, _symbol, 18)
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/PowerPerp.sol#L15

```solidity
51: constructor(uint256 susdRatio, uint256 susdLiqRatio, uint256 susdLiqBonus, ISystemManager _systemManager)
        Auth(msg.sender, Authority(address(0x0)))
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L51

```solidity
26: constructor(bytes32 _marketKey, string memory _name, string memory _symbol, ISystemManager _systemManager)
        ERC721(_name, _symbol)
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L26

```solidity
13: constructor(ISynthetix _synthetix, IExchangeRates _exchangeRates)
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SynthetixAdapter.sol#L13

```solidity
47: constructor(
        address _addressResolver,
        address _futureMarketManager,
        address _susd,
        bytes32 _baseAsset,
        bytes32 _perpMarketName
    ) Auth(msg.sender, Authority(address(0x0)))
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol#L47

```solidity
25: constructor(string memory name, string memory symbol) ERC20(name, symbol, 18)
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/VaultToken.sol#L25





### <a href="#Summary">[NC&#x2011;14]</a><a name="NC&#x2011;14"> NatSpec comments should be increased in contracts

It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation. In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability. https://docs.soliditylang.org/en/v0.8.15/natspec-format.html

#### <ins>Proof Of Concept</ins>


Various in-scope contracts

See `SystemManager.sol` for example

#### <ins>Recommended Mitigation Steps</ins>

NatSpec comments should be increased in contracts



### <a href="#Summary">[NC&#x2011;15]</a><a name="NC&#x2011;15"> Use a more recent version of Solidity

<a href="https://blog.soliditylang.org/2022/02/16/solidity-0.8.12-release-announcement/">0.8.12</a>: 
string.concat() instead of abi.encodePacked(<str>,<str>)

<a href="https://blog.soliditylang.org/2022/03/16/solidity-0.8.13-release-announcement/">0.8.13</a>: 
Ability to use using for with a list of free functions

<a href="https://blog.soliditylang.org/2022/05/18/solidity-0.8.14-release-announcement/">0.8.14</a>:

ABI Encoder: When ABI-encoding values from calldata that contain nested arrays, correctly validate the nested array length against calldatasize() in all cases.
Override Checker: Allow changing data location for parameters only when overriding external functions.

<a href="https://blog.soliditylang.org/2022/06/15/solidity-0.8.15-release-announcement/">0.8.15</a>:

Code Generation: Avoid writing dirty bytes to storage when copying bytes arrays.
Yul Optimizer: Keep all memory side-effects of inline assembly blocks.

<a href="https://blog.soliditylang.org/2022/08/08/solidity-0.8.16-release-announcement/">0.8.16</a>:

Code Generation: Fix data corruption that affected ABI-encoding of calldata values represented by tuples: structs at any nesting level; argument lists of external functions, events and errors; return value lists of external functions. The 32 leading bytes of the first dynamically-encoded value in the tuple would get zeroed when the last component contained a statically-encoded array.

<a href="https://blog.soliditylang.org/2022/09/08/solidity-0.8.17-release-announcement/">0.8.17</a>:

Yul Optimizer: Prevent the incorrect removal of storage writes before calls to Yul functions that conditionally terminate the external EVM call.

<a href="https://blog.soliditylang.org/2023/02/22/solidity-0.8.19-release-announcement/">0.8.19</a>:
SMTChecker: New trusted mode that assumes that any compile-time available code is the actual used code, even in external calls. 
Bug Fixes: 
- Assembler: Avoid duplicating subassembly bytecode where possible.
- Code Generator: Avoid including references to the deployed label of referenced functions if they are called right away.
- ContractLevelChecker: Properly distinguish the case of missing base constructor arguments from having an unimplemented base function.
- SMTChecker: Fix internal error caused by unhandled z3 expressions that come from the solver when bitwise operators are used.
- SMTChecker: Fix internal error when using the custom NatSpec annotation to abstract free functions.
- TypeChecker: Also allow external library functions in using for.

#### <ins>Proof Of Concept</ins>


```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityPool.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/LiquidityToken.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/PowerPerp.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortToken.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SynthetixAdapter.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol#L2

```solidity
pragma solidity 0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/VaultToken.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/libraries/SignedMath.sol#L2

```solidity
pragma solidity ^0.8.9;
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/utils/PauseModifier.sol#L3



#### <ins>Recommended Mitigation Steps</ins>

Consider updating to a more recent solidity version.





### <a href="#Summary">[NC&#x2011;16]</a><a name="NC&#x2011;16"> Public Functions Not Called By The Contract Should Be Declared External Instead

Contracts are allowed to override their parents’ functions and change the visibility from external to public.

#### <ins>Proof Of Concept</ins>


```solidity
function orderFee(int256 sizeDelta) public view returns (uint256 fees) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/Exchange.sol#L206


```solidity
function getMinCollateral(uint256 shortAmount, address collateral)
        public
        view
        override
        returns (uint256 collateralAmt)
    {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L153

```solidity
function canLiquidate(uint256 positionId) public view override returns (bool) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L192

```solidity
function maxLiquidatableDebt(uint256 positionId) public view override returns (uint256 maxDebt) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/ShortCollateral.sol#L215

```solidity
function getSynth(bytes32 key) public view override returns (address synth) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SynthetixAdapter.sol#L18

```solidity
function getCurrencyKey(address synth) public view override returns (bytes32 key) {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SynthetixAdapter.sol#L22

```solidity
function setStatusFunction(bytes32 key, bool status) public requiresAuth {
```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/SystemManager.sol#L89


### <a href="#Summary">[NC&#x2011;17]</a><a name="NC&#x2011;17"> Remove `forge-std` import

`forge-std` is used for logging and debugging purposes and should be removed when not used for development.

#### <ins>Proof Of Concept</ins>


```solidity
4: import {console2} from "forge-std/console2.sol";

```

https://github.com/code-423n4/2023-03-polynomial/tree/main/src/KangarooVault.sol#L4






