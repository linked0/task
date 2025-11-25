# My Positions Navigation Fix - 2025-11-24

## Objective
Fix navigation from "My Positions" page to market detail pages. Clicking on market titles in the Positions, Claims, and Open Orders tabs should navigate to the correct market group page.

## Problem
Market titles in the "My Positions" tabs were linking to individual market IDs instead of market group IDs, resulting in "Market not found" errors for grouped-binary markets (like FIFA World Cup).

## Solution

### Backend Changes

1. **`api/src/routes/redemption.ts`**
   - Added `marketGroupId` to the positions response
   - Positions now include both `marketId` and `marketGroupId` for proper navigation

2. **`api/src/routes/orders.ts`**
   - Updated user orders query to include market group relationship
   - Added `marketGroupId` to open orders response
   - Fixed `filledSize` calculation (was already working correctly)

### Frontend Changes

1. **`web/src/app/my-positions/page.tsx`**
   - Updated `MarketPosition` interface to include `marketGroupId?`
   - Updated `OpenOrder` interface to include `marketGroupId?`

2. **`web/src/components/my-positions/PositionsList.tsx`**
   - Updated interface to include `marketGroupId?`
   - Changed Link href to: `/market/${position.marketGroupId || position.marketId}`
   - Falls back to `marketId` for backward compatibility

3. **`web/src/components/my-positions/ClaimsList.tsx`**
   - Updated interface to include `marketGroupId?`
   - Changed Link href to: `/market/${position.marketGroupId || position.marketId}`

4. **`web/src/components/my-positions/OpenOrdersList.tsx`**
   - Updated interface to include `marketGroupId?`
   - Changed Link href to: `/market/${order.marketGroupId || order.marketId}`

## Testing
- Navigate to "My Positions" page
- Click on market titles in all three tabs (Positions, Claims, Open Orders)
- Should navigate to the correct market group detail page
- For FIFA World Cup markets, should show the full market group with all choices

## Known Issues / Future Work

### Limit Order Pricing (Discovered During Testing)
**Problem:** When placing a limit order at a price that doesn't exist in the orderbook (e.g., sell at 51¢ when best bid is 49¢), the system currently executes at the best available price instead of placing the order in the orderbook.

**Current Behavior:**
- Limit orders try to match exact price in orderbook
- If no match found, fall back to best bid/ask
- This means limit price is not respected if no matching order exists

**Correct Behavior (Not Yet Implemented):**
- If no matching order at limit price, **place order in orderbook** at user's price
- Wait for another user to match the order
- Only execute immediately if limit price crosses spread (better than best bid/ask)

**Implementation Plan (Option 1 - Full Limit Order Support):**
See `/Users/jay/work/task/nostra/tasks/limit-order-placement.md` for detailed implementation steps.

## Files Changed
- `api/src/routes/redemption.ts`
- `api/src/routes/orders.ts`
- `web/src/app/my-positions/page.tsx`
- `web/src/components/my-positions/PositionsList.tsx`
- `web/src/components/my-positions/ClaimsList.tsx`
- `web/src/components/my-positions/OpenOrdersList.tsx`

## Commit Message
See below for suggested commit message.
