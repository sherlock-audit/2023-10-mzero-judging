Young Umber Okapi

high

# Proposal fee of new cash token takes effect in the wrong epoch

## Summary

The proposal fee of the new cash tokens takes effect in the wrong epoch, leading to a loss of assets for the zero token holders and/or users.

## Vulnerability Detail

Per the remark in the fix review PDF shared during the contest, it was understood that the system/protocol is designed in a manner where the switch of cash token is coupled with the proposal fee to avoid unfair griefing issues.

![image-20240318145127303](https://github.com/sherlock-audit/2023-10-mzero-xiaoming9090/assets/102820284/5ce966c0-68e1-44a2-8f4e-6e81114ea889)

During the review, it was observed that the switch to the new cash token will only take effect in the next epoch, not in the current epoch. However, the proposal fee is changed in the current epoch, which is incorrect and not in sync with the switch of cash token.

In short, the proposal fee takes effect in the current epoch before the actual cash token switch occurs in the next epoch, which is not aligned with the goal of having the switch of cash token and proposal fee to be coupled together to prevent unfair griefing issues.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/StandardGovernor.sol#L171

```solidity
File: StandardGovernor.sol
171:     function setCashToken(address newCashToken_, uint256 newProposalFee_) external onlyZeroGovernor {
172:         _setCashToken(newCashToken_);
173: 
174:         IPowerToken(voteToken).setNextCashToken(newCashToken_);
175: 
176:         _setProposalFee(newProposalFee_);
177:     }
```

This issue only applied to Standard Governance, as they depend on the cash token address to carry out various operations, such as collecting proposal fees and auctioning. Only Zero Governance can change the cash token, and they can change it in any epoch (both Transfer and Voting Epoch).

The set proposal fee takes effect immediately, while the cash token change will only take effect from the next epoch onwards. Line 174 below executes the [`setNextCashToken`](https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L179) function, which will schedule the cash token change to take effect in the next epoch, while the execution of `_setProposalFee` function at Line 176 below will update the proposal fee immediately.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/StandardGovernor.sol#L171

```solidity
File: StandardGovernor.sol
171:     function setCashToken(address newCashToken_, uint256 newProposalFee_) external onlyZeroGovernor {
172:         _setCashToken(newCashToken_);
173: 
174:         IPowerToken(voteToken).setNextCashToken(newCashToken_);
175: 
176:         _setProposalFee(newProposalFee_);
177:     }
```

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/StandardGovernor.sol#L423

```solidity
File: StandardGovernor.sol
419:     /**
420:      * @dev   Set proposal fee to `newProposalFee_`.
421:      * @param newProposalFee_ The new proposal fee.
422:      */
423:     function _setProposalFee(uint256 newProposalFee_) internal {
424:         emit ProposalFeeSet(proposalFee = newProposalFee_);
425:     }
```

The following shows the impact of this issue:

1. At T1 of Epoch 1 (Transfer Epoch), the cash token is WETH, and the proposal fee is 1000 USD. Assuming that 1 WETH = 1000 USD, the proposal fee is set to 1e18.

2. At T2 of Epoch 1 (Transfer Epoch), the zero holders intended to change the cash token to M while keeping the proposal fee the same at 1000 USD. Assume 1M = 1USD. M token is denominated with a 6-decimal precision. So, to get 1000 USD, the proposal fee has to be set to (1000 * 1e6). The threshold proposal by zero holders is immediately approved and executed, triggering the `setCashToken` function. The `setCashToken` function will trigger the `setNextCashToken` function to queue the M Token as the protocol's cash token that will take effect in the next epoch. Next, it will call the `_setProposalFee`. The issue here is that the proposal fee of (1000 * 1e6) will immediately take effect.

3. At T3 of Epoch 1 (Transfer Epoch), when someone creates a new proposal, the cashToken = WETH, and the proposal fee = (1000 * 1e6). Thus, the proposer only needs to pay (1000 * 1e6) WETH, which costs around 0.000001 USD.
4. This led to a loss for the zero holders, as they only received 0.000001 USD instead of 1000 USD worth of proposal fee.

On the other hand, consider a scenario similar to the one above. The cash token is M token in the first place and 1000 M tokens (1000 * 1e6) are collected in advance as the proposal fee from the proposal. When the cash token is switched to WETH, several issues could occur:

- If the proposal is successful, governance will attempt to refund 1000e6 WETH (worth only 0.000001 USD) to the users, which will result in a loss for them.
- If the proposal fails, the governance will attempt to transfer 1000e6 WETH (worth only 0.000001 USD) instead of 1000 M Tokens to the zero holders, leading to a loss for the zero holders.

## Impact

Loss of assets for zero token holders and/or users.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/StandardGovernor.sol#L171

## Tool used

Manual Review

## Recommendation

When the `setCashToken` is called by the zero holders, both the new cash token and the new proposal fee should only take effect in the next epoch.