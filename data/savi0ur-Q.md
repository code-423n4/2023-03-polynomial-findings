# N-01 : Description not correct for the variable `minWithdrawDelay`

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L86-L87
```solidity
/// @notice Minimum deposit delay
uint256 public minWithdrawDelay;
```

# N-02: Natspec comment for `referralCode` missing

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L427-L430
```solidity
/// @notice Called by exchange when a new long position is created
/// @param amount Amount of square perp being longed
/// @param user Address of the user //@audit-issue @param missing for referralCode
function openLong(uint256 amount, address user, bytes32 referralCode)
```

# N-03: Natspec comment for `debt` missing

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L118-L121
```solidity
/// @notice Liquidate user collateral
/// @param positionId Position ID
/// @param user Address of the liquidator //@audit-issue @param for debt missing
function liquidate(uint256 positionId, uint256 debt, address user)
```

# N-04: Remove TODO

https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L43-L46
```solidity
// TODO: Write NFT Descriptor and replace this //@audit-issue remove todo
function tokenURI(uint256 tokenId) public view override returns (string memory) {
    return "";
}
```

