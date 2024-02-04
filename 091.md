Immense Cloud Pig

high

# `totalFee`(keeperFee + tradeFee) is not deducted from a user's `marginAdjustment` when the margin is being reduced but instead it is added

## Summary
A reduction `marginAdjustment` will be a negative number, negative numbers are reduced differently.

Negative numbers are reduced by (+)adding values to them.

## Vulnerability Detail
```solidity
// Fees come out from the margin if the margin is being reduced or remains unchanged (meaning the size is being modified).
        int256 marginAdjustment = (announcedAdjust.marginAdjustment > 0)
            ? announcedAdjust.marginAdjustment
            : announcedAdjust.marginAdjustment - int256(announcedAdjust.totalFee);// @audit-issue total fee isn't reduced from marginAdjustment but instead it is added.
```
lets say marginAdjustment of user is = -1000$ and total fee = 50$

Doing this ( `announcedAdjust.marginAdjustment - int256(announcedAdjust.totalFee)`) will be = -1000$ -50$ == -1050$

Doing the above ends up adding the fee to the negative marginAdjustment BUT the fee was supposed to be deducted from the negative marginAdjustment and not added.

Now the negative marginAdjustment that has totalFees added to it, is been sent out to account [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L237C9-L245C10)
```solidity
        if (announcedAdjust.marginAdjustment < 0) {
            // We send the user that much margin they requested during announceLeverageAdjust().
            // However their remaining margin is reduced by the fees.
            // It is accounted in announceLeverageAdjust().
            uint256 marginToWithdraw = uint256(announcedAdjust.marginAdjustment * -1);


            // Withdrawing margin from the vault and sending it to the user.
            vault.sendCollateral({to: _account, amount: marginToWithdraw});
        }

```


Hence the protocol will always lose totalFees on every reduction leverage adjustment, which is a loss as taking fees is usually one of the main ways protocols make profit in defi.

## Impact
This flaw will make the protocol always lose out on total fees on every *reduction* Leverage adjust order (I.e whenever marginAdjustment is a negative value).

High severity because Trade fee(which is part of totalFees) is a yield source for LPs and the LPs will always miss out on yields via reduction leverage adjust orders
## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L177-L180

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L237-L245
## Tool used

Manual Review

## Recommendation
Here : 
```solidity
  int256 marginAdjustment = (announcedAdjust.marginAdjustment > 0)
            ? announcedAdjust.marginAdjustment
            : announcedAdjust.marginAdjustment - int256(announcedAdjust.totalFee)
```

change this : `announcedAdjust.marginAdjustment - int256(announcedAdjust.totalFee)` to this : `announcedAdjust.marginAdjustment + int256(announcedAdjust.totalFee)` since `marginAdjustment` will be a negative number

Just number line math. 