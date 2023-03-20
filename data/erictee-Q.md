
### [L-01] ```require()``` should be used instead of ```assert()```


#### Impact
Prior to solidity version 0.8.0, hitting an assert consumes the remainder of the transaction’s available gas rather than returning it, as ```require()```/```revert()``` do. ```assert()``` should be avoided even past solidity version 0.8.0 as its [documentation](https://docs.soliditylang.org/en/v0.8.14/control-structures.html#panic-via-assert-and-error-via-require) states that “The assert function creates an error of type Panic(uint256). … Properly functioning code should never create a Panic, not even on invalid external input. If this happens, then there is a bug in your contract which you should fix”.


#### Findings:
```
src/LiquidityPool.sol:L220        assert(queuedDepositHead + count - 1 < nextQueuedDepositId);

src/LiquidityPool.sol:L285        assert(queuedWithdrawalHead + count - 1 < nextQueuedWithdrawalId);

```


### [L-02] Avoid floating pragmas for non-library contracts.


#### Impact
While floating pragmas make sense for libraries to allow them to be included with multiple different versions of applications, it may be a security risk for application implementations.

A known vulnerable compiler version may accidentally be selected or security tools might fall-back to an older compiler version ending up checking a different EVM compilation that is ultimately deployed on the blockchain.

It is recommended to pin to a concrete compiler version.

#### Findings:
```
src/PowerPerp.sol:L2     pragma solidity ^0.8.9;

src/ShortToken.sol:L2     pragma solidity ^0.8.9;

src/KangarooVault.sol:L2     pragma solidity ^0.8.9;

src/Exchange.sol:L2     pragma solidity ^0.8.9;

src/SynthetixAdapter.sol:L2     pragma solidity ^0.8.9;

src/ShortCollateral.sol:L2     pragma solidity ^0.8.9;

src/LiquidityToken.sol:L2     pragma solidity ^0.8.9;

src/LiquidityPool.sol:L2     pragma solidity ^0.8.9;

src/SystemManager.sol:L2     pragma solidity ^0.8.9;

src/utils/PauseModifier.sol:L3     pragma solidity ^0.8.9;
```



### [L-03] ```_safemint()``` should be used rather than ```_mint()``` wherever possible


#### Impact
```_mint()``` is [discouraged](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d4d8d2ed9798cc3383912a23b5e8d5cb602f7d4b/contracts/token/ERC721/ERC721.sol#L271) in favor of ```_safeMint()``` which ensures that the recipient is either an EOA or implements ```IERC721Receiver```. Both [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d4d8d2ed9798cc3383912a23b5e8d5cb602f7d4b/contracts/token/ERC721/ERC721.sol#L238-L250) and [solmate](https://github.com/transmissions11/solmate/blob/4eaf6b68202e36f67cab379768ac6be304c8ebde/src/tokens/ERC721.sol#L180) have versions of this function


#### Findings:
```
src/ShortToken.sol:L67            _mint(trader, positionId);
```


### [N-01] Incomplete Natspec

#### Findings:
File: [src/LiquidityPool.sol#L427-L430](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L427-L430)

```
/// @notice Called by exchange when a new long position is created

/// @param amount Amount of square perp being longed

/// @param user Address of the user

function openLong(uint256 amount, address user, bytes32 referralCode)
```
Missing `@param referralCode`.
