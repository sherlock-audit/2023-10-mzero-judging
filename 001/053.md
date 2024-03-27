Young Umber Okapi

high

# Users unable to receive back the fee for their successful proposal

## Summary

Users are unable to receive back the fee for their successful proposal if a reset occurs.

## Vulnerability Detail

Per the M0 whitepaper (https://docs.m0.org/m-0-documentation-portal/overview/whitepaper/iii.-governance/iii.ii-operation/iii.ii.ii-proposals/iii.ii.ii.i-standard-proposals), the proposal fee will be refunded to the proposers if the proposal is successful.

> A successful Standard Proposal will have its Proposal Fee available to be returned to the proposer.

Assume that the address of the current Standard Governor contract is `0xACE`.

Assume that Alice created a proposal and paid 1 WETH proposal fee in Epoch 98 (Transfer Phase). At the end of Epoch 99 (Voting Phase), Alice's proposal was successful, receiving 51 Yes and 50 No votes. Before Alice's proposal can be executed, the zero holders trigger a reset shortly after the end of Epoch 99, as shown in the timeline below.

![image-20240319173539336](https://github.com/sherlock-audit/2023-10-mzero-xiaoming9090/assets/102820284/96cc31dd-3a39-4255-9675-bedf82fb15a6)

As a result of the reset, a new `StandardGovernor` contract is deployed, and the `StandardGovernorDeployer.lastDeploy` state variable is updated to the new `StandardGovernor` contract with an address of `0xDAD`.

When Alice tries to execute her successful proposal, it will execute the Registrar functions such as `addToList` or `removeFromList`. However, the transaction will revert since these functions are guarded by the `onlyStandardOrEmergencyGovernor` modifier, which ensures that only the current standard governance can call the Registrar functions. Since Alice's proposal and deposited proposal fee are stored on the previous Standard Governor (`0xACE`), the caller to the Registrar contract will be `0xACE` (old governor) instead of `0xDAD` (new governor) and cause a revert.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/Registrar.sol#L54

```solidity
File: Registrar.sol
53:     /// @dev Revert if the caller is not the Standard Governor nor the Emergency Governor.
54:     modifier onlyStandardOrEmergencyGovernor() {
55:         _revertIfNotStandardOrEmergencyGovernor();
56:         _;
57:     }
..SNIP..
89:     function addToList(bytes32 list_, address account_) external onlyStandardOrEmergencyGovernor {
90:         _valueAt[_getIsInListKey(list_, account_)] = bytes32(uint256(1));
91: 
92:         emit AddressAddedToList(list_, account_);
93:     }
```

Since Alice's successful proposal cannot be executed without a revert, the code that is responsible for the refund at Line 139 below will never be reached. Thus, Alice will not be able to receive back her proposal fee even though the proposal is successful and she attempted to execute it before it expired (Before Epoch 101).

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/StandardGovernor.sol#L122

```solidity
File: StandardGovernor.sol
122:     function execute(
123:         address[] memory,
124:         uint256[] memory,
125:         bytes[] memory callDatas_,
126:         bytes32
127:     ) external payable returns (uint256 proposalId_) {
128:         // Proposals have voteStart=N and voteEnd=N, and can be executed only during epoch N+1.
129:         uint16 latestPossibleVoteStart_ = _clock() - 1;
130: 
131:         proposalId_ = _tryExecute(callDatas_[0], latestPossibleVoteStart_, latestPossibleVoteStart_);
132: 
133:         ProposalFeeInfo storage proposalFeeInfo_ = _proposalFees[proposalId_];
134:         uint256 proposalFee_ = proposalFeeInfo_.fee;
135:         address cashToken_ = proposalFeeInfo_.cashToken;
136: 
137:         if (proposalFee_ > 0) {
138:             delete _proposalFees[proposalId_];
139:             _transfer(cashToken_, _proposals[proposalId_].proposer, proposalFee_);
140:         }
141:     }
```

After Epoch 101, Alice's successful proposal will expire, and anyone can call the [`sendProposalFeeToVault`](https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/StandardGovernor.sol#L180) function to transfer Alice's deposited proposal fee to the vault and distribute it to the zero token holders.  

There are several issues here:

1. It is unfair for Alice, as her proposal indeed gained the support of the majority during the voting phase, and she should be entitled to receive her proposal fee back.
2. The zero holders should not be entitled to and receive Alice's proposal fee simply because a reset is triggered. In fact, the fee of all proposals impacted by the reset should be refunded to the proposer, not to the wallet of zero holders.

## Impact

Loss of proposal fee for the affected users.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/Registrar.sol#L54

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/StandardGovernor.sol#L122

## Tool used

Manual Review

## Recommendation

Update the logic to allow the users to claim their proposal fee if the proposal is successful (Yes > No after the voting phase). Alternatively, update the logic to automatically refund the fee of all proposals impacted by the reset.