# Issue H-1: All funds can be stolen from JOJODealer 

Source: https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/7 

## Found by 
0x52, 0xhashiman, T1MOH, bughuntoor, cawfree, giraffe, vvv
## Summary
`Funding._withdraw()` makes arbitrary call with user specified params. User can for example make ERC20 to himself and steal funds. 

## Vulnerability Detail
User can specify parameters `param` and `to` when withdraws:
```solidity
    function executeWithdraw(address from, address to, bool isInternal, bytes memory param) external nonReentrant {
        Funding.executeWithdraw(state, from, to, isInternal, param);
    }
```

In the end of `_withdraw()` function address `to` is called with that `bytes param`:
```solidity
    function _withdraw(
        Types.State storage state,
        address spender,
        address from,
        address to,
        uint256 primaryAmount,
        uint256 secondaryAmount,
        bool isInternal,
        bytes memory param
    )
        private
    {
        ...

        if (param.length != 0) {
@>          require(Address.isContract(to), "target is not a contract");
            (bool success,) = to.call(param);
            if (success == false) {
                assembly {
                    let ptr := mload(0x40)
                    let size := returndatasize()
                    returndatacopy(ptr, 0, size)
                    revert(ptr, size)
                }
            }
        }
    }
```

As an attack vector attacker can execute withdrawal of 1 wei to USDC contract and pass calldata to transfer arbitrary USDC amount to himself via USDC contract.

## Impact
All funds can be stolen from JOJODealer

## Code Snippet
https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/ed4a8483da11bcc04ced10de899038bcead087b3/smart-contract-EVM/src/libraries/Funding.sol#L173-L184

## Tool used

Manual Review

## Recommendation
Don't make arbitrary call with user specified params



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid because { This is valid and i can validate it with POC from report 076}



**JoscelynFarr**

Fixed PR: https://github.com/JOJOexchange/smart-contract-EVM/commit/763de53a36243490ef46a2c702c5a1480554f286

# Issue H-2: FundRateArbitrage is vulnerable to inflation attacks 

Source: https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/54 

## Found by 
0x52, Ignite, bughuntoor, detectiveking, giraffe, rvierdiiev
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid because { valid as watson demostrated how this implementation will lead to an inflation attack of the ERC4626 but its medium due to the possibility of it is very low and that front-tun in arbitrum is very unlikely }



# Issue M-1: `JUSDBankStorage::getTRate()`,`JUSDBankStorage::accrueRate()` are calculated differently, and the data calculation is biased, Causes the `JUSDBank` contract funciton result to be incorrect 

Source: https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/1 

## Found by 
FastTiger, T1MOH, bitsurfer, dany.armstrong90, detectiveking, joicygiore, rvierdiiev
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




## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid because { This is also valid and a dupp of 016}



**JoscelynFarr**

After internal discussion, we decide to accrue rate in the view which is `getTRate()` function

# Issue M-2: Project may be unable to be deployed on Arbitrum due to incompatibility with Shanghai hardfork 

Source: https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/32 

## Found by 
bareli, bughuntoor
## Summary
Project might be unusable upon deployment due to not supporting PUSH0 OPCODE 

## Vulnerability Detail
The project uses unsafe pragma (^0.8.20) which by default uses PUSH0 OPCODE. However, Arbitrum currently does not [support](https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/solidity-support#differences-from-solidity-on-ethereum) it. 

This means that the produced bytecode for the different contracts won't be compatible with Arbitrum as it does not yet support the Shanghai hard fork.


## Impact
Unusable contracts, will need redeploy


## Code Snippet
https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/JOJOExternal.sol#L6

## Tool used

Manual Review

## Recommendation
change pragma to 0.8.19 or change the EVM version 



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid because {invalid}



**nevillehuang**

@JoscelynFarr just to double confirm, in the foundry configuration `foundry.toml` you guys didn't set the evm_version to paris during the time of contest , so this is valid correct?

**JoscelynFarr**

yes, we did not set the evm_version to paris, so it is valid, and we will update the pragma to 0.8.19

# Issue M-3: Funding#requestWithdraw uses incorrect withdraw address 

Source: https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/53 

## Found by 
0x52, FastTiger, Varun\_05, dany.armstrong90
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid because { This is valid and a dupp of 082 with minimal impact}



# Issue M-4: Unsafe casting of Int256 perpNetValue leads to DOS 

Source: https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/65 

## Found by 
giraffe
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid because { It will not revert as its in int256 which is to hold negative values}



