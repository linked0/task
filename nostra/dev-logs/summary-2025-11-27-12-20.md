# Summary of Changes - Task 1: Claimed or Won in History

## Overview
Implemented the feature to display claimed winnings in the history list instead of removing them entirely. This ensures users have a record of their successful claims.

## Changes
1.  **Web Application (`nostra-server/web`)**:
    -   **`src/app/my-positions/page.tsx`**:
        -   Lifted `history` state and `fetchHistory` logic from `HistoryList` to the parent page component.
        -   Added `fetchHistory` to the `fetchData` sequence to ensure history is refreshed immediately after a successful claim.
        -   Passed `history` as a prop to `HistoryList`.
        -   Defined `HistoryItem` interface to fix type safety.
    -   **`src/components/my-positions/HistoryList.tsx`**:
        -   Updated component to accept `history` as a prop instead of fetching it internally.
        -   Fixed type errors related to `parseFloat` and `formatCurrency`.
        -   Ensured "Claimed" items (type `CLAIMED_HISTORY`) are displayed with a "Claimed" badge, while "Claimable" items (type `CLAIMABLE`) show the "Claim" button.

## Result
-   When a user claims a winning position, the "Claim" button changes to a loading state.
-   Upon success, the list refreshes.
-   The "Won" item (Claimable) is removed (as balance goes to 0).
-   A new "Claimed" item appears in the history list with the same value and a "Claimed" badge, providing immediate feedback and a permanent record.

## Original Request
> **Task-1: Claimed or Won in History**
> You remove the Claim item after a user claimed the winnings. But I think it's good for a user to see the history of their claims with the information of winnings. So you can keep the Claim item in the history but disable the claim button.
