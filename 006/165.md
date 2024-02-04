Loud Linen Eagle

medium

# The end time of the order's executability age is calculated incorrectly.

## Summary
The end time of order's executability age is calculated incorrectly, the result is that the order's executability age gets unexpectedly extended.

## Vulnerability Detail
The ending time of order's executability is calculated by `executableAtTime + vault.maxExecutabilityAge()`, which is wrong.
```solidity
650:    function _prepareExecutionOrder(address account, uint256 executableAtTime) internal {
651:->      if (block.timestamp > executableAtTime + vault.maxExecutabilityAge()) revert FlatcoinErrors.OrderHasExpired();
652:
653:        // Check that the minimum time delay is reached before execution
654:        if (block.timestamp < executableAtTime) revert FlatcoinErrors.ExecutableTimeNotReached(executableAtTime);
655:
656:        // Delete the order tracker from storage.
657:        delete _announcedOrder[account];
658:    }
```
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L650-L658


As shown in [FlatcoinVault.sol#L30-L34](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L30-L34), `minExecutabilityAge` and `maxExecutabilityAge` are both the amount of time expired between trade *announcement* and *execution*.
```solidity
30:    /// @notice The minimum time that needs to expire between trade announcement and execution.
31:    uint64 public minExecutabilityAge;
32:
33:    /// @notice The maximum amount of time that can expire between trade announcement and execution.
34:    uint64 public maxExecutabilityAge;
```
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L30-L34

While order's `executableAtTime` is calculated by announcement block timestamp plus `minExecutabilityAge` ([DelayedOrder.sol#L646]()), its ending time should be `executableAtTime + vault.maxExecutabilityAge() - vault.minExecutabilityAge()` instead.
```solidity
Function: _prepareAnnouncementOrder
646:        executableAtTime = uint64(block.timestamp + vault.minExecutabilityAge());
```
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L646


## Impact
Order's executability age gets unexpectedly extended.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L650-L658

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L30-L34

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L646

## Tool used

Manual Review

## Recommendation
In function `_prepareExecutionOrder`, calculate the ending executability age of a order as `executableAtTime + vault.maxExecutabilityAge() - vault.minExecutabilityAge()`.