Young Umber Okapi

high

# A different cash token pulled from the buyer wallet

## Summary

If the cash token is switched between the time the buyer submits its order and the order is executed, a different cash token will be pulled from the buyer's wallet, leading to a loss of assets for the affected users.

## Vulnerability Detail

Users can configure the `expiryEpoch_` of their buy order so that it is only valid until the end of the current transfer epoch OR is  valid across multiple transfer epochs (multiple auctions). 

Assume that Bob grants the Power Token contract max allowance for both WETH and M tokens, as the protocol's governance has switched the cash token between these two tokens in the past.

Assume that the current cash token is M token and the starting price is 100 M Tokens. Bob is willing to pay a starting price of 100 M Tokens for the amount of power tokens he wants to purchase. Thus, he immediately submits a buy order once the auction starts and sets the `expiryEpoch_` to allow his buy transaction to span multiple auctions.

Bob's buy transaction resides in the mempool for a period of time and is not executed in the current auction due to various reasons (e.g., low gas fee, congested network). It will be executed in a few auctions later. However, before that, a zero proposal to switch the cash token from M Token to WETH was executed, and the cash token changes took effect. A cash token change only requires 2 epochs to take effect, while an auction's buy order TX can span more than 2 epochs.

When Bob's buy transaction is executed next, the protocol will pull 100 WETH instead of 100 M Tokens from Bob's wallet and send them to the Zero Token's distribution vault to be distributed to zero token holders. The timing of the proposal execution to switch cash tokens might be accidental or intentional. One point to note here is that since zero token holders are getting the 100 WETH from Bob, they might be incentivized to do so intentionally. Thus, one cannot rule out such a possibility.

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

Loss of assets for the affected users.

## Code Snippet

https://github.com/sherlock-audit/2023-10-mzero/blob/main/ttg/src/PowerToken.sol#L124

## Tool used

Manual Review

## Recommendation

The issue will only occur if the buy order transaction spans multiple auctions and the cash token is updated.

Consider the following change to ensure the buy order is only executed with the intended cash token. In the above scenario, if the buy order were submitted with `paymentCashToken_` set to M Token, the transaction would revert to avoid pulling the incorrect tokens from the buyer.

```diff
function buy(
    uint256 minAmount_,
    uint256 maxAmount_,
    address destination_,
+	address paymentCashToken_,    
   uint16 expiryEpoch_
) external returns (uint240 amount_, uint256 cost_) {
    // NOTE: Buy order has an epoch-based expiration logic to avoid user's unfair purchases in subsequent auctions.
    //       Order should be typically valid till the end of current transfer epoch.
   if (_clock() > expiryEpoch_) revert ExpiredBuyOrder();

    if (minAmount_ == 0 || maxAmount_ == 0) revert ZeroPurchaseAmount();
	..SNIP..
+	if (paymentCashToken_ != cashToken()) revert CashTokenChanged();
    if (!ERC20Helper.transferFrom(cashToken(), msg.sender, vault, cost_)) revert TransferFromFailed();
}
```