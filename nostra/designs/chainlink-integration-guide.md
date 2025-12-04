# Chainlink Integration Guide for Nostra Market Resolution

## Table of Contents

1. [Overview](#overview)
2. [Chainlink Products for Market Resolution](#chainlink-products)
3. [Architecture Overview](#architecture)
4. [Step 1: Getting LINK Tokens](#step-1-getting-link)
5. [Step 2: Setting Up Chainlink Subscriptions](#step-2-subscriptions)
6. [Step 3: Smart Contract Integration](#step-3-contracts)
7. [Step 4: Frontend Resolution Page Integration](#step-4-frontend)
8. [Cost Estimation](#cost-estimation)
9. [Security Considerations](#security)
10. [Implementation Roadmap](#roadmap)

---

## Overview <a name="overview"></a>

### Why Chainlink for Market Resolution?

Currently, Nostra uses a manual admin resolution process:
1. Admin calls `/api/admin/resolve` endpoint
2. Server wallet calls `ResolutionOracle.adminFinalizeResolution()`
3. Market is resolved based on admin's decision

**Problems with Manual Resolution:**
- Centralized trust (admin can manipulate)
- No verifiable source of truth
- Users must trust the platform

**Chainlink Solution:**
- Decentralized oracle network
- Verifiable data from real-world sources
- Trustless, automated resolution
- Cryptographic proofs of data authenticity

### Chainlink Products Relevant for Prediction Markets

| Product | Use Case | Best For |
|---------|----------|----------|
| **Data Feeds** | Price data (BTC, ETH, stocks) | "Will BTC exceed $100K?" |
| **Functions** | Custom API calls | Sports scores, election results |
| **Automation** | Scheduled/conditional execution | Auto-resolve at deadline |
| **VRF** | Verifiable randomness | Lottery-style markets |

---

## Chainlink Products for Market Resolution <a name="chainlink-products"></a>

### 1. Chainlink Data Feeds (Price Feeds)

**What it does:** Provides real-time, tamper-proof price data for crypto assets, commodities, forex, etc.

**Best for markets like:**
- "Will BTC exceed $100,000 by Dec 31, 2025?"
- "Will ETH/USD be above $5,000?"
- "Will GOLD reach $3,000/oz?"

**How it works:**
```
Multiple Node Operators → Aggregate Data → On-chain Price Feed
                                              ↓
                              Your Contract reads price
                                              ↓
                              Compare to threshold → Resolve
```

**Supported Networks:**
- Ethereum, Polygon, BSC, Arbitrum, Optimism, Avalanche, etc.
- BSC Testnet: Supported ✓

**Cost:** FREE to read (no LINK required for reading Data Feeds)

### 2. Chainlink Functions

**What it does:** Execute custom JavaScript code that fetches data from any API, then returns result on-chain.

**Best for markets like:**
- "Will Lakers win NBA Championship 2025?"
- "Who will win the 2024 US Presidential Election?"
- "Will Tesla stock exceed $500?"

**How it works:**
```
Your Contract → Request → Chainlink DON (Decentralized Oracle Network)
                              ↓
                    Execute JavaScript code
                              ↓
                    Fetch from API (ESPN, AP News, etc.)
                              ↓
                    Return result on-chain
                              ↓
                    Your Contract receives answer → Resolve
```

**Cost:** Requires LINK tokens (pay per request)

### 3. Chainlink Automation (Keepers)

**What it does:** Automatically call your contract functions based on time or conditions.

**Best for:**
- Auto-resolve market at deadline
- Check resolution conditions periodically
- Trigger settlement when conditions met

**How it works:**
```
Register Upkeep → Chainlink monitors your contract
                              ↓
                    Condition met? (time or custom logic)
                              ↓
                    Chainlink calls your performUpkeep()
                              ↓
                    Your contract executes resolution
```

**Cost:** Requires LINK tokens (pay for gas + premium)

---

## Architecture Overview <a name="architecture"></a>

### Current Nostra Resolution Flow

```
Admin UI → API Server → ResolutionOracle → ConditionalTokens
              ↓
    Server Wallet signs tx
              ↓
    adminFinalizeResolution()
              ↓
    reportPayouts() → Market Resolved
```

### Proposed Chainlink Resolution Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    RESOLUTION FLOW                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Option A: Price-Based Markets (Data Feeds)                 │
│  ─────────────────────────────────────────                  │
│                                                             │
│  Market Deadline → Automation triggers                      │
│        ↓                                                    │
│  Contract reads Chainlink Price Feed                        │
│        ↓                                                    │
│  Compare price vs threshold                                 │
│        ↓                                                    │
│  Auto-resolve market                                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Option B: Event-Based Markets (Functions)                  │
│  ─────────────────────────────────────────                  │
│                                                             │
│  Market Deadline → Automation triggers                      │
│        ↓                                                    │
│  Contract calls Chainlink Functions                         │
│        ↓                                                    │
│  Functions fetches API (sports, elections, etc.)            │
│        ↓                                                    │
│  Result returned on-chain                                   │
│        ↓                                                    │
│  Auto-resolve market                                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Option C: Hybrid (Manual + Chainlink verification)         │
│  ─────────────────────────────────────────────              │
│                                                             │
│  Admin proposes resolution                                  │
│        ↓                                                    │
│  Challenge period (24-48 hours)                             │
│        ↓                                                    │
│  If challenged → Chainlink verifies                         │
│        ↓                                                    │
│  Finalize based on Chainlink result                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Contract Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTRACT STRUCTURE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ChainlinkResolutionOracle.sol                              │
│  ├── Inherits: FunctionsClient, AutomationCompatible        │
│  ├── Uses: AggregatorV3Interface (Price Feeds)              │
│  │                                                          │
│  ├── registerMarket()                                       │
│  │   - Store market config (type, threshold, dataSource)    │
│  │                                                          │
│  ├── checkUpkeep() [Automation]                             │
│  │   - Check if any market ready for resolution             │
│  │                                                          │
│  ├── performUpkeep() [Automation]                           │
│  │   - Trigger resolution for ready markets                 │
│  │                                                          │
│  ├── requestResolution() [Functions]                        │
│  │   - Send request to Chainlink Functions                  │
│  │                                                          │
│  ├── fulfillRequest() [Functions callback]                  │
│  │   - Receive result, resolve market                       │
│  │                                                          │
│  └── resolveWithPriceFeed()                                 │
│      - Read price feed, compare, resolve                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1: Getting LINK Tokens <a name="step-1-getting-link"></a>

### For Testnet (BSC Testnet / Polygon Amoy)

**Option A: Chainlink Faucet**
1. Go to https://faucets.chain.link/
2. Connect wallet
3. Select network (BSC Testnet, Polygon Amoy, etc.)
4. Request LINK tokens (free for testnet)

**Option B: Bridge from other testnet**
- Use Chainlink CCIP to bridge LINK between testnets

### For Mainnet

**Option A: Buy on DEX**
```
1. Have native tokens (BNB for BSC, MATIC for Polygon)
2. Go to DEX (PancakeSwap for BSC, QuickSwap for Polygon)
3. Swap native token for LINK
4. LINK contract addresses:
   - BSC: 0x404460C6A5EdE2D891e8297795264fDe62ADBB75
   - Polygon: 0x53E0bca35eC356BD5ddDFebbD1Fc0fD03FaBad39
```

**Option B: Buy on CEX and withdraw**
```
1. Buy LINK on Binance/Coinbase
2. Withdraw to your wallet on desired chain
3. Make sure to select correct network!
```

### How Much LINK Do You Need?

| Service | Cost per Request | Monthly Estimate (100 markets) |
|---------|------------------|-------------------------------|
| Data Feeds | FREE | $0 |
| Functions | ~0.2-0.5 LINK | 20-50 LINK (~$200-500) |
| Automation | ~0.1-0.3 LINK | 10-30 LINK (~$100-300) |

**Recommendation:** Start with 50-100 LINK for testnet development, 200+ LINK for mainnet launch.

---

## Step 2: Setting Up Chainlink Subscriptions <a name="step-2-subscriptions"></a>

### Chainlink Functions Subscription

**Step 2.1: Create Subscription**

1. Go to https://functions.chain.link/
2. Connect wallet
3. Click "Create Subscription"
4. Fund subscription with LINK
5. Note your `subscriptionId`

**Step 2.2: Add Consumer Contract**

After deploying your contract:
1. Go to subscription dashboard
2. Click "Add Consumer"
3. Enter your contract address
4. Confirm transaction

### Chainlink Automation Registration

**Step 2.3: Register Upkeep**

1. Go to https://automation.chain.link/
2. Connect wallet
3. Click "Register new Upkeep"
4. Select "Custom logic" or "Time-based"
5. Enter your contract address
6. Fund with LINK
7. Set gas limit (recommended: 500,000+)

**Configuration Options:**

| Option | Value | Description |
|--------|-------|-------------|
| Gas Limit | 500,000 | Max gas for performUpkeep |
| Starting Balance | 10 LINK | Initial funding |
| Min Balance | 2 LINK | Alert threshold |
| Check Interval | 30 seconds | How often to check |

---

## Step 3: Smart Contract Integration <a name="step-3-contracts"></a>

### 3.1 Install Dependencies

```bash
# Using npm
npm install @chainlink/contracts

# Using yarn
yarn add @chainlink/contracts

# Using forge
forge install smartcontractkit/chainlink
```

### 3.2 Price Feed Integration (Simplest)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "./IConditionalTokens.sol";

contract PriceFeedResolver {
    AggregatorV3Interface internal priceFeed;
    IConditionalTokens public ctf;

    struct PriceMarket {
        bytes32 questionId;
        uint256 threshold;      // Price threshold in 8 decimals
        bool resolveAbove;      // true = YES wins if price > threshold
        uint256 deadline;
        bool resolved;
    }

    mapping(bytes32 => PriceMarket) public markets;

    // BSC Testnet BTC/USD: 0x5741306c21795FdCBb9b265Ea0255F499DFe515C
    // BSC Mainnet BTC/USD: 0x264990fbd0A4796A3E3d8E37C4d5F87a3aCa5Ebf
    constructor(address _priceFeed, address _ctf) {
        priceFeed = AggregatorV3Interface(_priceFeed);
        ctf = IConditionalTokens(_ctf);
    }

    function registerPriceMarket(
        bytes32 questionId,
        uint256 threshold,
        bool resolveAbove,
        uint256 deadline
    ) external {
        markets[questionId] = PriceMarket({
            questionId: questionId,
            threshold: threshold,
            resolveAbove: resolveAbove,
            deadline: deadline,
            resolved: false
        });
    }

    function resolveMarket(bytes32 questionId) external {
        PriceMarket storage market = markets[questionId];
        require(!market.resolved, "Already resolved");
        require(block.timestamp >= market.deadline, "Too early");

        // Get latest price from Chainlink
        (, int256 price, , , ) = priceFeed.latestRoundData();

        // Determine winner
        bool yesWins;
        if (market.resolveAbove) {
            yesWins = price > int256(market.threshold);
        } else {
            yesWins = price < int256(market.threshold);
        }

        // Report payouts to CTF
        uint256[] memory payouts = new uint256[](2);
        payouts[0] = yesWins ? 1 : 0;  // YES outcome
        payouts[1] = yesWins ? 0 : 1;  // NO outcome

        ctf.reportPayouts(questionId, payouts);
        market.resolved = true;
    }

    function getLatestPrice() public view returns (int256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();
        return price;
    }
}
```

### 3.3 Chainlink Functions Integration (Advanced)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {FunctionsClient} from "@chainlink/contracts/src/v0.8/functions/v1_0_0/FunctionsClient.sol";
import {FunctionsRequest} from "@chainlink/contracts/src/v0.8/functions/v1_0_0/libraries/FunctionsRequest.sol";

contract FunctionsResolver is FunctionsClient {
    using FunctionsRequest for FunctionsRequest.Request;

    // Chainlink Functions configuration
    bytes32 public donId;
    uint64 public subscriptionId;
    uint32 public gasLimit = 300000;

    // Market data
    struct EventMarket {
        bytes32 questionId;
        string apiEndpoint;
        string jsonPath;
        uint256 deadline;
        bool resolved;
        bytes32 pendingRequestId;
    }

    mapping(bytes32 => EventMarket) public markets;
    mapping(bytes32 => bytes32) public requestToQuestion; // requestId -> questionId

    IConditionalTokens public ctf;

    // JavaScript source code for Chainlink Functions
    // This will be stored off-chain and referenced by hash
    string public sourceCode;

    event ResolutionRequested(bytes32 indexed questionId, bytes32 requestId);
    event ResolutionReceived(bytes32 indexed questionId, uint256 result);

    constructor(
        address router,
        bytes32 _donId,
        uint64 _subscriptionId,
        address _ctf
    ) FunctionsClient(router) {
        donId = _donId;
        subscriptionId = _subscriptionId;
        ctf = IConditionalTokens(_ctf);
    }

    function setSourceCode(string memory _source) external {
        sourceCode = _source;
    }

    function registerEventMarket(
        bytes32 questionId,
        string memory apiEndpoint,
        string memory jsonPath,
        uint256 deadline
    ) external {
        markets[questionId] = EventMarket({
            questionId: questionId,
            apiEndpoint: apiEndpoint,
            jsonPath: jsonPath,
            deadline: deadline,
            resolved: false,
            pendingRequestId: bytes32(0)
        });
    }

    function requestResolution(bytes32 questionId) external returns (bytes32) {
        EventMarket storage market = markets[questionId];
        require(!market.resolved, "Already resolved");
        require(block.timestamp >= market.deadline, "Too early");

        // Build the request
        FunctionsRequest.Request memory req;
        req.initializeRequestForInlineJavaScript(sourceCode);

        // Add arguments (API endpoint, JSON path)
        string[] memory args = new string[](2);
        args[0] = market.apiEndpoint;
        args[1] = market.jsonPath;
        req.setArgs(args);

        // Send request
        bytes32 requestId = _sendRequest(
            req.encodeCBOR(),
            subscriptionId,
            gasLimit,
            donId
        );

        market.pendingRequestId = requestId;
        requestToQuestion[requestId] = questionId;

        emit ResolutionRequested(questionId, requestId);
        return requestId;
    }

    // Chainlink Functions callback
    function fulfillRequest(
        bytes32 requestId,
        bytes memory response,
        bytes memory err
    ) internal override {
        bytes32 questionId = requestToQuestion[requestId];
        EventMarket storage market = markets[questionId];

        require(!market.resolved, "Already resolved");

        if (err.length > 0) {
            // Handle error - maybe allow retry
            revert("Functions request failed");
        }

        // Decode response (expecting 0 or 1)
        uint256 result = abi.decode(response, (uint256));

        // Report payouts
        uint256[] memory payouts = new uint256[](2);
        payouts[0] = result == 1 ? 1 : 0;  // YES wins if result == 1
        payouts[1] = result == 1 ? 0 : 1;  // NO wins if result == 0

        ctf.reportPayouts(questionId, payouts);
        market.resolved = true;

        emit ResolutionReceived(questionId, result);
    }
}
```

### 3.4 Example JavaScript Source for Chainlink Functions

```javascript
// This code runs in Chainlink Functions DON
// Store this and set via setSourceCode()

const apiEndpoint = args[0];  // e.g., "https://api.sportsdata.io/v3/nba/scores/json/GamesByDate/2025-06-15"
const jsonPath = args[1];     // e.g., "Games[0].Winner"

const response = await Functions.makeHttpRequest({
  url: apiEndpoint,
  headers: {
    "Ocp-Apim-Subscription-Key": secrets.apiKey  // API key stored in Chainlink secrets
  }
});

if (response.error) {
  throw Error("API request failed");
}

// Navigate JSON path
const pathParts = jsonPath.split(".");
let result = response.data;
for (const part of pathParts) {
  if (part.includes("[")) {
    const [key, index] = part.split("[");
    result = result[key][parseInt(index)];
  } else {
    result = result[part];
  }
}

// Return 1 if home team wins, 0 if away team wins
// Customize this logic based on your market
const homeTeamWins = result === "HomeTeam" ? 1 : 0;

return Functions.encodeUint256(homeTeamWins);
```

### 3.5 Automation Integration

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@chainlink/contracts/src/v0.8/automation/AutomationCompatible.sol";

contract AutomatedResolver is AutomationCompatibleInterface {

    struct Market {
        bytes32 questionId;
        uint256 deadline;
        bool resolved;
        MarketType marketType;
    }

    enum MarketType { PRICE_FEED, FUNCTIONS }

    Market[] public pendingMarkets;

    // Called by Chainlink Automation to check if work needed
    function checkUpkeep(bytes calldata)
        external
        view
        override
        returns (bool upkeepNeeded, bytes memory performData)
    {
        for (uint i = 0; i < pendingMarkets.length; i++) {
            Market memory m = pendingMarkets[i];
            if (!m.resolved && block.timestamp >= m.deadline) {
                upkeepNeeded = true;
                performData = abi.encode(i);
                return (upkeepNeeded, performData);
            }
        }
        return (false, "");
    }

    // Called by Chainlink Automation when checkUpkeep returns true
    function performUpkeep(bytes calldata performData) external override {
        uint256 index = abi.decode(performData, (uint256));
        Market storage market = pendingMarkets[index];

        require(!market.resolved, "Already resolved");
        require(block.timestamp >= market.deadline, "Too early");

        if (market.marketType == MarketType.PRICE_FEED) {
            _resolveWithPriceFeed(market.questionId);
        } else {
            _requestFunctionsResolution(market.questionId);
        }

        market.resolved = true;
    }

    function _resolveWithPriceFeed(bytes32 questionId) internal {
        // Implementation from PriceFeedResolver
    }

    function _requestFunctionsResolution(bytes32 questionId) internal {
        // Implementation from FunctionsResolver
    }
}
```

### 3.6 Network-Specific Addresses

**BSC Testnet:**
```solidity
// Chainlink Functions Router
address constant FUNCTIONS_ROUTER = 0x...;  // Check docs.chain.link

// Price Feeds
address constant BTC_USD = 0x5741306c21795FdCBb9b265Ea0255F499DFe515C;
address constant ETH_USD = 0x143db3CEEfbdfe5631aDD3E50f7614B6ba708BA7;
address constant BNB_USD = 0x2514895c72f50D8bd4B4F9b1110F0D6bD2c97526;

// Automation Registry
address constant AUTOMATION_REGISTRY = 0x...;
```

**Polygon Amoy (Testnet):**
```solidity
// Chainlink Functions Router
address constant FUNCTIONS_ROUTER = 0xC22a79eBA640940ABB6dF0f7982cc119578E11De;

// Price Feeds
address constant BTC_USD = 0xe7656e23fE8077D438aEfbec2fAbDf2D8e070C4f;
address constant ETH_USD = 0xF0d50568e3A7e8259E16663972b11910F89BD8e7;
```

---

## Step 4: Frontend Resolution Page Integration <a name="step-4-frontend"></a>

### 4.1 Update Market Creation

```typescript
// web/src/types/market.ts
interface Market {
  id: string;
  title: string;
  // ... existing fields

  // New Chainlink fields
  resolutionType: 'MANUAL' | 'PRICE_FEED' | 'CHAINLINK_FUNCTIONS';
  chainlinkConfig?: {
    priceFeedAddress?: string;
    priceThreshold?: number;
    resolveAbove?: boolean;
    apiEndpoint?: string;
    jsonPath?: string;
  };
}
```

### 4.2 Resolution Page Component

```tsx
// web/src/app/admin/resolve/[marketId]/page.tsx
"use client";

import { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

interface ChainlinkResolutionProps {
  market: Market;
}

export default function ChainlinkResolution({ market }: ChainlinkResolutionProps) {
  const [currentPrice, setCurrentPrice] = useState<number | null>(null);
  const [resolutionStatus, setResolutionStatus] = useState<string>('pending');
  const [loading, setLoading] = useState(false);

  // Fetch current price from Chainlink
  useEffect(() => {
    if (market.resolutionType === 'PRICE_FEED') {
      fetchCurrentPrice();
    }
  }, [market]);

  const fetchCurrentPrice = async () => {
    try {
      const provider = new ethers.JsonRpcProvider(process.env.NEXT_PUBLIC_RPC_URL);

      const priceFeedABI = [
        "function latestRoundData() view returns (uint80, int256, uint256, uint256, uint80)"
      ];

      const priceFeed = new ethers.Contract(
        market.chainlinkConfig!.priceFeedAddress!,
        priceFeedABI,
        provider
      );

      const [, price] = await priceFeed.latestRoundData();
      setCurrentPrice(Number(price) / 1e8); // Convert from 8 decimals
    } catch (error) {
      console.error('Failed to fetch price:', error);
    }
  };

  const triggerResolution = async () => {
    setLoading(true);
    try {
      if (!window.ethereum) throw new Error("No wallet found");

      const provider = new ethers.BrowserProvider(window.ethereum);
      const signer = await provider.getSigner();

      // Fetch resolver contract address from API
      const configResponse = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/config/contracts`);
      const config = await configResponse.json();

      if (market.resolutionType === 'PRICE_FEED') {
        const resolverABI = [
          "function resolveMarket(bytes32 questionId) external"
        ];

        const resolver = new ethers.Contract(
          config.contracts.PriceFeedResolver,
          resolverABI,
          signer
        );

        const tx = await resolver.resolveMarket(market.questionId);
        await tx.wait();

        setResolutionStatus('resolved');
      } else if (market.resolutionType === 'CHAINLINK_FUNCTIONS') {
        const resolverABI = [
          "function requestResolution(bytes32 questionId) external returns (bytes32)"
        ];

        const resolver = new ethers.Contract(
          config.contracts.FunctionsResolver,
          resolverABI,
          signer
        );

        const tx = await resolver.requestResolution(market.questionId);
        await tx.wait();

        setResolutionStatus('pending_chainlink');
      }
    } catch (error) {
      console.error('Resolution failed:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <Card>
      <CardHeader>
        <CardTitle>Market Resolution</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Resolution Type Badge */}
        <div className="flex items-center gap-2">
          <span className="text-sm text-gray-500">Resolution Type:</span>
          <span className={`px-2 py-1 rounded text-sm ${
            market.resolutionType === 'PRICE_FEED'
              ? 'bg-blue-100 text-blue-800'
              : market.resolutionType === 'CHAINLINK_FUNCTIONS'
              ? 'bg-purple-100 text-purple-800'
              : 'bg-gray-100 text-gray-800'
          }`}>
            {market.resolutionType}
          </span>
        </div>

        {/* Price Feed Info */}
        {market.resolutionType === 'PRICE_FEED' && (
          <div className="space-y-2">
            <div className="flex justify-between">
              <span>Current Price:</span>
              <span className="font-mono">
                ${currentPrice?.toLocaleString() ?? 'Loading...'}
              </span>
            </div>
            <div className="flex justify-between">
              <span>Threshold:</span>
              <span className="font-mono">
                ${market.chainlinkConfig?.priceThreshold?.toLocaleString()}
              </span>
            </div>
            <div className="flex justify-between">
              <span>Condition:</span>
              <span>
                YES wins if price {market.chainlinkConfig?.resolveAbove ? '>' : '<'} threshold
              </span>
            </div>

            {/* Current Prediction */}
            {currentPrice !== null && (
              <div className={`p-3 rounded ${
                (market.chainlinkConfig?.resolveAbove && currentPrice > market.chainlinkConfig.priceThreshold!) ||
                (!market.chainlinkConfig?.resolveAbove && currentPrice < market.chainlinkConfig!.priceThreshold!)
                  ? 'bg-green-100 text-green-800'
                  : 'bg-red-100 text-red-800'
              }`}>
                Current prediction: {
                  (market.chainlinkConfig?.resolveAbove && currentPrice > market.chainlinkConfig.priceThreshold!) ||
                  (!market.chainlinkConfig?.resolveAbove && currentPrice < market.chainlinkConfig!.priceThreshold!)
                    ? 'YES wins'
                    : 'NO wins'
                }
              </div>
            )}
          </div>
        )}

        {/* Functions Info */}
        {market.resolutionType === 'CHAINLINK_FUNCTIONS' && (
          <div className="space-y-2">
            <div className="flex justify-between">
              <span>API Endpoint:</span>
              <span className="font-mono text-sm truncate max-w-[200px]">
                {market.chainlinkConfig?.apiEndpoint}
              </span>
            </div>
            <div className="flex justify-between">
              <span>JSON Path:</span>
              <span className="font-mono text-sm">
                {market.chainlinkConfig?.jsonPath}
              </span>
            </div>
          </div>
        )}

        {/* Resolution Status */}
        <div className="flex justify-between items-center">
          <span>Status:</span>
          <span className={`px-2 py-1 rounded text-sm ${
            resolutionStatus === 'resolved'
              ? 'bg-green-100 text-green-800'
              : resolutionStatus === 'pending_chainlink'
              ? 'bg-yellow-100 text-yellow-800'
              : 'bg-gray-100 text-gray-800'
          }`}>
            {resolutionStatus === 'resolved' && 'Resolved'}
            {resolutionStatus === 'pending_chainlink' && 'Waiting for Chainlink...'}
            {resolutionStatus === 'pending' && 'Ready to resolve'}
          </span>
        </div>

        {/* Resolve Button */}
        <Button
          onClick={triggerResolution}
          disabled={loading || resolutionStatus === 'resolved'}
          className="w-full"
        >
          {loading ? 'Processing...' :
           resolutionStatus === 'resolved' ? 'Already Resolved' :
           market.resolutionType === 'CHAINLINK_FUNCTIONS' ? 'Request Chainlink Resolution' :
           'Resolve with Chainlink Price Feed'}
        </Button>

        {/* Chainlink Links */}
        <div className="text-xs text-gray-500 space-y-1">
          <a
            href={`https://data.chain.link/`}
            target="_blank"
            rel="noopener noreferrer"
            className="text-blue-600 hover:underline block"
          >
            View Price Feed on Chainlink →
          </a>
          {market.resolutionType === 'CHAINLINK_FUNCTIONS' && (
            <a
              href={`https://functions.chain.link/`}
              target="_blank"
              rel="noopener noreferrer"
              className="text-blue-600 hover:underline block"
            >
              View Functions Request on Chainlink →
            </a>
          )}
        </div>
      </CardContent>
    </Card>
  );
}
```

### 4.3 Market Creation Form Updates

```tsx
// Add to market creation form

<div className="space-y-4">
  <Label>Resolution Type</Label>
  <Select onValueChange={(v) => setResolutionType(v as ResolutionType)}>
    <SelectTrigger>
      <SelectValue placeholder="Select resolution type" />
    </SelectTrigger>
    <SelectContent>
      <SelectItem value="MANUAL">Manual (Admin resolves)</SelectItem>
      <SelectItem value="PRICE_FEED">Chainlink Price Feed</SelectItem>
      <SelectItem value="CHAINLINK_FUNCTIONS">Chainlink Functions (API)</SelectItem>
    </SelectContent>
  </Select>

  {resolutionType === 'PRICE_FEED' && (
    <>
      <div>
        <Label>Price Feed</Label>
        <Select onValueChange={setPriceFeed}>
          <SelectTrigger>
            <SelectValue placeholder="Select price feed" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="BTC_USD">BTC / USD</SelectItem>
            <SelectItem value="ETH_USD">ETH / USD</SelectItem>
            <SelectItem value="BNB_USD">BNB / USD</SelectItem>
          </SelectContent>
        </Select>
      </div>

      <div>
        <Label>Price Threshold ($)</Label>
        <Input
          type="number"
          placeholder="100000"
          onChange={(e) => setThreshold(Number(e.target.value))}
        />
      </div>

      <div className="flex items-center gap-2">
        <Checkbox
          checked={resolveAbove}
          onCheckedChange={setResolveAbove}
        />
        <Label>YES wins if price is ABOVE threshold</Label>
      </div>
    </>
  )}

  {resolutionType === 'CHAINLINK_FUNCTIONS' && (
    <>
      <div>
        <Label>API Endpoint</Label>
        <Input
          placeholder="https://api.example.com/results"
          onChange={(e) => setApiEndpoint(e.target.value)}
        />
      </div>

      <div>
        <Label>JSON Path to Result</Label>
        <Input
          placeholder="data.winner"
          onChange={(e) => setJsonPath(e.target.value)}
        />
      </div>
    </>
  )}
</div>
```

---

## Cost Estimation <a name="cost-estimation"></a>

### Monthly Cost Breakdown

| Component | Cost per Use | 100 Markets/Month |
|-----------|--------------|-------------------|
| Data Feeds (read) | FREE | $0 |
| Functions Request | ~0.25 LINK | 25 LINK (~$250) |
| Automation Check | ~0.01 LINK | 30 LINK (~$300)* |
| Automation Execute | ~0.1 LINK | 10 LINK (~$100) |
| **Total** | - | **~65 LINK (~$650)** |

*Automation checks every 30 seconds = 86,400 checks/month per market

### Cost Optimization Tips

1. **Batch resolutions** - Resolve multiple markets in one tx
2. **Longer check intervals** - Check every 5 min instead of 30 sec
3. **Use Price Feeds when possible** - They're FREE
4. **Cache Functions results** - Don't re-request for same data

---

## Security Considerations <a name="security"></a>

### Trust Assumptions

| Component | Trust Level | Risk |
|-----------|-------------|------|
| Chainlink Data Feeds | High (decentralized nodes) | Low |
| Chainlink Functions | Medium (relies on API) | Medium |
| API Sources | Varies by provider | Medium-High |
| Your Contract | Depends on audit | Medium |

### Security Checklist

- [ ] Validate Chainlink response data
- [ ] Handle Functions errors gracefully
- [ ] Implement fallback to manual resolution
- [ ] Add admin override for edge cases
- [ ] Set appropriate gas limits
- [ ] Monitor subscription balances
- [ ] Audit contract before mainnet

### Edge Cases to Handle

```solidity
// 1. Chainlink Functions timeout
function handleFunctionsTimeout(bytes32 questionId) external onlyAdmin {
    EventMarket storage market = markets[questionId];
    require(block.timestamp > market.deadline + 1 days, "Wait for timeout");
    // Allow admin fallback
}

// 2. Stale price data
function resolveMarket(bytes32 questionId) external {
    (, int256 price, , uint256 updatedAt, ) = priceFeed.latestRoundData();
    require(block.timestamp - updatedAt < 1 hours, "Stale price data");
    // ...
}

// 3. API returns unexpected data
// Handle in JavaScript source with try/catch
```

---

## Implementation Roadmap <a name="roadmap"></a>

### Phase 1: Price Feed Markets (Week 1-2)

- [ ] Deploy PriceFeedResolver contract
- [ ] Test with BTC/USD, ETH/USD price feeds
- [ ] Create "Will BTC exceed $X?" market
- [ ] Test resolution flow
- [ ] Update frontend resolution page

### Phase 2: Automation Integration (Week 3)

- [ ] Add AutomationCompatible to resolver
- [ ] Register Upkeep on Chainlink
- [ ] Test auto-resolution at deadline
- [ ] Monitor and tune gas settings

### Phase 3: Functions for Event Markets (Week 4-5)

- [ ] Set up Chainlink Functions subscription
- [ ] Deploy FunctionsResolver contract
- [ ] Write JavaScript source for sports API
- [ ] Test with real sports data
- [ ] Handle edge cases (API down, etc.)

### Phase 4: Production Polish (Week 6)

- [ ] Security audit
- [ ] Mainnet deployment
- [ ] Monitoring dashboards
- [ ] Documentation for market creators
- [ ] Admin fallback procedures

---

## Quick Reference

### Chainlink Docs Links

- [Data Feeds](https://docs.chain.link/data-feeds)
- [Functions](https://docs.chain.link/chainlink-functions)
- [Automation](https://docs.chain.link/chainlink-automation)
- [VRF](https://docs.chain.link/vrf)

### Contract Addresses (BSC Testnet)

```
LINK Token: 0x84b9B910527Ad5C03A9Ca831909E21e236EA7b06
VRF Coordinator: 0x6A2AAd07396B36Fe02a22b33cf443582f682c82f
Functions Router: Check docs.chain.link for latest
```

### Useful Commands

```bash
# Check LINK balance
cast call $LINK_TOKEN "balanceOf(address)" $YOUR_ADDRESS --rpc-url $RPC_URL

# Read price feed
cast call $BTC_USD_FEED "latestRoundData()" --rpc-url $RPC_URL

# Fund automation
cast send $AUTOMATION_REGISTRY "addFunds(uint256,uint96)" $UPKEEP_ID $AMOUNT --rpc-url $RPC_URL
```

---

---

## Code Comparison: Admin vs Chainlink Resolution <a name="code-comparison"></a>

### Current Admin Resolution Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CURRENT: ADMIN RESOLUTION                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Frontend (Admin UI)                                                    │
│       │                                                                 │
│       ▼                                                                 │
│  POST /api/admin/resolve                                                │
│  { marketId, winningOutcomeId }                                         │
│       │                                                                 │
│       ▼                                                                 │
│  API Server (admin.ts)                                                  │
│  ├── 1. Validate request                                                │
│  ├── 2. Fetch market from database                                      │
│  ├── 3. Connect to blockchain (SERVER_WALLET)                           │
│  ├── 4. Call ResolutionOracle.adminFinalizeResolution()                 │
│  ├── 5. Wait for tx confirmation                                        │
│  ├── 6. Update database (status: RESOLVED)                              │
│  └── 7. Cancel unfilled orders                                          │
│       │                                                                 │
│       ▼                                                                 │
│  ResolutionOracle Contract                                              │
│  └── reportPayouts() → ConditionalTokens                                │
│       │                                                                 │
│       ▼                                                                 │
│  Market Resolved ✓                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Current API Server Code (admin.ts)

```typescript
// api/src/routes/admin.ts - CURRENT IMPLEMENTATION

adminRoutes.post('/resolve', async (req, res) => {
  try {
    const { marketId, winningOutcomeId } = req.body;

    // 1. Validate
    if (!marketId || !winningOutcomeId) {
      return res.status(400).json({ error: 'Missing parameters' });
    }

    // 2. Fetch market
    const market = await prisma.market.findUnique({
      where: { id: marketId },
      include: { outcomes: true }
    });

    // 3. Connect blockchain with SERVER wallet
    const config = getBlockchainConfig();
    const provider = new ethers.JsonRpcProvider(config.rpcUrl);
    const operator = new ethers.Wallet(config.serverWalletPrivateKey, provider);

    // 4. Load ResolutionOracle contract
    const resolutionOracle = new ethers.Contract(
      config.resolutionOracleAddress,
      resolutionOracleABI,
      operator  // <-- Server wallet signs the transaction
    );

    // 5. Admin decides the winner manually
    const payouts = market.outcomes.map((outcome) => {
      return outcome.id === winningOutcomeId ? 100 : 0;
    });

    // 6. Call contract - SERVER WALLET PAYS GAS
    const tx = await resolutionOracle.adminFinalizeResolution(
      market.questionId,
      market.outcomes.length,
      payouts
    );
    await tx.wait();

    // 7. Update database
    await prisma.market.update({
      where: { id: marketId },
      data: { status: 'RESOLVED', resolvedOutcomeId: winningOutcomeId }
    });

    res.json({ success: true, txHash: tx.hash });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

**Key Points - Current Admin Resolution:**
- ❌ Admin manually decides winner
- ❌ Centralized trust (admin can manipulate)
- ❌ Server wallet pays gas
- ❌ No verification of real-world data

---

### New Chainlink Resolution Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NEW: CHAINLINK RESOLUTION                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Option A: PRICE FEED (Fully On-Chain)                                  │
│  ─────────────────────────────────────                                  │
│                                                                         │
│  Chainlink Automation (checks every 30s)                                │
│       │                                                                 │
│       ▼                                                                 │
│  ChainlinkResolver.checkUpkeep()                                        │
│  └── Is deadline passed? Is market unresolved?                          │
│       │ YES                                                             │
│       ▼                                                                 │
│  ChainlinkResolver.performUpkeep()                                      │
│  ├── Read Chainlink Price Feed                                          │
│  ├── Compare price vs threshold                                         │
│  ├── Determine winner (YES/NO)                                          │
│  └── Call reportPayouts()                                               │
│       │                                                                 │
│       ▼                                                                 │
│  Market Resolved ✓ (NO API SERVER INVOLVED!)                            │
│       │                                                                 │
│       ▼                                                                 │
│  API Server (webhook/event listener)                                    │
│  └── Update database on PayoutReported event                            │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Option B: CHAINLINK FUNCTIONS (API-Based)                              │
│  ─────────────────────────────────────────                              │
│                                                                         │
│  Frontend or Automation triggers                                        │
│       │                                                                 │
│       ▼                                                                 │
│  POST /api/resolution/trigger (lightweight)                             │
│       │                                                                 │
│       ▼                                                                 │
│  API Server (NEW: resolution.ts)                                        │
│  └── Just triggers contract, doesn't decide winner                      │
│       │                                                                 │
│       ▼                                                                 │
│  ChainlinkResolver.requestResolution()                                  │
│       │                                                                 │
│       ▼                                                                 │
│  Chainlink Functions DON                                                │
│  ├── Execute JavaScript code                                            │
│  ├── Fetch real API (ESPN, AP News, etc.)                               │
│  └── Return result on-chain                                             │
│       │                                                                 │
│       ▼                                                                 │
│  ChainlinkResolver.fulfillRequest() [callback]                          │
│  └── Receive result → reportPayouts()                                   │
│       │                                                                 │
│       ▼                                                                 │
│  Market Resolved ✓                                                      │
│       │                                                                 │
│       ▼                                                                 │
│  API Server (event listener)                                            │
│  └── Update database on PayoutReported event                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### New API Server Code (resolution.ts)

```typescript
// api/src/routes/resolution.ts - NEW CHAINLINK IMPLEMENTATION

import express from 'express';
import { ethers } from 'ethers';
import { PrismaClient } from '@prisma/client';
import { getBlockchainConfig } from '../config/blockchain.js';

const prisma = new PrismaClient();
export const resolutionRoutes = express.Router();

/**
 * POST /api/resolution/trigger
 *
 * NEW: Triggers Chainlink resolution - does NOT decide winner
 * The contract reads real data from Chainlink and decides automatically
 */
resolutionRoutes.post('/trigger', async (req, res) => {
  try {
    const { marketId } = req.body;

    // 1. Fetch market
    const market = await prisma.market.findUnique({
      where: { id: marketId },
      include: { outcomes: true }
    });

    if (!market) {
      return res.status(404).json({ error: 'Market not found' });
    }

    // 2. Check if market is ready for resolution
    if (market.status === 'RESOLVED') {
      return res.status(400).json({ error: 'Already resolved' });
    }

    if (new Date() < market.endDate) {
      return res.status(400).json({ error: 'Market deadline not reached' });
    }

    // 3. Connect to blockchain
    const config = getBlockchainConfig();
    const provider = new ethers.JsonRpcProvider(config.rpcUrl);

    // For triggering, user's wallet pays gas (or use relayer)
    // This is just a trigger - not the actual resolution decision

    // Option A: Price Feed - can be called by anyone
    if (market.resolutionType === 'PRICE_FEED') {
      const resolverABI = [
        "function resolveMarket(bytes32 questionId) external"
      ];

      const resolver = new ethers.Contract(
        config.chainlinkResolverAddress,
        resolverABI,
        provider
      );

      // Return the calldata for frontend to execute
      const calldata = resolver.interface.encodeFunctionData(
        'resolveMarket',
        [market.questionId]
      );

      return res.json({
        success: true,
        action: 'CALL_CONTRACT',
        contractAddress: config.chainlinkResolverAddress,
        calldata: calldata,
        message: 'Frontend should call this contract with user wallet'
      });
    }

    // Option B: Chainlink Functions
    if (market.resolutionType === 'CHAINLINK_FUNCTIONS') {
      const resolverABI = [
        "function requestResolution(bytes32 questionId) external returns (bytes32)"
      ];

      const resolver = new ethers.Contract(
        config.chainlinkResolverAddress,
        resolverABI,
        provider
      );

      const calldata = resolver.interface.encodeFunctionData(
        'requestResolution',
        [market.questionId]
      );

      return res.json({
        success: true,
        action: 'CALL_CONTRACT',
        contractAddress: config.chainlinkResolverAddress,
        calldata: calldata,
        message: 'Chainlink Functions will fetch real data and resolve'
      });
    }

    // Fallback to manual if no Chainlink config
    return res.status(400).json({
      error: 'Market not configured for Chainlink resolution',
      resolutionType: market.resolutionType
    });

  } catch (error: any) {
    console.error('Resolution trigger error:', error);
    res.status(500).json({ error: error.message });
  }
});

/**
 * GET /api/resolution/status/:marketId
 *
 * Check resolution status - reads from contract
 */
resolutionRoutes.get('/status/:marketId', async (req, res) => {
  try {
    const { marketId } = req.params;

    const market = await prisma.market.findUnique({
      where: { id: marketId }
    });

    if (!market) {
      return res.status(404).json({ error: 'Market not found' });
    }

    const config = getBlockchainConfig();
    const provider = new ethers.JsonRpcProvider(config.rpcUrl);

    // Check on-chain status
    const resolverABI = [
      "function markets(bytes32) view returns (bytes32 questionId, uint256 threshold, bool resolveAbove, uint256 deadline, bool resolved)"
    ];

    const resolver = new ethers.Contract(
      config.chainlinkResolverAddress,
      resolverABI,
      provider
    );

    const onChainMarket = await resolver.markets(market.questionId);

    res.json({
      marketId: market.id,
      databaseStatus: market.status,
      onChainResolved: onChainMarket.resolved,
      deadline: new Date(Number(onChainMarket.deadline) * 1000),
      resolutionType: market.resolutionType
    });

  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});
```

### Event Listener for Database Sync

```typescript
// api/src/services/ChainlinkEventListener.ts

import { ethers } from 'ethers';
import { PrismaClient } from '@prisma/client';
import { getBlockchainConfig } from '../config/blockchain.js';

const prisma = new PrismaClient();

/**
 * Listen for on-chain resolution events and sync database
 * This replaces the manual database update in admin resolution
 */
export function startChainlinkEventListener() {
  const config = getBlockchainConfig();

  // Use WebSocket for real-time events
  const provider = new ethers.WebSocketProvider(config.wsRpcUrl);

  const ctfABI = [
    "event PayoutRedemption(address indexed redeemer, bytes32 indexed conditionId, uint256[] payouts)"
  ];

  const conditionalTokens = new ethers.Contract(
    config.conditionalTokensAddress,
    ctfABI,
    provider
  );

  // Listen for resolution events
  conditionalTokens.on('PayoutRedemption', async (redeemer, conditionId, payouts, event) => {
    console.log(`🔔 PayoutRedemption detected: ${conditionId}`);

    try {
      // Find market by conditionId (need to store this mapping)
      const market = await prisma.market.findFirst({
        where: { conditionId: conditionId },
        include: { outcomes: true }
      });

      if (!market) {
        console.log(`   Market not found for conditionId: ${conditionId}`);
        return;
      }

      if (market.status === 'RESOLVED') {
        console.log(`   Market already resolved in database`);
        return;
      }

      // Determine winner from payouts
      const winningIndex = payouts.findIndex((p: bigint) => p > 0n);
      const winningOutcome = market.outcomes[winningIndex];

      // Update database
      await prisma.market.update({
        where: { id: market.id },
        data: {
          status: 'RESOLVED',
          resolvedOutcomeId: winningOutcome.id,
          resolvedAt: new Date(),
          resolutionTxHash: event.transactionHash
        }
      });

      await prisma.outcome.update({
        where: { id: winningOutcome.id },
        data: { isWinningOutcome: true }
      });

      // Cancel unfilled orders
      await prisma.order.updateMany({
        where: {
          marketId: market.id,
          status: { in: ['OPEN', 'PARTIALLY_FILLED'] }
        },
        data: { status: 'CANCELLED' }
      });

      console.log(`   ✅ Database synced: ${market.title} resolved to ${winningOutcome.title}`);

    } catch (error) {
      console.error('Error syncing resolution:', error);
    }
  });

  console.log('🎧 Chainlink event listener started');
}
```

---

### Side-by-Side Comparison

| Aspect | Admin Resolution | Chainlink Resolution |
|--------|------------------|---------------------|
| **Who decides winner?** | Admin (human) | Chainlink (oracle/data) |
| **Trust model** | Centralized (trust admin) | Decentralized (trust Chainlink) |
| **Data source** | Admin's judgment | Real data (price feeds, APIs) |
| **Gas payment** | Server wallet | User wallet or Automation |
| **API server role** | Decides & executes | Just triggers (or just listens) |
| **Database update** | Direct in API | Event listener syncs |
| **Manipulation risk** | High (admin can cheat) | Low (cryptographic proofs) |
| **Speed** | Instant | Seconds (price) to minutes (Functions) |
| **Cost** | Gas only | Gas + LINK tokens |

---

### Contract Flow Comparison

#### Current: Admin Resolution Contract Flow

```
┌──────────────────────────────────────────────────────────────┐
│  CURRENT CONTRACT FLOW                                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Server Wallet                                               │
│       │                                                      │
│       │ calls                                                │
│       ▼                                                      │
│  ResolutionOracle.adminFinalizeResolution()                  │
│  ├── require(msg.sender == owner)  ← Only admin can call     │
│  ├── payouts = [100, 0] or [0, 100]  ← Admin decides         │
│  └── calls ↓                                                 │
│       │                                                      │
│       ▼                                                      │
│  ConditionalTokens.reportPayouts(questionId, payouts)        │
│  ├── conditionId = keccak256(oracle, questionId, slots)      │
│  ├── require(payoutNumerators[conditionId].length == 0)      │
│  ├── Store payouts                                           │
│  └── emit ConditionResolution(conditionId, oracle, payouts)  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```solidity
// Current: ResolutionOracle.sol (simplified)

contract ResolutionOracle {
    IConditionalTokens public ctf;
    address public owner;

    function adminFinalizeResolution(
        bytes32 questionId,
        uint256 outcomeSlotCount,
        uint256[] calldata payouts
    ) external {
        require(msg.sender == owner, "Only admin");  // ❌ Centralized

        // Admin manually provides payouts - no verification
        ctf.reportPayouts(questionId, payouts);      // ❌ No data verification
    }
}
```

#### New: Chainlink Resolution Contract Flow

```
┌──────────────────────────────────────────────────────────────┐
│  CHAINLINK CONTRACT FLOW (Price Feed)                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Anyone (or Chainlink Automation)                            │
│       │                                                      │
│       │ calls                                                │
│       ▼                                                      │
│  ChainlinkResolver.resolveMarket(questionId)                 │
│  ├── require(block.timestamp >= deadline)                    │
│  ├── require(!markets[questionId].resolved)                  │
│  │                                                           │
│  │   // READ FROM CHAINLINK - NOT ADMIN                      │
│  ├── (,int256 price,,,) = priceFeed.latestRoundData() ← ✅  │
│  │                                                           │
│  │   // AUTOMATIC DECISION BASED ON DATA                     │
│  ├── if (price > threshold) payouts = [1, 0]                 │
│  │   else payouts = [0, 1]                                   │
│  │                                                           │
│  └── calls ↓                                                 │
│       │                                                      │
│       ▼                                                      │
│  ConditionalTokens.reportPayouts(questionId, payouts)        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```solidity
// New: ChainlinkResolver.sol (Price Feed)

contract ChainlinkResolver {
    IConditionalTokens public ctf;
    AggregatorV3Interface public priceFeed;

    struct PriceMarket {
        bytes32 questionId;
        uint256 threshold;
        bool resolveAbove;
        uint256 deadline;
        bool resolved;
    }

    mapping(bytes32 => PriceMarket) public markets;

    function resolveMarket(bytes32 questionId) external {
        PriceMarket storage market = markets[questionId];
        require(block.timestamp >= market.deadline, "Too early");
        require(!market.resolved, "Already resolved");

        // ✅ Read from Chainlink - decentralized data source
        (, int256 price, , uint256 updatedAt, ) = priceFeed.latestRoundData();

        // ✅ Verify data freshness
        require(block.timestamp - updatedAt < 1 hours, "Stale data");

        // ✅ Automatic decision based on real data
        bool yesWins = market.resolveAbove
            ? price > int256(market.threshold)
            : price < int256(market.threshold);

        uint256[] memory payouts = new uint256[](2);
        payouts[0] = yesWins ? 1 : 0;
        payouts[1] = yesWins ? 0 : 1;

        // ✅ Anyone can call - not just admin
        ctf.reportPayouts(questionId, payouts);
        market.resolved = true;
    }
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  CHAINLINK CONTRACT FLOW (Functions - Event Based)           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Step 1: Request                                             │
│  ─────────────────                                           │
│  Anyone calls requestResolution(questionId)                  │
│       │                                                      │
│       ▼                                                      │
│  ChainlinkResolver                                           │
│  ├── Build Functions request                                 │
│  ├── _sendRequest(source, args, subscriptionId)              │
│  └── Store requestId → questionId mapping                    │
│       │                                                      │
│       ▼                                                      │
│  Chainlink DON (Decentralized Oracle Network)                │
│  ├── Multiple nodes execute JavaScript                       │
│  ├── Fetch real API (sports, elections, etc.)                │
│  ├── Reach consensus on result                               │
│  └── Submit result on-chain                                  │
│                                                              │
│  Step 2: Fulfill (callback)                                  │
│  ─────────────────────────                                   │
│       │                                                      │
│       ▼                                                      │
│  ChainlinkResolver.fulfillRequest(requestId, response, err)  │
│  ├── Decode response (0 = NO wins, 1 = YES wins)             │
│  ├── Build payouts array                                     │
│  └── ctf.reportPayouts(questionId, payouts)                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```solidity
// New: ChainlinkResolver.sol (Functions - Event Based)

contract ChainlinkResolver is FunctionsClient {
    IConditionalTokens public ctf;

    mapping(bytes32 => bytes32) public requestToQuestion;  // requestId → questionId
    mapping(bytes32 => EventMarket) public eventMarkets;

    // Step 1: Anyone can trigger resolution request
    function requestResolution(bytes32 questionId) external returns (bytes32) {
        EventMarket storage market = eventMarkets[questionId];
        require(block.timestamp >= market.deadline, "Too early");
        require(!market.resolved, "Already resolved");

        // Build Chainlink Functions request
        FunctionsRequest.Request memory req;
        req.initializeRequestForInlineJavaScript(sourceCode);

        string[] memory args = new string[](2);
        args[0] = market.apiEndpoint;   // e.g., "https://api.espn.com/..."
        args[1] = market.jsonPath;       // e.g., "game.winner"
        req.setArgs(args);

        // ✅ Chainlink nodes will fetch real data
        bytes32 requestId = _sendRequest(
            req.encodeCBOR(),
            subscriptionId,
            gasLimit,
            donId
        );

        requestToQuestion[requestId] = questionId;
        return requestId;
    }

    // Step 2: Chainlink calls back with result
    function fulfillRequest(
        bytes32 requestId,
        bytes memory response,
        bytes memory err
    ) internal override {
        require(err.length == 0, "Request failed");

        bytes32 questionId = requestToQuestion[requestId];
        EventMarket storage market = eventMarkets[questionId];

        // ✅ Result comes from Chainlink, not admin
        uint256 result = abi.decode(response, (uint256));

        uint256[] memory payouts = new uint256[](2);
        payouts[0] = result == 1 ? 1 : 0;  // YES wins if result == 1
        payouts[1] = result == 1 ? 0 : 1;

        ctf.reportPayouts(questionId, payouts);
        market.resolved = true;
    }
}
```

---

### Migration Path: Admin → Chainlink

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MIGRATION STRATEGY                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Phase 1: Keep Both (Hybrid)                                            │
│  ───────────────────────────                                            │
│                                                                         │
│  ┌─────────────────────────────────────────┐                            │
│  │ Market Creation                         │                            │
│  ├─────────────────────────────────────────┤                            │
│  │ resolutionType: 'MANUAL' | 'PRICE_FEED' │                            │
│  │                | 'CHAINLINK_FUNCTIONS'  │                            │
│  └─────────────────────────────────────────┘                            │
│                                                                         │
│  if (resolutionType === 'MANUAL') {                                     │
│      // Use existing admin.ts                                           │
│      POST /api/admin/resolve                                            │
│  } else {                                                               │
│      // Use new Chainlink flow                                          │
│      POST /api/resolution/trigger                                       │
│  }                                                                      │
│                                                                         │
│  Phase 2: Default to Chainlink                                          │
│  ─────────────────────────────                                          │
│                                                                         │
│  - New markets default to Chainlink                                     │
│  - Manual only for edge cases                                           │
│  - Admin can override if Chainlink fails                                │
│                                                                         │
│  Phase 3: Fully Decentralized                                           │
│  ────────────────────────────                                           │
│                                                                         │
│  - Remove admin resolution                                              │
│  - All markets use Chainlink                                            │
│  - Dispute mechanism if needed                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Market Type | Chainlink Product | Cost | Complexity |
|-------------|-------------------|------|------------|
| Price-based ("Will BTC > $X?") | Data Feeds | FREE | Low |
| Time-based auto-resolve | Automation | ~$1/month | Medium |
| Event-based (sports, elections) | Functions | ~$2.50/request | High |

**Recommended Starting Point:**
1. Start with Price Feed markets (free, simple)
2. Add Automation for auto-resolve
3. Expand to Functions for event markets as needed
