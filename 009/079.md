Quick Corduroy Mouse

medium

# If `updateCollateralInterval` is 0, minters are penalized for undercollateralization with each update

## Summary
If the `updateCollateralInterval` value is set to 0, this leads to an incorrect calculation of the collateral amount, and minters get penalized for undercollateralization.

## Vulnerability Detail
When a minter updates the collateral value by calling the `MinterState::updateCollateral` function, there is a check and penaltization for missed updates.

In the `_getMissedCollateralUpdateParameters` function, we observe that if the `updateInterval` value is 0, it is considered as having no missed interval, so no penalties are imposed at all.

```solidity
// If brand new minter or `updateInterval_` is 0, then there is no missed interval charge at all.

if (lastUpdateTimestamp_ == 0 || updateInterval_ == 0) return (0, penalizeFrom_);
```

There is also test coverage for this scenario.

```solidity
function test_getMissedCollateralUpdateParameters_zeroNewUpdateInterval() external {
    (uint40 missedIntervals_, uint40 missedUntil_) = _minterGateway.getMissedCollateralUpdateParameters({
        lastUpdateTimestamp_: uint40(block.timestamp) - 48 hours, // This does not matter
        lastPenalizedUntil_: uint40(block.timestamp) - 24 hours, // This does not matter
        updateInterval_: 0
    });

    assertEq(missedIntervals_, 0);
    assertEq(missedUntil_, uint40(block.timestamp) - 24 hours);
}
```

After this check, the collateral is updated, and a new `lastUpdateTimestamp_` is set to the current `block.timestamp`.

Finally, the `_imposePenaltyIfUndercollateralized` function is called.

The issue arises after updating `lastUpdateTimestamp`. If `updateCollateralInterval` is 0, the `collateralOf` function starts returning 0 because it checks that:

`block.timestamp >= collateralExpiryTimestampOf(minter_)`

Where the expiry timestamp is the same as the current `block.timestamp`:

`_minterStates[minter_].updateTimestamp + updateCollateralInterval()`

## Impact

Thus, the miner is not penalized for missed updates, but with each update, they get a penalty for undercollateralization, and this penalty is calculated from the entire amount of the collateral.

```solidity
_imposePenalty(minter_, principalOfActiveOwedM_ - principalOfMaxAllowedActiveOwedM_);
```

As it is a low-probability case that can result in a fund loss (high impact), it might be considered a medium issue.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L202-L206
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L539

## Tool used

Manual Review

## Recommendation

```diff
-        if (block.timestamp >= collateralExpiryTimestampOf(minter_)) return 0;
+        if (block.timestamp > collateralExpiryTimestampOf(minter_)) return 0;
```