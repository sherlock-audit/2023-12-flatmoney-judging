Immense Cloud Pig

high

# wrong assumption in OracleModule._getOnchainPrice() causes overvalueing of rETH/ETH price

## Summary
price returned from call to oracle.latestRoundData() is multiplied by `(10 ** 10)` because  it is assumed to be in 1e8.
## Vulnerability Detail
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
            price = uint256(_price) * (10 ** 10); // convert Chainlink oracle decimals 8 -> 18 //@audit-issue
        } else {
            // Issue with onchain oracle indicates a serious problem
            revert FlatcoinErrors.PriceInvalid(FlatcoinErrors.PriceSource.OnChain);
        }
    }
```

The assumption is made that rETH/ETH price feed is returned in 8 decimals and is therefore multiplied by 1e10 to scale it up to 18 decimals . 

/ETH price feeds are returned in 18 decimals so this is wrong.  /USD ones are returned in 8 decimals.
rETH/ETH is an ETH price feed so its in 18 decimals.
## Impact
This means the price returned from OracleModule._getOnchainPrice() is overscaled making it have 28 decimals instead of 18 decimals. 

this causes an overvaluing of the rETH/ETH 
## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L141C5-L157C6
## Tool used

Manual Review

## Recommendation
don't multiply  the price by  `(10 ** 10)` here since its already in 18 decimals
```solidity
if (_price > 0) {
            price = uint256(_price) * (10 ** 10); // convert Chainlink oracle decimals 8 -> 18
        } 
```

