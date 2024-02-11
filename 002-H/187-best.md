Formal Eggshell Sidewinder

high

# Large amounts of points can be minted virtually without any cost

## Summary

Large amounts of points can be minted virtually without any cost. The points are intended to be used to exchange something of value. A malicious user could abuse this to obtain a large number of points, which could obtain excessive value and create unfairness among other protocol users.

## Vulnerability Detail

When depositing stable collateral, the LPs only need to pay for the keeper fee. The keeper fee will be sent to the caller who executed the deposit order.

When withdrawing stable collateral, the LPs need to pay for the keeper fee and withdraw fee. However, there is an instance where one does not need to pay for the withdrawal fee. Per the condition at Line 120 below, if the `totalSupply` is zero, this means that it is the final/last withdrawal. In this case, the withdraw fee will not be applicable and remain at zero.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L96

```solidity
File: StableModule.sol
096:     function executeWithdraw(
097:         address _account,
098:         uint64 _executableAtTime,
099:         FlatcoinStructs.AnnouncedStableWithdraw calldata _announcedWithdraw
100:     ) external whenNotPaused onlyAuthorizedModule returns (uint256 _amountOut, uint256 _withdrawFee) {
101:         uint256 withdrawAmount = _announcedWithdraw.withdrawAmount;
..SNIP..
112:         _burn(_account, withdrawAmount);
..SNIP..
118:         // Check that there is no significant impact on stable token price.
119:         // This should never happen and means that too much value or not enough value was withdrawn.
120:         if (totalSupply() > 0) {
121:             if (
122:                 stableCollateralPerShareAfter < stableCollateralPerShareBefore - 1e6 ||
123:                 stableCollateralPerShareAfter > stableCollateralPerShareBefore + 1e6
124:             ) revert FlatcoinErrors.PriceImpactDuringWithdraw();
125: 
126:             // Apply the withdraw fee if it's not the final withdrawal.
127:             _withdrawFee = (stableWithdrawFee * _amountOut) / 1e18;
128: 
129:             // additionalSkew = 0 because withdrawal was already processed above.
130:             vault.checkSkewMax({additionalSkew: 0});
131:         } else {
132:             // Need to check there are no longs open before allowing full system withdrawal.
133:             uint256 sizeOpenedTotal = vault.getVaultSummary().globalPositions.sizeOpenedTotal;
134: 
135:             if (sizeOpenedTotal != 0) revert FlatcoinErrors.MaxSkewReached(sizeOpenedTotal);
136:             if (stableCollateralPerShareAfter != 1e18) revert FlatcoinErrors.PriceImpactDuringFullWithdraw();
137:         }
```

When LPs deposit rETH and mint UNIT, the protocol will mint points to the depositor's account as per Line 84 below.

Assume that the vault has been newly deployed on-chain. Bob is the first LP to deposit rETH into the vault. Assume for a period of time (e.g., around 30 minutes), there are no other users depositing into the vault except for Bob.

Bob could perform the following actions to mint points for free:

- Bob announces a deposit order to deposit 100e18 rETH. Paid for the keeper fee. (Acting as a LP).
- Wait 10 seconds for the `minExecutabilityAge` to pass
- Bob executes the deposit order and mints 100e18 UNIT (Exchange rate 1:1). Protocol also mints 100e18 points to Bob's account. Bob gets back the keeper fee. (Acting as Keeper)
- Immediately after his `executeDeposit` TX, Bob inserts an "announce withdraw order" TX to withdraw all his 100e18 UNIT and pay for the keeper fee.
- Wait 10 seconds for the `minExecutabilityAge` to pass
- Bob executes the withdraw order and receives back his initial investment of 100e18 rETH. Since he is the only LP in the protocol, it is considered the final/last withdrawal, and he does not need to pay any withdraw fee. He also got back his keeper fee. (Acting as Keeper)

Each attack requires 20 seconds (10 + 10) to be executed. Bob could rinse and repeat the attack until he was no longer the only LP in the system, where he had to pay for the withdraw fee, which might make this attack unprofitable.

If Bob is the only LP in the system for 30 minutes, he could gain 9000e18 points (`(30 minutes / 20 seconds) * 100e18` ) for free as Bob could get back his keeper fee and does not incur any withdraw fee. The only thing that Bob needs to pay for is the gas fee, which is extremely cheap on L2 like Base.

```solidity
File: StableModule.sol
61:     function executeDeposit(
62:         address _account,
63:         uint64 _executableAtTime,
64:         FlatcoinStructs.AnnouncedStableDeposit calldata _announcedDeposit
65:     ) external whenNotPaused onlyAuthorizedModule returns (uint256 _liquidityMinted) {
66:         uint256 depositAmount = _announcedDeposit.depositAmount;
..SNIP..
70:         _liquidityMinted = (depositAmount * (10 ** decimals())) / stableCollateralPerShare(maxAge);
..SNIP..
75:         _mint(_account, _liquidityMinted);
76: 
77:         vault.updateStableCollateralTotal(int256(depositAmount));
..SNIP..
82:         // Mint points
83:         IPointsModule pointsModule = IPointsModule(vault.moduleAddress(FlatcoinModuleKeys._POINTS_MODULE_KEY));
84:         pointsModule.mintDeposit(_account, _announcedDeposit.depositAmount);
```

## Impact

Large amounts of points can be minted virtually without any cost. The points are intended to be used to exchange something of value. A malicious user could abuse this to obtain a large number of points, which could obtain excessive value from the protocol and create unfairness among other protocol users.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L96

## Tool used

Manual Review

## Recommendation

One approach that could mitigate this risk is also to impose withdraw fee for the final/last withdrawal so that no one could abuse this exception to perform any attack that was once not profitable due to the need to pay withdraw fee.

In addition, consider deducting the points once a position is closed or reduced in size so that no one can attempt to open and adjust/close a position repeatedly to obtain more points.