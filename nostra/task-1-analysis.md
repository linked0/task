# Task-1 Analysis: Discrepancy Between Simulation and Actual Results

**Date**: November 21, 2025, 15:07 KST  
**Issue**: Different P&L results between `brazil_win_simulation.ipynb` and `yarn result:world-cup`

---

## 🔍 Root Cause Analysis

### The Problem

**Simulation Results** (Expected after resolution):
```
Trader 1: +$8.82  (+0.88% ROI) ✅ Winner
Trader 2: +$8.46  (+0.85% ROI) ✅ Winner
Trader 3: +$8.11  (+0.81% ROI) ✅ Winner
Trader 4: -$20.00 (-2.00% ROI) ❌ Loser
```

**Actual Results** (Current state):
```
Trader 1: +$0.19  (+0.02% ROI) 
Trader 2: -$0.00  (-0.00% ROI) 
Trader 3: -$0.19  (-0.02% ROI) 
Trader 4: -$0.00  (-0.00% ROI) 
```

### Root Cause: **Market NOT Resolved**

The script output shows:
```
Status:        ACTIVE ⏳
Winner:        🏆 Brazil
```

This is **contradictory and confusing**:
- Status is `ACTIVE` (not `RESOLVED`)
- But it shows "Winner: 🏆 Brazil"

---

## 🐛 Issues Identified

### 1. **Market Status Issue**
The market group has `status = 'ACTIVE'` in the database, but the script is showing a winner. This suggests:
- The script is incorrectly detecting a "winner" when the market hasn't been resolved
- OR the market was partially resolved (one market resolved, but group status not updated)

### 2. **Token Value Calculation**
The script is calculating token values at **current market prices** instead of **resolved payout values**:

**Current logic** (in `participant-status.ts` lines 180-193):
```typescript
if (isResolved && winnerCountry) {
    if (countryName === winnerCountry) {
        totalTokenValue += yesTokens * 1.0;  // $1.00 per share
    } else {
        totalTokenValue += noTokens * 1.0;   // $1.00 per share
    }
} else {
    // Not resolved: use current prices
    const yesPrice = parseFloat(yesOutcome.currentPrice.toString());
    const noPrice = parseFloat(noOutcome.currentPrice.toString());
    totalTokenValue += (yesTokens * yesPrice) + (noTokens * noPrice);
}
```

Since `isResolved = false`, it's using current prices (52¢ for Brazil), not $1.00!

### 3. **Winner Detection Logic Bug**
The script shows "Winner: Brazil" even though status is `ACTIVE`. Let me check the logic:

```typescript
const resolvedMarket = marketGroup.markets.find(m => m.resolvedOutcomeId !== null);
const winningOutcome = resolvedMarket?.outcomes.find(o => o.id === resolvedMarket?.resolvedOutcomeId);
const winnerCountry = winningOutcome && resolvedMarket ? resolvedMarket.title.match(/Will (.+) win/)?.[1] : null;
```

This finds **any market** with a `resolvedOutcomeId`, even if the overall group status is still `ACTIVE`. This can happen if:
- One market (Brazil) was resolved individually
- But the market group status wasn't updated to `RESOLVED`

---

## 📊 Actual Current State

Based on the output, here's what actually happened:

**Trader Holdings**:
| Trader  | Brazil YES | France YES | Cash    | Token Value @ 52¢ | Total  | P&L   |
|---------|-----------|-----------|---------|-------------------|--------|-------|
| Trader 1| 19.61     | 0         | $990.00 | $10.19           | $1000.19| +$0.19|
| Trader 2| 19.23     | 0         | $990.00 | $10.00           | $1000.00| $0.00 |
| Trader 3| 18.87     | 0         | $990.00 | $9.81            | $999.81 | -$0.19|
| Trader 4| 0         | 39.22     | $980.00 | $20.00           | $1000.00| $0.00 |

**Why these P&L values?**
- Brazil YES is trading at ~52¢ (not $1.00 because not resolved)
- Trader 1 bought at 51¢ → 19.61 shares @ 52¢ = $10.19 value → +$0.19 profit
- Trader 2 bought at 52¢ → 19.23 shares @ 52¢ = $10.00 value → $0.00 (break even)
- Trader 3 bought at 53¢ → 18.87 shares @ 52¢ = $9.81 value → -$0.19 loss
- Trader 4 bought France at 51¢→ 39.22 shares @ 51¢ = $20.00 value → $0.00

These are **mark-to-market** values, not resolution payouts!

---

## ✅ What Needs to Happen

### Step 1: Resolve the Market
Execute the resolve function to:
1. Set market/group status to `RESOLVED`
2. Set `resolvedOutcomeId` to the winning outcome (Brazil YES)
3. Distribute payouts to winners (minus 4% fee)

### Step 2: Expected Results After Resolution

**After calling the resolve API:**
```
POST /api/admin/resolve
{
  "marketId": "<brazil-market-id>",
  "winningOutcomeId": "<brazil-yes-outcome-id>"
}
```

**Expected final balances:**
- **Trader 1**: $990 (cash) + $18.82 (payout) = **$1,008.82** → **+$8.82 profit**
- **Trader 2**: $990 (cash) + $18.46 (payout) = **$1,008.46** → **+$8.46 profit**
- **Trader 3**: $990 (cash) + $18.11 (payout) = **$1,008.11** → **+$8.11 profit**
- **Trader 4**: $980 (cash) + $0 (France lost) = **$980.00** → **-$20.00 loss**

**Fee distribution:**
- Total gross payout: $57.71 (19.61 + 19.23 + 18.87 shares)
- Fee (4%): $2.31
- Platform (2%): $1.15
- Creator (2%): $1.16
- Net payout: $55.39

---

## 🔧 Recommendations

### Option 1: Fix the Script Logic (Quick Fix)
Update `participant-status.ts` to check `marketGroup.status` instead of individual market resolution:

```typescript
const isResolved = marketGroup.status === 'RESOLVED';  // ← Use this
// Don't rely on resolvedMarket existing
```

### Option 2: Resolve the Market (Proper Solution)
Call the resolve API endpoint to properly resolve the market and distribute payouts.

### Option 3: Update Script Display
Change the script to be clearer about market state:
```typescript
if (marketGroup.status !== 'RESOLVED') {
    console.log(`Status:        ${colors.yellow}ACTIVE ⏳ (Not yet resolved)${colors.reset}`);
    console.log(`${colors.yellow}⚠️  Values shown are mark-to-market, not final payouts${colors.reset}`);
} else {
    console.log(`Status:        ${colors.green}RESOLVED ✅${colors.reset}`);
    console.log(`Winner:        ${colors.bright}${colors.green}🏆 ${winnerCountry}${colors.reset}`);
}
```

---

## 📝 Summary

**Is the simulation correct?** ✅ **YES**  
The simulation correctly calculates what WILL happen after resolution.

**Is the result script correct?** ⚠️ **PARTIALLY**  
It correctly shows current state but:
- Confusingly shows "Winner" when not resolved
- Doesn't clarify it's showing mark-to-market values

**Is the current code correct?** ⚠️ **PARTIALLY**  
- Trading logic: ✅ Works correctly
- Resolution logic: ❓ Exists but not yet executed
- Display logic: ⚠️ Confusing/misleading

**What's the fix?**  
Either:
1. Resolve the market properly (recommended)
2. Fix the script to clarify it's showing pre-resolution values

---

## Next Steps

1. **Clarify Intent**: Does the user want to:
   - Actually resolve the market now?
   - Fix the script to be clearer about pre-resolution state?
   - Both?

2. **If resolving**: Need to call the resolve API with proper parameters

3. **If fixing script**: Update display logic to avoid confusion
