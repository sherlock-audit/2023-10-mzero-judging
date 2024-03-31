Amusing Wooden Dragon

medium

# A malicious user could cause the `M` Minter to incur more penalties by calling `burnM()`

## Summary
A malicious user could cause the Minter to incur more penalties by calling `burnM()`
## Vulnerability Detail
Minters maintain their on-chain collateral value by calling `MinterGateway#updateCollateral()` periodically. They have to  pay penalty if they fail to call `updateCollateral()` within Update Collateral Interval of the previous time they called it. The penalty will be calculated as below at the time of next `updateCollateral()`:

𝑝𝑒𝑛𝑎𝑙𝑡𝑦<sub>𝑚𝑖𝑛𝑡𝑒𝑟</sub> = 𝑎𝑐𝑡𝑖𝑣𝑒𝑂𝑤𝑒𝑑𝑀<sub>𝑚𝑖𝑛𝑡𝑒𝑟</sub> * 𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚 * 𝑝𝑒𝑛𝑎𝑙𝑡𝑦𝑅𝑎𝑡𝑒

Any `M` holder can repay M owed by a Minter by calling `burnM()`.  The penalty will be updated in `burnM()` if the Minter miss collateral updates within Update Collateral Interval.  A malicious user could compound the penalty by periodically calling `burnM()`:

𝑝𝑒𝑛𝑎𝑙𝑡𝑦<sub>𝑚𝑖𝑛𝑡𝑒𝑟</sub> = 𝑎𝑐𝑡𝑖𝑣𝑒𝑂𝑤𝑒𝑑𝑀<sub>𝑚𝑖𝑛𝑡𝑒𝑟</sub> * ((1+𝑝𝑒𝑛𝑎𝑙𝑡𝑦𝑅𝑎𝑡𝑒)<sup> 𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚</sup> - 1) 

As we can see, a malicious user could theoretically increase the penalty of the Minter who missed 𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚 of Update Collateral Interval as below:
𝑎𝑐𝑡𝑖𝑣𝑒𝑂𝑤𝑒𝑑𝑀<sub>𝑚𝑖𝑛𝑡𝑒𝑟</sub> * ((1+𝑝𝑒𝑛𝑎𝑙𝑡𝑦𝑅𝑎𝑡𝑒)<sup> 𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚</sup> - 1)  - 𝑎𝑐𝑡𝑖𝑣𝑒𝑂𝑤𝑒𝑑𝑀<sub>𝑚𝑖𝑛𝑡𝑒𝑟</sub> * 𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚 * 𝑝𝑒𝑛𝑎𝑙𝑡𝑦𝑅𝑎𝑡𝑒 
=  𝑎𝑐𝑡𝑖𝑣𝑒𝑂𝑤𝑒𝑑𝑀<sub>𝑚𝑖𝑛𝑡𝑒𝑟</sub> * ((𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚! * 𝑝𝑒𝑛𝑎𝑙𝑡𝑦𝑅𝑎𝑡𝑒<sup>2</sup> / ((𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚-2)! * 2!) + (𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚! * 𝑝𝑒𝑛𝑎𝑙𝑡𝑦𝑅𝑎𝑡𝑒<sup>2</sup> / ((𝑚𝑖𝑠𝑠𝑒𝑑𝐼𝑛𝑡𝑒𝑟𝑣𝑎𝑙𝑠𝑁𝑢𝑚-3)! * 3!) + ...)

Copy below codes to [Integration.t.sol](https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/test/integration/Integration.t.sol) and run `forge test --match-test test_CompoundedPenaltyByCallingBurnM`

```solidity
    function test_CompoundedPenaltyByCallingBurnM() external {

        _minterGateway.activateMinter(_minters[0]);

        uint256 collateral = 1_000_000e6;
        _updateCollateral(_minters[0], collateral);
        _mintM(_minters[0], 10_000e6, _alice);
        //@audit-info save the initial state
        uint snapshot = vm.snapshot();
        vm.warp(block.timestamp + 10 days);
        //@audit-info The simple-interest penalty amount for missing 10 days of collateral update.
        uint penalty = _minterGateway.getPenaltyForMissedCollateralUpdates(_minters[0]);
        //@audit-info restore the initial state
        vm.revertTo(snapshot);
        uint compoundedPenalty = 0;
        //@audit-info Calculate the compound-interest penalty amount for missing 10 days of collateral update.
        for (uint i=0; i< 10; i++) {
            vm.warp(block.timestamp + 1 days);
            //@audit-info calculate accumulative penalty
            compoundedPenalty += _minterGateway.getPenaltyForMissedCollateralUpdates(_minters[0]);
            //@audit-info Alice triggers penalty updating by repaying a tiny amount of M for _minters[0]
            vm.prank(_alice);
            _minterGateway.burnM(_minters[0], 10);
            //@audit-info the penalty will be added to activeOwedM, then reset to 0 after calling burnM() 
            assertEq(_minterGateway.getPenaltyForMissedCollateralUpdates(_minters[0]), 0);
        }
        //@audit-info The compound-interest penalty over 10 days can be around 4.5% higher than the simple-interest penalty.
        assertApproxEqAbs((compoundedPenalty - penalty) * 1e4 / penalty, 450, 5);
    }
```
## Impact
A malicious user could cause the Minter to incur more penalties by calling `burnM()`
## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L343
## Tool used

Manual Review

## Recommendation
When `burnM()` is called to repay M owed by a Minter, consider imposing a minimum threshold on the amount of M to be repaid if the Minter is subject to penalty of missing collateral update.
