Hot Infrared Starling

high

# FundRateArbitrage is vulnerable to inflation attacks

## Summary

When index is calculated, it is figured by dividing the net value of the contract (including USDC held) by the current supply of earnUSDC. Through deposit and donation this ratio can be inflated. Then when others deposit, their deposit can be taken almost completely via rounding.

## Vulnerability Detail

[FundingRateArbitrage.sol#L98-L104](https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L98-L104)

    function getIndex() public view returns (uint256) {
        if (totalEarnUSDCBalance == 0) {
            return 1e18;
        } else {
            return SignedDecimalMath.decimalDiv(getNetValue(), totalEarnUSDCBalance);
        }
    }

Index is calculated is by dividing the net value of the contract (including USDC held) by the current supply of totalEarnUSDCBalance. This can be inflated via donation. Assume the user deposits 1 share then donates 100,000e6 USDC. The exchange ratio is now 100,000e18 which causes issues during deposits.

[FundingRateArbitrage.sol#L258-L275](https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L258-L275)

    function deposit(uint256 amount) external {
        require(amount != 0, "deposit amount is zero");
        uint256 feeAmount = amount.decimalMul(depositFeeRate);
        if (feeAmount > 0) {
            amount -= feeAmount;
            IERC20(usdc).transferFrom(msg.sender, owner(), feeAmount);
        }
        uint256 earnUSDCAmount = amount.decimalDiv(getIndex());
        IERC20(usdc).transferFrom(msg.sender, address(this), amount);
        JOJODealer(jojoDealer).deposit(0, amount, msg.sender);
        earnUSDCBalance[msg.sender] += earnUSDCAmount;
        jusdOutside[msg.sender] += amount;
        totalEarnUSDCBalance += earnUSDCAmount;
        require(getNetValue() <= maxNetValue, "net value exceed limitation");
        uint256 quota = maxUsdcQuota[msg.sender] == 0 ? defaultUsdcQuota : maxUsdcQuota[msg.sender];
        require(earnUSDCBalance[msg.sender].decimalMul(getIndex()) <= quota, "usdc amount bigger than quota");
        emit DepositToHedging(msg.sender, amount, feeAmount, earnUSDCAmount);
    }

Notice earnUSDCAmount is amount / index. With the inflated index that would mean that any deposit under 100,000e6 will get zero shares, making it exactly like the standard ERC4626 inflation attack.

## Impact

Subsequent user deposits can be stolen

## Code Snippet

[FundingRateArbitrage.sol#L258-L275](https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/FundingRateArbitrage.sol#L258-L275)

## Tool used

Manual Review

## Recommendation

Use a virtual offset as suggested by [OZ](https://docs.openzeppelin.com/contracts/4.x/erc4626) for their ERC4626 contracts