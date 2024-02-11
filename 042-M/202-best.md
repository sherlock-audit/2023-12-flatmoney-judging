Formal Eggshell Sidewinder

medium

# Existing limit order is not cancelled

## Summary

The existing limit order is not cancelled when a new limit order is announced. As a result, the keepers will be DOSed, which might degrade or halt the ability of the keeper to carry out their roles (e.g., executing orders and liquidating accounts).

For instance, if the liquidating of accounts cannot be executed or is being slowed down, the solvency of the protocol will be threatened, OR if the traders cannot have their positions adjusted (adding more margin) or closed in a timely manner, their positions might be at risk of being liquidated, leading to a loss for the traders. 

## Vulnerability Detail

Per the protocol team's response in the contest's Discord channel, it was understood the keepers operate as follows:

> Keepers are external bots. Anyone can create/operate a keeper. Keepers are incentivised to execute an order using keeper fees and incentivised to liquidate using liquidator fees. Keepers can choose to execute and order or leave it depending on if the order is profitable for them (execution cost is less than keeper fee) or not.
>
> Listening is done by events I believe.

When announcing a deposit, withdraw, open, adjust and close order, the code will check if the order already exists and expired. If so, it will cancel the existing order and emit the `FlatcoinEvents.OrderCancelled` to inform the keepers that this order has been cancelled.

However, when announcing a new limit order, the code does not check if there already exists a limit order and cancels it. As a result, it is possible for malicious users to DOS the keepers who are listening to `FlatcoinEvents.OrderAnnounced` event by executing a large number of `announceLimitOrder` TX continuously and emitting a large number of `FlatcoinEvents.OrderAnnounced`, thus overwhelming the keepers as they will see those announced orders that have not been canceled as valid.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L58

```solidity
File: LimitOrder.sol
58:     function announceLimitOrder(uint256 tokenId, uint256 priceLowerThreshold, uint256 priceUpperThreshold) external {
59:         uint64 executableAtTime = _prepareAnnouncementOrder();
60:         address positionOwner = _checkPositionOwner(tokenId);
61:         _checkThresholds(priceLowerThreshold, priceUpperThreshold);
62:         uint256 tradeFee = ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).getTradeFee(
63:             vault.getPosition(tokenId).additionalSize
64:         );
65: 
66:         _limitOrderClose[tokenId] = FlatcoinStructs.Order({
67:             orderType: FlatcoinStructs.OrderType.LimitClose,
68:             orderData: abi.encode(
69:                 FlatcoinStructs.LimitClose(tokenId, priceLowerThreshold, priceUpperThreshold, tradeFee)
70:             ),
71:             keeperFee: 0, // Not applicable for limit orders. Keeper fee will be determined at execution time.
72:             executableAtTime: executableAtTime
73:         });
74: 
75:         // Lock the NFT belonging to this position so that it can't be transferred to someone else.
76:         ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).lock(tokenId);
77: 
78:         emit FlatcoinEvents.OrderAnnounced({
79:             account: positionOwner,
80:             orderType: FlatcoinStructs.OrderType.LimitClose,
81:             keeperFee: 0
82:         });
83:     }
```

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L198

```solidity
File: LimitOrder.sol
197:     /// @dev This function HAS to be called as soon as the transaction flow enters an announce function.
198:     function _prepareAnnouncementOrder() internal returns (uint64 executableAtTime) {
199:         // Settle funding fees to not encounter the `MaxSkewReached` error.
200:         // This error could happen if the funding fees are not settled for a long time and the market is skewed long
201:         // for a long time.
202:         vault.settleFundingFees();
203: 
204:         executableAtTime = uint64(block.timestamp + vault.minExecutabilityAge());
205:     }
```

## Impact

Keepers will be DOSed, which might degrade or halt the ability of the keeper to carry out its roles (e.g., executing orders and liquidating accounts). If the keepers cannot carry out some of the roles, it will have various negative impacts on the protocols and users.

For instance, if the liquidating of accounts cannot be executed or is being slowed down, the solvency of the protocol will be threatened, OR if the traders cannot have their positions adjusted or closed in a timely manner, their positions might be at risk of being liquidated, leading to a loss for the traders. 

Lastly, from the perspective of keepers, they also wasted their gas executing orders that are no longer valid as the limit order has already been overwritten by a newer limit order, but they were not informed via the event.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L58

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L198

## Tool used

Manual Review

## Recommendation

If a limit order already exists, cancel the existing limit order and emit the `FlatcoinEvents.OrderCancelled` to inform the keepers that this order has been cancelled before announcing a new limit order.