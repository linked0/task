# Prerequisites
The balances of all traders are $10,000 USDC.
The creator has $100,000 USDC who is also an administrator of this platform.

You calculate the PnL of each trader based on the final resolution. And the creator would get the 2% of the Winners' profit. And the platform would also get the 2% of the Winner's profit. Currently, the creator and the platform are the same person, which mean the creator would get the 4% of the Winners' profit.

# Initial State for each test
💰 Initial mUSDC Token Balances:
SERVER_WALLET   100,000.00 mUSDC
TRADER_1        10,000.00 mUSDC
TRADER_2        10,000.00 mUSDC
TRADER_3        10,000.00 mUSDC
TRADER_4        10,000.00 mUSDC
TRADER_5        10,000.00 mUSDC
TRADER_6        10,000.00 mUSDC
TRADER_7        10,000.00 mUSDC
TRADER_8        10,000.00 mUSDC
TRADER_9        10,000.00 mUSDC
TRADER_10       10,000.00 mUSDC

**Market Configuration:**
- 5 Outcomes (Argentina, Brazil, France, England, Spain)
- Initial Price: **20% ($0.20)** per outcome
- Initial Liquidity: **$1000** per outcome

# Trade Tests

## Test 1: Basic Trading
World Cup 2026

### Steps
All traders have $10,000 USDC in their wallets at start.

1. Trader 1 Buy
- **Market Buy** $10 worth Brazil Yes token at ~21.0c (Ask price)
- **Note**: Initial price is 20c. Ask will be slightly higher (e.g., 21c).

2. Trader 2 Buy
- **Market Buy** $10 worth France Yes token at ~21.0c

3. Trader 3 Buy
- **Market Buy** $20 worth France No token at ~81.0c
- **Note**: Since Yes is ~20c, No is ~80c.

4. Trader 3 Try to Sell but not matched
- $10 worth France No token at 85.0c

### Resolution
Brazil Wins

### Result
- Trader 1: Filled ~47.6 shares @ 21.0c (Brazil YES)
    - Initial Balance: $10,000.00
    - Balance After Buy: $9,990.00
    - Gross PnL (If Brazil Wins): +$37.60 (47.6 * $1.00 - $10.00)
    - Fee (4% of Profit): -$1.50 ($37.60 * 0.04)
    - Net Profit: +$36.10
    - Balance After Claim: **$10,036.10**

- Trader 2: Filled ~47.6 shares @ 21.0c (France YES)
    - Initial Balance: $10,000.00
    - Balance After Buy: $9,990.00
    - Gross PnL (If Brazil Wins): -$10.00
    - Fee: $0.00
    - Net Profit: -$10.00
    - Balance After Claim: **$9,990.00**

- Trader 3:
    - Initial Balance: $10,000.00
    - Balance After Buy: $9,980.00
    - Step 3: Filled ~24.7 shares @ 81.0c (France NO buy)
    - Step 4: **Limit order placed** at 85.0c (France NO sell)
    - Total France NO position: 24.7 shares
    - Gross PnL (If Brazil Wins): +$4.70 (24.7 * $1.00 - $20.00)
    - Fee (4% of Profit): -$0.19 ($4.70 * 0.04)
    - Net Profit: +$4.51
    - Balance After Claim: **$10,004.51**

## Test 2: Liquidity & Limit Orders
World Cup 2026

### Steps
All traders have $10,000 USDC in their wallets at start.

1. Trader 1 Buy
- $20 worth Brazil Yes token at ~21.0c
- Result: Trader 1 owns ~95 shares (Cost: $20.00)

2. Trader 1 Limit Sell
- Sell **20 shares** of Brazil Yes token at **30.0c**
- This order should be placed in the orderbook as the current price (~21c) is lower than 30c.

3. Trader 2 Buy
- **Market Buy** **$120** worth of Brazil Yes token
- The seed liquidity is $1000. A $120 buy will move the price up but easily fill.
- It might hit Trader 1's limit sell at 30c if the price impact is high enough, but with $1000 liquidity, price impact will be lower than before.

### Resolution
Brazil Wins

### Result
- Trader 1:
    - **Initial Balance**: $10,000.00
    - **Step 1 (Buy)**: Buys $20 worth @ ~21.0c -> ~95 shares. Balance: $9,980.00.
    - **Step 2 (Limit Sell)**: Sells 20 shares @ 30.0c -> Receives $6.00 (if filled).
    - **Resolution (Win)**: Pays out $1.00 per share.

## Test 3: Order Cancellation (Safety Check)
### Objective
Verify that funds are correctly locked when placing an order and fully refunded when cancelling it.

### Steps
1. **Initial State**: Record Trader 1's Cash Balance.
2. **Place Limit Buy**:
    - Trader 1 places a **Limit Buy** for **100 shares** of "Brazil Yes" at **5.0c** (Deep OTM).
    - **Cost**: $5.00.
3. **Verify Order**:
    - Check that the order appears in the "Open Orders" tab.
4. **Cancel Order**:
    - Click "Cancel" on the order in the "Open Orders" tab.
5. **Verify Refund**:
    - Check that Trader 1's Cash Balance has **increased by $5.00**.

### Expected Result
✅ Funds are temporarily locked and then 100% returned.

## Test 4: Market Sell (Hitting the Bid)
### Objective
Verify that a user can exit a position by selling shares into existing liquidity (Seed Bids).

### Steps
1. **Buy Position**:
    - Trader 1 buys **$50** of "Brazil Yes" (Market Buy).
    - Price ~21c. Shares: ~238.
2. **Market Sell**:
    - Select "Max" (or 100%).
    - Execute **Market Sell**.
3. **Verify Execution**:
    - The trade should match against the **Seed Bids** (Buy Orders at ~19c).
    - Trader 1 should receive USDC back.
    - **Note**: Loss due to Spread (Buy 21c, Sell 19c).

### Expected Result
✅ Trader 1 successfully converts Shares back to USDC.
✅ Final Balance: **~$9,995.24** (Loss ~2c per share * 238 shares = ~$4.76).

## Test 7: Hedging (Buying Both Sides)
### Objective
Verify the outcome when a trader buys equal amounts of both outcomes (Yes and No).

### Steps
1. **Initial State**:
    - Market Liquidity: **$1000.00** per outcome.
    - Current Price: Yes ($0.20), No ($0.80).
2. **Buy Yes**:
    - Trader 1 executes **Market Buy** of **$50.00** for "Brazil Yes".
    - Price: ~21c. Shares: ~238.
3. **Buy No**:
    - Trader 1 executes **Market Buy** of **$50.00** for "Brazil No".
    - Price: ~81c. Shares: ~61.
4. **Resolve Market**:
    - Administrator resolves the market to **"Brazil Yes"**.
5. **Claim**:
    - Trader 1 claims winnings for "Brazil Yes".
    - "Brazil No" shares expire worthless.
6. **Verify Balance**:
    - **Payout**: 238 Yes shares * $1.00 * 0.96 (4% fee) = $228.48.
    - **Cost**: $100.00.
    - **Profit**: $128.48.
    - **Wait, why profit?**
        - Because you bought Yes at 20c (5x payout) and No at 80c (1.25x payout).
        - If you put equal dollars ($50 each), you are NOT perfectly hedged.
        - To hedge, you need equal **shares** (or dollar amounts proportional to price).
        - Here, you bet $50 on a 20% chance (High reward) and $50 on an 80% chance (Low reward).
        - Since the 20% chance won, you made a huge profit.
        - If Brazil LOST (80% chance), you would have only ~61 shares of No. Payout $61 * 0.96 = $58.56. Loss of ~$41.

### Expected Result
✅ Trader 1 makes profit because the "Underdog" (20%) won.

## Test 8: Market Trading (Buying Both Sides with Dollar Amount)
### Objective
Verify the outcome when a trader executes Market Buys for a specific dollar amount ($30) on both outcomes.

### Steps
1. **Initial State**: Trader 1 has $10,000.00 mUSDC.
2. **Buy Yes**:
    - Trader 1 Market Buys **$30.00** of "Brazil Yes".
    - Price ~21c. Shares: ~142.
3. **Buy No**:
    - Trader 1 Market Buys **$30.00** of "Brazil No".
    - Price ~81c. Shares: ~37.
4. **Verify Portfolio**:
    - **Cash Balance**: $9,940.00.
    - **Current Value**: ~$60.00.
5. **Resolve Market**:
    - Administrator resolves the market to **"Brazil Yes"**.
6. **Claim**:
    - Payout: 142 * $0.96 = $136.32.
    - Final Balance: $9,940 + $136 = $10,076.
    - **Again, profit because Underdog won.**

### Note on Hedging with 20/80 Split
Unlike the 50/50 split, buying equal dollar amounts is a **directional bet** on the underdog (Yes) because the Yes token has much higher leverage (5x) than the No token (1.25x).
