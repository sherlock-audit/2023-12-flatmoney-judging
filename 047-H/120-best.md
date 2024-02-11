Scruffy Parchment Llama

high

# High - UNIT LPs can have tokens drained by leverage traders

## Summary
Flatmoney protocol uses a lot of code forked from Synethix Perps smart contracts, which allow users to hold leveraged positions on a variety of assets. Synethix's leverage positions are denominated in their stablecoin, sUSD, which is pegged to a dollar. Users can close their position and mint and burn sUSD as needed, on the other hand, Flatmoney traders are paid out in rETH. 

## Vulnerability Detail
The structure of the Flatmoney protocol allows for users to open a leveraged position then close it with potentially more rETH than they entered with. The issue with this is that the rETH they get to exit with has been deposited by another user that is currently acting as a UNIT LP. So while the value in USD of the remaining rETH may equal what the user deposited at the time that a leveraged trader closed their position, this is absolutely subject to change and is completely dependent on the price of rETH. 

![image](https://github.com/sherlock-audit/2023-12-flatmoney-RohanNero/assets/100052099/16f61159-2409-49ff-935b-18af14bb4a93)

I know the protocol team has commented on this issue and is aware of it, but I still think it should be considered a vulnerability because of how easy it is for users to lose tokens, and because the users holding these tokens were looking for a safe haven from price volatility in a stablecoin alternative that protects them against traditional inflation. 

For example, consider this at a price of $1000:
1. Alice deposits 100 rETH using the `StableModule` to mint UNIT tokens.
2. Bob deposits 100 rETH using the `LeverageModule` to open a position with 2x leverage.
3. 100% price increase occurs.
4. Bob closes his position with a profit of $300,000 since his position is now worth $400,000 and he borrowed $100,000 originally. In other words, Bob is sent 150 rETH.
5. Now the protocol contains 50 rETH, the market can easily experience some drawdowns bringing the value back down. At this point if Alice withdraws, she has lost half of her token balance.

## Impact
Users who have deposited their rETH into the protocol to mint UNIT tokens and become an LP can have their tokens drained/taken by leverage traders who've made a profit easily.

## Code Snippet
```solidity
function test_drain_alice_with_leverage() public {
        vm.startPrank(admin);
        vaultProxy.setSkewFractionMax(10_000e18);
        uint256 stableDeposit = 100e18;
        uint256 aliceBalanceBefore = WETH.balanceOf(alice);      
        uint256 bobBalanceBefore = WETH.balanceOf(bob);
        announceAndExecuteDeposit({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: stableDeposit,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });
        uint aliceDepositBalance = WETH.balanceOf(alice);
        uint tokenId = announceAndExecuteLeverageOpen({
            traderAccount: bob,
            keeperAccount: keeper,
            margin: 100e18,
            additionalSize: 100e18,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });
        uint bobDepositBalance = WETH.balanceOf(bob);
        // Mock price to $2000 (100% increase)
        uint256 newCollateralPrice = 2000e8;
        setWethPrice(newCollateralPrice);
        uint256 stableCollateralPerShareBefore = stableModProxy.stableCollateralPerShare();
        uint256 wethBalanceOfVaultBefore = WETH.balanceOf(address(vaultProxy));
        uint256 stableBalanceOf = stableModProxy.balanceOf(alice);
        uint256 withdrawAmount = stableBalanceOf / 2;
        announceAndExecuteLeverageClose({
            tokenId: tokenId,
            traderAccount: bob,
            keeperAccount: keeper,
            oraclePrice: newCollateralPrice,
            keeperFeeAmount: 0
        });
        announceAndExecuteWithdraw({
            traderAccount: alice,
            keeperAccount: keeper,
            withdrawAmount: withdrawAmount,
            oraclePrice: originalPrice,
            keeperFeeAmount: 0
        });
        uint256 stableCollateralPerShareAfter = stableModProxy.stableCollateralPerShare();
        uint256 wethBalanceOfVaultAfter = WETH.balanceOf(address(vaultProxy));
        console2.log(aliceBalanceBefore); // Started with 100,000
        console2.log(afterDepositBalance); // Had 999,899.99 after depositing 100
        console2.log(WETH.balanceOf(alice)); // After withdrawing entire balance, Alice is left with 99,949.99
        console2.log(bobBalanceBefore); // Started with 100,000
        console2.log(bobDepositBalance); // Had 999,899.99 after opening position with 100
        console2.log(WETH.balanceOf(bob)); // After closing position bob is left with 100,049.99
        // Assert Alice's current balance = her initial balance - (half of her deposit + fees for executing two orders with keepers)
        assertEq(WETH.balanceOf(alice), aliceBalanceBefore - (stableDeposit / 2) - mockKeeperFee.getKeeperFee() * 2);
    }
```

[Synthetix](https://github.com/Synthetixio/synthetix/blob/cbd8666f4331ee95fcc667ec7345d13c8ba77efb/contracts/PerpsV2MarketBase.sol#L272C5-L275C6) `_profitLoss()`:

```solidity
function _profitLoss(Position memory position, uint price) internal pure returns (int pnl) {
        int priceShift = int(price).sub(int(position.lastPrice));
        return int(position.size).multiplyDecimal(priceShift);
    }
```

[Flatmoney](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/libraries/PerpMath.sol#L175) `_profitLoss()`:

```solidity
function _profitLoss(FlatcoinStructs.Position memory position, uint256 price) internal pure returns (int256 pnl) {
        int256 priceShift = int256(price) - int256(position.lastPrice);
        int256 profitLossTimesTen = (int256(position.additionalSize) * (priceShift) * 10) / int256(price);

        if (profitLossTimesTen % 10 != 0) {
            return profitLossTimesTen / 10 - 1;
        } else {
            return profitLossTimesTen / 10;
        }
    }
```

## Tool used

Manual Review

## Recommendation
Restructure how profit withdrawals/redemptions are handled to prevent users acting as UNIT LPs from losing their initial value OR amount no matter what, potentially implementing a system more similar to the Synethix Perps market that some of the code was forked from. This way users who have profited off a leveraged position can still claim their profits, whether it be in the form of an asset like sUSD or not, and users who have deposited rETH for UNIT tokens, can always withdraw tokens that are in value or amount at least equal to their initial deposit. Otherwise, the current protocol risk is way greater than other existing stablecoins and will likely result in users, who just wanted to retain their bag's value with no/low yield, losing their money.