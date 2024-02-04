Formal Eggshell Sidewinder

high

# Incorrect price used when updating the global position data

## Summary

Incorrect price used when updating the global position data leading to a loss of assets for LPs.

## Vulnerability Detail

Near the end of the liquidation process, the `updateGlobalPositionData` function at Line 159 will be executed to update the global position data. However, when executing the `updateGlobalPositionData` function, the code sets the price at Line 160 below to the position's last price (`position.lastPrice`), which is incorrect. The price should be set to the current price instead, and not the position's last price.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L160

```solidity
File: LiquidationModule.sol
082:     /// @notice Function to liquidate a position.
083:     /// @dev One could directly call this method instead of `liquidate(uint256, bytes[])` if they don't want to update the Pyth price.
084:     /// @param tokenId The token ID of the leverage position.
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
086:         FlatcoinStructs.Position memory position = vault.getPosition(tokenId);
087: 
088:         (uint256 currentPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice();
089: 
090:         // Settle funding fees accrued till now.
091:         vault.settleFundingFees();
092: 
093:         // Check if the position can indeed be liquidated.
094:         if (!canLiquidate(tokenId)) revert FlatcoinErrors.CannotLiquidate(tokenId);
095: 
096:         FlatcoinStructs.PositionSummary memory positionSummary = PerpMath._getPositionSummary(
097:             position,
098:             vault.cumulativeFundingRate(),
099:             currentPrice
100:         );
..SNIP..
159:         vault.updateGlobalPositionData({
160:             price: position.lastPrice,
161:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
162:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
163:         });
```

The reason why the `updateGlobalPositionData` function expects a current price to be passed in is that within the `PerpMath._profitLossTotal` function, it will compute the price shift between the current price and the last price to obtain the PnL of all the open positions. Also, per the comment at Line 170 below, it expects the current price of the collateral to be passed in.

Thus, it is incorrect to pass in the individual position's last/entry price, which is usually the price of the collateral when the position was first opened or adjusted some time ago.

Thus, if the last/entry price of the liquidated position is higher than the current price of collateral, the PnL will be inflated, indicating more gain for the long traders. Since this is a zero-sum game, this also means that the LP loses more assets than expected due to the inflated gain of the long traders.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

```solidity
File: FlatcoinVault.sol
168:     /// @notice Function to update the global position data.
169:     /// @dev This function is only callable by the authorized modules.
170:     /// @param _price The current price of the underlying asset.
171:     /// @param _marginDelta The change in the margin deposited total.
172:     /// @param _additionalSizeDelta The change in the size opened total.
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
```

## Impact

Loss of assets for the LP as mentioned in the above section.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L160

## Tool used

Manual Review

## Recommendation

Use the current price instead of liquidated position's last price when update the global position data

```diff
(uint256 currentPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice();
..SNIP..
vault.updateGlobalPositionData({
-    price: position.lastPrice,
+    price: currentPrice,    
    marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
    additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
});
```