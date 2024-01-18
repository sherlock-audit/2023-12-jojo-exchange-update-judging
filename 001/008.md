Glamorous Metal Albatross

high

# Users pay additional interest due to the way interest accrues

## Summary
In it's calculation protocol uses compound formula of calculating `tRate`.
How protocol tracks interest of borrowed JUSD? It writes `t0Amount`, `t0Amount = actualAmount / tRate`. Then tRate increases and on the moment of repay debt is calculated as `t0Amount * tRate`.

Suppose current situation:
1) Day 1 of protocol alive, `tRate = 1`, `borrowFeeRate = 0.2` which means 20% a year
2) User borrows 100 JUSD, his `t0Amount` is `100 / 1 = 100`
3) Year passes, now `accrueRate()` is called. `tRate = 1 * (0.2 + 1) = 1.2`. It means that now hi owes `100 * 1.2 = 120 JUSD`, accrued interest is 20 JUSD.

But now instead consider that `accrueRate()` was called 40 times with equal gaps during a year. In this case `tRate = (1 + 0.2 / 40) ^ 40 = 1.2207`, accrued interest is 22 JUSD which is 10% more interest than in previous example: 0.22 vs 0.2

Problem is that current implementation depends on frequency of calling `accrueRate()`

## Vulnerability Details
Here you can see the way formula is implemented. Note that `accrueRate()` is supposed to be called very frequently: on every borrow, repay and liquidate. 
```solidity
    function accrueRate() public {
        uint256 currentTimestamp = block.timestamp;
        if (currentTimestamp == lastUpdateTimestamp) {
            return;
        }
        uint256 timeDifference = block.timestamp - uint256(lastUpdateTimestamp);
        tRate = tRate.decimalMul((timeDifference * borrowFeeRate) / Types.SECONDS_PER_YEAR + 1e18);
        lastUpdateTimestamp = currentTimestamp;
    }
```

## Impact
User pays more interest than intended, for example extra 10% in above scenario

## Code Snippet
https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/ed4a8483da11bcc04ced10de899038bcead087b3/smart-contract-EVM/src/JUSDBankStorage.sol#L53-L66

## Tool used

Manual Review

## Recommendation
Used formula to accrue interest must be independent of update frequency. For example you can use Taylor expansion to calculate compound interest