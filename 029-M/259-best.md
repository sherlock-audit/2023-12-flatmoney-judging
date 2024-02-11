Sneaky Flaxen Rhino

medium

# Calculations in Keeperfee are too crude, making users pay more KeeperFee.

## Summary

In `KeeperFee.sol` , `KeeperFee` is calculated on a very rough estimate, making users pay more additional fees.

## Vulnerability Detail

In `KeeperFee.sol` , `getKeeperFee()' would return `KeeperFee` in collateral:

    function getKeeperFee() public view returns (uint256 keeperFeeCollateral) {

        ...    //get oracle feed and do some validation

        uint256 gasPriceL2 = _gasPriceOracle.gasPrice();
        uint256 overhead = _gasPriceOracle.overhead();
        uint256 l1BaseFee = _gasPriceOracle.l1BaseFee();
        uint256 decimals = _gasPriceOracle.decimals();
        uint256 scalar = _gasPriceOracle.scalar();

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

Currently, Transaction Fees on `OP STACK` chains are composed of an [Execution Gas Fee](https://docs.optimism.io/stack/transactions/fees#execution-gas-fee) and an [L1 Data Fee](https://docs.optimism.io/stack/transactions/fees#l1-data-fee). The total cost of a transaction is the sum of these two fees.

Since L1 fees often account for more than 80% of a transaction, we will only discuss L1 fees here.

According to [OP doc](https://docs.optimism.io/stack/transactions/fees#bedrock), 

the L1 Data Fee is calculated based on the following parameters:

- The RLP-encoded signed transaction.
- The current Ethereum base fee (trustlessly relayed from Ethereum).
- A fixed overhead cost for publishing a transaction (currently set to 188 gas).
- A dynamic overhead cost which scales with the size of the transaction (currently set to 0.684).
- 
The L1 Data Fee calculation first begins with counting the number of zero bytes and non-zero bytes in the transaction data. Each zero byte costs 4 gas and each non-zero byte costs 16 gas. This is the same way that Ethereum calculates the gas cost of transaction data.

    tx_data_gas = count_zero_bytes(tx_data) * 4 + count_non_zero_bytes(tx_data) * 16

After calculating the gas cost of the transaction data, the fixed and dynamic overhead values are applied.

    tx_total_gas = (tx_data_gas + fixed_overhead) * dynamic_overhead

Finally, the total L1 Data Fee is calculated by multiplying the total gas cost by the current Ethereum base fee.

    l1_data_fee = tx_total_gas * ethereum_base_fee

But in current `Keeperfee.sol`, 

    costOfExecutionGrossEth = ((((_gasUnitsL1 + overhead) * l1BaseFee * scalar) / 10 ** decimals) + (_gasUnitsL2 * gasPriceL2));

The `tx_data_gas` is just estimated to `_gasUnitsL1`. So no matter what transaction Keepers perform, users will always pay Keepers the same amount of gas.

## Impact

Because `KeeperFee` should be able to cover all kinds of executions in terms of gas, this crude approximation will force users to pay more for orders that actually consume relatively little gas, such as `executewithdraw`.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/KeeperFee.sol#L104-L105

## Tool used

Manual Review

## Recommendation

Use something like `payExecutionFee` in [GMX](https://github.com/gmx-io/gmx-synthetics/blob/main/contracts/gas/GasUtils.sol) to refund users for their additional cost.
