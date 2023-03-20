## Use practical code execution instead of `assert()`
In LiquidityPool.sol, the assert statement of `processDeposits()` and `processWithdraws()` may be replaced with a simple code logic that calculates `count` that will also help circumvent the overvalued input of `count` that reverts.

For instance, the following function may be refactored as follows:

[File: LiquidityPool.sol#L219-L242](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L219-L242)

```diff
-    function processDeposits(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_DEPOSITS") {
+    function processDeposits(uint256 count) external override nonReentrant whenNotPaused("POOL_PROCESS_DEPOSITS") {
-        assert(queuedDepositHead + count - 1 < nextQueuedDepositId);
+        uint256 count = nextQueuedDepositId - queuedDepositHead;
        uint256 tokenPrice = getTokenPrice();

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
    }
```
## Uninitialized variables
In LiquidityPool.sol, Exchange.sol, and KangarooVault.sol, consider initializing the settable variables in the constructor just to make sure function calls dependent on them are not going to break or malfunction due to calls being made before these variables manage to be set.

For instance, for failing failing to set [`baseTradingFee`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L665-L668) in time, it is going to directly affect [`getSlippageFee()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L367-L375), which in turns affects [`orderFee()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L391), [`openLong()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L441), [`closeLong()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L473), [`openShort()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L505), and [`closeShort()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L537) respectively. 

Although this can also be circumvented by `PauseModifier` upon contract deployment, it will require added work to maneuver since `systemManager.isPaused(key) == false` by default. 

## Zero value check at the constructor
Zero value checks should be implemented at the constructor to avoid accidental error(s) that could result in non-functional calls associated with it particularly when assigning immutable variables.

For instance, the constructor below may be refactored as follows:

[File: Exchange.sol#L67-L72](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L67-L72)

```diff
    constructor(uint256 _priceNormalizationFactor, ISystemManager _systemManager)
        Auth(msg.sender, Authority(address(0x0)))
    {
+        require(_priceNormalizationFactor != 0);
        PRICING_CONSTANT = _priceNormalizationFactor;
        systemManager = _systemManager;
    }
```
## Devoid of system documentation and complete technical specification
A system’s design specification and supporting documentation should be almost as important as the system’s implementation itself. Users rely on high-level documentation to understand the big picture of how a system works. Without spending time and effort to create palatable documentation, a user’s only resource is the code itself, something the vast majority of users cannot understand. Security assessments depend on a complete technical specification to understand the specifics of how a system works. When a behavior is not specified (or is specified incorrectly), security assessments must base their knowledge in assumptions, leading to less effective review. Maintaining and updating code relies on supporting documentation to know why the system is implemented in a specific way. If code maintainers cannot reference documentation, they must rely on memory or assistance to make high-quality changes. Currently, the only documentation available is a single README file, as well as code comments.

## Inadequate NatSpec
Solidity contracts can use a special form of comments, i.e., the Ethereum Natural Language Specification Format (NatSpec) to provide rich documentation for functions, return variables and more. Please visit the following link for further details:

https://docs.soliditylang.org/en/v0.8.16/natspec-format.html

Consider fully equipping all contracts with complete set of NatSpec to better facilitate users/developers interacting with the protocol's smart contracts.

For example, the following function instance could be better equipped with its missing `@param referralCode` and `@return totalCost`:

[File: LiquidityPool.sol#L427-L436](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L427-L436)

```solidity
    /// @notice Called by exchange when a new long position is created
    /// @param amount Amount of square perp being longed
    /// @param user Address of the user
    function openLong(uint256 amount, address user, bytes32 referralCode)
        external
        override
        onlyExchange
        nonReentrant
        returns (uint256 totalCost)
    {
```
## Non-compliant contract layout with Solidity's Style Guide
According to Solidity's Style Guide below:

https://docs.soliditylang.org/en/v0.8.17/style-guide.html

In order to help readers identify which functions they can call, and find the constructor and fallback definitions more easily, functions should be grouped according to their visibility and ordered in the following manner:

constructor, receive function (if exists), fallback function (if exists), external, public, internal, private

And, within a grouping, place the `view` and `pure` functions last.

Additionally, inside each contract, library or interface, use the following order:

type declarations, state variables, events, modifiers, functions

Consider adhering to the above guidelines for all contract instances entailed.

## On chain over off chain implementation of `referralCode`
`referralCode` currently works as a function parameter serving only to be emitted in the trading methods. Consider adopting a brief but comprehensive set of on chain logic that will make it permissionless instead of handling the affiliate programs off chain via backend API logging.      

## Constants should be defined rather than using magic numbers
Consider defining magic numbers to constants, serving to improve code readability and maintainability.

Here is an instance entailed pertaining to `2e18`:

[File: LiquidityPool.sol#L733](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L733)

```solidity
        delta = spotPrice.mulDivDown(2e18, pricingConstant);
```
