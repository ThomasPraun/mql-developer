# Architecture Patterns

Reference for EA project structure, architecture patterns, and design patterns in MQL4/MQL5.

## Table of Contents

- [Project Structure](#project-structure)
- [EA Architecture Patterns](#ea-architecture-patterns)
- [Design Patterns for MQL](#design-patterns-for-mql)
- [Include File Design (.mqh)](#include-file-design-mqh)
- [Complete Templates](#complete-templates)
- [Quick Reference: When to Use Each Pattern](#quick-reference-when-to-use-each-pattern)

---

## Project Structure

### Full Modular Structure (Recommended for Professional EAs)

```
MQL5/                              (or MQL4/)
├── Experts/
│   └── MyEA/
│       └── MyEA.mq5               // EA entry point (orchestrator)
├── Indicators/
│   └── MyIndicator.mq5
├── Scripts/
│   └── MyScript.mq5
├── Include/
│   ├── Core/
│   │   ├── CSignalBase.mqh        // Abstract signal interface
│   │   ├── CSignalMA.mqh          // MA crossover signal
│   │   ├── CSignalRSI.mqh         // RSI signal
│   │   ├── CTradeManager.mqh      // Order execution + retries
│   │   └── CRiskManager.mqh       // Position sizing + drawdown
│   ├── Filters/
│   │   ├── CFilterBase.mqh        // Abstract filter interface
│   │   ├── CTimeFilter.mqh        // Session/day-of-week filter
│   │   ├── CSpreadFilter.mqh      // Max spread filter
│   │   └── CVolatilityFilter.mqh  // ATR-based volatility filter
│   ├── Communication/
│   │   ├── CHttpClient.mqh        // WebRequest wrapper
│   │   └── CJsonHelper.mqh        // JSON build/parse
│   ├── UI/
│   │   └── CPanel.mqh             // On-chart trading panel
│   └── Utils/
│       ├── CLogger.mqh            // Logging utility
│       └── CSymbolHelper.mqh      // Multi-symbol helpers
└── Libraries/
    └── MyLibrary.mq5              // Compiled library (ex4/ex5)
```

### Simplified Single-File Structure (Small Projects)

For simple strategies, a single `.mq5` or `.mq4` file is acceptable:

```
MQL5/
├── Experts/
│   └── SimpleMA_EA.mq5            // Everything in one file
```

Use single-file when:
- Strategy has one signal, one entry logic, basic risk management
- No need for code reuse across EAs
- Quick prototyping or proof of concept

Move to modular when:
- Multiple signal types or strategies
- Shared code between EAs
- Complex risk or filter logic
- Team collaboration

---

## EA Architecture Patterns

### 1. Simple Single-File EA

Basic template with the three core event handlers. Suitable for simple strategies.

#### MQL4 Simple Template

```mql4
//+------------------------------------------------------------------+
//|                                              SimpleMA_EA.mq4     |
//+------------------------------------------------------------------+
#property copyright "Developer"
#property version   "1.00"
#property strict

//--- Input parameters
input int    InpMagicNumber  = 12345;     // Magic Number
input double InpLots         = 0.1;       // Lot Size
input int    InpStopLoss     = 50;        // Stop Loss (pips)
input int    InpTakeProfit   = 100;       // Take Profit (pips)
input int    InpFastMA       = 10;        // Fast MA Period
input int    InpSlowMA       = 20;        // Slow MA Period

//--- Global variables
double g_pipSize;
int    g_slippage = 3;

//+------------------------------------------------------------------+
//| Expert initialization                                             |
//+------------------------------------------------------------------+
int OnInit()
  {
   // Detect 4/5-digit broker
   if(Digits == 3 || Digits == 5)
     {
      g_pipSize   = Point * 10;
      g_slippage  = 30;
     }
   else
     {
      g_pipSize   = Point;
      g_slippage  = 3;
     }

   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
//| Expert deinitialization                                           |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {
   // Cleanup: remove objects, release handles
  }

//+------------------------------------------------------------------+
//| Expert tick function                                              |
//+------------------------------------------------------------------+
void OnTick()
  {
   // Only trade on new bar
   if(!IsNewBar())
      return;

   // Check if already in a position
   if(CountOpenOrders() > 0)
      return;

   // Generate signal
   int signal = GetSignal();

   // Execute trade
   if(signal == 1)       // Buy signal
      OpenBuy();
   else if(signal == -1) // Sell signal
      OpenSell();
  }

//+------------------------------------------------------------------+
//| Signal generation: MA crossover                                  |
//+------------------------------------------------------------------+
int GetSignal()
  {
   double fastMA_1 = iMA(NULL, 0, InpFastMA, 0, MODE_EMA, PRICE_CLOSE, 1);
   double fastMA_2 = iMA(NULL, 0, InpFastMA, 0, MODE_EMA, PRICE_CLOSE, 2);
   double slowMA_1 = iMA(NULL, 0, InpSlowMA, 0, MODE_EMA, PRICE_CLOSE, 1);
   double slowMA_2 = iMA(NULL, 0, InpSlowMA, 0, MODE_EMA, PRICE_CLOSE, 2);

   // Bullish crossover
   if(fastMA_2 <= slowMA_2 && fastMA_1 > slowMA_1)
      return 1;

   // Bearish crossover
   if(fastMA_2 >= slowMA_2 && fastMA_1 < slowMA_1)
      return -1;

   return 0;
  }

//+------------------------------------------------------------------+
//| Open Buy order                                                   |
//+------------------------------------------------------------------+
void OpenBuy()
  {
   double sl = (InpStopLoss  > 0) ? NormalizeDouble(Ask - InpStopLoss  * g_pipSize, Digits) : 0;
   double tp = (InpTakeProfit > 0) ? NormalizeDouble(Ask + InpTakeProfit * g_pipSize, Digits) : 0;

   int ticket = OrderSend(Symbol(), OP_BUY, InpLots, Ask, g_slippage, sl, tp,
                           "SimpleMA Buy", InpMagicNumber, 0, clrGreen);

   if(ticket < 0)
      PrintFormat("OrderSend BUY failed: error %d", GetLastError());
   else
      PrintFormat("BUY opened: ticket=%d price=%.5f", ticket, Ask);
  }

//+------------------------------------------------------------------+
//| Open Sell order                                                  |
//+------------------------------------------------------------------+
void OpenSell()
  {
   double sl = (InpStopLoss  > 0) ? NormalizeDouble(Bid + InpStopLoss  * g_pipSize, Digits) : 0;
   double tp = (InpTakeProfit > 0) ? NormalizeDouble(Bid - InpTakeProfit * g_pipSize, Digits) : 0;

   int ticket = OrderSend(Symbol(), OP_SELL, InpLots, Bid, g_slippage, sl, tp,
                           "SimpleMA Sell", InpMagicNumber, 0, clrRed);

   if(ticket < 0)
      PrintFormat("OrderSend SELL failed: error %d", GetLastError());
   else
      PrintFormat("SELL opened: ticket=%d price=%.5f", ticket, Bid);
  }

//+------------------------------------------------------------------+
//| Count open orders for this EA                                    |
//+------------------------------------------------------------------+
int CountOpenOrders()
  {
   int count = 0;
   for(int i = OrdersTotal() - 1; i >= 0; i--)
     {
      if(OrderSelect(i, SELECT_BY_POS, MODE_TRADES))
        {
         if(OrderSymbol() == Symbol() && OrderMagicNumber() == InpMagicNumber)
            count++;
        }
     }
   return count;
  }

//+------------------------------------------------------------------+
//| Detect new bar                                                   |
//+------------------------------------------------------------------+
bool IsNewBar()
  {
   static datetime lastBarTime = 0;
   datetime currentBarTime = iTime(NULL, 0, 0);

   if(currentBarTime != lastBarTime)
     {
      lastBarTime = currentBarTime;
      return true;
     }
   return false;
  }
```

#### MQL5 Simple Template

```mql5
//+------------------------------------------------------------------+
//|                                              SimpleMA_EA.mq5     |
//+------------------------------------------------------------------+
#property copyright "Developer"
#property version   "1.00"

#include <Trade\Trade.mqh>

//--- Input parameters
input int    InpMagicNumber  = 12345;     // Magic Number
input double InpLots         = 0.1;       // Lot Size
input int    InpStopLoss     = 50;        // Stop Loss (pips)
input int    InpTakeProfit   = 100;       // Take Profit (pips)
input int    InpFastMA       = 10;        // Fast MA Period
input int    InpSlowMA       = 20;        // Slow MA Period

//--- Global variables
CTrade g_trade;
double g_pipSize;
int    g_handleFastMA;
int    g_handleSlowMA;

//+------------------------------------------------------------------+
//| Expert initialization                                             |
//+------------------------------------------------------------------+
int OnInit()
  {
   // Detect pip size
   int digits = (int)SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);
   g_pipSize = (digits == 3 || digits == 5) ? _Point * 10 : _Point;

   // Configure CTrade
   g_trade.SetExpertMagicNumber(InpMagicNumber);
   g_trade.SetDeviationInPoints(10);
   g_trade.SetTypeFilling(DetectFillingPolicy());

   // Create indicator handles
   g_handleFastMA = iMA(_Symbol, PERIOD_CURRENT, InpFastMA, 0, MODE_EMA, PRICE_CLOSE);
   g_handleSlowMA = iMA(_Symbol, PERIOD_CURRENT, InpSlowMA, 0, MODE_EMA, PRICE_CLOSE);

   if(g_handleFastMA == INVALID_HANDLE || g_handleSlowMA == INVALID_HANDLE)
     {
      PrintFormat("Failed to create indicator handles");
      return(INIT_FAILED);
     }

   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
//| Expert deinitialization                                           |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {
   if(g_handleFastMA != INVALID_HANDLE) IndicatorRelease(g_handleFastMA);
   if(g_handleSlowMA != INVALID_HANDLE) IndicatorRelease(g_handleSlowMA);
  }

//+------------------------------------------------------------------+
//| Expert tick function                                              |
//+------------------------------------------------------------------+
void OnTick()
  {
   // Only trade on new bar
   if(!IsNewBar())
      return;

   // Check if already in position
   if(PositionSelect(_Symbol))
      return;

   // Get signal
   int signal = GetSignal();

   // Execute
   if(signal == 1)
      OpenBuy();
   else if(signal == -1)
      OpenSell();
  }

//+------------------------------------------------------------------+
//| Signal generation: MA crossover                                  |
//+------------------------------------------------------------------+
int GetSignal()
  {
   double fastMA[], slowMA[];
   ArraySetAsSeries(fastMA, true);
   ArraySetAsSeries(slowMA, true);

   if(CopyBuffer(g_handleFastMA, 0, 0, 3, fastMA) < 3) return 0;
   if(CopyBuffer(g_handleSlowMA, 0, 0, 3, slowMA) < 3) return 0;

   // Bullish crossover (bar[2] -> bar[1])
   if(fastMA[2] <= slowMA[2] && fastMA[1] > slowMA[1])
      return 1;

   // Bearish crossover
   if(fastMA[2] >= slowMA[2] && fastMA[1] < slowMA[1])
      return -1;

   return 0;
  }

//+------------------------------------------------------------------+
//| Open Buy                                                         |
//+------------------------------------------------------------------+
void OpenBuy()
  {
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double sl  = (InpStopLoss > 0)   ? NormalizeDouble(ask - InpStopLoss * g_pipSize, _Digits) : 0;
   double tp  = (InpTakeProfit > 0)  ? NormalizeDouble(ask + InpTakeProfit * g_pipSize, _Digits) : 0;

   if(!g_trade.Buy(InpLots, _Symbol, ask, sl, tp, "SimpleMA Buy"))
      PrintFormat("Buy failed: %d - %s", g_trade.ResultRetcode(), g_trade.ResultRetcodeDescription());
  }

//+------------------------------------------------------------------+
//| Open Sell                                                        |
//+------------------------------------------------------------------+
void OpenSell()
  {
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double sl  = (InpStopLoss > 0)   ? NormalizeDouble(bid + InpStopLoss * g_pipSize, _Digits) : 0;
   double tp  = (InpTakeProfit > 0)  ? NormalizeDouble(bid - InpTakeProfit * g_pipSize, _Digits) : 0;

   if(!g_trade.Sell(InpLots, _Symbol, bid, sl, tp, "SimpleMA Sell"))
      PrintFormat("Sell failed: %d - %s", g_trade.ResultRetcode(), g_trade.ResultRetcodeDescription());
  }

//+------------------------------------------------------------------+
//| Detect filling policy                                            |
//+------------------------------------------------------------------+
ENUM_ORDER_TYPE_FILLING DetectFillingPolicy()
  {
   long filling = SymbolInfoInteger(_Symbol, SYMBOL_FILLING_MODE);

   if((filling & SYMBOL_FILLING_FOK) == SYMBOL_FILLING_FOK)
      return ORDER_FILLING_FOK;

   if((filling & SYMBOL_FILLING_IOC) == SYMBOL_FILLING_IOC)
      return ORDER_FILLING_IOC;

   return ORDER_FILLING_RETURN;
  }

//+------------------------------------------------------------------+
//| Detect new bar                                                   |
//+------------------------------------------------------------------+
bool IsNewBar()
  {
   static datetime lastBarTime = 0;
   datetime currentBarTime = iTime(_Symbol, PERIOD_CURRENT, 0);

   if(currentBarTime != lastBarTime)
     {
      lastBarTime = currentBarTime;
      return true;
     }
   return false;
  }
```

---

### 2. Modular EA (Signal + Trade + Risk + Filter)

The recommended architecture for professional EAs. Each responsibility is isolated in its own class.

#### Signal Enum (shared definition)

```mql5
// File: Include/Core/CSignalBase.mqh
#ifndef CSIGNAL_BASE_MQH
#define CSIGNAL_BASE_MQH

//--- Signal types
enum ENUM_SIGNAL
  {
   SIGNAL_NO   = 0,    // No signal
   SIGNAL_BUY  = 1,    // Buy signal
   SIGNAL_SELL = -1    // Sell signal
  };

//+------------------------------------------------------------------+
//| Abstract base class for signal generators                        |
//+------------------------------------------------------------------+
class CSignalBase
  {
protected:
   string            m_symbol;
   ENUM_TIMEFRAMES   m_timeframe;
   string            m_name;

public:
                     CSignalBase(void) : m_symbol(_Symbol), m_timeframe(PERIOD_CURRENT), m_name("BaseSignal") {}
   virtual          ~CSignalBase(void) {}

   //--- Initialization
   virtual bool      Init(string symbol, ENUM_TIMEFRAMES timeframe)
     {
      m_symbol    = symbol;
      m_timeframe = timeframe;
      return true;
     }

   //--- Core method: must be overridden
   virtual ENUM_SIGNAL GenerateSignal(void) = 0;

   //--- Release indicator handles
   virtual void      Release(void) {}

   //--- Accessors
   string            Name(void) const { return m_name; }
  };

#endif
```

#### MA Crossover Signal Implementation

```mql5
// File: Include/Core/CSignalMA.mqh
#ifndef CSIGNAL_MA_MQH
#define CSIGNAL_MA_MQH

#include "CSignalBase.mqh"

//+------------------------------------------------------------------+
//| MA Crossover signal generator                                    |
//+------------------------------------------------------------------+
class CSignalMA : public CSignalBase
  {
private:
   int               m_fastPeriod;
   int               m_slowPeriod;
   ENUM_MA_METHOD    m_method;
   int               m_handleFast;
   int               m_handleSlow;

public:
                     CSignalMA(void) : m_fastPeriod(10), m_slowPeriod(20),
                                       m_method(MODE_EMA),
                                       m_handleFast(INVALID_HANDLE),
                                       m_handleSlow(INVALID_HANDLE)
     {
      m_name = "MA_Crossover";
     }

                    ~CSignalMA(void) { Release(); }

   //--- Configuration
   void              SetParameters(int fastPeriod, int slowPeriod, ENUM_MA_METHOD method)
     {
      m_fastPeriod = fastPeriod;
      m_slowPeriod = slowPeriod;
      m_method     = method;
     }

   //--- Initialization
   virtual bool      Init(string symbol, ENUM_TIMEFRAMES timeframe) override
     {
      if(!CSignalBase::Init(symbol, timeframe))
         return false;

      m_handleFast = iMA(m_symbol, m_timeframe, m_fastPeriod, 0, m_method, PRICE_CLOSE);
      m_handleSlow = iMA(m_symbol, m_timeframe, m_slowPeriod, 0, m_method, PRICE_CLOSE);

      if(m_handleFast == INVALID_HANDLE || m_handleSlow == INVALID_HANDLE)
        {
         PrintFormat("[%s] Failed to create MA handles", m_name);
         return false;
        }

      return true;
     }

   //--- Generate signal
   virtual ENUM_SIGNAL GenerateSignal(void) override
     {
      double fast[], slow[];
      ArraySetAsSeries(fast, true);
      ArraySetAsSeries(slow, true);

      if(CopyBuffer(m_handleFast, 0, 0, 3, fast) < 3) return SIGNAL_NO;
      if(CopyBuffer(m_handleSlow, 0, 0, 3, slow) < 3) return SIGNAL_NO;

      // Bullish crossover on completed bars [2] -> [1]
      if(fast[2] <= slow[2] && fast[1] > slow[1])
         return SIGNAL_BUY;

      // Bearish crossover
      if(fast[2] >= slow[2] && fast[1] < slow[1])
         return SIGNAL_SELL;

      return SIGNAL_NO;
     }

   //--- Release handles
   virtual void      Release(void) override
     {
      if(m_handleFast != INVALID_HANDLE) { IndicatorRelease(m_handleFast); m_handleFast = INVALID_HANDLE; }
      if(m_handleSlow != INVALID_HANDLE) { IndicatorRelease(m_handleSlow); m_handleSlow = INVALID_HANDLE; }
     }
  };

#endif
```

#### Trade Manager

```mql5
// File: Include/Core/CTradeManager.mqh
#ifndef CTRADE_MANAGER_MQH
#define CTRADE_MANAGER_MQH

#include <Trade\Trade.mqh>
#include <Trade\PositionInfo.mqh>

//+------------------------------------------------------------------+
//| Trade Manager: order execution with retry logic                  |
//+------------------------------------------------------------------+
class CTradeManager
  {
private:
   CTrade            m_trade;
   CPositionInfo     m_position;
   int               m_magicNumber;
   int               m_maxRetries;
   int               m_retryDelayMs;
   double            m_pipSize;

   //--- Detect filling policy
   ENUM_ORDER_TYPE_FILLING DetectFillingPolicy(string symbol)
     {
      long filling = SymbolInfoInteger(symbol, SYMBOL_FILLING_MODE);
      if((filling & SYMBOL_FILLING_FOK) == SYMBOL_FILLING_FOK)
         return ORDER_FILLING_FOK;
      if((filling & SYMBOL_FILLING_IOC) == SYMBOL_FILLING_IOC)
         return ORDER_FILLING_IOC;
      return ORDER_FILLING_RETURN;
     }

public:
                     CTradeManager(void) : m_magicNumber(0), m_maxRetries(3), m_retryDelayMs(500), m_pipSize(0) {}
                    ~CTradeManager(void) {}

   //--- Initialization
   bool              Init(int magicNumber, string symbol, int maxRetries = 3)
     {
      m_magicNumber  = magicNumber;
      m_maxRetries   = maxRetries;

      int digits = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);
      m_pipSize = (digits == 3 || digits == 5) ? SymbolInfoDouble(symbol, SYMBOL_POINT) * 10
                                                : SymbolInfoDouble(symbol, SYMBOL_POINT);

      m_trade.SetExpertMagicNumber(m_magicNumber);
      m_trade.SetDeviationInPoints(10);
      m_trade.SetTypeFilling(DetectFillingPolicy(symbol));

      return true;
     }

   //--- Open Buy with retry logic
   bool              OpenBuy(string symbol, double lots, double slPips, double tpPips, string comment = "")
     {
      for(int attempt = 0; attempt < m_maxRetries; attempt++)
        {
         double ask = SymbolInfoDouble(symbol, SYMBOL_ASK);
         int digits = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);
         double sl  = (slPips > 0) ? NormalizeDouble(ask - slPips * m_pipSize, digits) : 0;
         double tp  = (tpPips > 0) ? NormalizeDouble(ask + tpPips * m_pipSize, digits) : 0;

         if(m_trade.Buy(lots, symbol, ask, sl, tp, comment))
           {
            if(m_trade.ResultRetcode() == TRADE_RETCODE_DONE ||
               m_trade.ResultRetcode() == TRADE_RETCODE_PLACED)
              {
               PrintFormat("[TradeManager] BUY OK: ticket=%d price=%.5f",
                           m_trade.ResultDeal(), m_trade.ResultPrice());
               return true;
              }
           }

         PrintFormat("[TradeManager] BUY attempt %d failed: %d - %s",
                     attempt + 1, m_trade.ResultRetcode(), m_trade.ResultRetcodeDescription());
         Sleep(m_retryDelayMs);
        }
      return false;
     }

   //--- Open Sell with retry logic
   bool              OpenSell(string symbol, double lots, double slPips, double tpPips, string comment = "")
     {
      for(int attempt = 0; attempt < m_maxRetries; attempt++)
        {
         double bid = SymbolInfoDouble(symbol, SYMBOL_BID);
         int digits = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);
         double sl  = (slPips > 0) ? NormalizeDouble(bid + slPips * m_pipSize, digits) : 0;
         double tp  = (tpPips > 0) ? NormalizeDouble(bid - tpPips * m_pipSize, digits) : 0;

         if(m_trade.Sell(lots, symbol, bid, sl, tp, comment))
           {
            if(m_trade.ResultRetcode() == TRADE_RETCODE_DONE ||
               m_trade.ResultRetcode() == TRADE_RETCODE_PLACED)
              {
               PrintFormat("[TradeManager] SELL OK: ticket=%d price=%.5f",
                           m_trade.ResultDeal(), m_trade.ResultPrice());
               return true;
              }
           }

         PrintFormat("[TradeManager] SELL attempt %d failed: %d - %s",
                     attempt + 1, m_trade.ResultRetcode(), m_trade.ResultRetcodeDescription());
         Sleep(m_retryDelayMs);
        }
      return false;
     }

   //--- Close all positions for a symbol
   bool              CloseAll(string symbol)
     {
      bool allClosed = true;
      for(int i = PositionsTotal() - 1; i >= 0; i--)
        {
         if(m_position.SelectByIndex(i))
           {
            if(m_position.Symbol() == symbol && m_position.Magic() == m_magicNumber)
              {
               if(!m_trade.PositionClose(m_position.Ticket()))
                 {
                  PrintFormat("[TradeManager] Close failed: ticket=%d error=%d",
                              m_position.Ticket(), m_trade.ResultRetcode());
                  allClosed = false;
                 }
              }
           }
        }
      return allClosed;
     }

   //--- Check if position exists for symbol
   bool              HasPosition(string symbol)
     {
      for(int i = PositionsTotal() - 1; i >= 0; i--)
        {
         if(m_position.SelectByIndex(i))
           {
            if(m_position.Symbol() == symbol && m_position.Magic() == m_magicNumber)
               return true;
           }
        }
      return false;
     }

   //--- Accessors
   double            PipSize(void) const { return m_pipSize; }
   CTrade*           Trade(void)         { return &m_trade; }
  };

#endif
```

#### Risk Manager

```mql5
// File: Include/Core/CRiskManager.mqh
#ifndef CRISK_MANAGER_MQH
#define CRISK_MANAGER_MQH

//+------------------------------------------------------------------+
//| Risk Manager: position sizing and drawdown control               |
//+------------------------------------------------------------------+
class CRiskManager
  {
private:
   double            m_riskPercent;         // Risk per trade (%)
   double            m_maxDrawdownPercent;  // Max drawdown allowed (%)
   double            m_maxLots;             // Maximum lot size
   double            m_minLots;             // Minimum lot size
   double            m_initialBalance;      // Balance at EA start

public:
                     CRiskManager(void) : m_riskPercent(1.0), m_maxDrawdownPercent(20.0),
                                          m_maxLots(10.0), m_minLots(0.01),
                                          m_initialBalance(0) {}
                    ~CRiskManager(void) {}

   //--- Initialization
   bool              Init(double riskPercent, double maxDrawdownPercent,
                          double maxLots = 10.0, double minLots = 0.01)
     {
      m_riskPercent        = riskPercent;
      m_maxDrawdownPercent = maxDrawdownPercent;
      m_maxLots            = maxLots;
      m_minLots            = minLots;
      m_initialBalance     = AccountInfoDouble(ACCOUNT_BALANCE);

      if(m_initialBalance <= 0)
        {
         Print("[RiskManager] Invalid initial balance");
         return false;
        }

      return true;
     }

   //--- Calculate lot size based on risk percent and stop loss
   double            CalculateLots(string symbol, double slPips)
     {
      if(slPips <= 0)
         return m_minLots;

      double balance   = AccountInfoDouble(ACCOUNT_BALANCE);
      double riskMoney = balance * m_riskPercent / 100.0;

      // Get tick value for 1 lot
      double tickValue = SymbolInfoDouble(symbol, SYMBOL_TRADE_TICK_VALUE);
      double tickSize  = SymbolInfoDouble(symbol, SYMBOL_TRADE_TICK_SIZE);
      double point     = SymbolInfoDouble(symbol, SYMBOL_POINT);
      int    digits    = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);

      // Pip value calculation
      double pipSize  = (digits == 3 || digits == 5) ? point * 10 : point;
      double pipValue = tickValue * (pipSize / tickSize);

      if(pipValue <= 0)
         return m_minLots;

      double lots = riskMoney / (slPips * pipValue);

      // Normalize to lot step
      double lotStep = SymbolInfoDouble(symbol, SYMBOL_VOLUME_STEP);
      lots = MathFloor(lots / lotStep) * lotStep;

      // Clamp to limits
      lots = MathMax(lots, m_minLots);
      lots = MathMin(lots, m_maxLots);

      double symbolMaxLot = SymbolInfoDouble(symbol, SYMBOL_VOLUME_MAX);
      double symbolMinLot = SymbolInfoDouble(symbol, SYMBOL_VOLUME_MIN);
      lots = MathMax(lots, symbolMinLot);
      lots = MathMin(lots, symbolMaxLot);

      return NormalizeDouble(lots, 2);
     }

   //--- Check if drawdown limit is exceeded
   bool              IsDrawdownExceeded(void)
     {
      double equity  = AccountInfoDouble(ACCOUNT_EQUITY);
      double balance = AccountInfoDouble(ACCOUNT_BALANCE);

      // Use the higher of initial or current balance as reference
      double reference = MathMax(m_initialBalance, balance);

      if(reference <= 0)
         return false;

      double drawdownPercent = ((reference - equity) / reference) * 100.0;
      return (drawdownPercent >= m_maxDrawdownPercent);
     }

   //--- Check if trading is allowed from risk perspective
   bool              CanTrade(void)
     {
      if(IsDrawdownExceeded())
        {
         PrintFormat("[RiskManager] Drawdown limit reached (%.1f%%)", m_maxDrawdownPercent);
         return false;
        }

      // Check free margin
      double freeMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
      if(freeMargin < 100) // Minimum $100 free margin
        {
         Print("[RiskManager] Insufficient free margin");
         return false;
        }

      return true;
     }

   //--- Accessors
   double            RiskPercent(void)        const { return m_riskPercent; }
   double            MaxDrawdownPercent(void) const { return m_maxDrawdownPercent; }
  };

#endif
```

#### Filter Base and Implementations

```mql5
// File: Include/Filters/CFilterBase.mqh
#ifndef CFILTER_BASE_MQH
#define CFILTER_BASE_MQH

//+------------------------------------------------------------------+
//| Abstract filter base                                             |
//+------------------------------------------------------------------+
class CFilterBase
  {
protected:
   string            m_name;
   bool              m_enabled;

public:
                     CFilterBase(void) : m_name("BaseFilter"), m_enabled(true) {}
   virtual          ~CFilterBase(void) {}

   virtual bool      IsAllowed(void) = 0;

   void              SetEnabled(bool enabled) { m_enabled = enabled; }
   bool              IsEnabled(void)    const { return m_enabled; }
   string            Name(void)        const { return m_name; }
  };

#endif
```

```mql5
// File: Include/Filters/CTimeFilter.mqh
#ifndef CTIME_FILTER_MQH
#define CTIME_FILTER_MQH

#include "CFilterBase.mqh"

//+------------------------------------------------------------------+
//| Time/session filter                                              |
//+------------------------------------------------------------------+
class CTimeFilter : public CFilterBase
  {
private:
   int               m_startHour;
   int               m_endHour;
   bool              m_tradeFriday;
   bool              m_tradeSunday;

public:
                     CTimeFilter(void) : m_startHour(8), m_endHour(20),
                                         m_tradeFriday(true), m_tradeSunday(false)
     {
      m_name = "TimeFilter";
     }

   void              SetHours(int startHour, int endHour)
     {
      m_startHour = startHour;
      m_endHour   = endHour;
     }

   void              SetDays(bool tradeFriday, bool tradeSunday)
     {
      m_tradeFriday = tradeFriday;
      m_tradeSunday = tradeSunday;
     }

   virtual bool      IsAllowed(void) override
     {
      if(!m_enabled)
         return true;

      MqlDateTime dt;
      TimeCurrent(dt);

      // Day-of-week filter (0=Sunday, 5=Friday)
      if(dt.day_of_week == 0 && !m_tradeSunday) return false;
      if(dt.day_of_week == 5 && !m_tradeFriday) return false;
      if(dt.day_of_week == 6)                   return false; // Saturday

      // Hour filter
      if(m_startHour < m_endHour)
        {
         // Normal range: e.g. 8-20
         return (dt.hour >= m_startHour && dt.hour < m_endHour);
        }
      else
        {
         // Overnight range: e.g. 22-6
         return (dt.hour >= m_startHour || dt.hour < m_endHour);
        }
     }
  };

#endif
```

```mql5
// File: Include/Filters/CSpreadFilter.mqh
#ifndef CSPREAD_FILTER_MQH
#define CSPREAD_FILTER_MQH

#include "CFilterBase.mqh"

//+------------------------------------------------------------------+
//| Spread filter                                                    |
//+------------------------------------------------------------------+
class CSpreadFilter : public CFilterBase
  {
private:
   string            m_symbol;
   double            m_maxSpreadPips;

public:
                     CSpreadFilter(void) : m_symbol(_Symbol), m_maxSpreadPips(3.0)
     {
      m_name = "SpreadFilter";
     }

   void              SetParameters(string symbol, double maxSpreadPips)
     {
      m_symbol         = symbol;
      m_maxSpreadPips  = maxSpreadPips;
     }

   virtual bool      IsAllowed(void) override
     {
      if(!m_enabled)
         return true;

      long spreadPoints = SymbolInfoInteger(m_symbol, SYMBOL_SPREAD);
      double point      = SymbolInfoDouble(m_symbol, SYMBOL_POINT);
      int digits        = (int)SymbolInfoInteger(m_symbol, SYMBOL_DIGITS);
      double pipSize    = (digits == 3 || digits == 5) ? point * 10 : point;

      double spreadPips = spreadPoints * point / pipSize;

      return (spreadPips <= m_maxSpreadPips);
     }
  };

#endif
```

```mql5
// File: Include/Filters/CVolatilityFilter.mqh
#ifndef CVOLATILITY_FILTER_MQH
#define CVOLATILITY_FILTER_MQH

#include "CFilterBase.mqh"

//+------------------------------------------------------------------+
//| ATR-based volatility filter                                      |
//+------------------------------------------------------------------+
class CVolatilityFilter : public CFilterBase
  {
private:
   string            m_symbol;
   ENUM_TIMEFRAMES   m_timeframe;
   int               m_atrPeriod;
   double            m_minATRPips;
   double            m_maxATRPips;
   int               m_handleATR;

public:
                     CVolatilityFilter(void) : m_symbol(_Symbol), m_timeframe(PERIOD_CURRENT),
                                               m_atrPeriod(14),
                                               m_minATRPips(5.0), m_maxATRPips(50.0),
                                               m_handleATR(INVALID_HANDLE)
     {
      m_name = "VolatilityFilter";
     }

                    ~CVolatilityFilter(void)
     {
      if(m_handleATR != INVALID_HANDLE)
         IndicatorRelease(m_handleATR);
     }

   bool              Init(string symbol, ENUM_TIMEFRAMES timeframe, int atrPeriod,
                          double minATRPips, double maxATRPips)
     {
      m_symbol    = symbol;
      m_timeframe = timeframe;
      m_atrPeriod = atrPeriod;
      m_minATRPips = minATRPips;
      m_maxATRPips = maxATRPips;

      m_handleATR = iATR(m_symbol, m_timeframe, m_atrPeriod);
      return (m_handleATR != INVALID_HANDLE);
     }

   virtual bool      IsAllowed(void) override
     {
      if(!m_enabled)
         return true;

      double atr[];
      ArraySetAsSeries(atr, true);
      if(CopyBuffer(m_handleATR, 0, 1, 1, atr) < 1)
         return false;

      double point   = SymbolInfoDouble(m_symbol, SYMBOL_POINT);
      int digits     = (int)SymbolInfoInteger(m_symbol, SYMBOL_DIGITS);
      double pipSize = (digits == 3 || digits == 5) ? point * 10 : point;

      double atrPips = atr[0] / pipSize;

      return (atrPips >= m_minATRPips && atrPips <= m_maxATRPips);
     }
  };

#endif
```

#### Main EA: Orchestrator

```mql5
// File: Experts/ModularEA/ModularEA.mq5
#property copyright "Developer"
#property version   "1.00"

#include <Core/CSignalMA.mqh>
#include <Core/CTradeManager.mqh>
#include <Core/CRiskManager.mqh>
#include <Filters/CTimeFilter.mqh>
#include <Filters/CSpreadFilter.mqh>
#include <Filters/CVolatilityFilter.mqh>

//--- Input parameters
input int    InpMagicNumber       = 12345;     // Magic Number
input double InpRiskPercent       = 1.0;       // Risk per Trade (%)
input double InpMaxDrawdown       = 20.0;      // Max Drawdown (%)
input int    InpStopLoss          = 50;        // Stop Loss (pips)
input int    InpTakeProfit        = 100;       // Take Profit (pips)
input int    InpFastMA            = 10;        // Fast MA Period
input int    InpSlowMA            = 20;        // Slow MA Period
input int    InpStartHour         = 8;         // Trading Start Hour
input int    InpEndHour           = 20;        // Trading End Hour
input double InpMaxSpread         = 3.0;       // Max Spread (pips)

//--- Module instances
CSignalMA          *g_signal;
CTradeManager      *g_trade;
CRiskManager       *g_risk;
CTimeFilter        *g_timeFilter;
CSpreadFilter      *g_spreadFilter;

//+------------------------------------------------------------------+
//| Expert initialization                                             |
//+------------------------------------------------------------------+
int OnInit()
  {
   //--- Create modules
   g_signal      = new CSignalMA();
   g_trade       = new CTradeManager();
   g_risk        = new CRiskManager();
   g_timeFilter  = new CTimeFilter();
   g_spreadFilter = new CSpreadFilter();

   //--- Initialize signal
   g_signal.SetParameters(InpFastMA, InpSlowMA, MODE_EMA);
   if(!g_signal.Init(_Symbol, PERIOD_CURRENT))
     {
      Print("Signal initialization failed");
      return(INIT_FAILED);
     }

   //--- Initialize trade manager
   if(!g_trade.Init(InpMagicNumber, _Symbol))
     {
      Print("TradeManager initialization failed");
      return(INIT_FAILED);
     }

   //--- Initialize risk manager
   if(!g_risk.Init(InpRiskPercent, InpMaxDrawdown))
     {
      Print("RiskManager initialization failed");
      return(INIT_FAILED);
     }

   //--- Initialize filters
   g_timeFilter.SetHours(InpStartHour, InpEndHour);
   g_spreadFilter.SetParameters(_Symbol, InpMaxSpread);

   PrintFormat("ModularEA initialized on %s", _Symbol);
   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
//| Expert deinitialization                                           |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {
   if(g_signal      != NULL) { g_signal.Release(); delete g_signal; }
   if(g_trade       != NULL) { delete g_trade; }
   if(g_risk        != NULL) { delete g_risk; }
   if(g_timeFilter  != NULL) { delete g_timeFilter; }
   if(g_spreadFilter != NULL) { delete g_spreadFilter; }
  }

//+------------------------------------------------------------------+
//| Expert tick function                                              |
//+------------------------------------------------------------------+
void OnTick()
  {
   // Only process on new bar
   if(!IsNewBar())
      return;

   // Check filters
   if(!g_timeFilter.IsAllowed())   return;
   if(!g_spreadFilter.IsAllowed()) return;

   // Check risk
   if(!g_risk.CanTrade()) return;

   // Skip if already in position
   if(g_trade.HasPosition(_Symbol))
      return;

   // Generate signal
   ENUM_SIGNAL signal = g_signal.GenerateSignal();

   // Execute trade
   if(signal == SIGNAL_BUY)
     {
      double lots = g_risk.CalculateLots(_Symbol, InpStopLoss);
      g_trade.OpenBuy(_Symbol, lots, InpStopLoss, InpTakeProfit, "ModularEA Buy");
     }
   else if(signal == SIGNAL_SELL)
     {
      double lots = g_risk.CalculateLots(_Symbol, InpStopLoss);
      g_trade.OpenSell(_Symbol, lots, InpStopLoss, InpTakeProfit, "ModularEA Sell");
     }
  }

//+------------------------------------------------------------------+
//| Detect new bar                                                   |
//+------------------------------------------------------------------+
bool IsNewBar()
  {
   static datetime lastBarTime = 0;
   datetime currentBarTime = iTime(_Symbol, PERIOD_CURRENT, 0);
   if(currentBarTime != lastBarTime)
     {
      lastBarTime = currentBarTime;
      return true;
     }
   return false;
  }
```

---

### 3. State Machine Pattern

For EAs with complex lifecycle management. Each state has clear entry/exit conditions and transitions.

#### State Transition Diagram

```
                    ┌─────────┐
                    │  IDLE   │
                    └────┬────┘
                         │ OnInit success
                         v
               ┌─────────────────────┐
        ┌──────│ WAITING_FOR_SIGNAL  │<──────────────┐
        │      └─────────┬───────────┘               │
        │                │ Signal detected            │
        │                v                            │
        │      ┌─────────────────────┐               │
        │      │ OPENING_POSITION    │───────────┐   │
        │      └─────────┬───────────┘  Failed   │   │
        │                │ Success                │   │
        │                v                        v   │
        │      ┌─────────────────────┐     ┌─────────┐
        │      │ MANAGING_POSITION   │     │  ERROR  │
        │      └─────────┬───────────┘     └────┬────┘
        │                │ Exit condition        │ Recovery
        │                v                       │
        │      ┌─────────────────────┐          │
        │      │ CLOSING_POSITION    │──────────┘
        │      └─────────┬───────────┘
        │                │ Closed
        └────────────────┘
```

#### State Machine Code

```mql5
//+------------------------------------------------------------------+
//| State Machine EA                                                  |
//+------------------------------------------------------------------+
#property copyright "Developer"
#property version   "1.00"

#include <Trade\Trade.mqh>

//--- EA States
enum ENUM_EA_STATE
  {
   STATE_IDLE              = 0,
   STATE_WAITING_SIGNAL    = 1,
   STATE_OPENING_POSITION  = 2,
   STATE_MANAGING_POSITION = 3,
   STATE_CLOSING_POSITION  = 4,
   STATE_ERROR             = 5
  };

//--- Input parameters
input int    InpMagicNumber = 12345;
input double InpLots        = 0.1;
input int    InpStopLoss    = 50;
input int    InpTakeProfit  = 100;

//--- Globals
CTrade         g_trade;
ENUM_EA_STATE  g_state       = STATE_IDLE;
int            g_pendingSignal = 0;
int            g_errorCount  = 0;
datetime       g_errorTime   = 0;
ulong          g_positionTicket = 0;

//+------------------------------------------------------------------+
int OnInit()
  {
   g_trade.SetExpertMagicNumber(InpMagicNumber);
   g_state = STATE_WAITING_SIGNAL;
   PrintFormat("State -> WAITING_FOR_SIGNAL");
   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
void OnTick()
  {
   switch(g_state)
     {
      case STATE_WAITING_SIGNAL:
         HandleWaitingSignal();
         break;

      case STATE_OPENING_POSITION:
         HandleOpeningPosition();
         break;

      case STATE_MANAGING_POSITION:
         HandleManagingPosition();
         break;

      case STATE_CLOSING_POSITION:
         HandleClosingPosition();
         break;

      case STATE_ERROR:
         HandleError();
         break;

      default:
         g_state = STATE_WAITING_SIGNAL;
         break;
     }
  }

//+------------------------------------------------------------------+
//| Wait for entry signal                                            |
//+------------------------------------------------------------------+
void HandleWaitingSignal()
  {
   if(!IsNewBar())
      return;

   // Check for signal (example: price above/below MA)
   double ma[];
   ArraySetAsSeries(ma, true);
   int handle = iMA(_Symbol, PERIOD_CURRENT, 20, 0, MODE_SMA, PRICE_CLOSE);
   if(CopyBuffer(handle, 0, 1, 1, ma) < 1) return;
   IndicatorRelease(handle);

   double close = iClose(_Symbol, PERIOD_CURRENT, 1);

   if(close > ma[0])
      g_pendingSignal = 1;  // Buy
   else if(close < ma[0])
      g_pendingSignal = -1; // Sell
   else
      return;

   // Transition
   g_state = STATE_OPENING_POSITION;
   PrintFormat("State -> OPENING_POSITION (signal=%d)", g_pendingSignal);
  }

//+------------------------------------------------------------------+
//| Execute the trade                                                |
//+------------------------------------------------------------------+
void HandleOpeningPosition()
  {
   double pipSize = (_Digits == 3 || _Digits == 5) ? _Point * 10 : _Point;
   bool result = false;

   if(g_pendingSignal == 1)
     {
      double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
      double sl  = NormalizeDouble(ask - InpStopLoss * pipSize, _Digits);
      double tp  = NormalizeDouble(ask + InpTakeProfit * pipSize, _Digits);
      result = g_trade.Buy(InpLots, _Symbol, ask, sl, tp);
     }
   else if(g_pendingSignal == -1)
     {
      double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
      double sl  = NormalizeDouble(bid + InpStopLoss * pipSize, _Digits);
      double tp  = NormalizeDouble(bid - InpTakeProfit * pipSize, _Digits);
      result = g_trade.Sell(InpLots, _Symbol, bid, sl, tp);
     }

   if(result && (g_trade.ResultRetcode() == TRADE_RETCODE_DONE ||
                 g_trade.ResultRetcode() == TRADE_RETCODE_PLACED))
     {
      g_positionTicket = g_trade.ResultDeal();
      g_state = STATE_MANAGING_POSITION;
      PrintFormat("State -> MANAGING_POSITION (ticket=%d)", g_positionTicket);
     }
   else
     {
      g_errorCount++;
      g_errorTime = TimeCurrent();
      g_state = STATE_ERROR;
      PrintFormat("State -> ERROR (retcode=%d)", g_trade.ResultRetcode());
     }
  }

//+------------------------------------------------------------------+
//| Manage open position (trailing, partial close, etc.)             |
//+------------------------------------------------------------------+
void HandleManagingPosition()
  {
   // Check if position still exists
   bool positionExists = false;
   for(int i = PositionsTotal() - 1; i >= 0; i--)
     {
      ulong ticket = PositionGetTicket(i);
      if(PositionGetInteger(POSITION_MAGIC) == InpMagicNumber &&
         PositionGetString(POSITION_SYMBOL) == _Symbol)
        {
         positionExists = true;
         break;
        }
     }

   if(!positionExists)
     {
      // Position was closed (SL/TP hit)
      g_state = STATE_WAITING_SIGNAL;
      PrintFormat("State -> WAITING_FOR_SIGNAL (position closed by SL/TP)");
      return;
     }

   // Add trailing stop, break-even, partial close logic here
  }

//+------------------------------------------------------------------+
//| Close position                                                   |
//+------------------------------------------------------------------+
void HandleClosingPosition()
  {
   for(int i = PositionsTotal() - 1; i >= 0; i--)
     {
      ulong ticket = PositionGetTicket(i);
      if(PositionGetInteger(POSITION_MAGIC) == InpMagicNumber &&
         PositionGetString(POSITION_SYMBOL) == _Symbol)
        {
         if(g_trade.PositionClose(ticket))
           {
            g_state = STATE_WAITING_SIGNAL;
            PrintFormat("State -> WAITING_FOR_SIGNAL (position closed)");
            return;
           }
         else
           {
            g_state = STATE_ERROR;
            PrintFormat("State -> ERROR (close failed)");
            return;
           }
        }
     }

   // No position found to close
   g_state = STATE_WAITING_SIGNAL;
  }

//+------------------------------------------------------------------+
//| Error recovery                                                   |
//+------------------------------------------------------------------+
void HandleError()
  {
   // Wait 30 seconds before retry
   if(TimeCurrent() - g_errorTime < 30)
      return;

   if(g_errorCount >= 5)
     {
      PrintFormat("[ERROR] Max retries reached. EA stopped.");
      ExpertRemove();
      return;
     }

   // Reset to waiting state
   g_state = STATE_WAITING_SIGNAL;
   PrintFormat("State -> WAITING_FOR_SIGNAL (error recovery, count=%d)", g_errorCount);
  }

//+------------------------------------------------------------------+
bool IsNewBar()
  {
   static datetime lastBarTime = 0;
   datetime currentBarTime = iTime(_Symbol, PERIOD_CURRENT, 0);
   if(currentBarTime != lastBarTime)
     {
      lastBarTime = currentBarTime;
      return true;
     }
   return false;
  }
```

---

### 4. Multi-Timeframe Analysis

Check signals on higher timeframes and execute on lower timeframes for better precision.

#### MQL4 Multi-Timeframe

```mql4
//+------------------------------------------------------------------+
//| Multi-Timeframe Signal - MQL4                                    |
//| Chart: M15, Signal confirmation: H4                              |
//+------------------------------------------------------------------+
#property strict

input int InpMAPeriod = 20;

int GetMultiTFSignal()
  {
   //--- Higher timeframe trend (H4)
   double h4_ma_1 = iMA(NULL, PERIOD_H4, InpMAPeriod, 0, MODE_EMA, PRICE_CLOSE, 1);
   double h4_ma_2 = iMA(NULL, PERIOD_H4, InpMAPeriod, 0, MODE_EMA, PRICE_CLOSE, 2);
   double h4_close = iClose(NULL, PERIOD_H4, 1);

   int h4Trend = 0;
   if(h4_close > h4_ma_1 && h4_ma_1 > h4_ma_2)
      h4Trend = 1;   // Bullish trend on H4
   else if(h4_close < h4_ma_1 && h4_ma_1 < h4_ma_2)
      h4Trend = -1;  // Bearish trend on H4

   //--- Entry timeframe signal (M15 - current chart)
   double m15_fast_1 = iMA(NULL, PERIOD_M15, 10, 0, MODE_EMA, PRICE_CLOSE, 1);
   double m15_fast_2 = iMA(NULL, PERIOD_M15, 10, 0, MODE_EMA, PRICE_CLOSE, 2);
   double m15_slow_1 = iMA(NULL, PERIOD_M15, 20, 0, MODE_EMA, PRICE_CLOSE, 1);
   double m15_slow_2 = iMA(NULL, PERIOD_M15, 20, 0, MODE_EMA, PRICE_CLOSE, 2);

   // Buy: H4 bullish + M15 bullish crossover
   if(h4Trend == 1 && m15_fast_2 <= m15_slow_2 && m15_fast_1 > m15_slow_1)
      return 1;

   // Sell: H4 bearish + M15 bearish crossover
   if(h4Trend == -1 && m15_fast_2 >= m15_slow_2 && m15_fast_1 < m15_slow_1)
      return -1;

   return 0;
  }
```

#### MQL5 Multi-Timeframe

```mql5
//+------------------------------------------------------------------+
//| Multi-Timeframe Signal - MQL5                                    |
//| Uses indicator handles per timeframe                             |
//+------------------------------------------------------------------+
class CSignalMultiTF
  {
private:
   int               m_handleH4_MA;    // Trend MA on H4
   int               m_handleM15_Fast; // Entry fast MA on M15
   int               m_handleM15_Slow; // Entry slow MA on M15
   string            m_symbol;

public:
                     CSignalMultiTF(void) : m_handleH4_MA(INVALID_HANDLE),
                                            m_handleM15_Fast(INVALID_HANDLE),
                                            m_handleM15_Slow(INVALID_HANDLE) {}

                    ~CSignalMultiTF(void) { Release(); }

   bool              Init(string symbol)
     {
      m_symbol = symbol;

      // Create handles for different timeframes
      m_handleH4_MA    = iMA(m_symbol, PERIOD_H4, 20, 0, MODE_EMA, PRICE_CLOSE);
      m_handleM15_Fast = iMA(m_symbol, PERIOD_M15, 10, 0, MODE_EMA, PRICE_CLOSE);
      m_handleM15_Slow = iMA(m_symbol, PERIOD_M15, 20, 0, MODE_EMA, PRICE_CLOSE);

      if(m_handleH4_MA == INVALID_HANDLE ||
         m_handleM15_Fast == INVALID_HANDLE ||
         m_handleM15_Slow == INVALID_HANDLE)
        {
         Print("Failed to create MTF indicator handles");
         return false;
        }

      return true;
     }

   int               GetSignal(void)
     {
      //--- Get H4 trend data
      double h4MA[];
      ArraySetAsSeries(h4MA, true);
      if(CopyBuffer(m_handleH4_MA, 0, 0, 3, h4MA) < 3) return 0;

      // Also need H4 close
      double h4Close[];
      ArraySetAsSeries(h4Close, true);
      if(CopyClose(m_symbol, PERIOD_H4, 0, 2, h4Close) < 2) return 0;

      int h4Trend = 0;
      if(h4Close[1] > h4MA[1] && h4MA[1] > h4MA[2])
         h4Trend = 1;
      else if(h4Close[1] < h4MA[1] && h4MA[1] < h4MA[2])
         h4Trend = -1;

      if(h4Trend == 0)
         return 0;

      //--- Get M15 crossover data
      double fast[], slow[];
      ArraySetAsSeries(fast, true);
      ArraySetAsSeries(slow, true);
      if(CopyBuffer(m_handleM15_Fast, 0, 0, 3, fast) < 3) return 0;
      if(CopyBuffer(m_handleM15_Slow, 0, 0, 3, slow) < 3) return 0;

      // Buy: H4 up + M15 bullish cross
      if(h4Trend == 1 && fast[2] <= slow[2] && fast[1] > slow[1])
         return 1;

      // Sell: H4 down + M15 bearish cross
      if(h4Trend == -1 && fast[2] >= slow[2] && fast[1] < slow[1])
         return -1;

      return 0;
     }

   void              Release(void)
     {
      if(m_handleH4_MA    != INVALID_HANDLE) { IndicatorRelease(m_handleH4_MA);    m_handleH4_MA    = INVALID_HANDLE; }
      if(m_handleM15_Fast != INVALID_HANDLE) { IndicatorRelease(m_handleM15_Fast); m_handleM15_Fast = INVALID_HANDLE; }
      if(m_handleM15_Slow != INVALID_HANDLE) { IndicatorRelease(m_handleM15_Slow); m_handleM15_Slow = INVALID_HANDLE; }
     }
  };
```

---

### 5. Multi-Symbol EA

Trade multiple symbols from a single EA instance. Uses `OnTimer` instead of `OnTick` because `OnTick` only fires for the chart symbol.

```mql5
//+------------------------------------------------------------------+
//| Multi-Symbol EA                                                   |
//| Uses OnTimer for symbol-independent processing                    |
//+------------------------------------------------------------------+
#property copyright "Developer"
#property version   "1.00"

#include <Trade\Trade.mqh>

//--- Input parameters
input string InpSymbols       = "EURUSD,GBPUSD,USDJPY,AUDUSD"; // Symbols (comma-separated)
input int    InpMagicNumber   = 55555;
input double InpLots          = 0.1;
input int    InpTimerSeconds  = 5;  // Check interval (seconds)

//--- Symbol management
string g_symbols[];
int    g_symbolCount;
CTrade g_trade;

//--- Per-symbol indicator handles
int    g_handleMA[];  // One handle per symbol

//+------------------------------------------------------------------+
int OnInit()
  {
   // Parse symbol list
   g_symbolCount = StringSplit(InpSymbols, ',', g_symbols);
   if(g_symbolCount <= 0)
     {
      Print("No symbols specified");
      return(INIT_FAILED);
     }

   // Trim whitespace from symbol names
   for(int i = 0; i < g_symbolCount; i++)
      StringTrimLeft(StringTrimRight(g_symbols[i]));

   // Validate symbols and select them in Market Watch
   for(int i = 0; i < g_symbolCount; i++)
     {
      if(!SymbolSelect(g_symbols[i], true))
        {
         PrintFormat("Symbol %s not available", g_symbols[i]);
         return(INIT_FAILED);
        }
     }

   // Create indicator handles for each symbol
   ArrayResize(g_handleMA, g_symbolCount);
   for(int i = 0; i < g_symbolCount; i++)
     {
      g_handleMA[i] = iMA(g_symbols[i], PERIOD_H1, 20, 0, MODE_EMA, PRICE_CLOSE);
      if(g_handleMA[i] == INVALID_HANDLE)
        {
         PrintFormat("Failed to create MA handle for %s", g_symbols[i]);
         return(INIT_FAILED);
        }
     }

   // Configure trade
   g_trade.SetExpertMagicNumber(InpMagicNumber);
   g_trade.SetDeviationInPoints(10);

   // Start timer
   if(!EventSetTimer(InpTimerSeconds))
     {
      Print("Failed to set timer");
      return(INIT_FAILED);
     }

   PrintFormat("Multi-Symbol EA started with %d symbols", g_symbolCount);
   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {
   EventKillTimer();

   for(int i = 0; i < g_symbolCount; i++)
     {
      if(g_handleMA[i] != INVALID_HANDLE)
         IndicatorRelease(g_handleMA[i]);
     }
  }

//+------------------------------------------------------------------+
//| Timer event: process all symbols                                 |
//+------------------------------------------------------------------+
void OnTimer()
  {
   for(int i = 0; i < g_symbolCount; i++)
     {
      ProcessSymbol(g_symbols[i], g_handleMA[i]);
     }
  }

//+------------------------------------------------------------------+
//| Process one symbol                                               |
//+------------------------------------------------------------------+
void ProcessSymbol(string symbol, int maHandle)
  {
   // Check for new bar on this symbol
   if(!IsNewBarForSymbol(symbol))
      return;

   // Skip if already in position for this symbol
   if(HasPositionForSymbol(symbol))
      return;

   // Generate signal
   double ma[], close[];
   ArraySetAsSeries(ma, true);
   ArraySetAsSeries(close, true);

   if(CopyBuffer(maHandle, 0, 0, 3, ma) < 3) return;
   if(CopyClose(symbol, PERIOD_H1, 0, 3, close) < 3) return;

   // Set filling policy per symbol
   long filling = SymbolInfoInteger(symbol, SYMBOL_FILLING_MODE);
   if((filling & SYMBOL_FILLING_FOK) == SYMBOL_FILLING_FOK)
      g_trade.SetTypeFilling(ORDER_FILLING_FOK);
   else if((filling & SYMBOL_FILLING_IOC) == SYMBOL_FILLING_IOC)
      g_trade.SetTypeFilling(ORDER_FILLING_IOC);
   else
      g_trade.SetTypeFilling(ORDER_FILLING_RETURN);

   // Simple signal: price crosses above/below MA
   if(close[2] <= ma[2] && close[1] > ma[1])
     {
      double ask = SymbolInfoDouble(symbol, SYMBOL_ASK);
      g_trade.Buy(InpLots, symbol, ask, 0, 0, "MultiSym Buy");
      PrintFormat("[%s] BUY signal executed", symbol);
     }
   else if(close[2] >= ma[2] && close[1] < ma[1])
     {
      double bid = SymbolInfoDouble(symbol, SYMBOL_BID);
      g_trade.Sell(InpLots, symbol, bid, 0, 0, "MultiSym Sell");
      PrintFormat("[%s] SELL signal executed", symbol);
     }
  }

//+------------------------------------------------------------------+
//| New bar detection per symbol                                     |
//+------------------------------------------------------------------+
bool IsNewBarForSymbol(string symbol)
  {
   // Use a static array to track bar times per symbol
   static datetime barTimes[];
   static string   barSymbols[];
   static int      barCount = 0;

   datetime currentBar = iTime(symbol, PERIOD_H1, 0);

   // Find existing entry
   for(int i = 0; i < barCount; i++)
     {
      if(barSymbols[i] == symbol)
        {
         if(barTimes[i] != currentBar)
           {
            barTimes[i] = currentBar;
            return true;
           }
         return false;
        }
     }

   // New symbol: add entry
   barCount++;
   ArrayResize(barTimes, barCount);
   ArrayResize(barSymbols, barCount);
   barTimes[barCount - 1]   = currentBar;
   barSymbols[barCount - 1] = symbol;
   return true;
  }

//+------------------------------------------------------------------+
//| Check for existing position on a symbol                          |
//+------------------------------------------------------------------+
bool HasPositionForSymbol(string symbol)
  {
   for(int i = PositionsTotal() - 1; i >= 0; i--)
     {
      ulong ticket = PositionGetTicket(i);
      if(PositionGetString(POSITION_SYMBOL) == symbol &&
         PositionGetInteger(POSITION_MAGIC) == InpMagicNumber)
         return true;
     }
   return false;
  }
```

---

## Design Patterns for MQL

### Singleton Pattern (for Managers)

Ensure only one instance of a manager class exists. Useful for trade managers, loggers, or config objects.

```mql5
//+------------------------------------------------------------------+
//| Singleton Trade Manager                                          |
//+------------------------------------------------------------------+
class CTradeManagerSingleton
  {
private:
   static CTradeManagerSingleton *m_instance;
   CTrade            m_trade;
   int               m_magicNumber;

   // Private constructor prevents direct instantiation
                     CTradeManagerSingleton(void) : m_magicNumber(0) {}
                    ~CTradeManagerSingleton(void) {}

public:
   //--- Get the single instance
   static CTradeManagerSingleton* GetInstance(void)
     {
      if(m_instance == NULL)
         m_instance = new CTradeManagerSingleton();
      return m_instance;
     }

   //--- Destroy the instance (call in OnDeinit)
   static void       DestroyInstance(void)
     {
      if(m_instance != NULL)
        {
         delete m_instance;
         m_instance = NULL;
        }
     }

   //--- Public interface
   void              Init(int magicNumber)
     {
      m_magicNumber = magicNumber;
      m_trade.SetExpertMagicNumber(m_magicNumber);
     }

   bool              Buy(string symbol, double lots, double sl, double tp)
     {
      double ask = SymbolInfoDouble(symbol, SYMBOL_ASK);
      return m_trade.Buy(lots, symbol, ask, sl, tp);
     }

   bool              Sell(string symbol, double lots, double sl, double tp)
     {
      double bid = SymbolInfoDouble(symbol, SYMBOL_BID);
      return m_trade.Sell(lots, symbol, bid, sl, tp);
     }
  };

// Static member initialization (must be at file scope)
CTradeManagerSingleton* CTradeManagerSingleton::m_instance = NULL;

// Usage:
// OnInit:  CTradeManagerSingleton::GetInstance().Init(12345);
// OnTick:  CTradeManagerSingleton::GetInstance().Buy("EURUSD", 0.1, sl, tp);
// OnDeinit: CTradeManagerSingleton::DestroyInstance();
```

### Strategy Pattern (Interchangeable Signals)

Swap signal algorithms at runtime or configuration time without modifying the EA.

```mql5
//+------------------------------------------------------------------+
//| Strategy Pattern: interchangeable signal implementations         |
//+------------------------------------------------------------------+

// Base (already defined in CSignalBase.mqh above)
// class CSignalBase { virtual ENUM_SIGNAL GenerateSignal() = 0; };

// Implementation 1: MA Crossover (see CSignalMA above)

// Implementation 2: RSI Overbought/Oversold
class CSignalRSI : public CSignalBase
  {
private:
   int               m_period;
   double            m_overbought;
   double            m_oversold;
   int               m_handleRSI;

public:
                     CSignalRSI(void) : m_period(14), m_overbought(70), m_oversold(30),
                                        m_handleRSI(INVALID_HANDLE)
     {
      m_name = "RSI_Signal";
     }

                    ~CSignalRSI(void) { Release(); }

   void              SetParameters(int period, double overbought, double oversold)
     {
      m_period     = period;
      m_overbought = overbought;
      m_oversold   = oversold;
     }

   virtual bool      Init(string symbol, ENUM_TIMEFRAMES timeframe) override
     {
      if(!CSignalBase::Init(symbol, timeframe))
         return false;

      m_handleRSI = iRSI(m_symbol, m_timeframe, m_period, PRICE_CLOSE);
      return (m_handleRSI != INVALID_HANDLE);
     }

   virtual ENUM_SIGNAL GenerateSignal(void) override
     {
      double rsi[];
      ArraySetAsSeries(rsi, true);
      if(CopyBuffer(m_handleRSI, 0, 0, 3, rsi) < 3) return SIGNAL_NO;

      // RSI crosses above oversold -> Buy
      if(rsi[2] <= m_oversold && rsi[1] > m_oversold)
         return SIGNAL_BUY;

      // RSI crosses below overbought -> Sell
      if(rsi[2] >= m_overbought && rsi[1] < m_overbought)
         return SIGNAL_SELL;

      return SIGNAL_NO;
     }

   virtual void      Release(void) override
     {
      if(m_handleRSI != INVALID_HANDLE) { IndicatorRelease(m_handleRSI); m_handleRSI = INVALID_HANDLE; }
     }
  };

// --- EA usage: select strategy via input parameter ---
// input int InpSignalType = 0;  // 0=MA, 1=RSI
//
// CSignalBase *g_signal;
//
// int OnInit() {
//    if(InpSignalType == 0) {
//       CSignalMA *sig = new CSignalMA();
//       sig.SetParameters(10, 20, MODE_EMA);
//       g_signal = sig;
//    } else {
//       CSignalRSI *sig = new CSignalRSI();
//       sig.SetParameters(14, 70, 30);
//       g_signal = sig;
//    }
//    g_signal.Init(_Symbol, PERIOD_CURRENT);
// }
```

### Observer Pattern (Event Notification)

Useful for inter-module communication: when a trade is opened, notify the logger, the panel, the risk manager, etc.

```mql5
//+------------------------------------------------------------------+
//| Observer Pattern for trade events                                |
//+------------------------------------------------------------------+

//--- Event types
enum ENUM_TRADE_EVENT
  {
   EVENT_POSITION_OPENED  = 0,
   EVENT_POSITION_CLOSED  = 1,
   EVENT_ORDER_FAILED     = 2,
   EVENT_SL_HIT           = 3,
   EVENT_TP_HIT           = 4
  };

//--- Observer interface
class ITradeObserver
  {
public:
   virtual          ~ITradeObserver(void) {}
   virtual void      OnTradeEvent(ENUM_TRADE_EVENT event, string symbol,
                                  double price, double lots) = 0;
  };

//--- Subject: manages observers and notifies them
class CTradeEventPublisher
  {
private:
   ITradeObserver   *m_observers[];
   int               m_count;

public:
                     CTradeEventPublisher(void) : m_count(0) {}
                    ~CTradeEventPublisher(void) {}

   void              Subscribe(ITradeObserver *observer)
     {
      m_count++;
      ArrayResize(m_observers, m_count);
      m_observers[m_count - 1] = observer;
     }

   void              Notify(ENUM_TRADE_EVENT event, string symbol,
                            double price, double lots)
     {
      for(int i = 0; i < m_count; i++)
        {
         if(m_observers[i] != NULL)
            m_observers[i].OnTradeEvent(event, symbol, price, lots);
        }
     }
  };

//--- Concrete observer: Logger
class CTradeLogger : public ITradeObserver
  {
public:
   virtual void      OnTradeEvent(ENUM_TRADE_EVENT event, string symbol,
                                  double price, double lots) override
     {
      string eventName;
      switch(event)
        {
         case EVENT_POSITION_OPENED: eventName = "OPENED";  break;
         case EVENT_POSITION_CLOSED: eventName = "CLOSED";  break;
         case EVENT_ORDER_FAILED:    eventName = "FAILED";  break;
         case EVENT_SL_HIT:          eventName = "SL_HIT";  break;
         case EVENT_TP_HIT:          eventName = "TP_HIT";  break;
         default:                    eventName = "UNKNOWN"; break;
        }
      PrintFormat("[TradeLog] %s %s price=%.5f lots=%.2f", eventName, symbol, price, lots);
     }
  };

//--- Concrete observer: Stats tracker
class CTradeStats : public ITradeObserver
  {
private:
   int               m_totalTrades;
   int               m_wins;
   int               m_losses;

public:
                     CTradeStats(void) : m_totalTrades(0), m_wins(0), m_losses(0) {}

   virtual void      OnTradeEvent(ENUM_TRADE_EVENT event, string symbol,
                                  double price, double lots) override
     {
      if(event == EVENT_POSITION_OPENED) m_totalTrades++;
      if(event == EVENT_TP_HIT)          m_wins++;
      if(event == EVENT_SL_HIT)          m_losses++;
     }

   double            WinRate(void) const
     {
      int completed = m_wins + m_losses;
      return (completed > 0) ? (double)m_wins / completed * 100.0 : 0;
     }
  };

// --- Usage ---
// CTradeEventPublisher g_publisher;
// CTradeLogger         g_logger;
// CTradeStats          g_stats;
//
// OnInit:
//   g_publisher.Subscribe(&g_logger);
//   g_publisher.Subscribe(&g_stats);
//
// After trade execution:
//   g_publisher.Notify(EVENT_POSITION_OPENED, "EURUSD", 1.10500, 0.1);
```

---

## Include File Design (.mqh)

### Header Guard Pattern

Prevents double-inclusion compilation errors. Required for every `.mqh` file.

```mql5
#ifndef MY_CLASS_MQH
#define MY_CLASS_MQH

// Class definition here

#endif // MY_CLASS_MQH
```

Convention: guard name = filename in uppercase with dots replaced by underscores.
- `CTradeManager.mqh` -> `CTRADE_MANAGER_MQH`
- `CSignalBase.mqh` -> `CSIGNAL_BASE_MQH`

### Class Design Conventions

| Convention | Rule | Example |
|------------|------|---------|
| One class per file | Class name matches filename | `CTradeManager` in `CTradeManager.mqh` |
| Class prefix | `C` for classes | `CTradeManager`, `CRiskManager` |
| Interface prefix | `I` for interfaces / abstract bases | `ITradeObserver` |
| Enum prefix | `ENUM_` for enums | `ENUM_SIGNAL`, `ENUM_EA_STATE` |
| Member variables | `m_` prefix | `m_magicNumber`, `m_symbol` |
| Input parameters | Only at EA level | Never inside `.mqh` classes |
| Configuration | Pass via constructor or `Init()` method | `Init(symbol, timeframe)` |
| Destruction | Release handles in destructor | `~CSignalMA() { Release(); }` |

### Input Parameters Rule

Input parameters (`input`) should only appear in the main `.mq5`/`.mq4` EA file. Classes receive their configuration through constructor parameters or `Init()` methods:

```mql5
// CORRECT: input at EA level, passed to class
// In MyEA.mq5:
input int InpFastMA = 10;

int OnInit()
  {
   CSignalMA *signal = new CSignalMA();
   signal.SetParameters(InpFastMA, 20, MODE_EMA);  // Config passed in
   signal.Init(_Symbol, PERIOD_CURRENT);
  }

// WRONG: input inside .mqh class
// class CSignalMA {
//    input int m_fastPeriod = 10;  // DO NOT do this
// };
```

---

## Complete Templates

### Modular EA Composition Template (MQL5)

Shows the recommended way to compose a modular EA from reusable components.

```mql5
//+------------------------------------------------------------------+
//|                                            ModularTemplate.mq5   |
//| Template showing modular EA composition                          |
//+------------------------------------------------------------------+
#property copyright "Developer"
#property version   "1.00"

//--- Include modules
#include <Core/CSignalBase.mqh>
#include <Core/CSignalMA.mqh>
#include <Core/CTradeManager.mqh>
#include <Core/CRiskManager.mqh>
#include <Filters/CTimeFilter.mqh>
#include <Filters/CSpreadFilter.mqh>
#include <Filters/CVolatilityFilter.mqh>

//--- Input parameters: Strategy
input string          InpSection1      = "=== Strategy ===";  // --------
input int             InpFastMA        = 10;           // Fast MA Period
input int             InpSlowMA        = 20;           // Slow MA Period

//--- Input parameters: Risk
input string          InpSection2      = "=== Risk ===";      // --------
input double          InpRiskPercent   = 1.0;          // Risk per Trade (%)
input double          InpMaxDrawdown   = 20.0;         // Max Drawdown (%)
input int             InpStopLoss      = 50;           // Stop Loss (pips)
input int             InpTakeProfit    = 100;          // Take Profit (pips)

//--- Input parameters: Filters
input string          InpSection3      = "=== Filters ===";   // --------
input int             InpStartHour     = 8;            // Start Hour (server time)
input int             InpEndHour       = 20;           // End Hour (server time)
input double          InpMaxSpread     = 3.0;          // Max Spread (pips)

//--- Input parameters: General
input string          InpSection4      = "=== General ===";   // --------
input int             InpMagicNumber   = 12345;        // Magic Number

//--- Module pointers
CSignalMA             *g_signal;
CTradeManager         *g_trade;
CRiskManager          *g_risk;
CTimeFilter           *g_timeFilter;
CSpreadFilter         *g_spreadFilter;

//+------------------------------------------------------------------+
//| Expert initialization                                             |
//+------------------------------------------------------------------+
int OnInit()
  {
   //--- Allocate modules
   g_signal       = new CSignalMA();
   g_trade        = new CTradeManager();
   g_risk         = new CRiskManager();
   g_timeFilter   = new CTimeFilter();
   g_spreadFilter = new CSpreadFilter();

   //--- Configure and initialize Signal
   g_signal.SetParameters(InpFastMA, InpSlowMA, MODE_EMA);
   if(!g_signal.Init(_Symbol, PERIOD_CURRENT))
     {
      Print("INIT FAILED: Signal module");
      return(INIT_FAILED);
     }

   //--- Configure and initialize Trade Manager
   if(!g_trade.Init(InpMagicNumber, _Symbol))
     {
      Print("INIT FAILED: Trade module");
      return(INIT_FAILED);
     }

   //--- Configure and initialize Risk Manager
   if(!g_risk.Init(InpRiskPercent, InpMaxDrawdown))
     {
      Print("INIT FAILED: Risk module");
      return(INIT_FAILED);
     }

   //--- Configure Filters
   g_timeFilter.SetHours(InpStartHour, InpEndHour);
   g_spreadFilter.SetParameters(_Symbol, InpMaxSpread);

   //--- Ready
   PrintFormat("EA initialized: %s | Magic=%d | Risk=%.1f%%",
               _Symbol, InpMagicNumber, InpRiskPercent);
   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
//| Expert deinitialization                                           |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {
   //--- Release and delete modules (reverse order of creation)
   if(g_spreadFilter != NULL) { delete g_spreadFilter; g_spreadFilter = NULL; }
   if(g_timeFilter   != NULL) { delete g_timeFilter;   g_timeFilter   = NULL; }
   if(g_risk         != NULL) { delete g_risk;         g_risk         = NULL; }
   if(g_trade        != NULL) { delete g_trade;        g_trade        = NULL; }
   if(g_signal       != NULL) { g_signal.Release(); delete g_signal; g_signal = NULL; }
  }

//+------------------------------------------------------------------+
//| Expert tick function                                              |
//+------------------------------------------------------------------+
void OnTick()
  {
   //--- Step 1: Only on new bar
   if(!IsNewBar())
      return;

   //--- Step 2: Check filters
   if(!g_timeFilter.IsAllowed())
     return;

   if(!g_spreadFilter.IsAllowed())
     return;

   //--- Step 3: Check risk
   if(!g_risk.CanTrade())
     return;

   //--- Step 4: Skip if already in position
   if(g_trade.HasPosition(_Symbol))
     return;

   //--- Step 5: Generate signal
   ENUM_SIGNAL signal = g_signal.GenerateSignal();
   if(signal == SIGNAL_NO)
     return;

   //--- Step 6: Calculate position size
   double lots = g_risk.CalculateLots(_Symbol, InpStopLoss);

   //--- Step 7: Execute trade
   if(signal == SIGNAL_BUY)
      g_trade.OpenBuy(_Symbol, lots, InpStopLoss, InpTakeProfit, "Buy Signal");
   else if(signal == SIGNAL_SELL)
      g_trade.OpenSell(_Symbol, lots, InpStopLoss, InpTakeProfit, "Sell Signal");
  }

//+------------------------------------------------------------------+
//| Detect new bar                                                   |
//+------------------------------------------------------------------+
bool IsNewBar()
  {
   static datetime lastBarTime = 0;
   datetime currentBarTime = iTime(_Symbol, PERIOD_CURRENT, 0);
   if(currentBarTime != lastBarTime)
     {
      lastBarTime = currentBarTime;
      return true;
     }
   return false;
  }
```

---

## Quick Reference: When to Use Each Pattern

| Scenario | Pattern |
|----------|---------|
| Simple strategy, single indicator, quick prototype | Simple Single-File EA |
| Professional EA, multiple signals, reusable code | Modular EA (Signal + Trade + Risk + Filter) |
| Complex lifecycle, multiple phases, error recovery | State Machine |
| Trend on H4, entry on M15 | Multi-Timeframe Analysis |
| Trade EURUSD + GBPUSD + USDJPY from one EA | Multi-Symbol EA |
| Global managers (trade, logger) | Singleton |
| Swap signal algorithms without changing EA | Strategy |
| Decouple modules, event-driven updates | Observer |
