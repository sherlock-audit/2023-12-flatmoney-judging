Formal Eggshell Sidewinder

high

# Malicious users can position themselves ahead of liquidation to profit from the price increase

## Summary

Malicious users can position themselves ahead of liquidation to profit from the price increase. As a result, malicious users could siphon the gains of the benign LPs, leading to the loss of assets for the victims.

## Vulnerability Detail

Liquidation of large positions creates an opportunity for LPs to position themselves ahead of a liquidation and potentially profit from an increase in price (stableCollateralPerShare)

Assuming that there is a large position called $P_x$ that is on the verge of being liquidated. If the $P_x$​ gets liquidated, it is expected that a significant amount of remaining margins within this position go to the LPs, as shown in Line 130 below

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
130:             // Adjust the stable collateral total to account for user's remaining margin.
131:             // If the remaining margin is greater than 0, this goes to the LPs.
132:             // Note that {`remainingMargin` - `profitLoss`} is the same as {`marginDeposited` + `accruedFunding`}.
133:             vault.updateStableCollateralTotal(int256(remainingMargin) - positionSummary.profitLoss);
```

When the remaining margin goes to the LPs, the `stableCollateralPerShare` will increase.

Bob (a malicious user) could announce a deposit order with a large amount of rETH. To prevent other keepers from executing his order, he intentionally set the keeper fee to zero so that no one would be incentivized to execute it.

In the test scripts, the `minExecutabilityAge` is set to 10 seconds, and `maxExecutabilityAge` is set to 1 minute. Thus, Bob's order will be valid for execution for 1 minute (30 blocks) after the initial 10-second (5 blocks) wait, within $T1$ and $T2$​ as shown below.

When $P_x$ is liquidated at any point of time within $T1$ and $T2$​, Bob will front-run the liquidation TX, and position his deposit order execution TX in front of the liquidation TX. Let's assume that Bob's deposit will result in Bob owning half of the shares in the vault. As a result, 50% of the gain (from the remaining margin) transferred to LPs after the liquidation will belong to Bob.

This is a zero-sum game. If Bob has not carried out this transaction, all the gain will go to the LPs. In this case, due to the planned timing of Bob's transactions, he managed to obtain most of the gains here. Bob can proceed to withdraw his shares/UNITs to realize his profit. As long as the gain is larger than the withdrawal fee, this is profitable for Bob. Bob should have done some math beforehand to ensure that he only targets the liquidation of positions that are profitable after accounting for the withdrawal fee.

![image](https://github.com/sherlock-audit/2023-12-flatmoney-xiaoming9090/assets/102820284/135a4e0e-6fba-44c4-b0c3-58a267e27c1b)

If the liquidation of $P_x$ does not occur within $T1$ and $T2$​​, Bob could simply cancel the order after it expires, and he will get back the full deposit amount without incurring any fee. Gas fees are extremely cheap on L2 (Base), where FlatCoin is deployed on. Thus, gas cost would not have a significant impact on the profitability of this attack.

If there are a number of such positions on the verge of being liquidated, he could repeatedly announce and cancel his deposit order for an extended amount of time to wait for the opportunity to front-run them.

## Impact

Malicious users could siphon the gains of the benign LPs, leading to the loss of assets for the victims.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

## Tool used

Manual Review

## Recommendation

To prevent the opportunistic ordering of transactions surrounding liquidations such as the one described in this report, SNX adopts the following solutions:

- A two-stage liquidation process (https://sips.synthetix.io/sips/sip-2005/)
- Forced liquidation for liquidating positions that can cause a significant price impact on the system if it is liquidated. These positions can only be liquidated by approved accounts