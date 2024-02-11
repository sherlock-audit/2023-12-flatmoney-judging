Formal Eggshell Sidewinder

medium

# Assumption that newest timestamp equate to fresher price

## Summary

The assumption that the newest timestamp equates to a fresher price does not hold for all scenarios. The price used for various core activities (e.g., liquidation, open/close position) within the protocol will not be as accurate as expected. As such, some values (PnL, margin) will either be slightly overvalued or undervalued.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L106

```solidity
File: OracleModule.sol
102:     /// @notice Returns the latest 18 decimal price of asset from either Pyth.network or Chainlink.
103:     /// @dev It verifies the Pyth network price against Chainlink price (ensure that it is within a threshold).
104:     /// @return price The latest 18 decimal price of asset.
105:     /// @return timestamp The timestamp of the latest price.
106:     function _getPrice(uint32 maxAge) internal view returns (uint256 price, uint256 timestamp) {
107:         (uint256 onchainPrice, uint256 onchainTime) = _getOnchainPrice(); // will revert if invalid
108:         (uint256 offchainPrice, uint256 offchainTime, bool offchainInvalid) = _getOffchainPrice();
109:         bool offchain;
110: 
111:         uint256 priceDiff = (int256(onchainPrice) - int256(offchainPrice)).abs();
112:         uint256 diffPercent = (priceDiff * 1e18) / onchainPrice;
113:         if (diffPercent > maxDiffPercent) revert FlatcoinErrors.PriceMismatch(diffPercent);
114: 
115:         if (offchainInvalid == false) {
116:             // return the freshest price
117:             if (offchainTime >= onchainTime) {
118:                 price = offchainPrice;
119:                 timestamp = offchainTime;
120:                 offchain = true;
121:             } else {
122:                 price = onchainPrice;
123:                 timestamp = onchainTime;
124:             }
125:         } else {
126:             price = onchainPrice;
127:             timestamp = onchainTime;
128:         }
129: 
130:         // Check that the timestamp is within the required age
131:         if (maxAge < type(uint32).max && timestamp + maxAge < block.timestamp) {
132:             revert FlatcoinErrors.PriceStale(
133:                 offchain ? FlatcoinErrors.PriceSource.OffChain : FlatcoinErrors.PriceSource.OnChain
134:             );
135:         }
136:     }
```

Per the code and comment on Lines 116-117 above, it assumes that the price with the latest timestamp is the freshest price that reflects the price closest to the real-time market price. However, this assumption does not hold for all scenarios.

The Chainlink oracle used within Flatcoin protocol is not the new Low-latency Chainlink Data Streams being used. It uses the traditional Chainlink's Data Feed, which is a geographically distributed network of nodes that take time to reach consensus and finalize updates, thus having a higher latency for the market price to be reflected on the on-chain data feed.

The Pyth oracle is similar to Chainlink's Data Streams, as they both adopt a pull-based design and support low-latency data. Thus, the market price gets reflected on-chain faster. As such, even if the timestamp of Pyth's price is older than the Chainlink's price, it might be fresher than the Chainlink price in some cases.

This will be especially true when both of the timestamps are close to each other, and the Chainlink's timestamp is a few seconds (e.g., 1-3s) newer than Pyth's timestamp. In this case, it is likely that the older Pyth's price will be the one that is more closely aligned with the market price.

## Impact

The price used for various core activities (e.g., liquidation, open/close position) within the protocol will not be as accurate as expected. As such, some values (PnL, margin) will either be slightly overvalued or undervalued.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L106

## Tool used

Manual Review

## Recommendation

The Chainlink Oracle used is a traditional oracle with higher latency, while the Pyth oracle used is a low-latency oracle. Thus, they are not actually comparable in a sense. 

If the intention is to select the latest of both oracles, both oracles should be of a similar type and be a low-latency oracles. In this case, the Pyth oracle should match with the newer [Chainlink's Data Streams](https://docs.chain.link/data-streams) that also provide low-latency delivery of market data. Thus, consider switching the Chainlink's Price Feed to Chainlink's Data Streams in the future once they are publicly available.