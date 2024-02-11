Exotic Goldenrod Duck

high

# Protocol won't be able to get rETH/USD price from OracleModule

## Summary
Protocol won't be able to get rETH/USD price from OracleModule because rETH/USD is supported by Chainlink on Base Chain.

## Vulnerability Detail
FlatMoney uses rETH as collateral and the price will be retrieved from Pyth Network Oracle, as said in the Audit Page:
> **Are there any off-chain mechanisms or off-chain procedures for the protocol (keeper bots, input validation expectations, etc)?**

> The protocol uses Pyth Network collateral (rETH) price feed. This is an offchain price that is pulled by the keeper and pushed onchain at time of any order execution.

While Pyth is used as **Offchain Oracle**, Chainlink Price Feed is used as **Onchain oracle**, when [get price](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L86) from **OracleModule**, protocol will check the **diffPercent** between the 2 oracles and the transaction will revert if **diffPercent** is larger than **maxDiffPercent**:
```solidity
        uint256 priceDiff = (int256(onchainPrice) - int256(offchainPrice)).abs();
        uint256 diffPercent = (priceDiff * 1e18) / onchainPrice;
        if (diffPercent > maxDiffPercent) revert FlatcoinErrors.PriceMismatch(diffPercent);
```

Because it only makes sense to compare the price of the same collateral token, as rETH/USD price is retrieved from Pyth Oracle, the Chainlink Price Feed should also return rETH/USD. Unfortunately, rETH/USD is not supported by Chainlink on Base Chain, so there is no way to get rETH/USD price. 

Protocol may choose an alternative Chainlink Price Feed, for example, ETH/USD, however, the price difference between ETH and rETH can be significant (at the time of wrting, [RETH / ETH](https://www.coingecko.com/en/coins/rocket-pool-eth/eth) is `1.1`), results in **diffPercent** being much larger than a rational **maxDiffPercent** and transaction will always revert, protocol won't be able to get price from OracleModule and this renders the whole protocol useless.

## Impact
Protocol won't be able to get price from **OracleModule**, and protocol becomes useless.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L106

## Tool used
Manual Review

## Recommendation
Protocol should combine 2 Chainlink Price Feeds ([ETH/USD](0x71041dddad3595F9CEd3DcCFBe3D1F4b0a16Bb70) and [rETH/ETH](0xf397bF97280B488cA19ee3093E81C0a77F02e9a5)) to get the rETH/USD price data on Base Chain.