Mythical Rouge Mule

high

# Mint overflow

## Summary
The `mint()` can overflow in some case and pass the check.

## Vulnerability Detail
In `_mint()`, it check if `newPrincipalOfTotalActiveOwedM_` plus `_getPrincipalAmountRoundedUp(totalInactiveOwedM)` bigger than `type(uint112).max` or not. 
```solidity
        unchecked {
            uint256 newPrincipalOfTotalActiveOwedM_ = uint256(principalOfTotalActiveOwedM_) + principalAmount_;

            // As an edge case precaution, prevent a mint that, if all owed M (active and inactive) was converted to
            // a principal active amount, would overflow the `uint112 principalOfTotalActiveOwedM`.
            if (
                // NOTE: Round the principal up for worst case.
                newPrincipalOfTotalActiveOwedM_ + _getPrincipalAmountRoundedUp(totalInactiveOwedM) >= type(uint112).max
            ) {
                revert OverflowsPrincipalOfTotalOwedM();
            }

            principalOfTotalActiveOwedM = uint112(newPrincipalOfTotalActiveOwedM_);
            _rawOwedM[msg.sender] += principalAmount_; // Treat rawOwedM as principal since minter is active.
        }
        
        IMToken(mToken).mint(destination_, amount_);
```
But because of the `unchecked`, it can overflow in some case and then pass the check. Maybe the case is like that: 
```solidity
pragma solidity =0.8.23;

contract Test {
    uint240 constant UINT240MAX = type(uint240).max;
    uint256 constant UINT256MAX = type(uint256).max;

    function test() public pure{
        unchecked{
            if(UINT256MAX-1000000 + UINT240MAX > UINT240MAX) {
                revert();
            }
        } 
    }
}
```
The `test()` will not revert. The same theory is in `_mint()`,it may overflow and pass the check in some extreme cases, 

## Impact
The totalSupply will be wrong. The caller maybe mint a lot of tokens.

## Code Snippet
- https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MToken.sol#L235-L247
- https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L298-L312
## Tool used

Manual Review

## Recommendation
Don't use unchecked in `_mint()`.