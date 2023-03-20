| Total Low issues |
|------------------|

| Risk   | Issues Details                                                                                              | Number        |
|--------|-------------------------------------------------------------------------------------------------------------|---------------|
| [L-01] | Critical changes should use-two step procedure                                                              | 2             |
| [L-02] | Use `safeMint` instead of mint for ERC721                                                                   | 1             |
| [L-03] | The first timestamp should be based on when funding rate was last updated                                   | 1             |
| [L-04] | Integer overflow by unsafe casting                                                                          | 1             |
| [L-05] | Missing checks for `address(0)`                                                                             | 1             |
| [L-06] | Use `require()` instead of `assert()`                                                                       | 2             |
| [L-07] | Lack of event emit                                                                                          | 3             |
| [L-08] | Fees are not capped                                                                                         | 3             |
| [L-09] | The `nonReentrant` modifier should occur before all other modifiers                                         | 21            |
| [L-10] | Loss of precision due to rounding                                                                           | 2             |
| [L-11] | Value is not validated to be different than the existing one                                                | 20            |
| [L-12] | ShortToken implmentation is not fully up to EIP-721's specification	                                        | 1             |
| [L-13] | Avoid shadowing inherited state variables                          	                                        | 2             |
| [L-14] | Lack of access control in `setVault()` function leave it vulnerable to frontrunning attack                  | 1             |

| Total Non-Critical issues |
|---------------------------|

| Risk    | Issues Details                                                                                            | Number        |
|---------|-----------------------------------------------------------------------------------------------------------|---------------|
| [NC-01] | Include return parameters in NatSpec comments                                                             | All Contracts |
| [NC-02] | Function writing does not comply with the `Solidity Style Guide`                                          | All Contracts |
| [NC-03] | Mark visibility of `init()` functions as external                                                         | 1             |
| [NC-04] | Reusable require statements should be changed to a modifier                                               | 3             |
| [NC-05] | The protocol should include NatSpec                                                                       | All Contracts |
| [NC-06] | Constants in comparisons should appear on the left side                                                   | 37            |
| [NC-07] | Use a more recent version of solidity                                                                     | All Contracts |
| [NC-08] | Contracts should have full test coverage                                                                  | All Contracts |
| [NC-09] | Add a timelock to critical functions                                                                      | 12            |
| [NC-10] | Need Fuzzing test                                                                                         | All Contracts |
| [NC-11] | Lock pragmas to specific compiler version                                                                 | All Contracts |
| [NC-12] | Consider using `delete` rather than assigning zero to clear values                                        | 11            |
| [NC-13] | For functions, follow Solidity standard naming conventions                                                | 4             |
| [NC-14] | Events that mark critical parameter changes should contain both the old and the new value                 | 12            |
| [NC-15] | Add NatSpec Mapping comment                                                                               | 8             |
| [NC-16] | Use SMTChecker                                                                                            |               |
 
## [L-01] Critical changes should use-two step procedure

#### Description

The following contracts ([KangarooVault](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol), [LiquidityPool](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol)) have a function that allows them to change the `feeReceipient` to a different address. If the sender accidentally uses an invalid address for which they do not have the private key, then the protocol will lose fees.

#### Lines of code 

```solidity
    function setFeeReceipient(address _feeReceipient) external requiresAuth {
        feeReceipient = _feeReceipient;
    }
```

- [LiquidityPool.sol#L650-L652](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L650-L652)

```solidity
    function setFeeReceipient(address _feeReceipient) external requiresAuth {
        require(_feeReceipient != address(0x0));

        emit UpdateFeeReceipient(feeReceipient, _feeReceipient);

        feeReceipient = _feeReceipient;
    }
```

- [KangarooVault.sol#L465-L471](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L465-L471)

#### Recommended Mitigation Steps

Consider adding two step procedure on the critical functions where the first is announcing a pending fee receipient and the new address should then claim its ownership.

## [L-02] Use `safeMint` instead of mint for ERC721

#### Description

Users could lost their NFTs if `msg.sender` is a contract address that does not support `ERC721`, the NFT can be frozen in the contract forever.

As per the documentation of EIP-721:

> A wallet/broker/auction application MUST implement the wallet interface if it will accept safe transfers.

> Ref: https://eips.ethereum.org/EIPS/eip-721

As per the documentation of ERC721.sol by Openzeppelin:

> Ref: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L274-L285

#### Lines of code 

```solidity
            _mint(trader, positionId);
```

- [ShortToken.sol#L67](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L67)

#### Recommended Mitigation Steps

Use [`_safeMint`](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC721.sol#L193-L202) instead of [`mint`](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC721.sol#L157-L170) to check received address support for ERC721 implementation.

## [L-03] The first timestamp should be based on when funding rate was last updated

#### Description

The first timestamp should be based on when funding rate was last updated, so that an appropriate duration of the cycle occurs, rather than during deployment.

#### Lines of code 

```solidity
    uint256 public fundingLastUpdated = block.timestamp;
```

- [Exchange.sol#L65](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L65)

## [L-04] Integer overflow by unsafe casting

#### Description

Keep in mind that the version of solidity used, despite being greater than 0.8, does not prevent integer overflows during casting, it only does so in mathematical operations.

It is necessary to safely convert between the different numeric types.

#### Lines of code 

```solidity
        uint256 newNormalizationFactor = normalizationFactor.mulWadDown(uint256(normalizationUpdate));
```

- [Exchange.sol#L198](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L198)

```solidity
        normalizationFactor = normalizationFactor.mulWadDown(uint256(normalizationUpdate));
```

- [Exchange.sol#L419](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L419)

```solidity
            uint256 marginWithdrawing = uint256(-marginDelta);
```

- [KangarooVault.sol#L408](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L408)

```solidity
            uint256 marginAdding = uint256(marginDelta);
```

- [KangarooVault.sol#L414](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L414)

```solidity
        positionData.lastSizeDelta = uint256(int256(position.size));
```

- [KangarooVault.sol#L602](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L602)

```solidity
        uint256 currentSize = uint256(int256(position.size));
```

- [KangarooVault.sol#L620](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L620)

```solidity
        positionData.lastSizeDelta = uint256(int256(position.size));
```

- [KangarooVault.sol#L710](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L710)

```solidity
        uint256 currentSize = uint256(int256(position.size));
```

- [KangarooVault.sol#L737](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L737)

```solidity
            uint256 availableFunds = uint256(int256(totalFunds) - usedFunds);
```

- [LiquidityPool.sol#L295](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L295)

```solidity
        totalValue -= uint256((int256(amountOwed) + usedFunds));
```

- [LiquidityPool.sol#L362](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L362)

```solidity
        require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
```

- [LiquidityPool.sol#L517](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L517)

```solidity
        require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
```

- [LiquidityPool.sol#L616](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L616)

```solidity
        require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
```

- [LiquidityPool.sol#L809](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L809)

```solidity
        return uint256(signedAbs(x));
```

- [SignedMath.sol#L10](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/libraries/SignedMath.sol#L10)

#### Recommended Mitigation Steps

Use OpenZeppelin [safeCast](https://docs.openzeppelin.com/contracts/3.x/api/utils#SafeCast) library.

## [L-05] Missing checks for `address(0)`

#### Description

Check of `address(0)` to protect the code from `(0x0000000000000000000000000000000000000000)` address problem just in case. This is best practice or instead of suggesting that they verify `_address != address(0)`, you could add some good NatSpec comments explaining what is valid and what is invalid and what are the implications of accidentally using an invalid address.

#### Lines of code 

```solidity
    function setFeeReceipient(address _feeReceipient) external requiresAuth {
```

- [KangarooVault.sol#L465](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L465)

#### Recommended Mitigation Steps

Add checks for `address(0)` when assigning values to address state variables.

## [L-06] Use `require()` instead of `assert()`

#### Description

Assert should not be used except for tests, require should be used

Prior to Solidity 0.8.0, pressing a confirm consumes the remainder of the process's available gas instead of returning it, as `request()/revert()` did.

The big difference between the two is that the `assert()` function when false, uses up all the remaining gas and reverts all the changes made.

Meanwhile, a `require()` statement when false, also reverts back all the changes made to the contract but does refund all the remaining gas fees we offered to pay. This is the most common Solidity function used by developers for debugging and error handling.

Assertion should be avoided even after solidity version 0.8.0, because its documentation states "The Assert function generates an error of type `Panic(uint256)`. Code that works properly should never Panic, even on invalid external input. If this happens, you need to fix it in your contract. there's a mistake".

#### Lines of code 

```solidity
        assert(queuedDepositHead + count - 1 < nextQueuedDepositId);
```

- [LiquidityPool.sol#L220](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L220)

```solidity
        assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);
```

- [LiquidityPool.sol#L285](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L285)

#### Recommended Mitigation Steps

Use `require()` instead of `assert()`

## [L-07] Lack of event emit

#### Description

The below methods do not emit an event when the state changes, something that it's very important for dApps and users.

#### Lines of code 

```solidity
    function setFeeReceipient(address _feeReceipient) external requiresAuth {
        feeReceipient = _feeReceipient;
    }
```

- [LiquidityPool.sol#L650-L652](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L650-L652)

```solidity
    function setVault(address _vault) external {
        if (vault != address(0x0)) {
            revert();
        }
        vault = _vault;
    }
```

- [VaultToken.sol#L35-L40](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L35-L40)

```solidity
    receive() external payable {
        (bool success,) = feeReceipient.call{value: msg.value}("");
        require(success);
    }
```

- [KangarooVault.sol#L454-L457](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L454-L457)

#### Recommended Mitigation Steps

Emit event.

## [L-08] Fees are not capped

#### Description

Fees are not capped, which makes the protocol less decentralized. Thus, increasing the likelihood of losing trust from users.

#### Lines of code 

```solidity
    function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
        emit UpdateFees(depositFee, _depositFee, withdrawalFee, _withdrawalFee);
        depositFee = _depositFee;
        withdrawalFee = _withdrawalFee;
    }
```

- [LiquidityPool.sol#L657-L661](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L657-L661)

```solidity
    function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
        emit UpdateBaseTradingFee(baseTradingFee, _baseTradingFee);
        baseTradingFee = _baseTradingFee;
    }
```

- [LiquidityPool.sol#L665-L668](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L665-L668)

```solidity
    function setDevFee(uint256 _devFee) external requiresAuth {
        emit UpdateDevFee(devFee, _devFee);
        devFee = _devFee;
    }
```

- [LiquidityPool.sol#L672-L675](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L672-L675)

## [L-09] The `nonReentrant` modifier should occur before all other modifiers

#### Description

This is a best-practice to protect against reentrancy in other modifiers.

#### Lines of code 

```solidity
    function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {
```

- [KangarooVault.sol#L376](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L376)

```solidity
    function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {
```

- [KangarooVault.sol#L383](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L383)

```solidity
    function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {
```

- [KangarooVault.sol#L389](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L389)

```solidity
    function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {
```

- [KangarooVault.sol#L395](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L395)

```solidity
    function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
```

- [KangarooVault.sol#L401](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L401)

```solidity
    function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {
```

- [KangarooVault.sol#L424](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L424)

```solidity
    function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
```

- [KangarooVault.sol#L436](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L436)

```solidity
    function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```

- [KangarooVault.sol#L450](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L450)

```solidity
    function openLong(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCost)
    {
```

- [LiquidityPool.sol#L430-L436](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L430-L436)

```solidity
    function closeLong(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCost)
    {
```

- [LiquidityPool.sol#L462-L468](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L462-L468)

```solidity
    function openShort(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCost)
    {
```

- [LiquidityPool.sol#L494-L500](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L494-L500)

```solidity
    function closeShort(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCost)
    {
```

- [LiquidityPool.sol#L526-L532](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L526-L532)

```solidity
    function liquidate(uint256 amount) external override onlyExchange nonReentrant {
```

- [LiquidityPool.sol#L557](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L557)

```solidity
    function hedgePositions() external override requiresAuth nonReentrant {
```

- [LiquidityPool.sol#L568](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L568)

```solidity
    function rebalanceMargin(int256 marginDelta) external requiresAuth nonReentrant {
```

- [LiquidityPool.sol#L591](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L591)

```solidity
    function increaseMargin(uint256 additionalMargin) external requiresAuth nonReentrant {
```

- [LiquidityPool.sol#L613](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L613)

```solidity
    function placeQueuedOrder() external requiresAuth nonReentrant {
```

- [LiquidityPool.sol#L692](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L692)

```solidity
    function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {
```

- [LiquidityPool.sol#L704](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L704)

```solidity
    function collectCollateral(address collateral, uint256 positionId, uint256 amount)
        external
        onlyExchange
        nonReentrant
    {
```

- [ShortCollateral.sol#L85-L89](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L85-L89)

```solidity
    function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {
```

- [ShortCollateral.sol#L106](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L106)

```solidity
    function liquidate(uint256 positionId, uint256 debt, address user)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCollateralReturned)
    {
```

- [ShortCollateral.sol#L121-L127](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L121-L127)

#### Recommended Mitigation Steps

Use the `nonReentrant` modifier first.

## [L-10] Loss of precision due to rounding

#### Description

Loss of precision due to the nature of arithmetics and rounding errors.

#### Lines of code 

```solidity
        uint256 region = size / standardSize;
```

- [LiquidityPool.sol#L371](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L371)

```solidity
        maxDebt = position.shortAmount / 2;
```

- [ShortCollateral.sol#L237](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L237)

## [L-11] Value is not validated to be different than the existing one

#### Description

Value is not validated to be different than the existing one. Queueing the same value will cause multiple abnormal events to be emitted, will ultimately result in a no-op situation.

#### Lines of code 

```solidity
    function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {
```

- [Exchange.sol#L216](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L216)

```solidity
    function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
```

- [Exchange.sol#L223](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L223)

```solidity
    function setFeeReceipient(address _feeReceipient) external requiresAuth {

```

- [LiquidityPool.sol#L650](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L650)

```solidity
    function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
```

- [LiquidityPool.sol#L657](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L657)

```solidity
    function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
```

- [LiquidityPool.sol#L665](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L665)

```solidity
    function setDevFee(uint256 _devFee) external requiresAuth {
```

- [LiquidityPool.sol#L672](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L672)

```solidity
    function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
```

- [LiquidityPool.sol#L679](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L679)

```solidity
    function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
```

- [LiquidityPool.sol#L685](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L685)

```solidity
    function setStatusFunction(bytes32 key, bool status) public requiresAuth {
```

- [SystemManager.sol#L89](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L89)

```solidity
    function setFeeReceipient(address _feeReceipient) external requiresAuth {
```

- [KangarooVault.sol#L465](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L465)

```solidity
    function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
```

- [KangarooVault.sol#L476](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L476)

```solidity
    function setSynthetixTracking(bytes32 _code) external requiresAuth {
```

- [KangarooVault.sol#L487](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L487)

```solidity
    function setReferralCode(bytes32 _code) external requiresAuth {
```

- [KangarooVault.sol#L492](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L492)

```solidity
    function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
```

- [KangarooVault.sol#L499](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L499)

```solidity
    function setMaxDepositAmount(uint256 _maxAmt) external requiresAuth {
```

- [KangarooVault.sol#L506](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L506)

```solidity
    function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
```

- [KangarooVault.sol#L514](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L514)

```solidity
    function setPriceImpactDelta(uint256 _delta) external requiresAuth {
```

- [KangarooVault.sol#L522](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L522)

```solidity
    function setLeverage(uint256 _lev) external requiresAuth {
```

- [KangarooVault.sol#L529](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L529)

```solidity
    function setCollRatio(uint256 _ratio) external requiresAuth {
```

- [KangarooVault.sol#L537](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L537)

#### Recommended Mitigation Steps

Add a `require()` statement to check that the new value is different than the current one.

## [L-12] ShortToken implmentation is not fully up to EIP-721's specification

#### Description

`tokenURI()` return an empty string which could cause unexpected behavior in the future due to non-compliance with EIP-721 standard.

#### Lines of code 

```solidity
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        return "";
    }
```

- [ShortToken.sol#L44-L46](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L44-L46)

#### Recommended Mitigation Steps

`tokenURI()` should return something and throws if `tokenId` is not a valid NFT.

## [L-13] Avoid shadowing inherited state variables

#### Description

In `VaultToken.sol` there is a local variables named `name` `symbol`, but there is state variables in the inherited contract ( `ERC20.sol`) with the same name. This use causes compilers to issue warnings, negatively affecting checking and code readability.

#### Lines of code

```solidity
    constructor(string memory name, string memory symbol) ERC20(name, symbol, 18) {}
```

- [VaultToken.sol#L25](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L25)

#### Recommended Mitigation Steps

Avoid using variables with the same name.

## [L-14] Lack of access control in `setVault()` function leave it vulnerable to frontrunning attack

#### Description

[`setVault()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L35-L40) function has no access control and can be called by anyone which leave it vulnerable to frontrunning attack, and since this function can only be set once, thus it may force a redeployment.

#### Lines of code 

```solidity
    function setVault(address _vault) external {
        if (vault != address(0x0)) {
            revert();
        }
        vault = _vault;
    }
```

- [VaultToken.sol#L35-L40](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L35-L40)

#### Recommended Mitigation Steps

Add access control to [`setVault()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L35-L40) to protect the function and make more robust.

## [NC-01] Include return parameters in NatSpec comments

#### Description

If Return parameters are declared, you must prefix them with `/// @return`.
Some code analysis programs do analysis by reading [NatSpec](https://docs.soliditylang.org/en/v0.8.15/natspec-format.html) details, if they can't see the `@return` tag, they do incomplete analysis.

#### Lines of code 

- [All Contracts](https://github.com/code-423n4/2023-03-polynomial/tree/main/src)

#### Recommended Mitigation Steps

Include `@return` parameters in NatSpec comments

## [NC-02] Function writing does not comply with the `Solidity Style Guide`

#### Description

Ordering helps readers identify which functions they can call and to find the constructor and fallback definitions easier. But there are contracts in the project that do not comply with this.

Functions should be grouped according to their visibility and ordered:

- `constructor()`
- `receive()`  
- `fallback()`  
- `external / public / internal / private`
- `view / pure`

#### Lines of code 

- [All Contracts](https://github.com/code-423n4/2023-03-polynomial/tree/main/src)

#### Recommended Mitigation Steps

Follow Solidity Style Guide.

## [NC-03] Mark visibility of `init()` functions as external

#### Description

- If someone wants to extend via inheritance, it might make more sense that the overridden `init()` function calls the internal {...}_init function, not the parent public `init()` function.

- External instead of public would give more the sense of the `init()` functions to behave like a constructor (only called on deployment, so should only be called externally)

- Security point of view, it might be safer so that it cannot be called internally by accident in the child contract

- It might cost a bit less gas to use external over public

- It is possible to override a function from external to public ("opening it up") but it is not possible to override a function from public to external ("narrow it down").

> Ref: https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3750

#### Lines of code 

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

- [SystemManager.sol#L62-L70](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L62-L70)

#### Recommended Mitigation Steps

Change the visibility of `init()` functions to external

## [NC-04] Reusable require statements should be changed to a modifier

#### Description

Reusable require statements should be changed to a modifier.

#### Lines of code 

```solidity
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
```

- [ShortToken.sol#L93](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L93)

```solidity
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
```

- [ShortToken.sol#L98](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L98)

```solidity
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
```

- [ShortToken.sol#L103](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L103)

## [NC-05] The protocol should include NatSpec

#### Description

It is recommended that Solidity contracts are fully annotated using NatSpec, it is clearly stated in the Solidity official documentation.

- In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability. 

- Some code analysis programs do analysis by reading NatSpec details, if they can't see the tags `(@param, @dev, @return)`, they do incomplete analysis.

#### Lines of code 

- [All Contracts](https://github.com/code-423n4/2023-03-polynomial/tree/main/src)

#### Recommended Mitigation Steps

Include [`NatSpec`](https://docs.soliditylang.org/en/v0.8.15/natspec-format.html) comments in the codebase.

## [NC-06] Constants in comparisons should appear on the left side

#### Description

Constants in comparisons should appear on the left side, doing so will prevent typo [bug](https://www.moserware.com/2008/01/constants-on-left-are-better-but-this.html).

```solidity
            require(holdings == 0, "Shouldn't have long positions to close short postions");
```

#### Lines of code 

```solidity
            require(shortPositions == 0, "Short position must be closed before opening");
```

- [Exchange.sol#L236](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L236)

```solidity
            require(holdings == 0, "Long position must be closed before opening");
```

- [Exchange.sol#L248](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L248)

```solidity
            require(shortPositions == 0, "Shouldn't have short positions to close long postions");
```

- [Exchange.sol#L291](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L291)

```solidity
            require(holdings == 0, "Shouldn't have long positions to close short postions");
```

- [Exchange.sol#L302](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L302)

```solidity
        if (positionData.positionId == 0) {
```

- [KangarooVault.sol#L188](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L188)

```solidity
        if (positionData.positionId == 0) {
```

- [KangarooVault.sol#L219](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L219)

```solidity
            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
```

- [KangarooVault.sol#L249](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L249)

```solidity
            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minWithdrawDelay) {
```

- [KangarooVault.sol#L275](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L275)

```solidity
            if (availableFunds == 0) {
```

- [KangarooVault.sol#L281](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L281)

```solidity
        if (totalFunds == 0) {
```

- [KangarooVault.sol#L342](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L342)

```solidity
        if (positionData.positionId == 0) {
```

- [KangarooVault.sol#L347](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L347)

```solidity
        require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);
```

- [KangarooVault.sol#L557](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L557)

```solidity
        require(delayedOrder.sizeDelta == 0);
```

- [KangarooVault.sol#L560](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L560)

```solidity
        if (positionData.positionId == 0) {
```

- [KangarooVault.sol#L590](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L590)

```solidity
        require(positionData.pendingLongPerp > 0 && positionData.pendingShortPerp == 0);
```

- [KangarooVault.sol#L614](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L614)

```solidity
        require(delayedOrder.sizeDelta == 0);
```

- [KangarooVault.sol#L617](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L617)

```solidity
            if (positionData.shortAmount == 0) {
```

- [KangarooVault.sol#L658](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L658)

```solidity
        require(positionData.positionId != 0 && positionData.pendingLongPerp == 0 && positionData.pendingShortPerp == 0);
```

- [KangarooVault.sol#L671](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L671)

```solidity
        require(delayedOrder.sizeDelta == 0);
```

- [KangarooVault.sol#L674](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L674)

```solidity
        require(positionData.pendingLongPerp == 0 && positionData.pendingShortPerp > 0);
```

- [KangarooVault.sol#L731](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L731)

```solidity
        require(delayedOrder.sizeDelta == 0);
```

- [KangarooVault.sol#L734](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L734)

```solidity
            if (positionData.shortAmount == 0) {
```

- [KangarooVault.sol#L774](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L774)

```solidity
            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
```

- [LiquidityPool.sol#L226](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L226)

```solidity
            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minWithdrawDelay) {
```

- [LiquidityPool.sol#L291](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L291)

```solidity
            if (availableFunds == 0) {
```

- [LiquidityPool.sol#L297](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L297)

```solidity
        if (totalFunds == 0) {
```

- [LiquidityPool.sol#L341](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L341)

```solidity
        if (skew == 0) {
```

- [LiquidityPool.sol#L348](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L348)

```solidity
        require(order.sizeDelta == 0);
```

- [LiquidityPool.sol#L695](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L695)

```solidity
        require(shortToken.balanceOf(_to) == 0, "Receiver has short positions");
```

- [PowerPerp.sol#L41](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L41)

```solidity
        require(shortToken.balanceOf(_to) == 0, "Receiver has short positions");
```

- [PowerPerp.sol#L46](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L46)

```solidity
        if (positionId == 0) {
```

- [ShortToken.sol#L55](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L55)

```solidity
            if (position.shortAmount == 0) {
```

- [ShortToken.sol#L82](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L82)

```solidity
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
```

- [ShortToken.sol#L93](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L93)

```solidity
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
```

- [ShortToken.sol#L98](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L98)

```solidity
        require(powerPerp.balanceOf(_to) == 0, "Receiver has long positions");
```

- [ShortToken.sol#L103](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L103)

#### Recommended Mitigation Steps

Constants should appear on the left side:

```solidity
            require(0 == holdings, "Shouldn't have long positions to close short postions");
```

## [NC-07] Use a more recent version of solidity

#### Description

For security, it is best practice to use the [latest Solidity version](https://github.com/ethereum/solidity/blob/develop/Changelog.md).

#### Lines of code 

- [All Contracts](https://github.com/code-423n4/2023-03-polynomial/tree/main/src)

#### Recommended Mitigation Steps

Old version of Solidity is used `(^0.8.9)`, newer version can be used `(0.8.19)`.

## [NC-08] Contracts should have full test coverage

#### Description

While 100% code coverage does not guarantee that there are no bugs, it often will catch easy-to-find bugs, and will ensure that there are fewer regressions when the code invariably has to be modified. Furthermore, in order to get full coverage, code authors will often have to re-organize their code so that it is more modular, so that each component can be tested separately, which reduces interdependencies between modules and layers, and makes for code that is easier to reason about and audit.

#### Lines of code 

```
- What is the overall line coverage percentage provided by your tests?: 85
```

- [All Contracts](https://github.com/code-423n4/2023-03-polynomial/tree/main/src)

#### Recommended Mitigation Steps

Line coverage percentage should be 100%.

## [NC-09] Add a timelock to critical functions

#### Description

It is a good practice to give time for users to react and adjust to critical changes. A timelock provides more guarantees and reduces the level of trust required, thus decreasing risk for users. It also indicates that the project is legitimate.

#### Lines of code 

```solidity
    function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
```

- [Exchange.sol#L223](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L223)

```solidity
    function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
```

- [LiquidityPool.sol#L657](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L657)

```solidity
    function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
```

- [LiquidityPool.sol#L665](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L665)

```solidity
    function setDevFee(uint256 _devFee) external requiresAuth {
```

- [LiquidityPool.sol#L672](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L672)

```solidity
    function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
```

- [LiquidityPool.sol#L679](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L679)

```solidity
    function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
```

- [LiquidityPool.sol#L685](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L685)

```solidity
    function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
```

- [KangarooVault.sol#L476](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L476)

```solidity
    function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
```

- [KangarooVault.sol#L499](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L499)

```solidity
    function setMaxDepositAmount(uint256 _maxAmt) external requiresAuth {
```

- [KangarooVault.sol#L506](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L506)

```solidity
    function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
```

- [KangarooVault.sol#L514](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L514)

```solidity
    function setPriceImpactDelta(uint256 _delta) external requiresAuth {
```

- [KangarooVault.sol#L522](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L522)

```solidity
    function setCollRatio(uint256 _ratio) external requiresAuth {
```

- [KangarooVault.sol#L537](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L537)

#### Recommended Mitigation Steps

Consider adding a timelock to the critical changes.

## [NC-10] Need Fuzzing test

#### Description

As Alberto Cuesta Canada said: Fuzzing is not easy, the tools are rough, and the math is hard, but it is worth it. Fuzzing gives me a level of confidence in my smart contracts that I didn’t have before. Relying just on unit testing anymore and poking around in a testnet seems reckless now.

Ref: https://medium.com/coinmonks/smart-contract-fuzzing-d9b88e0b0a05

#### Lines of code 

- [All Contracts](https://github.com/code-423n4/2023-03-polynomial/tree/main/src)

#### Recommended Mitigation Steps

Use should fuzzing test like Echidna.

## [NC-11] Lock pragmas to specific compiler version

#### Description

Pragma statements can be allowed to float when a contract is intended for consumption by other developers, as in the case with contracts in a library or EthPM package. Otherwise, the developer would need to manually update the pragma in order to compile locally. 

> Ref: https://swcregistry.io/docs/SWC-103

#### Lines of code 

- [All Contracts](https://github.com/code-423n4/2023-03-polynomial/tree/main/src)

#### Recommended Mitigation Steps

[Ethereum Smart Contract Best Practices](https://consensys.github.io/smart-contract-best-practices/development-recommendations/solidity-specific/locking-pragmas/): Lock pragmas to specific compiler version. 

## [NC-12] Consider using `delete` rather than assigning zero to clear values    

#### Description

The `delete` keyword more closely matches the semantics of what is being done, and draws more attention to the changing of state, which may lead to a more thorough audit of its associated logic.

#### Lines of code

```solidity
            current.depositedAmount = 0;
```

- [LiquidityPool.sol#L239](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L239)

```solidity
                current.withdrawnTokens = 0;
```

- [LiquidityPool.sol#L329](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L329)

```solidity
            current.depositedAmount = 0;
```

- [KangarooVault.sol#L262](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L262)

```solidity
                current.withdrawnTokens = 0;
```

- [KangarooVault.sol#L328](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L328)

```solidity
                positionData.premiumCollected = 0;
```

- [KangarooVault.sol#L641](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L641)

```solidity
        positionData.pendingLongPerp = 0;
```

- [KangarooVault.sol#L667](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L667)

```solidity
            positionData.premiumCollected = 0;
```

- [KangarooVault.sol#L714](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L714)

```solidity
        positionData.pendingShortPerp = 0;
```

- [KangarooVault.sol#L779](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L779)

```solidity
        positionData.positionId = 0;
```

- [KangarooVault.sol#L783](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L783)

```solidity
        positionData.premiumCollected = 0;
```

- [KangarooVault.sol#L794](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L794)

```solidity
        positionData.totalMargin = 0;
```

- [KangarooVault.sol#L795](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L795)

#### Recommended Mitigation Steps

Use the `delete` keyword.

## [NC-13] For functions, follow Solidity standard naming conventions

#### Description

The protocol don't follow solidity standard naming convention.

> Ref: https://docs.soliditylang.org/en/v0.8.17/style-guide.html#naming-conventions

#### Lines of code 

```solidity
    function signedAbs(int256 x) internal pure returns (int256) {
```

- [SignedMath.sol#L5](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/libraries/SignedMath.sol#L5)

```solidity
    function abs(int256 x) internal pure returns (uint256) {
```

- [SignedMath.sol#L9](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/libraries/SignedMath.sol#L9)

```solidity
    function max(int256 x, int256 y) internal pure returns (int256) {
```

- [SignedMath.sol#L13](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/libraries/SignedMath.sol#L13)

```solidity
    function min(int256 x, int256 y) internal pure returns (int256) {
```

- [SignedMath.sol#L17](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/libraries/SignedMath.sol#L17)

#### Recommended Mitigation Steps

Follow solidity standard [naming convention](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#naming-conventions).

## [NC-14] Events that mark critical parameter changes should contain both the old and the new value

#### Description

Events that mark critical parameter changes should contain both the old and the new value.

#### Lines of code 

```solidity
    function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
```

- [Exchange.sol#L223](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L223)

```solidity
    function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
```

- [LiquidityPool.sol#L657](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L657)

```solidity
    function setBaseTradingFee(uint256 _baseTradingFee) external requiresAuth {
```

- [LiquidityPool.sol#L665](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L665)

```solidity
    function setDevFee(uint256 _devFee) external requiresAuth {
```

- [LiquidityPool.sol#L672](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L672)

```solidity
    function setPerpPriceImpactDelta(uint256 _perpPriceImpactDelta) external requiresAuth {
```

- [LiquidityPool.sol#L679](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L679)

```solidity
    function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
```

- [LiquidityPool.sol#L685](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L685)

```solidity
    function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
```

- [KangarooVault.sol#L476](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L476)

```solidity
    function setMinDepositAmount(uint256 _minAmt) external requiresAuth {
```

- [KangarooVault.sol#L499](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L499)

```solidity
    function setMaxDepositAmount(uint256 _maxAmt) external requiresAuth {
```

- [KangarooVault.sol#L506](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L506)

```solidity
    function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
```

- [KangarooVault.sol#L514](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L514)

```solidity
    function setPriceImpactDelta(uint256 _delta) external requiresAuth {
```

- [KangarooVault.sol#L522](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L522)

```solidity
    function setCollRatio(uint256 _ratio) external requiresAuth {
```

- [KangarooVault.sol#L537](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L537)

#### Recommended Mitigation Steps

Add the old value to the event.

## [NC-15] Add NatSpec Mapping comment  

#### Description

Add NatSpec comments describing mapping keys and values.

```solidity
    mapping(bytes32 => bool) public isPaused;
```

#### Lines of code 

```solidity
    mapping(uint256 => QueuedDeposit) public depositQueue;
```

- [KangarooVault.sol#L151](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L151)

```solidity
    mapping(uint256 => QueuedWithdraw) public withdrawalQueue;
```

- [KangarooVault.sol#L154](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L154)

```solidity
    mapping(uint256 => QueuedDeposit) public depositQueue;
```

- [LiquidityPool.sol#L144](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L144)

```solidity
    mapping(uint256 => QueuedWithdraw) public withdrawalQueue;
```

- [LiquidityPool.sol#L147](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L147)

```solidity
    mapping(bytes32 => Collateral) public collaterals;
```

- [ShortCollateral.sol#L34](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L34)

```solidity
    mapping(uint256 => UserCollateral) public userCollaterals;
```

- [ShortCollateral.sol#L37v](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L37)

```solidity
    mapping(uint256 => ShortPosition) public shortPositions;
```

- [ShortToken.sol#L24](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L24)

```solidity
    mapping(bytes32 => bool) public isPaused;
```

- [SystemManager.sol#L45](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L45)

#### Recommended Mitigation Steps

```solidity
/// @dev bytes32(Key) => bool(status)
    mapping(bytes32 => bool) public isPaused;
```

## [NC-16] Use SMTChecker

#### Description

The highest tier of smart contract behavior assurance is formal mathematical verification. All assertions that are made are guaranteed to be true across all inputs → The quality of your asserts is the quality of your verification.

> Ref: https://twitter.com/0xOwenThurm/status/1614359896350425088?t=dbG9gHFigBX85Rv29lOjIQ&s=19
