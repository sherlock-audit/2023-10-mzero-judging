Young Umber Okapi

high

# Minting can incur bad debt

## Summary

Minting can incur bad debt as the collateral check does not consider the existing penalties incurred by the minters. As a result, the protocol could incur excessive bad debt, the M tokens will not be backed by sufficient collateral assets, there will be an increased risk of insolvency, and the M token might depeg.

## Vulnerability Detail

When the minters call the `proposeMint` and `mintM` functions to mint M tokens, the `_revertIfUndercollateralized` function will be executed to ensure the account is not undercollateralized.

However, the `_revertIfUndercollateralized` function does not consider the account's pending penalty to the owned principal due to missed collateral updates. Thus, the owned M used in the check is undervalued, allowing the account to continue to mint additional M tokens even if the account is already under-collateralized when factoring in the pending penalties incurred.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L1018

```solidity
File: MinterGateway.sol
1018:     function _revertIfUndercollateralized(address minter_, uint240 additionalOwedM_) internal view {
1019:         uint256 maxAllowedActiveOwedM_ = maxAllowedActiveOwedMOf(minter_);
1020: 
1021:         // If the minter's max allowed active owed M is greater than the max uint240, then it's definitely greater than
1022:         // the max possible active owed M for the minter, which is capped at the max uint240.
1023:         if (maxAllowedActiveOwedM_ >= type(uint240).max) return;
1024: 
1025:         unchecked {
1026:             uint256 finalActiveOwedM_ = uint256(activeOwedMOf(minter_)) + additionalOwedM_;
1027: 
1028:             if (finalActiveOwedM_ > maxAllowedActiveOwedM_) {
1029:                 revert Undercollateralized(finalActiveOwedM_, maxAllowedActiveOwedM_);
1030:             }
1031:         }
1032:     }
```

In addition, the `activeOwedMOf` function used in Line 1026 above in the `_revertIfUndercollateralized` function also does not take into consideration of the account's pending penalties.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L513

```solidity
File: MinterGateway.sol
513:     function activeOwedMOf(address minter_) public view returns (uint240) {
514:         // NOTE: This should also include the present value of unavoidable penalities. But then it would be very, if
515:         //       not impossible, to determine the `totalActiveOwedM` to the same standards.
516:         return
517:             _minterStates[minter_].isActive
518:                 ? _getPresentAmount(uint112(_rawOwedM[minter_])) // Treat rawOwedM as principal since minter is active.
519:                 : 0;
520:     }
```

The following simplified example illustrates the issue. Assume the minter rate is zero for simplicity's sake.

1. At T1, Bob's `maxAllowedActiveOwedM_` is 100 based on his collateral assets, and he minted 50 M tokens. Thus, his `activeOwedMOf` will be 50 M tokens.

2. At T50, Bob did not call the collateral updates for many epochs. As a result, he accumulated a penalty of 100 M tokens for all his missed updates. If one calls the `getPenaltyForMissedCollateralUpdates` against Bob's account, it will return 100 M tokens as Bob's penalty. If someone intends to repay Bob's debt now, they must repay a total debt of 150 M tokens (50 + 100). Bob's account is technically underwater at this point (Debt > `maxAllowedActiveOwedM_`).

3. At T51, Bob attempts to mint an additional 45 M tokens. When the `_revertIfUndercollateralized` function is executed to check whether Bob's account is undercollateralized via the following logic and determine that Bob's account is still healthy:

   ```solidity
   maxAllowedActiveOwedM_ = 100
   activeOwedMOf(Bob) = 50
   
   finalActiveOwedM_ = activeOwedMOf(Bob) + additionalOwedM_ = 50 + 45 = 95
   
   finalActiveOwedM_ > maxAllowedActiveOwedM_
   95 > 100 => False => No Revert
   ```

4. The protocol will proceed to mint 45 M tokens for Bob even when his account is already technically undercollateralized. If someone intends to repay Bob's debt now, they must repay a total debt of 195 M tokens (50 + 100 + 45), which shows that the debt further increases after the mint. This indicates that the collateral checks and the mechanism to ensure the protocol's solvency are inadequate.

## Impact

The protocol could incur excessive bad debt, the M tokens will not be backed by sufficient collateral assets, there will be an increased risk of insolvency, and the M token might depeg.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L1018

## Tool used

Manual Review

## Recommendation

When determining if an account is undercollateralized during minting, the protocol should consider the account's debt in its entirety, including the debt from the accumulated penalties. This will prevent minters whose accounts are already in an unhealthy position due to incurred penalties from minting even more M tokens, which, in turn, incur even more debt that results in their account being in the worst state after the transaction.