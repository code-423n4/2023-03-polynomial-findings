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

FILE : 2023-03-polynomial/src/LiquidityPool.sol

   295: uint256 availableFunds = uint256(int256(totalFunds) - usedFunds);

[LiquidityPool.sol#L295](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L295)

Recommended Mitigation Steps:
Consider using OpenZeppelin’s SafeCast library to prevent unexpected overflows when casting from uint256.

##

### [L-2] Lack of checks-effects-interactions

It’s recommended to execute external calls after state changes, to prevent reetrancy bugs.

FILE : 2023-03-polynomial/src/ShortCollateral.sol

[ShortCollateral.sol#L85-L101](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortCollateral.sol#L85-L101)

FILE : 2023-03-polynomial/src/KangarooVault.sol

[KangarooVault.sol#L220-L224](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L220-L224)


##

### [L-3] Use safeMint variant instead of normal mint function

Te safeMint function helps to prevent common errors and vulnerabilities in ERC20 token contracts, such as integer overflows, invalid recipient addresses, and unexpected behavior due to insufficient checks. It can also help to protect against potential attacks or exploits that may try to exploit these types of vulnerabilities.

FILE : 2023-03-polynomial/src/Exchange.sol

     241:  powerPerp.mint(msg.sender, params.amount);

[Exchange.sol#L241](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L241)

FILE : 2023-03-polynomial/src/LiquidityPool.sol

    189:   liquidityToken.mint(user, tokensToMint);

[LiquidityPool.sol#L189](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L189)

    235:  liquidityToken.mint(current.user, tokensToMint);

[LiquidityPool.sol#L235](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L235)

FILE : 2023-03-polynomial/src/KangarooVault.sol

    191: VAULT_TOKEN.mint(user, tokensToMint);

[KangarooVault.sol#L191](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L191)

    258: VAULT_TOKEN.mint(current.user, tokensToMint);

[KangarooVault.sol#L258](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L258)

    

##

### [L-4] LACK OF CHECKS THE INTEGER RANGES

The integer ranges not checked before assigning the values to state variables. This will cause the serious damage if we assigns unexpected values to state variables and functions 

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

FIE : 2023-03-polynomial/src/LiquidityPool.sol

the amount not checked with zero value 

[LiquidityPool.sol#L184-L195](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L184-L195)

[LiquidityPool.sol#L247-L250](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L247-L250)

[LiquidityPool.sol#L657-L661](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L657-L661)

[LiquidityPool.sol#L665-L668](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L665-L668)

[LiquidityPool.sol#L672-L682](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L672-L682)

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

### [L-6] LACK OF CHECKS ADDRESS(0)

The following methods have a lack of checks if the received argument is an address, it’s good practice in order to reduce human error to check that the address specified in the function is different than address(0)

FILE : FILE : 2023-03-polynomial/src/LiquidityPool.sol

  function setFeeReceipient(address _feeReceipient) external requiresAuth {
        feeReceipient = _feeReceipient;
    }

[LiquidityPool.sol#L650-L652](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L650-L652)

##

### [L-7] CRITICAL ADDRESS CHANGES SHOULD USE TWO-STEP PROCEDURE 

The critical procedures should be two step process.
See similar findings in previous Code4rena contests for reference:
(https://code4rena.com/reports/2022-06-illuminate/#2-critical-changes-should-use-two-step-procedure)

FILE : FILE : 2023-03-polynomial/src/LiquidityPool.sol

  function setFeeReceipient(address _feeReceipient) external requiresAuth {
        feeReceipient = _feeReceipient;
    }

[LiquidityPool.sol#L650-L652](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L650-L652)

##

### [L-8] Solmate's SafeTransferLib doesn't check whether the ERC20 contract exists

Solmate's SafeTransferLib, which is often used to interact with non-compliant/unsafe ERC20 tokens, does not check whether the ERC20 contract exists. The following code will not revert in case the token doesn't exist (yet)

FILE : 2023-03-polynomial/src/KangarooVault.sol

  207: SUSD.safeTransferFrom(msg.sender, address(this), amount);

[KangarooVault.sol#L207](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L207)

##

### [L-9] User balance is not checked before proceed initiateDeposit()

As per current implementations user can call initiateDeposit() function without having enough balance in user account 

FILE : 2023-03-polynomial/src/KangarooVault.sol

[KangarooVault.sol#L183-L208](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L183-L208)  

Recombined Mitigation:

require (address(user).balance > amount, "Not enough balance ");

##

### [L-10] A single point of failure

The requiresAuth role has a single point of failure and requiresAuth can use critical a few functions.

Even if protocol admins/developers are not malicious there is still a chance for Owner keys to be stolen. In such a case, the attacker can cause serious damage to the project due to important functions. In such a case, users who have invested in project will suffer high financial losses

requiresAuth  functions :

FILE : 2023-03-polynomial/src/Exchange.sol

   216: function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {

   223: function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {

[Exchange.sol#L216-L226](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L216-L226)

FILE : 2023-03-polynomial/src/KangarooVault.sol

     function setFeeReceipient(address _feeReceipient) external requiresAuth {
        require(_feeReceipient != address(0x0));

        emit UpdateFeeReceipient(feeReceipient, _feeReceipient);

        feeReceipient = _feeReceipient;
     }

[KangarooVault.sol#L465-L471](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L465-L471)  

   function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
        require(_performanceFee <= 1e17 && _withdrawalFee <= 1e16);

        emit UpdateFees(performanceFee, withdrawalFee, _performanceFee, _withdrawalFee);

        performanceFee = _performanceFee;
        withdrawalFee = _withdrawalFee;
    }

[KangarooVault.sol#L476-L483](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L476-L483)

[KangarooVault.sol#L487-L550](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L487-L550)

##

### [L-11] Front running attacks by the requiresAuth 

All parameters values are not a constant value and can be changed with setter functions, before a function using the setter methods state variables value in the project, setter functions can be triggered by requiresAuth and operations can be blocked

FILE : 2023-03-polynomial/src/Exchange.sol

   216: function setSkewNormalizationFactor(uint256 _skewNormalizationFactor) external requiresAuth {

   223: function setMaxFundingRate(uint256 _maxFundingRate) external requiresAuth {

[Exchange.sol#L216-L226](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L216-L226)

FILE : 2023-03-polynomial/src/KangarooVault.sol

     function setFeeReceipient(address _feeReceipient) external requiresAuth {
        require(_feeReceipient != address(0x0));

        emit UpdateFeeReceipient(feeReceipient, _feeReceipient);

        feeReceipient = _feeReceipient;
     }

[KangarooVault.sol#L465-L471](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L465-L471)  

   function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
        require(_performanceFee <= 1e17 && _withdrawalFee <= 1e16);

        emit UpdateFees(performanceFee, withdrawalFee, _performanceFee, _withdrawalFee);

        performanceFee = _performanceFee;
        withdrawalFee = _withdrawalFee;
    }

[KangarooVault.sol#L476-L483](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L476-L483)

[KangarooVault.sol#L487-L550](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L487-L550)


##
# NON CRITICAL FINDINGS

##

### [NC-1] USE A MORE RECENT VERSION OF SOLIDITY

Use a solidity version of at least 0.8.12 to get string.concat() to be used instead of abi.encodePacked(<str>,<str>)

FILE : 2023-03-polynomial/src/Exchange.sol

    2: pragma solidity ^0.8.9;

[Exchange.sol#L2](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L2)\

FILE : 2023-03-polynomial/src/LiquidityPool.sol

    2: pragma solidity ^0.8.9;

[LiquidityPool.sol#L2](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L2)

FILE : 2023-03-polynomial/src/KangarooVault.sol

   2: pragma solidity ^0.8.9;

[KangarooVault.sol#L2](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L2)

##

### [NC-2] NOT USING THE NAMED RETURN VARIABLES ANYWHERE IN THE FUNCTION IS CONFUSING

[Exchange.sol#L87-L92](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L87-L92)

[Exchange.sol#L167](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L167)
[Exchange.sol#L155](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L155)
[xchange.sol#L186](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L186)
[Exchange.sol#L100-L105](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L100-L105)
[Exchange.sol#L205-L206](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L205-L206)
[LiquidityPool.sol#L430-L435](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L430-L435)
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

### [NC-5] Use @dev or @param to explain the state variables . @notice tag used for explaining the functions 

CONTEXT
ALL CONTRACTS

[Exchange.sol#L40-L65](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L40-L65)

##

### [NC-6] NATSPEC is Missing 

[Exchange.sol#L67-L72](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L67-L72)
[Exchange.sol#L74-L79](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L74-L79)
[LiquidityPool.sol#L27-L49](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L27-L49)

##

### [NC-7] GENERATE PERFECT CODE HEADERS EVERY TIME

Description
I recommend using header for Solidity code layout and readability

[Headers](https://github.com/transmissions11/headers)

##

### [NC-8]  NO SAME VALUE INPUT CONTROL

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

FILE : FILE : 2023-03-polynomial/src/LiquidityPool.sol

  function setFeeReceipient(address _feeReceipient) external requiresAuth {
        feeReceipient = _feeReceipient;
    }

[LiquidityPool.sol#L650-L652](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L650-L652)

##

### [NC-9] NATSPEC COMMENTS SHOULD BE INCREASED IN CONTRACTS

CONTEXT
ALL CONTRACTS

It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation.
In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability.
(https://docs.soliditylang.org/en/v0.8.15/natspec-format.html)

Recommendation
NatSpec comments should be increased in Contracts

##

### [NC-10] Pragma float

All the contracts in scope are floating the pragma version.

Recommendation
Locking the pragma helps to ensure that contracts do not accidentally get deployed using an outdated compiler version.

Note that pragma statements can be allowed to float when a contract is intended for consumption by other developers, as in the case with contracts in a library or a package.

##

### [NC-11] GENERATE PERFECT CODE HEADERS EVERY TIME

Description
I recommend using header for Solidity code layout and readability

[PERFECT CODE HEADERS](https://github.com/transmissions11/headers)

##

### [NC-12] Shorter inheritance list

FILE : 2023-03-polynomial/src/LiquidityPool.sol

  27: contract LiquidityPool is ILiquidityPool, Auth, ReentrancyGuard, PauseModifier {

[LiquidityPool.sol#L27](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L27)

##

### [NC-13] Uppercase immutable variables

FILE : 2023-03-polynomial/src/LiquidityPool.sol

   56: bytes32 public immutable baseAsset;

[LiquidityPool.sol#L56](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L56)

FILE : 2023-03-polynomial/src/KangarooVault.sol

   60:  bytes32 public immutable name;

[KangarooVault.sol#L60](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L60)

FILE : 2023-03-polynomial/src/PowerPerp.sol

[PowerPerp.sol#L9-L10](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/PowerPerp.sol#L9-L10)

FILE : 2023-03-polynomial/src/ShortToken.sol

[ShortToken.sol#L9](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortToken.sol#L9)  
[ShortToken.sol#L11](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortToken.sol#L11)

##

### [NC-14] Inside the immutable header there are normal state variables also declared. This is not a good code practice 

FILE : 2023-03-polynomial/src/LiquidityPool.sol

[LiquidityPool.sol#L52-L68](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L52-L68)

##

### [NC-15] Use require instead of assert

The Solidity assert() function is meant to assert invariants. Properly functioning code should never reach a failing assert statement.

FILE : 2023-03-polynomial/src/LiquidityPool.sol

   220: assert(queuedDepositHead + count - 1 < nextQueuedDepositId);

[LiquidityPool.sol#L220](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L220)

##

### [NC-16] Use any error codes with descriptive reasons of failure instead of empty return 

FILE : 2023-03-polynomial/src/LiquidityPool.sol

   227:  return;

[LiquidityPool.sol#L227](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L227)

##

### [NC-17] Wrong description for variable

FILE : 2023-03-polynomial/src/LiquidityPool.sol

    86: /// @notice Minimum deposit delay
    87: uint256 public minWithdrawDelay;

Recommendations:

    86: /// @notice Minimum withdraw delay
    87: uint256 public minWithdrawDelay;

##

### [NC-18]  Add a timelock to critical functions

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

FILE : 2023-03-polynomial/src/KangarooVault.sol

     function setFeeReceipient(address _feeReceipient) external requiresAuth {
        require(_feeReceipient != address(0x0));

        emit UpdateFeeReceipient(feeReceipient, _feeReceipient);

        feeReceipient = _feeReceipient;
     }

[KangarooVault.sol#L465-L471](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L465-L471)  

   function setFees(uint256 _performanceFee, uint256 _withdrawalFee) external requiresAuth {
        require(_performanceFee <= 1e17 && _withdrawalFee <= 1e16);

        emit UpdateFees(performanceFee, withdrawalFee, _performanceFee, _withdrawalFee);

        performanceFee = _performanceFee;
        withdrawalFee = _withdrawalFee;
    }

[KangarooVault.sol#L476-L483](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L476-L483)

[KangarooVault.sol#L487-L550](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L487-L550)

##

### [NC-19] The nonReentrant modifier should occur before all other modifiers

This is a best-practice to protect against reentrancy in other modifiers

FILE : 2023-03-polynomial/src/LiquidityPool.sol

      function closeLong(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCost)

[LiquidityPool.sol#L462-L467](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L462-L467)

    function openShort(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCost)

[LiquidityPool.sol#L494-L499](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L494-L499)

[LiquidityPool.sol#L526-L531](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L526-L531)

[LiquidityPool.sol#L557](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L557)
[LiquidityPool.sol#L591](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L591)  

[LiquidityPool.sol#L613](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L613)

FILE : 2023-03-polynomial/src/KangarooVault.sol

   376:  function openPosition(uint256 amt, uint256 minCost) external requiresAuth nonReentrant {

  383:   function closePosition(uint256 amt, uint256 maxCost) external requiresAuth nonReentrant {

   389:    function clearPendingOpenOrders(uint256 maxCost) external requiresAuth nonReentrant {
 
  395:   function clearPendingCloseOrders(uint256 minCost) external requiresAuth nonReentrant {

  401:   function transferPerpMargin(int256 marginDelta) external requiresAuth nonReentrant {
   
   424:   function addCollateral(uint256 additionalCollateral) external requiresAuth nonReentrant {

   436:  function removeCollateral(uint256 collateralToRemove) external requiresAuth nonReentrant {

   450:  function executePerpOrders(bytes[] calldata priceUpdateData) external payable requiresAuth nonReentrant {

   











FILE : 2023-03-polynomial/src/LiquidityPool.sol
FILE : 2023-03-polynomial/src/Exchange.sol

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