Formal Eggshell Sidewinder

high

# Incorrect handling of PnL during liquidation

## Summary

The incorrect handling of PnL during liquidation led to an error in the protocol's accounting mechanism, which might result in various issues, such as the loss of assets and the stable collateral total being inflated.

## Vulnerability Detail

#### First Example

Assume a long position with the following state:

- Margin Deposited = +20
- Accrued Funding = -100
- Profit & Loss (PnL) = +100
- Liquidation Margin = 30
- Liquidation Fee = 25
- Settled Margin = Margin Deposited + Accrued Funding + PnL = 20

Let the current `StableCollateralTotal` be $x$ and `marginDepositedTotal` be $y$ at the start of the liquidation.

Firstly, the `settleFundingFees()` function will be executed at the start of the liquidation process. The effect of the `settleFundingFees()` function is shown below. The long trader's `marginDepositedTotal` will be reduced by 100, while the LP's `stableCollateralTotal` will increase by 100.

```solidity
settleFundingFees() = Short/LP need to pay Long 100

marginDepositedTotal = marginDepositedTotal + funding fee
marginDepositedTotal = y + (-100) = (y - 100)

stableCollateralTotal = x + (-(-100)) = (x + 100)
```

Since the position's settle margin is below the liquidation margin, the position will be liquidated.

At Line 109, the condition `(settledMargin > 0)` will be evaluated as `True`. At Line 123:

```solidity
if (uint256(settledMargin) > expectedLiquidationFee)
if (+20 > +25) => False
liquidatorFee = settledMargin
liquidatorFee = +20
```

The `liquidationFee` will be to +20 at Line 127 below. This basically means that all the remaining margin of 20 will be given to the liquidator, and there should be no remaining margin for the LPs.

At Line 133 below, the `vault.updateStableCollateralTotal` function will be executed:

```solidity
vault.updateStableCollateralTotal(remainingMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal(0 - (+100));
vault.updateStableCollateralTotal(-100);

stableCollateralTotal = (x + 100) - 100 = x
```

When `vault.updateStableCollateralTotal` is set to `-100`, `stableCollateralTotal` is equal to $x$.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
102:         // Check that the total margin deposited by the long traders is not -ve.
103:         // To get this amount, we will have to account for the PnL and funding fees accrued.
104:         int256 settledMargin = positionSummary.marginAfterSettlement;
105: 
106:         uint256 liquidatorFee;
107: 
108:         // If the settled margin is greater than 0, send a portion (or all) of the margin to the liquidator and LPs.
109:         if (settledMargin > 0) {
110:             // Calculate the liquidation fees to be sent to the caller.
111:             uint256 expectedLiquidationFee = PerpMath._liquidationFee(
112:                 position.additionalSize,
113:                 liquidationFeeRatio,
114:                 liquidationFeeLowerBound,
115:                 liquidationFeeUpperBound,
116:                 currentPrice
117:             );
118: 
119:             uint256 remainingMargin;
120: 
121:             // Calculate the remaining margin after accounting for liquidation fees.
122:             // If the settled margin is less than the liquidation fee, then the liquidator fee is the settled margin.
123:             if (uint256(settledMargin) > expectedLiquidationFee) {
124:                 liquidatorFee = expectedLiquidationFee;
125:                 remainingMargin = uint256(settledMargin) - expectedLiquidationFee;
126:             } else {
127:                 liquidatorFee = uint256(settledMargin);
128:             }
129: 
130:             // Adjust the stable collateral total to account for user's remaining margin.
131:             // If the remaining margin is greater than 0, this goes to the LPs.
132:             // Note that {`remainingMargin` - `profitLoss`} is the same as {`marginDeposited` + `accruedFunding`}.
133:             vault.updateStableCollateralTotal(int256(remainingMargin) - positionSummary.profitLoss);
134: 
135:             // Send the liquidator fee to the caller of the function.
136:             // If the liquidation fee is greater than the remaining margin, then send the remaining margin.
137:             vault.sendCollateral(msg.sender, liquidatorFee);
138:         } else {
139:             // If the settled margin is -ve then the LPs have to bear the cost.
140:             // Adjust the stable collateral total to account for user's profit/loss and the negative margin.
141:             // Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
142:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
143:         }
```

Next, the `vault.updateGlobalPositionData` function [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L159) will be executed.

```solidity
vault.updateGlobalPositionData({marginDelta: -(position.marginDeposited + positionSummary.accruedFunding)})
vault.updateGlobalPositionData({marginDelta: -(20 + (-100))})
vault.updateGlobalPositionData({marginDelta: 80})

profitLossTotal = 100
newMarginDepositedTotal = globalPositions.marginDepositedTotal + marginDelta + profitLossTotal
newMarginDepositedTotal = (y - 100) + 80 + 100 = (y + 80)

stableCollateralTotal = stableCollateralTotal + -PnL
stableCollateralTotal = x + (-100) = (x - 100)
```

The final `newMarginDepositedTotal` is $y + 80$ and `stableCollateralTotal` is $x -100$, which is incorrect. In this scenario

- There is no remaining margin for the LPs, as all the remaining margin has been sent to the liquidator as a fee. The remaining margin (settled margin) is also not negative. Thus, there should not be any loss on the `stableCollateralTotal`. The correct final `stableCollateralTotal` should be $x$.
- The final `newMarginDepositedTotal` is $y + 80$, which is incorrect as this indicates that the long trader's pool has gained 80 ETH, which should not be the case when a long position is being liquidated.

#### Second Example

The current price of rETH is \$1000.

Let's say there is a user A (Alice) who makes a deposit of 5 rETH as collateral for LP.

Let's say another user, Bob (B), comes up, deposits 2 rETH as a margin, and creates a position with a size of 5 rETH, basically creating a perfectly hedged market. Since this is a perfectly hedged market, the accrued funding fee will be zero for the context of this example.

Total collateral in the system = 5 rETH + 2 rETH = 7 rETH

After some time, the price of rETH drop to \$500. As a result, Bob's position is liquidated as its settled margin is less than zero.

$$
settleMargin = 2\  rETH + \frac{5 \times (500 - 1000)}{500}
= 2\ rETH - 5\ rETH = -3\ rETH
$$

During the liquidation, the following code is executed to update the LP's stable collateral total:

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal(-3 rETH - (-5 rETH));
vault.updateStableCollateralTotal(+2);
```

LP's stable collateral total increased by 2 rETH.

Subsequently, the `updateGlobalPositionData` function will be executed.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L159

```solidity
File: LiquidationModule.sol
85:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
159:         vault.updateGlobalPositionData({
160:             price: position.lastPrice,
161:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
162:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
163:         });
```

Within the `updateGlobalPositionData` function, the `profitLossTotal` at Line 179 will be -5 rETH. This means that the long trader (Bob) has lost 5 rETH.

At Line 205 below, the PnL of the long traders (-5 rETH) will be transferred to the LP's stable collateral total. In this case, the LPs gain 5 rETH.

Note that the LP's stable collateral total has been increased by 2 rETH earlier and now we are increasing it by 5 rETH again. Thus, the total gain by LPs is 7 rETH. If we add 7 rETH to the original stable collateral total, it will be 7 rETH + 5 rETH = 12 rETH. However, this is incorrect because we only have 7 rETH collateral within the system, as shown at the start.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

```solidity
File: FlatcoinVault.sol
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

#### Third Example

At $T0$, the marginDepositedTotal = 70 ETH, stableCollateralTotal = 100 ETH, vault's balance = 170 ETH

| Bob's Long Position                                          | Alice (LP)          |
| ------------------------------------------------------------ | ------------------- |
| Margin = 70 ETH<br />Position Size = 500 ETH<br />Leverage = (500 + 20) / 20 = 26x<br />Liquidation Fee = 50 ETH<br />Liquidation Margin = 60 ETH<br />Entry Price = \$1000 per ETH | Deposited = 100 ETH |

At $T1$​, the position's settled margin falls to 60 ETH (margin = +70, accrued fee = -5, PnL = -5) and is subjected to liquidation.

Firstly, the `settleFundingFees()` function will be executed at the start of the liquidation process. The effect of the `settleFundingFees()` function is shown below. The long trader's `marginDepositedTotal` will be reduced by 5, while the LP's `stableCollateralTotal` will increase by 5.

```solidity
settleFundingFees() = Long need to pay short 5

marginDepositedTotal = marginDepositedTotal + funding fee
marginDepositedTotal = 70 + (-5) = 65

stableCollateralTotal = 100 + (-(-5)) = 105
```

Next, [this part](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L109) of the code will be executed to send a portion of the liquidated position's margin to the liquidator and LPs.

```solidity
settledMargin > 0 => True
(settledMargin > expectedLiquidationFee) => (+60 > +50) => True
remainingMargin = uint256(settledMargin) - expectedLiquidationFee = 60 - 50 = 10
```

50 ETH will be sent to the liquidator and the remaining 10 ETH should goes to the LPs.

```solidity
vault.updateStableCollateralTotal(remainingMargin - positionSummary.profitLoss) =>
stableCollateralTotal = 105 ETH + (remaining margin - PnL)
stableCollateralTotal = 105 ETH + (10 ETH - (-5 ETH))
stableCollateralTotal = 105 ETH + (15 ETH) = 120 ETH
```

Next, the `vault.updateGlobalPositionData` function [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L159) will be executed.

```solidity
vault.updateGlobalPositionData({marginDelta: -(position.marginDeposited + positionSummary.accruedFunding)})
vault.updateGlobalPositionData({marginDelta: -(70 + (-5))})
vault.updateGlobalPositionData({marginDelta: -65})

profitLossTotal = -5
newMarginDepositedTotal = globalPositions.marginDepositedTotal + marginDelta + profitLossTotal
newMarginDepositedTotal = 70 + (-65) + (-5) = 0

stableCollateralTotal = stableCollateralTotal + -PnL
stableCollateralTotal = 120 + (-(5)) = 125
```

The reason why the profitLossTotal = -5 is because there is only one (1) position in the system. So, this loss actually comes from the loss of Bob's position.

The `newMarginDepositedTotal = 0` is correct. This is because the system only has 1 position, which is Bob's position; once the position is liquidated, there should be no margin deposited left in the system.

However, `stableCollateralTotal = 125` is incorrect. Because the vault's collateral balance now is 170 - 50 (send to liquidator) = 120. Thus, the tracked balance and actual collateral balance are not in sync.

## Impact

The following is a list of potential impacts of this issue:

- First Example: LPs incur unnecessary losses during liquidation, which would be avoidable if the calculations were correctly implemented from the start.
- Second Example: An error in the protocol's accounting mechanism led to an inflated increase in the LPs' stable collateral total, which in turn inflated the number of tokens users can withdraw from the system.
- Third Example: The accounting error led to the tracked balance and actual collateral balance not being in sync.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

## Tool used

Manual Review

## Recommendation

To remediate the issue, the `profitLossTotal` should be excluded within the `updateGlobalPositionData` function during liquidation. 

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

The following attempts to apply the above fix to the three (3) examples described in the report to verify that it is working as intended.

#### First Example

Let the current `StableCollateralTotal` be $x$ and `marginDepositedTotal` be $y$ at the start of the liquidation.

- **During funding settlement:** 

StableCollateralTotal = $x$​​ + 100

marginDepositedTotal = $y$ - 100

- **During updateStableCollateralTotal:**

```solidity
vault.updateStableCollateralTotal(int256(remainingMargin) - positionSummary.profitLoss);
vault.updateStableCollateralTotal(0 - (+100));
vault.updateStableCollateralTotal(-100);
```

StableCollateralTotal = ($x$ + 100) - 100 = $x$

- **During Global Position Update:** 

marginDelta = -(position.marginDeposited + positionSummary.accruedFunding) = -(20 + (-100)) = 80

newMarginDepositedTotal = marginDepositedTotal + marginDelta = ($y$ - 100) + 80 = ($y$ - 20)

No change to StableCollateralTotal here. Remain at $x$

- **Conclusion:** 

1) The LPs should not gain or lose in this scenario. Thus, the fact that the StableCollateralTotal remains as $x$ before and after the liquidation is correct.
2) The `marginDepositedTotal` is ($y$ - 20) is correct because the liquidated position's remaining margin is 20 ETH. Thus, when this position is liquidated, 20 ETH should be deducted from the `marginDepositedTotal`
3) No revert during the execution.

#### Second Example

- **During updateStableCollateralTotal:**

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal(-3 rETH - (-5 rETH));
vault.updateStableCollateralTotal(+2);
```

StableCollateralTotal = 5 + 2 = 7 ETH

- **During Global Position Update:**

marginDelta = -(position.marginDeposited + positionSummary.accruedFunding) = -(2 + 0) = -2

marginDepositedTotal = marginDepositedTotal + marginDelta = 2 + (-2) = 0

- **Conclusion:**

StableCollateralTotal = 7 ETH, marginDepositedTotal = 0 (Total 7 ETH tracked in the system)

Balance of collateral in the system = 7 ETH. Thus, both values are in sync. No revert.

#### Third Example

- **During funding settlement (Transfer 5 from Long to LP):** 

marginDepositedTotal = 70 + (-5) = 65

StableCollateralTotal = 100 + 5 = 105

- **Transfer fee to Liquidator**

50 ETH sent to the liquidator from the system: Balance of collateral in the system = 170 ETH - 50 ETH = 120 ETH

- **During updateStableCollateralTotal:**

```solidity
vault.updateStableCollateralTotal(remainingMargin - positionSummary.profitLoss) =>
stableCollateralTotal = 105 ETH + (remaining margin - PnL)
stableCollateralTotal = 105 ETH + (10 ETH - (-5 ETH))
stableCollateralTotal = 105 ETH + (15 ETH) = 120 ETH
```

StableCollateralTotal = 120 ETH

- **During Global Position Update:** 

marginDelta= -(position.marginDeposited + positionSummary.accruedFunding) = -(70 + (-5)) = -65

marginDepositedTotal = 65 + (-65) = 0

- **Conclusion:**

StableCollateralTotal = 120 ETH, marginDepositedTotal = 0 (Total 120 ETH tracked in the system)

Balance of collateral in the system = 120 ETH. Thus, both values are in sync. No revert.