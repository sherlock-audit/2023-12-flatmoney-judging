Sneaky Flaxen Rhino

medium

# `Keeperfee` will be miscalculated after Ecotone upgrade and cannot be upgraded

## Summary

`Keeperfee` will be miscalculated after Ecotone upgrade on OP stack.

## Vulnerability Detail

Base Mainnet transaction fees are composed of an [Execution Gas Fee](https://docs.optimism.io/stack/transactions/fees#execution-gas-fee) and an [L1 Data Fee](https://docs.optimism.io/stack/transactions/fees#l1-data-fee). The total cost of a transaction is the sum of these two fees. 

Prior to the Ecotone upgrade, the L1 Data Fee is calculated based on the following parameters:

- The RLP-encoded signed transaction.
- The current Ethereum base fee (trustlessly relayed from Ethereum).
- A fixed overhead cost for publishing a transaction (currently set to 188 gas).
- A dynamic overhead cost which scales with the size of the transaction (currently set to 0.684).

The L1 Data Fee calculation first begins with counting the number of zero bytes and non-zero bytes in the transaction data. Each zero byte costs 4 gas and each non-zero byte costs 16 gas. This is the same way that Ethereum calculates the gas cost of transaction data.

    tx_data_gas = count_zero_bytes(tx_data) * 4 + count_non_zero_bytes(tx_data) * 16

In current `Keeperfee.sol`, `_gasUnitsL1` is used as an estimate of `tx_data_gas`.

After calculating the gas cost of the transaction data, the fixed and dynamic overhead values are applied.

    tx_total_gas = (_gasUnitsL1 + fixed_overhead) * dynamic_overhead

Finally, the total L1 Data Fee is calculated by multiplying the total gas cost by the current Ethereum base fee.

    l1_data_fee = tx_total_gas * ethereum_base_fee

The above formula is exactly the logic executed by `getKeeperFee`:

    function getKeeperFee() public view returns (uint256 keeperFeeCollateral) {
        ...
        uint256 gasPriceL2 = _gasPriceOracle.gasPrice();
        uint256 overhead = _gasPriceOracle.overhead();
        uint256 l1BaseFee = _gasPriceOracle.l1BaseFee();
        uint256 decimals = _gasPriceOracle.decimals();
        uint256 scalar = _gasPriceOracle.scalar();
        //@Audit Before Ecotone upgrade!
        uint256 costOfExecutionGrossEth = ((((_gasUnitsL1 + overhead) * l1BaseFee * scalar) / 10 ** decimals) +
            (_gasUnitsL2 * gasPriceL2));
        uint256 costOfExecutionGrossUSD = costOfExecutionGrossEth.mulDiv(ethPrice18, _UNIT); // fee priced in USD
        uint256 maxProfitMargin = _profitMarginUSD.max(costOfExecutionGrossUSD.mulDiv(_profitMarginPercent, _UNIT)); // additional USD profit for the keeper
        uint256 costOfExecutionNet = costOfExecutionGrossUSD + maxProfitMargin; // fee priced in USD

        keeperFeeCollateral = (_keeperFeeUpperBound.min(costOfExecutionNet.max(_keeperFeeLowerBound))).mulDiv(
            _UNIT,
            collateralPrice
        ); // fee priced in collateral
    }

But, after the Ecotone upgrade, batch transactions will be sent to L1 as 4844 blobs instead of through L1 calldata. This updated function uses the following parameters:

- The RLP-encoded signed transaction.
- The current Ethereum base fee and/or blob base fee (trustlessly relayed from Ethereum).
- Two new scalar parameters that independently scale the base fee blob base fee.

At the exact point of the Ecotone upgrade, the dynamic overhead parameter value is used to initialize the Ecotone base fee scalar, and blob base fee is set to 0. The overhead parameter from the previous function becomes ignored.

The Ecotone L1 Data Fee calculation begins with counting the number of zero bytes and non-zero bytes in the transaction data. Each zero byte costs 4 gas and each non-zero byte costs 16 gas. This value, when divided by 16, can be thought of as a rough estimate of the size of the transaction data after compression.

    tx_compressed_size = [(count_zero_bytes(tx_data)*4 + count_non_zero_bytes(tx_data)*16)] / 16

Next, the two scalars are applied to the base fee and blob base fee parameters to compute a weighted gas price multiplier.

    weighted_gas_price = 16*base_fee_scalar*base_fee + blob_base_fee_scalar*blob_base_fee

The l1 data fee is then:

    l1_data_fee = tx_compressed_size * weighted_gas_price

Recall that `base_fee_scalar` is set to `dynamic_overhead` and `blob_base_fee_scalar` is 0 immediately following the upgrade. Because the old overhead parameter becomes ignored, new L1 data prices will be lower than before the fork. 

It's worth noting that `KeeperFee.sol` doesn't inherit `ModuleUpgradeable.sol` or any proxy patterns, so it's currently not updatable. Users have to pay more for `keeperfee` than they should after the upgrade.

## Impact

Since the Ecotone upgrade(along with Dencun upgrade) will take place around the time Flatcoin is deployed on BASE mainnet, such error would affect users to pay more Keeperfee.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/KeeperFee.sol#L104-L105

## Tool used

Manual Review

## Recommendation

Let `Keeperfee.sol` inherit `ModuleUpgradeable.sol` so contract could be updated after Ecotone upgrade.
