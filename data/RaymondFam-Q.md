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

## Unspecific compiler version pragma
For some source-units the compiler version pragma is very unspecific, i.e. ^0.8.9. While this often makes sense for libraries to allow them to be included with multiple different versions of an application, it may be a security risk for the actual application implementation itself. A known vulnerable compiler version may accidentally be selected or security tools might fall-back to an older compiler version ending up actually checking a different EVM compilation that is ultimately deployed on the blockchain.

Avoid floating pragmas where possible. It is highly recommend pinning a concrete compiler version (latest without security issues) in at least the top-level “deployed” contracts to make it unambiguous which compiler version is being used. Rule of thumb: a flattened source-unit should have at least one non-floating concrete solidity compiler version pragma.

## Use a more recent version of solidity
The protocol adopts version 0.8.9 on writing contracts. For better security, it is best practice to use the latest Solidity version, 0.8.17.

Security fix list in the versions can be found in the link below:

https://github.com/ethereum/solidity/blob/develop/Changelog.md

## Solidity Compiler optimization could be problematic
```
hardhat.config.js:
  29  module.exports = {
  30:   solidity: {
  31:     compilers: [
  32:       {
  33:         version: "0.8.9",
  34:         settings: {
  35:           optimizer: {
  36:               enabled: true,
  37:               runs: 1000000
  38
            }
```
Description: The protocol has enabled optional compiler optimizations in Solidity. There have been several optimization bugs with security implications. Moreover, optimizations are actively being developed. Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them.

Therefore, it is unclear how well they are being tested and exercised. High-severity security issues due to optimization bugs have occurred in the past. A high-severity bug in the emscripten-generated solc-js compiler used by Truffle and Remix persisted until late 2018. The fix for this bug was not reported in the Solidity CHANGELOG.

Another high-severity optimization bug resulting in incorrect bit shift results was patched in Solidity 0.5.6. More recently, another bug due to the incorrect caching of keccak256 was reported. A compiler audit of Solidity from November 2018 concluded that the optional optimizations may not be safe. It is likely that there are latent bugs related to optimization and that new bugs will be introduced due to future optimizations.

Exploit Scenario A latent or future bug in Solidity compiler optimizations—or in the Emscripten transpilation to solc-js—causes a security vulnerability in the contracts.

Recommendation: Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug. Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

## Use `delete` to clear variables
`delete a` assigns the initial value for the type to `a`. i.e. for integers it is equivalent to `a = 0`, but it can also be used on arrays, where it assigns a dynamic array of length zero or a static array of the same length with all elements reset. For structs, it assigns a struct with all members reset. Similarly, it can also be used to set an address to zero address or a boolean to false. It has no effect on whole mappings though (as the keys of mappings may be arbitrary and are generally unknown). However, individual keys and what they map to can be deleted: If `a` is a mapping, then `delete a[x]` will delete the value stored at x.

The delete key better conveys the intention and is also more idiomatic.

For instance, the `a[x] = false` instance below may be refactored as follows:

[File: KangarooVault.sol#L719](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L719)

```diff
-        lastTradeData.isOpen = false;
+        delete lastTradeData.isOpen;
```
## Tokens accidentally sent to the contract cannot be recovered
It is deemed unrecoverable if the tokens accidentally arrive at the contract addresses, which has happened to many popular projects. Consider adding a recovery code to your critical contracts.

## Gas griefing/theft is possible on unsafe external call
`return` data (bool success,) has to be stored due to EVM architecture, if in a usage like below, ‘out’ and ‘outsize’ values are given (0,0). Thus, this storage disappears and may come from external contracts a possible gas grieving/theft problem is avoided as denoted in the link below:

https://twitter.com/pashovkrum/status/1607024043718316032?t=xs30iD6ORWtE2bTTYsCFIQ&s=19

Here are the two instances entailed:

[File: KangarooVault.sol#L454-L457](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L454-L457)
[File: LiquidityPool.sol#L708-L713](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L708-L713)

```solidity
        (bool success,) = feeReceipient.call{value: msg.value}("");
        require(success);
```
## More in-depth tests needed
Consider extending the test suites to include edge cases, and where possible implement fuzzing that will often unfold many unexpected incidents that could break. This will greatly make the code safer and more robust to vulnerability attacks.