Young Umber Okapi

high

# Multiple withdrawals

## Summary

A minter can withdraw multiple times, leading to a loss of assets for the gateway/protocol.

## Vulnerability Detail

Assume the validator threshold is two (2). The ecosystem has four validators, $V_1, V_2, V_3, V_4$.

Bob obtains the following four signatures, which all attest that Bob's collateral value is 100 from all four validators, even though only two signatures are needed to meet the threshold requirement. 

1. $Sig_1$ signed by $V_1$ with timestamp ($T1$)
2. $Sig_2$ signed by $V_2$ with timestamp ($T3$)
3. $Sig_3$ signed by $V_3$ with timestamp ($T5$​​)
4. $Sig_4$ signed by $V_4$ with timestamp ($T7$)

Note: It is normal for the timestamp to be slightly different within the obtained signatures because it is not possible for all validators to be in sync and generate a signature with the same timestamp.

After obtaining the four signatures, Bob performed the following actions:

1. Bob submits $Sig_1$ and $Sig_2$ to the `_updateCollateral` function. It will set the `_minterStates[Bob].collateral` to 100, and the `_minterStates[Bob].updateTimestamp` to the minimum timestamp across the two provided signatures, which will be $T1$.

2. Bob withdraws 100 collateral assets from the protocol
3. Bob submits $Sig_3$ and $Sig_4$ to the `_updateCollateral` function again. Since the timestamp of both signatures is larger than $T1$, they will be accepted. Bob's  `_minterStates[Bob].collateral` will be updated to 100 again.
4. Bob withdraws 100 collateral assets from the protocol

As shown above, Bob could withdraw from the multiple times, leading to a loss of assets for the gateway/protocol.

The following diagram visualizes the subset of signatures to be used for the first and second update collateral operations.

![image-20240324110036481](https://github.com/sherlock-audit/2023-10-mzero-xiaoming9090/assets/102820284/ccc81221-9060-43ef-ba60-1fb3aefda99f)

## Impact

Loss of assets.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L168

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L868

## Tool used

Manual Review

## Recommendation

This issue is caused by the fact that a minter can obtain multiple signatures from multiple validators before the update collateral is executed. The minter can choose to submit only a subset of the obtained signatures (e.g., $Sig_1, Sig_2$) to update the collateral value and then use another subset of the obtained signatures (e.g., $Sig_3, Sig_4$) to update the collateral value again later to reset the on-chain collateral value.

An important point to note is that this issue cannot be remediated by setting `_minterStates[Bob].updateTimestamp` to the maximum timestamp across the submitted signatures, as the above scenario described, will still succeed if the maximum timestamp is used. Thus, it is an entirely different issue and root cause from another replay attack bug I have reported. A more sophisticated solution will be needed to remediate this.

