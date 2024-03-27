Magic Sand Wolverine

medium

# The functions `MinterGateway._rate()` and `MToken._rate()` may return a rate value of 0 which may cause issues

## Summary

There are issues with `MinterGateway._rate()` and `MToken._rate()` where the returned rate may be 0, which may cause a number of further issues due to wrong calculations that are based on the rate.

## Vulnerability Detail

The functions `MinterGateway._rate()` and `MToken._rate()` may return 0 (line 986 in MinterGateway.sol and line 455 MToken.sol), if the staticcall to the rate model fails which is supposed to fetch the rate.

As a result when `ContinuousIndexing.updateIndex()` is called (either for the `MToken` or for the `MinterGateway` contract, since both are inheriting from the `ContinuousIndexing` contract), the `rate_` may be 0 (line 39 ContinuousIndexing.sol) since it fetches it's value from `MinterGateway._rate()`, which is then assigned to `_latestRate` state variable on line 45 in ContinuousIndexing.sol.

Both `MToken.currentIndex()` and `MinterGateway.currentIndex()` may then calculate a wrong current index value since both functions are relying on `_latestRate` which may be 0 as shown above. Also the state variable  `ContinuousIndexing.latestIndex` will have a wrong value assigned (line 44 ContinuousIndexing.sol).

## Impact

Wherever the protocol calls the `currentIndex()` method and wherever the state var `ContinuousIndexing.latestIndex` is used, there may be an issue:

* `MinterGateway.excessOwedM()` may calculate a wrong value for `totalOwedM_` which is based on the `currentIndex()`, line 474 MinterGateway.sol.

* `MinterGateway._getPresentAmount()` is relying on `currentIndex()`, line 946 MinterGateway.sol. Thus `MinterGateway.totalActiveOwedM()` may be calculated wrongly which calls `_getPresentAmount()` line 455 MinterGateway.sol, thus `StableEarnerRateModel.rate()` may be calculated wrongly which is relying on `MinterGateway.totalActiveOwedM()` line 69 StableEarnerRateModel.sol.

* `MToken._getPresentAmount()` is relying on `currentIndex()`, line 421 MToken.sol. Thus the MToken balance `balanceOf()` an account might be calculated wrongly (line 148 MToken.sol), which may be abused by a malicious actor.

* `ContinuousIndexing._getPrincipalAmountRoundedDown()` and `ContinuousIndexing._getPrincipalAmountRoundedUp()` uses `currentIndex()`, line 64 and 73 in ContinuousIndexing.sol. Thus `MinterGateway.burnM()` may burn a wrong amount of M tokens since it is relying on `_getPrincipalAmountRoundedDown()` line 325 MinterGateway.sol. Also when `MinterGateway.mintM()` is called, a wrong `principalAmount_` may be calculated using `ContinuousIndexing._getPrincipalAmountRoundedUp()` (line 293 MinterGateway.sol), which is then assigned to `_rawOwedM[msg.sender]` line 311 MinterGateway.sol, which is the owed M of the minter. A malicious minter may exploit this issue.

In summary, a number of issues may arise if the rate fetched by `MinterGateway._rate()` or `MToken._rate()` would be 0 and not reverted.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L981-L987

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MToken.sol#L450-L456

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MinterGateway.sol#L683-L698

https://github.com/sherlock-audit/2023-10-mzero/blob/main/protocol/src/MToken.sol#L158-L173


## Tool used

Manual Review

## Recommendation

Consider handling the case where `MinterGateway._rate()` or `MToken._rate()` return 0 as the rate value. For example by reverting or by using the latest rate that was fetched before, if the latest rate is not 0 as well.
