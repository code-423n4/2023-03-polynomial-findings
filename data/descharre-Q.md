# Summary
## Low Risk
|ID     | Finding| Instances |
|:----: | :---           |   :----:         |
|L-01       |Unspecific compiler version pragma|1|
|L-02       |SafeTransfer is not used consistently|2|
|L-03       |Missing 0 address check in the constructor|3|

## Non critical
|ID     | Finding| Instances |
|:----: | :---           |   :----:         |
|N-01       |NatSpec is incomplete| 1|
|N-02       |Foundry import should be removed| 1|
|N-03       |UPPER_CASE_WITH_UNDERSCORES should only be used for constants| 4|
|N-04       |Missing check to see if open short position actually exists| 1|
|N-05       |Fees are not being set in the constructor| 1|
|N-06       |Tokens to feerecipient shouldn't be transferred every trade| 1|
|N-07       |Use a more recent version of solidity| 1|
|N-08       |Missing check for open trades| 1|
|N-09       |Use super instead of ERC20| 2|
|N-10       |Make variables immutable or create setters| 2|
|N-11       |Timestamps don't need to be bigger than uint48| 4|
|N-12       |Use require instead of assert when it's not something that never should be false| 2|
|N-13       |Event is missing indexed fields| 2|

# Details
## Low Risk
## [L-01] Unspecific compiler version pragma
Contracts should be deployed with the same compiler version and flags that they have been tested the most with. Locking the pragma helps ensure that contracts do not accidentally get deployed using, for example, the latest compiler which may have higher risks of undiscovered bugs. Contracts may also be deployed by others and the pragma indicates the compiler version intended by the original authors.
## [L-02] SafeTransfer is not used consistently
The solmate safeTransfer library is used for weird ERC20 tokens that don't return a boolean on transfers. The library is used in the protocol but not consistently.
- [LiquidityPool.sol#L254](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L254)
- [KangarooVault.sol#L549](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L549)
## [L-03] Missing 0 address checks in constructor
Allowing zero-addresses will lead to contract reverts and force redeployments if there are no setters for such address variables.
- [ShortToken.sol#L30](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L30): `systemManager` is immutable
- [KangarooVault.sol#L165-L171](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L165-L171)
- [ShortCollateral.sol#L61](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L61)
## Non critical
## [N-01] NatSpec is incomplete
Return tag is always missing from the NatSpec comments, as well as some functions that have no NatSpec comments.

Sometimes a param tag is missing
- [LiquidityPool.sol#L780](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L780)
- [LiquidityPool.sol#L798](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L798)
## [N-02] Foundry import should be removed
[KangarooVault](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L4) still has the foundry console import. This should be used solely for debugging and is not supposed to be pushed onto the mainnet. Import should be removed.
## [N-03] UPPER_CASE_WITH_UNDERSCORES should only be used for constants
According to the solidity style guide, upper case should only be used for constants and not immutable variables. Use for immutable variables the normal mixedCase.
- [SystemManager.sol#L24-L27](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L24-L27)
- [KangarooVault.sol#L60-L78](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L60-L78)
- [LiquidityPool.sol#L65](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L65)
- [Exchange.sol#L34](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L34)
## [N-04] Missing check to see if short position actually exists
When you open a new short trade with a positionId different from 0. If this is the first position the transaction will revert without any explanation. A good idea would be add a string to the require statement.

[ShortToken.sol#L69](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L69)
```diff
    if (params.positionId != 0) {
-       require(trader == ownerOf(positionId));
+ 		require(trader == ownerOf(positionId), "No open position");
    }
```
## [N-05] Fees are not being set in the constructor
For safety measures it's best to initialize the fees in a constructor so they can't be accidently kept at 0. In [LiquidityPool](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L90-L99) they can only be set with setters. It's also way more easier to just initialize them in the constructor instead of calling all the setters functions.
## [N-06] Tokens to feerecipient shouldn't be transferred every trade
When a user opens or close a trade, he wants to save as much money as possible and unnecessary spending doesn't contribute to that. When a user does a trade, the fee for the protocol gets transferred in the same transaction. It's better to get the feeRecipient to claim his fees on a regular basis when he want and not every trade.

This can be done by keeping track of the amount of protocolFees and make a seperate claimFunction that can only be called by the feeRecipient.

- [LiquidityPool.sol#L450](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L450)
- [LiquidityPool.sol#L482](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L482)
- [LiquidityPool.sol#L514](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L514)
- [LiquidityPool.sol#L546](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L546)
## [N-07] Use a more recent version of solidity
Using very old versions of Solidity prevents benefits of bug fixes and newer security checks.
## [N-08] Missing check for open trades
When you close a long trade, there is no check to see if the trade actually exists. Without the check you can be closing an empty long which will just be a waste of gas. When close a long a trade when there is none open, it will result in a panic check of underflow because solmate does not check the balance of the account before burning.

[Exchange.sol#L288-L329](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L288-L329)
```diff
L289:
        if (params.isLong) {
            uint256 shortPositions = shortToken.balanceOf(msg.sender);
            require(shortPositions == 0, "Shouldn't have short positions to close long postions");
+           require(powerPerp.balanceOf(msg.sender)>=0, "No open long position");

L
```
## [N-09] Use super instead of ERC20
To clarify that inheritance is used, it's better to make use of the super keyword.
[PowerPerp.sol#L40-L48](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/PowerPerp.sol#L40-L48)
```diff
    function transfer(address _to, uint256 _amount) public override returns (bool) {
        require(shortToken.balanceOf(_to) == 0, "Receiver has short positions");
-       return ERC20.transfer(_to, _amount);
+       return super.transfer(_to, _amount);
    }

    function transferFrom(address _from, address _to, uint256 _amount) public override returns (bool) {
        require(shortToken.balanceOf(_to) == 0, "Receiver has short positions");
-       return ERC20.transferFrom(_from, _to, _amount);
+       return super.transferFrom(_from, _to, _amount);
    }
```
## [N-10] Make variables immutable or create setters
When a variable has no setter and can't be modified it should be made immutable
- [SynthetixAdapter.sol#L10-L11](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SynthetixAdapter.sol#L10-L11)
- [SystemManager.sol#L30-L31](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/SystemManager.sol#L30-L31)
## [N-11] Timestamps don't need to be bigger than uint48
When you have a variable that uses block.timestamp, it doesn't need to be bigger than uit48. Even with 10000 blocks per second, it would be enough until year 2861.
With this knowledge some structs can maybe more optimized if you make another variable smaller so they fit in one storage slot.
- [KangarooVault.sol#L48](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L48)
- [KangarooVault.sol#L56](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L56)
- [LiquidityPool.sol#L40](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L40)
- [LiquidityPool.sol#L48](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L48)
## [N-12] Use require instead of assert when it's not something that never should be false
The following assert statements check an input parameter so it's possible that it returns false by inputting a wrong value. Statement should be require instead.
- [LiquidityPool.sol#L220](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L220)
- [LiquidityPool.sol#L285](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L285)
## [N-13] Event is missing indexed fields
When an event is missing indexed fields it can be hard for off-chain scripts to efficiently filter those events.
- [KangarooVault.sol#L808-L897](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/KangarooVault.sol#L808-L897)
- [LiquidityPool.sol#L841-L973](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L841-L973)