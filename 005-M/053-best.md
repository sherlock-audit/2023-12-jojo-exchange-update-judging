Hot Infrared Starling

medium

# Funding#requestWithdraw uses incorrect withdraw address

## Summary

When requesting a withdraw, `msg.sender` is used in place of the `from` address. This means that withdraws cannot be initiated on behalf of other users. This will break integrations that depend on this functionality leading to irretrievable funds.

## Vulnerability Detail

[Funding.sol#L69-L82](https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/libraries/Funding.sol#L69-L82)

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

As shown above the withdraw is accidentally queue to `msg.sender` NOT the `from` address. This means that all withdraws started on behalf of another user will actually trigger a withdraw from the `operator`. The result is that withdraw cannot be initiated on behalf of other users, even if the allowance is set properly, leading to irretrievable funds

## Impact

Requesting withdraws for other users is broken and strands funds

## Code Snippet

[Funding.sol#L69-L82](https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/libraries/Funding.sol#L69-L82)

## Tool used

Manual Review

## Recommendation

Change all occurrences of `msg.sender` in stage changes to `from` instead.