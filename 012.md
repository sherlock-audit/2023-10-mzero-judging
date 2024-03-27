Mythical Rouge Mule

high

# Signature front-run

Hign
## Summary
If users call the function that relates to `_transferWithAuthorization()`, the hacker can get users' signature by memory pool and front run  `cancelAuthorization()` so that users' tx will fail.
## Vulnerability Detail

Anyone can call `cancelAuthorization()` since it doesn't check the `msg.sender`. 
```solidity
    /// @inheritdoc IERC3009
    function cancelAuthorization(address authorizer_, bytes32 nonce_, bytes memory signature_) external {
        _revertIfInvalidSignature(authorizer_, _getCancelAuthorizationDigest(authorizer_, nonce_), signature_);
        _cancelAuthorization(authorizer_, nonce_);
    }

    /// @inheritdoc IERC3009
    function cancelAuthorization(address authorizer_, bytes32 nonce_, bytes32 r_, bytes32 vs_) external {
        _revertIfInvalidSignature(authorizer_, _getCancelAuthorizationDigest(authorizer_, nonce_), r_, vs_);
        _cancelAuthorization(authorizer_, nonce_);
    }

    /// @inheritdoc IERC3009
    function cancelAuthorization(address authorizer_, bytes32 nonce_, uint8 v_, bytes32 r_, bytes32 s_) external {
        _revertIfInvalidSignature(authorizer_, _getCancelAuthorizationDigest(authorizer_, nonce_), v_, r_, s_);
        _cancelAuthorization(authorizer_, nonce_);
    }

    function _cancelAuthorization(address authorizer_, bytes32 nonce_) internal {
        _revertIfAuthorizationAlreadyUsed(authorizer_, nonce_);

        authorizationState[authorizer_][nonce_] = true;

        emit AuthorizationCanceled(authorizer_, nonce_);
    }
```
As long as hackers monitor MemPool, they can obtain information such as signature, nonce, and authorizer. Then front run the function `_transferWithAuthorization()` (no matter `transferWithAuthorization()` or `receiveWithAuthorization()`) by `_cancelAuthorization()`, making it impossible for users to use transfer functions. The `authorizationState [authorizer_] [nonce_]` is set to true, signatures cannot be used again to make changes because of unidirectionality.

## Impact
All transfers related to signatures will fail

## Code Snippet
- https://github.com/sherlock-audit/2023-10-mzero/blob/main/common/src/ERC3009.sol#L175-L191
- https://github.com/sherlock-audit/2023-10-mzero/blob/main/common/src/ERC3009.sol#L251-L257

## Tool used

Manual Review

## Recommendation
When calling `_cancelAuthorization()`, check whether `msg. sender` is the authorizer or not.
```solidity
    function _cancelAuthorization(address authorizer_, bytes32 nonce_) internal {
    	// add logic: if msg.sender is not authorizer_, tx revert.
    
        _revertIfAuthorizationAlreadyUsed(authorizer_, nonce_);

        authorizationState[authorizer_][nonce_] = true;

        emit AuthorizationCanceled(authorizer_, nonce_);
    }
```
