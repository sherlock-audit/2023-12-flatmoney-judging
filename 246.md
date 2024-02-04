Sneaky Flaxen Rhino

high

# Keepers can be forced to waste gas with a modified `onERC721Received()`

## Summary

The `onERC721Received()` function of the contract address is called when the keeper tries to mint a new position to the user by `executeOpen`. This allows a malicious user to frontrun keeper's transactions to self-destruct the contract and deploy new malicious bytecode to grief the keeper.

## Vulnerability Detail

When calculating the value of `Keeperfee`, `_gasUnitsL2` and `_gasUnitsL1` are used as an estimate of the gas required to execute the order:

    uint256 costOfExecutionGrossEth = ((((_gasUnitsL1 + overhead) * l1BaseFee * scalar) / 10 ** decimals) + (_gasUnitsL2 * gasPriceL2));

So, Keeper does not get paid more for consuming more gas.

Consider such attack path:

1. Attacker set up an normal contract wallet with `onERC721Received()` and self-destruct function.
2. Attacker `announceOpen` a position, start listening for keeper transactions.
3. A Keeper notice the order and calculated it to be a profitable deal, send the tx into mempool.
4. Attacker frontrun the order, self-destruct his contract on the address and re-deploy one with malicious `onERC721Received()`, which execute complex logic until the gaslimit is reached.
5. Attacker repeat the above steps until the Keeper's ETH balance is drained.

## Impact

Since keeper is automatically executed by the script, attacker can drain **ALL Keeper**'s balance through the above attack path at 0 cost.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L111
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/KeeperFee.sol#L104-L105

## Tool used

Manual Review

## Recommendation

Consider using something like [GasUtils](https://github.com/gmx-io/gmx-synthetics/blob/main/contracts/gas/GasUtils.sol) on GMX to validate execution Gas and make sure it is not too big.
