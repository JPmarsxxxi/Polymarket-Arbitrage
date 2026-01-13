# Polymarket Arbitrage Bot - Project Context

**Project Start Date:** January 12, 2026  
**Developer:** Emmanuel (MSc Data Science, University of Hertfordshire)  
**Current Phase:** Phase 1 - Notebook Development & Testing

---

## Project Overview

Building an automated arbitrage trading bot for Polymarket prediction markets that exploits the "$1.00 full-set invariant" - where buying both YES and NO outcome tokens for less than $1.00 combined creates a risk-free profit opportunity after merging tokens back to USDC.

### Core Strategy
- **Arbitrage Type:** Long arbitrage (buy both sides when cost < $1.00)
- **Profit Mechanism:** Buy YES + NO tokens → Merge on-chain → Receive $1.00 USDC
- **Expected Edge:** 5-50 basis points per trade (after fees)
- **Target Frequency:** 5-30 arbitrage opportunities per day

---

## Current Constraints & Budget

### Financial Constraints
- **Infrastructure Budget:** $0/month (100% free tier)
- **Starting Capital:** $50-100 USDC for testing
- **Scale-up Capital:** $200-500 after proven profitability
- **No paid services** until generating consistent profit ($500+/month)

### Technical Constraints
- **RPC Provider:** Alchemy free tier (300M requests/month - more than sufficient)
- **Server:** Local laptop initially, migrate to Oracle Cloud free tier if needed
- **Latency Target:** <1 second execution (competitive with 80% of retail traders)
- **Markets Monitored:** 1-5 high-volume markets initially

### Success Criteria for Upgrade Decision
- Consistent profitability: $5-20/day for 2+ weeks
- Capital grown to $500-1000 through reinvestment
- Clear bottleneck from free tier latency
- **Then:** Upgrade to paid VPS ($20-40/month) + Alchemy paid ($49/month)

---

## Architecture & Tech Stack

### Development Approach
```
Phase 1: Jupyter Notebook Testing (Current - Week 1)
├─ Test WebSocket connections
├─ Verify order book parsing
├─ Calculate opportunities manually
├─ Test single order placement (dry run + small real trade)
├─ Validate fee calculations
└─ Measure baseline latency

Phase 2: Production Python Scripts (Week 2-3)
├─ Event-driven architecture
├─ Full error handling
├─ Structured logging
├─ 24/7 execution capability
└─ Inventory management

Phase 3: Optimization & Scaling (Month 2+)
├─ Multi-market scanning
├─ Advanced strategies (short arb, copy trading)
├─ Paid infrastructure migration
└─ Capital scaling to $2k-5k
```

### Tech Stack (100% Free)

**Core Dependencies:**
```python
# Official & Essential
py-clob-client          # Polymarket CLOB SDK
web3                    # Polygon blockchain interactions
websocket-client        # WebSocket streaming
python-dotenv           # Environment variables
asyncio                 # Async execution

# Speed Optimizations (Free)
uvloop                  # 2x faster event loop (vs asyncio)
orjson                  # 3x faster JSON parsing (vs json)
aiohttp                 # Async HTTP (faster than requests)

# Data & Analysis
pandas                  # Order book analysis (optional)
numpy                   # Calculations (optional)
```

**Infrastructure:**
- **Python Version:** 3.11 (fastest available)
- **RPC Endpoint:** Alchemy free tier (https://polygon-mainnet.g.alchemy.com/v2/YOUR_KEY)
- **WebSocket:** Polymarket CLOB (wss://ws-subscriptions-clob.polymarket.com)
- **Blockchain:** Polygon (low gas fees, ~$0.02-0.05 per merge)

---

## System Architecture

### Component Overview
```
┌─────────────────────────────────────────────────────────┐
│  STREAMING LAYER                                        │
│  ├─ WebSocket Client (RealtimeClient)                  │
│  │   └─ Subscribe to market channel (order books)      │
│  └─ Message Parser (orjson for speed)                  │
└─────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│  STRATEGY LAYER                                         │
│  ├─ Order Book State Management                        │
│  ├─ Opportunity Detection (_check_opportunity)         │
│  │   └─ Calculate: edge = 1.0 - (buy_yes + buy_no)     │
│  └─ Fee Model (CRITICAL)                               │
│      └─ total_cost = prices + maker_fee + taker_fee +  │
│                      gas_cost + slippage_buffer         │
└─────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│  EXECUTION LAYER                                        │
│  ├─ Trading Client (TradingClient)                     │
│  │   ├─ create_limit_order() - CLOB off-chain         │
│  │   ├─ get_order_status() - check fills              │
│  │   └─ cancel_order() - cancel unfilled              │
│  └─ Web3 Client (blockchain interactions)              │
│      ├─ approve_usdc() - one-time setup                │
│      ├─ merge() - YES + NO → $1.00 USDC (profit lock)  │
│      └─ get_balances() - check USDC/token holdings     │
└─────────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│  MONITORING & SAFETY                                    │
│  ├─ Structured Logging (JSON logs)                     │
│  ├─ Performance Metrics (latency, fills, PnL)          │
│  ├─ Risk Management                                    │
│  │   ├─ Min/max trade size enforcement                │
│  │   ├─ Execution cooldown (prevent spam)             │
│  │   └─ Inventory tracking (detect stuck positions)   │
│  └─ Alerts (merge failures, disconnections)            │
└─────────────────────────────────────────────────────────┘
```

---

## Critical Execution Flow

### Arbitrage Detection & Execution

**Step 1: Order Book Update (WebSocket)**
```python
# Triggered on every order book change
event: "book"
payload: {
    "asset_id": "0x123...",  # YES or NO token
    "bids": [[price, size], ...],  # Buy orders
    "asks": [[price, size], ...]   # Sell orders
}

# Update local state
if token == YES_TOKEN:
    self.orderbook["yes_asks"] = asks
    self.orderbook["yes_bids"] = bids
```

**Step 2: Opportunity Calculation**
```python
# Get best prices
yes_ask = orderbook["yes_asks"][0]["price"]  # Price to BUY YES
no_ask = orderbook["no_asks"][0]["price"]    # Price to BUY NO

# Calculate cost and edge
long_cost = yes_ask + no_ask
gross_edge = 1.0 - long_cost

# CRITICAL: Subtract ALL costs
maker_fee = 0.001  # 0.1% per side
taker_fee = 0.002  # 0.2% per side
total_fees = (maker_fee + taker_fee) * 2  # Both YES and NO orders
gas_cost_per_dollar = 0.05 / trade_size_usd  # Merge transaction
slippage_buffer = 0.002  # 0.2% safety margin

net_edge = gross_edge - total_fees - gas_cost_per_dollar - slippage_buffer

# Execute if profitable
if net_edge > profit_threshold:  # e.g., 0.01 = 1%
    execute_long_arb(yes_ask, no_ask, net_edge)
```

**Step 3: Parallel Order Placement**
```python
# CRITICAL: Both orders must be placed simultaneously
async def execute_long_arb(yes_price, no_price, edge):
    # Calculate size
    usd_budget = min(max_trade_usd, usdc_balance * 0.9)  # Keep 10% buffer
    size = usd_budget / (yes_price + no_price)
    
    # Place both orders in parallel
    yes_task = create_limit_order(YES_TOKEN, "BUY", yes_price, size)
    no_task = create_limit_order(NO_TOKEN, "BUY", no_price, size)
    
    yes_result, no_result = await asyncio.gather(yes_task, no_task)
    
    return yes_result, no_result
```

**Step 4: Fill Verification & Merge**
```python
# Wait for both orders to fill
async def wait_for_fills(yes_order_id, no_order_id, timeout=30):
    start_time = time.time()
    
    while time.time() - start_time < timeout:
        yes_status = await get_order_status(yes_order_id)
        no_status = await get_order_status(no_order_id)
        
        yes_filled = yes_status["size_matched"]
        no_filled = no_status["size_matched"]
        
        # Both filled?
        if yes_status["status"] == "MATCHED" and no_status["status"] == "MATCHED":
            # Merge the minimum (handle partial fills)
            merge_size = min(yes_filled, no_filled)
            
            # THIS IS WHERE PROFIT LOCKS IN
            tx_hash = await merge(condition_id, merge_size)
            await wait_for_confirmation(tx_hash)
            
            return {"success": True, "profit": merge_size * edge}
        
        await asyncio.sleep(0.5)  # Check every 500ms
    
    # Timeout: cancel unfilled orders
    if yes_status["status"] != "MATCHED":
        await cancel_order(yes_order_id)
    if no_status["status"] != "MATCHED":
        await cancel_order(no_order_id)
    
    return {"success": False, "reason": "timeout"}
```

---

## Fee Model (CRITICAL FOR PROFITABILITY)

### Complete Cost Breakdown

**Per-Trade Costs:**
```python
# Example with $100 trade size
trade_size_usd = 100.00

# 1. CLOB Trading Fees (varies by market)
maker_fee_pct = 0.001   # 0.1% if your order adds liquidity
taker_fee_pct = 0.002   # 0.2% if your order takes liquidity
# Assume worst case: both are taker
clob_fees = trade_size_usd * taker_fee_pct * 2  # $0.40 (both YES and NO)

# 2. Blockchain Gas (Polygon merge transaction)
gas_cost_usd = 0.05  # ~$0.05 for merge (varies with network)

# 3. Slippage (price moves between detection and execution)
slippage_bps = 20  # 20 basis points = 0.2%
slippage_cost = trade_size_usd * 0.002  # $0.20

# 4. Total Costs
total_costs = clob_fees + gas_cost_usd + slippage_cost
total_costs = 0.40 + 0.05 + 0.20 = $0.65

# 5. Required Edge
min_edge_pct = total_costs / trade_size_usd
min_edge_pct = 0.65 / 100 = 0.0065 = 0.65% = 65 bps

# So profit_threshold should be > 0.0065 (0.65%)
# Safe value: 0.01 (1%) to account for execution variance
```

**Dynamic Fee Calculation:**
```python
def calculate_min_edge(trade_size_usd, market_fees):
    """
    Calculate minimum edge needed for profitability.
    
    Args:
        trade_size_usd: Dollar amount of trade
        market_fees: Dict with 'maker' and 'taker' fee rates
    
    Returns:
        Minimum edge (as decimal, e.g., 0.01 = 1%)
    """
    # Worst case: both orders are taker
    clob_fees = trade_size_usd * market_fees['taker'] * 2
    
    # Gas cost (relatively fixed on Polygon)
    gas_cost = 0.05
    
    # Slippage (scales with trade size, worse for larger trades)
    slippage = trade_size_usd * 0.002
    
    # Total costs
    total = clob_fees + gas_cost + slippage
    
    # As percentage of trade
    min_edge = total / trade_size_usd
    
    # Add 30% safety buffer
    return min_edge * 1.3
```

---

## Risk Management & Safety

### Position Limits
```python
# Per-trade limits
MIN_TRADE_USD = 10.0      # Below this, fees eat profit
MAX_TRADE_USD = 100.0     # Cap exposure per arb
MAX_TOTAL_EXPOSURE = 500.0  # Total capital at risk

# Execution controls
EXECUTION_COOLDOWN = 5.0  # Seconds between arb attempts
ORDER_TIMEOUT = 30.0      # Cancel if not filled in 30s
MAX_DAILY_LOSSES = 50.0   # Stop bot if lose $50 in one day
```

### Error Handling Scenarios

**Scenario 1: One Leg Fills, Other Doesn't**
```python
# DANGER: You now have inventory risk
# Solution: Cancel unfilled leg immediately
if yes_filled and not no_filled:
    await cancel_order(no_order_id)
    # Now holding YES tokens - need exit strategy
    # Option A: Sell YES tokens immediately
    # Option B: Wait for event resolution (riskier)
```

**Scenario 2: Merge Transaction Fails**
```python
# DANGER: You have full set but didn't get USDC
# Solution: Retry merge with higher gas
try:
    tx_hash = await merge(condition_id, size)
    receipt = await w3.eth.wait_for_transaction_receipt(tx_hash)
except Exception as e:
    logger.critical("MERGE_FAILED", error=str(e), size=size)
    # Retry with 2x gas price
    tx_hash = await merge(condition_id, size, gas_multiplier=2.0)
```

**Scenario 3: WebSocket Disconnects Mid-Arb**
```python
# DANGER: Lose order book state, might miss fills
# Solution: Track all open orders, reconnect immediately
async def on_disconnect():
    logger.warning("websocket_disconnected")
    # Get all open orders
    open_orders = await get_open_orders()
    # Reconnect
    await ws_client.reconnect()
    # Resubscribe to markets
    await ws_client.subscribe_orderbook(token_ids)
    # Resume monitoring open orders
```

### Monitoring & Alerts

**Key Metrics to Track:**
```python
metrics = {
    # Performance
    "opportunities_detected": 0,     # How many arbs found
    "executions_attempted": 0,       # How many tried
    "executions_successful": 0,      # How many completed
    "execution_success_rate": 0.0,   # Should be >80%
    
    # Profitability
    "gross_profit_usdc": 0.0,        # Before fees
    "net_profit_usdc": 0.0,          # After all costs
    "avg_profit_per_trade": 0.0,     # Should be >$0.50
    
    # Latency
    "avg_detection_to_order_ms": 0,  # Should be <500ms
    "avg_order_to_fill_ms": 0,       # Should be <5000ms
    "avg_fill_to_merge_ms": 0,       # Should be <3000ms
    
    # Risk
    "current_inventory_usdc": 0.0,   # Stuck positions
    "failed_merges": 0,              # Should be 0
    "partial_fills": 0,              # Should be <20%
}
```

---

## Latency Optimization Strategy

### Free Tier Latency Budget
```
Target: <1 second total execution time

WebSocket message received          0ms
├─ Parse JSON (orjson)            +3ms
├─ Update order book              +2ms  
├─ Calculate opportunity          +2ms
├─ Check balances (cached)        +1ms
├─ Place both orders            +100ms  (Alchemy free tier RPC)
├─ Wait for order matching      +200ms
└─ Submit merge transaction     +100ms
─────────────────────────────────────
TOTAL: ~408ms ✅ Competitive

Blockchain confirmation:       +2500ms  (Polygon block time, can't optimize)
─────────────────────────────────────
END-TO-END: ~2.9 seconds
```

### Code-Level Optimizations (Free)

**1. Use Fast JSON Parser:**
```python
# ❌ Slow (standard library)
import json
data = json.loads(message)

# ✅ Fast (3x faster, free)
import orjson
data = orjson.loads(message)
```

**2. Use Fast Event Loop:**
```python
# ❌ Slow (standard asyncio)
import asyncio

# ✅ Fast (2x faster, free)
import uvloop
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
```

**3. Minimize Calculations in Hot Path:**
```python
# ❌ Slow (recalculate every message)
def check_opportunity(yes, no):
    fees = (0.001 + 0.002) * 2  # Recalculating
    edge = 1.0 - yes - no - fees
    
# ✅ Fast (precalculate once)
class Strategy:
    def __init__(self):
        self.total_fees = (0.001 + 0.002) * 2  # Calculate once
        
    def check_opportunity(self, yes, no):
        edge = 1.0 - yes - no - self.total_fees  # Just subtract
```

**4. Cache Frequently Accessed Data:**
```python
# ❌ Slow (RPC call every check)
balance = await get_usdc_balance()

# ✅ Fast (cache, update every 10s)
class BalanceCache:
    def __init__(self):
        self.balance = 0.0
        self.last_update = 0
        
    async def get_balance(self):
        if time.time() - self.last_update > 10:  # Update every 10s
            self.balance = await fetch_balance()
            self.last_update = time.time()
        return self.balance
```

---

## Market Selection Strategy

### Target Market Criteria
```python
# Focus on high-volume markets for better arb frequency
target_markets = {
    "high_priority": [
        # Crypto markets (volatile, fast-moving)
        "bitcoin-above-100k-by-2025",
        "ethereum-above-5k",
        
        # Political markets (high volume during elections)
        "2024-presidential-election",
        
        # Financial markets (professional traders = tight spreads but arbs exist)
        "fed-rate-cut-march-2025",
    ],
    
    "medium_priority": [
        # Sports markets (retail heavy, more arbs)
        "super-bowl-winner-2025",
        "nba-champion-2025",
    ],
    
    "low_priority": [
        # Niche markets (low volume = rare arbs)
        "will-ai-pass-turing-test",
    ]
}

# Selection criteria:
# 1. Daily volume > $100k (more opportunities)
# 2. Binary markets only (simpler to arb than categorical)
# 3. Resolution date > 7 days (avoid rush before resolution)
# 4. Spread typically > 2% (room for arb after fees)
```

---

## Development Roadmap

### Week 1: Notebook Testing (Current Phase)
**Goals:**
- [ ] Set up Alchemy free account + get RPC key
- [ ] Test WebSocket connection to Polymarket
- [ ] Fetch real order book data for 1 market
- [ ] Calculate arb opportunities manually
- [ ] Verify fee calculations with real market data
- [ ] Test single order placement (dry run mode)
- [ ] Test single order placement ($10 real trade)
- [ ] Measure actual latency from laptop
- [ ] Document baseline performance metrics

**Success Criteria:**
- WebSocket streaming works reliably
- Can detect arb opportunities in real-time
- Can place and cancel orders successfully
- Latency < 1 second
- Understand actual fee structure

### Week 2: Production Script Development
**Goals:**
- [ ] Build `core/websocket_client.py`
- [ ] Build `core/trading_client.py`
- [ ] Build `strategies/arbitrage_strategy.py`
- [ ] Implement complete execution flow
- [ ] Add error handling for all failure modes
- [ ] Build monitoring/logging system
- [ ] Test with $50-100 capital (10-20 trades)

**Success Criteria:**
- Bot runs autonomously for 24+ hours
- No crashes or unhandled exceptions
- Successfully executes 5+ arbs
- Net positive PnL after fees

### Week 3: Optimization & Risk Management
**Goals:**
- [ ] Add inventory tracking
- [ ] Implement position limits
- [ ] Add emergency stop mechanisms
- [ ] Optimize fee calculations (per-market)
- [ ] Add performance metrics dashboard
- [ ] Scale to monitoring 2-3 markets
- [ ] Test with $200-500 capital

**Success Criteria:**
- Execution success rate >80%
- Average profit per trade >$0.50
- No merge failures
- Consistent daily profitability ($5-20/day)

### Week 4: Deployment & Scaling Decision
**Goals:**
- [ ] Deploy to Oracle Cloud free VPS (if needed)
- [ ] Run 24/7 for 7 days
- [ ] Calculate actual ROI
- [ ] Document all edge cases encountered
- [ ] Make upgrade decision (stay free vs. paid)

**Success Criteria:**
- 7 days of profitable operation
- Capital grown to $300-700
- Clear understanding of bottlenecks
- Decision: scale with paid infra or optimize free tier further

---

## Configuration Management

### Environment Variables (.env)
```bash
# Blockchain & RPC
POLYGON_RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/YOUR_FREE_KEY
POLYGON_CHAIN_ID=137

# Wallet
WALLET_ADDRESS=0x...
WALLET_PRIVATE_KEY=0x...  # NEVER commit to git

# Polymarket API (generated from py_clob_client)
POLYMARKET_API_KEY=...
POLYMARKET_API_SECRET=...
POLYMARKET_API_PASSPHRASE=...
POLYMARKET_HOST=https://clob.polymarket.com

# Strategy Parameters
DRY_RUN_MODE=true  # Set to false for real trading
PROFIT_THRESHOLD=0.01  # 1% minimum edge
MIN_TRADE_USD=10.0
MAX_TRADE_USD=100.0
MAX_TOTAL_EXPOSURE=500.0
EXECUTION_COOLDOWN=5.0

# Target Markets (comma-separated condition IDs)
TARGET_MARKETS=0x123...,0x456...,0x789...

# Monitoring
LOG_LEVEL=INFO  # DEBUG for development
ENABLE_ALERTS=false  # Email/Telegram alerts (future)
```

### Config Class (config/settings.py)
```python
from pydantic import BaseSettings
from typing import List

class PolymarketSettings(BaseSettings):
    # Blockchain
    polygon_rpc_url: str
    polygon_chain_id: int = 137
    wallet_address: str
    wallet_private_key: str
    
    # Polymarket
    polymarket_api_key: str
    polymarket_api_secret: str
    polymarket_api_passphrase: str
    polymarket_host: str = "https://clob.polymarket.com"
    
    # Strategy
    dry_run_mode: bool = True
    profit_threshold: float = 0.01
    min_trade_usd: float = 10.0
    max_trade_usd: float = 100.0
    max_total_exposure: float = 500.0
    execution_cooldown: float = 5.0
    
    # Markets
    target_markets: List[str]
    
    # Monitoring
    log_level: str = "INFO"
    enable_alerts: bool = False
    
    class Config:
        env_file = ".env"
```

---

## Testing Strategy

### Unit Tests (Optional but Recommended)
```python
# tests/test_fee_calculation.py
def test_fee_calculation():
    strategy = ArbitrageStrategy(...)
    
    # Test case: $100 trade
    min_edge = strategy.calculate_min_edge(100.0)
    assert min_edge > 0.005  # Should be >0.5%
    assert min_edge < 0.015  # Should be <1.5%

# tests/test_opportunity_detection.py
def test_opportunity_detection():
    # Mock order book
    yes_ask = 0.52
    no_ask = 0.46
    
    # Should detect (cost = 0.98 < 1.00)
    is_arb = check_opportunity(yes_ask, no_ask)
    assert is_arb == True
    
    # Should NOT detect (cost = 0.99, edge too small after fees)
    is_arb = check_opportunity(0.50, 0.49)
    assert is_arb == False
```

### Integration Tests
```python
# tests/test_websocket_integration.py
async def test_websocket_connection():
    """Test that WebSocket connects and receives messages"""
    client = RealtimeClient()
    await client.connect()
    
    # Should receive message within 5 seconds
    message = await asyncio.wait_for(client.receive(), timeout=5.0)
    assert message is not None
    assert "event_type" in message

# tests/test_order_placement.py
async def test_order_placement_dry_run():
    """Test order placement in dry run mode"""
    trading_client = TradingClient(dry_run=True)
    
    result = await trading_client.create_limit_order(
        token_id="0x123...",
        side="BUY",
        price=0.52,
        size=10.0
    )
    
    assert result["success"] == True
    assert "order_id" in result
```

### Manual Testing Checklist
- [ ] WebSocket stays connected for 1+ hour
- [ ] Order book updates correctly on every message
- [ ] Arb detection triggers on real opportunities
- [ ] Orders are placed with correct prices
- [ ] Orders can be canceled successfully
- [ ] Merge transaction completes successfully
- [ ] Balance updates after merge
- [ ] Bot recovers from WebSocket disconnect
- [ ] Bot handles insufficient balance gracefully
- [ ] Bot stops trading after hitting loss limit

---

## Known Risks & Mitigation

### Risk 1: Inventory Risk (One Leg Fills)
**Scenario:** YES order fills, NO order doesn't  
**Impact:** Holding YES tokens = exposed to event outcome  
**Mitigation:**
- Order timeout (30s max)
- Immediate cancel of unfilled leg
- Emergency exit strategy (sell position at market)

### Risk 2: Merge Transaction Failure
**Scenario:** Have full set but merge fails  
**Impact:** Capital locked in tokens instead of USDC  
**Mitigation:**
- Retry with higher gas
- Alert + manual intervention
- Keep 10% USDC buffer for gas

### Risk 3: Fee Miscalculation
**Scenario:** Actual fees higher than estimated  
**Impact:** Negative PnL despite "profitable" arbs  
**Mitigation:**
- Fetch real fee structure per market
- Conservative profit threshold (1% vs 0.5%)
- Track actual vs expected profit
- Adjust threshold based on empirical data

### Risk 4: Slippage on Large Orders
**Scenario:** Order moves price before fill  
**Impact:** Pay worse price than expected  
**Mitigation:**
- Small trade sizes initially ($10-100)
- Check order book depth before execution
- Limit order (not market order)

### Risk 5: Smart Contract Risk
**Scenario:** CTF contract bug or exploit  
**Impact:** Loss of funds in merge  
**Mitigation:**
- Start with small capital ($50-100)
- Use audited contracts only (Polymarket = ChainSecurity audited)
- Never hold positions >24 hours

### Risk 6: Market Manipulation
**Scenario:** Fake arb opportunity (bait)  
**Impact:** Order fills at bad price  
**Mitigation:**
- Verify both sides before execution
- Cooldown between executions
- Monitor for unusual patterns

---

## Success Metrics & KPIs

### Daily Metrics
```python
daily_report = {
    # Opportunity metrics
    "opportunities_detected": 0,
    "opportunities_per_hour": 0.0,
    
    # Execution metrics
    "executions_attempted": 0,
    "executions_successful": 0,
    "execution_rate": 0.0,  # Should be >80%
    
    # Profitability
    "gross_profit": 0.0,
    "total_fees_paid": 0.0,
    "net_profit": 0.0,  # Should be positive
    "roi_percent": 0.0,  # Should be >1% per day
    
    # Performance
    "avg_latency_ms": 0,  # Should be <1000ms
    "failed_merges": 0,  # Should be 0
    "partial_fills": 0,
    
    # Risk
    "max_drawdown": 0.0,  # Should be <5%
    "inventory_at_eod": 0.0,  # Should be 0
}
```

### Weekly Review Questions
1. Is net profit consistently positive? (If not, why?)
2. Is execution rate >80%? (If not, latency issue?)
3. Are fees as expected? (Adjust threshold if higher)
4. Any merge failures? (Critical issue if yes)
5. Ready to scale capital? (If profitable 5/7 days)

---

## Next Steps (Immediate Actions)

### Action Items for Week 1
1. **Set up Alchemy account** (5 minutes)
   - Go to alchemy.com
   - Create free account
   - Create app on Polygon network
   - Copy RPC URL to .env file

2. **Set up Polygon wallet** (10 minutes)
   - Install MetaMask or use existing wallet
   - Switch to Polygon network
   - Get wallet address and private key
   - Transfer $50-100 USDC to Polygon

3. **Create Jupyter notebook** (30 minutes)
   - Install dependencies
   - Test WebSocket connection
   - Print live order book data

4. **Test single API call** (20 minutes)
   - Fetch market metadata
   - Parse token IDs
   - Get current balances

5. **Calculate first arb opportunity** (60 minutes)
   - Stream order book updates
   - Calculate edge
   - Log when opportunity appears

**Total time to first working prototype: ~2 hours**

---

## Resources & References

### Official Documentation
- Polymarket CLOB API: https://docs.polymarket.com/
- py-clob-client: https://github.com/Polymarket/py-clob-client
- Polygon RPC: https://docs.polygon.technology/
- Web3.py: https://web3py.readthedocs.io/

### Code References
- Original blog post: Jakub's Polymarket Trading Framework
- NautilusTrader Polymarket adapter: https://nautilustrader.io/docs/integrations/polymarket/

### Learning Resources
- Prediction market mechanics
- Statistical arbitrage concepts
- Event-driven architecture patterns
- Async Python best practices

---

## Change Log

**2026-01-12:**
- Initial project setup
- Defined architecture and constraints
- Established free-tier approach
- Created development roadmap

---

## Notes & Observations

**Key Insights:**
- Most retail arb bots are NOT optimized (opportunity for us)
- Free tier is sufficient for 1-5 markets (300M Alchemy requests >> our needs)
- Speed optimization matters, but strategy correctness matters MORE
- Fee modeling is THE most critical component (5-50 bps edge)
- Starting small ($50) is actually BETTER (learn without risk)

**Future Enhancements (Post-Profitability):**
- Multi-market scanning (3-10 markets)
- Short arbitrage (sell both sides if have inventory)
- Copy trading (follow smart wallets)
- Cross-market arbitrage (same event, different markets)
- Statistical arbitrage (predict mispricing)
- Upgrade to paid VPS + RPC (lower latency)

**Questions to Revisit:**
- What's the actual average arb frequency? (TBD after Week 1)
- What's the real average profit per arb? (TBD after Week 2)
- Is free tier latency actually a bottleneck? (TBD after testing)
- Should we use Oracle Cloud VPS or stay on laptop? (TBD after Week 1 metrics)
