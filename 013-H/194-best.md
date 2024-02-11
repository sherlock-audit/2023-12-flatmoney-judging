Formal Eggshell Sidewinder

high

# Malicious keepers can manipulate the price when executing an order

## Summary

Malicious keepers can manipulate the price when executing an order by selecting a price in favor of either the LPs or long traders, leading to a loss of assets to the victim's party.

## Vulnerability Detail

When the keeper executes an order, it was understood from the protocol team that the protocol expects that the keeper must also update the Pyth price to the latest one available off-chain. In addition, the [contest page](https://github.com/sherlock-audit/2023-12-flatmoney-xiaoming9090?tab=readme-ov-file#q-are-there-any-off-chain-mechanisms-or-off-chain-procedures-for-the-protocol-keeper-bots-input-validation-expectations-etc) mentioned that "an offchain price that is pulled by the keeper and pushed onchain at time of any order execution".

This requirement must be enforced to ensure that the latest price is always used.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L386

```solidity
File: DelayedOrder.sol
378:     function executeOrder(
379:         address account,
380:         bytes[] calldata priceUpdateData
381:     )
382:         external
383:         payable
384:         nonReentrant
385:         whenNotPaused
386:         updatePythPrice(vault, msg.sender, priceUpdateData)
387:         orderInvariantChecks(vault)
388:     {
389:         // Settle funding fees before executing any order.
390:         // This is to avoid error related to max caps or max skew reached when the market has been skewed to one side for a long time.
391:         // This is more important in case the we allow for limit orders in the future.
392:         vault.settleFundingFees();
..SNIP..
410:     }
```

However, this requirement can be bypassed by malicious keepers. A keeper could skip or avoid the updating of the Pyth price by passing in an empty `priceUpdateData` array, which will pass the empty array to the `OracleModule.updatePythPrice` function.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/abstracts/OracleModifiers.sol#L12

```solidity
File: OracleModifiers.sol
10:     /// @dev Important to use this modifier in functions which require the Pyth network price to be updated.
11:     ///      Otherwise, the invariant checks or any other logic which depends on the Pyth network price may not be correct.
12:     modifier updatePythPrice(
13:         IFlatcoinVault vault,
14:         address sender,
15:         bytes[] calldata priceUpdateData
16:     ) {
17:         IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).updatePythPrice{value: msg.value}(
18:             sender,
19:             priceUpdateData
20:         );
21:         _;
22:     }
```

When the Pyth's `Pyth.updatePriceFeeds` function is executed, the `updateData` parameter will be set to an empty array.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L64

```solidity
File: OracleModule.sol
64:     function updatePythPrice(address sender, bytes[] calldata priceUpdateData) external payable nonReentrant {
65:         // Get fee amount to pay to Pyth
66:         uint256 fee = offchainOracle.oracleContract.getUpdateFee(priceUpdateData);
67: 
68:         // Update the price data (and pay the fee)
69:         offchainOracle.oracleContract.updatePriceFeeds{value: fee}(priceUpdateData);
70: 
71:         if (msg.value - fee > 0) {
72:             // Need to refund caller. Try to return unused value, or revert if failed
73:             (bool success, ) = sender.call{value: msg.value - fee}("");
74:             if (success == false) revert FlatcoinErrors.RefundFailed();
75:         }
76:     }
```

Inspecting the source code of Pyth's on-chain contract, the [`Pyth.updatePriceFeeds`](https://goerli.basescan.org/address/0xf5bbe9558f4bf37f1eb82fb2cedb1c775fa56832#code#F24#L75) function will not perform any update since the `updateData.length` will be zero in this instance.

https://goerli.basescan.org/address/0xf5bbe9558f4bf37f1eb82fb2cedb1c775fa56832#code#F24#L75

```solidity
function updatePriceFeeds(
    bytes[] calldata updateData
) public payable override {
    uint totalNumUpdates = 0;
    for (uint i = 0; i < updateData.length; ) {
        if (
            updateData[i].length > 4 &&
            UnsafeCalldataBytesLib.toUint32(updateData[i], 0) ==
            ACCUMULATOR_MAGIC
        ) {
            totalNumUpdates += updatePriceInfosFromAccumulatorUpdate(
                updateData[i]
            );
        } else {
            updatePriceBatchFromVm(updateData[i]);
            totalNumUpdates += 1;
        }

        unchecked {
            i++;
        }
    }
    uint requiredFee = getTotalFee(totalNumUpdates);
    if (msg.value < requiredFee) revert PythErrors.InsufficientFee();
}
```

The keeper is permissionless, thus anyone can be a keeper and execute order on the protocol. If this requirement is not enforced, keepers who might also be LPs (or collude with LPs) can choose whether to update the Pyth price to the latest price or not, depending on whether the updated price is in favor of the LPs. For instance, if the existing on-chain price (\$1000 per ETH) is higher than the latest off-chain price (\$950 per ETH), malicious keepers will use the higher price of \$1000 to open the trader's long position so that its position's entry price will be set to a higher price of \$1000. When the latest price of \$950 gets updated, the longer position will immediately incur a loss of \$50. Since this is a zero-sum game, long traders' loss is LPs' gain.

Note that per the current design, when the open long position order is executed at $T2$, any price data with a timestamp between $T1$ and $T2$ is considered valid and can be used within the `executeOpen` function to execute an open order. Thus, when the malicious keeper uses an up-to-date price stored in Pyth's on-chain contract, it will not revert as long as its timestamp is on or after $T1$.

![image](https://github.com/sherlock-audit/2023-12-flatmoney-xiaoming9090/assets/102820284/8ca41703-96c6-4cfe-bfc5-161f59ecaddf)

Alternatively, it is also possible for the opposite scenario to happen where the keepers favor the long traders and choose to use a lower older price on-chain to execute the order instead of using the latest higher price. As such, the long trader's position will be immediately profitable after the price update. In this case, the LPs are on the losing end.

Sidenote: The oracle's `maxDiffPercent` check will not guard against this attack effectively. For instance, in the above example, if the Chainlink price is \$975 and the `maxDiffPercent` is 5%, the Pyth price of \$950 or \$1000 still falls within the acceptable range. If the `maxDiffPercent` is reduced to a smaller margin, it will potentially lead to a more serious issue where all the transactions get reverted when fetching the price, breaking the entire protocol.

## Impact

Loss of assets as shown in the scenario above.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L386

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/abstracts/OracleModifiers.sol#L12

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L64

## Tool used

Manual Review

## Recommendation

Ensure that the keepers must update the Pyth price when executing an order. Perform additional checks against the `priceUpdateData` submitted by the keepers to ensure that it is not empty and `priceId` within the `PriceInfo` matches the price ID of the collateral (rETH), so as to prevent malicious keeper from bypassing the price update by passing in an empty array or price update data that is not mapped to the collateral (rETH).