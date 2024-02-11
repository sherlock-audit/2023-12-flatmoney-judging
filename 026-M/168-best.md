Loud Linen Eagle

high

# User's FMPs may get locked forever when `unlockTaxVest` gets decreased.

## Summary
Owner can change `unlockTaxVest`, which can affect the calculation of `unlockTax`. However the unlock tax calculation system does not work well in such situation, and may cause user's FMP to be locked in the contract forever.

## Vulnerability Detail
When minting FMPs for a user, the FMPs are locked for a period of time as vesting period (namely `unlockTaxVest`). The unlock time is calculated as `block.timestamp + unlockTaxVest` (To make it simple, we assume that the user has not own FMP before).
```solidity
142:    function _setMintUnlockTime(address account, uint256 mintAmount) internal returns (uint256 newUnlockTime) {
143:        uint256 lockedAmount = _lockedAmount[account];
144:        uint256 unlockTimeBefore = unlockTime[account];
145:
146:        if (unlockTimeBefore <= block.timestamp) {
147:->          newUnlockTime = block.timestamp + unlockTaxVest;
148:        } else {
149:            uint256 newUnlockTimeAmount = (block.timestamp + unlockTaxVest) * mintAmount;
150:            uint256 oldUnlockTimeAmount = unlockTimeBefore * (lockedAmount - mintAmount);
151:            newUnlockTime = (newUnlockTimeAmount + oldUnlockTimeAmount) / lockedAmount;
152:        }
153:
154:->      unlockTime[account] = newUnlockTime;
155:    }
```
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L142-L155

If the user wants to unlock his FMPs before the unlocking time, some of his FMPs are transfered to treasury as `unlockTax`. The `unlockTax` is calculated as `(unlockingTime - block.timestamp) / unlockTaxVest`. Notice that `unlockingTime` is set during minting and will not change thereafter (without new mints for the user), but `unlockTaxVest` may change if owner changes it. This may lead to unexpected `unlockTax`.
```solidity
120:    function getUnlockTax(address account) public view returns (uint256 unlockTax) {
121:        if (unlockTime[account] <= block.timestamp) return 0;
122:
123:->      uint256 timeLeft = unlockTime[account] - block.timestamp;
124:
125:->      unlockTax = timeLeft._divideDecimal(unlockTaxVest);
126:
127:->      assert(unlockTax <= 1e18);
128:    }
```
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L120-L128

**POC**:
1. Let's assumet `unlockTaxVest` is 12 months.
2. Alice deposits some collaterals and gets 200 FMPs, these FMPs' unlock time is 12 months.
3. 6 months later, owner changes `unlockTaxVest` to 5 months.
4. Alice wants to `unlock` her FMPs, the `unlockTax` is first calculated as `(12 months - 6 months) / 5 months = 1.2e18`, which will revert as `assert(unlockTax <= 1e18)` ([L127](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L127)). Alice will not be able to unlock her FMP.


## Impact
User's FMP get locked in the contract forever due to descresed `unlockTaxVest`.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L142-L155

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L120-L128

## Tool used

Manual Review

## Recommendation
Save the `unlockTaxVest` with `unlockTime` for every user at minting time, and use it instead of the `unlockTaxVest` state variable to calculate the `unlockTax`. Remember to update user's `unlockTaxVest` when new minting's `unlockTaxVest` is different from the saved one.