Formal Eggshell Sidewinder

high

# Vault Inflation Attack

## Summary

Malicious users can perform an inflation attack against the vault to steal the assets of the victim.

## Vulnerability Detail

A malicious user can perform a donation to execute a classic first depositor/ERC4626 inflation Attack against the FlatCoin vault. The general process of this attack is well-known, and a detailed explanation of this attack can be found in many of the resources such as the following:

- https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks
- https://mixbytes.io/blog/overview-of-the-inflation-attack

In short, to kick-start the attack, the malicious user will often usually mint the smallest possible amount of shares (e.g., 1 wei) and then donate significant assets to the vault to inflate the number of assets per share. Subsequently, it will cause a rounding error when other users deposit.

However, in Flatcoin, there are various safeguards in place to mitigate this attack. Thus, one would need to perform additional steps to workaround/bypass the existing controls.

Let's divide the setup of the attack into two main parts:

1. Malicious user mint 1 mint of share
2. Donate or transfer assets to the vault to inflate the assets per share

#### Part 1 - Malicious user mint 1 mint of share

Users could attempt to mint 1 wei of share. However, the validation check at Line 79 will revert as the share minted is less than `MIN_LIQUIDITY` = 10_000. However, this minimum liquidation requirement check can be bypassed.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L61

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
71: 
72:         if (_liquidityMinted < _announcedDeposit.minAmountOut)
73:             revert FlatcoinErrors.HighSlippage(_liquidityMinted, _announcedDeposit.minAmountOut);
74: 
75:         _mint(_account, _liquidityMinted);
76: 
77:         vault.updateStableCollateralTotal(int256(depositAmount));
78: 
79:         if (totalSupply() < MIN_LIQUIDITY) // @audit-info MIN_LIQUIDITY = 10_000
80:             revert FlatcoinErrors.AmountTooSmall({amount: totalSupply(), minAmount: MIN_LIQUIDITY});
```

First, Bob mints 10000 wei shares via `executeDeposit` function. Next, Bob withdraws 9999 wei shares via the `executeWithdraw`. In the end, Bob successfully owned only 1 wei share, which is the prerequisite for this attack.

#### Part 2 - Donate or transfer assets to the vault to inflate the assets per share

The vault tracks the number of collateral within the state variables. Thus, simply transferring rETH collateral to the vault directly will not work, and the assets per share will remain the same.

To work around this, Bob creates a large number of accounts (with different wallet addresses). He could choose any or both of the following methods to indirectly transfer collateral to the LP pool/vault to inflate the assets per share:

1) Open a large number of leveraged long positions with the intention of incurring large amounts of losses. The long positions' losses are the gains of the LPs, and the collateral per share will increase.
2) Open a large number of leveraged long positions till the max skew of 120%. Thus, this will cause the funding rate to increase, and the long will have to pay the LPs, which will also increase the collateral per share.

#### Triggering rounding error

The `stableCollateralPerShare` will be inflated at this point. Following is the formula used to determine the number of shares minted to the depositor.

If the `depositAmount` by the victim is not sufficiently large enough, the amount of shares minted to the depositor will round down to zero.

```solidity
_collateralPerShare = (stableBalance * (10 ** decimals())) / totalSupply;
_liquidityMinted = (depositAmount * (10 ** decimals())) / _collateralPerShare
```

Finally, the attacker withdraws their share from the pool. Since they are the only ones with any shares, this withdrawal equals the balance of the vault. This means the attacker also withdraws the tokens deposited by the victim earlier.

## Impact

Malicous users could steal the assets of the victim.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L61

## Tool used

Manual Review

## Recommendation

A `MIN_LIQUIDITY` amount of shares needs to exist within the vault to guard against a common inflation attack.

However, the current approach of only checking if the `totalSupply() < MIN_LIQUIDITY` is not sufficient, and could be bypassed by making use of the withdraw function.

A more robust approach to ensuring that there is always a minimum number of shares to guard against inflation attack is to mint a certain amount of shares to zero address (dead address) during contract deployment (similar to what has been implemented in Uniswap V2). 