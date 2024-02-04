Sweet Onyx Goat

high

# Leverage opened Order cannot be closed if  protocol is paused

## Summary
When the protocol is paused, `executeClose` function is set to revert when called . This causes closing of limit orders to be impossible, leading to two issues. The first is that users with existing trades are unable to close their position to recover their collateral. The second is that protocol could end up being stuck with a lot of bad debt.

## Vulnerability Detail
This Function can only be called when the protocol is not not paused.

```solidity
 function executeClose(
        address _account,
        address _keeper,
        FlatcoinStructs.Order calldata _order
    ) external whenNotPaused onlyAuthorizedModule returns (int256 settledMargin) {
```
## Impact
Outstanding limit orders cannot be closed, freezing user funds and causing the protocol to take on bad debt

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L255

## Tool used

Manual Review

## Recommendation

Allow `executeClose` function to be called when protocol is Paused.
