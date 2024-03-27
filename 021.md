Dizzy Cream Kitten

high

# Some power token holders(other than  initial accounts defined in  PowerBootstrapToken) will lose previous tokens when calling the function buy(powertoken contract).

## Summary
If a power token holder is not an initial token holder( initial accounts defined in  PowerBootstrapToken) but the user has power tokens in his account,he will lose those previous power tokens when calling the  function buy(powertoken contract).


## Vulnerability Detail
1. Let’s assume, Alice has 1000 power tokens and she is not the initial token holder.

2. Alice calls the function buy(contract PowerToken) for buying 1500 power tokens.

3. This buy function calls _mint function which leads to call _sync function which leads to call function _bootstrap.

4. See function _bootstrap, as alice is not initial account,so alice’s bootstrapBalance = 0,now alice’s balance and voting power will be rewritten/updated to 0 i.e   _balances[account_].push(AmountSnap(bootstrapEpoch, 0));
        _votingPowers[account_].push(AmountSnap(bootstrapEpoch, 0));

5. So Alice's previous balance was 1000 but now the balance is 0. After alice is minted 1500 power tokens , now alice’s balance is 1500, but alice balance should be 1500+1000 = 2500.

## Impact
Some power token holders(other than  initial accounts defined in  PowerBootstrapToken) will lose previous tokens when calling the function buy(powertoken contract).

## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L272
## Tool used

Manual Review

## Recommendation
