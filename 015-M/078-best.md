Curly Shamrock Cow

high

# The mint and burn function is vulnerable to Sandwitch Attack at `Minter Rate` update.

## Summary
If an attacker finds a `proposal` in the mempool that reduces the `Minter Rate`, he or she can retain `MToken` for free through a Sandwitch attack.
## Vulnerability Detail
The `MinterGateway.sol#_rate` function and `MinterRateModel.sol#rate` function are as follows.
```solidity
    function _rate() internal view override returns (uint32 rate_) {
        (bool success_, bytes memory returnData_) = rateModel().staticcall(
            abi.encodeWithSelector(IRateModel.rate.selector)
        );

        rate_ = (success_ && returnData_.length >= 32) ? UIntMath.bound32(abi.decode(returnData_, (uint256))) : 0;
    }
```

```solidity
    function rate() external view returns (uint256 rate_) {
        return uint256(ITTGRegistrar(ttgRegistrar).get(_BASE_MINTER_RATE));
    }
```
As shown on the right, `rate_` recalled in the `MinterGateway.sol#_rate` function is the `Minter Rate` value currently set in TTG.
Additionally, the `MinterGateway.sol#currentIndex` function is as follows.
```solidity
    function currentIndex() public view override(ContinuousIndexing, IContinuousIndexing) returns (uint128) {
        // NOTE: Safe to use unchecked here, since `block.timestamp` is always greater than `latestUpdateTimestamp`.
        unchecked {
            return
                // NOTE: Cap the index to `type(uint128).max` to prevent overflow in present value math.
                UIntMath.bound128(
                    ContinuousIndexingMath.multiplyIndicesUp(
                        latestIndex,
                        ContinuousIndexingMath.getContinuousIndex(
                            ContinuousIndexingMath.convertFromBasisPoints(_latestRate),
                            uint32(block.timestamp - latestUpdateTimestamp)
                        )
                    )
                );
        }
    }
```

As shown on the right, the `currentIndex` value changes exponentially in proportion to `_latestRate`.
The attacker uses the above facts to carry out the following attack when the `Minter Rate` becomes small.

1. Call `mintM()` when `Minter Rate` is high.
  For example, let's say `currentIndex=5`.
  At this time, if you mint 1000 `MToken`, `PrincipalAmount` increases by `1000/5 = 200`.
2. ‘Minter Rate’ has decreased.
3. The attacker calls `ContinuousIndexing.sol#updateIndex` to change the `currentIndex` value.
   For example, let's decrement and say `currentIndex = 4`.
4.At this time, the attacker calls `burnM()`. At this time, since the `Principal Amount` is 200, only `200 * 4 = 800` `MToken` are needed.
Therefore, the attacker benefited from 200 `MToken`.
## Impact
An attacker can receive `MToken` without any compensation.
## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L261
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L322
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L981-L984
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/rateModels/MinterRateModel.sol#L35-L37
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L683
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L981-L984
## Tool used

Manual Review

## Recommendation
Implement pause(), unpause() mechanism when the TTG updates `Minter Rate`.