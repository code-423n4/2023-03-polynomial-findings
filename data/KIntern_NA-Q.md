# [2023-03-polynomial] QA Reports

###### tags: `c4`, `2023-03-polynomial`, `QA`
## Summary

### Low Risk Issues
| |Issue|Instances|
|-|:-|:-:|
| [L&#x2011;01] | Missing the check if `totalFunds` ≥ `usedFunds` in LiquidityPool.closeLong() | 1 | 
| [L&#x2011;02] | Function `LiquidityPool.processWithdraws()` doesn’t return when totalFunds < usedFunds | 1 | 

### None Critical Issues
| |Issue|Instances|
|-|:-|:-:|
| [NC&#x2011;01] | Missing the check if `totalFunds` ≥ `usedFunds` in LiquidityPool.closeLong() | 1 |
## Low risk issues
### [L-01] Missing the check if `totalFunds` ≥ `usedFunds` in LiquidityPool.closeLong()
* Code snippet: https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L462-L489
*  In LiquidityPool contract, `totalFunds` is total funds of contract (in SUSD), comes from depositing and profits. `usedFunds` is the funds that contract used (to margin and settle long/short).
* In function `openShort`, there is a check that `totalFunds` need to be >= `usedFunds`:
```solidity=
function openShort(uint256 amount, address user, bytes32 referralCode)
    external
    override
    onlyExchange
    nonReentrant
    returns (uint256 totalCost)
{
    ...
    usedFunds += int256(totalCost + hedgingFees + externalFee);
    require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
    ...
}
```
* However, there is no check in function `closeLong`, even this function will increase `usedFunds`. Then `usedFunds` can be greater than `totalFunds` after that
### [L-02] Function `LiquidityPool.processWithdraws()` doesn’t return when totalFunds < usedFunds
* Code snippet: https://github.com/code-423n4/2023-03-polynomial/blob/main/src/LiquidityPool.sol#L295
```solidity=
uint256 availableFunds = uint256(int256(totalFunds) - usedFunds);
if (availableFunds == 0) {
    return;
}
```
If `totalFunds` < `usedFunds`, `int256(totalFunds) - usedFunds` will be < 0, then `availableFunds` is very large. Then it will not return here, and still execute the next part of function `processWithdraws` with very large `availableFunds`.

## None Critical Issues
### [NC-01] Missing check if result of `getFundingRate()` is valid
Code snippet: https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L410
There is no check if the return of `getFundingRate` is valid in function `_updateFundingRate`: 
```solidity=
///url = https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L410
function _updateFundingRate() internal {
    (int256 fundingRate,) = getFundingRate();
    ...
```
Then Exchange contract might use the invalid fundingRate to update the `normalizationFactor` since `_updateFundingRate` is triggered during trades.
