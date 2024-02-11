Bright Butter Mongoose

medium

# ```OracleModule::_getOnchainPrice``` has no check for round completeness

## Summary
The ```OracleModule::_getOnchainPrice``` utilizes Chainlink's ```latestRoundData()``` to fetch the latest price data. However, it does not check for the completeness of the round, potentially leading to the use of stale or incorrect price data.

## Vulnerability Detail
The ```getOnchainPrice``` call out to an oracle with ``latestRoundData()``` to get the price of asset. Although the returned timestamp is checked, there is no check for round completeness.

According to Chainlink's documentation (ref. https://docs.chain.link/docs/historical-price-data/#historical-rounds), this function does not error if no answer has been reached but returns 0 or outdated round data. The external Chainlink oracle, which provides index price information to the system, introduces risk inherent to any dependency on third-party data sources. 

In summary:
- The latestRoundData() function may return data from an incomplete round, which has not been confirmed by the oracle network.
- The contract only checks the timestamp of the price update, not the round's completeness.
- There is no validation to ensure that the price is from the most recent and agreed-upon round.

## Impact
- The use of stale or incorrect price data can lead to inaccurate calculations within the smart contract.
- Financial decisions based on this data, such as liquidations, may be incorrect, potentially causing fund loss or unfair liquidations.
- The integrity and reliability of the contract's financial mechanisms are compromised, which could lead to a loss of user trust and financial damages.

## Code Snippet
Link to the code snippet: https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L141-L157

```solidity
function _getOnchainPrice() internal view returns (uint256 price, uint256 timestamp) {
        IChainlinkAggregatorV3 oracle = onchainOracle.oracleContract;
        if (address(oracle) == address(0)) revert FlatcoinErrors.ZeroAddress("oracle");

@>        (, int256 _price, , uint256 updatedAt, ) = oracle.latestRoundData();
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

## Tool used

Manual Review

## Recommendation
Modify the ```_getOnchainPrice``` function to include a check for round completeness by verifying that the ```answeredInRound``` value is greater than or equal to the ```roundID```.

```diff
function _getOnchainPrice() internal view returns (uint256 price, uint256 timestamp) {
        IChainlinkAggregatorV3 oracle = onchainOracle.oracleContract;
        if (address(oracle) == address(0)) revert FlatcoinErrors.ZeroAddress("oracle");

-       (, int256 _price, , uint256 updatedAt, ) = oracle.latestRoundData();
+      (uint80 roundID, int256 _price, , uint256 updatedAt, uint80 answeredInRound) = oracle.latestRoundData();

+     if (answeredInRound < roundID) {
+        revert FlatcoinErrors.RoundIncomplete(FlatcoinErrors.PriceSource.OnChain);
 +   }

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

