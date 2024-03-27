Young Umber Okapi

high

# Using minimum timestamp across all signatures could drain the gateway

## Summary

Using the minimum timestamp across all signatures could drain the gateway.

## Vulnerability Detail

Assume the validator threshold is three (3). The ecosystem has four validators, $V_1, V_2, V_3, V_4$.

Assume that Bob obtains the following three signatures, which all attest that Bob's collateral value is 100, and submit them to the `updateCollateral` function. ($T6$ and $T7$ means that the timestamp is 1 second apart)

1. $Sig_1$ signed by $V_1$ with timestamp ($T1$)
2. $Sig_2$ signed by $V_2$ with timestamp ($T6$)
3. $Sig_3$ signed by $V_3$ with timestamp ($T7$)

After the collateral is updated, the `_updateCollateral` function will set the `_minterStates[Bob].collateral` to 100, and the `_minterStates[Bob].updateTimestamp` to the minimum timestamp across the three provided signatures, which will be $T1$​.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L168

```solidity
File: MinterGateway.sol
168:     function updateCollateral(
..SNIP..
180:         // Verify that enough valid signatures are provided, and get the minimum timestamp across all valid signatures.
181:         minTimestamp_ = _verifyValidatorSignatures(
182:             msg.sender,
183:             collateral_,
184:             retrievalIds_,
185:             metadataHash_,
186:             validators_,
187:             timestamps_,
188:             signatures_
189:         );
..SNIP..
204:         _updateCollateral(msg.sender, safeCollateral_, minTimestamp_);
..SNIP..
210:         updateIndex();
211:     }
..SNIP..
868:     function _updateCollateral(address minter_, uint240 amount_, uint40 newTimestamp_) internal {
869:         uint40 lastUpdateTimestamp_ = _minterStates[minter_].updateTimestamp;
870: 
871:         // MinterGateway already has more recent collateral update
872:         if (newTimestamp_ <= lastUpdateTimestamp_) revert StaleCollateralUpdate(newTimestamp_, lastUpdateTimestamp_);
873: 
874:         _minterStates[minter_].collateral = amount_;
875:         _minterStates[minter_].updateTimestamp = newTimestamp_;
876:     }
```

Bob withdraws all 100 collateral assets from the protocol, and Bob's on-chain collateral value becomes zero.

Bob obtains a signature from a malicious validator ($V_4$) that states he has collateral assets of 100 and a timestamp of $T2$.

Bob has the following signatures with him, which all attest that Bob's collateral value is 100. He then submits them to the `updateCollateral` function.

1. $Sig_2$ signed by $V_2$ with timestamp ($T6$) - Old signature
2. $Sig_3$ signed by $V_3$ with timestamp ($T7$) - Old signature
3. $Sig_4$ signed by $V_4$ with timestamp ($T2$) - New signature

Since the timestamp of the above three signatures is larger than $𝑇1$, they will be accepted. Bob's on-chain collateral will be updated to 100 again, and the minimum timestamp is set to $T2$. Bob could proceed to withdraw 100 collateral assets from the protocol again.

Bob and malicious $V_4$ could repeat the same attack multiple times by generating a new signature with a timestamp incremented by one second (e.g., $T3, T4...$) until the gateway is drained.

Assume the validator threshold is 5. The gateway's security is provided by requiring at least five (5) validators to authorize an update collateral transaction. If one validator is compromised, the assets with the gateway remain secure, as attackers would need to compromise the other four validators to steal the assets, which is difficult to achieve as they might be owned by different entities. This is the main purpose for adopting a multi-sig design for all kinds of systems from a security perspective.

However, as shown in the above scenario, if one validator is compromised, the gateway's security will be defeated, and multi-sig becomes single-sig. The attacker simply needs one signature from the compromised validator and reuses the old signatures from other validators to steal the gateway's assets. In a sense, the multi-sig design within the gateway does not provide additional security compared to a single-sig design.

The following diagram visualizes the subset of signatures to be used for the first and second update collateral operations.

![image-20240324111949140](https://github.com/sherlock-audit/2023-10-mzero-xiaoming9090/assets/102820284/028cbba1-eed5-49be-a5e6-df30d073e8c3)

## Impact

Loss of assets.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L168

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L868

## Tool used

Manual Review

## Recommendation

Consider setting the `_minterStates[Bob].updateTimestamp` to the maximum timestamp across all the provided signatures. If the update timestamp is set to $T6$, all subsequent signatures provided must be larger than $T6$. Any signature that does not meet this requirement must be rejected.

Guarding against potential replay attacks under all circumstances is the top priority of any protocol/system and is non-negotiable. Its importance significantly outweighs other requirements of the protocol/system, such as gas efficiency or data freshness. 

Avoid relying solely on off-chain agreements for security and assume that all validators always act in good faith. Even if the validator's owner or operator has no malicious intention, their validators might have already been compromised and controlled by hackers, and they are not aware of it. Off-chain validators can be compromised by hackers in many ways (e.g., network attacks and private key/password leakage). Many of the largest Defi hacks are still caused by traditional web2/off-chain attacks. (Refer to https://rekt.news/leaderboard and https://rekt.news/ronin-rekt/).

To protect the protocol's users and assets, defense-in-depth measures should be adopted in the event an off-chain agreement/security is broken.
