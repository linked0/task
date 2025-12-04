# Task 3: Batching Transactions & Signatures Design

## Objective
Reduce the number of user interactions (wallet popups) required for trading, specifically for:
1.  **Batch Execution**: Performing multiple actions (e.g., buying multiple outcomes) in a single transaction.
2.  **Batch Creation**: Placing multiple limit orders without signing each one individually.

## Proposed Solution

### 1. Batch Execution (Transactions) using Multicall

**Problem**: If a user wants to buy YES shares in 3 different markets, they currently send 3 transactions.
**Solution**: Implement `Multicall` in the `CTFExchange` contract.

**Contract Changes (`nostra-contracts`)**:
- Inherit from `@openzeppelin/contracts/utils/Multicall.sol` in `CTFExchange`.
- This adds the `multicall(bytes[] calldata data)` function.
- Users can bundle calls to `fillOrder`, `matchOrders`, `cancelOrder`, `deposit`, etc.

**Frontend Changes (`web`)**:
- Use `multicall` when the user performs a "Batch Action" (e.g., "Bet on All" or "Hedge").

### 2. Batch Creation (Signatures) using Session Keys

**Problem**: Creating multiple limit orders requires signing an EIP-712 message for *each* order.
**Solution**: Implement **Session Keys (Delegates)**. The user approves a temporary key (Session Key) to sign orders on their behalf.

**Contract Changes (`nostra-contracts`)**:
- **New Mixin**: `contracts/exchange/mixins/Delegates.sol` (or add to `Auth.sol`).
    ```solidity
    mapping(address => mapping(address => bool)) public delegates;
    event DelegateUpdated(address indexed user, address indexed delegate, bool status);

    function setDelegate(address delegate, bool status) external {
        delegates[msg.sender][delegate] = status;
        emit DelegateUpdated(msg.sender, delegate, status);
    }
    ```
- **Update `Signatures.sol`**:
    - Modify `verifyEOASignature`:
        ```solidity
        function verifyEOASignature(address signer, address maker, bytes32 structHash, bytes memory signature) internal view returns (bool) {
            // Allow if signer is maker OR signer is a delegate of maker
            if (signer != maker && !delegates[maker][signer]) return false;
            return verifyECDSASignature(signer, structHash, signature);
        }
        ```
    - Note: `validateOrderSignature` needs to be `view` (it is), but `delegates` mapping access is fine.
    - **Important**: `Signatures` mixin currently doesn't have access to `delegates`. We need to expose `delegates` via an interface or inherit `Delegates` in `Signatures` (or `CTFExchange` overrides `validateOrderSignature` and checks delegates).
    - **Better Approach**: `CTFExchange` inherits `Delegates`. `CTFExchange` overrides `validateOrderSignature` (which is virtual in `Signatures`).

**Frontend Changes (`web`)**:
- Generate a local key pair (Session Key) in the browser.
- Prompt user to `setDelegate(sessionKeyAddress, true)` (One transaction).
- Store Session Key securely (e.g., session storage).
- When placing orders, sign with Session Key. No wallet popup required.

## Implementation Roadmap

### Phase 1: Multicall (Immediate)
1.  Add `Multicall` inheritance to `CTFExchange`.
2.  Redeploy.
3.  Update Frontend to use `multicall` for batch trades.

### Phase 2: Session Keys (Future Scope)
*Note: To be implemented in a later iteration to further reduce signature fatigue.*
1.  Implement `Delegates` logic in contracts.
2.  Update `validateOrderSignature`.
3.  Redeploy.
4.  Implement Session Key management in Frontend.

## Considerations
- **Security**: Session keys should have an expiration or scope. However, for MVP, a simple boolean delegate is sufficient (User can revoke).
- **Gas**: `Multicall` adds slight overhead but saves significant gas compared to multiple txs.
- **Complexity**: Session Keys require careful frontend state management.
