Energetic Brown Cougar

high

# Liqudity providers effectively lost their Rocket pool staking rewards while integrating with the protocol

## Summary
If we treat betting on price of crypto assets such as BTC/ETH as a ````50:50```` win game, then betting on price of interest bearing asset such as stETH/rETH is more likely a ````55:45```` game. As the long side has an inherent advantage, which would cause liqudity providers of Flatcoin effectively lost their Rocket pool staking rewards while integrating with the protocol.

## Vulnerability Detail
Let's say the initial states are
```solidity
initRatio = 1.1 ETH/rETH
initPriceOfETH = $2000
initPriceOfrETH = $2200
Alice.LongPostitionsSize = 1rETH
Alice.EntryPrice = $2200
```
Some time later, the ratio increases to ````1.2```` due to Rocket pool's staking reward, and the ETH price keeps same, then we get
```solidity
ratio = 1.2ETH/rETH
newPriceOfETH = $2000
newPriceOfrETH = $2400
Alice.PnL = 1rETH * (newPriceOfrETH - initPriceOfrETH) / newPriceOfrETH = 1 * (2400 - 2200) / 2400 = 0.083rETH
Alice.PnLInUSD = 0.083rETH * $2400 = $200
```
We can see the long side Alice happens to win the $200 Rocket pool's staking reward while ETH price keeps steady. And Liqudity providers of Flatcoin is the losing side. They are effectively losing their Rocket pool staking rewards while betting with traders on rETH's price.

## Impact
Liqudity providers effectively lost their Rocket pool staking rewards.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/StableModule.sol#L61


## Tool used

Manual Review

## Recommendation
Not 100% sure if replacing with ETH oracle price can work with rETH as collateral, but looks like it's an option.
