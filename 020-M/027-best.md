Dizzy Cream Kitten

medium

# There will always be less supply of M tokens than owedM(active/inactive) in the protocol.

## Summary

## Vulnerability Detail
See  function mintM, principalAmount_ = _getPrincipalAmountRoundedUp(amount_);

In function _getPrincipalAmountRoundedUp(amount_), amount(minted M) are divided by currentindex , so owed M( (    _rawOwedM[msg.sender] += principalAmount_;)
for users is less than actual minted  M.

When users calls function burnM which calls the function _repayForActiveMinter, 
See function _repayForActiveMinter, 
Here  amount_ = _getPresentAmount(principalAmount_);principal amount is multiplied by currentindex, as now current index is increased more than previous index(during minting). So users have to burn(pay) more M tokens(more M tokens than actual minting M+penalty M).

When  function updateIndex is called,   excessOwedM(penalty M) are minted.

There are 3 times when  owedM increases than totalsupply of M tokens. Those are:
1. Imposes penalty if minter missed collateral updates.
. 
2. Imposes penalty if minter is undercollateralized.


4. When users are minted M tokens, then _rawOwedM for minter is less than actual minted M as minted M is divided by currentindex.but when Burning(repay) M tokens,then  _rawOwedM[minter] is multiplied by currentindex, now currentindex is larger than previous currentindex(during minting M), so users have to pay more M tokens.

But see the function excessOwedM, only 1 and 2 scenarios(above explained) are accounted for excessowedM. 3 no scenario( explained above) is not accounted when minting excessowedM in   function updateIndex.so there will be always less supply of M tokens than owed M in the protocol.

## Impact
There will always be less supply of M tokens than owedM(active/inactive) in the protocol.so some users may not pay full owed M.

## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L435

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L801
## Tool used

Manual Review

## Recommendation
Apply the 3 no scenario(explained above) when minting excessowed M tokens. 
