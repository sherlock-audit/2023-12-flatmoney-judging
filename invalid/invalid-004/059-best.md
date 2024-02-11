Bald Scarlet Locust

high

# Pricefeed update can be bypassed

## Summary
Since there are no permissioned keepers in the system it allows anyone to handle the execution of Limit order execution, Liquidations and Order execution.

## Vulnerability Detail


The issue comes from the following functions 

[executeOrder](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L380)

[executeLimitOrder](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L121)

[liquidate](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L77)

all the functions call updatePythPrice and allow the caller to specify the priceUpdateData to update, this is a problem since the keeper can execute the function with an incorrect pricefeed leading to the pyth priceFeed not being updated and allowing for using to either front running the chainlink price update or to take advantage of the pyth price being from the past by buying with a not updated price and then updating the price which could allow for a user to gain a small risk free profit.

This also allows for keepers to decide if a user should get the current price or an updated price is could depend grief the user if the not updated price is 100 and the updated price would be 105 the keeper can decide if they want to give the use a worse price or a better price.

## Impact
Keepers can update the price in their own favor for gain or against other users for losses

## Code Snippet
```Solidity
    function liquidate(
        uint256 tokenID,
        bytes[] calldata priceUpdateData
    ) external payable whenNotPaused updatePythPrice(vault, msg.sender, priceUpdateData){
```

```Solidity
    function executeLimitOrder(
        uint256 tokenId,
        bytes[] calldata priceUpdateData
    ) external payable nonReentrant updatePythPrice(vault, msg.sender, priceUpdateData) orderInvariantChecks(vault) {
```

```Solidity
    function executeOrder(
        address account,
        bytes[] calldata priceUpdateData
    )
        external
        payable
        nonReentrant
        whenNotPaused
        updatePythPrice(vault, msg.sender, priceUpdateData) 
        orderInvariantChecks(vault)
    {
```
## Tool used

Manual Review

## Recommendation
consider changing the updatePythPrice to always update the correct price feed instead of updating what the keeper passes in.
