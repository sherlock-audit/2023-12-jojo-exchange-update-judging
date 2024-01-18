Suave Mint Hare

medium

# Unsafe casting of Int256 perpNetValue leads to DOS

## Summary
In FundingRateArbitrage.sol, the function `getNetValue()` does `SafeCast.toUint256(perpNetValue)`. When `perpNetValue` is negative, this cast will revert.

## Vulnerability Detail
```solidity
function getNetValue() public view returns (uint256) {
        ...
        (int256 perpNetValue,,,) = JOJODealer(jojoDealer).getTraderRisk(address(this));
        
        //@audit when perpNetValue is negative Safecast will revert
        return
            SafeCast.toUint256(perpNetValue) + collateralAmount.decimalMul(collateralPrice) + usdcBuffer - jusdBorrowed;
    }
```

`perpNetValue` can be a negative value when the trades (by contract Owner) are losing money (confirmed with Sponsor). This will result in SafeCast reverting:
```solidity
function toUint256(int256 value) internal pure returns (uint256) {
        require(value >= 0, "SafeCast: value must be positive");
        return uint256(value);
    }
```

## Impact
All deposit and withdraw functions in FundingRateArbitrage.sol are DOS-ed as they rely on `getNetValue()` and will revert when called. The impact is significant because users are likely to want to withdraw if the trades are losing money, but are unable to do so due to the bug described. 

## Code Snippet
https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L94

## Tool used
Manual Review

## Recommendation
Add additional if/else conditions in deposit and withdraw to handle a negative perpNetValue scenario and possible overall negative netValue scenario. 