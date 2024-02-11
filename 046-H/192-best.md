Formal Eggshell Sidewinder

high

# Liquidation will result in an underflow revert

## Summary

Liquidation will result in an underflow revert. Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation cannot be executed due to the revert described in the above scenario, underwater positions and bad debt will accumulate in the protocol, threatening the solvency of the protocol.

In addition, the proper function of the protocol relies on the correct accounting of the collateral in the vault and collateral owned by long traders and LPs. If the accounting is off, the vault will be broken.

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

Alice's long position is underwater (settleMargin < 0), so it can be liquidated. 

Since the liquidated position's settledMargin is less than 0, the code at Line 142 below will be executed.

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal((3 ETH + (-4 ETH)) - (-4 ETH)); // This effectively remove the PnL component from the equation
vault.updateStableCollateralTotal(3 ETH);
```

After the `updateStableCollateralTotal` function is executed, the `stableCollateralTotal` will become 15 ETH (12 + 3), the `GlobalPositions.marginDepositedTotal` will remain at 8 ETH, and the total balance of ETH in the vault will remain at 20 ETH.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
109:         if (settledMargin > 0) {
..SNIP..
138:         } else {
139:             // If the settled margin is -ve then the LPs have to bear the cost.
140:             // Adjust the stable collateral total to account for user's profit/loss and the negative margin.
141:             // Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
142:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
143:         }
..SNIP..
159:         vault.updateGlobalPositionData({
160:             price: position.lastPrice,
161:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
162:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
163:         });
```

Subsequently, the `vault.updateGlobalPositionData` will be executed. The `marginDelta` will be set to -3 ETH, as shown below:

```solidity
marginDelta = -(position.marginDeposited + positionSummary.accruedFunding)
marginDelta = -(3 ETH + 0)
marginDelta = -3 ETH
```

Line 179 below within the `updateGlobalPositionData` function will compute the total PnL of all the opened long positions.

```solidity
priceShift = current price - last price
priceShift = $600 - $1000 = -$400

profitLossTotal = (globalPosition.sizeOpenedTotal * priceShift) / current price
profitLossTotal = (12 ETH * -$400) / $600
profitLossTotal = -8 ETH
```

At Line 184 below, the `newMarginDepositedTotal` will be set to as follows:

```solidity
newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta + profitLossTotal
newMarginDepositedTotal = 8 ETH + (-3 ETH) + (-8 ETH) = -3 ETH
```

As `newMarginDepositedTotal` is less than zero, the code at Line 192 will trigger a revert, causing the liquidation TX to revert.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

```solidity
File: FlatcoinVault.sol
168:     /// @notice Function to update the global position data.
169:     /// @dev This function is only callable by the authorized modules.
170:     /// @param _price The current price of the underlying asset.
171:     /// @param _marginDelta The change in the margin deposited total.
172:     /// @param _additionalSizeDelta The change in the size opened total.
173:     function updateGlobalPositionData(
174:         uint256 _price,
175:         int256 _marginDelta,
176:         int256 _additionalSizeDelta
177:     ) external onlyAuthorizedModule {
178:         // Get the total profit loss and update the margin deposited total.
179:         int256 profitLossTotal = PerpMath._profitLossTotal({globalPosition: _globalPositions, price: _price});
180: 
181:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
182:         // However, since the funding fees are settled at the same time as the global position data is updated,
183:         // we can ignore the funding fees here.
184:         int256 newMarginDepositedTotal = int256(_globalPositions.marginDepositedTotal) + _marginDelta + profitLossTotal;
185: 
186:         // Check that the sum of margin of all the leverage traders is not negative.
187:         // Rounding errors shouldn't result in a negative margin deposited total given that
188:         // we are rounding down the profit loss of the position.
189:         // If anything, after closing the last position in the system, the `marginDepositedTotal` should can be positive.
190:         // The margin may be negative if liquidations are not happening in a timely manner.
191:         if (newMarginDepositedTotal < 0) {
192:             revert FlatcoinErrors.InsufficientGlobalMargin();
193:         }
194: 
195:         _globalPositions = FlatcoinStructs.GlobalPositions({
196:             marginDepositedTotal: uint256(newMarginDepositedTotal),
197:             sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
198:             lastPrice: _price
199:         });
200: 
201:         // Profit loss of leverage traders has to be accounted for by adjusting the stable collateral total.
202:         // Note that technically, even the funding fees should be accounted for when computing the stable collateral total.
203:         // However, since the funding fees are settled at the same time as the global position data is updated,
204:         // we can ignore the funding fees here
205:         _updateStableCollateralTotal(-profitLossTotal);
206:     }
```

Let's assume that there is no revert for the sake of verifying the correctness of the accounting used in the later portion of the code within the liquidation function.

In this case, the latest `marginDepositedTotal` will be set to -3 ETH. 

Next, the `_updateStableCollateralTotal(-profitLossTotal);` at Line 205 above will be executed, and the `stableCollateralTotal` will be set to 23 ETH.

```solidity
stableCollateralTotal = stableCollateralTotal + (-profitLossTotal)
stableCollateralTotal = 15 ETH + (-(-8 ETH))
stableCollateralTotal = 15 ETH + (8 ETH)
stableCollateralTotal = 23 ETH
```

This shows that accounting is incorrect, as it is not possible for the LPs to own 23 ETH when there are only 20 ETH balance as collateral in the vault.

In conclusion, there are two (2) issues identified here:

1. Liquidation cannot be carried out due to revert
2. Even if there is no revert, the accounting of collateral is off.

## Impact

Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation cannot be executed due to the revert described in the above scenario, underwater positions and bad debt will accumulate in the protocol, threatening the solvency of the protocol.

In addition, the proper function of the protocol relies on the correct accounting of the collateral in the vault and collateral owned by long traders and LPs. If the accounting is off, the vault will be broken.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

## Tool used

Manual Review

## Recommendation

In the above scenario, to prevent an underflow revert when computing the new `newMarginDepositedTotal` and fixing the incorrect balance issue, the `profitLossTotal` should be excluded within the `updateGlobalPositionData` function during liquidation. 

```diff
- profitLossTotal = PerpMath._profitLossTotal(...)

- newMarginDepositedTotal = globalPositions.marginDepositedTotal + _marginDelta + profitLossTotal
+ newMarginDepositedTotal = globalPositions.marginDepositedTotal + _marginDelta

if (newMarginDepositedTotal < 0) {
    revert FlatcoinErrors.InsufficientGlobalMargin();
}

_globalPositions = FlatcoinStructs.GlobalPositions({
    marginDepositedTotal: uint256(newMarginDepositedTotal),
    sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
    lastPrice: _price
});
        
- _updateStableCollateralTotal(-profitLossTotal);
```

The existing `updateGlobalPositionData` function still needs to be used for other functions besides liquidation. As such, consider creating a separate new function (e.g., updateGlobalPositionDataDuringLiquidation) solely for use during the liquidation that includes the above fixes.

**Verification of solution**

Let's verify if the fixes work as intended using the same example in the report.

The following means that the Initial Deposited Margin (3 ETH) of Alice's position is being transferred to the LPs.

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal((3 ETH + (-4 ETH)) - (-4 ETH)); // This effectively remove the PnL component from the equation
vault.updateStableCollateralTotal(3 ETH);
```

After the `updateStableCollateralTotal` function is executed, the `stableCollateralTotal` will become 15 ETH (12 + 3), the `GlobalPositions.marginDepositedTotal` will remain at 8 ETH, and the total balance of ETH in the vault will remain at 20 ETH.

The same values as the earlier example, except that the formula has changed. The `newMarginDepositedTotal` is left with 5 ETH, which is correct because this represents the ETH margin deposited by Bob's existing position in the system.

```solidity
newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta
newMarginDepositedTotal = 8 ETH + (-3 ETH) = 5 ETH
```

The `newMarginDepositedTotal` is above 0, so there is no revert, which is good.

```solidity
stableCollateralTotal = stableCollateralTotal + 0
stableCollateralTotal = 15 ETH (No change, remain the same)
```

In the end, `newMarginDepositedTotal` is 5 ETH and `stableCollateralTotal` is 15 ETH. There are 20 ETH balance as collateral in the vault.

```solidity
(newMarginDepositedTotal + stableCollateralTotal) == (20 ETH balance as collateral in the vault)
```

Thus, they are in sync now. Also, there is no revert or underflow error.