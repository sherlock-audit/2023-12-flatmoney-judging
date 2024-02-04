Formal Eggshell Sidewinder

high

# No incentive to liquidate positions where `settledMargin < 0`

## Summary

There is no incentive to liquidation position where the `settledMargin` is less than 0, which affects the solvency of the protocol.

## Vulnerability Detail

Prompt liquidation of positions is important to the solvency of protocol. Liquidators are incentivized to promptly liquidate such "underwater" positions by receiving a liquidation fee for doing so.

However, if the position's `settledMargin` is less than zero, no liquidation fee/incentive will be given to the liquidator. As such, any liquidator that performs the liquidation against these accounts will have to bear the cost of execution.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L142

```solidity
File: LiquidationModule.sol
108:         // If the settled margin is greater than 0, send a portion (or all) of the margin to the liquidator and LPs.
109:         if (settledMargin > 0) {
..SNIP..
138:         } else {
139:             // If the settled margin is -ve then the LPs have to bear the cost.
140:             // Adjust the stable collateral total to account for user's profit/loss and the negative margin.
141:             // Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
142:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
143:         }
```

## Impact

Prompt liquidation of positions is important to the solvency of protocol. These underwater accounts (`settledMargin < 0`) do not provide any liquidation fee as an incentive. Thus, liquidators will not be incentivized to perform liquidation against these accounts and will avoid liquidating these accounts as they will have to bear the gas cost of executing the transaction. As a result, underwater positions and bad debt accumulate in the protocol, threatening the solvency of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L142

## Tool used

Manual Review

## Recommendation

Consider incentivizing the liquidators/keepers to liquidate underwater accounts where it's `settledMargin < 0` so that these positions can be liquidated promptly.