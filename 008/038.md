Amusing Wooden Dragon

medium

# The sum of all voting power might be different from the sum of all power token balance due to the flaw of inflation mechanism

## Summary
The sum of all voting power might be different from the sum of all power token balance due to the flaw of inflation mechanism
## Vulnerability Detail
The total supply of Power Token will be inflated if there is at least one active standard proposal in the voting epoch. A voter can inflate their own `Power` balance as well as the `Power` balances of all delegators by voting on all active proposals. 
The sponsor clearly declared  the `Main Invariants` in [their TTG spec](https://github.com/MZero-Labs/sherlock-contest-docs/blob/main/eng-specs/M%5E0%20TTG%20Engineering%20Spec_v1.2.pdf):
> 1. 𝑃𝑂𝑊𝐸𝑅 𝑡𝑜𝑡𝑎𝑙𝑉𝑜𝑡𝑖𝑛𝑔𝑃𝑜𝑤𝑒𝑟<sub>𝑑𝑒𝑙𝑒𝑔𝑎𝑡𝑒𝑠</sub> == 𝑃𝑂𝑊𝐸𝑅 𝑡𝑜𝑡𝑎𝑙𝑆𝑢𝑝𝑝𝑙𝑦<sub>h𝑜𝑙𝑑𝑒𝑟𝑠</sub>
> 2. ∑𝑃𝑂𝑊𝐸𝑅 𝑣𝑜𝑡𝑖𝑛𝑔𝑃𝑜𝑤𝑒𝑟<sub>𝑑𝑒𝑙𝑒𝑔𝑎𝑡𝑒𝑠</sub> >= ∑𝑃𝑂𝑊𝐸𝑅 𝑏𝑎𝑙𝑎𝑛𝑐𝑒𝑂𝑓<sub>h𝑜𝑙𝑑𝑒𝑟𝑠</sub> , 𝑎𝑡 𝑉𝑜𝑡𝑖𝑛𝑔 𝑒𝑝𝑜𝑐h
> 3. ∑𝑃𝑂𝑊𝐸𝑅 𝑣𝑜𝑡𝑖𝑛𝑔𝑃𝑜𝑤𝑒𝑟<sub>𝑑𝑒𝑙𝑒𝑔𝑎𝑡𝑒𝑠</sub> == ∑𝑃𝑂𝑊𝐸𝑅 𝑏𝑎𝑙𝑎𝑛𝑐𝑒𝑂𝑓<sub>h𝑜𝑙𝑑𝑒𝑟𝑠</sub> , 𝑎𝑡 𝑇𝑟𝑎𝑛𝑠𝑓𝑒𝑟 𝑒𝑝𝑜𝑐h
> 4. 𝑎𝑚𝑜𝑢𝑛𝑡𝑇𝑜𝐴𝑢𝑐𝑡𝑖𝑜𝑛<sub>𝑡1</sub> = 𝑎𝑚𝑜𝑢𝑛𝑡𝑇𝑜𝐴𝑢𝑐𝑡𝑖𝑜𝑛<sub>𝑡0</sub> + 𝐼𝑛𝑓𝑙𝑎𝑡𝑖𝑜𝑛<sub>𝑖𝑛𝑎𝑐𝑡𝑖𝑣𝑒𝑃𝑎𝑟𝑡𝑖𝑐𝑖𝑝𝑎𝑛𝑡𝑠</sub>

However, item 3 can be broken due to precision loss when calculating inflation. 
The inflation rate of Power Token is set to 10% by the sponsor. Here is an example illustrating how the inflation problem occurs:
- Alice has 4 power tokens
- Bob has 6 power tokens
- Alice delegates her voting power to Bob
- Bob has 10 voting power after Alice's delegation
- Bob participates all active proposals and get 10% inflation: 10 * 10% = 1
- The totalSupply increases 1
- The voting power of Bob increases to 11
- However, the power balance of Bob keeps unchanged after inflation: 6 + 6 * 10% = 6
- The power balance of Alice keeps unchanged after inflation: 4 + 4 * 10% = 4
As we can see, the total voting power of Alice and Bob is 11, however the sum of their power token balance is 10


Update  [Integration.t.sol](https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/test/integration/Integration.t.sol) to change initial balances firstly:
```diff
-   uint256[] internal _initialPowerBalances = [55, 25, 20];
+   uint256[] internal _initialPowerBalances = [5505, 2495, 2000]; 
``` 
Then copy below codes to [Integration.t.sol](https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/test/integration/Integration.t.sol) and run `forge test --match-test test_SumOfAllBalanceNotEqualToSumOfAllVotingPower`
```solidity
    function test_SumOfAllBalanceNotEqualToSumOfAllVotingPower() external {
        IStandardGovernor standardGovernor_ = IStandardGovernor(_registrar.standardGovernor());
        IPowerToken powerToken_ = IPowerToken(_registrar.powerToken());

        _warpToNextTransferEpoch();
        //@audit-info Alice delegates her voting power to Bob
        vm.prank(_alice);
        powerToken_.delegate(_bob);

        address[] memory targets_ = new address[](1);
        targets_[0] = address(standardGovernor_);

        uint256[] memory values_ = new uint256[](1);

        bytes32 key_ = "TEST_KEY";
        bytes32 value_ = "TEST_VALUE";

        bytes[] memory callDatas_ = new bytes[](1);
        callDatas_[0] = abi.encodeWithSelector(standardGovernor_.setKey.selector, key_, value_);

        string memory description_ = "Update config key/value pair";

        uint256 proposalFee_ = standardGovernor_.proposalFee();

        _cashToken1.mint(_alice, proposalFee_);
        
        vm.startPrank(_alice);
        _cashToken1.approve(address(standardGovernor_), proposalFee_);
        uint256 proposalId_ = standardGovernor_.propose(targets_, values_, callDatas_, description_);
        vm.stopPrank();
        assertEq(_cashToken1.balanceOf(_alice), 0);
        assertEq(_cashToken1.balanceOf(address(standardGovernor_)), proposalFee_);

        _warpToNextVoteEpoch();

        vm.prank(_bob);
        standardGovernor_.castVote(proposalId_, 1);
        vm.prank(_carol);
        standardGovernor_.castVote(proposalId_, 1);

        _warpToNextTransferEpoch();

        uint totalSupply = powerToken_.balanceOf(_alice) + powerToken_.balanceOf(_bob) + powerToken_.balanceOf(_carol);
        uint totalVotingPower = powerToken_.getVotes(_alice) + powerToken_.getVotes(_bob) + powerToken_.getVotes(_carol);
        //@audit-info the sum of all power token balance is not equal to the sum of all voting power
        assertNotEq(totalSupply, totalVotingPower);
        vm.prank(_alice);
        powerToken_.delegate(address(0));
        vm.startPrank(_bob);
        powerToken_.transfer(_alice, powerToken_.balanceOf(_bob));
        vm.stopPrank();
        //@audit-info Bob has no any delegaotr and power token, but he has 1 voting power.
        assertEq(powerToken_.balanceOf(_bob), 0);
        assertEq(powerToken_.getVotes(_bob), 1);
    } 
```

## Impact
The invariant that `∑𝑃𝑂𝑊𝐸𝑅 𝑣𝑜𝑡𝑖𝑛𝑔𝑃𝑜𝑤𝑒𝑟𝑑𝑒𝑙𝑒𝑔𝑎𝑡𝑒𝑠 == ∑𝑃𝑂𝑊𝐸𝑅 𝑏𝑎𝑙𝑎𝑛𝑐𝑒𝑂𝑓h𝑜𝑙𝑑𝑒𝑟𝑠 , 𝑎𝑡 𝑇𝑟𝑎𝑛𝑠𝑓𝑒𝑟 𝑒𝑝𝑜𝑐h` is broken

## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/abstract/EpochBasedInflationaryVoteToken.sol#L105-L119
## Tool used

Manual Review

## Recommendation
Finding a solution to this problem seems incredibly challenging because precision loss can occur anytime.