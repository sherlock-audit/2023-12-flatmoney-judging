Flaky Myrtle Porcupine

medium

# Incorrect KeeperFee

## Summary
Keeper fee uses the wrong heartbeat when reading a chainlink oracle.
## Vulnerability Detail
The `KeeperFee.sol` is the contract that calculates the `keeperFee` for the order execution. It uses the price of Ethereum to determine the execution cost based on a few different variables that are queried from the `_gasPriceOracle`.

The issue is that the `_STALENESS_PERIOD` is set as constant (wrong one). It is set as `1 day` whereas the heartbeat for the `ETH/USD` chainlink feed is recommended to be `1200s` on the `BASE network`.

This can be seen here in the official docs  
https://docs.chain.link/data-feeds/price-feeds/addresses?network=base&page=1&search=eth%2F

This can lead to the use of a stale price for the keeper fee calculation.
## Impact
The use of a stale price for `keeperFee` calculation can lead to a scenario where keepers execute orders based on a stale price. Because of that, the users could pay less for fees if the price goes up but this is not yet reflected on chain and keepers get less collateral for their service. It also goes the other way around if the price of Ethereum goes down and is not yet updated.
## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/KeeperFee.sol#L29

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/KeeperFee.sol#L95

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/KeeperFee.sol#L109-L111
## Tool used
Manual Review

## Recommendation
Use the correct heartbeat of 1200s.