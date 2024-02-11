Formal Eggshell Sidewinder

medium

# Positions are immediately liquidated after protocol resumes

## Summary

Long traders whose positions become liquidatable during a pause will not be able to top up their margin to save their position or avoid liquidation. As a result, their positions are immediately liquidated after protocol resumes.

## Vulnerability Detail

When the protocol is paused, the long traders will not be able to top up their positions if they are on the verge of falling below the liquidation margin. During the period when the protocol is paused, market fluctuation continues to happen and the price of the rETH might drop, leading to a loss to the long trader's positions. Some of the long positions will inevitably cross the liquidation margin and become liquidable.

As soon as the protocol is unpaused, these long traders' positions will be immediately liquidated by the keeper bots, with the only possibility to save their position being if the long traders front-running the keeper bots.

This situation unfairly disadvantages long traders as they became subject to liquidation through no fault of their own. Upon resuming the protocol, long traders will be immediately liquidated, unfairly disadvantaging the long traders and giving a huge advantage to the Liquidator.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

```solidity
File: LiquidationModule.sol
82:     /// @notice Function to liquidate a position.
83:     /// @dev One could directly call this method instead of `liquidate(uint256, bytes[])` if they don't want to update the Pyth price.
84:     /// @param tokenId The token ID of the leverage position.
85:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
86:         FlatcoinStructs.Position memory position = vault.getPosition(tokenId);
87: 
88:         (uint256 currentPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice();
```

## Impact

Long traders whose positions become liquidatable during a pause will not be able to top up their margin to save their position or avoid liquidation.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

## Tool used

Manual Review

## Recommendation

To fix the game theory such that neither Long Traders nor Liquidators are unfairly favored after the protocol is resumed, there should be a grace period during which long traders cannot be liquidated. This grace period could be equal to the period that protocol was paused with a hard cap of a maximum number of hours, which provides even fairness to both long traders and liquidators.

#### Reference

- https://dacian.me/lending-borrowing-defi-attacks#heading-borrower-immediately-liquidated-after-repayments-resume (Research Article)
- https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/117 (Past Report)