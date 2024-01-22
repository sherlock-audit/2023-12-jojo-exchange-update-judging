Original Powder Kookaburra

medium

# Project may be unable to be deployed on Arbitrum due to incompatibility with Shanghai hardfork

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
