Formal Eggshell Sidewinder

high

# Oracle will not failover as expected during liquidation

## Summary

Oracle will not failover as expected during liquidation. If the liquidation cannot be executed due to the revert described in the following scenario, underwater positions and bad debt accumulate in the protocol, threatening the solvency of the protocol.

## Vulnerability Detail

The liquidators have the option to update the Pyth price during liquidation. If the liquidators do not intend to update the Pyth price during liquidation, they have to call the second `liquidate(uint256 tokenId)` function at Line 85 below directly, which does not have the `updatePythPrice` modifier.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L75

```solidity
File: LiquidationModule.sol
75:     function liquidate(
76:         uint256 tokenID,
77:         bytes[] calldata priceUpdateData
78:     ) external payable whenNotPaused updatePythPrice(vault, msg.sender, priceUpdateData) {
79:         liquidate(tokenID);
80:     }
81: 
82:     /// @notice Function to liquidate a position.
83:     /// @dev One could directly call this method instead of `liquidate(uint256, bytes[])` if they don't want to update the Pyth price.
84:     /// @param tokenId The token ID of the leverage position.
85:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
86:         FlatcoinStructs.Position memory position = vault.getPosition(tokenId);
```

It was understood from the protocol team that the rationale for allowing the liquidators to execute a liquidation without updating the Pyth price is to ensure that the liquidations will work regardless of Pyth's working status, in which case Chainlink is the fallback, and the last oracle price will be used for the liquidation.

However, upon further review, it was found that the fallback mechanism within the FlatCoin protocol does not work as expected by the protocol team.

Assume that Pyth is down. In this case, no one would be able to fetch the latest off-chain price from Pyth network and update Pyth on-chain contract. As a result, the prices stored in the Pyth on-chain contract will become outdated and stale. 

When liquidation is executed in FlatCoin protocol, the following `_getPrice` function will be executed to fetch the price. Line 107 below will fetch the latest price from Chainlink, while Line 108 below will fetch the last available price on the Pyth on-chain contract. When the Pyth on-chain prices have not been updated for a period of time, the deviation between `onchainPrice` and `offchainPrice` will widen till a point where `diffPercent > maxDiffPercent` and a revert will occur at Line 113 below, thus blocking the liquidation from being carried out. As a result, the liquidation mechanism within the FlatCoin protocol will stop working.

Also, the protocol team's goal of allowing the liquidators to execute a liquidation without updating the Pyth price to ensure that the liquidations will work regardless of Pyth's working status will not be achieved.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L113

```solidity
File: OracleModule.sol
102:     /// @notice Returns the latest 18 decimal price of asset from either Pyth.network or Chainlink.
103:     /// @dev It verifies the Pyth network price against Chainlink price (ensure that it is within a threshold).
104:     /// @return price The latest 18 decimal price of asset.
105:     /// @return timestamp The timestamp of the latest price.
106:     function _getPrice(uint32 maxAge) internal view returns (uint256 price, uint256 timestamp) {
107:         (uint256 onchainPrice, uint256 onchainTime) = _getOnchainPrice(); // will revert if invalid
108:         (uint256 offchainPrice, uint256 offchainTime, bool offchainInvalid) = _getOffchainPrice();
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
131:         if (maxAge < type(uint32).max && timestamp + maxAge < block.timestamp) {
132:             revert FlatcoinErrors.PriceStale(
133:                 offchain ? FlatcoinErrors.PriceSource.OffChain : FlatcoinErrors.PriceSource.OnChain
134:             );
135:         }
136:     }
```

## Impact

The liquidation mechanism is the core component of the protocol and is important to the solvency of the protocol. If the liquidation cannot be executed due to the revert described in the above scenario, underwater positions and bad debt accumulate in the protocol threaten the solvency of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L75

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L113C10-L113C92

## Tool used

Manual Review

## Recommendation

Consider implementing a feature to allow the protocol team to disable the price deviation check so that the protocol team can disable it in the event that Pyth network is down for an extended period of time.