Dry Lime Stork

medium

# Penalty charge is not considered during recalculation of `principalOfTotalActiveOwedM` while deactivating a minter results misscount throughout various functions.

## Summary
During deactivating a minter `_rawOwedM[minter]` is deducted from `principalOfTotalActiveOwedM`, however if the minter missed any update before calling the deactivate function the penalty rate is not accounted in deduction. But it is accounted to calculate `totalInactiveOwedM_`.
## Vulnerability Detail

See this part of `deactivateMinter()`:
```solidity
File: src/MinterGateway.sol
449:         uint112 principalOfActiveOwedM_ = principalOfActiveOwedMOf(minter_); // uint112(_rawOwedM[minter_])
450: 
451:         unchecked {
452:             // As an edge case precaution, if the resulting principal plus penalties is greater than the max uint112,
453:             // then max out the principal.
454:             // newPrincipalOfOwedM = rawedMOf[minter] + fee for missed update, means it is checking that if minter has failed any update, if yes then add the fee
455:             // While calculating 'inactiveMint' the fee is also considered.
456:             // If the minter missed any update then the fee will be accounted in rawedMOf[minter] and thus it increases the 'principalOfTotalActiveOwedM'
457:             uint112 newPrincipalOfOwedM_ = UIntMath.bound112(
458:                 uint256(principalOfActiveOwedM_) + _getPenaltyPrincipalForMissedCollateralUpdates(minter_) // @note this is the updated rawedMOf[minter_]
459:             );
460: 
461:             inactiveOwedM_ = _getPresentAmount(newPrincipalOfOwedM_); // _getPresentAmountRoundedUp(principalAmount_, currentIndex())
462: 
463:             // Treat rawOwedM as principal since minter is active.
464:             // @audit Are principalOfActiveOwedM_ & inactiveOwedM_ same ?? => NO
465:             
466:             principalOfTotalActiveOwedM -= principalOfActiveOwedM_; // @audit newPrincipalOfOwedM_ should be deducted
467:             totalInactiveOwedM += inactiveOwedM_;
468:         }
469: 
```
As we know: principalOfActiveOwedM_ = _rawOwedM[minter]
When a minter is penalized for missed collateral update, its penalty charge i.e penaltyPrincipal which is `(principalOfActiveOwedM_ * missedInterval * penaltyRate) / 10_000` is added in his M balance i.e it increases the value of principalOfActiveOwedM_. 
See this line from _imposePenalty():
```solidity
File: src/MinterGateway.sol
836:             _rawOwedM[minter_] += uint112(penaltyPrincipal_); // Treat rawOwedM as principal since minter is active.
```
But while deactivating the minter the current logic is ignoring the penaltyCharge & it deducting just M balance without penalty charge. If deactivateMinter would not be called then the charge would have been added to the active M balance.
We can also see that the `newPrincipalOfOwedM` was added in totalInactiveOwedM, i.e it considered the penalty charge.

> _Due to ease of calculation & to prove some point i changed visibility, from internal to public, of 2 functions: 1. `_getPresentAmount()` & 2. `_getPenaltyPrincipalForMissedCollateralUpdates()`_

<details>
  <summary>POC</summary>

Run this test in MinterGateway.t.sol.

  ```solidity
function test_M() external {

        uint240 collateral = 1e5;
        uint256[] memory retrievalIds = new uint256[](0);
        uint40 signatureTimestamp = uint40(block.timestamp);

        address[] memory validators = new address[](1);
        validators[0] = _validator1;

        uint256[] memory timestamps = new uint256[](1);
        timestamps[0] = signatureTimestamp;

        bytes[] memory signatures = new bytes[](1);
        signatures[0] = _getCollateralUpdateSignature(
            address(_minterGateway),
            _minter1,
            collateral,
            retrievalIds,
            bytes32(0),
            signatureTimestamp,
            _validator1Pk
        );
        console.log("-------------------- 1st call to updateCollateral -------------------");
        

        vm.prank(_minter1);
        _minterGateway.updateCollateral(collateral, retrievalIds, bytes32(0), validators, timestamps, signatures);

        console.log("activeOwedMOf after calling updateCollateral() first time:", _minterGateway.activeOwedMOf(_minter1));
        console.log("PrincipalOfActiveOwedMOfMinter after calling updateCollateral() first time:", _minterGateway.principalOfActiveOwedMOf(_minter1));
       
        
        uint startTime = block.timestamp;

        skip(100); 
        vm.prank(_minter1);
        uint48 mintId = _minterGateway.proposeMint(5000, _minter1);

        (,,, uint240 amount_) = _minterGateway.mintProposalOf(_minter1);
       // console.log("After proposing mint 1st time the to be minted amount:", amount_);
        
        skip(1000);
        vm.prank(_minter1);

//        // @note calling updateIndex() multiple times has no effect on activeOwedOf
       
        (uint112 principalAmount, uint240 amount) = _minterGateway.mintM(uint256(mintId));

        console.log("activeOwedMOf after calling mint() first time:", _minterGateway.activeOwedMOf(_minter1));
        console.log("PrincipalOfActiveOwedMOfMinter after calling mint() first time:", _minterGateway.principalOfActiveOwedMOf(_minter1));

        
        console.log("-------------------- Now 2nd call to updateCollateral -------------------");
        
        skip(500);  
        uint256[] memory timestamps1 = new uint256[](1);
        timestamps1[0] = uint40(block.timestamp);
        bytes[] memory signatures1 = new bytes[](1);
        signatures1[0] = _getCollateralUpdateSignature(
            address(_minterGateway),
            _minter1,
            9000,
            retrievalIds,
            bytes32(0),
            uint40(block.timestamp),
            _validator1Pk
        );

        uint endTime = block.timestamp;
         vm.prank(_minter1);
         _minterGateway.updateCollateral(9000, retrievalIds, bytes32(0), validators, timestamps1, signatures1);
          
          console.log("activeOwedMOf after calling updateCollateral() 2nd time:", _minterGateway.activeOwedMOf(_minter1));
           console.log("PrincipalOfActiveOwedMOfMinter after calling updateCollateral() 2nd time:", _minterGateway.principalOfActiveOwedMOf(_minter1));
        
        vm.startPrank(_minter1);
          uint48 mintId2 = _minterGateway.proposeMint(2000, _minter1);
          skip(1000);
         (uint112 _principalAmount, uint240 _amount) = _minterGateway.mintM(uint256(mintId2)); 
        vm.stopPrank();
          console.log("activeOwedMOf after calling mint() 2nd time:", _minterGateway.activeOwedMOf(_minter1));
           console.log("PrincipalOfActiveOwedMOfMinter after calling mint() 2nd time:", _minterGateway.principalOfActiveOwedMOf(_minter1));
           skip(2000);
           console.log("principalOfActiveOwedMOf:", _minterGateway.principalOfActiveOwedMOf(_minter1));
        uint112 newPrincipalOfOwedM = _minterGateway.principalOfActiveOwedMOf(_minter1) + _minterGateway._getPenaltyPrincipalForMissedCollateralUpdates(_minter1);
        uint240 inactiveOwedM = _minterGateway._getPresentAmount(newPrincipalOfOwedM);
        console.log("newPrincipalOfOwedM:",newPrincipalOfOwedM);
        console.log("inactiveOwedM:",inactiveOwedM);
        console.log("Penalty for missed collateral update:",_minterGateway._getPenaltyPrincipalForMissedCollateralUpdates(_minter1));
}

 ```
Logs:
```solidity
  -------------------- 1st call to updateCollateral -------------------
  activeOwedMOf after calling updateCollateral() first time: 0
  PrincipalOfActiveOwedMOfMinter after calling updateCollateral() first time: 0
  activeOwedMOf after calling mint() first time: 5001
  PrincipalOfActiveOwedMOfMinter after calling mint() first time: 5000
  -------------------- Now 2nd call to updateCollateral -------------------
  activeOwedMOf after calling updateCollateral() 2nd time: 5001
  PrincipalOfActiveOwedMOfMinter after calling updateCollateral() 2nd time: 5000
  activeOwedMOf after calling mint() 2nd time: 7001
  PrincipalOfActiveOwedMOfMinter after calling mint() 2nd time: 7000
  principalOfActiveOwedMOf: 7000
  newPrincipalOfOwedM: 7070
  inactiveOwedM: 7071
  Penalty for missed collateral update: 70
```
As we can see, due to penalty charge `newPrincipalOfOwedM` is 7070 but **M** balance is 7000. 
</details>


## Impact
1. `_getPresentAmount()` will return low value. Run this in given test file:
```solidity
console.log("getPresentAmount for newPrincipalOfOwedM:",inactiveOwedM);
console.log("getPresentAmount for principalOfActiveOwedM:", _minterGateway._getPresentAmount(_minterGateway.principalOfActiveOwedMOf(_minter1)));
```
Logs:
```solidity
getPresentAmount for newPrincipalOfOwedM: 7071
getPresentAmount for principalOfActiveOwedM: 7001
```
2. `excessOwedM()` will be miscalculated.
```solidity
File: src/MinterGateway.sol
542:         uint240 totalOwedM_ = _getPresentAmountRoundedDown(principalOfTotalActiveOwedM, currentIndex()) + 
543:             totalInactiveOwedM;
544:      
545:         unchecked {
546:             if (totalOwedM_ > totalMSupply_) return totalOwedM_ - totalMSupply_;
547:         }

```
3. penalty charge can be miscalculated in `_imposePenalty()`.
```solidity
File: src/MinterGateway.sol
825:             if (newPrincipalOfTotalActiveOwedM_ > type(uint112).max) {      
826:                 penaltyPrincipal_ = type(uint112).max - principalOfTotalActiveOwedM; // @audit here
827:                 newPrincipalOfTotalActiveOwedM_ = type(uint112).max;  
828:             }

```
## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L411
## Tool used

Manual Review, Foundry

## Recommendation
Change this line:
```solidity
principalOfTotalActiveOwedM -= principalOfActiveOwedM_;
```
to
```solidity
principalOfTotalActiveOwedM -= newPrincipalOfOwedM_;
```