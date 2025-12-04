# Trading.sol USDC Refund Bug Fix

**Date**: 2024-12-02
**Status**: Fix implemented, awaiting deployment
**Severity**: Critical (affects all user balances)

## Problem Description

When users executed BUY trades, their portfolio balance would incorrectly jump from ~$1,000 to ~$56,000. The bug caused the exchange to credit ALL deposited USDC to the trading user instead of properly calculating any refund.

### Observed Behavior
- TRADER_4 started with $1,000 deposited
- After executing a $10 BUY trade, balance jumped to $55,990
- Pattern: Each trader's balance difference was ~$1,000 (the amount each deposited)

## Root Cause Analysis

**Location**: `packages/contracts/contracts/exchange/mixins/Trading.sol` lines 188-189

**Original Code**:
```solidity
// Refund any leftover tokens pulled from the taker to the taker order
uint256 refund = _getBalance(makerAssetId);
if (refund > 0) _transfer(address(this), takerOrder.maker, makerAssetId, refund);
```

**The Bug**:
For COMPLEMENTARY matches (BUY vs SELL orders), the code calls `_getBalance(0)` for USDC refunds. This returns `IERC20(collateral).balanceOf(address(this))` - the **entire ERC20 USDC balance** of the exchange (~$56k), not just leftover from the trade.

**Why It Happened**:
- COMPLEMENTARY matches only modify internal `balances[]` mapping
- No actual ERC20 transfers occur during the trade
- The ERC20 balance never changes, so `_getBalance(0)` returns ALL deposited USDC
- This entire amount was incorrectly "refunded" to the taker

**Why CTF Tokens Work Correctly**:
- For CTF tokens (ERC1155), `_getBalance(tokenId)` returns the exchange's actual token balance for that specific tokenId
- CTF tokens are only temporarily held during matching, so any balance IS the actual refund

## The Fix

### 1. Modified `_fillMakerOrders` to return collateral consumed
```solidity
function _fillMakerOrders(Order memory takerOrder, Order[] memory makerOrders, uint256[] memory makerFillAmounts)
    internal
    returns (uint256 totalCollateralConsumed)  // NEW: returns consumed amount
{
    // ... tracking logic added
}
```

### 2. Modified `_fillMakerOrder` to calculate consumption
```solidity
// Track collateral consumed for COMPLEMENTARY matches only.
if (matchType == MatchType.COMPLEMENTARY && takerOrder.side == Side.BUY) {
    collateralConsumed = taking - fee;  // USDC paid to maker seller
} else {
    collateralConsumed = 0;  // MINT/MERGE use actual ERC20 transfers
}
```

### 3. Updated refund logic in `_matchOrders`
```solidity
uint256 refund;
if (makerAssetId == 0 && totalCollateralConsumed > 0) {
    // COMPLEMENTARY BUY: calculate refund manually
    refund = making > totalCollateralConsumed ? making - totalCollateralConsumed : 0;
} else {
    // MINT/MERGE or CTF tokens: use actual balance (works correctly)
    refund = _getBalance(makerAssetId);
}
```

## Match Type Behavior Summary

| Match Type | Taker | Maker | ERC20 Transfers | Refund Method |
|------------|-------|-------|-----------------|---------------|
| COMPLEMENTARY | BUY | SELL | Internal only | Calculate manually |
| COMPLEMENTARY | SELL | BUY | Internal only | N/A (taker receives) |
| MINT | BUY | BUY | Via ConditionalTokens | Use `_getBalance` |
| MERGE | SELL | SELL | Via ConditionalTokens | Use `_getBalance` |

## Files Changed

- `packages/contracts/contracts/exchange/mixins/Trading.sol`
  - `_matchOrders`: Updated refund calculation
  - `_fillMakerOrders`: Now returns `totalCollateralConsumed`
  - `_fillMakerOrder`: Now returns `collateralConsumed`

## Testing

- Compilation: Successful
- Unit Tests: 199 passing (3 pre-existing failures unrelated to this fix)
- Pre-existing test failures are in `MatchOrders.test.ts` due to token ID setup issues

## Deployment Steps

```bash
cd /Users/jay/work/nostra-contracts
yarn compile        # Already done
yarn deploy:local   # Redeploy to local testnet
```

## Verification

After deployment, test with a fresh trader:
1. Deposit $1,000 USDC
2. Execute a $10 BUY trade
3. Verify portfolio shows ~$990 (not $56k)

## Lessons Learned

1. When using internal balance systems, be careful with `_getBalance` for collateral
2. COMPLEMENTARY matches behave differently from MINT/MERGE (no actual ERC20 movement)
3. The Polymarket CTF Exchange architecture assumes certain patterns that may not hold when modified
