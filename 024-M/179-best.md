Formal Eggshell Sidewinder

high

# Deposit does not round in favor of the vault

## Summary

When calculating how many shares/UNIT to be minted to a user for a certain amount of the collateral they deposit, it does not round in favor of the vault, leading to users getting more shares than expected.

## Vulnerability Detail

The `stableCollateralPerShare` function computes the number of collaterals per share. Note that in Line 214, the calculation is rounded down. This means that if the `stableBalance` is 19 and `totalSupply` is 10, the result will be 1 rETH/share in Solidity instead of 1.9 rETH/share if floating point is supported in other languages.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L214

```solidity
File: StableModule.sol
208:     function stableCollateralPerShare(uint32 _maxAge) public view returns (uint256 _collateralPerShare) {
209:         uint256 totalSupply = totalSupply();
210: 
211:         if (totalSupply > 0) {
212:             uint256 stableBalance = stableCollateralTotalAfterSettlement(_maxAge);
213: 
214:             _collateralPerShare = (stableBalance * (10 ** decimals())) / totalSupply;
215:         } else {
216:             // no shares have been minted yet
217:             _collateralPerShare = 1e18;
218:         }
219:     }
```

When the `_collateralPerShare` result returned from the `stableCollateralPerShare` function is rounded down, one share is worth fewer collateral tokens (rETH). Intuitively, this means that the share becomes cheaper. As such, you can receive more shares for a specific amount of assets you deposited since the price of each asset is now cheaper.

The `stableCollateralPerShare` function is used within the `executeDeposit` function to compute the price per share (PPS) to determine the number of shares/UNIT to be minted to the depositor. Since the PPS is rounded down (cheaper than expected)(e.g. 1 rETH per share instead of 1.9 rETH per share), this means that the depositor will get more shares/UNIT than expected. Thus, the vault rounds in favor of the users instead of itself.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L70

```solidity
File: StableModule.sol
61:     function executeDeposit(
62:         address _account,
63:         uint64 _executableAtTime,
64:         FlatcoinStructs.AnnouncedStableDeposit calldata _announcedDeposit
65:     ) external whenNotPaused onlyAuthorizedModule returns (uint256 _liquidityMinted) {
66:         uint256 depositAmount = _announcedDeposit.depositAmount;
67: 
68:         uint32 maxAge = _getMaxAge(_executableAtTime);
69: 
70:         _liquidityMinted = (depositAmount * (10 ** decimals())) / stableCollateralPerShare(maxAge);
```

Assume that Bob deposits 20 rETH:

- With rounding error (PPS = 1) - 20/1 = 20 shares minted (Current vault implementati)
- Without rounding error (PPS = 1.9) - 10.5 shares minted

The above shows that the vault does not round in favor of itself.

## Impact

As shown in the example above, users are getting more shares than expected (20 vs 10.5).

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L214

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L70

## Tool used

Manual Review

## Recommendation

A rule of thumb when implementing a vault is to always round in favor of the vault instead of the users.

When calculating many shares/UNIT to be minted to a user for a certain amount of the collateral they deposit, it should always be rounded in favor of the vault instead of the users.

It is fine to use the existing `stableCollateralPerShare` for withdrawal, as rounding down means fewer assets per share, and the users will receive fewer assets in exchange for their shares/UNITs, thus favoring the vault.

However, it is not unsafe to use the existing `stableCollateralPerShare` function for deposit, as it is rounding in favor of the users. The formula used for computing the `stableCollateralPerShare` should be rounded up as follows (Assuming using [Solmate's Math](https://github.com/transmissions11/solmate/blob/c892309933b25c03d32b1b0d674df7ae292ba925/src/utils/FixedPointMathLib.sol#L53))

```solidity
depositAmount.mulDivUp((10 ** decimals()), totalSupply);
```