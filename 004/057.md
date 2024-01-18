Main Emerald Salmon

high

# FundingRateArbitrage contract can be drained due to rounding error

## Summary

In the `requestWithdraw`, rounding in the wrong direction is done which can lead to contract being drained. 

## Vulnerability Detail

In the `requestWithdraw` function in `FundingRateArbitrage`, we find the following lines of code:

```solidity
jusdOutside[msg.sender] -= repayJUSDAmount;
uint256 index = getIndex();
uint256 lockedEarnUSDCAmount = jusdOutside[msg.sender].decimalDiv(index);
require(
     earnUSDCBalance[msg.sender] >= lockedEarnUSDCAmount, "lockedEarnUSDCAmount is bigger than earnUSDCBalance"
);
withdrawEarnUSDCAmount = earnUSDCBalance[msg.sender] - lockedEarnUSDCAmount;
```

Because we round down when calculating `lockedEarnUSDCAmount`, `withdrawEarnUSDCAmount` is higher than it should be, which leads to us allowing the user to withdraw more than we should allow them to given the amount of JUSD they repaid. 

The execution of this is a bit more complicated, let's go through an example. We will assume there's a bunch of JUSD existing in the contract and the attacker is the first to deposit. 

Steps:

1. The attacker deposits 1 unit of USDC and then manually sends in another 100 * 10^6 - 1 (not through deposit, just a transfer). The share price / price per earnUSDC will now be $100. Exactly one earnUSDC is in existence at the moment. 
2. Next the attacker creates a new EOA and deposits a little over $101 worth of USDC (so that after fees we can get to the $100), giving one earnUSDC to the EOA. The attacker will receive around $100 worth of JUSD from doing this. 
3. Attacker calls `requestWithdraw` with `repayJUSDAmount = 1` with the second newly created EOA
4. `lockedEarnUSDCAmount` is rounded down to 0 (since `repayJUSDAmount` is subtracted from jusdOutside[msg.sender]
5. `withdrawEarnUSDCAmount` will be `1`
6. After `permitWithdrawRequests` is called, attacker will be able to withdraw the $100 they deposited through the second EOA (granted, they lost the deposit and withdrawal fees) while only having sent `1` unit of `JUSD` back. This leads to massive profit for the attacker. 

Attacker can repeat steps 2-6 constantly until the contract is drained of JUSD. 
 
## Impact

All JUSD in the contract can be drained

## Code Snippet

https://github.com/JOJOexchange/smart-contract-EVM/blob/main/src/FundingRateArbitrage.sol#L283-L300

## Tool used

Manual Review

## Recommendation

Round up instead of down 
