# LOW FINDINGS 

##

### [L-1] Consider using OpenZeppelin’s SafeCast library to prevent unexpected overflows when casting from uint256

FILE : 2023-03-polynomial/src/Exchange.sol

        175: int256 usdSkew = wadMul(skew, int256(indexPrice));
        // Normalized skew (Skew in to funding rate)
        177: int256 normalizedSkew = wadDiv(usdSkew, int256(skewNormalizationFactor));
        178: int256 maxFunding = int256(maxFundingRate);

[Exchange.sol#L175-L178](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L175-L178)

      193: int256 currentTimeStamp = int256(block.timestamp);
      194: int256 fundingLastUpdatedTimestamp = int256(fundingLastUpdated);

[Exchange.sol#L193-L194](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L193-L194)

     413: int256 currentTimeStamp = int256(block.timestamp);
     414: int256 fundingLastUpdatedTimestamp = int256(fundingLastUpdated);

[Exchange.sol#L413-L414](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L413-L414)

Recommended Mitigation Steps:
Consider using OpenZeppelin’s SafeCast library to prevent unexpected overflows when casting from uint256.

##

### [L-2] Lack of checks-effects-interactions

It’s recommended to execute external calls after state changes, to prevent reetrancy bugs.

FILE : 2023-03-polynomial/src/ShortCollateral.sol

[ShortCollateral.sol#L85-L101](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortCollateral.sol#L85-L101)

##

### [L-3] Use safeMint variant instead of normal mint function

Te safeMint function helps to prevent common errors and vulnerabilities in ERC20 token contracts, such as integer overflows, invalid recipient addresses, and unexpected behavior due to insufficient checks. It can also help to protect against potential attacks or exploits that may try to exploit these types of vulnerabilities.

FILE : 2023-03-polynomial/src/Exchange.sol

  241:  powerPerp.mint(msg.sender, params.amount);

[Exchange.sol#L241](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L241)

##

### [L-4] LACK OF CHECKS THE INTEGER RANGES

The integer ranges not checked before assigning the values to state variables. This will cause the serious damage if we assigns unexpected values to state variables

FILE : 2023-03-polynomial/src/Exchange.sol

_priceNormalizationFactor > 0 value is not checked. As per current implamentations its possible to apply 0 to   PRICING_CONSTANT. This will cause serious problems for exchange.sol contract

   constructor(uint256 _priceNormalizationFactor, ISystemManager _systemManager)
        Auth(msg.sender, Authority(address(0x0)))
    {
        PRICING_CONSTANT = _priceNormalizationFactor;
        systemManager = _systemManager;
    }

[Exchange.sol#L67-L72](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L67-L72)

function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {
        emit UpdateSkewNormalizationFactor(skewNormalizationFactor, _skewNormalizationFactor);
        skewNormalizationFactor = _skewNormalizationFactor;
    }

[Exchange.sol#L216-L219](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L216-L219)

      function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
        emit UpdateMaxFundingRate(maxFundingRate, _maxFundingRate);
        maxFundingRate = _maxFundingRate;
    }

[Exchange.sol#L223-L226](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L223-L226)

FILE : 2023-03-polynomial/src/ShortCollateral.sol

     function collectCollateral(address collateral, uint256 positionId, uint256 amount)
        external
        onlyExchange
        nonReentrant
    {
        ERC20(collateral).safeTransferFrom(address(exchange), address(this), amount);

[ShortCollateral.sol#L85-L90] (https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortCollateral.sol#L85-L90)


Recommended Mitigation:

require(VARIABLE > 0, "Zero value"); 

##

### [L-5] LOSS OF PRECISION DUE TO ROUNDING

FILE : 2023-03-polynomial/src/Exchange.sol

  191: fundingRate = fundingRate / 1 days;

[Exchange.sol#L191](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L191)

   411: fundingRate = fundingRate / 1 days;

[Exchange.sol#L411](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L411)




##

### [L-4] 


# NON CRITICAL FINDINGS

##

### [NC-1] USE A MORE RECENT VERSION OF SOLIDITY

Use a solidity version of at least 0.8.12 to get string.concat() to be used instead of abi.encodePacked(<str>,<str>)

FILE : 2023-03-polynomial/src/Exchange.sol

    2: pragma solidity ^0.8.9;

[Exchange.sol#L2](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L2)

##

### [NC-2] NOT USING THE NAMED RETURN VARIABLES ANYWHERE IN THE FUNCTION IS CONFUSING

[Exchange.sol#L87-L92](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L87-L92)

[Exchange.sol#L167](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L167)
[Exchange.sol#L155](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L155)
[xchange.sol#L186](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L186)
[Exchange.sol#L100-L105](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L100-L105)
[Exchange.sol#L205-L206](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L205-L206)
[]()
[]()
[]()
[]()

##

### [NC-3] INCLUDE RETURN PARAMETERS IN NATSPEC COMMENTS

[Exchange.sol#L87-L92](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L87-L92)
[Exchange.sol#L98-L105](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L98-L105)
[Exchange.sol#L154-L155](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L154-L155)
[Exchange.sol#L166-L167](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L166-L167)
[Exchange.sol#L184-L186](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L184-L186)
[Exchange.sol#L205-L206](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L205-L206)

##

### [NC-4] Contract layout and order of functions

The Solidity style guide [recommends](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-layout) declaring events should declare bellow the state variables 

 But Exchange.sol file evens are declared bottom of the file 

Please declare the events as per solidity style guide 

FILE : 2023-03-polynomial/src/Exchange.sol

[Exchange.sol#L409-L522](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L409-L522)

##

### [NC-4] Use @dev or @param to explain the state variables . @notice tag used for explaining the functions 

CONTEXT
ALL CONTRACTS

[Exchange.sol#L40-L65](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L40-L65)

##

### [NC-5] NATSPEC is Missing 

[Exchange.sol#L67-L72](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L67-L72)
[Exchange.sol#L74-L79](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L74-L79)

##

### [NC-6] GENERATE PERFECT CODE HEADERS EVERY TIME

Description
I recommend using header for Solidity code layout and readability

[Headers](https://github.com/transmissions11/headers)

##

### [NC-7]  NO SAME VALUE INPUT CONTROL

FILE : 2023-03-polynomial/src/Exchange.sol

   function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {
        emit UpdateSkewNormalizationFactor(skewNormalizationFactor, _skewNormalizationFactor);
        skewNormalizationFactor = _skewNormalizationFactor;
    }

[Exchange.sol#L216-L219](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L216-L219)

      function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {
        emit UpdateMaxFundingRate(maxFundingRate, _maxFundingRate);
        maxFundingRate = _maxFundingRate;
    }

[Exchange.sol#L223-L226](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L223-L226)

##

### [NC-8] NATSPEC COMMENTS SHOULD BE INCREASED IN CONTRACTS

CONTEXT
ALL CONTRACTS

It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation.
In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability.
(https://docs.soliditylang.org/en/v0.8.15/natspec-format.html)

Recommendation
NatSpec comments should be increased in Contracts










NC-1	Missing checks for address(0) when assigning values to address state variables	1
NC-2	require() / revert() statements should have descriptive reason strings	89
NC-3	TODO Left in the code	1
NC-4	Event is missing indexed fields	67
NC-5	Constants should be defined rather than using magic numbers	1
NC-6	Functions not used internally could be marked external	26
NC-7	Typos

L-1	Do not use deprecated library functions	7
L-2	Empty Function Body - Consider commenting why	1
L-3	Initializers could be front-run	1
L-4	Unsafe ERC20 operation(s)	5