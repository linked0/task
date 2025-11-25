# Task: Implement Limit Order Placement in Orderbook

## Background
Currently, when a user places a limit order at a price that doesn't exist in the orderbook, the system executes at the best available price. This is incorrect behavior - limit orders should be placed in the orderbook and wait for a matching counterparty.

## Current Behavior
**Example: Selling at 51¢ when best bid is 49¢**
- ❌ System executes trade at 49¢ (best bid)
- ❌ User's desired price of 51¢ is ignored

## Correct Behavior
**Example: Selling at 51¢ when best bid is 49¢**
- ✅ System creates a new sell order at 51¢ in the database
- ✅ Order appears in the orderbook for others to see
- ✅ Order waits until someone matches it (buys at 51¢ or higher)
- ✅ Only executes immediately if limit price crosses spread (e.g., sell limit at 48¢ when best bid is 49¢)

## Implementation Steps

### 1. Backend: Create User Order Endpoint
**File:** `api/src/routes/orders.ts`

Add new endpoint: `POST /api/orders/user/create`

```typescript
/**
 * POST /api/orders/user/create
 * Create a new user limit order and save to database
 * 
 * Request body:
 * - outcomeId: string
 * - tokenId: string
 * - side: 'BUY' | 'SELL'
 * - price: string (decimal, e.g., "0.51")
 * - size: string (shares, e.g., "10.5")
 * - userAddress: string
 * - signature: string (EIP-712 signed order)
 */
```

**Steps:**
1. Validate request parameters
2. Create EIP-712 signed order structure
3. Verify signature matches user address
4. Save order to database with status 'OPEN'
5. Set maker = userAddress, taker = ZeroAddress (any operator can fill)
6. Calculate and store makerAmount/takerAmount based on price and size
7. Return created order

### 2. Frontend: Detect When to Place vs Execute

**File:** `web/src/hooks/useTrade.ts`

Update `executeTrade` function to:

1. **Check if order can be matched immediately:**
   ```typescript
   const canExecuteImmediately = (side: string, targetPrice: number, orders: MakerOrder[]) => {
     if (side === 'buy') {
       // Can buy if our limit price >= best ask
       const bestAsk = parseFloat(orders[0]?.price || '999');
       return targetPrice >= bestAsk;
     } else {
       // Can sell if our limit price <= best bid
       const bestBid = parseFloat(orders[0]?.price || '0');
       return targetPrice <= bestBid;
     }
   };
   ```

2. **Branch execution:**
   ```typescript
   if (canExecuteImmediately(params.side, targetPrice, relevantOrders)) {
     // Execute immediately (current flow)
     const makerOrder = relevantOrders[0];
     // ... existing execution logic
   } else {
     // Place limit order in orderbook
     await placeUserLimitOrder(params);
   }
   ```

3. **Implement `placeUserLimitOrder` function:**
   - Sign EIP-712 order with user's wallet
   - Call `POST /api/orders/user/create` endpoint
   - Show success message: "Order placed at {price}. It will execute when matched."
   - Refresh orderbook to show new order

### 3. Update Order Repository

**File:** `api/src/db/repositories/OrderRepository.ts`

Ensure the following methods exist:
- `create()` - Create new order (should already exist)
- `findByUser()` - Get all orders for a user address
- `updateAfterFill()` - Update order after partial/full fill (should already exist)
- `cancelOrder()` - Allow users to cancel unfilled orders (NEW)

### 4. Display User Orders in Orderbook

**File:** `web/src/app/market/[id]/page.tsx`

Update orderbook display to:
- Highlight user's own orders (different color/background)
- Show "Cancel" button next to user's unfilled orders
- Allow clicking "Cancel" to remove order from orderbook

### 5. Backend: Order Cancellation

**File:** `api/src/routes/orders.ts`

Add endpoint: `DELETE /api/orders/:orderId`
- Verify user owns the order
- Update order status to 'CANCELLED'
- Remove from active orderbook

### 6. Testing Scenarios

**Test 1: Limit Sell Above Best Bid**
1. Best bid is 49¢
2. User places limit sell at 51¢ for 10 shares
3. ✅ Order appears in orderbook at 51¢
4. ✅ Order does NOT execute immediately
5. Another user buys at 51¢
6. ✅ Order executes

**Test 2: Limit Sell Below Best Bid (Crosses Spread)**
1. Best bid is 49¢
2. User places limit sell at 48¢ for 10 shares
3. ✅ Order executes IMMEDIATELY at 49¢ (better price for seller!)

**Test 3: Limit Buy Below Best Ask**
1. Best ask is 51¢
2. User places limit buy at 49¢ for $10
3. ✅ Order appears in orderbook at 49¢
4. ✅ Order does NOT execute immediately
5. Another user sells at 49¢
6. ✅ Order executes

**Test 4: Order Cancellation**
1. User places limit order
2. Order appears in "My Orders" and orderbook
3. User clicks "Cancel"
4. ✅ Order removed from orderbook
5. ✅ Order status = 'CANCELLED' in database

## Database Schema (Verify)

`Order` table should have:
- `id` (UUID)
- `orderHash` (string, unique)
- `salt` (string)
- `maker` (address) - User's address for user orders
- `signer` (address)
- `taker` (address) - ZeroAddress for public orders
- `outcomeId` (FK)
- `side` (enum: BUY, SELL)
- `price` (decimal)
- `originalSize` (decimal)
- `remainingSize` (decimal)
- `status` (enum: OPEN, PARTIALLY_FILLED, FILLED, CANCELLED)
- `makerAmount` (bigint)
- `takerAmount` (bigint)
- `signature` (string)
- `createdAt` (timestamp)
- `updatedAt` (timestamp)

## Priority
**High** - This is core orderbook functionality and current behavior is misleading to users.

## Estimated Effort
- Backend: 4-6 hours
- Frontend: 3-4 hours
- Testing: 2 hours
- **Total: ~10 hours**

## Dependencies
- EIP-712 signing (already implemented in `useTrade.ts`)
- Order database schema (already exists)
- CTFExchange contract (already deployed)

## Success Criteria
- Users can place limit orders at any price
- Orders appear in orderbook immediately
- Orders only execute when price is matched
- Users can cancel unfilled orders
- Order matching logic respects limit prices
