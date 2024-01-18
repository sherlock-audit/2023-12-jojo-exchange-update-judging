Breezy Cinnabar Troll

medium

# `JUSDBankStorage::getTRate()`,`JUSDBankStorage::accrueRate()` are calculated differently, and the data calculation is biased, Causes the `JUSDBank` contract funciton result to be incorrect

## Summary
```js
    function accrueRate() public {
        uint256 currentTimestamp = block.timestamp;
        if (currentTimestamp == lastUpdateTimestamp) {
            return;
        }
        uint256 timeDifference = block.timestamp - uint256(lastUpdateTimestamp);
@>         tRate = tRate.decimalMul((timeDifference * borrowFeeRate) / Types.SECONDS_PER_YEAR + 1e18);
        lastUpdateTimestamp = currentTimestamp;
    }

    function getTRate() public view returns (uint256) {
        uint256 timeDifference = block.timestamp - uint256(lastUpdateTimestamp);
@>        return tRate + (borrowFeeRate * timeDifference) / Types.SECONDS_PER_YEAR;
    }
```
`JUSDBankStorage::getTRate()`,`JUSDBankStorage::accrueRate()` are calculated differently, and the data calculation is biased, resulting in the JUSDBank contract not being executed correctly
## Vulnerability Detail
The wrong result causes the funciton calculation results of `JUSDBank::_isAccountSafe()`, `JUSDBank::flashLoan()`, `JUSDBank::_handleBadDebt`, etc. to be biased,and all functions that call the relevant function will be biased
## Impact
Causes the `JUSDBank` contract funciton result to be incorrect
## Code Snippet

https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/JUSDBankStorage.sol#L53-L67

## Tool used

Manual Review
## POC
Please add the test code to `JUSDViewTest.t.sol` for execution
```js
    function testTRateDeviation() public {
        console.log(block.timestamp);
        console.log(jusdBank.lastUpdateTimestamp());
        vm.warp(block.timestamp + 18_356 days);
        jusdBank.accrueRate();
        console.log("tRate value than 2e18:", jusdBank.tRate());
        // block.timestamp for every 1 increment
        vm.warp(block.timestamp + 1);
        uint256 getTRateNum = jusdBank.getTRate();
        jusdBank.accrueRate();
        uint256 tRateNum = jusdBank.tRate();
        console.log("block.timestamp for every 1 increment, deviation:", tRateNum - getTRateNum);
        // block.timestamp for every 1 days increment
        vm.warp(block.timestamp + 1 days);
        getTRateNum = jusdBank.getTRate();
        jusdBank.accrueRate();
        tRateNum = jusdBank.tRate();
        console.log("block.timestamp for every 1 days increment, deviation:", tRateNum - getTRateNum);
    }
```
## Recommendation
Use the same calculation formula:
```diff
    function accrueRate() public {
        uint256 currentTimestamp = block.timestamp;
        if (currentTimestamp == lastUpdateTimestamp) {
            return;
        }
        uint256 timeDifference = block.timestamp - uint256(lastUpdateTimestamp);
         tRate = tRate.decimalMul((timeDifference * borrowFeeRate) / Types.SECONDS_PER_YEAR + 1e18);
        lastUpdateTimestamp = currentTimestamp;
    }

    function getTRate() public view returns (uint256) {
        uint256 timeDifference = block.timestamp - uint256(lastUpdateTimestamp);
-       return tRate + (borrowFeeRate * timeDifference) / Types.SECONDS_PER_YEAR;
+       return  tRate.decimalMul((timeDifference * borrowFeeRate) / Types.SECONDS_PER_YEAR + 1e18);
    }
```

