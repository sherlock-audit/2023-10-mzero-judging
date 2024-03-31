Curly Shamrock Cow

medium

# The `MinterGateway.sol#_verifyvalidatorSignatures` function calculates `minTimestamp` incorrectly.

## Summary
The` MinterGateway.sol#_verifyvalidatorSignatures` function returns `minTimestamp_` as the minimum value of the first `updateCollateralValidatorThreshold` valid `timestamps` in the `signature_` array, regardless of the total number of signatures entered.
This can be detrimental to `minters` when they use more signatures than the `updateCollateralValidatorThreshold` number and the `minter`'s `updateTimestamp` is pulled.
## Vulnerability Detail
The ` MinterGateway.sol#_verifyvalidatorSignatures` function is as follows.

```solidity
    function _verifyValidatorSignatures(
        address minter_,
        uint256 collateral_,
        uint256[] calldata retrievalIds_,
        bytes32 metadataHash_,
        address[] calldata validators_,
        uint256[] calldata timestamps_,
        bytes[] calldata signatures_
    ) internal view returns (uint40 minTimestamp_) {
        uint256 threshold_ = updateCollateralValidatorThreshold();

        minTimestamp_ = uint40(block.timestamp);

        // Stop processing if there are no more signatures or `threshold_` is reached.
        for (uint256 index_; index_ < signatures_.length && threshold_ > 0; ++index_) {
            unchecked {
                // Check that validator address is unique and not accounted for
                // NOTE: We revert here because this failure is entirely within the minter's control.
                if (index_ > 0 && validators_[index_] <= validators_[index_ - 1]) revert InvalidSignatureOrder();
            }

            // Check that validator is approved by TTG.
            if (!isValidatorApprovedByTTG(validators_[index_])) continue;

            // Check that the timestamp is not 0.
            if (timestamps_[index_] == 0) revert ZeroTimestamp();

            // Check that the timestamp is not in the future.
            if (timestamps_[index_] > uint40(block.timestamp)) revert FutureTimestamp();

            // NOTE: Need to store the variable here to avoid a stack too deep error.
            bytes32 digest_ = _getUpdateCollateralDigest(
                minter_,
                collateral_,
                retrievalIds_,
                metadataHash_,
                timestamps_[index_]
            );

            // Check that ECDSA or ERC1271 signatures for given digest are valid.
            if (!SignatureChecker.isValidSignature(validators_[index_], digest_, signatures_[index_])) continue;

            // Find minimum between all valid timestamps for valid signatures.
            minTimestamp_ = UIntMath.min40(minTimestamp_, uint40(timestamps_[index_]));

            unchecked {
                --threshold_;
            }
        }

        // NOTE: Due to STACK_TOO_DEEP issues, we need to refetch `requiredThreshold_` and compute the number of valid
        //       signatures here, in order to emit the correct error message. However, the code will only reach this
        //       point to inevitably revert, so the gas cost is not much of a concern.
        if (threshold_ > 0) {
            uint256 requiredThreshold_ = updateCollateralValidatorThreshold();

            unchecked {
                // NOTE: By this point, it is already established that `threshold_` is less than `requiredThreshold_`.
                revert NotEnoughValidSignatures(requiredThreshold_ - threshold_, requiredThreshold_);
            }
        }
    }
    }
```
This function computes `minTimestamp_` as the minimum of the first valid `updateCollateralValidatorThreshold` `timestamp`s in the `signatures_` array. In fact, this is disadvantageous to `minter`s. 
Let's look at an example when `updateCollateralValidatorThreshold() = 2`.
If the four timestamps are 5,6,2,1 and all four are valid, then `min(5,6) = 5`, so `minTimestamp_ = 5`. In fact, in this case, it is more accurate to set `minTimestamp_ = 1` rather than `minTimestamp_ = 5`. 
`minter`s do not change the order of the four timestamps from 5,6,2,1 to 1,2,5,6. 
Because validators_must be arranged in order, the order cannot be changed. Therefore, the current `MinterGateway.sol#_verifyValidatorSignatures` function is unfavorable to `Minter` and causes `minter`s anxiety.
## Impact
Users feel resistance to this protocol because they incur unexpected losses.
## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L1045-L1107
## Tool used

Manual Review

## Recommendation
Modify the ` MinterGateway.sol#_verifyvalidatorSignatures` function as follows.

```solidity
    function _verifyValidatorSignatures(
        address minter_,
        uint256 collateral_,
        uint256[] calldata retrievalIds_,
        bytes32 metadataHash_,
        address[] calldata validators_,
        uint256[] calldata timestamps_,
        bytes[] calldata signatures_
    ) internal view returns (uint40 minTimestamp_) {
        uint256 threshold_ = updateCollateralValidatorThreshold();

        minTimestamp_ = uint40(block.timestamp);

        // Stop processing if there are no more signatures or `threshold_` is reached.
-       for (uint256 index_; index_ < signatures_.length && threshold_ > 0; ++index_) {
+       for (uint256 index_; index_ < signatures_.length; ++index_) {
            unchecked {
                // Check that validator address is unique and not accounted for
                // NOTE: We revert here because this failure is entirely within the minter's control.
                if (index_ > 0 && validators_[index_] <= validators_[index_ - 1]) revert InvalidSignatureOrder();
            }

            // Check that validator is approved by TTG.
            if (!isValidatorApprovedByTTG(validators_[index_])) continue;

            // Check that the timestamp is not 0.
            if (timestamps_[index_] == 0) revert ZeroTimestamp();

            // Check that the timestamp is not in the future.
            if (timestamps_[index_] > uint40(block.timestamp)) revert FutureTimestamp();

            // NOTE: Need to store the variable here to avoid a stack too deep error.
            bytes32 digest_ = _getUpdateCollateralDigest(
                minter_,
                collateral_,
                retrievalIds_,
                metadataHash_,
                timestamps_[index_]
            );

            // Check that ECDSA or ERC1271 signatures for given digest are valid.
            if (!SignatureChecker.isValidSignature(validators_[index_], digest_, signatures_[index_])) continue;

            // Find minimum between all valid timestamps for valid signatures.
            minTimestamp_ = UIntMath.min40(minTimestamp_, uint40(timestamps_[index_]));

+           unchecked {
+               --threshold_;
+           }
        }

        // NOTE: Due to STACK_TOO_DEEP issues, we need to refetch `requiredThreshold_` and compute the number of valid
        //       signatures here, in order to emit the correct error message. However, the code will only reach this
        //       point to inevitably revert, so the gas cost is not much of a concern.
-       if (threshold_ > 0) {
+       if (threshold_ - signatures_.length > 0) {
            uint256 requiredThreshold_ = updateCollateralValidatorThreshold();

            unchecked {
                // NOTE: By this point, it is already established that `threshold_` is less than `requiredThreshold_`.
                revert NotEnoughValidSignatures(requiredThreshold_ - threshold_, requiredThreshold_);
            }
        }
    }
```