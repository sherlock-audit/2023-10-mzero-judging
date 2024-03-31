Young Umber Okapi

high

# Users might receive worthless Power Tokens during the auction

## Summary

Users might receive worthless Power Tokens during the auction, leading to a loss of assets for the affected users.

## Vulnerability Detail

Users can configure the `expiryEpoch_` of their buy order so that it is only valid until the end of the current transfer epoch OR is valid across multiple transfer epochs (multiple auctions). 

A reset can occur at any epoch (or any point in time) as long as the threshold of the zero proposal is reached.

Assume that Bob is willing to pay any amount to purchase 100 Power Tokens, and he submits a buy order transaction (TX) on-chain.

Bob's buy order TX will be executed in the current or subsequent auctions as he sets the `expiryEpoch_` to span across multiple auctions. Unfortunately, before Bob's buy order TX is executed, a reset TX is executed, and a new Power Token contract is deployed. The old Power Token is now useless/worthless as it no longer has any voting power over the protocol.

When Bob's buy order TX finally gets executed, the protocol will pull 10 WETH from Bob's account and mint 100 old Power Tokens to Bob. As a result, Bob paid 10 WETH for the old Power Tokens that are useless/worthless.

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L124

```solidity
File: PowerToken.sol
124:     function buy(
125:         uint256 minAmount_,
126:         uint256 maxAmount_,
127:         address destination_,
128:         uint16 expiryEpoch_
129:     ) external returns (uint240 amount_, uint256 cost_) {
130:         // NOTE: Buy order has an epoch-based expiration logic to avoid user's unfair purchases in subsequent auctions.
131:         //       Order should be typically valid till the end of current transfer epoch.
132:         if (_clock() > expiryEpoch_) revert ExpiredBuyOrder();
133:         if (minAmount_ == 0 || maxAmount_ == 0) revert ZeroPurchaseAmount();
134: 
135:         uint240 amountToAuction_ = amountToAuction();
136:         uint240 safeMinAmount_ = UIntMath.safe240(minAmount_);
137:         uint240 safeMaxAmount_ = UIntMath.safe240(maxAmount_);
138: 
139:         amount_ = amountToAuction_ > safeMaxAmount_ ? safeMaxAmount_ : amountToAuction_;
140: 
141:         if (amount_ < safeMinAmount_) revert InsufficientAuctionSupply(amountToAuction_, safeMinAmount_);
142: 
143:         emit Buy(msg.sender, amount_, cost_ = getCost(amount_));
144: 
145:         _mint(destination_, amount_);
146: 
147:         // NOTE: Not calling `distribute` on vault since anyone can do it, anytime, and this contract should not need to
148:         //       know how the vault works
149:         if (!ERC20Helper.transferFrom(cashToken(), msg.sender, vault, cost_)) revert TransferFromFailed();
150:     }
```

## Impact

Loss of assets for the affected users as they receive the worthless power tokens.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L124

## Tool used

Manual Review

## Recommendation

Consider creating a separate auction contract called `Auction` with a `buy` function. When the `Auction.buy` function gets executed, it fetches the current Power Token within the protocol via the `Registrar.powerToken` function and calls its internal mint function to mint the Power Tokens for the buyer.

This approach will ensure that the buyer receives the current/latest version of the Power Token even if a reset occurs. It will also decouple the auction feature from the effect/impact of the reset.