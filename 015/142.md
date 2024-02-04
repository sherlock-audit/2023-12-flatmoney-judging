Orbiting Cyan Finch

medium

# StableModule.stableCollateralPerShare may return 0 in edge case

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