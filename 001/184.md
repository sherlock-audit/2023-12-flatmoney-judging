Formal Eggshell Sidewinder

high

# Last long position that is underwater will be stuck

## Summary

The last long position in the system cannot be liquidated, leading to a loss for the LPs as the liquidated position's margin cannot be transferred to the LPs.

## Vulnerability Detail

At the end of the liquidation process, it will call the `updateGlobalPositionData` function.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L161

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
159:         vault.updateGlobalPositionData({
160:             price: position.lastPrice,
161:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
162:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
163:         });
```

Assume that the last LONG position is underwater (settledMargin < 0). For instance, the state of the last long position is as follows:

- Margin = 20 ETH
- Accrued Funding = 0 (For simplicity)
- PnL = -100 ETH

At some point of the liquidation process, the `updateGlobalPositionData` will be executed to update the global position data:

```solidity
vault.updateGlobalPositionData({marginDelta: -(position.marginDeposited + positionSummary.accruedFunding)});
vault.updateGlobalPositionData({marginDelta: -(20 + 0)});
vault.updateGlobalPositionData({marginDelta: -20});

profitLossTotal = -100

newMarginDepositedTotal = marginDepositedTotal + _marginDelta + profitLossTotal
newMarginDepositedTotal = 20 + (-20) + (-100)
newMarginDepositedTotal = -20
```

The `profitLossTotal` is `-100` as it represents the loss incurred by the last long position in the system.

Since the final `newMarginDepositedTotal` is `-20`, Line 192 below will revert and cause the liquidation to fail.

```solidity
File: FlatcoinVault.sol
173:     function updateGlobalPositionData(
..SNIP..
191:         if (newMarginDepositedTotal < 0) {
192:             revert FlatcoinErrors.InsufficientGlobalMargin();
193:         }
```

As a result, it is not possible to liquidate the last long position in the system.

## Impact

The last long position in the system cannot be liquidated, leading to a loss for the LPs as the liquidated position's margin cannot be transferred to the LPs.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L161

## Tool used

Manual Review

## Recommendation

Update the liquidation function so that the last long position in the system can be liquidated successfully.