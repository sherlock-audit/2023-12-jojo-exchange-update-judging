Original Powder Kookaburra

high

# Adversary can DoS all users from regular withdraws

## Summary
Adversary can DoS all users from regular withdraws

## Vulnerability Detail
In `JOJODealer`, whenever users would like to withdraw, they'd have to first request a withdraw and then after `withdrawTimelock` passes, they can call `executeWithdraw`. 
```solidity
    function requestWithdraw(
        Types.State storage state,
        address from,
        uint256 primaryAmount,
        uint256 secondaryAmount
    )
        external
    {
        require(isWithdrawValid(state, msg.sender, from, primaryAmount, secondaryAmount), Errors.WITHDRAW_INVALID);
        state.pendingPrimaryWithdraw[msg.sender] = primaryAmount;
        state.pendingSecondaryWithdraw[msg.sender] = secondaryAmount;
        state.withdrawExecutionTimestamp[msg.sender] = block.timestamp + state.withdrawTimeLock;
        emit RequestWithdraw(msg.sender, primaryAmount, secondaryAmount, state.withdrawExecutionTimestamp[msg.sender]);
    }
```
In order for a withdraw request to succeed, it needs to pass the `isWithdrawValid`
```solidity
    function isWithdrawValid(
        Types.State storage state,
        address spender,
        address from,
        uint256 primaryAmount,
        uint256 secondaryAmount
    )
        internal
        view
        returns (bool)
    {
        return spender == from
            || (
                state.primaryCreditAllowed[from][spender] >= primaryAmount
                    && state.secondaryCreditAllowed[from][spender] >= secondaryAmount
            );
    }
```

The problem here is that the request allows for both `primaryAmount` and `secondaryAmount` to be exactly 0. Any user can call `requestWithdraw` for another user with both values being 0 and the check above would pass, effectively overwriting any current request. 

A malicious user could back-run any withdraw request with such 0 values and overwrite it, basically cancelling it.

## Impact
Adversary can fully DoS regular withdraws. 

## Code Snippet
https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/libraries/Funding.sol#L51C1-L67C6

## Tool used

Manual Review

## Recommendation
Upon requesting a withdraw, check that at least one of the values is non-zero. 
