Formal Eggshell Sidewinder

high

# Losses of some long traders can eat into the margins of others

## Summary

The losses of some long traders can eat into the margins of others, resulting in those affected long traders being unable to withdraw their margin and profits, leading to a loss of assets for the long traders.

## Vulnerability Detail

At $T0$, the current price of ETH is \$1000 and assume the following state:

| Alice's Long Position 1                                     | Bob's Long Position 2                                       | Charles (LP)     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------- |
| Position Size = 6 ETH<br />Margin = 3 ETH<br />Last Price (entry price) = \$1000 | Position Size = 6 ETH<br />Margin = 5 ETH<br />Last Price (entry price) = \$1000 | Deposited 12 ETH |

- The `stableCollateralTotal` will be 12 ETH
- The `GlobalPositions.marginDepositedTotal` will be 8 ETH (3 + 5)
- The `globalPosition.sizeOpenedTotal` will be 12 ETH (6 + 6)
- The total balance of ETH in the vault is 20 ETH. 


As this is a perfectly hedged market, the accrued fee will be zero, and ignored in this report for simplicity's sake.

At $T1$, the price of the ETH drops from \$1000 to \$600. At this point, the settle margin of both long positions will be as follows:

| Alice's Long Position 1                                     | Bob's Long Position 2                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| priceShift = Current Price - Last Price = \$600 - \$1000 = -\$400<br />PnL = (Position Size * priceShift) / Current Price = (6 ETH * -\$400) / \$400 = -4 ETH<br />settleMargin = marginDeposited + PnL = 3 ETH + (-4 ETH) = -1 ETH | PnL = -4 ETH (Same calculation)<br />settleMargin = marginDeposited + PnL = 5 ETH + (-4 ETH) = 1 ETH |

Alice's long position is underwater (settleMargin < 0), so it can be liquidated. When the liquidation is triggered, it will internally call the `updateGlobalPositionData` function. Even if the liquidation does not occur, any of the following actions will also trigger the `updateGlobalPositionData` function internally:

- executeOpen
- executeAdjust
- executeClose

The purpose of the `updateGlobalPositionData` function is to update the global position data. This includes getting the total profit loss of all long traders (Alice & Bob), and updating the margin deposited total + stable collateral total accordingly.

Assume that the `updateGlobalPositionData` function is triggered by one of the above-mentioned functions. Line 179 below will compute the total PnL of all the opened long positions.

```solidity
priceShift = current price - last price
priceShift = $600 - $1000 = -$400

profitLossTotal = (globalPosition.sizeOpenedTotal * priceShift) / current price
profitLossTotal = (12 ETH * -$400) / $600
profitLossTotal = -8 ETH
```

The `profitLossTotal` is -8 ETH. This is aligned with what we have calculated earlier, where Alice's PnL is -4 ETH and Bob's PnL is -4 ETH (total = -8 ETH loss). 

At Line 184 below, the `newMarginDepositedTotal` will be set to as follows (ignoring the `_marginDelta` for simplicity's sake)

```solidity
newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta + profitLossTotal
newMarginDepositedTotal = 8 ETH + 0 + (-8 ETH) = 0 ETH
```

What happened above is that 8 ETH collateral is deducted from the long traders and transferred to LP. When `newMarginDepositedTotal` is zero, this means that the long trader no longer owns any collateral. This is incorrect, as Bob's position should still contribute 1 ETH remaining margin to the long trader's pool.

Let's review Alice's Long Position 1: Her position's settled margin is -1 ETH. When the settled margin is -ve then the LPs have to bear the cost of loss per the comment [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L139). However, in this case, we can see that it is Bob (long trader) instead of LPs who are bearing the cost of Alice's loss, which is incorrect.

Let's review Bob's Long Position 2: His position's settled margin is 1 ETH. If his position's liquidation margin is $LM$, Bob should be able to withdraw $1\  ETH - LM$ of his position's margin. However, in this case, the `marginDepositedTotal` is already zero, so there is no more collateral left on the long trader pool for Bob to withdraw, which is incorrect.

With the current implementation, the losses of some long traders can eat into the margins of others, resulting in those affected long traders being unable to withdraw their margin and profits.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

```solidity
being File: FlatcoinVault.sol
173:     function updateGlobalPositionData(
174:         uint256 _price,
175:         int256 _marginDelta,
176:         int256 _additionalSizeDelta
177:     ) external onlyAuthorizedModule {
178:         // Get the total profit loss and update the margin deposited total.
179:         int256 profitLossTotal = PerpMath._profitLossTotal({globalPosition: _globalPositions, price: _price});
180: 
181:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
182:         // However, since the funding fees are settled at the same time as the global position data is updated,
183:         // we can ignore the funding fees here.
184:         int256 newMarginDepositedTotal = int256(_globalPositions.marginDepositedTotal) + _marginDelta + profitLossTotal;
185: 
186:         // Check that the sum of margin of all the leverage traders is not negative.
187:         // Rounding errors shouldn't result in a negative margin deposited total given that
188:         // we are rounding down the profit loss of the position.
189:         // If anything, after closing the last position in the system, the `marginDepositedTotal` should can be positive.
190:         // The margin may be negative if liquidations are not happening in a timely manner.
191:         if (newMarginDepositedTotal < 0) {
192:             revert FlatcoinErrors.InsufficientGlobalMargin();
193:         }
194: 
195:         _globalPositions = FlatcoinStructs.GlobalPositions({
196:             marginDepositedTotal: uint256(newMarginDepositedTotal),
197:             sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
198:             lastPrice: _price
199:         });
200: 
201:         // Profit loss of leverage traders has to be accounted for by adjusting the stable collateral total.
202:         // Note that technically, even the funding fees should be accounted for when computing the stable collateral total.
203:         // However, since the funding fees are settled at the same time as the global position data is updated,
204:         // we can ignore the funding fees here
205:         _updateStableCollateralTotal(-profitLossTotal);
206:     }
```

## Impact

Loss of assets for the long traders as the losses of some long traders can eat into the margins of others, resulting in those affected long traders being unable to withdraw their margin and profits.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

## Tool used

Manual Review

## Recommendation

The following are the two issues identified earlier and the recommended fixes:

**Issue 1**

> Let's review Alice's Long Position 1: Her position's settled margin is -1 ETH. When the settled margin is -ve then the LPs have to bear the cost of loss per the comment [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L139). However, in this case, we can see that it is Bob (long trader) instead of LPs who are bearing the cost of Alice's loss, which is incorrect.

Fix: Alice -1 ETH loss should be borne by the LP, not the long traders. The stable collateral total of LP should be deducted by 1 ETH to bear the cost of the loss.

**Issue 2**

> Let's review Bob's Long Position 2: His position's settled margin is 1 ETH. If his position's liquidation margin is $LM$, Bob should be able to withdraw $1\  ETH - LM$ of his position's margin. However, in this case, the `marginDepositedTotal` is already zero, so there is no more collateral left on the long trader pool for Bob to withdraw, which is incorrect.

Fix: Bob should be able to withdraw $1\  ETH - LM$ of his position's margin regardless of the PnL of other long traders. Bob's margin should be isolated from Alice's loss.