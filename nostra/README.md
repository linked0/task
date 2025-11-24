# World Cup Market Simulation - Jupyter Notebook

## ✅ Setup Complete

All required packages have been installed via `uv`.

## Running the Notebook

The notebook is already running! You started it with:
```bash
uv run jupyter lab
```

## Installed Packages

- ✅ `jupyterlab` - Jupyter Lab interface
- ✅ `pandas` - Data manipulation
- ✅ `numpy` - Numerical computing
- ✅ `matplotlib` - Plotting
- ✅ `seaborn` - Statistical visualization
- ✅ `web3` - Ethereum blockchain interaction

## Usage Notes

### ⚠️ Important: Remove the pip install cell

The first code cell in the notebook tries to run `!pip install web3` which will fail because:
1. We're using `uv` for package management (not pip)
2. All packages are already installed

**Action**: In Jupyter Lab, either:
- **Skip** that first cell (don't run it)
- **Delete** that cell entirely
- **Convert** it to a markdown cell

### Running the Simulation

After skipping/removing the pip cell, run all remaining cells in order. They should work perfectly now!

## What This Notebook Does

Simulates a **GROUPED_BINARY** prediction market for the 2026 FIFA World Cup:
- 5 separate YES/NO markets (Brazil, Argentina, France, Germany, Spain)
- 10 traders making bets
- Arbitrage detection
- Market resolution with fee distribution
- Web3 integration for token balances

## Troubleshooting

If you need to restart Jupyter Lab:
```bash
# Stop the current session (Ctrl+C in terminal)
# Then restart:
uv run jupyter lab
```
