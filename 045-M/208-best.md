Energetic Brown Cougar

medium

# Unfair distribution while ````stableCollateralTotal```` can not cover all traders' profit

## Summary
In the situation that ````stableCollateralTotal```` can't cover all traders' profit, traders who close their positions early can get 100% profit, traders who close lately can get less profit, or in the worst case get nothing. This is not a fair distribution design.

## Vulnerability Detail
Let's say current ````stableCollateralTotal = 10rETH````, and all traders' profit are like
```solidity
Alice = 5 rETH
Bob = 5 rETH
Carl = 2 rETH
```
If Alice and Bob close their position early than Carl, they get all 10ETH. While Carl tries to close his position, there is no penny left. A fair distribution should look like
```solidity
ratio = 10rETH / 12rETH = 5 / 6
Alice = 5 * 5 / 6 = 25 / 6 = 4.17 rETH
Bob = 5 * 5 / 6 = 25 / 6 = 4.17 rETH
Carl = 2 * 5 / 6 = 1.67 rETH
```

## Impact
In this case, part of users would get far less profit than fair distribution.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/LeverageModule.sol#L255


## Tool used

Manual Review

## Recommendation
see PoC
