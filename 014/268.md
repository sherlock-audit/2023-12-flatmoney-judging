Sparkly Rouge Goose

high

# Reentrancy attack can be done in mint function due to not following CEI pattern

## Summary
The _mint function does not follow the CEI( Check Effect Interaction) pattern, hence its vulnerable to the reentrancy attack.

## Vulnerability Detail
```
File:src/LeverageModule.sol
function _mint(address _to) internal returns (uint256 _tokenId) {
        _tokenId = tokenIdNext;

        _safeMint(_to, tokenIdNext);

        tokenIdNext += 1;
    }
```
Here CEI pattern is not followed, first mint is done and then only the tokenIdNext is upgraded. This can result in reentrancy if the mint function transfers the call to the user such as via hooks or any other way.This would allow a malicious attacker to call mint() again and again.

## Impact
As attacker can mint the token again and again there can be the problem in the token id indexing.
## Code Snippet
```
File:src/LeverageModule.sol
438. function _mint(address _to) internal returns (uint256 _tokenId) {
        _tokenId = tokenIdNext;

        _safeMint(_to, tokenIdNext);

        tokenIdNext += 1;
    }
```
## Tool used

Manual Review

## Recommendation
It is advised to follow checks effects interaction pattern.