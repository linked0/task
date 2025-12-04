# Task 2: Deposit/Withdrawal Model Design

## Objective
Implement a Deposit/Withdrawal system where users deposit USDC into the `CTFExchange` contract. Trading will then settle using these internal balances instead of performing ERC20 transfers for every trade.

## Current State
- **Contracts**: `CTFExchange` uses `AssetOperations` which performs `ERC20.transferFrom` for every trade involving collateral.
- **Frontend**: No deposit/withdrawal UI. Users trade directly from their wallet balance.

## Proposed Solution

### 1. Smart Contract Changes (`nostra-contracts`)

**New Mixin**: `contracts/exchange/mixins/Balances.sol`
```solidity
abstract contract Balances {
    // Mapping of user address to collateral balance
    mapping(address => uint256) public balances;

    event Deposit(address indexed user, uint256 amount);
    event Withdraw(address indexed user, uint256 amount);

    function deposit(uint256 amount) external virtual;
    function withdraw(uint256 amount) external virtual;
}
```

**Modify `CTFExchange.sol`**:
- Inherit from `Balances`.
- Implement `deposit`:
    - `IERC20(collateral).transferFrom(msg.sender, address(this), amount)`
    - `balances[msg.sender] += amount`
- Implement `withdraw`:
    - Check `balances[msg.sender] >= amount`
    - `balances[msg.sender] -= amount`
    - `IERC20(collateral).transfer(msg.sender, amount)`

**Modify `AssetOperations.sol`**:
- Update `_transferCollateral(address from, address to, uint256 value)`:
    - **Case 1: User to Contract (e.g. Taker buying)**:
        - If `from` is a user (not `address(this)`):
            - Check `balances[from] >= value`
            - `balances[from] -= value`
            - `balances[to] += value` (if `to` is also internal? No, `to` might be `address(this)` for minting?)
    - **Refinement**:
        - The Exchange logic often moves funds between User and Exchange (for minting) or User and User (for matching).
        - If User A buys from User B:
            - User A pays USDC -> User B.
            - Internal transfer: `balances[A] -= val`, `balances[B] += val`.
        - If User A mints (buys from pool):
            - User A pays USDC -> Exchange (to mint).
            - Internal transfer: `balances[A] -= val`. Exchange holds the USDC as collateral for the minted tokens.

    - **Revised `_transferCollateral` logic**:
        - If `from != address(this)`: Deduct from `balances[from]`.
        - If `to != address(this)`: Add to `balances[to]`.
        - If `from == address(this)`: (Exchange paying out?) - This happens during `merge` (redemption) or if Exchange is the maker?
            - If Exchange is just holding funds, `address(this)` balance isn't tracked in `balances` mapping (it's the real ERC20 balance).
            - Wait, `balances` tracks *User* claims on the USDC.
            - The *Real* USDC balance of the contract = Sum(balances) + Collateral for Open Positions.

**Migration/Deployment**:
- This requires redeploying `CTFExchange`.
- `MarketFactory` might need to know about the new Exchange? (Usually decoupled).

### 2. Frontend Implementation (`web`)

**UI Components**:
- **Deposit/Withdraw Modal**:
    - Triggered by "Deposit" button in Header.
    - Tabs: Deposit, Withdraw.
    - **Deposit**: Input amount -> Approve USDC -> Deposit (call Contract).
    - **Withdraw**: Input amount -> Withdraw (call Contract).
- **Header / Portfolio**:
    - **Update Logic**: Change "Balance" display to show the **Internal Exchange Balance** (from `CTFExchange.balances(user)`) instead of the Wallet Balance.
    - **Wallet Balance**: Show as secondary info (e.g., "Wallet: $500") inside the Deposit modal.

**Integration**:
- Update `useUserBalance` hook (or create `useExchangeBalance`) to fetch `ctfExchange.balances(user)`.
- Ensure all "Available to Trade" checks use this internal balance.

## Security Considerations (CRITICAL)
- **Reentrancy**:
    - `deposit` and `withdraw` MUST use the `nonReentrant` modifier.
    - Although USDC is not reentrant (usually), we must defend against any ERC20 behavior.
- **Checks-Effects-Interactions Pattern**:
    - **Withdraw**:
        1. Check balance (`require`).
        2. Update balance (`balances[msg.sender] -= amount`).
        3. Transfer tokens (`IERC20.transfer`).
- **Overflow/Underflow**: Solidity 0.8.x handles this automatically, but logic must be sound.
- **Access Control**: Ensure `deposit` and `withdraw` are permissionless but strictly tied to `msg.sender`.

## Step-by-Step Plan
1.  **Contracts**: Create `Balances` mixin with security modifiers.
2.  **Contracts**: Update `CTFExchange` to inherit `Balances`.
3.  **Contracts**: Update `AssetOperations` to use internal accounting for Collateral.
4.  **Contracts**: Write tests for Deposit/Withdraw/Trade flow.
5.  **Frontend**: Implement Deposit/Withdraw UI.

---

## Addendum: Market Resolution Fix (2024-12-03)

### Bug: "Condition not found" Error During Market Resolution

**Symptom**: When resolving markets via `/api/admin/resolve`, the transaction fails with `"Condition not found"`.

**Root Cause**: The `admin.ts` was calling `conditionalTokens.reportPayouts()` directly from the server wallet. However, `reportPayouts` uses `msg.sender` to derive the `conditionId`:

```solidity
// In ConditionalTokens.reportPayouts():
bytes32 conditionId = getConditionId(msg.sender, questionId, outcomeSlotCount);
```

When the market was created via `MarketFactory.createBinaryMarket()`, the condition was prepared with **ResolutionOracle** as the oracle:

```solidity
// In MarketFactory.createBinaryMarket():
ctf.prepareCondition(oracle, questionId, 2);  // oracle = ResolutionOracle address
```

So the stored `conditionId = keccak256(ResolutionOracle, questionId, 2)`.

But when the server wallet called `reportPayouts` directly, it looked up:
`keccak256(serverWallet, questionId, 2)` - which doesn't exist!

### The Fix

**Files Changed**:
1. `api/src/config/blockchain.ts` - Added `resolutionOracleAddress` to config
2. `api/src/routes/admin.ts` - Changed to use ResolutionOracle
3. `api/src/abis/ResolutionOracle.json` - Added ABI file (copied from nostra-contracts)

**Code Change** (admin.ts):
```typescript
// BEFORE (wrong):
const tx = await conditionalTokens.reportPayouts(questionId, payouts);

// AFTER (correct):
const tx = await resolutionOracle.adminFinalizeResolution(
  questionId,
  outcomeSlotCount,  // 2 for binary markets
  payouts
);
```

**Why This Works**:
- `ResolutionOracle.adminFinalizeResolution()` is an `onlyOwner` function
- The server wallet is the owner of ResolutionOracle
- ResolutionOracle internally calls `ctf.reportPayouts(questionId, payouts)`
- Since ResolutionOracle is `msg.sender`, the conditionId matches what was prepared

---

## Addendum: Redemption Payout Fix (2024-12-03)

### Bug: Payout Goes to Wallet Instead of Exchange Balance

**Symptom**: After claiming winnings, Portfolio shows incorrect value. User deposited $1000, won $9.23, but Portfolio shows $990 (the remaining exchange balance) instead of $1009.23.

**Root Cause**: The `ConditionalTokens.redeemPositions()` function sends the USDC payout directly to `msg.sender`'s wallet:

```solidity
// In ConditionalTokens.redeemPositions() - line 187:
require(collateralToken.transfer(msg.sender, totalPayout), "Transfer failed");
```

This bypasses the deposit/withdrawal model. The payout goes to the user's wallet, but the frontend displays the exchange balance (from `balances[user]`), not the wallet balance.

### The Fix

**File Changed**: `web/src/app/my-positions/page.tsx`

**Code Change** (handleClaim function):
```typescript
// After redeemPositions tx.wait(), auto-deposit payout into exchange:

const payoutAmount = data.expectedPayout;
if (payoutAmount && payoutAmount > 0) {
    // Fetch contract addresses
    const configResponse = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/config/contracts`);
    const configData = await configResponse.json();
    const exchangeAddress = configData.contracts.CTFExchange;
    const usdcAddress = configData.contracts.MockUSDC;

    // Approve and deposit
    const usdc = new ethers.Contract(usdcAddress, [...], signer);
    const exchange = new ethers.Contract(exchangeAddress, [...], signer);
    const payoutWei = ethers.parseUnits(payoutAmount.toString(), 6);

    await usdc.approve(exchangeAddress, payoutWei);
    await exchange.deposit(payoutWei);
}
```

**Why This Works**:
- After `redeemPositions`, USDC lands in the user's wallet
- We immediately approve the exchange to spend that USDC
- Then deposit it into the exchange, updating `balances[user]`
- Portfolio now correctly shows the total including winnings

### Withdraw Verification

The withdraw function in `DepositWithdrawModal.tsx` correctly transfers all funds to wallet:
```typescript
const exchange = new ethers.Contract(exchangeAddress, [
    "function withdraw(uint256 amount)"
], signer);
const amountWei = ethers.parseUnits(amount, 6);
await exchange.withdraw(amountWei);
```

This calls `Balances._withdraw()` which:
1. Checks `balances[user] >= amount`
2. Deducts from `balances[user]`
3. Transfers USDC to user via `safeTransfer`
