Amusing Wooden Dragon

medium

# The proposer of the `setProposalFee()` has a chance to retrieve their proposal fee after the RESET event, while others do not have this opportunity.

## Summary
When RESET event happens, `PowerToken`, `StandardGovernor` and `EmergencyGovernor` will be redeployed. 
**Certora** identified one problem of `proposalFee`: 
> Proposal   fee is   not  returned   to   proposer  after RESET event

The sponsor stated in [Audit review](https://docs.google.com/document/d/1_q6pQSx9X3nXQJ66TreNJmFfj--c7IFuy9GQ-mU0eJo/edit#heading=h.w34wioy0ru1o) that it is by design.

`StandardGovernor` supports five types of proposals:
- addToList
- removeFromList
- removeFromAndAddToList
- setKey
- setProposalFee

However, the proposal `setProposalFee` might have chance to be executed successfully and return the proposal fee to the proposer while other proposals will revert.

## Vulnerability Detail
When a proposal in `StandardGovernor` succeeds, any one can call `StandardGovernor#execute()` to execute the succeeded proposal. 
Once executed, the proposal fee will be returned to the proposer.
Let's take a look how `addToList` proposal is executed:
First, `addToList()` can only be called by `StandardGovernor` itself:
```solidity
    function addToList(bytes32 list_, address account_) external onlySelf {
        _addToList(list_, account_);
    }
```
Then, `registrar#addToList()` will be called to finish the rest:
```solidity
    function _addToList(bytes32 list_, address account_) internal {
        IRegistrar(registrar).addToList(list_, account_);
    }
```
The function `registrar#addToList()` can only be accessed by `StandardGovernor` or `EmergencyGovernor`. Either `StandardGovernor` or `EmergencyGovernor` will be always the `lastDeploy()` of its deployer:
```solidity
    function addToList(bytes32 list_, address account_) external onlyStandardOrEmergencyGovernor {
        _valueAt[_getIsInListKey(list_, account_)] = bytes32(uint256(1));

        emit AddressAddedToList(list_, account_);
    }
    modifier onlyStandardOrEmergencyGovernor() {
        _revertIfNotStandardOrEmergencyGovernor();
        _;
    }
    function _revertIfNotStandardOrEmergencyGovernor() internal view {
        if (msg.sender != standardGovernor() && msg.sender != emergencyGovernor()) {
            revert NotStandardOrEmergencyGovernor();
        }
    }
    function emergencyGovernor() public view returns (address) {
        return IEmergencyGovernorDeployer(emergencyGovernorDeployer).lastDeploy();
    }

    /// @inheritdoc IRegistrar
    function standardGovernor() public view returns (address) {
        return IStandardGovernorDeployer(standardGovernorDeployer).lastDeploy();
    }
```
In case of RESET event, the proposals in old `StandardGovernor` should fail to execute due to access control:
- The caller is stale and `standardGovernorDeployer.lastDeploy()` has been updated.

However, `setProposalFee()` doesn't have to access any call in `registrar`. It can be executed without any restrictions:
```solidity
    function setProposalFee(uint256 newProposalFee_) external onlySelfOrEmergencyGovernor {
        _setProposalFee(newProposalFee_);
    }
    function _setProposalFee(uint256 newProposalFee_) internal {
        emit ProposalFeeSet(proposalFee = newProposalFee_);
    }
```
Copy below codes to [resetToPowerHolders.t.soll](https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/test/integration/zero-governor/reset/reset-to-power-holders/resetToPowerHolders.t.sol) and run `forge test --match-test test_ExecuteOldProposalAfterResetToPowerHolders`
```solidity
    function test_ExecuteOldProposalAfterResetToPowerHolders() external {
        //@audit-info create setProposalFee proposal in StandardGovernor
        address[] memory targets_ = new address[](1);
        targets_[0] = address(_standardGovernor);

        uint256[] memory values_ = new uint256[](1);

        bytes[] memory callDatas_ = new bytes[](1);
        callDatas_[0] = abi.encodeWithSelector(_standardGovernor.setProposalFee.selector, 1);
        _warpToNextTransferEpoch();
        vm.prank(_dave);
        uint standProposalId = _standardGovernor.propose(targets_, values_, callDatas_, "");
        _warpToNextVoteEpoch();
        vm.prank(_alice);
        _standardGovernor.castVote(standProposalId, 1);

        //@audit-info create resetToPowerHolders proposal in ZeroGovernor
        address[] memory targetsZeroGovernor_ = new address[](1);
        targetsZeroGovernor_[0] = address(_zeroGovernor);

        bytes[] memory callDatasZeroGovernor_ = new bytes[](1);
        callDatasZeroGovernor_[0] = abi.encodeWithSelector(_zeroGovernor.resetToPowerHolders.selector);

        string memory description_ = "Reset to Power holders";

        vm.prank(_dave);
        uint256 proposalId_ = _zeroGovernor.propose(targetsZeroGovernor_, values_, callDatasZeroGovernor_, description_);

        vm.prank(_dave);
        _zeroGovernor.castVote(proposalId_, 1);

        address nextStandardGovernor_ = IStandardGovernorDeployer(_registrar.standardGovernorDeployer()).nextDeploy();

        _zeroGovernor.execute(targetsZeroGovernor_, values_, callDatasZeroGovernor_, keccak256(bytes(description_)));
        //@audit-info RESET succeeded and new standardGovernor is deployed.
        assertEq(_registrar.standardGovernor(), nextStandardGovernor_);
        //@audit-info setProposalFee proposal in old StandardGovernor can be executed successfully and the proposal fee is return to the proposer dave
        _warpToNextEpoch();
        vm.expectEmit();
        emit IGovernor.ProposalExecuted(standProposalId);
        vm.expectEmit();
        emit IStandardGovernor.ProposalFeeSet(1);
        vm.expectEmit();
        emit IERC20.Transfer(address(_standardGovernor), _dave, _standardProposalFee);
        _standardGovernor.execute(targets_, values_, callDatas_, keccak256(bytes("")));   
    }
```
## Impact
When a RESET event happens, the response mechanism for unexpired proposals in old `StandardGovernor` lacks consistency, resulting in the design being broken.
## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/StandardGovernor.sol#L225-L227
## Tool used

Manual Review

## Recommendation
The design should remain consistent. Either all proposal fees should be sent to the vault, or they should be returned to the proposers.