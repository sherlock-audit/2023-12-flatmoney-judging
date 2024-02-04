Formal Eggshell Sidewinder

high

# Asymmetry in profit and loss (PnL) calculations

## Summary

An asymmetry arises in profit and loss (PnL) calculations due to relative price changes. This discrepancy emerges when adjustments to a position lead to differing PnL outcomes despite equivalent absolute price shifts in rETH, leading to loss of assets.

## Vulnerability Detail

#### Scenario 1

Assume at $T0$, the price of rETH is \$1000. Bob opened a long position with the following state:

- Position Size = 40 ETH
- Margin = $x$ ETH

At $T2$, the price of rETH increased to \$2000. Thus, Bob's PnL is as follows: he gains 20 rETH.

```solidity
PnL = Position Size * Price Shift / Current Price
PnL = Position Size * (Current Price - Last Price) / Current Price
PnL = 40 rETH * ($2000 - $1000) / $2000
PnL = $40000 / $2000 = 20 rETH
```

Important Note: In terms of dollars, each ETH earns \$1000. Since the position held 40 ETH, the position gained \$40000.

#### Scenario 2

Assume at $T0$, the price of rETH is \$1000. Bob opened a long position with the following state:

- Position Size = 40 ETH
- Margin = $x$ ETH

At $T1$, the price of rETH dropped to \$500. An adjustment is executed against Bob's long position, and a `newMargin` is computed to account for the PnL accrued till now, as shown in Line 191 below. Thus, Bob's PnL is as follows: he lost 40 rETH.

```solidity
PnL = Position Size * Price Shift / Current Price
PnL = Position Size * (Current Price - Last Price) / Current Price
PnL = 40 rETH * ($500 - $1000) / $500
PnL = -$20000 / $500 = -40 rETH
```

At this point, the position's `marginDeposited` will be $(x - 40)\ rETH$ and `lastPrice` set to \$500.

Important Note 1: In terms of dollars, each ETH lost $500. Since the position held 40 ETH, the position lost \$20000

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L211

```solidity
File: LeverageModule.sol
190:         // This accounts for the profit loss and funding fees accrued till now.
191:         uint256 newMargin = (marginAdjustment +
192:             PerpMath
193:                 ._getPositionSummary({position: position, nextFundingEntry: cumulativeFunding, price: adjustPrice})
194:                 .marginAfterSettlement).toUint256();
..SNIP..
211:         vault.setPosition(
212:             FlatcoinStructs.Position({
213:                 lastPrice: adjustPrice,
214:                 marginDeposited: newMargin,
215:                 additionalSize: newAdditionalSize,
216:                 entryCumulativeFunding: cumulativeFunding
217:             }),
218:             announcedAdjust.tokenId
219:         );
```

At $T2$, the price of rETH increases from \$500 to \$2000. Thus, Bob's PnL is as follows:

```solidity
PnL = Position Size * Price Shift / Current Price
PnL = Position Size * (Current Price - Last Price) / Current Price
PnL = 40 rETH * ($2000 - $500) / $500
PnL = $60000 / $2000 = 30 rETH
```

At this point, the position's `marginDeposited` will be $(x - 40 + 30)\ rETH$, which is equal to $(x - 10)\ rETH$. This effectively means that Bob has lost 10 rETH of the total margin he deposited.

Important Note 2: In terms of dollars, each ETH gains $1500. Since the position held 40 ETH, the position gained \$60000.

Important Note 3: If we add up the loss of \$20000 at 𝑇1 and the gain of \$60000 at 𝑇2, the overall PnL is a gain of \$40000 at the end.

#### Analysis

The final PnL of a position should be equivalent regardless of the number of adjustments/position updates made between $T0$ and $T2$. However, the current implementation does not conform to this property. Bob gains 20 rETH in the first scenario, while Bob loses 10 rETH in the second scenario. 

There are several reasons that lead to this issue:

- The PnL calculation emphasizes relative price changes (percentage) rather than absolute price changes (dollar value). This leads to asymmetric rETH outcomes for the same absolute dollar gains/losses. If we have used the dollar to compute the PnL, both scenarios will return the same correct result, with a gain of $40000 at the end, as shown in the examples above. (Refer to the important note above)
- The formula for PnL calculation is sensitive to the proportion of the price change relative to the current price. This causes the rETH gains/losses to be non-linear even when the absolute dollar gains/losses are the same.

#### Extra Example

The current approach to computing the PnL will also cause issues in another area besides the one shown above. The following example aims to demonstrate that it can cause a desync between the PnL accumulated by the global positions AND the PnL of all the individual open positions in the system.

The following shows the two open positions owned by Alice and Bob. The current price of ETH is \$1000 and the current time is $T0$

| Alice's Long Position                             | Bob's Long Position                              |
| ------------------------------------------------- | ------------------------------------------------ |
| Position Size = 100 ETH<br />Entry Price = \$1000 | Position Size = 50 ETH<br />Entry Price = \$1000 |

At $T1$, the price of ETH drops from \$1000 to $750, and the `updateGlobalPositionData` function is executed. The `profitLossTotal` is computed as below. Thus, the `marginDepositedTotal` decreased by 50 ETH.

```solidity
priceShift = $750 - $1000 = -$250
profitLossTotal = (globalPosition.sizeOpenedTotal * priceShift) / price
profitLossTotal = (150 ETH * -$250) / $750 = -50 ETH
```

At $T2$, the price of ETH drops from \$750 to \$500, and the `updateGlobalPositionData` function is executed. The `profitLossTotal` is computed as below. Thus, the `marginDepositedTotal` decreased by 75 ETH.

```solidity
priceShift = $500 - $750 = -$250
profitLossTotal = (globalPosition.sizeOpenedTotal * priceShift) / price
profitLossTotal = (150 ETH * -$250) / $500 = -75 ETH
```

In total, the `marginDepositedTotal` decreased by 125 ETH (50 + 75), which means that the long traders lost 125 ETH from $T0$ to $T2$.

However, when we compute the loss of Alice and Bob's positions at $T2$, they lost a total of 150 ETH, which deviated from the loss of 125 ETH in the global position data.

```solidity
Alice's PNL
priceShift = current price - entry price = $500 - $1000 = -$500
PnL = (position size * priceShift) / current price
PnL = (100 ETH * -$500) / $500 = -100 ETH

Bob's PNL
priceShift = current price - entry price = $500 - $1000 = -$500
PnL = (position size * priceShift) / current price
PnL = (50 ETH * -$500) / $500 = -50 ETH
```

## Impact

Loss of assets, as demonstrated in the second scenario in the first example above. The tracking of profit and loss, which is the key component within the protocol, both on the position level and global level, is broken.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L211

## Tool used

Manual Review

## Recommendation

Consider tracking the PnL in dollar value/term to ensure consistency between the rETH and dollar representations of gains and losses.

#### Appendix

Compared to SNX V2, it is not vulnerable to this issue. The reason is that in SNX V2 when it computes the PnL, it does not "scale" down the result by the price. The PnL in SNXv2 is simply computed in dollar value ($positionSize \times priceShift$), while FlatCoin protocol computes in collateral (rETH) term ($\frac{positionSize \times priceShift}{price}$).

https://github.com/Synthetixio/synthetix/blob/1cfafd30deb4511cf885b4bc3cc4e9c970356800/contracts/PerpsV2MarketBase.sol#L261

```solidity
function _profitLoss(Position memory position, uint price) internal pure returns (int pnl) {
    int priceShift = int(price).sub(int(position.lastPrice));
    return int(position.size).multiplyDecimal(priceShift);
}
```

https://github.com/Synthetixio/synthetix/blob/1cfafd30deb4511cf885b4bc3cc4e9c970356800/contracts/PerpsV2MarketBase.sol#L278

```solidity
/*
 * The initial margin of a position, plus any PnL and funding it has accrued. The resulting value may be negative.
 */
function _marginPlusProfitFunding(Position memory position, uint price) internal view returns (int) {
    int funding = _accruedFunding(position, price);
    return int(position.margin).add(_profitLoss(position, price)).add(funding);
}
```