# Quick Start Guide - Local Testing

## Setup Instructions

### 1. Clone the Repository (if you haven't already)
```bash
git clone https://github.com/JPmarsxxxi/Polymarket-Arbitrage.git
cd Polymarket-Arbitrage
```

### 2. Create Virtual Environment (Recommended)
```bash
# Create virtual environment
python3 -m venv venv

# Activate it
# On Mac/Linux:
source venv/bin/activate
# On Windows:
venv\Scripts\activate
```

### 3. Install Dependencies
```bash
pip install -r requirements.txt
```

### 4. Configure Environment Variables
Your `.env` file is already configured with:
- ✅ Alchemy RPC URL
- ✅ Wallet Address

**You're ready to test!**

### 5. Run the Testing Notebook
```bash
# Start Jupyter
jupyter notebook

# Or use JupyterLab
jupyter lab
```

Then open: `notebooks/02_complete_testing.ipynb`

## What the Notebook Will Test

**Section 1:** Environment setup ✅
**Section 2:** Load .env configuration ✅
**Section 3:** Connect to Polygon via Alchemy 🌐
**Section 4:** Check your USDC balance 💰
**Section 5:** Summary & next steps 📋

## Expected Results

After running all cells, you should see:
- ✅ Python 3.11+ confirmed
- ✅ All packages imported successfully
- ✅ Configuration loaded from .env
- ✅ Connected to Polygon (Chain ID 137)
- 📊 Your wallet balances (MATIC and USDC)

## Troubleshooting

### If imports fail:
```bash
# Make sure virtual environment is activated
pip install -r requirements.txt
```

### If .env not loading:
The notebook should find it automatically if you run from the project directory.

### If Alchemy connection fails:
- Check your API key is correct in `.env`
- Verify your Alchemy account is active
- Make sure you're using the Polygon endpoint (not Ethereum)

## Next Steps After Testing

Once the notebook runs successfully:
1. ✅ Generate Polymarket API credentials
2. ✅ Test WebSocket connection
3. ✅ Fetch real market data
4. ✅ Build arbitrage detection logic
5. ✅ Test order placement (dry run)

---

**Ready to proceed!** 🚀

Let me know which section fails (if any) and I'll help debug!
