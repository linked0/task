# Limit Order Placement Implementation - 2025-11-24

## Objective
Implement proper limit order functionality where users can place orders in the orderbook at their desired price, instead of always executing at the best available price.

## Problem
Previously, when a user set a limit price (e.g., sell at 51¢) but there was no matching order in the orderbook (best bid 49¢), the system would execute the trade at 49¢, ignoring the user's desired price.

## Solution Implemented

### 1. Backend: User Order Endpoints

**File:** `api/src/routes/orders.ts`

Added two new endpoints:

#### `POST /api/orders/user/create`
- Accepts a signed EIP-712 order from the user
- Validates the order and outcome
- Calculates price and size from makerAmount/takerAmount
- Saves the order to the database with status 'OPEN'
- Returns order confirmation

#### `DELETE /api/orders/user/:orderId`
- Allows users to cancel their unfilled or partially filled orders
- Verifies user ownership
- Updates order status to 'CANCELLED'

### 2. Frontend: Limit Order Detection

**File:** `web/src/hooks/useTrade.ts`

Updated the trade execution logic to detect when to place vs execute:

**For Buying:**
- If limit price >= best ask: Execute immediately (can cross spread)
- If limit price < best ask: Place limit order in orderbook

**For Selling:**
- If limit price <= best bid: Execute immediately (can cross spread)
- If limit price > best bid: Place limit order in orderbook

### 3. Limit Order Placement Function

Added `placeLimitOrder()` helper function that:
1. Calculates makerAmount and takerAmount based on price and size
2. Creates EIP-712 signed order structure
3. Signs the order with user's wallet (MetaMask)
4. Submits to backend `POST /api/orders/user/create`
5. Shows success message to user

## How It Works

### Example 1: Limit Sell Above Best Bid
**Scenario:** Best bid is 49¢, user wants to sell at 51¢

1. User sets limit price to 51¢  
2. System detects: 51¢ > 49¢ (limit price above best bid)  
3. System calls `placeLimitOrder()` instead of executing  
4. Order is signed and saved to database  
5. Order appears in orderbook at 51¢  
6. Order waits until someone buys at 51¢ or higher  

### Example 2: Limit Sell Below Best Bid (Crosses Spread)
**Scenario:** Best bid is 49¢, user wants to sell at 48¢

1. User sets limit price to 48¢  
2. System detects: 48¢ <= 49¢ (limit price at or below best bid)  
3. System executes immediately at 49¢ (better price for seller!)  
4. Trade completes instantly  

### Example 3: Limit Buy Below Best Ask
**Scenario:** Best ask is 51¢, user wants to buy at 49¢

1. User sets limit price to 49¢  
2. System detects: 49¢ < 51¢ (limit price below best ask)  
3. System calls `placeLimitOrder()` instead of executing  
4. Order is signed and saved to database  
5. Order appears in orderbook at 49¢  
6. Order waits until someone sells at 49¢ or lower  

## Files Changed

### Backend
- `api/src/routes/orders.ts` - Added user order creation and cancellation endpoints

### Frontend  
- `web/src/hooks/useTrade.ts` - Added limit order detection and placement logic

## Database Schema
Orders are saved with:
- `maker`: User's wallet address
- `side`: 'BUY' or 'SELL'
- `price`: Calculated from makerAmount/takerAmount
- `originalSize`: Total shares in the order
- `remainingSize`: Unfilled shares (starts equal to originalSize)
- `status`: 'OPEN', 'PARTIALLY_FILLED', 'FILLED', or 'CANCELLED'
- `signature`: EIP-712 signature from user

## User Experience

**Before (Broken):**
- User sets limit sell at 51¢
- Trade executes at 49¢ (best bid)
- ❌ User's price ignored

**After (Fixed):**
- User sets limit sell at 51¢  
- Order placed in orderbook: "Limit order placed at 51¢ for X shares"  
- ✅ Order waits for match at user's price  
- User sees their order in the orderbook  
- Can cancel order if desired  

## Next Steps (Future Enhancements)

1. **Display User Orders in Orderbook**
   - Highlight user's own orders with different color
   - Show "Cancel" button next to user orders
   
2. **My Orders Tab**
   - Add "Open Orders" section to My Positions page
   - Show all unfilled limit orders
   - One-click cancellation

3. **Order Notifications**
   - WebSocket notification when limit order is matched
   - Email/push notifications (optional)

4. **Advanced Order Types**
   - Stop-loss orders
   - Fill-or-kill orders
   - Iceberg orders

## Testing

To test the new functionality:

1. **Test Limit Sell Above Best Bid**
   ```
   - Best bid: 49¢
   - Set limit sell at 51¢
   - Expected: Order placed in orderbook
   - Check: yarn show:world-cup should show order
   ```

2. **Test Limit Sell Below Best Bid**
   ```
   - Best bid: 49¢
   - Set limit sell at 48¢
   - Expected: Executes immediately at 49¢
   ```

3. **Test Order Cancellation**
   ```
   - Place limit order
   - Call DELETE /api/orders/user/:orderId
   - Expected: Order status = 'CANCELLED'
   ```

## Notes

- Orders expire after 7 days (can be configured)
- Users need to approve ConditionalTokens for both buy and sell orders
- EIP-712 signing ensures orders are secure and can't be forged
- Orders are stored in database and can be queried by outcome/market
- The orderbook now includes both server liquidity and user limit orders

##  Commit Message

```
feat: implement full limit order placement in orderbook

- Add POST /api/orders/user/create endpoint for user limit orders
- Add DELETE /api/orders/user/:orderId for order cancellation  
- Update trade logic to detect when to place vs execute orders
- Limit orders now wait in orderbook instead of executing at market price

When a user's limit price doesn't cross the spread, the order is now
placed in the orderbook and waits for a matching counterparty, rather
than immediately executing at the best available price.

Example:
- Sell at 51¢ when best bid is 49¢ → order placed at 51¢ (waits)
- Sell at 48¢ when best bid is 49¢ → executes at 49¢ (better price!)

Files changed:
- api/src/routes/orders.ts
- web/src/hooks/useTrade.ts
```
