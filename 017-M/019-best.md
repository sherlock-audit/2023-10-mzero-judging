Droll Iron Ferret

medium

# EIP-5805 non-compliance

## Summary

EpochBasedVoteToken and EpochBasedInflationaryToken fail to comply with the inherited ERC5805 standard.

## Vulnerability Detail

EpochBasedVoteToken and EpochBasedInflationaryToken inherit ERC5805 but are non-compliant in two ways:

1. The EIP states: "Tokens that are delegated to address(0) should not be tracked." However, this is not the case. Instead, tokens that are delegated to address(0) are self-delegated:

```solidity
// `delegatee_` will be `delegator_` (the default) if `delegatee_` was passed in as `address(0)`.
address newNonZeroDelegatee_ = _getDefaultIfZero(newDelegatee_, delegator_);
```

```solidity
function _getDefaultIfZero(address input_, address default_) internal pure returns (address) {
    return input_ == address(0) ? default_ : input_;
}
```

2. The EIP also states: "For all accounts `a != 0` and all timestamp `t < clock`, `getPastVotes(a, t)` SHOULD be the sum of the “balances” of all the accounts that delegated to `a` when `clock` overtook `t`." This is not the case with EpochBasedInflationaryVoteToken since unrealized inflation must first be manually realized by calling `_sync` to update past balances:

```solidity
function _sync(address account_) internal virtual {
    // Realized the account's unrealized inflation since its last sync.
    _addBalance(account_, _getUnrealizedInflation(account_, _clock()));
}
```

## Impact

Failure to comply to EIP's can lead to unexpected effects.

## Code Snippet

- https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/abstract/EpochBasedVoteToken.sol#L185
- https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/abstract/EpochBasedInflationaryVoteToken.sol#L135-L138

## Tool used

Manual Review

## Recommendation

Comply to the EIP specifications or otherwise clearly indicate that the contract is not fully ERC5805 compliant.