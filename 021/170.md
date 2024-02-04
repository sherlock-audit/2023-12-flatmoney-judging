Orbiting Cyan Finch

medium

# In executeOrder, OracleModule.getPrice(maxAge) may revert because maxAge is too small

## Summary

[[executeOrder](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L378-L381)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L378-L381) will call different functions to process the order according to the type of order, and these functions will call `OracleModule.getPrice(maxAge)` to get the price. `maxAge` is equal to the current `block.timestamp - order.executableAtTime`. If `maxAge` is too small (for example, 0-3), then `OracleModule.getPrice(maxAge)` may revert [[here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L132)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L132) even though the price is fresh.

## Vulnerability Detail

Below we use the `LeverageAdjust` type Order to describe this issue.

```data
minExecutabilityAge is 5s  
maxExecutabilityAge is 60s
```

1.  Alice deposits margin on her position A via `DelayedOrder.announceLeverageAdjust`. Because if the price of the collateral continues to dump, position A will be liquidated. A pending `LeverageAdjust` order is created. Assume `block.timestamp = 1707000000`, so `order.executableAtTime = block.timestamp + minExecutabilityAge = 1707000005`. The expiration time of this order is `order.executableAtTime + maxExecutabilityAge = 1707000065`.
2.  The order has `keeperFee` paid to the keeper, so the keepers will compete with each other as long as there is an order that can be executed. One keeper monitored the `FlatcoinEvents.OrderAnnounced` event, it requested the API to obtain `priceUpdateData`. Assume that the fresh price's `publishTime` is 1707000003.
3.  After minExecutabilityAge seconds (block.timestamp=1707000005), the keeper executes the order via `DelayedOrder.executeOrder`. The `priceUpdateData` argument is obtained in step 2. Eventually tx will revert.

Let’s analyze the reasons for revert. The call stack of [[DelayedOrder.executeOrder](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L378-L381)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L378-L381) is as follows:

```flow
DelayedOrder.executeOrder
  //This modifer will write priceUpdatedata to pyth oracle. Note: priceUpdatedata is from step 2
  updatePythPrice(vault, msg.sender, priceUpdateData)
    vault.settleFundingFees()
    _executeLeverageAdjust(account)
      //here checking whether order can be executed
      _prepareExecutionOrder(account, order.executableAtTime)
      LeverageModule.executeAdjust
L152    uint32 maxAge = _getMaxAge(_order.executableAtTime);
L159    (uint256 adjustPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice({
            maxAge: maxAge
        });
        ......
      ......
    ......
  ......
```

L152, `maxAge = block.timestamp - _order.executableAtTime = 1707000005 - 1707000005 = 0`

L159, the `maxAge` argument passed into `OracleModule.getPrice` is 0.

```solidity
File: flatcoin-v1\src\OracleModule.sol
094:     function getPrice(uint32 maxAge) public view returns (uint256 price, uint256 timestamp) {
095:->       (price, timestamp) = _getPrice(maxAge);
096:     }

106:     function _getPrice(uint32 maxAge) internal view returns (uint256 price, uint256 timestamp) {
107:->       (uint256 onchainPrice, uint256 onchainTime) = _getOnchainPrice(); // will revert if invalid
108:->       (uint256 offchainPrice, uint256 offchainTime, bool offchainInvalid) = _getOffchainPrice();
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
131:->       if (maxAge < type(uint32).max && timestamp + maxAge < block.timestamp) {
132:             revert FlatcoinErrors.PriceStale(
133:                 offchain ? FlatcoinErrors.PriceSource.OffChain : FlatcoinErrors.PriceSource.OnChain
134:             );
135:         }
136:     }
```

L107, onchainTime = `updatedAt` returned by `oracle.latestRoundData()` from chainlink. `updatedAt` depends on the symbol's heartBeat. The heartbeats of almost chainlink price feeds are based on hours (such as 24 hours, 1 hour, etc.). Therefore, `onchainTime` is always lagging time, and the probability of being equal to `block.timestamp` is low. 

L108, `offchainTime = publishTime of priceUpdateData = 1707000003`

L117-124, `timestamp = max(onchainTime, offchainTime)`, in most cases `offchainTime` is larger.

L131, in this case `maxAge=0`,

```data
maxAge < type(uint32).max && timestamp + maxAge < block.timestamp =>
0 < type(uint32).max && 1707000003 + 0 < 1707000005 =>
0 < type(uint32).max && 1707000003 < 1707000005
```

So if statement is met, tx revert.

## Impact

- Pending orders cannot be executed in time. For time-sensitive protocol, this is unacceptable.
- The keeper obviously submitted the fresh price, but the tx failed with `FlatcoinErrors.PriceStale`.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L94-L96

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L159-L161

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L270-L272

## Tool used

Manual Review

## Recommendation

The cases in the report need to be considered.

```fix
File: flatcoin-v1\src\LeverageModule.sol
449:     function _getMaxAge(uint64 _executableAtTime) internal view returns (uint32 _maxAge) {
450:-        return (block.timestamp - _executableAtTime).toUint32();
450:+        return (block.timestamp - _executableAtTime + vault.minExecutabilityAge()).toUint32();
451:     }
```