# Testing Guide - New Features Implementation

Complete testing guide for all newly implemented features in the Nostra prediction market platform.

## 📋 Features to Test

1. ✅ MULTI_CHOICE Market Type
2. ✅ Market Creation Page Selector
3. ✅ Market Resolution Function (4% Fee)
4. ✅ Orderbook Depth (2 → 5)
5. ✅ Hybrid Market Maker Bot

---

## Prerequisites

### Start the Development Servers

```bash
# Terminal 1 - API Server
cd ~/work/nostra-server/api
yarn dev

# Terminal 2 - Web Frontend
cd ~/work/nostra-server/web
yarn dev
```

**URLs:**
- Frontend: http://localhost:3000
- API: http://localhost:4001

---

## Test 1: MULTI_CHOICE Market Type

### 1.1 Create a Multi-Choice Market

```bash
cd ~/work/nostra-server/api

# Create a multi-choice market (FIFA World Cup Winner)
yarn create:world-cup-mc
```

**Expected Output:**
```
✅ Multi-choice market created successfully
   Market ID: <id>
   Market Type: MULTI_CHOICE
   Outcomes: 5 choices (Brazil, Argentina, France, Germany, England)
```

### 1.2 Verify in Database

```bash
# Check market type
yarn tsx -e "
import { prisma } from './src/db/client.js';
const market = await prisma.market.findFirst({
  where: { marketType: 'MULTI_CHOICE' },
  include: { outcomes: true }
});
console.log('Market Type:', market.marketType);
console.log('Outcomes:', market.outcomes.length);
process.exit(0);
"
```

**Expected:**
```
Market Type: MULTI_CHOICE
Outcomes: 5
```

### 1.3 View in Frontend

1. Open http://localhost:3000
2. Look for "FIFA World Cup 2026 Winner" market
3. **Expected UI:**
   - Single card with all 5 choices listed
   - Each choice shows: Name, Price (¢), Probability (%)
   - Radio button selection
   - "Buy Selected Choice" button

**Verification:**
- ✅ Market displays as multi-choice (not 5 separate binary markets)
- ✅ Can select one choice
- ✅ Probabilities sum to ~100%

---

## Test 2: Market Creation Page Selector

### 2.1 Access Market Creation Page

1. Open http://localhost:3000/create-market
2. **Expected UI:**
   - **Tabs at top:** "Grouped Binary (Polymarket)" | "Multi-Choice"
   - Info alert explaining the difference
   - Form fields that change based on selection

### 2.2 Test GROUPED_BINARY Mode

1. Select "Grouped Binary (Polymarket)" tab
2. **Expected:**
   - Label: "Group Question"
   - Example: "Who will win the MVP award?"
   - Description: "Creates separate YES/NO binary markets for each player"
   - Preview shows: Multiple binary markets

### 2.3 Test MULTI_CHOICE Mode

1. Click "Multi-Choice" tab
2. **Expected:**
   - Label: "Main Question"
   - Example: "Who will win MVP?"
   - Description: "Pick ONE winner from multiple choices"
   - Preview shows: Single market with multiple choices

### 2.4 Create Test Market

**Grouped Binary Test:**
1. Select "Grouped Binary" tab
2. Enter group question: "Who will win the 2025 Super Bowl MVP?"
3. Add 3 players: Patrick Mahomes, Joe Burrow, Josh Allen
4. Click "Create Market Group"
5. **Expected:** Success message, 3 binary markets created

**Multi-Choice Test:**
1. Select "Multi-Choice" tab
2. Enter question: "Who will be 2025 NBA Finals MVP?"
3. Add 4 players: LeBron James, Nikola Jokic, Giannis, Luka Doncic
4. Click "Create Multi-Choice Market"
5. **Expected:** Success message, 1 market with 4 outcomes created

**Verification:**
- ✅ Tab switching works smoothly
- ✅ Labels and descriptions update correctly
- ✅ Both market types create successfully
- ✅ Markets appear correctly on home page

---

## Test 3: Market Resolution Function

### 3.1 Prepare Test Market

```bash
cd ~/work/nostra-server/api

# Create test market (World Series MVP)
yarn create:mvp

# Provision tokens
yarn provision:mvp

# Check market status
yarn status:mvp
```

### 3.2 Place Some Trades

1. Open market detail page: http://localhost:3000/market/1
2. Buy some shares in different outcomes (simulate user trading)
3. Use demo trader to create activity:

```bash
# Run demo trader to create trading activity
yarn trader:demo
```

### 3.3 Test Resolution (Admin Only)

**Important:** Resolution UI only appears for admin wallet address.

1. Open market detail page
2. Scroll to bottom of Trading Panel
3. **Expected Admin UI:**
   - Section: "Admin: Resolve Market"
   - Dropdown: "Select Winning Outcome"
   - Button: "Resolve Market" (green)
   - Warning: "This will distribute payouts with 4% fee deduction"

4. Select winning outcome from dropdown
5. Click "Resolve Market"
6. **Expected:**
   - Confirmation dialog
   - Success alert with details:
     - Total Value Locked
     - Platform Fee (4%)
     - Payout Pool (96%)
     - Winners Count

### 3.4 Verify Resolution

```bash
# Check market status after resolution
cd ~/work/nostra-server/api
yarn tsx -e "
import { prisma } from './src/db/client.js';
const market = await prisma.market.findFirst({
  where: { status: 'RESOLVED' },
  include: {
    outcomes: true,
    positions: { include: { user: true } }
  }
});
console.log('Status:', market.status);
console.log('Resolved Outcome:', market.resolvedOutcomeId);
console.log('Winners:', market.positions.filter(p => p.realizedProfit > 0).length);
process.exit(0);
"
```

**Verification:**
- ✅ Market status changed to "RESOLVED"
- ✅ Winning outcome marked correctly
- ✅ Winners received payouts (96% of total)
- ✅ User cash balances updated
- ✅ Platform kept 4% fee

---

## Test 4: Orderbook Depth (2 → 5)

### 4.1 Seed Orders

```bash
cd ~/work/nostra-server/api

# Seed orders for a market
yarn seed-orders
```

### 4.2 View Orderbook

1. Open any market detail page
2. Look at "Orderbook" section
3. **Expected:**
   - **5 rows** in "Asks" (Sell orders) - previously only 2
   - **5 rows** in "Bids" (Buy orders) - previously only 2
   - Each row shows: Price, Shares, Total

**Example:**
```
ASKS (Sell YES)
Price    Shares    Total
52¢      100       $52.00
53¢      150       $79.50
54¢      200       $108.00
55¢      250       $137.50
56¢      300       $168.00

BIDS (Buy YES)
48¢      100       $48.00
47¢      150       $70.50
46¢      200       $92.00
45¢      250       $112.50
44¢      300       $132.00
```

**Verification:**
- ✅ Exactly 5 asks displayed (not 2)
- ✅ Exactly 5 bids displayed (not 2)
- ✅ Best ask at top, worst at bottom
- ✅ Best bid at top, worst at bottom

---

## Test 5: Hybrid Market Maker Bot

### 5.1 Configure Market for Bot

First, mark a market group for bot handling:

```bash
cd ~/work/nostra-server/api

yarn tsx -e "
import { prisma } from './src/db/client.js';
const group = await prisma.marketGroup.findFirst({
  where: { name: { contains: 'World Series MVP' } }
});
await prisma.marketGroup.update({
  where: { id: group.id },
  data: {
    liquidityMode: 'ACTIVE_ORDERS',
    botUpperThreshold: 1.05,
    botLowerThreshold: 0.95,
    botMaxPosition: 200,
    status: 'ACTIVE'
  }
});
console.log('✅ Market configured for bot:', group.name);
process.exit(0);
"
```

### 5.2 Test Dry-Run Mode

```bash
cd ~/work/nostra-server/api

# Run bot in dry-run mode (no actual orders)
yarn bot:hybrid:dry
```

**Expected Console Output:**
```
🤖 Hybrid Market Maker Bot Starting...

Configuration:
  RPC URL: https://bnb-testnet.g.alchemy.com/v2/...
  Polling Interval: 30000 ms
  Max Concurrent Groups: 50
  Total Capital Limit: 10000 USDC
  Default Spread: 0.02
  Min Profit Per Set: 0.02 USDC
  Cooldown Period: 60000 ms
  Dry Run Mode: ✅ ENABLED

⚠️  DRY RUN MODE: No orders will be posted to the blockchain

✅ Bot initialized successfully

🚀 Starting bot...

Cycle 1 (2025-11-20 14:30:00)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Discovered 1 active market groups

📊 Priority Ranking:
1. World Series MVP (Priority: 45.0, Volume: 0)

💼 Market Making: World Series MVP
   🔧 [DRY RUN] Would post buy/sell orders with 2¢ spread...
   📝 Order details:
      - Outcome: Shohei Ohtani YES
        Buy: 48¢ for 416.67 shares
        Sell: 52¢ for 384.62 shares
   [Similar for other outcomes...]

📊 Cycle Statistics:
   Orders Posted: 0 (dry-run)
   Arbitrages Executed: 0
   Estimated Profit: $0.00
   Active Markets: 1/1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Waits 30 seconds for next cycle...]
```

**Verification:**
- ✅ Bot discovers the configured market group
- ✅ Calculates priority score
- ✅ Logs market making orders (but doesn't post them)
- ✅ Shows cycle every 30 seconds
- ✅ No actual orders in database

### 5.3 Check for Arbitrage (Optional)

Manually create arbitrage opportunity:

```bash
yarn tsx -e "
import { prisma } from './src/db/client.js';
// Update prices to create arbitrage (sum > 105%)
const outcomes = await prisma.outcome.findMany({
  where: { market: { marketGroup: { name: { contains: 'MVP' } } } },
  include: { market: true }
});
// Set YES prices to sum to 110%
await prisma.outcome.update({
  where: { id: outcomes[0].id },
  data: { currentPrice: 0.30 }
});
await prisma.outcome.update({
  where: { id: outcomes[1].id },
  data: { currentPrice: 0.30 }
});
await prisma.outcome.update({
  where: { id: outcomes[2].id },
  data: { currentPrice: 0.30 }
});
await prisma.outcome.update({
  where: { id: outcomes[3].id },
  data: { currentPrice: 0.20 }
});
console.log('✅ Created arbitrage: sum = 110%');
process.exit(0);
"
```

Then watch bot detect it:
```
💰 Arbitrage Opportunity: World Series MVP
   Sum of YES prices: 110.0%
   Expected profit: $0.10 per set
   🔧 [DRY RUN] Would post SELL orders for all YES outcomes...
```

**Verification:**
- ✅ Bot detects when sum ≠ 100%
- ✅ Calculates expected profit
- ✅ Prioritizes arbitrage over market making
- ✅ In dry-run, logs but doesn't execute

### 5.4 Test Production Mode (Careful!)

**⚠️  Warning:** This will post actual orders to the blockchain!

```bash
# Make sure bot wallet has USDC and MATIC for gas
cd ~/work/nostra-server/api

# Check balances first
yarn check-balances

# Run bot in production mode
yarn bot:hybrid
```

**Expected:**
```
Dry Run Mode: ❌ DISABLED

Cycle 1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💼 Market Making: World Series MVP
   🔧 Posting buy/sell orders with 2¢ spread...
   ✅ Posted 10 orders (5 buy, 5 sell)
   📊 Order hashes:
      - 0xabc123... (BUY Ohtani YES @ 48¢)
      - 0xdef456... (SELL Ohtani YES @ 52¢)
      [...]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 5.5 Verify Orders in Database

```bash
yarn tsx -e "
import { prisma } from './src/db/client.js';
const orders = await prisma.order.findMany({
  where: {
    maker: '0x6eDF62fc2dbDef1C050E295679a2fA8051F11621', // Bot address
    status: 'OPEN'
  },
  include: { outcome: true }
});
console.log('Active bot orders:', orders.length);
orders.forEach(o => {
  console.log(\`  \${o.side} \${o.outcome.title} @ \${o.price}¢ - \${o.remainingSize} shares\`);
});
process.exit(0);
"
```

**Verification:**
- ✅ Orders posted to database
- ✅ Buy/sell pairs for each outcome
- ✅ Spread matches configuration (2¢)
- ✅ Status is "OPEN"
- ✅ Orders have valid EIP-712 signatures

### 5.6 Test Graceful Shutdown

Press `Ctrl+C` in bot terminal.

**Expected:**
```
📡 Received SIGINT, shutting down gracefully...
✅ Bot stopped successfully

📊 Final Statistics:
  Total Orders Posted: 10
  Total Arbitrages Executed: 0
  Total Profit: 0.000000 USDC
  Active Market Groups: 1
```

**Verification:**
- ✅ Bot stops cleanly
- ✅ Shows final statistics
- ✅ No errors during shutdown

---

## Test 6: Integration Test (All Features)

### Complete User Journey

1. **Create Multi-Choice Market**
   - Use creation page with "Multi-Choice" tab
   - Create "2025 NBA MVP" with 5 players

2. **Bot Provides Liquidity**
   - Configure market for bot: `liquidityMode: ACTIVE_ORDERS`
   - Start bot: `yarn bot:hybrid`
   - Verify orders appear in orderbook (depth = 5)

3. **Users Trade**
   - Open market detail page
   - Buy shares from bot's sell orders
   - Sell shares to bot's buy orders
   - Watch orderbook update

4. **Market Resolution**
   - Wait for market to end (or manually set endTime in past)
   - Admin selects winning outcome
   - Click "Resolve Market"
   - Verify 4% fee deduction
   - Verify winners receive payouts

### Expected Results:
- ✅ Multi-choice market displays correctly
- ✅ Bot posts 5 levels of orders
- ✅ Trades execute against bot orders
- ✅ Orderbook updates after trades
- ✅ Resolution distributes payouts correctly
- ✅ Platform receives 4% fee
- ✅ Winners receive 96% of pool

---

## Troubleshooting

### Bot Not Starting

**Error:** "BOT_PRIVATE_KEY environment variable is required"
**Solution:** Set `BOT_PRIVATE_KEY` in `api/.env`

**Error:** "RPC_URL environment variable is required"
**Solution:** Set `RPC_URL` in `api/.env`

### Bot Not Finding Markets

**Check market configuration:**
```bash
yarn tsx -e "
import { prisma } from './src/db/client.js';
const groups = await prisma.marketGroup.findMany({
  where: { liquidityMode: 'ACTIVE_ORDERS' }
});
console.log('Markets configured for bot:', groups.length);
process.exit(0);
"
```

**Solution:** Ensure `liquidityMode: ACTIVE_ORDERS` and `status: ACTIVE`

### Resolution Button Not Showing

**Issue:** You're not logged in as admin wallet

**Check admin address:**
```bash
# Check which address is admin
grep DEPLOYER_PRIVATE_KEY api/.env
# Derive address from that private key
```

**Solution:** Connect wallet with admin address

### Orderbook Shows Only 2 Rows

**Issue:** Not enough orders in database

**Solution:** Run `yarn seed-orders` to populate orderbook

---

## Success Criteria

### ✅ All Tests Pass When:

1. **Multi-Choice Markets:**
   - Can create via UI with tab selector
   - Display as single market with multiple choices
   - Trades execute correctly

2. **Market Creation:**
   - Both tabs work (Grouped Binary, Multi-Choice)
   - Labels and descriptions update dynamically
   - Preview shows correct structure

3. **Market Resolution:**
   - Admin UI appears for admin wallet
   - Resolution executes successfully
   - 4% fee deducted correctly
   - Winners receive payouts

4. **Orderbook Depth:**
   - Shows 5 asks (not 2)
   - Shows 5 bids (not 2)
   - Updates after trades

5. **Hybrid Bot:**
   - Auto-discovers markets from database
   - Posts market making orders
   - Detects arbitrage opportunities
   - Dry-run mode works
   - Production mode posts real orders
   - Graceful shutdown works

---

## Performance Benchmarks

Expected performance metrics:

| Metric | Target | Actual |
|--------|--------|--------|
| Bot discovery time | < 5s | _____ |
| Order posting time | < 10s | _____ |
| Market creation time | < 3s | _____ |
| Resolution time | < 5s | _____ |
| Orderbook render time | < 1s | _____ |

Fill in "Actual" column during testing.

---

## Next Steps After Testing

If all tests pass:
1. ✅ Deploy to staging environment
2. ✅ Run bot on testnet for 24 hours
3. ✅ Monitor for errors/issues
4. ✅ Fine-tune bot parameters based on performance
5. ✅ Document any edge cases found
6. ✅ Prepare for mainnet deployment

---

## Support

If you encounter issues:
1. Check console for error messages
2. Review logs in bot terminal
3. Verify environment variables in `api/.env`
4. Check database state with SQL queries
5. Test in dry-run mode first

**All features tested and working?** You're ready for production! 🚀
