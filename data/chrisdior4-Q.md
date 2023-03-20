# QA report

## Low Risk
| L-N    |Issue|
|:------:|:----|
| [L&#x2011;01] | Open TODO in the code | 9 |
| [L&#x2011;02] | Unsafe call to ERC20::transfer can result in stuck funds | 1 |
| [L&#x2011;03] | Missing zero address checks | 2 |

Total: 3 issues

## Non-critical

| N-N    |Issue|
|:------:|:----|
| [N&#x2011;01] | Missing friendly revert strings | 2 |
| [N&#x2011;02] | Typos | 6 |
| [N&#x2011;03] | Incomplete NatSpec | 11 |
| [N&#x2011;04] | Unused variable, import and event | 19 |
| [N&#x2011;05] | Consider using custom errors instead of require statements with string error | 31 |
| [N&#x2011;06] | Solidity safe pragma best practices are not used | 2 |
| [N&#x2011;07] | Events and structs declaration | 1 |
| [N&#x2011;08] | Wrong comment in `LiquidityPool.sol` | 1 |
| [N&#x2011;09] | Use the delete keyword instead of assigning the default value of variables | 1 |

Total: 9 issues

## Low Risk

### [L-01] Open TODO in the code

There is an open TODO in `ShortToken.sol` this implies changes that might not be audited. Resolve it or remove it.

```solidity
// TODO: Write NFT Descriptor and replace this
```

Lines of code: 

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/ShortToken.sol#L43


### [L-02] Unsafe call to ERC20::transfer can result in stuck funds

File: `KangarooVault.sol`

In the `saveToken` method we have the following code for transferring ERC20 tokens:

```solidity
function saveToken(address token, address receiver, uint256 amt) external requiresAuth {
  require(token != address(SUSD));
  ERC20(token).transfer(receiver, amt);
    }
```

The problem is that the transfer function from ERC20 returns a bool to indicate if the transfer was a success or not. As there are some tokens that do not revert on failure but instead return false (one such example is ZRX) and also Polynomial should work with all types of ERC20 tokens since it might be integrated with a protocol that does that, not checking the return value can result in tokens getting stuck. 

If an ERC20::transfer call fails it will lead to stuck funds for a user. This only happens with a special class of ERC20 tokens though.

Lines of code: 

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L549

Use OpenZeppelin’s SafeERC20 library and change transfer to safeTransfer

### [L-03] Missing zero address check 

File: `LiquidityPool.sol`

In `deposit` assets are sent to `user` but his address is not checked. This way there is a risk for newly minted liquidity tokens to stuck forever in 0 address.

```solidity
 function deposit(uint256 amount, address user) external override nonReentrant whenNotPaused("POOL_DEPOSIT") {
 .....
 liquidityToken.mint(user, tokensToMint);
```
Lines of code:

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L184

Add zero address check for `user` parameter in  `deposit()`.

There are a lot of other instances in the codebase that are missing 0 address checks, consider adding them.

## Non-critical

### [NC-01]  Missing friendly revert strings
  Here, a friendly message should exist for users to understand what went wrong:

File: `Exchange.sol`

```solidity
require(totalCost <= maxCost);
````

```solidity
require(params.collateral == shortPosition.collateral);
```

```solidity
require(totalCost >= minCost);
```

```solidity
require(shortToken.ownerOf(params.positionId) == msg.sender);
```

```solidity
require(shortPosition.collateral == params.collateral);
```

```solidity
require(maxDebtRepayment > 0);
```

```solidity
require(positionId != 0);
```

```solidity
require(!isInvalid);

require(shortToken.ownerOf(positionId) == msg.sender);
```


### [NC-02] Typos

`postions` -> `positions`

`Exchnage` -> `Exchange`

`Receipient` -> `Recipient` 

`depositted` -> `deposited`

`Perfomance` -> `Performance`


### [NC-03] Incomplete NatSpec
Contracts are missing @param and @return fields in a lot of public and external functions.  NatSpec documentation to all public methods and variables is essential for better understanding of the code by developers and auditors and is strongly recommended. Consider adding a complete NatSpec.



### [NC-04] Unused variable, import and event
File: `Exchange.sol`

```solidity
ERC20 public SUSD;
```

`SUSD` variable is not used, so it can be removed.

File: `LiquidityPool.sol` 

```solidity
import {IFuturesMarket} from "./interfaces/synthetix/IFuturesMarket.sol";
```

remove the import because it is not used.

File: `LiquidityPool.sol` 

```solidity
event UpdateFeeReceipient(address oldFeeReceipient, address newFeeReceipient);
```

Lines of code:

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/Exchange.sol#L50
- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L18
- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L944



### [NC-05] Consider using custom errors instead of require statements with string error

The project's contracts are using require statements with string error. However custom errors reduce the contract size and can provide easier integration with a protocol. Consider using those instead of require statements with string error.

### [NC-06] Solidity safe pragma best practices are not used

Always use a stable pragma to be certain that you deterministically compile the Solidity code to the same bytecode every time. All the contracts are currently using floatable 0.8.9 version.


### [NC-07] Events and structs declaration

Files: `Exchange.sol` and `LiquidityPool.sol`

It is a best practice to be declared in the interface not in the implementation contract.


### [NC-08] Wrong comment in `LiquidityPool.sol` 

```solidity
/// @notice Minimum deposit delay
    uint256 public minWithdrawDelay;
```

Change the comment like that:

```solidity
/// @notice Minimum withdraw delay
    uint256 public minWithdrawDelay;
```


also there is a commented section called "immutables" but not all variables in this section are immutable, either remove the section comment or move out the variables that are not immutable out of this section

```solidity
/// -----------------------------------------------------------------------
    /// Immutables
    /// -----------------------------------------------------------------------

    /// @notice The asset being traded in this market
    bytes32 public immutable baseAsset;

    /// @notice Power Perp Instance
    IPowerPerp public powerPerp;

    /// @notice Power Perp Exchange Instance
    IExchange public exchange;

    /// @notice SUSD Instance
    ERC20 public immutable SUSD;

    /// @notice Liquidity Token Instance
    ILiquidityToken public liquidityToken;
````



### [NC-09] Use the delete keyword instead of assigning the default value of variables

File: `LiquidityPool.sol`

In processDeposits method the `current.depositedAmount` is reset by assigning it to 0. It is a best practice to just use the `delete` keyword instead, there is no need to manually assign the type's default values.

```solidity
current.depositedAmount = 0;
```

Lines of code:

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L239