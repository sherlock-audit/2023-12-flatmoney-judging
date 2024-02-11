Fresh Seafoam Narwhal

medium

# Chainlink Oracle will return the wrong price for asset if underlying aggregator hits minAnswer

## Summary
Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band. The result is that if an asset experiences a huge drop in value (i.e. LUNA crash) the price of the oracle will continue to return the minPrice instead of the actual price of the asset. This would allow user to continue borrowing with the asset but at the wrong price. This is exactly what happened to [Venus on BSC when LUNA imploded](https://rekt.news/venus-blizz-rekt/).

## Vulnerability Detail
OracleModule#_getOnchainPrice pulls the associated aggregator and requests round data from it. ChainlinkAggregators have minPrice and maxPrice circuit breakers built into them. This means that if the price of the asset drops below the minPrice, the protocol will continue to value the token at minPrice instead of it's actual value. This will allow users to take out huge amounts of bad debt and bankrupt the protocol.

Example: TokenA has a minPrice of $1. The price of TokenA drops to $0.10. The aggregator still returns $1 which is 10x it's actual value.

Note: Chainlink oracles are used a just one piece of the OracleAggregator system and it is assumed that using a combination of other oracles, a scenario like this can be avoided. However this is not the case because the other oracles also have their flaws that can still allow this to be exploited. As an example if the chainlink oracle is being used with a UniswapV3Oracle which uses a long TWAP then this will be exploitable when the TWAP is near the minPrice on the way down. In a scenario like that it wouldn't matter what the third oracle was because it would be bypassed with the two matching oracles prices. If secondary oracles like Band are used a malicious user could DDOS relayers to prevent update pricing. Once the price becomes stale the chainlink oracle would be the only oracle left and it's price would be used.

## Impact
`_getOnchainPrice` will return wrong price

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L145

## Tool used

Manual Review

## Recommendation
use min/max circuit breaker