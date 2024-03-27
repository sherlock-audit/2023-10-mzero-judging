Droll Iron Ferret

medium

# Voters can receive inflation from voting with 0 voting weight

## Summary

Users can vote on proposals with zero voting weight before syncing their bootstrap balance then receive inflation for the vote based on their bootstrapped balance.

## Vulnerability Detail

It's possible to vote on StandardGovernor proposals with 0 voting power. It's also possible for users to have a balance from the old Power token which can be bootstrapped when they initially call sync. 

If voters have an unsynced bootstrap balance when they vote on all proposals, it will call `_markParticipation`, which will only sync their balance after voting, but will still inflate their balance according to the amount of votes they would have had if they were previously synced, i.e. if they had a voting weight. This happens because bootstrapping sets the user balance and votingPowers all the way back at the bootstrap epoch.

## Impact

Votes in the epoch will be successfully cast with zero voting weight, essentially as though no votes were cast at all, but users will receive the full amount of inflation for voting.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/abstract/EpochBasedInflationaryVoteToken.sol#L105-L119

## Tool used

Manual Review

## Recommendation

Prevent users from voting with zero voting weight.