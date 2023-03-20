# QA Report

## Summary

### Low Risk Issues

- [L-1] Vault assignment in VaultToken can be frontrunned
- [L-2] Invalid and stale prices from Synthethix are not validated
- [L-3] KangarooVault.removeCollateral doesn't remove the collateral from the position
- [L-4] Admin role is overloaded with Keeper responsibilities
- [L-5] Missing minimum values for deposits and withdraw delay
- [L-6] Spamming deposit and withdraw queues
- [L-7] Missing checks for address(0) on LiquidityPool deposit and withdraw functions

### Non-Critical Issues

- [NC-1] Replace `feeReceipient` with `feeRecipient`
- [NC-2] Unused interface
- [NC-3] Contracts do not inherit from their corresponding interfaces
- [NC-4] Remove console log import
- [NC-5] Use revert instead of assert

## Low Risk Issues

### [L-1] Vault assignment in VaultToken can be frontrunned

Anyone can call the `setVault` function in `VaultToken`, and thus frontrun the assignment of its corresponding vault.

```solidity
35:    function setVault(address _vault) external {
36:        if (vault != address(0x0)) {
37:            revert();
38:        }
39:        vault = _vault;
40:    }
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L35-L40)

#### Recommended Mitigation Steps

Add a modifier so that only an authorized user can call it.

### [L-2] Invalid and stale prices from Synthethix are not validated

There are multiple methods receiving an `isInvalid` response related to asset prices from Synthethix that are not reverting when an invalid price arrives. Additionally, there are two function implementing the check on a wrong way.

This is not affecting any external facing method that modifies state because they luckily revert due to some other call to a function that implements it correctly. Otherwise it could have led to worse consequences.

Nevertheless, some public views are affected, and internal methods as well, that could lead to medium or high risk issues if changes are made, or an external integration relies on those public views.

#### Context

Invalid price rates from Syntethix contracts are defined as:

```
    // Rate can be invalid either due to:
    //  1. Returned as invalid from ExchangeRates - due to being stale, or flagged by oracle.
    //  2, Out of deviation dounds w.r.t. to previously stored rate or if there is no
    //  valid stored rate, w.r.t. to previous 3 oracle rates.
```

[Link to code](https://github.com/Synthetixio/synthetix/blob/v2.83.1/contracts/ExchangeCircuitBreaker.sol#L56-L60)

Prices equal to zero are also invalid if used via the function `assetPrice` which internally calls:

```solidity
    /*
     * The current base price from the oracle, and whether that price was invalid. Zero prices count as invalid.
     * Public because used both externally and internally
     */
    function _assetPrice() internal view returns (uint price, bool invalid) {
        (price, invalid) = _exchangeRates().rateAndInvalid(_baseAsset());
        // Ensure we catch uninitialised rates or suspended state / synth
        invalid = invalid || price == 0 || _systemStatus().synthSuspended(_baseAsset());
        return (price, invalid);
    }
```

[Link to code](https://github.com/Synthetixio/synthetix/blob/b41a5771ac782898b7d01f602514a05ff93e5b8c/contracts/PerpsV2MarketBase.sol#L593-L602)

#### Proof of concept

`_getDelta()` and `_calculateMargin` functions apply a wrong logic while checking invalid prices:

Given `!isInvalid || spotPrice > 0`, it will bypass the check for every positive `spotPrice` despite the result being `isInvalid == true`;

```solidity
// File: src/LiquidityPool.sol

414    function getDelta() external view override returns (uint256) {
415        return _getDelta();
416    }

727:     function _getDelta() internal view returns (uint256 delta) {
728:         (uint256 spotPrice, bool isInvalid) = baseAssetPrice();
729:         uint256 pricingConstant = exchange.PRICING_CONSTANT();
730:
731:         require(!isInvalid || spotPrice > 0); // @audit invalid check
732:
733:         delta = spotPrice.mulDivDown(2e18, pricingConstant);
734:         delta = delta.mulWadDown(exchange.normalizationFactor());
735:     }
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L727-L735)

```solidity
// File: src/LiquidityPool.sol

764:     function _calculateMargin(int256 size) internal view returns (uint256 margin) {
765:         (uint256 spotPrice, bool isInvalid) = baseAssetPrice();
766:
767:         require(!isInvalid || spotPrice > 0); // @audit invalid check
768:
769:         uint256 absSize = size.abs();
770:         margin = absSize.mulDivDown(spotPrice, futuresLeverage);
771:     }
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L764-L771)

In addition, multiple functions omit reverting when they receive an `isInvalid` value. In some cases, another check is made on the same function, which would prevent any error. But on other cases, there is no other check on the same function that makes it revert.

```solidity
// File: src/Exchange.sol

190:         (int256 fundingRate,) = getFundingRate();

410:         (int256 fundingRate,) = getFundingRate();
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L190)

```solidity
// File: src/KangarooVault.sol

437:        (uint256 markPrice,) = LIQUIDITY_POOL.getMarkPrice();

568:        (uint256 markPrice,) = LIQUIDITY_POOL.getMarkPrice();

784:        (uint256 totalMargin,) = PERP_MARKET.remainingMargin(address(this));
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L437)

```solidity
// File: src/LiquidityPool.sol

388:        (uint256 markPrice,) = exchange.getMarkPrice();

594:        (uint256 currentMargin,) = perpMarket.remainingMargin(address(this));

775:        (uint256 margin,) = perpMarket.remainingMargin(address(this));
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L775)

```solidity
// File: src/ShortCollateral.sol

133:        (uint256 markPrice,) = exchange.getMarkPrice();
134:        (uint256 collateralPrice,) = synthetixAdapter.getAssetPrice(currencyKey);
```

#### Recommended Mitigation Steps

Fix the wrong price check on `LiquidityPool` functions.

Price rates == 0 are invalid in Synthetix, so that validation may be omited. I'm leaving it in case their implementations change in the future.

```diff
// File: src/LiquidityPool.sol

     function _getDelta() internal view returns (uint256 delta) {
         (uint256 spotPrice, bool isInvalid) = baseAssetPrice();
         uint256 pricingConstant = exchange.PRICING_CONSTANT();

-         require(!isInvalid || spotPrice > 0);
+         require(!isInvalid && spotPrice > 0);

         delta = spotPrice.mulDivDown(2e18, pricingConstant);
         delta = delta.mulWadDown(exchange.normalizationFactor());
     }

     function _calculateMargin(int256 size) internal view returns (uint256 margin) {
         (uint256 spotPrice, bool isInvalid) = baseAssetPrice();

-         require(!isInvalid || spotPrice > 0);
+         require(!isInvalid && spotPrice > 0);

         uint256 absSize = size.abs();
         margin = absSize.mulDivDown(spotPrice, futuresLeverage);
     }
```

Double-check the missing `isInvalid` return values from the functions described in the Proof of Concept section, and add a `require` to validate them.

### [L-3] KangarooVault.removeCollateral doesn't remove the collateral from the position

The function updates the `KangarooVault` storage variables, but doesn't call the `EXCHANGE.removeCollateral()` function to actually remove the collateral from the position.

### Impact

This will lead to a permanent inconsistent state of the `KangarooVault` `usedFunds` and `positionData.totalCollateral` storage variables.

### Proof of Concept

The `removeCollateral` function is missing the call to `EXCHANGE.removeCollateral()`:

```solidity
// File: src/KangarooVault.sol

434:     /// @notice Remove collateral from Power Perp position
435:     /// @param collateralToRemove Amount of collateral to remove
436:     function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
437:         (uint256 markPrice,) = LIQUIDITY_POOL.getMarkPrice();
438:         uint256 minColl = positionData.shortAmount.mulWadDown(markPrice);
439:         minColl = minColl.mulWadDown(collRatio);
440:
441:         require(positionData.totalCollateral >= minColl + collateralToRemove);
442:
443:         usedFunds -= collateralToRemove; // @audit modified despite the collateral wasn't removed
444:         positionData.totalCollateral -= collateralToRemove; // @audit modified despite the collateral wasn't removed
445:
446:         emit RemoveCollateral(positionData.positionId, collateralToRemove);
447:     }
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L434-L447)

This is how its `addCollateral` counterpart achieves its mission:

```solidity
// File: src/KangarooVault.sol

422:     /// @notice Add additional collateral to Power Perp position
423:     /// @param additionalCollateral Amount of collateral to add
424:     function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {
425:         SUSD.safeApprove(address(EXCHANGE), additionalCollateral);
426:         EXCHANGE.addCollateral(positionData.positionId, additionalCollateral); // @audit-info adds the collateral here
427:
428:         usedFunds += additionalCollateral;
429:         positionData.totalCollateral += additionalCollateral;
430:
431:         emit AddCollateral(positionData.positionId, additionalCollateral);
432:     }
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L422-L432)

#### Recommended Mitigation Steps

Add the corresponding call to `EXCHANGE.removeCollateral()`:

```diff
// File: src/KangarooVault.sol

     /// @notice Remove collateral from Power Perp position
     /// @param collateralToRemove Amount of collateral to remove
     function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
         (uint256 markPrice,) = LIQUIDITY_POOL.getMarkPrice();
         uint256 minColl = positionData.shortAmount.mulWadDown(markPrice);
         minColl = minColl.mulWadDown(collRatio);

         require(positionData.totalCollateral >= minColl + collateralToRemove);

         usedFunds -= collateralToRemove;
         positionData.totalCollateral -= collateralToRemove;

+        EXCHANGE.removeCollateral(positionData.positionId, collateralToRemove);

         emit RemoveCollateral(positionData.positionId, collateralToRemove);
     }
```

### [L-4] Admin role is overloaded with Keeper responsibilities

Separation of responsibilities into different actors reduces the risk of exposing admin accounts by using them for maintenance or operative tasks.

There is a block of functions in the `KangarooVault` contract encapsulated under "Keeper actions" that uses the admin `requiresAuth` modifier.

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L369-L452)

#### Recommended Mitigation Steps

Create an specific Keeper role to operate on those functions.

### [L-5] Missing minimum values for deposits and withdraw delay

Assigning a zero value to `minDepositDelay` and `minWithdrawDelay` will make withdrawals and deposits collect no fee, as they can be done via the `queueDeposit()` and `queueWithdraw()` functions on the same block, making the `deposit` and `withdraw` functions obsolete.

The affected code:

```solidity
// File: src/LiquidityPool.sol

684:     /// @notice Update delays for deposit and withdraw
685:     function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
686:         emit UpdateDelays(minDepositDelay, _minDepositDelay, minWithdrawDelay, _minWithdrawDelay);
687:         minDepositDelay = _minDepositDelay;
688:         minWithdrawDelay = _minWithdrawDelay;
689:     }

// function queueDeposit(uint256 amount, address user)
210:     newDeposit.requestedTime = block.timestamp;

// function processDeposits(uint256 count)
226:     if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
227:         return;
228:     }

// function queueWithdraw(uint256 tokens, address user)
275:     newWithdraw.requestedTime = block.timestamp;

// function processWithdraws(uint256 count)
291:     if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minWithdrawDelay) {
292:          return;
293:     }
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L684-L689)

#### Recommended Mitigation Steps

Add a minimum value to `_minDepositDelay` and `_minWithdrawDelay`:

```diff
// File: src/LiquidityPool.sol

     /// @notice Update delays for deposit and withdraw
     function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
+        require(_minDepositDelay > 0, "minimum deposit delay cannot be zero");
+        require(_minWithdrawDelay > 0, "minimum withdraw delay cannot be zero");
         emit UpdateDelays(minDepositDelay, _minDepositDelay, minWithdrawDelay, _minWithdrawDelay);
         minDepositDelay = _minDepositDelay;
         minWithdrawDelay = _minWithdrawDelay;
     }
```

### [L-6] Spamming deposit and withdraw queues

The `queueDeposit()` and `queueWithdraw()` functions in the `LiquidityPool` accept an `amount == 0`, meaning that deposits and withdrawals can be queued with no extra cost other than gas, thus facilitating spamming the queues that will have to be later processed.

Same thing happens on the `initiateWithdrawal()` function on the `KangarooVault` contract.

#### Recommended Mitigation Steps

Add a check to provide a minimum value for `amount`.

`KangarooVault` is already performing this check for the `initiateDeposit()` function, so this change will also improve consistency of the codebase.

```diff
// File: src/LiquidityPool.sol

200:    function queueDeposit(uint256 amount, address user)
201:        external
202:        override
203:        nonReentrant
204:        whenNotPaused("POOL_QUEUE_DEPOSIT")
205:    {
+           require(amount > 0, "amount cannot be 0");

264:    function queueWithdraw(uint256 tokens, address user)
265:        external
266:        override
267:        nonReentrant
268:        whenNotPaused("POOL_QUEUE_WITHDRAW")
269:    {
+           require(amount > 0, "amount cannot be 0");
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L200-L280)

```diff
// File: KangarooVault:
215:    function initiateWithdrawal(address user, uint256 tokens) external nonReentrant {
216:        require(user != address(0x0));
+           require(tokens > 0, "tokens cannot be 0");
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L215-L239)

### [L-7] Missing checks for address(0) on LiquidityPool deposit and withdraw functions

Deposit and withdraw functions on the liquidity pool do not check that the user is not the `address(0)`. That can lead to loss of assets if not properly set by the user.

`KangarooVault` is already performing these checks for its counterpart deposit and withdraw functions. So, adding this change will also improve consistency of the codebase.

#### Recommended Mitigation Steps

Consider adding:

```diff
// File: src/LiquidityPool.sol

184:    function deposit(uint256 amount, address user) external override nonReentrant whenNotPaused("POOL_DEPOSIT") {
+           require(user != address(0), "cannot be address(0)");

200:    function queueDeposit(uint256 amount, address user)
201:        external
202:        override
203:        nonReentrant
204:        whenNotPaused("POOL_QUEUE_DEPOSIT")
205:    {
+           require(user != address(0), "cannot be address(0)");

247:    function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {
+           require(user != address(0), "cannot be address(0)");

264:    function queueWithdraw(uint256 tokens, address user)
265:        external
266:        override
267:        nonReentrant
268:        whenNotPaused("POOL_QUEUE_WITHDRAW")
269:    {
+           require(user != address(0), "cannot be address(0)");
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L184-L280)

## Non-Critical Issues

### [NC-1] Replace `feeReceipient` with `feeRecipient`

`feeReceipient` is defined as a public facing variable on `KangarooVault` and `LiquidityPool` contracts, as well as the `ILiquidityPool` interface. Public facing variables shouldn't have typos.

```solidity
// File: src/KangarooVault.sol
106:     address public feeReceipient;
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L106)

```solidity
// File: src/LiquidityPool.sol
102:     address public feeReceipient;
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L102)

```solidity
// File: src/ILiquidityPool.sol
51:     function feeReceipient() external view returns (address);
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ILiquidityPool.sol#L51)

#### Recommended Mitigation Steps

Fix the typo on public facing variables and interfaces. Also consider fixing it on internal methods with similar typo errors, like `setFeeReceipient` for example.

### [NC-2] Unused interface

`KangarooVault` imports and inherits `PauseModifier` but it doesn't use the `whenNotPaused` modifier.

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L19-L21)

#### Recommended Mitigation Steps

Double-check the contract shouldn't use it for its functions and remove the import.

```diff
// File: src/KangarooVault.sol

-  import {PauseModifier} from "./utils/PauseModifier.sol";

-  contract KangarooVault is Auth, ReentrancyGuard, PauseModifier {
+  contract KangarooVault is Auth, ReentrancyGuard {
```

### [NC-3] Contracts do not inherit from their corresponding interfaces

Implementing the corresponding interface for a contract makes it sure that it is called correctly outside of it, and also preventing attempts to call not implemented methods.

Some contracts do not implement their corresponding interface:

```solidity
File: src/LiquidityToken.sol

7: contract LiquidityToken is ERC20 { // @audit missing ILiquidityToken
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol#L7)

```solidity
File: src/PowerPerp.sol

8: contract PowerPerp is ERC20 { // @audit missing IPowerPerp
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L8)

```solidity
File: src/ShortToken.sol

8: contract ShortToken is ERC721 { // @audit missing IShortToken
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L8)

```solidity
File: src/VaultToken.sol

6: contract VaultToken is ERC20 { // @audit missing IVaultToken
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L6)

#### Recommended Mitigation Steps

Add the corresponding interface inheritance to the described contracts.

### [NC-4] Remove console log import

`console2` should only be used for debugging and not on the final release. Remove the import:

```diff
- import {console2} from "forge-std/console2.sol";
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L4)

### [NC-5] Use revert instead of assert

From [Solidity documentation](https://docs.soliditylang.org/en/v0.8.19/control-structures.html#panic-via-assert-and-error-via-require):

"The assert function creates an error of type Panic(uint256). The same error is created by the compiler in certain situations as listed below.

Assert should only be used to test for internal errors, and to check invariants. Properly functioning code should never create a Panic, not even on invalid external input. If this happens, then there is a bug in your contract which you should fix."

Consider replacing the `assert` with `require` statements in these places:

```solidity
File: src/LiquidityPool.sol

220: assert(queuedDepositHead + count - 1 < nextQueuedDepositId);
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L220)

```solidity
File: src/LiquidityPool.sol

285: assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);
```

[Link to code](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L285)
