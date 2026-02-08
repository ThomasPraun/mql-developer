# Backtesting & Optimization

## Table of Contents

- [Strategy Tester Modes](#strategy-tester-modes)
- [Minimum Requirements for Valid Backtest](#minimum-requirements-for-valid-backtest)
- [OnTester() - Custom Optimization Criterion](#ontester---custom-optimization-criterion-mql5)
- [TesterStatistics() Constants](#testerstatistics-constants)
- [Key Performance Metrics](#key-performance-metrics)
- [Walk-Forward Analysis](#walk-forward-analysis)
- [Avoiding Overfitting](#avoiding-overfitting)
- [Monte Carlo Simulation](#monte-carlo-simulation)
- [Multi-Currency Testing (MQL5)](#multi-currency-testing-mql5)
- [Optimization Modes (MT5)](#optimization-modes-mt5)
- [Frame Functions (Inter-Pass Communication)](#frame-functions-inter-pass-communication)
- [Practical Backtesting Workflow](#practical-backtesting-workflow)

---

## Strategy Tester Modes

| Mode | Speed | Accuracy | When to Use |
|------|-------|----------|-------------|
| Every tick based on real ticks | Slowest | Most realistic | Final validation with broker-specific ticks |
| Every tick | Slow | High | Final validation, scalpers, intra-bar logic |
| 1-minute OHLC | Medium | Good | Most strategies (4 ticks per M1 bar) |
| Open prices only | Fast | Low | Bar-open strategies, rapid optimization |
| Math calculations | Instant | N/A | Pure computation without ticks |

Best practice: Optimize with Open Prices first, validate winners with Every Tick.

## Minimum Requirements for Valid Backtest

- 1,000+ trades minimum (ideally 2,000+)
- 2+ full market cycles (bull + bear + range)
- Multiple years of data
- Consistent spread settings (or variable spread)

## OnTester() - Custom Optimization Criterion (MQL5)

```mql5
double OnTester() {
    double profit = TesterStatistics(STAT_PROFIT);
    double profitFactor = TesterStatistics(STAT_PROFIT_FACTOR);
    double sharpeRatio = TesterStatistics(STAT_SHARPE_RATIO);
    double recoveryFactor = TesterStatistics(STAT_RECOVERY_FACTOR);
    double maxDrawdown = TesterStatistics(STAT_EQUITY_DD_RELATIVE);
    double totalTrades = TesterStatistics(STAT_TRADES);

    if(totalTrades < 100) return 0;     // Reject small samples
    if(maxDrawdown > 25) return 0;      // Reject excessive DD
    if(profitFactor < 1.3) return 0;    // Reject low PF

    return sharpeRatio * recoveryFactor * MathSqrt(totalTrades);
}
```

## TesterStatistics() Constants

Complete table:

- STAT_PROFIT - Net profit
- STAT_GROSS_PROFIT - Gross profit
- STAT_GROSS_LOSS - Gross loss
- STAT_TRADES - Total trades
- STAT_PROFIT_TRADES - Winning trades
- STAT_LOSS_TRADES - Losing trades
- STAT_PROFIT_FACTOR - Gross profit / gross loss
- STAT_EXPECTED_PAYOFF - Expected payoff per trade
- STAT_SHARPE_RATIO - Sharpe ratio
- STAT_RECOVERY_FACTOR - Net profit / max drawdown
- STAT_EQUITY_DD - Max equity drawdown in money
- STAT_EQUITY_DD_PERCENT - Max equity drawdown %
- STAT_EQUITY_DD_RELATIVE - Relative equity drawdown %
- STAT_BALANCE_DD - Max balance drawdown
- STAT_BALANCE_DD_PERCENT - Max balance drawdown %
- STAT_MAX_PROFITTRADE - Largest profitable trade
- STAT_MAX_LOSSTRADE - Largest losing trade
- STAT_CONPROFITMAX - Max consecutive profit
- STAT_CONLOSSMAX - Max consecutive loss
- STAT_SHORT_TRADES - Short trades
- STAT_LONG_TRADES - Long trades
- STAT_WIN_SHORT_TRADES - Winning short trades
- STAT_WIN_LONG_TRADES - Winning long trades

## Key Performance Metrics

| Metric | Formula | Good Value | Excellent |
|--------|---------|------------|-----------|
| Sharpe Ratio | (Mean Return - Rf) / StdDev | > 1.0 | > 2.0 |
| Profit Factor | Gross Profit / Gross Loss | > 1.5 | > 2.0 |
| Recovery Factor | Net Profit / Max DD | > 3.0 | > 5.0 |
| Max Drawdown | Peak-to-trough % | < 20% | < 10% |
| Expected Payoff | Net Profit / Total Trades | > 0 | Context |
| Win Rate | Winning / Total | Context | Context |

## Walk-Forward Analysis

### Methodology

```
Window 1: [===== IN-SAMPLE =====][= OOS =]
Window 2:    [===== IN-SAMPLE =====][= OOS =]
Window 3:       [===== IN-SAMPLE =====][= OOS =]
Window 4:          [===== IN-SAMPLE =====][= OOS =]
```

### Steps

1. Split data into overlapping windows (e.g., 12-month IS, 3-month OOS)
2. Optimize on in-sample period
3. Test optimized params on out-of-sample period
4. Slide window forward, repeat
5. Accept if OOS performance >= 50-80% of IS
6. MT5 supports natively: Strategy Tester > Settings > Forward period

### Walk-Forward Efficiency

WFE = (OOS annualized return) / (IS annualized return)

- WFE > 0.5 = acceptable
- WFE > 0.7 = good
- WFE < 0.3 = likely overfitted

## Avoiding Overfitting

### Principles

1. **Parameter stability**: Good params have good neighbors. If X=14 works but X=13 and X=15 don't, it's curve-fitted
2. **Fewer parameters**: Each extra parameter increases overfitting risk exponentially
3. **Out-of-sample testing**: Reserve 25-30% of data untouched
4. **Cross-market validation**: Test EURUSD strategy on GBPUSD
5. **Regime awareness**: Test across trending AND ranging markets

### Signs of Overfitting

- Sharp performance drop in forward test
- Many parameters (>5 optimized)
- Strategy only works on specific date range
- Unrealistically high backtest metrics
- Parameter sensitivity (small changes cause large performance swings)

## Monte Carlo Simulation

### Purpose

Test if results are due to skill or luck by randomizing trade sequence.

### Implementation

```mql5
double OnTester() {
    // Collect all trade P&Ls
    double trades[];
    // ... populate from history ...

    int simulations = 1000;
    double worstDD95;

    for(int sim = 0; sim < simulations; sim++) {
        // Fisher-Yates shuffle
        // Calculate max drawdown for shuffled sequence
    }

    // Sort DDs, find 95th percentile
    // Reject if 95th percentile DD > 30%

    return profit / (1 + worstDD95);
}
```

### What Monte Carlo Tells You

- Expected range of drawdowns (not just the one historical path)
- Probability of ruin at different risk levels
- Confidence interval for returns
- If strategy is fragile (high variance across simulations)

## Multi-Currency Testing (MQL5)

MT5 Strategy Tester supports multi-symbol natively:

```mql5
int OnInit() {
    // Reference other symbols to include them in test
    int handle_eur = iMA("EURUSD", PERIOD_H1, 14, 0, MODE_SMA, PRICE_CLOSE);
    int handle_gbp = iMA("GBPUSD", PERIOD_H1, 14, 0, MODE_SMA, PRICE_CLOSE);
    // Tester auto-synchronizes all referenced symbols
    return INIT_SUCCEEDED;
}
```

## Optimization Modes (MT5)

| Mode | Description |
|------|-------------|
| Slow (Complete) | Tests every combination (exhaustive) |
| Fast (Genetic) | Genetic algorithm, finds near-optimal efficiently |
| Custom max/min | Optimizes by OnTester() return value |

### Cloud Computing

MQL5 Cloud Network distributes optimization across thousands of agents worldwide.

## Frame Functions (Inter-Pass Communication)

```mql5
// In EA (agent): send data at end of each pass
double OnTester() {
    uchar data[];
    // serialize results into data
    FrameAdd("Results", 1, profit, data);
    return profit;
}

// In terminal: receive during optimization
void OnTesterPass() {
    ulong pass; string name; long id; double value; uchar data[];
    while(FrameNext(pass, name, id, value, data)) {
        PrintFormat("Pass #%d: profit=%.2f", pass, value);
    }
}

// Control functions
void OnTesterInit() { /* before optimization starts */ }
void OnTesterDeinit() { /* after optimization finishes, aggregate results */ }
```

## Practical Backtesting Workflow

1. **Quick scan**: Open Prices Only, wide parameter ranges, genetic optimization
2. **Narrow down**: Reduce ranges around promising areas, complete optimization
3. **Validate**: Every Tick mode with best parameters
4. **Walk-forward**: Set forward period, verify OOS performance
5. **Monte Carlo**: Randomize trade sequence, check robustness
6. **Multi-market**: Test on correlated instruments
7. **Demo forward test**: Run on demo account for 1-3 months minimum
