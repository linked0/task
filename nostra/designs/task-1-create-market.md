# Task 1: Create Market Implementation Design

## Objective
Enable the "Create Market" page to functional correctly by implementing the backend endpoint to create markets on-chain and store them in the database.

## Current State
- **Frontend**: `web/src/app/create-market/page.tsx` exists but points to a non-existent endpoint `/api/markets/grouped`. It also displays instructions to run a manual script.
- **Backend**: `api/src/routes/markets.ts` lacks the `POST /api/markets/grouped` endpoint.
- **Contracts**: `MarketFactory` contract exists and has `createBinaryMarket` function.
- **Script**: `scripts/interact/world-series-mvp/01-create-market.ts` exists for manual creation.

## Important Note: Multi-Choice Market Limitation
`MarketFactory.createMultipleChoiceMarket` does **not** register tokens with the `CTFExchange`, making them untradable on the current exchange implementation.
*   **Impact**: We cannot create true Multi-Choice markets (A vs B vs C) that are tradable.
*   **Solution**: We use **Grouped Binary Markets** (Polymarket style).
    *   "Who will win?" -> [Will A win? (YES/NO)], [Will B win? (YES/NO)].
    *   Each option is a separate Binary Market.
    *   This is fully supported by the current contracts and exchange.

## Liquidity Provisioning

### 2. Frontend Implementation (`web/src/app/create-market/page.tsx`)
*   **UI Components:**
    *   Market Details Form (Question, Description, End Date).
    *   **Liquidity Input:** Field for "Initial Liquidity" (e.g., 100 USDC).
    *   **Outcome Configuration:** For Multi-choice, allow adding outcomes.
*   **Interaction Flow:**
    1.  **Approve USDC:** User approves `ConditionalTokens` to spend Liquidity Amount.
    2.  **Create Market:** Call `MarketFactory.createMarket()`.
    3.  **Mint Tokens:** Call `ConditionalTokens.splitPosition()` to mint outcome tokens for the Creator.
    4.  **Sign Orders:** User signs EIP-712 orders to sell the minted tokens (seeding the book).
    5.  **Submit to API:** POST `/api/markets/grouped` with market details and signed orders.

## Proposed Solution

### 1. Backend Implementation (`api`)

**New Endpoint**: `POST /api/markets/grouped`

**Request Body**:
```json
{
  "groupQuestion": "Who will win...?",
  "category": "Sports",
  "players": [
    { "name": "Player A", "description": "...", "imageUrl": "https://..." },
    { "name": "Player B", "description": "...", "imageUrl": "" }
  ],
  "resolutionTime": 1735689600, // Unix timestamp
  "endTime": 1735603200, // Unix timestamp
  "marketType": "grouped-binary"
}
```

**Logic**:
1.  **Validation**: Check inputs.
2.  **Blockchain Interaction**:
    *   Load Admin Private Key from environment variables (`ADMIN_PRIVATE_KEY`).
    *   Initialize `BlockchainService` with signer.
    *   For each player:
        *   Generate `questionId` (using `ethers.id` + timestamp/salt).
        *   Call `marketFactory.createBinaryMarket(...)`.
        *   Wait for transaction confirmation.
        *   Parse `MarketCreated` event to get `conditionId` and `tokenIds`.
    *   **Retry Logic**:
        *   Wrap the blockchain call in a retry loop.
        *   `MAX_RETRIES = 3`.
        *   `RETRY_DELAY = 2000` ms.
        *   If a transaction fails (e.g., network timeout, RPC error), wait and retry.
        *   If it fails after 3 attempts, mark this specific market creation as `FAILED` in the response/DB but continue processing other players if possible (or abort depending on strictness). *Recommendation: Abort and return partial success details.*
3.  **Database Storage**:
    *   Create `MarketGroup` record.
    *   For each player, create `Market` record with the returned `conditionId` and the provided `imageUrl`.
    *   Create `Outcome` records (YES/NO) with returned `tokenIds`.
4.  **Response**: Return the created market group details.

**File Changes**:
- `api/src/routes/markets.ts`: Add `POST /grouped` route.
- `api/src/services/MarketService.ts`: Add `createGroupedMarket` method (refactoring logic out of routes).
- `api/.env`: Ensure `ADMIN_PRIVATE_KEY` is present.

### 2. Frontend Implementation (`web`)

**Updates**:
- Remove the "Run the deployment script" instruction card.
- Update the success message to indicate markets are live on-chain.
- Ensure the form sends the correct data structure.
- **Image Support**:
    - Add an optional "Image URL" input field for each player.
    - **Fallback UI**: If `imageUrl` is empty or fails to load, display a placeholder avatar using the first letter of the player's name (e.g., "S" for Shohei, "W" for Who will win).
- **Navigation**:
    - On success, redirect the user to the **Market Detail Page** of the newly created market group (e.g., `/market/[marketGroupId]`).
- **My Markets Tab**:
    - Update `web/src/app/my-positions/page.tsx` to add a new tab: **"Markets"**.
    - Display a list of markets created by the current user.
    - Columns: Title, Volume, Status, Created At, **Actions**.
    - **Actions**:
        - **Update Button**: Enabled *only* before the market starts (e.g., before `startTime` or first trade).
        - Opens a modal to edit metadata (Description, Image URL). *Note: On-chain data (Question, Outcomes) cannot be changed.*

**File Changes**:
- `web/src/app/create-market/page.tsx`: Update UI text, error handling, and add Image URL input.
- `web/src/app/my-positions/page.tsx`: Add "Markets" tab with Update functionality.
- `api/src/routes/users.ts`: Add `GET /:address/markets` endpoint to fetch created markets.
- `api/src/routes/markets.ts`: Add `PUT /grouped/:id` (or `/markets/:id`) to update metadata.

## Database Schema
- **Update Required**: Add `imageUrl` column to `Market` table.
```prisma
model Market {
  // ... existing fields
  imageUrl String? @map("image_url") // URL for market/player image
}

model Outcome {
  // ... existing fields
  imageUrl String? @map("image_url") // URL for outcome image (for future multi-choice support)
}
```
- Run `npx prisma migrate dev` to apply changes.

## Considerations
- **Gas Fees**: The Admin wallet will pay for gas. Ensure it is funded.
- **Latency**: Creating multiple markets on-chain sequentially might take time (block times). The UI should show a loading state (e.g., "Creating market 1 of 3...").
- **Error Handling**: If one market fails, we should handle partial creation or retry.

## Step-by-Step Plan
1.  **Backend**: Implement `MarketService.createGroupedMarket`.
2.  **Backend**: Add route `POST /api/markets/grouped`.
3.  **Frontend**: Update `create-market/page.tsx`.
4.  **Test**: Create a market via UI and verify it appears in DB and on-chain (via logs).

## Image Upload Implementation
We use **Multer**, a node.js middleware for handling `multipart/form-data`, which is primarily used for uploading files.

### Implementation Details
1.  **Backend (`api`)**:
    *   **Middleware**: `multer` is configured to store uploaded files in the local `api/uploads` directory.
    *   **Route**: `POST /api/upload` accepts a single file, saves it with a unique name (timestamp + random), and returns the relative URL (e.g., `/uploads/12345.png`).
    *   **Static Serving**: `express.static` serves the `uploads` directory at the `/uploads` path.
    *   **Market Creation**: The `POST /api/markets/grouped` endpoint was updated to accept an `imageUrl` field and store it in the `MarketGroup` record.

2.  **Frontend (`web`)**:
    *   **UI**: Added a file input field for "Market Image".
    *   **Logic**: When a file is selected, it is immediately uploaded to `/api/upload`. The returned URL is stored in state and then sent as part of the market creation payload.
