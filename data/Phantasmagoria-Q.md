## Low Risk Issues

### [L-1] Admin can set any amount of fee
In setFees() and setDevFee() functions of LiquidityPool.sol contract admin can set any fee. This can potentially be a problem if the admin sets an exorbitant or unreasonable fee, as it can negatively impact the liquidity and usability of the pool.

Since the admin has the ability to set the fee, they could potentially manipulate the system in their favor, for example, by setting a very high fee that discourages users from participating in the pool, or by setting a lower fee for themselves or their friends, giving them an unfair advantage.
```
657: function setFees(uint256 _depositFee, uint256 _withdrawalFee) external requiresAuth {
658:         emit UpdateFees(depositFee, _depositFee, withdrawalFee, _withdrawalFee);
659:         depositFee = _depositFee;
660:         withdrawalFee = _withdrawalFee;
661: }

672: function setDevFee(uint256 _devFee) external requiresAuth {
673:         emit UpdateDevFee(devFee, _devFee);
674:         devFee = _devFee;
675: }
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L657-L661
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L672-L675
##### Mitigation 
Add a require statement that will prevent the admin from setting high fees

### [L-2] Event should be emitted in setters
Setters should emit an event
```
File: src/LiquidityPool.sol

650: function setFeeReceipient(address _feeReceipient) external requiresAuth {
651:         feeReceipient = _feeReceipient;
652: }
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L650-L652

```
File: src/VaultToken.sol

35: function setVault(address _vault) external {
36:         if (vault != address(0x0)) {
37:             revert();
38:         }
39:         vault = _vault;
40: }
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L35-L40

Additionally, an event should be emitted in the receive() function of the KangarooVault.sol contract. 
```
454: receive() external payable {
455:         (bool success,) = feeReceipient.call{value: msg.value}("");
456:         require(success);
457: }
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L454-L457

In the same function in LiquidityPool.sol such event is present
```
708: receive() external payable {
709:         (bool success,) = feeReceipient.call{value: msg.value}("");
710:         require(success);
711: 
712:         emit ReceiveEther(msg.sender, msg.value);
713: }
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L708-L713

### [L-3] `minWithdrawDelay` and `minDepositDelay` variables can be set to an invalid value
According to the documentation minimum withdrawal time ranges between 6-24 hours.  
"Since there is no round system in the newer version, there will be a minimum withdrawal time to exit  from the position. The min withdrawal time ranges between 6-24hours"
But in `setDelays()` of `KangarooVault.sol` and `setMinDelays()` of `LiquidityPool.sol` there is no check to ensure that delays was set in appropriate ranges
It is a good to add a check to ensure that the delays are set within the appropriate range. This would help prevent errors or vulnerabilities that could occur if the delays were set to an invalid value.
```
File: src/KangarooVault.sol

514: function setDelays(uint256 _depositDelay, uint256 _withdrawDelay) external requiresAuth {
515:         emit UpdateDelays(minDepositDelay, _depositDelay, minWithdrawDelay, _withdrawDelay);
516:         minDepositDelay = _depositDelay;
517:         minWithdrawDelay = _withdrawDelay;
518: }
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L514-L518

```
File: src/LiquidityPool.sol

685: function setMinDelays(uint256 _minDepositDelay, uint256 _minWithdrawDelay) external requiresAuth {
686:         emit UpdateDelays(minDepositDelay, _minDepositDelay, minWithdrawDelay, _minWithdrawDelay);
687:         minDepositDelay = _minDepositDelay;
688:         minWithdrawDelay = _minWithdrawDelay;
689: }
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L685-L689

##### Mitigation
Add check to ensure that the delays are set within the appropriate range

### [L-4] Gas griefing/theft is possible on unsafe external call
return data (bool success,) has to be stored due to EVM architecture, if in a usage like below, ‘out’ and ‘outsize’ values are given (0,0) . Thus, this storage disappears and may come from external contracts a possible Gas griefing/theft problem is avoided
```
File: src/KangarooVault.sol

455: (bool success,) = feeReceipient.call{value: msg.value}("");
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L455
```
File: src/LiquidityPool.sol

709: (bool success,) = feeReceipient.call{value: msg.value}("");
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L709

### [L-5] Misleading comment
Have to be "Minimum withdrawal delay" instead of "Minimum deposit delay"
```
File: src/LiquidityPool.sol

86: /// @notice Minimum deposit delay
87: uint256 public minWithdrawDelay;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L86-L87

### [L-6] Floating pragma
Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

All contracts except /src/VaultToken.sol
## Non-Critical Issues

### [NC-1] Use require() instead of assert()
Properly functioning code should never reach a failing assert statement. If it happened, it would indicate the presence of a bug in the contract. A failing assert uses all the remaining gas, which can be financially painful for a user.
```
File: src/LiquidityPool.sol

220: assert(queuedDepositHead + count - 1 < nextQueuedDepositId);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L220
```
File: src/LiquidityPool.sol

285: assert(queuedDepositHead + count - 1 < nextQueuedDepositId);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L285

### [NC-2] Absent natspec comments
Reasons why you might want to use natspec comments in your Solidity smart contracts:
1) Improving code readability: Natspec comments can help make your code more understandable and easier to read for both other developers and non-technical stakeholders.
2) Facilitating automated documentation generation: Natspec comments can be used by tools like Solidity's built-in documentation generator or external tools like Doxygen or Sphinx to automatically generate documentation for your contract.
3) Clarifying contract behavior: By providing clear descriptions of the expected behavior of your contract's functions, natspec comments can help prevent misunderstandings or errors that might arise if users misinterpret the intended behavior.
4) Enhancing contract security: Natspec comments can help identify potential security vulnerabilities in your contract by highlighting its intended behavior, which can make it easier for other developers to review your code and identify potential issues.
Contracts that don't have natspec comments:
[src/VaultToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol)
[src/LiquidityToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityToken.sol)
[src/ShortToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol)
[src/PowerPerp.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol)

### [NC-3] Mark visibility of init(…) functions as external
External instead of public would give more the sense of the init(…) functions to behave like a constructor (only called on deployment, so should only be called externally).
Security point of view, it might be safer so that it cannot be called internally by accident in the child contract.
```
File: src/SystemManager.sol

62: function init(
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L62

### [NC-4] Contract layout and order of functions
Inside each contract, library or interface, use the following order:
1.Type declarations
2.State variables
3.Events
4.Modifiers
5.Functions

Reference: https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-layout

Events in the following contracts are at the bottom. Consider moving them to the top:
[src/ShortCollateral.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol)
[src/SystemManager.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol)
[src/ShortToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol)

### [NC-5] Follow a standard convention function names such as using get for getter (view/pure) functions
Getter functions are used to retrieve state variables in a contract without modifying them. It is common to prefix these functions with "get". For example:
```
function getBalance() public view returns(uint256) {
    return balance;
}
```
Instances:
```
File: src/interfaces/IExchange.sol

51: function normalizationFactor() external view returns (uint256);
53: function skewNormalizationFactor() external view returns (uint256);
55: function maxFundingRate() external view returns (uint256);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IExchange.sol
```
File: src/interfaces/ILiquidityPool.sol
39: function minDepositDelay() external view returns (uint256);
41: function minWithdrawDelay() external view returns (uint256);
43: function baseTradingFee() external view returns (uint256);
45: function depositFee() external view returns (uint256);
47: function withdrawalFee() external view returns (uint256);
49: function devFee() external view returns (uint256);
53: function totalQueuedDeposits() external view returns (uint256);
55: function totalQueuedWithdrawals() external view returns (uint256);
69: function queuedPerpSize() external view returns (int256);
71: function futuresLeverage() external view returns (uint256);
73: function perpPriceImpactDelta() external view returns (uint256);
75: function standardSize() external view returns (uint256);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ILiquidityPool.sol
```
File: src/interfaces/synthetix/IFuturesMarket.sol
11: function marketKey() external view returns (bytes32 key);
13: function baseAsset() external view returns (bytes32 key);
15: function marketSize() external view returns (uint128 size);
17: function marketSkew() external view returns (int128 skew);
19: function fundingLastRecomputed() external view returns (uint32 timestamp);
21: function fundingSequence(uint256 index) external view returns (int128 netFunding);
34: function currentFundingRate() external view returns (int256 fundingRate);
36: function unrecordedFunding() external view returns (int256 funding, bool invalid);
38: function fundingSequenceLength() external view returns (uint256 length);
46: function accruedFunding(address account) external view returns (int256 funding, bool invalid);
48: function remainingMargin(address account) external view returns (uint256 marginRemaining, bool invalid);
50: function accessibleMargin(address account) external view returns (uint256 marginAccessible, bool invalid);
52: function liquidationPrice(address account) external view returns (uint256 price, bool invalid);
54: function liquidationFee(address account) external view returns (uint256);
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IFuturesMarket.sol
```
File: src/ShortToken.sol

44: function tokenURI(uint256 tokenId) public view override returns (string memory) {
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol

### [NC-6] require() / revert() statements should have descriptive reason strings
It is considered a best practice to always include a descriptive reason string when using the require() and revert() statements. This reason string explains why the condition or parameter failed, and helps in debugging and understanding the cause of the error.

The reason string is also useful for users of the contract, as it provides clear and concise feedback on what went wrong and how to resolve the issue. By providing descriptive and informative reason strings, developers can improve the usability and security of their smart contracts, and make it easier for others to interact with their code.
```
File: src/VaultToken.sol

37: revert();
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol#L37

### [NC-7] Upgrade solidity version
For security, it is best practice to use the latest Solidity version.
Instead of 0.8.9 newer version can be used 0.8.17
All contracts

### [NC-8] Natspec comments should be increased
Some functions in the following contracts doesn't have natspec comments
Functions that don't have natspec comments:
```
File: src/SystemManager.sol

refreshSynthetixAddresses(), setStatusFunction()
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol
```
File: src/KangarooVault.sol

_resetTrade(), _clearPendingCloseOrders(), _closePosition(), _clearPendingOpenOrders(), _openPosition()
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol
```
File: src/LiquidityPool.sol

_placeDelayedOrder(),  executePerpOrders(), getSlippageFee()
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol

### [NC-9] revert should be used instead of return;
There are cases where certain instances return without performing any actions, it may be worth considering implementing a "revert" statement with a descriptive string to explain why this is the case.
[src/KangarooVault.sol#L250](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L250)
[src/KangarooVault.sol#L276](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L276)
[src/KangarooVault.sol#L282](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L282)
[src/KangarooVault.sol#L309](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L309)
[src/LiquidityPool.sol#L227](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L227)
[src/LiquidityPool.sol#L292](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L292)
[src/LiquidityPool.sol#L298](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L298)
[src/LiquidityPool.sol#L317](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L317)
[src/LiquidityPool.sol#L789](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L789)

### [NC-10] Use delete instead of zero assignment
Instead of assigning a value of zero to a variable, it is possible to use the "delete" keyword.
```
File: src/KangarooVault.sol

328: current.withdrawnTokens = 0;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L328
```
File: src/KangarooVault.sol

641: positionData.premiumCollected = 0;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L641
```
File: src/KangarooVault.sol

667: positionData.pendingLongPerp = 0;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L667
```
File: src/KangarooVault.sol

779: positionData.pendingShortPerp = 0;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L779
```
File: src/KangarooVault.sol

783: positionData.positionId = 0;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L783
```
File: src/KangarooVault.sol

794: positionData.premiumCollected = 0;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L794
```
File: src/KangarooVault.sol

795: positionData.totalMargin = 0;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L795
```
File: src/KangarooVault.sol

796: usedFunds = 0;
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L796

### [NC-11] Generate perfect code headers
You can generate headers with the following [tool](https://github.com/transmissions11/headers)
Example:
```
/*//////////////////////////////////////////////////////////////
                          Admin functions
//////////////////////////////////////////////////////////////*/
```
Contracts where tool can be used:
[VaultToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/VaultToken.sol)
[ShortCollateral.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol)
[LiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol)
[KangarooVault.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol)
[Exchange.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol)
[interfaces/IExchange.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IExchange.sol)
[interfaces/ILiquidityToken.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ILiquidityToken.sol)
[interfaces/IShortCollateral.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IShortCollateral.sol)
[interfaces/ILiquidityPool.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/ILiquidityPool.sol)
[interfaces/IPowerPerp.sol](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/IPowerPerp.sol)

### [NC-12] 404 links
Following links do not work
```
File: src/interfaces/synthetix/IExchangeRates.sol

4: https://docs.synthetix.io/contracts/source/interfaces/iexchangerates
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IExchangeRates.sol#L4

```
File: src/interfaces/synthetix/IAddressResolver.sol

4: https://docs.synthetix.io/contracts/source/interfaces/iaddressresolver
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IAddressResolver.sol#L4


### [NC-13] Remove console2.sol
```
File: src/KangarooVault.sol

4: import {console2} from "forge-std/console2.sol";
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L4

### [NC-14] Missing space in comment
Add space to the comment 
```
File: src/interfaces/synthetix/IExchangeRates.sol

1: //SPDX-License-Identifier:MIT
```
https://github.com/code-423n4/2023-03-polynomial/blob/main/src/interfaces/synthetix/IExchangeRates.sol#L1
