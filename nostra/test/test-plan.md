# Prerequisites
The balances of all traders are $1000 USDC.
The creator has $100000 USDC who is also a administrator of this platform.

You calculate the PnL of each trader based on the final resolution. And the creator would get the 2% of the Winners' profit. And the platform would also get the 2% of the Winner's profit. Currently, the creator and the platform are the same person, which mean the creator would get the 4% of the Winners' profit.

# Initial State for each test
💰 Initial mUSDC Token Balances:
SERVER_WALLET   100,000.00 mUSDC
TRADER_1        1,000.00 mUSDC
TRADER_2        1,000.00 mUSDC
TRADER_3        1,000.00 mUSDC
TRADER_4        1,000.00 mUSDC
TRADER_5        1,000.00 mUSDC
TRADER_6        1,000.00 mUSDC
TRADER_7        1,000.00 mUSDC
TRADER_8        1,000.00 mUSDC
TRADER_9        1,000.00 mUSDC
TRADER_10       1,000.00 mUSDC

# Trade Tests

## Test 1: DONE!

World Cup 2026

### Steps
All traders have $1000 USDC in their wallets at start.

1. Trader 1 Buy
- **Market Buy** $10 worth Brazil Yes token at 51.0c

2. Trader 2 Buy
- **Market Buy** $10 worth France Yes token at 51.0c

3. Trader 3 Buy
- **Market Buy** $20 worth France No token at 51.0c

4. Trader 3 Try to Sell but not matched
- $10 worth France No token at 55.0c

### Resolution
Brazil Wins


### Result
- Trader 1: Filled 19.61 shares @ 51.0c (Brazil YES)
    - Initial Balance: $1000.00
    - Balance After Buy: $990.00
    - Gross PnL (If Brazil Wins): +$9.61 (19.61 * $1.00 - $10.00)
    - Fee (4% of Profit): -$0.38 ($9.61 * 0.04)
    - Net Profit: +$9.23
    - Balance After Claim: **$1009.23**

- Trader 2: Filled 19.61 shares @ 51.0c (France YES)
    - Initial Balance: $1000.00
    - Balance After Buy: $990.00
    - Gross PnL (If Brazil Wins): -$10.00
    - Fee: $0.00
    - Net Profit: -$10.00
    - Balance After Claim: **$990.00**

- Trader 3:
    - Initial Balance: $1000.00
    - Balance After Buy: $980.00
    - Step 3: Filled 39.21 shares @ 51.0c (France NO buy)
    - Step 4: **Limit order placed** at 55.0c for ~18.18 shares (France NO sell)
        - Best bid was 49.0c, limit price 55.0c > best bid
        - Order placed in orderbook, waiting for match
        - Status: OPEN (unfilled)
    - Total France NO position: 39.21 shares
    - Gross PnL (If Brazil Wins): +$19.21 (39.21 * $1.00 - $20.00)
    - Fee (4% of Profit): -$0.77 ($19.21 * 0.04)
    - Net Profit: +$18.44
    - Balance After Claim: **$1018.44**
    - **Note:** The 55.0c limit sell order is automatically cancelled upon market resolution.

- Creator (Administrator):
    - Initial Balance: $100,000.00
    - Trading PnL (Counterparty to Traders): -$18.82
        - vs Trader 1: -$9.61
        - vs Trader 2: +$10.00
        - vs Trader 3: -$19.21
    - Fee Income (4% of Winners' Profit): +$1.15
        - From Trader 1: +$0.38
        - From Trader 3: +$0.77
    - Net PnL: -$17.67
    - Balance After Resolution: $99,982.33

### Limit Order Test Verification
✅ **Test Passed**: Limit order at 55.0c was placed in orderbook instead of executing at 49.0c best bid
- Expected behavior: Order placed when limit price > best bid
- Actual behavior: Order correctly placed in database with status 'OPEN'
- Order visible in orderbook for other traders to match

<br>

## Test 2: DONE!
World Cup 2026

### Steps
All traders have $1000 USDC in their wallets at start.

1. Trader 1 Buy
- $20 worth Brazil Yes token at 51.0c
- Result: Trader 1 owns ~39.21 shares (Cost: $20.00)

2. Trader 1 Limit Sell
- Sell **20 shares** of Brazil Yes token at **60.0c**
- This order should be placed in the orderbook as the current price (approx 51c) is lower than 60c.
- Expected Proceeds: 20 shares * $0.60 = $12.00

3. Trader 2 Buy
- **Market Buy** **$120** worth of Brazil Yes token
- The seed liquidity is $100 (distributed 51c-55c). A $120 buy will clear all seed orders and consume Trader 1's limit sell at 60.0c.

### Resolution
Brazil Wins

### Result
- Trader 1:
    - **Initial Balance**: $1,000.00
    - **Step 1 (Buy)**: Buys $20 worth @ 51.0c -> 39.21 shares. Balance: $980.00.
    - **Step 2 (Limit Sell)**: Sells 20 shares @ 60.0c -> Receives $12.00. Balance: $992.00.
    - **Remaining Position**: 19.21 shares.
    - **Resolution (Win)**:
        - Payout: $19.21 (19.21 * $1.00)
        - Cost Basis (Remaining): ~$9.80
        - Profit: ~$9.41
        - Fee (4%): ~$0.45
        - Net Claim: **~$18.76**
    - **Final Balance**: **~$1,010.76**

- Trader 2:
    - **Initial Balance**: $1,000.00
    - **Step 3 (Buy)**: Tries to buy $120. **Max Liquidity is ~$92** ($80 Remaining Seed + $12 Trader 1).
    - **Filled**: ~$92.00. **Unspent**: ~$28.00.
    - **Balance After Trade**: ~$908.00 ($1,000 - $92).
    - **Shares Received**: ~168 shares (148 Seed + 20 Trader 1).
    - **Resolution (Win)**:
        - Payout: ~$168.00
        - Profit: ~$76.00 ($168 - $92)
        - Fee (4%): ~$3.60
        - Net Claim: **~$206.40** (Payout - Fee)
    - **Final Balance**: **~$1,086.40**

- Creator (Administrator/Liquidity Provider):
    - **Check P&L**:
        - Provided initial liquidity (Seed Orders).
        - Sold shares to Trader 2 during the market buy.
        - Verify collected fees from trading volume.
        - Verify change in USDC balance and share inventory.
    - **Check Fees (Upon Resolution)**:
        - **Platform Fee**: 2% of Winners' Profit.
        - **Creator Fee**: 2% of Winners' Profit.
        - **Total Fee Income**: 4% of Winners' Profit (since Creator = Platform).

### Limit Order Test Verification
✅ **Expected**: Trader 1's Limit Sell order (Maker) for 20 shares gets filled by Trader 2's Market Buy (Taker).

## Test 3: DONE!
Order Cancellation (Safety Check)
### Objective
Verify that funds are correctly locked when placing an order and fully refunded when cancelling it.

### Steps
1. **Initial State**: Record Trader 1's Cash Balance.
2. **Place Limit Buy**:
    - Trader 1 places a **Limit Buy** for **100 shares** of "Brazil Yes" at **10.0c** (Deep OTM, unlikely to fill).
    - **Cost**: $10.00.
3. **Verify Order**:
    - Check that the order appears in the "Open Orders" tab.
    - **Note**: Since orders are off-chain (EIP-712), the **Cash Balance** (on-chain) will **NOT** decrease immediately. Funds are only transferred upon execution.
    - *Optional*: If the UI implemented "Available Balance", it would decrease, but currently we display on-chain balance.
4. **Cancel Order**:
    - Click "Cancel" on the order in the "Open Orders" tab.
    - Confirm the transaction.
5. **Verify Refund**:
    - Check that the order disappears from the list.
    - Check that Trader 1's Cash Balance has **increased by $10.00** (returning to Initial State).

### Expected Result
✅ Funds are temporarily locked and then 100% returned. No gas/fees should be lost (except testnet gas).

## Test 4: DONE!
Market Sell (Hitting the Bid)
### Objective
Verify that a user can exit a position by selling shares into existing liquidity (Seed Bids).

### Steps
1. **Buy Position**:
    - Trader 1 buys **$50** of "Brazil Yes" (Market Buy).
    - Note the number of shares received (e.g., ~98 shares).
2. **Check Portfolio**:
    - Verify "My Positions" shows the new shares.
3. **Market Sell**:
    - Go to the "Sell" tab.
    - Select "Max" (or 100%).
    - Execute **Market Sell**.
4. **Verify Execution**:
    - The trade should match against the **Seed Bids** (Buy Orders).
    - Trader 1 should receive USDC back.
    - **Note**: The amount returned will be slightly less than $50 due to the **Spread** (Buying at Ask, Selling at Bid).

### Expected Result
✅ Trader 1 successfully converts Shares back to USDC.
✅ Cash Balance increases.
✅ Position size goes to 0 (or near 0).
✅ **Final Balance**: **~$998.02** (Initial $1000 -> Buy $50 -> Sell $48.02 -> Loss $1.98).


## Test 5: DONE!
Sweeping the Orderbook (Multi-Level Fill)
### Objective
Verify that a large Market Order correctly sweeps through multiple price levels until filled or liquidity is exhausted, and that all partial fills are correctly recorded.

### Steps
1. **Initial State**: Ensure Trader 1 has  USDC.
2. **Large Market Buy**:
    - Trader 1 attempts to buy **** of "Brazil Yes".
    - **Note**: Assuming Seed Liquidity is distributed across 51c, 52c, 53c, 54c, 55c.
3. **Execution**:
    - The order should fill the 51c bucket first.
    - Then 52c, and so on.
    - It should stop when  is spent OR liquidity runs out.
4. **Verify History**:
    - Check "History" tab.
    - It should show **multiple "Bought" entries** (e.g., Bought @ 51c, Bought @ 52c...).
    - **Total Cost** should sum to ~ (or max liquidity).
5. **Verify Portfolio**:
    - "My Positions" should show the weighted average price (e.g., ~53c).

### Expected Result
✅ Order fills across multiple levels.
✅ History shows individual trade executions for each price level.
✅ Average price in Portfolio reflects the blended cost.

<br>

## Test 6: DONE!
Redemption (Claiming Winnings)
### Objective
Verify that a user can claim their winnings after a market resolves, and that the payout calculation (including fees) is correct.

### Steps
1. **Setup**:
    - Trader 1 executes **Market Buy** of **$51.00** for "Brazil Yes" (approx 100 shares).
    - Trader 2 executes **Market Buy** of **$51.00** for "Brazil No" (approx 100 shares).
2. **Resolve Market**:
    - Administrator resolves the market to **"Brazil Yes"**.
3. **Check UI**:
    - Trader 1's "My Positions" should show "Claimable" or "Won".
    - Trader 2's position should show "Lost" or value $0.00.
4. **Claim**:
    - Trader 1 clicks "Claim" button.
    - Confirm transaction.
5. **Verify Balance**:
    - Trader 1's balance should increase by **$96.00** (100 shares * $1.00 * 0.96).
    - **Calculation**:
        - Gross Payout = 100 shares * $1.00 = $100.00.
        - Fee = $100.00 * 4% = $4.00.
        - Net Payout = $100.00 - $4.00 = $96.00.
6. **Verify History**:
    - History should show a "Claimed" entry with the correct amount.

### Expected Result
✅ Trader 1 successfully claims winnings.
✅ Balance increases by $96.00.
✅ Position is removed from "My Positions".

<br>

## Test 7
Hedging (Buying Both Sides)
### Objective
Verify the outcome when a trader buys equal amounts of both outcomes (Yes and No) and claims the winnings.

### Steps
1.- **Initial State**:
    - Trader 1 Balance: $1,000.00.
    - Market Liquidity: **$100.00** per outcome (seeded by Server).
    - Market: "Will Brazil win the 2026 FIFA World Cup?"
    - Current Price: Yes ($0.50), No ($0.50).
2. **Buy Yes**:
    - Trader 1 executes **Market Buy** of **$51.00** for "Brazil Yes".
    - Cost: $50.00.
    - Balance: $950.00.
3. **Buy No**:
    - Trader 1 executes **Market Buy** of **$51.00** for "Brazil No".
    - Cost: $50.00.
    - Balance: $900.00.
4. **Verify Portfolio (Mark-to-Market)**:
    - **Cash Balance**: $900.00.
    - **Current Value**: ~$96.00 (Assuming price settles to 0.48c-0.50c).
    - **Portfolio Value**: ~$996.00.
    - **Note**: The portfolio value reflects the current market price of the shares. It may show a small unrealized loss due to the spread paid during entry.
5. **Resolve Market**:
    - Administrator resolves the market to **"Brazil Yes"**.
6. **Claim**:
    - Trader 1 claims winnings for "Brazil Yes".
    - "Brazil No" shares expire worthless.
7. **Verify Balance**:
    - **Payout**: 100 Yes shares * $1.00 * 0.96 (4% fee) = $96.00.
    - **Final Balance**: $900.00 (Cash) + $96.00 (Payout) = **$996.00**.

### Expected Result
✅ Trader 1 ends with $996.00.
✅ The $4.00 loss corresponds exactly to the 4% fee on the winning payout ($100 * 4%).

<br>

## Test 8: DONE!
Market Trading (Buying Both Sides with Dollar Amount)
### Objective
Verify the outcome when a trader executes Market Buys for a specific dollar amount ($30) on both outcomes.

### Steps
1. **Initial State**: Trader 1 has $1,000.00 mUSDC.
2. **Buy Yes**:
    - Trader 1 Market Buys **$30.00** of "Brazil Yes".
    - **Execution**: Fills against the orderbook (e.g., avg price ~51c).
    - **Shares Received**: ~$58.82 shares ($30 / 0.51).
    - **Balance**: $970.00.
3. **Buy No**:
    - Trader 1 Market Buys **$30.00** of "Brazil No".
    - **Execution**: Fills against the orderbook (e.g., avg price ~51c).
    - **Shares Received**: ~$58.82 shares ($30 / 0.51).
    - **Balance**: $940.00.
4. **Verify Portfolio (Mark-to-Market)**:
    - **Cash Balance**: $940.00.
    - **Current Value**: ~$58.82 (Since Price(Yes) + Price(No) = $1.00).
    - **Portfolio Value**: ~$998.82.
    - **Unrealized Loss**: ~$1.18.
5. **Resolve Market**:
    - Administrator resolves the market to **"Brazil Yes"**.
6. **Claim**:
    - Trader 1 claims winnings for "Brazil Yes".
    - "Brazil No" shares expire worthless.
7. **Verify Balance**:
    - **Payout**: ~58.82 Yes shares * $1.00 * 0.96 (4% fee) = ~$56.47.
    - **Final Balance**: $940.00 (Cash) + $56.47 (Payout) = **~$996.47**.
    - **Net Loss**: ~$3.53.

### Expected Result
✅ Trader 1 successfully executes dollar-amount market buys.
✅ Final balance reflects losses from both spread/slippage and fees.

### Under the Hood: Understanding the Unrealized Loss
Why does the portfolio show a loss immediately after buying both sides?

1.  **The Formula**:
    - The system calculates Current Value as: `(Shares_Yes * Price_Yes) + (Shares_No * Price_No)`.
    - It does **not** use your entry price. It uses the **current market price**.

2.  **The Mechanism**:
    - **Yes, absolutely.** The backend explicitly enforces `Price(Yes) + Price(No) = 1.0` at all times before resolution.
    - Whenever a trade occurs on one side (e.g., Yes moves to 0.51), the system automatically updates the other side (No moves to 0.49).
    - This ensures that the market probability always sums to 100% ($1.00).

3.  **The Result**:
    - **Cost**: You paid ~$60.00 (avg $0.51 each) to enter.
    - **Value**: You hold $58.82 worth of positions.
    - **Loss**: The $1.18 difference is the **Spread** you paid to the market makers.
