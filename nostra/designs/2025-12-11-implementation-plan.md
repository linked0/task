# Comprehensive Implementation Plan: Market Creation, Deposits & Batching

**Date**: 2025-12-11
**Status**: DRAFT
**Author**: Antigravity

## 1. Executive Summary

This document outlines the implementation plan for three critical tasks: **Market Creation Improvements**, **Deposit/Withdrawal Model**, and **Batch Transactions**.

**Key Finding**: A review of the `nostra-contracts` codebase confirms that **Smart Contract logic is largely READY** for Tasks 2 and 3.
*   `CTFExchange.sol` already inherits `Multicall`.
*   `CTFExchange.sol` already uses `AssetOperations` which inherits `Balances.sol` (implementing internal `balances` and `deposit`/`withdraw` logic).

Therefore, the work will focus primarily on **Frontend (Web)** and **Backend (API)** integration.

---

## 2. Task 1: Market Creation & UI Fixes

**Objective**: Fix the functional and UX issues on the "Create Market" page, aligning it with the robust `create.ts` backend script.

### 2.1. Current Issues
*   Page relies on manual "Mint" and "Approve" steps which are clunky.
*   Creates single markets instead of Market Groups.
*   Does not trigger the Liquidity Bot.
*   Contains placeholder/example text instead of ghost text.

### 2.2. Implementation Steps (Frontend)

1.  **Component Refactor**: Rewrite `create-market/page.tsx`.
    *   **Input State**: Initialize with empty strings (remove hardcoded "Who will win...").
    *   **Category Selection**: Fetch dynamic categories from `GET /api/categories` (need to implement this endpoint if missing, or mock for now).
    *   **Structure**: Change "Players" input array to "Outcomes".

2.  **Submission Logic ("Create Market Group")**:
    *   **Remove**: The sequential "Create -> Mint -> Approve -> Seed" flow for the user.
    *   **New Flow**:
        1.  User clicks "Create".
        2.  Frontend prompts user to sign `MarketFactory.createBinaryMarket` transactions (one per outcome).
        3.  Frontend collects `conditionId`s and `tokenIds`.
        4.  Frontend sends **Single Payload** to `POST /api/markets/grouped`.
        5.  **Payload**: Includes `liquidityConfig: { mode: 'ACTIVE_ORDERS' }`.

3.  **Backend Alignment**:
    *   Ensure `POST /api/markets/grouped` handles the reception of the payload and correctly triggers the Bot (or sets the flag for the Bot to pick up).

### 2.3. Deliverables
*   Working `create-market` page.
*   Successful creation of a Market Group (e.g., "Fed Decision") with multiple outcomes.
*   Liquidity automatically provided by Bot (no user manual seeding).

---

## 3. Task 2: Deposit / Withdrawal Model

**Objective**: Move from "Wallet Balance" trading to "Exchange Balance" trading to reduce transaction friction and sign-offs.

### 3.1. Contract Verification (Confirmed)
*   `CTFExchange.sol` logic:
    *   `deposit(amount)` -> calls `_deposit` (Updates `balances[msg.sender]`, pulls USDC).
    *   `withdraw(amount)` -> calls `_withdraw` (Checks `balances`, pushes USDC).
    *   `fillOrder` -> calls `_transferCollateral` which uses internal `balances`.
*   **Action**: No Solidity changes required.

### 3.2. Implementation Steps (Frontend)

1.  **Global State Update**:
    *   Update `UserContext` or `useUserBalance` hook.
    *   **Fetch**: Call `ctfExchange.balances(userAddress)` to get the "Trading Balance".
    *   **Display**: Show this "Trading Balance" in the Header and Portfolio.

2.  **Deposit/Withdraw Modal**:
    *   Create `components/modals/DepositWithdrawModal.tsx`.
    *   **UI**:
        *   **Deposit Tab**: Input Amount -> "Approve USDC" (if needed) -> "Deposit" (calls `exchange.deposit()`).
        *   **Withdraw Tab**: Input Amount -> "Withdraw" (calls `exchange.withdraw()`).
        *   Show "Wallet Balance" vs "Exchange Balance" clearly.

3.  **Trading Flow Check**:
    *   Verify that `fillOrder` (Market Buy) works with internal balance. (Contract handles this, Frontend just needs to *allow* the tx without checking Wallet USDC balance, checking Exchange Balance instead).

### 3.3. Deliverables
*   "Deposit" button in Header.
*   Functional Modal for Depositing and Withdrawing USDC.
*   Correct Balance display in UI.

---

## 4. Task 3: Batch Transactions

**Objective**: Reduce user interactions for makers and takers.

### 4.1. Batch Execution (Taker / Backend)

**Problem**: `MarketOrderService` currently matches orders one-by-one, potentially causing race conditions or slow fills.

**Implementation**:
1.  **Backend (`nostra-server`)**:
    *   Modify `MarketOrderService.ts`.
    *   Algorithm:
        1.  Fetch Orderbook.
        2.  Calculate orders needed to fill requested amount.
        3.  Prepare arrays: `orders[]`, `fillAmounts[]`.
        4.  **Execute**: Call `ctfExchange.fillOrders(orders, fillAmounts)` (Batch Fill).
    *   **Benefit**: Atomic fill, one transaction for the Relayer/Server.

### 4.2. Batch Cancellation (Maker / Frontend)

**Problem**: Cancelling 5 orders requires 5 signatures.

**Implementation**:
1.  **Frontend (`web`)**:
    *   Update `OpenOrdersTable.tsx`.
    *   Add **Checkboxes** to select multiple orders.
    *   Add **"Cancel Selected"** button.
    *   **Logic**:
        *   Construct `cancelOrder(order)` calls.
        *   Encode function data.
        *   Execute **One Transaction**: `ctfExchange.multicall([call1, call2, call3...])`.

### 4.3. Deliverables
*   Backend service using `fillOrders`.
*   Frontend "Cancel Selected" button working with `multicall`.

---

## 5. Execution Order

1.  **Frontend - Deposit UI** (Task 2): Prerequisite for testing trading with new balance model.
2.  **Frontend - Market Creation** (Task 1): Fix the broken page.
3.  **Frontend - Batch Cancel** (Task 3): Easy win with `Multicall`.
4.  **Backend - Batch Fill** (Task 3): Optimization for order matching.
