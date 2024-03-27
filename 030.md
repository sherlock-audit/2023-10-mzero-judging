Rough Violet Copperhead

high

# Bootstrapping is likely to destroy a large portion of users' Power Tokens

## Summary
The `INITIAL_SUPPLY` constant used to calculate bootstrap balances is too small, and many Power Token holders' balances will round  down to zero. 
## Vulnerability Detail
`_getBootstrapBalance()` calculates the balance to bootstrap to Power Token holders when the Power Token is migrated/redeployed. The balance is calculated as `INITIAL_SUPPLY * User's Balance * Total Supply`.
```solidity
    /**
     * @dev    This is the portion of the initial supply commensurate with
     *         the account's portion of the bootstrap supply.
     * @param  account_ The account to get the bootstrap balance for.
     * @param  epoch_   The epoch to get the bootstrap balance at.
     * @return The bootstrap balance of `account_` at `epoch_`.
     */
    function _getBootstrapBalance(address account_, uint16 epoch_) internal view returns (uint240) {
        unchecked {
            // NOTE: Can safely cast `pastBalanceOf` since the constructor already establishes that the total supply of
            //       the bootstrap token is less than `type(uint240).max`. Can do math unchecked since
            //       `pastBalanceOf * INITIAL_SUPPLY <= type(uint256).max`.
            return
                uint240(
                    (IEpochBasedVoteToken(bootstrapToken).pastBalanceOf(account_, epoch_) * INITIAL_SUPPLY) /
                        _bootstrapSupply
                );
        }
    }
```
The problem is that the `INITIAL_SUPPLY` constant is `10_000` (see code snippet link), which means that many power token balances will be rounded down to zero and destroyed.

Let's see an example to see how this can be economically significant. Since the Power Token is a decentralized governance token, we can assume that its market cap will be large enough so that it's not feasible for an attacker to buy a large amount of tokens and disrupt governance. A very conservative estimate of the market cap could be 10 million USD.

We can see in this example that any balance of Power Token that is worth less than `1,000` USD will be destroyed, because `1_000*10_000/10_000_000 == 1`.

If the market cap is 100 million USD, then any holding worth less than 10,000 USD will be destroyed.

## Impact
Users can lose a significant amount of money when the Power Token is redeployed.

## Code Snippet
https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L325-L337
https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L43
## Tool used

Manual Review

## Recommendation
Fix is simple, just set `INITIAL_SUPPLY` larger. Total supply overflow is not a concern, it won't happen for many years and when it does the Power Token can be redeployed and bootstrapped again.