Strong Daffodil Capybara

medium

# Unhandled return values of `transfer` and `transferFrom` functions

## Summary
The behavior of ERC20 implementations can sometimes be unpredictable. In some instances, the `transfer` and `transferFrom` functions may return 'false' to signal a failure instead of following the standard protocol of reverting. To enhance safety and reliability, it is recommended to encapsulate these calls within `require()` statements to properly handle any potential failures. 

## Vulnerability Detail
The vulnerability in the ERC20 implementation relates to the unsafe usage of `transfer` and `transferFrom` functions in the smart contract code.

## Impact
This vulnerability can lead to unpredictable behavior and may result in funds being lost or transactions not executing as expected. In some cases, these functions may return 'false' to signal a failure instead of reverting as per the standard ERC20 protocol. This non-standard behavior can have adverse consequences for users and the contract itself.

## Code Snippet

Unsafe `transfer` and `transferFrom` calls have been identified in specific locations, warranting extra caution. Here is where I found it:


https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L263


https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L266


https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L313


https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L315

## Tool used

Manual Review

## Recommendation
Check the return value and revert on `0`/`false` or use OpenZeppelin’s `SafeERC20` wrapper functions.