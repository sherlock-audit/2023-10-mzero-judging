Rough Violet Copperhead

medium

# `MinterGateway.updateCollateral()` signatures may be replayed to mint uncollateralized MToken in case of two collateral updates around the same time

## Summary
It is possible for a minter who obtains two sets of signatures to do two collateral updates around the same time to replay the first set of signatures to set the collateral value higher than the actual offchain holdings and mint uncollateralized MToken for profit.
## Vulnerability Detail
There are a few details that make this attack possible. The first is that validator timestamps can be different for the same call to `updateCollateral()`.
```solidity
    function updateCollateral(
        // Verify that enough valid signatures are provided, and get the minimum timestamp across all valid signatures.
        minTimestamp_ = _verifyValidatorSignatures(
```

The second is that there is no nonce used in validator signatures for `updateCollateral()`.
```solidity
    function _getUpdateCollateralDigest(
        address minter_,
        uint256 collateral_,
        uint256[] calldata retrievalIds_,
        bytes32 metadataHash_,
        uint256 timestamp_
    ) internal view returns (bytes32) {
        return
            _getDigest(
                keccak256(
                    abi.encode(
                        UPDATE_COLLATERAL_TYPEHASH,
                        minter_,
                        collateral_,
                        keccak256(abi.encodePacked(retrievalIds_)),
                        metadataHash_,
                        timestamp_
                    )
                )
            );
    }
```
These combine with the third detail to enable the attack. 

When pending retrievals are executed in `updateCollateral()`, any retrieval that has already been executed will be ignored instead of reverting the transaction:
```solidity
    function _resolvePendingRetrievals(
        address minter_,
        uint256[] calldata retrievalIds_
    ) internal returns (uint240 totalResolvedCollateralRetrieval_) {
        for (uint256 index_; index_ < retrievalIds_.length; ++index_) {
            uint48 retrievalId_ = UIntMath.safe48(retrievalIds_[index_]);
            uint240 pendingCollateralRetrieval_ = _pendingCollateralRetrievals[minter_][retrievalId_];

            if (pendingCollateralRetrieval_ == 0) continue;

            unchecked {
                // NOTE: The `proposeRetrieval` function already ensures that the sum of all
                // `_pendingCollateralRetrievals` is not larger than `type(uint240).max`.
                totalResolvedCollateralRetrieval_ += pendingCollateralRetrieval_;
            }

            delete _pendingCollateralRetrievals[minter_][retrievalId_];

            emit RetrievalResolved(retrievalId_, minter_);
        }
```

Normally, replaying a set of signatures for `updateCollateral()` won't do anything since the timestamp check in `_updateCollateral()` will typically prevent 'rollbacks'. However, if the minter updates the collateral twice around the same time there is an edge case where replaying the first update can improperly increase the collateral.

For this edge case to occur, a minter needs to submit two withdrawal requests and then obtain two sets of validator signatures to approve these requests. The first withdrawal request needs to have N+1 validator signatures, where N is the threshold. Also, there must be a difference of at least two seconds between the two smallest timestamps of the first set of signatures, and the smallest timestamp of the second set of signatures must be between the two smallest timestamps of the first set of signatures.

Example scenario:
1. Minter submits one withdrawal request to withdraw some small amount of collateral, and obtains the first set of validator signatures. The minter submits another withdrawal request right after to withdraw some large amount of the collateral, and obtains the second set of validator signatures.
2. The minter calls `updateCollateral()` with the first set of signatures, omitting the signature with the second smallest timestamp. The minter's collateral is decreased by a small amount.
3. The minter calls `updateCollateral()` with the second set of signatures, decreasing the collateral by a large amount (ideally the minter wants to withdraw the entire collateral).
4. The minter retrieves the offchain collateral from the protocol.
5. The minter calls `updateCollateral()` with the first set of signatures again, this time omitting the signature with the smallest timestamp and including the signature with the second smallest timestamp. The result is that the collateral is set to the large amount of collateral that the minter had before the second withdrawal.
6. The minter mints uncollateralized MTokens and sells them for profit.

There are some other variations possible for this attack, but I suspect this one is the only likely one.
## Impact
Theft of funds, uncollateralized MToken minting.

## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L181
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L958-L978
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L834-L860

## Tool used

Manual Review

## Recommendation
The scenario outlined will not be possible if `_resolvePendingRetrievals()` reverts for already executed retrievals. Another fix is to use a nonce so that `updateCollateral()` signatures can't be replayed.