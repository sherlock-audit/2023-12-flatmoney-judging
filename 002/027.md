Swift Peach Hippo

medium

# Missing checks for whether Base Sequencer is active

## Summary
OracleModule doesn't check whether the Base Sequencer is active when using prices from chainlink oracle.


## Vulnerability Detail
Due to Chainlink [documentation](https://docs.chain.link/data-feeds/l2-sequencer-feeds) contracts that use its functionality on L2s should check the sequencer state each time requesting prices. In another case could situation when the sequencer is down but oracle continues to consume outdated prices.There is no check in _getOnchainPrice().
```solidity
function _getOnchainPrice() internal view returns (uint256 price, uint256 timestamp) {
        IChainlinkAggregatorV3 oracle = onchainOracle.oracleContract;
        if (address(oracle) == address(0)) revert FlatcoinErrors.ZeroAddress("oracle");

        (, int256 _price, , uint256 updatedAt, ) = oracle.latestRoundData();
        timestamp = updatedAt;
        // check Chainlink oracle price updated within `maxAge` time.
        if (block.timestamp > timestamp + onchainOracle.maxAge)
            revert FlatcoinErrors.PriceStale(FlatcoinErrors.PriceSource.OnChain);

        if (_price > 0) {
            price = uint256(_price) * (10 ** 10); // convert Chainlink oracle decimals 8 -> 18
        } else {
            // Issue with onchain oracle indicates a serious problem
            revert FlatcoinErrors.PriceInvalid(FlatcoinErrors.PriceSource.OnChain);
        }
    }

```

## Impact
The price may be outdated


## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L141-L157
## Tool used

Manual Review

## Recommendation
Check that sequencer is not down.


