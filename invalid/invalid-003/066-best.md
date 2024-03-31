Young Umber Okapi

high

# A portion of the user's bootstrapped balance is lost

## Summary

A portion of the user's bootstrapped balance is lost after a reset.

## Vulnerability Detail

Assume the following example. The values chosen are intentionally simplified and magnified to demonstrate the issue, but this does not affect the validity of this issue.

- Two power token holders (Alice and Bob), each of them own 50 Power Tokens (PT).
- Inflation = 50%
- Both Alice and Bob delegate all their voting power to Charles. Charles has a total of 100 voting power.
- Total supply of PT = 100

![image-20240319185624715](https://github.com/sherlock-audit/2023-10-mzero-xiaoming9090/assets/102820284/cd5d1803-93eb-4843-8e46-83449adf589b)

Near the end of Epoch 99 (Voting Phase), Charles voted for all proposals. Since Charles' voting power is 100, an inflation of 50% will be 50 (`inflation_ = 50`). In Line 117 below, the total supply will be 150 after inflation of 50 is added.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/abstract/EpochBasedInflationaryVoteToken.sol#L105

```solidity
File: EpochBasedInflationaryVoteToken.sol
105:     function _markParticipation(address delegatee_) internal onlyDuringVoteEpoch {
106:         uint16 currentEpoch_ = _clock();
107: 
108:         // Revert if could not update, as it means the delegatee has already participated in this epoch.
109:         if (!_update(_participations[delegatee_], currentEpoch_)) revert AlreadyParticipated();
110: 
111:         _sync(delegatee_);
112: 
113:         uint240 inflation_ = _getInflation(_getVotes(delegatee_, currentEpoch_));
114: 
115:         // NOTE: Cannot sync here because it would prevent `delegatee_` from getting inflation if their delegatee votes.
116:         // NOTE: Don't need to sync here because participating has no effect on the balance of `delegatee_`.
117:         _addTotalSupply(inflation_);
118:         _addVotingPower(delegatee_, inflation_);
119:     }
```

One epoch later (Epoch 100), a reset to power holders is triggered, and a new Power Token contract is deployed with the previous Power Token contract as the bootstrap. At Line 106, the `bootstrapEpoch` is set to the previous epoch (Epoch 99). At Line 107, the `bootstrapSupply_` is set to the total supply of the previous epoch, which is `150`.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L106

```solidity
File: PowerToken.sol
095:     constructor(
096:         address bootstrapToken_,
097:         address standardGovernor_,
098:         address cashToken_,
099:         address vault_
100:     ) EpochBasedInflationaryVoteToken("Power Token", "POWER", 0, ONE / 10) {
101:         if ((bootstrapToken = bootstrapToken_) == address(0)) revert InvalidBootstrapTokenAddress();
102:         if ((standardGovernor = standardGovernor_) == address(0)) revert InvalidStandardGovernorAddress();
103:         if ((_nextCashToken = cashToken_) == address(0)) revert InvalidCashTokenAddress();
104:         if ((vault = vault_) == address(0)) revert InvalidVaultAddress();
105: 
106:         uint16 bootstrapEpoch_ = bootstrapEpoch = (_clock() - 1);
107:         uint256 bootstrapSupply_ = IEpochBasedVoteToken(bootstrapToken_).pastTotalSupply(bootstrapEpoch_);
108: 
109:         if (bootstrapSupply_ == 0) revert BootstrapSupplyZero();
110: 
111:         if (bootstrapSupply_ > type(uint240).max) revert BootstrapSupplyTooLarge();
112: 
113:         _bootstrapSupply = uint240(bootstrapSupply_);
114: 
115:         _addTotalSupply(INITIAL_SUPPLY);
```

When Alice or Bob triggers the sync function of the new Power Token contract, the following function will be triggered:

```solidity
_bootstrap() > _getBootstrapBalance(account_, bootstrapEpoch) > 

(IEpochBasedVoteToken(bootstrapToken).pastBalanceOf(account_, bootstrapEpoch) * INITIAL_SUPPLY) /_bootstrapSupply

(IEpochBasedVoteToken(bootstrapToken).pastBalanceOf(account_, Epoch_99) * 10_000) / 150

(50 * 10_000) / 150 = 3333
```

The issue is that in Epoch 99, both Alice and Bob's balance is 50. When Alice or Bob bootstraps their account after the reset, they will only receive 3333 PT (`50/150 * 10000`) instead of 5000 (Missing 1667 PT). This translates to a total of 1/3 of the initial supply being lost. Alice and Bob both lost 1667 PT each.

Alice or Bob could attempt to perform a sync on the old Power Token contract on Epoch 100 to increase their balance. However, it will not resolve this issue because the newly gained balance is added to the current epoch (Epoch 100) instead of the bootstrap epoch (Epoch 99) in the previous Power Token contract. Only the bootstrap epoch (99) is relevant after a reset.

## Impact

Loss of power tokens for the affected users.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/abstract/EpochBasedInflationaryVoteToken.sol#L105

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L106

## Tool used

Manual Review

## Recommendation

During bootstrapping after a reset, the distribution of the initial supply should take into consideration of users' unrealized inflation.