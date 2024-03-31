Droll Iron Ferret

medium

# Minters that are earning and active are unable to burn their full balance of owed M

## Summary

Rounding error causes the owed M balance to be greater than the earning M balance, causing a revert when attempting to burn the full owed balance.

**Note:** A similar issue also exists in `MToken._transfer`, wherein the earning amount subtracted is the rounded up principal amount, which could also trigger a revert.

## Vulnerability Detail

When you mint M as an active minter that is also earning, we first compute the principal amount of owed M, which we round up to benefit the protocol:

```solidity
principalAmount_ = _getPrincipalAmountRoundedUp(amount_);
...
_rawOwedM[msg.sender] += principalAmount_;
```

We then compute the principal amount of earning M, which we round down to benefit the protocol:

```solidity
_addEarningAmount(recipient_, _getPrincipalAmountRoundedDown(safeAmount_));
```

At this point, the minter will likely have an owed M amount which is 1 greater than their earning amount.

The problem with this is when we go to burn the full amount of owed M, we try to subtract the rounded up principal amount:

```solidity
_subtractEarningAmount(account_, _getPrincipalAmountRoundedUp(UIntMath.safe240(amount_)));
```

Since the rounded up principal is likely greater than the rounded down value we used to store the earning amount, this will revert because the `principalAmount > rawBalance`:

```solidity
if (rawBalance_ < principalAmount_) revert InsufficientBalance(account_, rawBalance_, principalAmount_);
```

## Impact

Minters that are earning and active are unable to burn their full balance.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MToken.sol#L215

## Tool used

Manual Review

## Recommendation

Consider burning the minimum of the provided amount and the full available amount to burn.