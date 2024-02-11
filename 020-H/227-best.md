Sneaky Flaxen Rhino

high

# Malicious User can create a position that nobody can liquidate

## Summary

In function `executeClose`, after burning an NFT, it will only detect and clear existing limit orders, but not detect existing delayed orders. This allows an attacker to add margin and increase position size to a `tokenID` where no NFT exists to create a position that cannot be liquidated. The platform risks having bad debt that cannot be eliminated.

## Vulnerability Detail

In flatcoin protocol, user can close a position by calling `announceLimitOrder` or `announceLeverageClose`. When keepers try to execute the limit order or the delayed order, function `executeClose` in LeverageModule is called. In `executeClose`, there is a check for cancellation of any existing limit orders on the position, but **does not cancel an existing Delayed order** if `executeClose` is called by `executeLimitOrder()`!

    function executeClose(
        address _account,
        address _keeper,
        FlatcoinStructs.Order calldata _order
    ) external whenNotPaused onlyAuthorizedModule returns (int256 settledMargin) {
            ...
            // Delete position storage
            vault.deletePosition(announcedClose.tokenId);
        }

        // Cancel any existing limit order on the position
        ILimitOrder(vault.moduleAddress(FlatcoinModuleKeys._LIMIT_ORDER_KEY)).cancelExistingLimitOrder(
            announcedClose.tokenId
        );

        // A position NFT has to be unlocked before burning otherwise, the transfer to address(0) will fail.
        _unlock(announcedClose.tokenId);
        _burn(announcedClose.tokenId);

        vault.updateStableCollateralTotal(int256(announcedClose.tradeFee)); // pay the trade fee to stable LPs

        // Settle the collateral.
        vault.sendCollateral({to: _keeper, amount: _order.keeperFee}); // pay the keeper their fee
        vault.sendCollateral({to: _account, amount: uint256(settledMargin) - totalFee}); // transfer remaining amount to the trader

        emit FlatcoinEvents.LeverageClose(announcedClose.tokenId);
    }

When performing leverage adjustments, since the validation of the caller's possession of the NFT is performed at `announceLeverageAdjust`, `executeAdjust` does not have a test for the existence of a current position, and allows a position with all parameters 0 to operate normally.

Consider path below:

1. Bob has a position and calls `announceLimitOrder` to create a limit order that can be executed immediately.
2. Bob calls `announceLeverageAdjust` to create a delayed order to add some margin and increase position size. Since the limit order has not been executed at this time, all validations in `announceLeverageAdjust` will pass.
3. a keeper execute the limit order to close Bob's position, and clear its storage. At this point, since only the presence of a limit order is checked, the delayed order is still valid.
4. a keeper execute the delayed order, "create" a new position on the same tokenID. Because the NFT has been destroyed, its owner is address 0. Nobody can liquidate or close this position because OZ's ERC721 doesn't allow burning non-existent token.

## Impact

Attackers can create positions which nobody can liquidate, so there is a risk that the platform will incur bad debts that cannot be eliminated.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L255-L317
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L147-L248

## Tool used

Manual Review

## Recommendation

In function executeClose, only cancel any existing limit order on the position is not enough. delayed order should also be canceled.
