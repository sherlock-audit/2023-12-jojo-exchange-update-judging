Suave Mint Hare

medium

# Missing Sequencer uptime feed check in Oracles can cause unfair liquidations and other issues

## Summary
A common finding for L2 oracles - sequencer uptime feed checks are missing in all the JOJO oracle adaptors. This could cause unfair liquidations when the sequencer goes down then comes back up. Chainlink prices could also be stale and be used by malicious actors to take advantage of the sequencer downtime.

## Vulnerability Detail
OracleAdaptor.sol and OracleAdaptorWstETH.sol are both used extensively in the protocol and receive prices primarily from Chainlink. There is no check that the Arbitrum sequencer is down. 

Chainlink explains their Sequencer Uptime Feeds [here](https://docs.chain.link/data-feeds/l2-sequencer-feeds).
See [article](https://medium.com/@lopotras/l2-sequencer-and-stale-oracle-prices-bug-54a749417277) for further elaboration on bug.
## Impact
Users can get unfairly liquidated because they cannot react to price movements when the sequencer is down and when the sequencer comes back up, all price updates will immediately become available. See previous [report](https://solodit.xyz/issues/m-2-missing-sequencer-uptime-feed-check-can-cause-unfair-liquidations-on-arbitrum-sherlock-none-perennial-git) on the same issue.

Price received may also be stale and Users can get better borrows if the price is above the actual price or avoid liquidations if the price is under the actual price.

## Code Snippet
https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/oracle/OracleAdaptor.sol#L63
https://github.com/sherlock-audit/2023-12-jojo-exchange-update/blob/main/smart-contract-EVM/src/oracle/OracleAdaptorWstETH.sol#L40

## Tool used
Manual Review

## Recommendation
Implement code example from Chainlink: [https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code)