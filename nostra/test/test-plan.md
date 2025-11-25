# Prerequisites
The balances of all traders are $1000 USDC.
The creator has $100000 USDC who is also a administrator of this platform.

You calculate the PnL of each trader based on the final resolution. And the creator would get the 2% of the Winners' profit. And the platform would also get the 2% of the Winner's profit. Currently, the creator and the platform are the same person, which mean the creator would get the 4% of the Winners' profit.

# Limit Test

## Test 1
World Cup 2026

### Steps
All traders have $1000 USDC in their wallets at start.

1. Trader 1 Buy
- $10 worth Brazil Yes token at 51.0c

2. Trader 2 Buy
- $10 worth France Yes token at 51.0c

3. Trader 3 Buy
- $20 worth France No token at 51.0c

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
    - Balance After Claim: $1009.23 ($990.00 + $19.23)

- Trader 2: Filled 19.61 shares @ 51.0c (France YES)
    - Initial Balance: $1000.00
    - Balance After Buy: $990.00
    - Gross PnL (If Brazil Wins): -$10.00
    - Fee: $0.00
    - Net Profit: -$10.00
    - Balance After Claim: $990.00

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
    - Balance After Claim: $1018.44 ($980.00 + $38.44)
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

# Market Test 