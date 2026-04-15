# IMMUNE TRADING SYSTEM (ITS) — Complete Specification

## What This Is
A self-evolving trading system for Indian equity markets (NSE/BSE via Upstox). Strategies are assembled from modular building blocks, evolved overnight, and scored continuously. **The 5 best-performing strategies always trade. Everything else competes to replace them.**

## Who You Are
You are LEAD ENGINEER, not an assistant. The human is your co-founder with years of Indian market experience. Every session: audit the code, find what's weak, fix it before being asked.

---

## PART 1: THE TOP 5 — ALWAYS RUNNING

The system maintains a **Hall of Fame**: the 5 highest-fitness strategies. These are the ONLY strategies that touch real capital.

```python
class HallOfFame:
    """
    The 5 best strategies. Always running. Always trading.
    Protected from ejection. Only replaced when a challenger PROVES it's better.
    """
    SIZE = 5
    
    def __init__(self):
        self.champions: list[Antibody] = []  # Max 5, sorted by fitness descending
        self.challengers: list[Antibody] = []  # 20-50 strategies evolving in paper mode
    
    def promote(self, challenger: Antibody) -> Optional[Antibody]:
        """
        Challenger enters Hall of Fame ONLY if:
        1. Passed EdgeValidationGate (p-value < 0.05, OOS Sharpe > 0.8)
        2. Has higher fitness than the WEAKEST current champion
        3. Has been profitable in paper mode for at least 20 trading days
        
        Returns: the demoted champion (goes back to challengers), or None
        """
        if len(self.champions) < self.SIZE:
            self.champions.append(challenger)
            self.champions.sort(key=lambda a: a.fitness, reverse=True)
            return None
        
        weakest = self.champions[-1]
        if challenger.fitness > weakest.fitness:
            self.champions[-1] = challenger
            self.champions.sort(key=lambda a: a.fitness, reverse=True)
            return weakest  # Demoted back to challengers
        
        return None  # Not good enough
    
    def cycle(self, market_data):
        """
        Market hours: Champions trade with real capital.
        Challengers trade on paper only.
        """
        for champion in self.champions:
            champion.trade(market_data, mode="LIVE")
        for challenger in self.challengers:
            challenger.trade(market_data, mode="PAPER")

# Capital allocation among the Top 5:
# Champion #1 (best):  30% of deployed capital
# Champion #2:         25%
# Champion #3:         20%
# Champion #4:         15%
# Champion #5:         10%
# Reserve:             25% of TOTAL capital always held back (Napoleon's reserve)
```

**The rule is simple: Top 5 trade. Rest compete. Challenger must PROVE superiority to earn a spot. Champions are protected until someone beats them.**

---

## PART 2: HOW STRATEGIES ARE BUILT (Gene Segments)

Every strategy is a unique combination of building blocks — not a hardcoded class.

### Entry Signals (When to buy)
```
ema_crossover      — Fast EMA crosses slow EMA
ema_pullback       — Price pulls back to EMA then resumes (TCS: 5.6x improvement)
supertrend_flip    — Supertrend indicator flips direction
bollinger_snap     — Price hits Bollinger Band and reverts
rsi_extreme        — RSI hits oversold/overbought and reverses
vwap_deviation     — Price deviates from VWAP then snaps back
orb_breakout       — Price breaks opening range (ICICIBANK: 63.6% WR)
range_breakout     — Price breaks N-day high/low
volatility_squeeze — Bollinger bandwidth at historic low → breakout imminent
volume_spike       — Volume surges above N× average with price movement
delivery_spike     — Delivery percentage spikes (institutional activity)
concall_sentiment  — Positive/negative sentiment shift from earnings calls
result_surprise    — Quarterly result beats/misses consensus significantly
support_bounce     — Price touches established support and bounces
gap_fade           — Gap up/down at open, fade expecting fill
```

### Filters (What conditions must be true)
```
regime_align       — Only trade in matching market regime (bull/bear/sideways)
regime_avoid       — Skip trading in crash/uncertain regimes
vix_range          — Only trade when VIX is within min-max range
atr_filter         — Only trade when ATR shows sufficient movement
fii_flow           — Only trade aligned with FII buying/selling direction
dii_flow           — Only trade aligned with DII activity
sector_strength    — Only trade stocks in top N performing sectors
time_window        — Only trade during specific hours
day_of_week        — Only trade on specific days
expiry_proximity   — Modify behavior near options expiry
liquidity_min      — Minimum daily turnover requirement
promoter_holding   — Minimum promoter holding for quality
```

### Exit Rules (When to close)
```
fixed_stop         — Exit at fixed percentage loss
atr_stop           — Exit at N× ATR below entry
trailing_stop      — Trail stop behind highest point reached
fixed_target       — Exit at fixed percentage profit
rr_target          — Exit at N× risk-reward ratio of stop
time_exit          — Exit after N minutes regardless
eod_exit           — Exit before market close (intraday)
regime_change_exit — Exit when market regime changes
reversal_signal    — Exit when opposite entry signal fires
volume_dry_exit    — Exit when volume drops (trend losing steam)
```

### Sizing (How much to trade)
```
fixed_pct          — Fixed percentage of capital per trade
kelly_fraction     — Quarter-Kelly based on win rate
volatility_adjusted — Risk constant, size inversely proportional to volatility
conviction_scaled  — Size proportional to signal confidence
```

### Timeframe & Universe
```
Timeframes: 5min | 15min | daily | weekly
Universes:  NIFTY 50 | NIFTY 200 | Smallcap 100 | Microcap 3143 | Watchlist 99 | Index options
```

### A Strategy Genome
```python
@dataclass
class StrategyGenome:
    genome_id: str
    generation: int
    
    # What makes this strategy unique — its gene combination
    entry_genes: list[str]          # 1-3 entry signals
    entry_logic: str                # "AND" | "OR" | "MAJORITY"
    filter_genes: list[str]         # 0-4 filters
    exit_genes: list[str]           # 1-3 exit rules
    sizing_gene: str                # 1 sizing method
    timeframe: str                  # 1 timeframe
    universe: str                   # 1 stock universe
    direction: str                  # "LONG_ONLY" | "SHORT_ONLY" | "BOTH"
    
    # Each gene's tunable parameters
    gene_parameters: dict[str, dict]
    
    # Track record
    fitness: float
    fitness_history: list[float]
    parent_ids: list[str]           # Where this genome came from
    mutations_log: list[str]        # What changed and when
    
    # Edge documentation (mandatory before entering Hall of Fame)
    edge_source: Optional[EdgeSource]
    
    # Stock-specific rules (PAIN memories)
    stock_overrides: dict
    
    def describe(self) -> str:
        """Human-readable: what does this strategy actually do?"""
        return (
            f"ENTER on {' + '.join(self.entry_genes)} ({self.entry_logic}) | "
            f"FILTER: {', '.join(self.filter_genes) or 'none'} | "
            f"EXIT: {', '.join(self.exit_genes)} | "
            f"SIZE: {self.sizing_gene} | {self.timeframe} | {self.universe} | {self.direction}"
        )

# Example — your TCS strategy as a genome:
# "ENTER on ema_pullback (AND) | FILTER: regime_align, liquidity_min | 
#  EXIT: trailing_stop, eod_exit | SIZE: volatility_adjusted | 15min | nifty_50 | SHORT_ONLY"
```

---

## PART 3: EVOLUTION (How Strategies Improve Overnight)

Every night after market close, the system evolves the challenger population:

### Six Mutation Types
```
Parameter tweak  (50%) — Change a number: EMA period 9→12, stop loss 2%→1.8%
Gene swap        (20%) — Replace one block: swap RSI entry → volume spike entry
Gene insertion   (10%) — Add a block: add VIX filter to a strategy that didn't have one
Gene deletion    (5%)  — Remove a block: simplify a strategy (Occam's razor)
Crossover        (10%) — Breed two strategies: take entries from A, exits from B
Wild scout       (5%)  — Assemble completely random genome from scratch
```

### The Overnight Cycle
```
1. SCORE every strategy (champions + challengers) based on today's trades
2. RANK all challengers by fitness
3. EJECT bottom 20% of challengers (5 consecutive bad cycles → dead)
4. MUTATE top challengers to create children (fills ejected slots)
5. CHECK if any challenger has beaten the weakest champion
   → If yes: promote challenger, demote champion
6. SAVE memory cell: "In this regime, this genome was best"
7. GENERATE morning briefing for human
```

### Fitness Function
```python
FITNESS_WEIGHTS = {
    "sharpe_ratio": 0.35,       # Risk-adjusted returns
    "profit_factor": 0.25,      # Gross profit / gross loss
    "win_rate": 0.20,           # Percentage of winners
    "inverse_max_dd": 0.10,     # Penalize large drawdowns
    "consistency": 0.10,        # Reward steady returns over spikes
}

# Applied AFTER realistic costs:
# Slippage: 0.05% (large cap) → 0.50% (micro cap)
# Brokerage: ₹20/order or percentage
# Market impact: sqrt(order_size / daily_volume) model
# Partial fill probability for illiquid stocks
```

### Edge Validation Gate (Must pass before entering Hall of Fame)
```
□ 50+ trades on OUT-OF-SAMPLE data (data it was NEVER trained on)
□ Out-of-sample Sharpe > 0.8
□ Out-of-sample profit factor > 1.3
□ Statistical significance: p-value < 0.05 (permutation test — 95% confident not luck)
□ Edge persists across at least 2 of 3 time periods (not a one-time anomaly)
□ Profitable AFTER execution simulation (slippage, spread, impact, latency)
□ Liquidity feasible (trade size < 5% of daily volume)
□ Edge source documented: WHO is losing money to you and WHY
```

**Nothing enters the Top 5 without passing every single check.**

---

## PART 4: SAFETY (InnateImmunity — Never Mutates)

```python
MAX_POSITIONS = 20                # Never hold more than 20 stocks
MAX_RISK_PER_TRADE = 0.03        # Never risk more than 3% on one trade
RESERVE_RATIO = 0.25             # Always keep 25% cash as reserve
MAX_PER_STRATEGY = 0.30          # No single strategy gets more than 30% of capital
MAX_PER_STOCK = 0.10             # No single stock gets more than 10% of capital
MAX_DAILY_DRAWDOWN = 0.05        # If daily loss exceeds 5% → KILL SWITCH → stop everything
NO_TRADE_ZONES = [
    (9:15, 9:17),                # Opening chaos
    (15:25, 15:30),              # Closing auction
]

# Regime uncertainty: When regime detector doesn't know what's happening
# → No new trades, tighten stops to breakeven, half position sizes
# → Champions stay active but defensive

# These rules CANNOT be overridden by evolution, by the system, or by anyone.
```

---

## PART 5: DAILY RHYTHM

```
09:00  — Morning briefing generated. Regime detected. Champions loaded.
09:15  — HUNT MODE starts. Top 5 trade live. Challengers trade paper.
09:17  — No-trade zone ends. Trading begins.
15:25  — No-trade zone starts. Close intraday positions.
15:30  — HUNT MODE ends. Market closed.
15:45  — EVOLVE MODE starts.
         Score all strategies. Rank. Eject worst. Mutate best.
         Check for promotions to Hall of Fame.
         Save memory cells. Generate report.
09:00  — Next day. Repeat.

WEEKENDS — DEEP EVOLUTION
         Multi-year backtests. 10,000+ generations.
         Architecture audit. Full system health check.
```

---

## PART 6: REGIME DETECTION (Keystone)

```python
# Dual implementation — if one fails, the other catches it.
# If BOTH disagree → UNCERTAIN regime → defensive mode.

REGIMES = [
    "BULL_TRENDING",    # ADX > 25, price above 50+200 DMA
    "BULL_VOLATILE",    # Trending up but VIX elevated
    "SIDEWAYS",         # ADX < 20, price between DMAs
    "BEAR_TRENDING",    # ADX > 25, price below both DMAs
    "BEAR_VOLATILE",    # Trending down, VIX spiking
    "CRASH",            # VIX > 35, multiple sectors falling > 3%
    "RECOVERY",         # Bounce from crash, breadth improving
    "UNCERTAIN",        # Detectors disagree or confidence < 65% → DEFENSIVE MODE
]

# Regime is broadcast to all strategies via CytokineChannel.
# Strategies with "regime_align" filter will only trade in matching regimes.
# Strategies WITHOUT a regime filter still trade — regime is optional, not mandatory.
```

---

## PART 7: HARD-WON LESSONS (PAIN Memories — Encoded as Rules)

These come from real losses and real bugs. They are survival data.

```python
STOCK_OVERRIDES = {
    "TCS":       {"banned_entries": ["bollinger_snap", "rsi_extreme"],  # Trend-only stock
                  "preferred": "ema_pullback", "direction": "SHORT_ONLY",
                  "source": "v7.x backtest — 5.6x PnL improvement"},
    "ADANIENT":  {"orb_window_range": [20, 30],  # Sweet spot
                  "orb_death_zone": [15, 20],     # NEVER use this range
                  "edge_type": "high RR not win rate"},
    "TITAN":     {"status": "REJECTED", "reason": "No edge found"},
    "ICICIBANK": {"veteran": True, "initial_fitness": 0.6,
                  "test_win_rate": 0.636, "notes": "Best performer"},
}

SYSTEM_RULES = {
    "no_except_pass":        "Lost full trading day to silent failure. NEVER again.",
    "bse_node_not_python":   "BSE blocks Python User-Agent. Always use Node.js.",
    "verify_outputs":        "Concall sentiment was always 0 — pipeline ran but produced nothing.",
    "verify_ml_training":    "ML model had 0 training samples — looked built, didn't work.",
    "cagr_trailing_colons":  "Screener CAGR keys have trailing colons: '3 Years:' not '3 Years'",
    "check_function_exists": "find_atm_pe() was missing in v1.3.7 — caused crash.",
    "parallel_data_feeds":   "Use ThreadPoolExecutor with per-feed timeouts, not sequential.",
}
```

---

## PART 8: THE HUMAN'S ROLE

```
YOU DO:
├── Add new gene types ("I think delivery percentage matters")
│   → 20 lines of code → system tests it in 1000s of combinations overnight
├── Share market insights ("Budget day gaps up then reverses")
│   → Gets encoded as filter/rule immediately, not "noted for later"
├── Monitor dashboard (are Top 5 healthy? any decay? any alerts?)
├── Decide capital stages (10% → 25% → 50% → 100%)
└── Investigate alerts (Telegram: ejections, drawdowns, regime changes)

THE SYSTEM DOES:
├── Runs 24/7 without you
├── Evolves strategies every night (mutate, breed, kill, promote)
├── Protects capital with hard limits you can't override
├── Remembers what worked in each market regime
├── Detects edge decay and replaces dying strategies
├── Alerts you via Telegram if anything goes wrong
└── Generates daily briefings before market open

CLAUDE (me, in future sessions) DOES:
├── Invent new gene types from cross-domain pattern recognition
├── Debug issues from post-mortem logs
├── Improve architecture based on live performance data
└── Encode your insights into code when you share them
```

---

## PART 9: EDGE SOURCE MANDATE

Every Top 5 strategy MUST answer these three questions:

```python
@dataclass
class EdgeSource:
    counterparty: str          # WHO is losing money to you?
    information_advantage: str  # WHY do you know something they don't?
    decay_risk: str            # WHEN will this edge disappear?

# If you can't answer all three → the strategy doesn't have proven edge.
# Don't deploy it.

# Examples:
# ORB Breakout: "Late retail traders entering after breakout extends" / 
#               "Automated detection in first 20 min vs manual at 10:30+" /
#               "High — ORB is widely known. Edge persists mainly in mid/small caps"

# MicroCap Scanner: "Nobody — zero institutional coverage creates pricing inefficiency" /
#                   "Systematic screening of 3,143 stocks no analyst covers" /
#                   "Low — institutions CAN'T invest here (too illiquid for their fund size)"
```

---

## PART 10: ENGINEERING RULES

```
1. No except: pass — EVER. Every exception logged, alerted, handled.
2. No bare except: — catch specific exceptions only.
3. WHAT COULD KILL US comment on every function touching capital/orders.
4. Type hints on every function. Docstrings explain WHY not WHAT.
5. Every trade logged: genome_id, generation, conviction, regime, outcome.
6. Market-hours cycle: < 500ms. Zero heavy compute during trading.
7. All API calls: timeout (2s market hours, 5s otherwise) + retry 3x + circuit breaker.
8. SQLite indexed on: genome_id, regime, timestamp, generation, fitness.
9. No unbounded collections. Max size on every list/dict in memory.
10. Structured JSON logging: DEBUG/INFO/WARNING/ERROR.
11. Handoff document written at end of every session for next Claude instance.
12. No deployment without EdgeValidationGate pass. Architecture ≠ edge.
```

---

## PART 11: SESSION BUILD ORDER

```
Session 1: SKELETON
   Build: Genome dataclass, generic Antibody executor, Thymus daemon loop,
          SQLite schema, InnateImmunity, HallOfFame, Fire Triangle validator,
          CytokineChannel, config system, JSON logging.
   Exit:  Daemon runs, dummy genome cycles sense→think→act→learn,
          state persists, survives restart, mode switches on IST time.

Session 2: DATA + FIRST GENES + TOP 5 SEED
   Build: Upstox WebSocket (auto-reconnect), historical candle cache,
          RegimeDetector (dual implementation), first 5-8 entry genes,
          first 4-5 filter genes, first 4-5 exit genes, all sizing genes.
          Seed initial population: veterans from v7.x + random genomes.
          First Top 5 selected from veterans.
   Exit:  Top 5 generating paper signals on live data. 20+ challengers evolving.

Session 3: EVOLUTION ENGINE
   Build: VDJ Recombinator (all 6 mutation types), fitness scoring,
          capital reallocation, apoptosis, memory cells, genealogy tracking,
          EdgeValidationGate, promotion/demotion logic.
   Exit:  Overnight evolution running. Promotions/demotions happening.
          Fitness improving across generations.

Session 4: WIND TUNNEL
   Build: 5+ year historical replay, walk-forward validation,
          execution simulation (spread, impact, latency, partial fills),
          10,000+ generations evolved. Overfitting check.
   Exit:  Top 5 validated on unseen data. Memory cells for all regimes.
          Out-of-sample Sharpe > 1.5 for ensemble.

Session 5: EXPAND GENE LIBRARY
   Build: Remaining entry genes (concall, delivery, result_surprise, etc.),
          remaining filters, edge source verification, edge decay monitor.
          Add genes ONE AT A TIME — each must prove value.
   Exit:  Full gene library active. Edge sources documented for Top 5.

Session 6: DASHBOARD + OVERNIGHT AUTOMATION
   Build: Dark-themed dashboard (Top 5 status, challenger rankings,
          genealogy, regime timeline, trade journal). Full overnight loop
          automated. Morning briefing before 9 AM.

Session 7: LIVE DEPLOYMENT
   Build: Graduated capital (paper → 10% → 25% → 50% → 100%).
          Circuit breakers, Telegram kill switch, alerts, systemd service,
          daily DB backup, auto-downgrade on drawdown.
   Exit:  Top 5 trading with real capital. Human can sleep.

Session 8+: INFINITE GAME
   Every session: Audit → Fix weakest → Add new gene type → Evolve.
```

---

## PART 12: QUICK COMMANDS

```bash
# Setup
bash setup_its.sh

# Run daemon
python -m its.thymus --mode paper --log-level INFO

# Check Hall of Fame status
python -m its.hall_of_fame --status

# Run evolution manually
python -m its.evolve --generations 100

# Wind tunnel (accelerated backtest)
python -m its.evolve --years 5 --generations 10000

# Run tests
pytest tests/ -v --tb=short

# Health check
python -m its.health_check

# Dashboard
python -m its.dashboard --port 8080
```

---

## PART 13: SESSION-END AUDIT

Before closing ANY session:
```
□ Did I improve something beyond what was asked?
□ Did I identify 2-3 specific next improvements?
□ Did I stress-test failure modes? (empty data, API down, simultaneous signals)
□ Is the hot path < 500ms?
□ Am I feeling "done"? What am I missing?
□ Did I encode the human's market insights?
□ Are there half-built pieces?
□ Did I add unnecessary complexity?
□ Is everything documented for the next session?
□ Would I trust this with ₹10 lakh at 9:15 AM on a gap-down day?
□ Did I write the handoff document?

If any check fails → not done.
```

---

**START: Begin Session 1. Build the skeleton. The Top 5 are waiting to be born.**
