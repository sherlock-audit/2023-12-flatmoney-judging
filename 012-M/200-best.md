Formal Eggshell Sidewinder

medium

# Unlock of points can be delayed infinitely

## Summary

The unlock of points can be delayed infinitely as long as the users interact with the protocol frequently. As a result, points that have already been unlocked (already passed the `unlockTaxVest`) will be locked up again, preventing users from exchanging their points for something of value.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L142

```solidity
File: PointsModule.sol
139:     /// @notice Sets the unlock time for newly minted points.
140:     /// @dev    If the user has existing locked points, then the new unlock time is calculated based on the existing locked points.
141:     ///         The newly minted points are included in the `lockedAmount` calculation.
142:     function _setMintUnlockTime(address account, uint256 mintAmount) internal returns (uint256 newUnlockTime) {
143:         uint256 lockedAmount = _lockedAmount[account];
144:         uint256 unlockTimeBefore = unlockTime[account];
145: 
146:         if (unlockTimeBefore <= block.timestamp) {
147:             newUnlockTime = block.timestamp + unlockTaxVest;
148:         } else {
149:             uint256 newUnlockTimeAmount = (block.timestamp + unlockTaxVest) * mintAmount;
150:             uint256 oldUnlockTimeAmount = unlockTimeBefore * (lockedAmount - mintAmount);
151:             newUnlockTime = (newUnlockTimeAmount + oldUnlockTimeAmount) / lockedAmount;
152:         }
153: 
154:         unlockTime[account] = newUnlockTime;
155:     }
```

Based on the above code, the unlock time can be continuously delayed by normal user actions:

1. **Initial Mint**: A user mints 100 tokens at time `t0`. The initial unlock time is set to `t0 + 360 days` (360 days being the `unlockTaxVest` period).

2. **Additional Mint after 180 Days**: The user mints an additional 50 tokens 180 days after the initial mint, the new unlock time will be as follows:

```solidity
newUnlockTimeAmount = (block.timestamp + unlockTaxVest) * mintAmount;
newUnlockTimeAmount = ((T0 + 180) + 360) * 50;
newUnlockTimeAmount = (T0 + 540) * 50;

oldUnlockTimeAmount = (unlockTimeBefore) * (lockedAmount - mintAmount);
oldUnlockTimeAmount = (T0 + 360) * (150 - 50);
oldUnlockTimeAmount = (T0 + 360) * 100;

newUnlockTime = (newUnlockTimeAmount + oldUnlockTimeAmount) / lockedAmount;
newUnlockTime = [(T0 + 540) * 50 + (T0 + 360) * 100] / 150
newUnlockTime = (3T0 + 1260) / 3
newUnlockTime = (T0 + 420)
```

The new unlock date (T0 + 420) is 60 days later than the initial one (T0 + 360).

Therefore, each subsequent minting of tokens can extend the unlock period. This can lead to a situation where users who frequently mint additional tokens or interact with the protocol may continuously push back their unlock time.

The only way to unlock the point entirely is to stop using the protocol for 360 days so as to prevent the unlock date from being pushed further.

## Impact

Points that have already been unlocked (already passed the `unlockTaxVest`) will be locked up again, preventing users from exchanging their points for something of value

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L142

## Tool used

Manual Review

## Recommendation

Consider not delaying the unlock date for existing points when new points are minted. Implement a more sophisticated approach to tracking the unlock date of the points minted at different points in time.