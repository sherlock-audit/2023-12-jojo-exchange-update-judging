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

**IAm0x52**

Fix looks good. To must now be a whitelisted contract

# Issue H-2: FundingRateArbitrage contract can be drained due to rounding error 

Source: https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/57 

## Found by 
detectiveking
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid because { This is valid and also a dupp of 054 due to the same underlying cause of first deposit attack; but in this the watson explained the exploit scenario of the inflation attack}



**nevillehuang**

request poc

**sherlock-admin**

PoC requested from @detectiveking123

Requests remaining: **4**

**JoscelynFarr**

I think this issue is similar to https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/54

**nevillehuang**

@JoscelynFarr Seems right, @detectiveking123 do you agree that this seems to be related to a typical first depositor inflation attack.

**detectiveking123**

@nevillehuang I am not exactly sure how this should be judged. 

The attack that I describe here chains two separate vulnerabilities together (one of which is the rounding error and the other which is the same root cause as the share inflation attack) to drain all the funds existing in the contract, which is clearly a high. **It also doesn't rely on any front-running on Arbitrum assumptions, while the other issue does. In fact, no interaction from any other users is necessary for the attacker to drain all the funds.** The exploit that is described in the other issue cannot actually drain all the funds in the contract like this one can, but simply drain user deposits if they can frontrun them. 

To clarify, the rounding error that I describe here is different from the rounding error described in the ERC4626 inflation style exploit (so I guess there are two separate rounding errors that optimally should be chained together for this exploit). 

Do you still want me to provide a code POC here? I already have an example in the issue description of how the attack can be performed. 

**nevillehuang**

@detectiveking123 Yes, please provide me a coded PoC in 1-2 days so that I can verify the draining impact, because it does share similar root causes of direct donation of funds as the first inflation attack.

**detectiveking123**

@nevillehuang let me get it to you by tomorrow

**detectiveking123**

@nevillehuang 

```
    function testExploit() public {
        jusd.mint(address(fundingRateArbitrage), 5000e6);
        // net value starts out at 0 :)
        console.log(fundingRateArbitrage.getNetValue());

        vm.startPrank(Owner); 
        fundingRateArbitrage.setMaxNetValue(10000000e6); 
        fundingRateArbitrage.setDefaultQuota(10000000e6); 
        vm.stopPrank(); 
                 
        initAlice();
        // Alice deposits twice
        fundingRateArbitrage.deposit(1);
        USDC.transfer(address(fundingRateArbitrage), 100e6);
        fundingRateArbitrage.deposit(100e6);
        vm.stopPrank();

        vm.startPrank(alice);
        fundingRateArbitrage.requestWithdraw(1);
        fundingRateArbitrage.requestWithdraw(1);
        vm.stopPrank();

        vm.startPrank(Owner); 
        uint256[] memory requestIds = new uint256[](2);
        requestIds[0] = 0; 
        requestIds[1] = 1;
        fundingRateArbitrage.permitWithdrawRequests(requestIds); 
        vm.stopPrank(); 

        // Alice is back to her initial balance, but now has a bunch of extra JUSD deposited for her into jojodealer!
        console.log(USDC.balanceOf(alice));
        (,uint secondaryCredit,,,) = jojoDealer.getCreditOf(alice);
        console.log(secondaryCredit);
    }
```

Add this to `FundingRateArbitrageTest.t.sol`

You will also need to add:

```
    function transfer(address to, uint256 amount) public override returns (bool) {
        address owner = _msgSender();
        _transfer(owner, to, amount);
        return true;
    }
```

to TestERC20

And change initAlice to:

```
    function initAlice() public {
        USDC.mint(alice, 300e6 + 1);
        jusd.mint(alice, 300e6 + 1);
        vm.startPrank(alice);
        USDC.approve(address(fundingRateArbitrage), 300e6 + 1);
        jusd.approve(address(fundingRateArbitrage), 300e6 + 1); 
    }
```
**FYI for this exploit the share inflation is helpful but not necessary**. The main issue is the rounding down of `lockedEarnUSDCAmount` in `requestWithdraw`. Even if the share price is 1 cent for example, we will slowly be able to drain JUSD from the contract. An assumption for profitability is that the share price is nontrivial though (so if it's really small it won't be profitable for the attacker b/c of gas fees and deposit fees, though you can still technically drain). 



**nevillehuang**

This issue is exactly the same as #21 and the **original** submission shares the same root cause of depositor inflation to make the attack feasible, given share price realistically won't be of such a low price. I will be duplicating accordingly. Given and subsequent deposits can be drained, I will be upgrading to high severity

@detectiveking123 If you want to escalate feel free,I will maintain my stance here.

**IAm0x52**

Same fix as #54

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

**JoscelynFarr**

Fixed PR: https://github.com/JOJOexchange/smart-contract-EVM/commit/4b591c9fb0a232f784919905752c8e68d32b39ff

**IAm0x52**

Fix looks good. Math has been updated to use full precision

# Issue M-2: Funding#requestWithdraw uses incorrect withdraw address 

Source: https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/53 

## Found by 
0x52, FastTiger, OrderSol, Varun\_05, bughuntoor, dany.armstrong90
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



**detectiveking123**

@nevillehuang do you believe this has enough impact to be considered valid?

**JoscelynFarr**

Fixed PR: https://github.com/JOJOexchange/smart-contract-EVM/commit/82b5c85c9999ace00265382e7d1bc83036685069

**IAm0x52**

Fix looks good. Now uses from instead of msg.sender

**nevillehuang**

@detectiveking123 @JoscelynFarr https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/30#issuecomment-1927840325

# Issue M-3: FundRateArbitrage is vulnerable to inflation attacks 

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



**detectiveking123**

Escalate

I am not completely sure about the judgement here and am therefore escalating to get @Czar102 's opinion on how this should be judged.

I believe that #56 and #21 should be treated as different issues than this one. I am not even sure if this issue and other duplicates are valid, as they rely on the front-running on Arbitrum assumption, which has not been explicitly confirmed to be valid or invalid on Sherlock. 

Please take a look at the thread on #56 to better understand the differences. But the TLDR is:

1. This issue requires Arbitrum frontrunning to work, the other one in #56 doesn't
2. The one in #56 takes advantage of a separate rounding error as well to fully drain funds inside the contract


**sherlock-admin**

> Escalate
> 
> I am not completely sure about the judgement here and am therefore escalating to get @Czar102 's opinion on how this should be judged.
> 
> I believe that #56 and #21 should be treated as different issues than this one. I am not even sure if this issue and other duplicates are valid, as they rely on the front-running on Arbitrum assumption, which has not been explicitly confirmed to be valid or invalid on Sherlock. 
> 
> Please take a look at the thread on #56 to better understand the differences. But the TLDR is:
> 
> 1. This issue requires Arbitrum frontrunning to work, the other one in #56 doesn't
> 2. The one in #56 takes advantage of a separate rounding error as well to fully drain funds inside the contract
> 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**JoscelynFarr**

Fixed PR: https://github.com/JOJOexchange/smart-contract-EVM/commit/b3cf3d6d6b761059f814efb84af53c5ead4f6446

**nevillehuang**

@detectiveking123 

- I think both this issue and your issue implies that the user needs to be the first depositor, especially the scenario highlighted. This does not explicitly requires a front-run, given a meticulous user can make their own EV calculations.

This issue:

> Assume the user deposits 1 share then donates 100,000e6 USDC. The exchange ratio is now 100,000e18 which causes issues during deposits.

Your issue:
> The execution of this is a bit more complicated, let's go through an example. We will assume there's a bunch of JUSD existing in the contract and the attacker is the first to deposit.

- I think one of the watsons in the discord channel highlighted a valid design of the protocol to mitigate this issue, where withdrawals must be explicitly requested. Because of this, this could possibly be medium severity.

**giraffe0x**

Disagree that the request/permit design prevents this. 

As described in code comments for `requestWithdraw()`: "The main purpose of this function is to capture the interest and avoid the DOS attacks". It is unlikely that withdraw requests are individually scrutinized and manually permitted by owner but automatically executed by bots in batches. Even if the owner does monitor each request, it would be tricky to spot dishonest deposits/withdrawals. 

It is better to implement native contract defence a classic ERC4626 attack. Should be kept as a high finding. 

**nevillehuang**

@giraffe0x Again you are speculating on off-chain mechanisms. While it is a valid concern, the focus should be on contract level code logic, and sherlocks assumption is that admin will make the right decisions when permitting withdrawal requests.

Also, here is the most recent example of where first depositor inflation is rated as medium:

https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/55

**detectiveking123**

@nevillehuang 

"I think both this issue and your issue implies that the user needs to be the first depositor, especially the scenario highlighted. This does not explicitly requires a front-run, given a meticulous user can make their own EV calculations."

How would you run this attack without front-running? Share inflation attacks explicitly require front-running

**nevillehuang**

@detectiveking123 I agree that the only possible reason for this issue to be valid is if 

- It so happens that a depositor is the first depositor, and he made his own EV calculations and realized so based on the shares obtained and current exchange ratios (slim chance of happening given front-running is a non-issue on arbitrum, but not impossible)
- If the attack does not explicitly require a first depositor inflation attack (to my knowledge not possible), which I believe none of the **original** issues showed a scenario/explanation it is so (Even #21 and #57 is highlighting a first depositor scenario)

If both of the above scenario does not apply, all of the issues and its duplicates should be low severity.

**IAm0x52**

Fix looks good. Adds a virtual offset which prevents this issue.

**Evert0x**

Front-running isn't necessary as the attacker can deposit 1 wei + donate and just wait for someone to make a deposit under 100,000e6.

But this way the attack is still risky to execute as the attacker will lose the donated USDC in case he isn't the first depositor. 

However, this can be mitigated if the attacker created a contract the does the 1 wei deposit + donate action in a single transaction BUT revert in case it isn't the first deposit in the protocol.

Planning to reject escalation and keep issue state as is.  

**detectiveking123**

@Evert0x not sure that makes sense, if you just wait for someone then they should just not deposit (it's a user mistake to deposit, they should be informed that they'll retrieve no shares back in the UI).


**Czar102**

After a discussion with @Evert0x, planning to make it a Medium severity issue – frontend can display information for users not to fall victim to this exploit by displaying a number of output shares.
Even though frontrunning a tx can't be done easily (there is no mempool), one can obtain information about a transaction being submitted in another way. Since this puts severe constraints on this being exploitable, planning to consider it a medium severity issue.

**detectiveking123**

@Czar102 My issue that has been duplicated with this (https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/57) drains the entire contract (clearly a high) and requires no front-running. The purpose of the escalation was primarily to request deduplication. 

**Czar102**

Planning to consider #57 a separate High severity issue. #57 linked this finding together with another bug (ability to withdraw 1 wei of vault token in value for free) to construct a more severe exploit.

Also, planning to consider this issue a Medium severity one, as mentioned above.

**deadrosesxyz**

With all due respect, this is against the rules

> Issues identifying a core vulnerability can be considered duplicates.
Scenario A:
There is a root cause/error/vulnerability A in the code. This vulnerability A -> leads to two attack paths:
- B -> high severity path
- C -> medium severity attack path/just identifying the vulnerability.
Both B & C would not have been possible if error A did not exist in the first place. In this case, both B & C should be put together as duplicates.

**detectiveking123**

It's worth noting that you can still drain the contract with the exploit described in #57 and #21, even without share inflation (The main issue is a rounding issue that allows you to get one more share than intended, so your profit will be the current share value). If the share price is trivial though, the exploiter will likely lose money to gas fees while draining. 

This is why I said I'm not sure about the judging of this issue in the initial escalation, as it seems rather subjective. 

**Czar102**

@deadrosesxyz There are two different vulnerabilities requiring two different fixes. In the fragment of the docs you quoted, all B and C are a result of a single vulnerability A, which is not the case here.

**Czar102**

Planning to make #57 a separate high, I don't see how is #21 presenting the same vulnerability. This issue and duplicates (including #21) will be considered a Medium.

**IAm0x52**

Why exactly would it be high and this one medium? Both rely on being first depositor (not frontrunning) and inflation

Edit: As stated in the other submission it is technically possible outside of first depositor but profit would be marginal and wouldn't cover gas costs. IMO hard to even call it a different exploit when both have the same prerequisites and same basic attack structure (first depositor and inflation).

**Czar102**

#57 presents a way to withdraw equivalent of 1 wei of the vault token for free. This issue and duplicates present a way to inflate the share price being the first depositor, and in case of the frontend displaying all needed information, one needs to frontrun a deposit transaction to execute the attack.

**Czar102**

Result:
Medium
Has duplicates

**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [detectiveking123](https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/54/#issuecomment-1913270709): accepted

