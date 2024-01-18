Hot Infrared Starling

high

# Liquidator can extract additional value from liquidations by selling more collateral than necessary

## Summary

`JUSDBank#liquidate` allows the liquidator to take more collateral than necessary as long as they return the equivalent value at the end of the call. The issue is that oracle prices and spot prices frequently diverge, even if only slightly. The liquidator can abuse this and liquidate much more collateral than necessary to profit off this difference.

## Vulnerability Detail

[JUSDBank.sol#L194-L199](https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/JUSDBank.sol#L194-L199)

    require(
        IERC20(primaryAsset).balanceOf(address(this)) - primaryLiquidatedAmount
            >= liquidateData.liquidatedRemainUSDC,
        Errors.LIQUIDATED_AMOUNT_NOT_ENOUGH
    );
    IERC20(primaryAsset).safeTransfer(liquidated, liquidateData.liquidatedRemainUSDC);

When a user is liquidated, up to the entire amount of collateral can be taken from the user even if it's more than is necessary. To make up for this, the USDC value (as determined by an oracle) must be returned for all excess collateral. The issue with this is that spot prices (usually AMM) frequently diverge from oracle prices. This allows a malicious liquidator to extract excess value from a user by selling all their collateral. Take the following example:

1. User deposits 200 ETH and takes out a loan for 100,000
2. After some time the price of ETH drops and the loan is now under margin
3. Current oracle price is 2000 but spot price is 2020
4. Assuming a 10% discount the liquidator is allowed to take:
    100,000 (2000 * 0.9) = 55.55 ETH
5. The liquidator should profit 10,000 for liquidation
6. However they can profit further by selling all collateral
7. Liquidator liquidates all 200 ETH (144.45 extra) which can be sold at 2020
8. They are only obligated to repay 144.45 * 2000 and gets to pocket the other 2889

## Impact

Excess collateral can be maliciously sold for profit at expense of user being liquidated

## Code Snippet

[JUSDBank.sol#L148-L209](https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/JUSDBank.sol#L148-L209)

## Tool used

Manual Review

## Recommendation

Only allow the required amount to be sold
