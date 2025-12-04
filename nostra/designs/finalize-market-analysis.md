# Finalize Market Code Analysis

**Date**: November 21, 2025, 15:10 KST  
**File**: `/Users/jay/work/nostra-server/api/src/routes/admin.ts`  
**Endpoint**: `POST /api/admin/resolve`

---

## 🔍 Code Review: Current Implementation

### **Frontend** (`/web/src/app/market/[id]/page.tsx`)

**Location**: Lines 437-485

```typescript
const handleResolveMarket = async () => {
  if (!winningOutcomeId) {
    alert('Please select a winning outcome');
    return;
  }

  if (!confirm('Are you sure you want to resolve this market? This action cannot be undone.')) {
    return;
  }

  try {
    let outcomeIdToResolve = winningOutcomeId;
    let marketIdToResolve = marketId;

    // For grouped binary: find YES outcome of selected choice
    const selectedChoice = marketGroup?.choices.find(c => c.id === winningOutcomeId);

    if (selectedChoice && selectedChoice.yesOutcomeId) {
      outcomeIdToResolve = selectedChoice.yesOutcomeId;
      marketIdToResolve = selectedChoice.id;
    }

    const response = await fetch(`${API_URL}/api/admin/resolve`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        marketId: marketIdToResolve,
        winningOutcomeId: outcomeIdToResolve
      })
    });

    const data = await response.json();
    if (data.success) {
      alert('Market resolved successfully!');
      window.location.reload();
    } else {
      alert(`Failed to resolve: ${data.error}`);
    }
  } catch (error) {
    console.error('Error resolving market:', error);
    alert('Failed to resolve market');
  }
};
```

✅ **Frontend looks good!**

---

### **Backend** (`/api/src/routes/admin.ts`)

**Location**: Lines 37-145

```typescript
adminRoutes.post('/resolve', async (req, res) => {
  try {
    const { marketId, winningOutcomeId } = req.body;

    if (!marketId || !winningOutcomeId) {
      return res.status(400).json({
        success: false,
        error: 'Missing marketId or winningOutcomeId',
      });
    }

    console.log(`⚖️ Resolving market ${marketId} with winner ${winningOutcomeId}`);

    // 1. Update Market Status
    const market = await prisma.market.update({
      where: { id: marketId },
      data: {
        status: 'RESOLVED',
        resolvedOutcomeId: winningOutcomeId,
        resolvedAt: new Date(),
      },
    });

    // 2. Update Winning Outcome
    await prisma.outcome.update({
      where: { id: winningOutcomeId },
      data: { isWinningOutcome: true },
    });

    // 3. Distribute Payouts (Off-chain ledger update)
    // Find all positions for the winning outcome
    const winningPositions = await prisma.position.findMany({
      where: {
        marketId,
        outcomeId: winningOutcomeId,
        shares: { gt: 0 },
      },
      include: { user: true },
    });

    const FEE_RATE = 0.04; // 4% fee

    for (const position of winningPositions) {
      const shares = Number(position.shares);
      const grossPayout = shares * 1.0;
      const fee = grossPayout * FEE_RATE;
      const netPayout = grossPayout - fee;

      // Update User Balance
      await prisma.user.update({
        where: { id: position.userId },
        data: {
          cashBalance: { increment: netPayout },
          portfolioValue: { decrement: position.currentValue },
        },
      });

      // Update Position (Mark as redeemed/closed)
      await prisma.position.update({
        where: { id: position.id },
        data: {
          shares: 0,
          currentValue: 0,
          realizedProfit: { increment: netPayout - Number(position.totalInvested) },
          totalInvested: 0,
        },
      });
    }

    // 4. Handle Losing Positions
    const losingPositions = await prisma.position.findMany({
      where: {
        marketId,
        outcomeId: { not: winningOutcomeId },
        shares: { gt: 0 },
      },
    });

    for (const position of losingPositions) {
      await prisma.position.update({
        where: { id: position.id },
        data: {
          currentValue: 0,
          realizedProfit: { decrement: Number(position.totalInvested) },
          shares: 0,
          totalInvested: 0,
        },
      });
    }

    res.json({
      success: true,
      message: `Market resolved. Distributed payouts to ${winningPositions.length} winners.`,
    });

  } catch (error: any) {
    console.error('Error resolving market:', error);
    res.status(500).json({
      success: false,
      error: error.message || 'Failed to resolve market',
    });
  }
});
```

---

## ⚠️ CRITICAL ISSUES FOUND

### **Issue #1: Wrong Data Model ❌**

**Problem**: The code looks for `positions` in a `Position` table with a `user` relation.

**Reality**: Based on the `yarn result:world-cup` output and trading functionality, traders hold **tokens on-chain** (ConditionalTokens contract), NOT in a database `Position` table!

**Evidence**:
```
Token Holdings:
   Brazil 🏆:
      YES: 19.61 tokens 
```

These are **blockchain tokens**, not database positions!

### **Issue #2: Payout Mechanism Wrong ❌**

**Current Implementation**: Updates `cashBalance` in database
```typescript
await prisma.user.update({
  where: { id: position.userId },
  data: {
    cashBalance: { increment: netPayout },
  },
});
```

**Reality**: Need to transfer **on-chain USDC** (MockUSDC contract) to winner wallets!

The traders have:
- Private keys (TRADER_1_PRIVATE_KEY, etc.)
- Wallet addresses (0x6eDF...)
- On-chain USDC balances
- On-chain conditional tokens

**There is NO database tracking of user balances!**

### **Issue #3: Market Group Status Not Updated ❌**

The code only updates individual `market.status`, but doesn't update `marketGroup.status`. This is why the script shows:
```
Status:        ACTIVE ⏳
Winner:        🏆 Brazil
```

One market is resolved, but the group is still ACTIVE!

### **Issue #4: No On-Chain Redemption ❌**

**Missing**: Call to `CTFExchange.redeemPositions()` or `ConditionalTokens.redeemPositions()` to convert winning tokens to USDC.

In prediction markets, winners must:
1. Burn their winning conditional tokens
2. Receive USDC in return (1 token → $1.00 USDC, minus fees)

Currently, the code **doesn't interact with the blockchain at all**!

### **Issue #5: Fee Distribution Missing ❌**

Fees calculated but **never distributed**:
```typescript
const fee = grossPayout * FEE_RATE;
const netPayout = grossPayout - fee;
```

Where does the `fee` go? Should be:
- Platform (Nostra): 2%
- Market Creator (Jay): 2%

But there's **no code** sending fees to these addresses!

---

## ✅ What SHOULD Happen

### **Correct Resolution Flow**:

#### **Step 1: Market Resolution (Database)**
```typescript
// Update individual market
await prisma.market.update({
  where: { id: marketId },
  data: {
    status: 'RESOLVED',
    resolvedOutcomeId: winningOutcomeId,
    resolvedAt: new Date(),
  },
});

// Update market group if this is the winning market in a group
const market = await prisma.market.findUnique({
  where: { id: marketId },
  include: { group: true }
});

if (market.group) {
  await prisma.marketGroup.update({
    where: { id: market.groupId },
    data: {
      status: 'RESOLVED',
      resolvedAt: new Date(),
    },
  });
}
```

#### **Step 2: Get All Token Holders (Blockchain)**
```typescript
const config = getBlockchainConfig();
const provider = new ethers.JsonRpcProvider(config.rpcUrl);
const operator = new ethers.Wallet(config.serverWalletPrivateKey, provider);

// Load ConditionalTokens contract
const conditionalTokens = new ethers.Contract(
  getContractAddress(network, 'ConditionalTokens'),
  ConditionalTokensABI,
  operator
);

// Get winning outcome details
const outcome = await prisma.outcome.findUnique({
  where: { id: winningOutcomeId }
});

// Query all trades to find who holds winning tokens
const trades = await prisma.trade.findMany({
  where: {
    outcomeId: winningOutcomeId,
  },
  include: { user: true },
});

// Or scan blockchain events for token transfers
```

#### **Step 3: Redemption & Payout (Blockchain)**

This depends on your smart contract design. Two options:

**Option A: Automatic Server Redemption**
```typescript
for (const holder of tokenHolders) {
  const balance = await conditionalTokens.balanceOf(
    holder.address,
    outcome.tokenId
  );

  if (balance > 0) {
    // Server redeems on behalf of user (requires approval)
    const tx = await ctfExchange.redeemPosition(
      market.conditionId,
      [1, 0], // [YES, NO] - YES won
      holder.address,
      balance
    );
    await tx.wait();

    console.log(`Redeemed ${ethers.formatUnits(balance, 6)} shares for ${holder.address}`);
  }
}
```

**Option B: Users Redeem Themselves**
```typescript
// Just update status, users call redemption themselves
// Simpler, but requires user action
console.log('Market resolved. Users can now redeem their winning tokens.');
```

#### **Step 4: Fee Distribution (Blockchain)**
```typescript
// Calculate total fees collected
const totalFees = winnersShares * 1.0 * FEE_RATE;
const platformFee = totalFees * 0.5;
const creatorFee = totalFees * 0.5;

// Transfer USDC to platform
const mockUSDC = new ethers.Contract(
  getContractAddress(network, 'MockUSDC'),
  MockUSDCABI,
  operator
);

await mockUSDC.transfer(PLATFORM_ADDRESS, ethers.parseUnits(platformFee.toString(), 6));
await mockUSDC.transfer(CREATOR_ADDRESS, ethers.parseUnits(creatorFee.toString(), 6));
```

---

## 🚨 Why Current Code Won't Work

1. **No Position table exists** with user balances
2. **No cashBalance field** in users
3. **Traders hold tokens on-chain**, not in database
4. **Resolution requires blockchain transactions**, not just database updates
5. **Fees never get distributed** to platform/creator

---

## 📝 Recommended Fix

### **Option 1: Server-Managed Redemption**

**Pros**: 
- Automatic payouts
- Users get USDC immediately
- Can deduct fees automatically

**Cons**:
- Requires server to have token approval
- Gas costs
- Complex error handling

**Implementation**: Full blockchain integration as shown above

### **Option 2: User-Managed Redemption (Simpler)**

**Pros**:
- No server blockchain transactions
- Users control their tokens
- No approval needed

**Cons**:
- Users must manually redeem
- Requires frontend redemption UI
- Can't force fee collection

**Implementation**:
```typescript
// Just mark as resolved
await prisma.market.update({
  where: { id: marketId },
  data: {
    status: 'RESOLVED',
    resolvedOutcomeId: winningOutcomeId,
    resolvedAt: new Date(),
  },
});

// Emit event or notify users
console.log('Market resolved! Winners can redeem tokens for USDC.');
```

### **Option 3: Hybrid (Recommended)**

1. Server updates database status
2. Server calls smart contract's `finalizeMarket()` function
3. Smart contract handles redemption logic
4. Users claim via frontend when ready

---

## 🎯 Next Steps

1. **Clarify Requirements**:
   - How should redemption work?
   - Server-managed or user-managed?
   - Are fees mandatory or optional?

2. **Review Smart Contracts**:
   - Does CTFExchange have redemption functions?
   - How are fees handled on-chain?

3. **Fix Implementation**:
   - Remove database Position logic
   - Add blockchain interaction
   - Handle fees properly
   - Update market group status

4. **Test Resolution**:
   - Try resolving World Cup market
   - Verify token redemption
   - Check USDC distribution
   - Confirm fees distributed

---

## 📊 Expected Result After Proper Resolution

**Current** (market NOT resolved):
```
Trader 1: $990 USDC + 19.61 tokens @ 52¢ = $1000.19
```

**After Resolution** (tokens redeemed):
```
Trader 1: $990 + $18.82 payout = $1008.82 USDC
          (19.61 tokens × $1.00 × 96% after fees)
```

**Fees Collected**:
```
Platform: $1.15
Creator:  $1.16
Total:    $2.31
```

---

## 💡 Summary

The current "Finalize Market" code is **fundamentally broken** because:
1. ❌ Uses wrong data model (Position table doesn't exist)
2. ❌ No blockchain interaction (just database updates)
3. ❌ Fees calculated but never distributed
4. ❌ Doesn't update market group status
5. ❌ Won't actually pay winners or burn tokens

**The code needs a complete rewrite** to properly interact with the blockchain and handle token redemption.
