# Liquidity Provision UI Concept ("Mint & Sell")

This document outlines the User Interface for the "Mint & Sell" mechanism, which allows users (Creators or Market Makers) to provide liquidity by depositing USDC, minting outcome tokens, and immediately placing sell orders on the book.

## 1. Entry Point
**Where:** "Market Detail Page" -> "Liquidity" Tab (or a "Provide Liquidity" button near the Buy/Sell widget).

## 2. The Modal / Widget Design

The interface is designed to be simple, abstracting away the complex "Mint -> Split -> Sell" logic into a single "Deposit & Set Odds" flow.

### UI Layout Mockup

```tsx
// This is a conceptual representation of the UI component
import { useState } from 'react';

export const AddLiquidityWidget = () => {
  const [depositAmount, setDepositAmount] = useState(1000); // USDC
  const [yesProbability, setYesProbability] = useState(60); // 60%

  // Derived values
  const noProbability = 100 - yesProbability;
  const yesPrice = (yesProbability / 100).toFixed(2);
  const noPrice = (noProbability / 100).toFixed(2);
  
  return (
    <div className="p-6 bg-gray-900 rounded-xl border border-gray-700 w-[480px]">
      <h2 className="text-xl font-bold text-white mb-4">Add Liquidity</h2>
      
      {/* 1. Deposit Section */}
      <div className="mb-6">
        <label className="text-gray-400 text-sm">Deposit Amount (USDC)</label>
        <div className="flex items-center mt-2 bg-gray-800 rounded-lg p-3 border border-gray-600">
          <input 
            type="number" 
            value={depositAmount}
            onChange={(e) => setDepositAmount(Number(e.target.value))}
            className="bg-transparent text-white text-2xl w-full focus:outline-none"
          />
          <span className="text-blue-400 font-bold">USDC</span>
        </div>
        <p className="text-xs text-gray-500 mt-1">Balance: $5,420.00</p>
      </div>

      {/* 2. Odds Setting Section */}
      <div className="mb-6">
        <label className="text-gray-400 text-sm mb-2 block">Set Initial Odds</label>
        
        {/* Visual Slider */}
        <div className="flex items-center justify-between mb-2">
          <span className="text-green-400 font-bold">YES {yesProbability}%</span>
          <span className="text-red-400 font-bold">NO {noProbability}%</span>
        </div>
        <input 
          type="range" 
          min="1" max="99" 
          value={yesProbability} 
          onChange={(e) => setYesProbability(Number(e.target.value))}
          className="w-full h-2 bg-gray-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
        />
        <p className="text-xs text-gray-500 mt-2 text-center">
          Drag to adjust the starting price for YES and NO.
        </p>
      </div>

      {/* 3. Action Preview (The "Under the Hood" visualization) */}
      <div className="bg-gray-800/50 rounded-lg p-4 mb-6 border border-gray-700 border-dashed">
        <h3 className="text-xs font-semibold text-gray-400 uppercase tracking-wider mb-3">
          Action Preview
        </h3>
        
        {/* Step A: Mint */}
        <div className="flex justify-between items-center mb-2 text-sm">
          <span className="text-gray-300">1. Mint Tokens</span>
          <span className="text-white">
            {depositAmount} <span className="text-green-400">YES</span> + {depositAmount} <span className="text-red-400">NO</span>
          </span>
        </div>

        {/* Step B: Sell Orders */}
        <div className="space-y-2">
          <div className="flex justify-between items-center text-sm bg-gray-800 p-2 rounded">
            <span className="text-gray-300">2. Place Sell Order (YES)</span>
            <span className="text-white font-mono">
              Sell {depositAmount} @ <span className="text-green-400">${yesPrice}</span>
            </span>
          </div>
          <div className="flex justify-between items-center text-sm bg-gray-800 p-2 rounded">
            <span className="text-gray-300">3. Place Sell Order (NO)</span>
            <span className="text-white font-mono">
              Sell {depositAmount} @ <span className="text-red-400">${noPrice}</span>
            </span>
          </div>
        </div>
      </div>

      {/* 4. Submit Button */}
      <button className="w-full bg-blue-600 hover:bg-blue-500 text-white font-bold py-4 rounded-lg transition-colors shadow-lg shadow-blue-900/20">
        Mint & Provide Liquidity
      </button>
      
      <p className="text-xs text-gray-500 mt-4 text-center">
        You can cancel these orders and withdraw your USDC at any time if they are not filled.
      </p>
    </div>
  );
};
```

## 3. Key UX Features

1.  **Single Input for Deposit**: Users don't need to manually calculate how many YES/NO tokens they get. 1 USDC = 1 YES + 1 NO.
2.  **Probability Slider**: Instead of typing prices manually (which can be confusing), users set the "Probability" (Odds). The system automatically calculates the limit price ($0.60 for 60%).
3.  **Visual Feedback**: The "Action Preview" clearly shows that *two* separate sell orders will be placed.
4.  **Safety Note**: Reminds users that this is reversible (Cancel -> Merge -> Withdraw) *unless* someone trades against them.

## 4. Backend / Contract Flow

When the user clicks "Mint & Provide Liquidity":

1.  **Approve USDC**: User approves the Proxy contract to spend `depositAmount` USDC.
2.  **Batch Transaction**:
    *   `ctfExchange.splitPosition(...)`: Mints `depositAmount` of YES and NO.
    *   `ctfExchange.createOrder(...)`: Creates a Limit Sell Order for YES at `yesPrice`.
    *   `ctfExchange.createOrder(...)`: Creates a Limit Sell Order for NO at `noPrice`.

*Note: This requires the "Batch Creation" feature (or multiple signatures) discussed in the Batch Plan.*
