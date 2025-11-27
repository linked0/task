# Batch Feature Implementation Plan

## 1. Problem Statement
The user experiences two main friction points:
1.  **"Too many signing"**: When placing or cancelling multiple orders (Maker activity), the user has to sign a transaction/message for each individual order.
2.  **"Waiting for completing transaction"**: When executing trades (Taker activity), the user waits for sequential transactions, or the backend processes them one by one, leading to slow execution.

## 2. Current State Analysis

### Contracts (`nostra-contracts`)
*   **`CTFExchange.sol`**:
    *   ✅ **Batch Execution**: Supports `fillOrders(Order[] memory orders, ...)` and `matchOrders`.
    *   ✅ **Batch Cancellation**: Supports `cancelOrders(Order[] memory orders)`.
    *   ❌ **Batch Creation**: Does **not** support Session Keys or Bulk Signatures. `Signatures.sol` strictly enforces `signer == maker`, preventing delegation to a server-side signer (Session Key).

### Backend (`nostra-server`)
*   **`MarketOrderService.ts`**: Currently fetches the single `bestOrder` and executes it. It does not "sweep" the order book using `fillOrders`.
*   **`TradeExecutionService.ts`**: Currently implements AMM-style `splitPosition`/`mergePositions` logic, which is separate from the Order Book matching logic.

## 3. Proposed Solutions

### A. Batch Execution (Taker Optimization)
**Goal**: Allow a single Market Buy/Sell to fill multiple orders instantly without multiple transactions.

*   **Implementation**:
    1.  Modify `MarketOrderService.ts` to fetch *all* orders needed to fill the requested amount (sweeping the book).
    2.  Construct an array of `orders` and `fillAmounts`.
    3.  Call `ctfExchange.fillOrders()` (or `matchOrders`) in a single transaction.
*   **Impact**: Faster execution, lower gas fees, atomic fill (all or nothing).

### B. Batch Cancellation (Maker Optimization)
**Goal**: Allow users to cancel multiple open orders with a single signature/transaction.

*   **Implementation**:
    1.  **Frontend**: Add checkboxes to the "Open Orders" list in `MyPositionsPage`.
    2.  **Frontend**: Create a "Cancel Selected" button.
    3.  **Frontend**: When clicked, aggregate the selected orders and call `exchange.cancelOrders(orders)`.
*   **Impact**: Significantly reduces "waiting for transaction" when managing orders.

### C. Batch Creation (Maker Optimization)
**Goal**: Allow users to place multiple orders without signing each one individually.

*   **Challenge**: The current contract (`Signatures.sol`) requires `signer == maker`. This prevents using a "Session Key" (where the user approves a temporary key for the server to sign on their behalf).
*   **Options**:
    1.  **Contract Upgrade (Recommended)**: Update `Signatures.sol` to support a `DELEGATE` or `POLY_PROXY` signature type. This would allow `isValidSignature` to return `true` if the signer is an approved delegate of the maker.
    2.  **EIP-712 Bulk Signing**: Update the contract to verify a signature of a Merkle Root of orders. This is complex and less flexible than Session Keys.
*   **Recommendation**: If contract upgrades are possible, implement **Session Keys**. This allows "One-Click Trading" and is essential for Market Maker bots.

## 4. Implementation Roadmap

### Phase 1: Low Hanging Fruit (No Contract Changes)
1.  **Batch Cancellation**: Implement `cancelOrders` in the Frontend.
2.  **Batch Execution**: Update `MarketOrderService` to use `fillOrders`.

### Phase 2: Advanced Features (Contract Upgrade Required)
1.  **Session Keys**: Upgrade `Signatures.sol` and `Auth.sol` to allow delegate signing.
2.  **Market Maker Bot**: Once Session Keys are live, build a bot that can place/cancel orders automatically without user intervention.

## 5. Immediate Next Steps
1.  **Frontend**: Add "Select All" and "Cancel Selected" to `OpenOrders` component.
2.  **Backend**: Refactor `MarketOrderService.executeMarketOrder` to use `fillOrders`.
