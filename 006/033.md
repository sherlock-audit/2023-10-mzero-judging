Amusing Wooden Dragon

medium

# An earner can still continue earning even after being removed from the approved list.

## Summary
An earner can still continue earning even after being removed from the approved list.

## Vulnerability Detail
A `M` holder is eligible to earn the `Earner Rate` when they are approved by TTG.  The approved `M` holder can call [`startEarning()`](https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MToken.sol#L100-L103) then begin to earn the `Earner Rate`. They also can [`stopEarning()`](https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MToken.sol#L106-L108) to quit earning. 

However, when an approved `M` holder is disapproved, only the disapproved holder themselves can choose to stop earning; no one else has the authority to force them to quit earning.

`Earner Rate` is calculated in [`StableEarnerRateModel#rate()`](https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/rateModels/StableEarnerRateModel.sol) as below:
```solidity
    function rate() external view returns (uint256) {
        uint256 safeEarnerRate_ = getSafeEarnerRate(
            IMinterGateway(minterGateway).totalActiveOwedM(),
            IMToken(mToken).totalEarningSupply(),
            IMinterGateway(minterGateway).minterRate()
        );

        return UIntMath.min256(maxRate(), (RATE_MULTIPLIER * safeEarnerRate_) / ONE);
    }

    function getSafeEarnerRate(
        uint240 totalActiveOwedM_,
        uint240 totalEarningSupply_,
        uint32 minterRate_
    ) public pure returns (uint32) {
        // solhint-disable max-line-length
        // When `totalActiveOwedM_ >= totalEarningSupply_`, it is possible for the earner rate to be higher than the
        // minter rate and still ensure cashflow safety over some period of time (`RATE_CONFIDENCE_INTERVAL`). To ensure
        // cashflow safety, we start with `cashFlowOfActiveOwedM >= cashFlowOfEarningSupply` over some time `dt`.
        // Effectively: p1 * (exp(rate1 * dt) - 1) >= p2 * (exp(rate2 * dt) - 1)
        //          So: rate2 <= ln(1 + (p1 * (exp(rate1 * dt) - 1)) / p2) / dt
        // 1. totalActive * (delta_minterIndex - 1) >= totalEarning * (delta_earnerIndex - 1)
        // 2. totalActive * (delta_minterIndex - 1) / totalEarning >= delta_earnerIndex - 1
        // 3. 1 + (totalActive * (delta_minterIndex - 1) / totalEarning) >= delta_earnerIndex
        // Substitute `delta_earnerIndex` with `exponent((earnerRate * dt) / SECONDS_PER_YEAR)`:
        // 4. 1 + (totalActive * (delta_minterIndex - 1) / totalEarning) >= exponent((earnerRate * dt) / SECONDS_PER_YEAR)
        // 5. ln(1 + (totalActive * (delta_minterIndex - 1) / totalEarning)) >= (earnerRate * dt) / SECONDS_PER_YEAR
        // 6. ln(1 + (totalActive * (delta_minterIndex - 1) / totalEarning)) * SECONDS_PER_YEAR / dt >= earnerRate

        // When `totalActiveOwedM_ < totalEarningSupply_`, the instantaneous earner cash flow must be less than the
        // instantaneous minter cash flow. To ensure instantaneous cashflow safety, we we use the derivatives of the
        // previous starting inequality, and substitute `dt = 0`.
        // Effectively: p1 * rate1 >= p2 * rate2
        //          So: rate2 <= p1 * rate1 / p2
        // 1. totalActive * minterRate >= totalEarning * earnerRate
        // 2. totalActive * minterRate / totalEarning >= earnerRate
        // solhint-enable max-line-length

        if (totalActiveOwedM_ == 0) return 0;

        if (totalEarningSupply_ == 0) return type(uint32).max;

        if (totalActiveOwedM_ <= totalEarningSupply_) {//@audit-info rate is slashed
            // NOTE: `totalActiveOwedM_ * minterRate_` can revert due to overflow, so in some distant future, a new
            //       rate model contract may be needed that handles this differently.
            return uint32((uint256(totalActiveOwedM_) * minterRate_) / totalEarningSupply_);
        }

        uint48 deltaMinterIndex_ = ContinuousIndexingMath.getContinuousIndex(
            ContinuousIndexingMath.convertFromBasisPoints(minterRate_),
            RATE_CONFIDENCE_INTERVAL
        );//@audit-info deltaMinterIndex for 30 days

        // NOTE: `totalActiveOwedM_ * deltaMinterIndex_` can revert due to overflow, so in some distant future, a new
        //       rate model contract may be needed that handles this differently.
        int256 lnArg_ = int256(
            _EXP_SCALED_ONE +
                ((uint256(totalActiveOwedM_) * (deltaMinterIndex_ - _EXP_SCALED_ONE)) / totalEarningSupply_)
        );

        int256 lnResult_ = wadLn(lnArg_ * _WAD_TO_EXP_SCALER) / _WAD_TO_EXP_SCALER;

        uint256 expRate_ = (uint256(lnResult_) * ContinuousIndexingMath.SECONDS_PER_YEAR) / RATE_CONFIDENCE_INTERVAL;

        if (expRate_ > type(uint64).max) return type(uint32).max;

        // NOTE: Do not need to do `UIntMath.safe256` because it is known that `lnResult_` will not be negative.
        uint40 safeRate_ = ContinuousIndexingMath.convertToBasisPoints(uint64(expRate_));

        return (safeRate_ > type(uint32).max) ? type(uint32).max : uint32(safeRate_);
    }
```
As we can see, the rate may vary due to the changes in `MToken#totalEarningSupply()`, therefore the earning of fixed principal amount could be decreased if `totalEarningSupply()` increases.  In some other cases the total earning rewards increases significantly if `totalEarningSupply()` increases, resulting in less `excessOwedM` sending to `ttgVault` when [`MinterGateway#updateIndex()`](https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L432-L449) is called.

Copy below codes to [Integration.t.sol](https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/test/integration/Integration.t.sol) and run `forge test --match-test test_aliceStillEarnAfterDisapproved`
```solidity
    function test_AliceStillEarnAfterDisapproved() external {

        _registrar.updateConfig(MAX_EARNER_RATE, 40000);
        _minterGateway.activateMinter(_minters[0]);

        uint256 collateral = 1_000_000e6;
        _updateCollateral(_minters[0], collateral);

        _mintM(_minters[0], 400e6, _bob);
        _mintM(_minters[0], 400e6, _alice);
        uint aliceInitialBalance = _mToken.balanceOf(_alice);
        uint bobInitialBalance = _mToken.balanceOf(_bob);
        //@audit-info alice and bob had the same M balance
        assertEq(aliceInitialBalance, bobInitialBalance);
        //@audit-info alice and bob started earning
        vm.prank(_alice);
        _mToken.startEarning();
        vm.prank(_bob);
        _mToken.startEarning();

        vm.warp(block.timestamp + 1 days);
        uint aliceEarningDay1 = _mToken.balanceOf(_alice) - aliceInitialBalance;
        uint bobEarningDay1 = _mToken.balanceOf(_bob) - bobInitialBalance;
        //@audit-info Alice and Bob have earned the same M in day 1
        assertNotEq(aliceEarningDay1, 0);
        assertEq(aliceEarningDay1, bobEarningDay1);
        //@audit-info Alice was removed from earner list
        _registrar.removeFromList(TTGRegistrarReader.EARNERS_LIST, _alice);
        vm.warp(block.timestamp + 1 days);
        uint aliceEarningDay2 = _mToken.balanceOf(_alice) - aliceInitialBalance - aliceEarningDay1;
        uint bobEarningDay2 = _mToken.balanceOf(_bob) - bobInitialBalance - bobEarningDay1;
        //@audit-info Alice still earned M in day 2 even she was removed from earner list, the amount of which is same as Bob's earning
        assertNotEq(aliceEarningDay2, 0);
        assertEq(aliceEarningDay2, bobEarningDay2);

        uint earnerRateBefore = _mToken.earnerRate();
        //@audit-info Only Alice can stop herself from earning
        vm.prank(_alice);
        _mToken.stopEarning();
        uint earnerRateAfter = _mToken.earnerRate();
        //@audit-info The earning rate was almost doubled after Alice called `stopEarning`
        assertApproxEqRel(earnerRateBefore*2, earnerRateAfter, 0.01e18);
        vm.warp(block.timestamp + 1 days);
        uint aliceEarningDay3 = _mToken.balanceOf(_alice) - aliceInitialBalance - aliceEarningDay1 - aliceEarningDay2;
        uint bobEarningDay3 = _mToken.balanceOf(_bob) - bobInitialBalance - bobEarningDay1 - bobEarningDay2;
        //@audit-info Alice earned nothing 
        assertEq(aliceEarningDay3, 0);
        //@audit-info Bob's earnings on day 3 were almost twice as much as what he earned on day 2.
        assertApproxEqRel(bobEarningDay2*2, bobEarningDay3, 0.01e18);
    }
```
## Impact
- The earnings of eligible users could potentially be diluted.
- The `excessOwedM` to ZERO vault holders could be diluted
## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MToken.sol#L106-L108

## Tool used

Manual Review

## Recommendation
Introduce a method that allows anyone to stop the disapproved earner from earning:
```solidity
    function stopEarning(address account_) external {
        if (_isApprovedEarner(account_)) revert IsApprovedEarner();
        _stopEarning(account_);
    }
```
