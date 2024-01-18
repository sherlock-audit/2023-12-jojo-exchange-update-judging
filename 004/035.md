Original Powder Kookaburra

high

# `requestWithdraw` logic is flawed and may cause a user to simply burn their money

## Summary
`requestWithdraw` logic is flawed and may cause a user to simply burn their money

## Vulnerability Detail
Let's look at the code in `requestWithdraw`
```solidity
    function requestWithdraw(uint256 repayJUSDAmount) external returns (uint256 withdrawEarnUSDCAmount) {
        IERC20(jusd).safeTransferFrom(msg.sender, address(this), repayJUSDAmount);
        require(repayJUSDAmount <= jusdOutside[msg.sender], "Request Withdraw too big");
        jusdOutside[msg.sender] -= repayJUSDAmount;
        uint256 index = getIndex();
        uint256 lockedEarnUSDCAmount = jusdOutside[msg.sender].decimalDiv(index);
        require(
            earnUSDCBalance[msg.sender] >= lockedEarnUSDCAmount, "lockedEarnUSDCAmount is bigger than earnUSDCBalance"
        );
        withdrawEarnUSDCAmount = earnUSDCBalance[msg.sender] - lockedEarnUSDCAmount;
        withdrawalRequests.push(WithdrawalRequest(withdrawEarnUSDCAmount, msg.sender, false));
        require(
            withdrawEarnUSDCAmount.decimalMul(index) >= withdrawSettleFee, "Withdraw amount is smaller than settleFee"
        );
        earnUSDCBalance[msg.sender] = lockedEarnUSDCAmount;
        uint256 withdrawIndex = withdrawalRequests.length - 1;
        emit RequestWithdrawFromHedging(msg.sender, repayJUSDAmount, withdrawEarnUSDCAmount, withdrawIndex);
        return withdrawIndex;
    }
```

The way it calculates withdraw amount is completely flawed and needs major refactoring. 
To simply give an example: 
 - A user deposits 1000 USDC at `index == 1e18`
 - Index drops to 0.5e18 (e.g. due to borrow liquidation or bad perp trade) 
 - User now wants to claim 500 of their JUSD within the contract, so they invoke `requestWithdraw(500)`.
 - If we look at the workflow -> `jusdOutside[msg.sender] - repayJusdAmount = 500`.
 `lockedEarnUSDCAmount = 500 / 0.5 = 1000`
`WithdrawEarnUSDCAmount = 1000 - 1000 = 0` (we'll consider `withdrawSettleFee` to be 0 for simplicity. even if it is not, issue is still valid)
`EarnUSDCBalance = 500`

The user just gave 500 JUSD to the contract and burned 500 `earnUSDCBalance` and created a 0-value withdraw request. 

## Impact
Loss of funds

## Code Snippet
https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L282C1-L300C6

## Tool used

Manual Review

## Recommendation
Fix is non-trivial, function will need to be rewritten. 
