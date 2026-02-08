# MQL5 Language Reference

Comprehensive reference for MQL5 development on MetaTrader 5.

## Table of Contents

- [Key Differences from MQL4](#key-differences-from-mql4)
- [OOP Features](#oop-features)
- [Trade Functions](#trade-functions)
- [Event Handlers](#event-handlers)
- [Standard Library](#standard-library)
- [Key Enumerations](#key-enumerations)
- [MQL5-Specific Features](#mql5-specific-features)
- [MQL5 vs MQL4 Summary Table](#mql5-vs-mql4-summary-table)

---

## Key Differences from MQL4

### Architectural Paradigm

MQL5 is a full object-oriented language (C++-like) compared to MQL4's procedural (C-like) approach. The most fundamental difference is the trade model:

| Concept | Description |
|---------|-------------|
| **Order** | A trade request sent to the server. Can be market (executed immediately) or pending (waits for price). |
| **Deal** | An executed trade operation resulting from a filled order. Recorded in history. |
| **Position** | The net result of one or more deals on a symbol. This is the current open exposure. |

### Account Models

| Model | Behavior | Notes |
|-------|----------|-------|
| **Netting** | One position per symbol. New deals increase/decrease/reverse the single position. | Default for most brokers. `ACCOUNT_MARGIN_MODE_RETAIL_NETTING`. |
| **Hedging** | Multiple independent positions per symbol. Each has its own ticket. | Closer to MQL4 behavior. `ACCOUNT_MARGIN_MODE_RETAIL_HEDGING`. |

Detect at runtime:

```mql5
ENUM_ACCOUNT_MARGIN_MODE marginMode = (ENUM_ACCOUNT_MARGIN_MODE)AccountInfoInteger(ACCOUNT_MARGIN_MODE);
if(marginMode == ACCOUNT_MARGIN_MODE_RETAIL_HEDGING)
   Print("Hedging account");
```

---

## OOP Features

### Classes

```mql5
class CMyClass
{
private:
   int    m_id;
   double m_value;
   string m_name;

protected:
   double CalculateInternal() const { return m_value * 2.0; }

public:
   // Default constructor
   CMyClass() : m_id(0), m_value(0.0), m_name("") {}

   // Parametric constructor with initialization list
   CMyClass(int id, double value, string name)
      : m_id(id), m_value(value), m_name(name) {}

   // Copy constructor
   CMyClass(const CMyClass &other)
      : m_id(other.m_id), m_value(other.m_value), m_name(other.m_name) {}

   // Destructor
   ~CMyClass() { Print("Destroyed: ", m_name); }

   // Const method (does not modify object state)
   int    GetId()    const { return m_id; }
   double GetValue() const { return m_value; }
   string GetName()  const { return m_name; }

   // Mutator methods
   void SetValue(double value) { m_value = value; }

   // Regular method
   void PrintInfo()
   {
      PrintFormat("ID=%d, Value=%.2f, Name=%s", m_id, m_value, m_name);
   }
};
```

### Inheritance

```mql5
class CBase
{
public:
   virtual void Process() { Print("CBase::Process"); }
   virtual string GetType() const { return "Base"; }
};

class CDerived : public CBase  // Single inheritance only
{
public:
   void Process() override { Print("CDerived::Process"); }  // override keyword
   string GetType() const override final { return "Derived"; }  // final = no further override
};

class CFinalClass final : public CDerived  // final class = cannot be inherited
{
public:
   void Process() override { Print("CFinalClass::Process"); }
   // Cannot override GetType() because it's marked final in CDerived
};
```

### Virtual Functions and Polymorphism

```mql5
// Abstract class with pure virtual function
class CSignal
{
public:
   virtual int  GetSignal() = 0;   // Pure virtual (= 0), makes class abstract
   virtual void Release() { delete &this; }
   virtual ~CSignal() {}
};

class CMASignal : public CSignal
{
public:
   int GetSignal() override
   {
      // MA crossover logic
      return 1;  // Buy
   }
};

class CRSISignal : public CSignal
{
public:
   int GetSignal() override
   {
      // RSI overbought/oversold logic
      return -1;  // Sell
   }
};

// Dynamic dispatch with pointer arrays
CSignal *signals[];
void OnInit()
{
   ArrayResize(signals, 2);
   signals[0] = new CMASignal();   // new allocates on heap
   signals[1] = new CRSISignal();
}

void OnDeinit(const int reason)
{
   for(int i = 0; i < ArraySize(signals); i++)
      delete signals[i];           // delete frees heap memory
}

void OnTick()
{
   for(int i = 0; i < ArraySize(signals); i++)
      Print("Signal[", i, "]: ", signals[i].GetSignal());  // Polymorphic call
}
```

### Interfaces

```mql5
// interface keyword: no data members, all methods implicitly pure virtual
interface ITradeExecutor
{
   bool Execute(string symbol, double volume, int direction);
   void Cancel();
};

interface ILogger
{
   void Log(string message);
   void LogError(string message);
};

// Multiple interface inheritance is allowed
class CSmartTrader : public ITradeExecutor, public ILogger
{
private:
   string m_logPrefix;

public:
   CSmartTrader(string prefix) : m_logPrefix(prefix) {}

   // Must implement ALL interface methods
   bool Execute(string symbol, double volume, int direction) override
   {
      Log(StringFormat("Executing %s %.2f lots dir=%d", symbol, volume, direction));
      return true;
   }

   void Cancel() override { Log("Trade cancelled"); }

   void Log(string message) override
   {
      Print(m_logPrefix, ": ", message);
   }

   void LogError(string message) override
   {
      Print(m_logPrefix, " ERROR: ", message);
   }
};
```

### Operator Overloading

```mql5
class CPrice
{
public:
   double value;

   CPrice() : value(0.0) {}
   CPrice(double v) : value(v) {}

   // Binary operators
   CPrice operator+(const CPrice &rhs) const { return CPrice(value + rhs.value); }
   CPrice operator-(const CPrice &rhs) const { return CPrice(value - rhs.value); }
   CPrice operator*(double factor)      const { return CPrice(value * factor); }
   CPrice operator/(double divisor)     const { return CPrice(value / divisor); }

   // Comparison operators
   bool operator==(const CPrice &rhs) const { return MathAbs(value - rhs.value) < 0.000001; }
   bool operator!=(const CPrice &rhs) const { return !(this == rhs); }
   bool operator<(const CPrice &rhs)  const { return value < rhs.value; }
   bool operator>(const CPrice &rhs)  const { return value > rhs.value; }

   // Assignment operators
   CPrice *operator=(const CPrice &rhs)  { value = rhs.value; return &this; }
   CPrice *operator+=(const CPrice &rhs) { value += rhs.value; return &this; }

   // Unary operators
   CPrice operator-() const { return CPrice(-value); }
   CPrice operator++()    { value += _Point; return this; }   // prefix
   CPrice operator++(int) { CPrice tmp = this; value += _Point; return tmp; }  // postfix

   // Indexing operator (for array-like objects)
   // double operator[](int index) const { ... }
};
```

### Templates

```mql5
// Function template
template<typename T>
T Max(T a, T b)
{
   return (a > b) ? a : b;
}

// Class template
template<typename T>
class CStack
{
private:
   T     m_data[];
   int   m_top;

public:
   CStack() : m_top(-1) { ArrayResize(m_data, 0); }

   void Push(T item)
   {
      m_top++;
      ArrayResize(m_data, m_top + 1);
      m_data[m_top] = item;
   }

   T Pop()
   {
      if(m_top < 0) return (T)NULL;
      T item = m_data[m_top];
      m_top--;
      ArrayResize(m_data, m_top + 1);
      return item;
   }

   int Size() const { return m_top + 1; }
   bool IsEmpty() const { return m_top < 0; }
};

// Usage
CStack<double> priceStack;
priceStack.Push(1.2345);
double val = priceStack.Pop();

int maxVal = Max<int>(10, 20);      // explicit type
double maxD = Max(1.5, 2.5);        // type deduction
```

---

## Trade Functions

### CTrade Class

Include: `#include <Trade/Trade.mqh>`

#### Configuration

```mql5
#include <Trade/Trade.mqh>
CTrade trade;

void OnInit()
{
   trade.SetExpertMagicNumber(12345);
   trade.SetDeviationInPoints(10);
   trade.SetTypeFillingBySymbol(_Symbol);  // Auto-detect filling (PREFERRED)
   trade.SetAsyncMode(false);              // true = non-blocking OrderSendAsync
}
```

#### CTrade Methods Summary

| Method | Description |
|--------|-------------|
| `Buy(vol, sym, price, sl, tp, comment)` | Market buy (price=0 for current Ask) |
| `Sell(vol, sym, price, sl, tp, comment)` | Market sell (price=0 for current Bid) |
| `BuyLimit(vol, price, sym, sl, tp, type_time, exp, comment)` | Pending buy below market |
| `SellLimit(vol, price, sym, sl, tp, type_time, exp, comment)` | Pending sell above market |
| `BuyStop(vol, price, sym, sl, tp, type_time, exp, comment)` | Pending buy above market |
| `SellStop(vol, price, sym, sl, tp, type_time, exp, comment)` | Pending sell below market |
| `PositionOpen(sym, type, vol, price, sl, tp, comment)` | Open position (alternative) |
| `PositionModify(sym_or_ticket, sl, tp)` | Modify SL/TP |
| `PositionClose(sym_or_ticket, deviation)` | Close entire position |
| `PositionClosePartial(sym_or_ticket, vol, deviation)` | Partial close |
| `PositionCloseBy(ticket, ticketOpposite)` | Close by opposite (hedging) |
| `OrderOpen(sym, type, vol, limitPrice, price, sl, tp, type_time, exp, comment)` | Generic pending |
| `OrderModify(ticket, price, sl, tp, type_time, exp)` | Modify pending |
| `OrderDelete(ticket)` | Delete pending |

#### Result Access

```mql5
if(trade.Buy(0.1, _Symbol))
{
   uint   retcode = trade.ResultRetcode();
   string desc    = trade.ResultRetcodeDescription();
   ulong  deal    = trade.ResultDeal();
   ulong  order   = trade.ResultOrder();
   double volume  = trade.ResultVolume();
   double price   = trade.ResultPrice();
}
```

### Native Trade Functions

#### Position / Order / History Access

```mql5
// Positions (open trades)
int total = PositionsTotal();
for(int i = 0; i < total; i++) {
   ulong ticket = PositionGetTicket(i);
   // PositionGetString(POSITION_SYMBOL), PositionGetInteger(POSITION_TYPE)
   // PositionGetDouble(POSITION_VOLUME/POSITION_PRICE_OPEN/POSITION_SL/POSITION_TP/POSITION_PROFIT)
   // PositionGetInteger(POSITION_MAGIC/POSITION_IDENTIFIER)
}
PositionSelect(_Symbol);           // Select by symbol (netting)
PositionSelectByTicket(ticket);    // Select by ticket (hedging)

// Pending orders
int totalOrd = OrdersTotal();
for(int i = 0; i < totalOrd; i++) {
   ulong ticket = OrderGetTicket(i);
   // OrderGetString(ORDER_SYMBOL), OrderGetInteger(ORDER_TYPE)
   // OrderGetDouble(ORDER_VOLUME_CURRENT/ORDER_PRICE_OPEN/ORDER_SL/ORDER_TP)
}

// History (deals + orders)
HistorySelect(from, to);                    // Select time range
HistorySelectByPosition(positionId);        // Select by position ID
int deals = HistoryDealsTotal();            // HistoryDealGetTicket(i)
int orders = HistoryOrdersTotal();          // HistoryOrderGetTicket(i)
// HistoryDealGetDouble(ticket, DEAL_PROFIT/DEAL_COMMISSION/DEAL_SWAP)
// HistoryDealGetInteger(ticket, DEAL_TYPE/DEAL_ENTRY)
```

#### OrderSend / OrderSendAsync

```mql5
MqlTradeRequest request = {};
MqlTradeResult  result  = {};

request.action    = TRADE_ACTION_DEAL;
request.symbol    = _Symbol;
request.volume    = 0.1;
request.type      = ORDER_TYPE_BUY;
request.price     = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
request.sl        = request.price - 500 * _Point;
request.tp        = request.price + 1000 * _Point;
request.deviation = 10;
request.magic     = 12345;
request.type_filling = ORDER_FILLING_FOK;

if(!OrderSend(request, result))
   PrintFormat("Error %d: retcode=%u", GetLastError(), result.retcode);

// Async: OrderSendAsync(request, result) - handle in OnTradeTransaction()
```

> **Production patterns** (retry logic, filling policy detection, error handling, trailing stops, risk management): See [trading-operations.md](trading-operations.md)

---

## Event Handlers

### Complete Event Handler Table

| Handler | Context | Parameters | Description |
|---------|---------|------------|-------------|
| `OnInit()` | EA, Indicator, Script | None. Returns `int` (INIT_SUCCEEDED, INIT_FAILED, etc.) | Called once on load/recompile. Initialize resources. |
| `OnDeinit(const int reason)` | EA, Indicator, Script | `reason`: REASON_PROGRAM, REASON_REMOVE, REASON_RECOMPILE, REASON_CHARTCHANGE, REASON_CHARTCLOSE, REASON_PARAMETERS, REASON_ACCOUNT, REASON_TEMPLATE, REASON_INITFAILED, REASON_CLOSE | Called on unload. Cleanup resources, delete objects. |
| `OnTick()` | EA | None. Returns `void` | Called on every new tick. Main EA logic. |
| `OnTimer()` | EA, Indicator | None. Returns `void` | Called on timer event. Set with `EventSetTimer(seconds)` or `EventSetMillisecondTimer(ms)`. Kill with `EventKillTimer()`. |
| `OnTrade()` | EA | None. Returns `void` | Called when a trade event occurs (order placed, modified, deleted, deal executed, position changed). No parameters about what changed. |
| `OnTradeTransaction()` | EA | `const MqlTradeTransaction &trans, const MqlTradeRequest &request, const MqlTradeResult &result` | Called with details about each trade transaction. More granular than `OnTrade()`. |
| `OnBookEvent(const string &symbol)` | EA | `symbol`: Symbol name | Called when Depth of Market changes. Must call `MarketBookAdd(symbol)` first. Call `MarketBookRelease(symbol)` in OnDeinit. |
| `OnChartEvent()` | EA, Indicator | `const int id, const long &lparam, const double &dparam, const string &sparam` | Chart events: mouse clicks, key presses, object events, custom events. |
| `OnCalculate()` | Indicator | Two forms (see below) | Called on every new tick. Main indicator calculation logic. |
| `OnTester()` | EA | None. Returns `double` | Called after backtesting. Return custom optimization criterion. |
| `OnTesterInit()` | EA | None. Returns `void` | Called before optimization starts (in the optimization agent). |
| `OnTesterPass()` | EA | None. Returns `void` | Called when an optimization pass result is received. Use `FrameFirst()`/`FrameNext()`. |
| `OnTesterDeinit()` | EA | None. Returns `void` | Called after optimization ends. Cleanup. |

### OnTradeTransaction Example

```mql5
void OnTradeTransaction(const MqlTradeTransaction &trans,
                        const MqlTradeRequest &request,
                        const MqlTradeResult &result)
{
   // trans.type is ENUM_TRADE_TRANSACTION_TYPE:
   //   TRADE_TRANSACTION_ORDER_ADD      - new order added
   //   TRADE_TRANSACTION_ORDER_UPDATE   - order updated
   //   TRADE_TRANSACTION_ORDER_DELETE   - order removed from active
   //   TRADE_TRANSACTION_DEAL_ADD       - deal executed
   //   TRADE_TRANSACTION_DEAL_UPDATE    - deal updated
   //   TRADE_TRANSACTION_DEAL_DELETE    - deal deleted
   //   TRADE_TRANSACTION_HISTORY_ADD    - order moved to history
   //   TRADE_TRANSACTION_HISTORY_UPDATE - history order updated
   //   TRADE_TRANSACTION_HISTORY_DELETE - history order deleted
   //   TRADE_TRANSACTION_POSITION       - position changed (not related to deal)
   //   TRADE_TRANSACTION_REQUEST        - trade request processed

   switch(trans.type)
   {
      case TRADE_TRANSACTION_DEAL_ADD:
         PrintFormat("Deal added: ticket=%I64u, symbol=%s, type=%d, volume=%.2f, price=%.5f",
                     trans.deal, trans.symbol, trans.deal_type, trans.volume, trans.price);

         // For async trading: match via trans.order to your sent order
         if(HistoryDealSelect(trans.deal))
         {
            double profit = HistoryDealGetDouble(trans.deal, DEAL_PROFIT);
            long   entry  = HistoryDealGetInteger(trans.deal, DEAL_ENTRY);
            if(entry == DEAL_ENTRY_IN)
               Print("Position opened");
            else if(entry == DEAL_ENTRY_OUT)
               PrintFormat("Position closed, profit=%.2f", profit);
         }
         break;

      case TRADE_TRANSACTION_REQUEST:
         PrintFormat("Request processed: retcode=%u, order=%I64u",
                     result.retcode, result.order);
         break;
   }
}
```

### OnTester Example

```mql5
double OnTester()
{
   // Custom optimization criterion
   // Return value is used as the optimization target when "Custom max" is selected

   double profit = TesterStatistics(STAT_PROFIT);
   double dd     = TesterStatistics(STAT_EQUITY_DD_RELATIVE);  // % max drawdown
   int    trades = (int)TesterStatistics(STAT_TRADES);
   double pf     = TesterStatistics(STAT_PROFIT_FACTOR);

   // Example: profit factor weighted by number of trades, penalize low trade count
   if(trades < 30 || dd > 30.0)
      return 0.0;

   // Calmar-like ratio: profit / drawdown
   double criterion = (dd > 0.0) ? profit / dd : 0.0;
   return criterion;
}
```

### OnCalculate Forms

```mql5
// Form 1: Short form (for indicators that use price data directly)
int OnCalculate(const int rates_total,
                const int prev_calculated,
                const int begin,
                const double &price[])
{
   for(int i = (prev_calculated > 0 ? prev_calculated - 1 : 0); i < rates_total; i++)
   {
      // price[] corresponds to the "Apply to" setting (Close, Open, High, etc.)
      Buffer[i] = price[i];
   }
   return rates_total;
}

// Form 2: Full form (access to OHLCV and time arrays)
int OnCalculate(const int rates_total,
                const int prev_calculated,
                const datetime &time[],
                const double &open[],
                const double &high[],
                const double &low[],
                const double &close[],
                const long &tick_volume[],
                const long &volume[],
                const int &spread[])
{
   for(int i = (prev_calculated > 0 ? prev_calculated - 1 : 0); i < rates_total; i++)
   {
      Buffer[i] = (high[i] + low[i] + close[i]) / 3.0;  // Typical price
   }
   return rates_total;
}
```

### OnBookEvent Example

```mql5
int OnInit()
{
   MarketBookAdd(_Symbol);  // Subscribe to DOM updates
   return INIT_SUCCEEDED;
}

void OnDeinit(const int reason)
{
   MarketBookRelease(_Symbol);  // Unsubscribe from DOM
}

void OnBookEvent(const string &symbol)
{
   if(symbol != _Symbol) return;

   MqlBookInfo book[];
   if(MarketBookGet(symbol, book))
   {
      for(int i = 0; i < ArraySize(book); i++)
      {
         PrintFormat("Type=%s, Price=%.5f, Volume=%.0f",
            (book[i].type == BOOK_TYPE_SELL) ? "ASK" : "BID",
            book[i].price, book[i].volume);
      }
   }
}
```

---

## Standard Library

### Trade Classes (`#include <Trade/...>`)

| Class | Include | Purpose |
|-------|---------|---------|
| `CTrade` | `<Trade/Trade.mqh>` | Trade execution: market orders, pending orders, position/order management. |
| `CPositionInfo` | `<Trade/PositionInfo.mqh>` | Query open position properties (symbol, type, volume, SL, TP, profit, etc.). |
| `COrderInfo` | `<Trade/OrderInfo.mqh>` | Query pending order properties. |
| `CDealInfo` | `<Trade/DealInfo.mqh>` | Query historical deal properties. |
| `CSymbolInfo` | `<Trade/SymbolInfo.mqh>` | Symbol properties: bid, ask, point, digits, spread, volume limits, session times, etc. |
| `CAccountInfo` | `<Trade/AccountInfo.mqh>` | Account properties: balance, equity, margin, free margin, leverage, etc. |
| `CTerminalInfo` | `<Trade/TerminalInfo.mqh>` | Terminal info: connected, trade allowed, community account, path, etc. |

```mql5
#include <Trade/Trade.mqh>
#include <Trade/PositionInfo.mqh>
#include <Trade/SymbolInfo.mqh>
#include <Trade/AccountInfo.mqh>

CTrade        trade;
CPositionInfo posInfo;
CSymbolInfo   symInfo;
CAccountInfo  accInfo;

void OnInit()
{
   symInfo.Name(_Symbol);
   trade.SetExpertMagicNumber(12345);
   trade.SetTypeFillingBySymbol(_Symbol);

   PrintFormat("Balance=%.2f, Leverage=%d", accInfo.Balance(), accInfo.Leverage());
   PrintFormat("Point=%.5f, Digits=%d, Spread=%d",
               symInfo.Point(), symInfo.Digits(), symInfo.Spread());
}

void OnTick()
{
   symInfo.RefreshRates();  // Must refresh before accessing prices

   double ask = symInfo.Ask();
   double bid = symInfo.Bid();

   // Iterate open positions
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(posInfo.SelectByIndex(i))
      {
         if(posInfo.Symbol() == _Symbol && posInfo.Magic() == 12345)
         {
            PrintFormat("Pos: type=%s, vol=%.2f, profit=%.2f",
               posInfo.TypeDescription(), posInfo.Volume(), posInfo.Profit());
         }
      }
   }
}
```

### Indicator Classes (`#include <Indicators/...>`)

#### Trend Indicators

| Class | Indicator |
|-------|-----------|
| `CiMA` | Moving Average |
| `CiADX` | Average Directional Index |
| `CiBands` | Bollinger Bands |
| `CiIchimoku` | Ichimoku Kinko Hyo |
| `CiParabolicSAR` | Parabolic SAR |
| `CiStdDev` | Standard Deviation |
| `CiDEMA` | Double Exponential MA |
| `CiTEMA` | Triple Exponential MA |
| `CiFrAMA` | Fractal Adaptive MA |
| `CiAMA` | Adaptive MA |
| `CiVIDyA` | Variable Index Dynamic Average |
| `CiEnvelopes` | Envelopes |

#### Oscillators

| Class | Indicator |
|-------|-----------|
| `CiMACD` | MACD |
| `CiRSI` | Relative Strength Index |
| `CiStochastic` | Stochastic Oscillator |
| `CiCCI` | Commodity Channel Index |
| `CiATR` | Average True Range |
| `CiMomentum` | Momentum |
| `CiOsMA` | Moving Average of Oscillator |
| `CiWPR` | Williams' Percent Range |
| `CiRVI` | Relative Vigor Index |
| `CiForce` | Force Index |
| `CiDeMarker` | DeMarker |
| `CiBearsPower` | Bears Power |
| `CiBullsPower` | Bulls Power |
| `CiTriX` | Triple Exponential MA Oscillator |
| `CiAO` | Awesome Oscillator |
| `CiAC` | Accelerator Oscillator |

#### Volume Indicators

| Class | Indicator |
|-------|-----------|
| `CiVolumes` | Volumes |
| `CiOBV` | On Balance Volume |
| `CiMFI` | Money Flow Index |
| `CiAD` | Accumulation/Distribution |

#### Usage Pattern

```mql5
#include <Indicators/Trend.mqh>
#include <Indicators/Oscilators.mqh>

CiMA   ma;
CiRSI  rsi;
CiMACD macd;

int OnInit()
{
   // Create(symbol, timeframe, period, shift, method, applied_price)
   if(!ma.Create(_Symbol, PERIOD_CURRENT, 20, 0, MODE_SMA, PRICE_CLOSE))
      return INIT_FAILED;

   // Create(symbol, timeframe, period, applied_price)
   if(!rsi.Create(_Symbol, PERIOD_CURRENT, 14, PRICE_CLOSE))
      return INIT_FAILED;

   // Create(symbol, timeframe, fast_ema, slow_ema, signal, applied_price)
   if(!macd.Create(_Symbol, PERIOD_CURRENT, 12, 26, 9, PRICE_CLOSE))
      return INIT_FAILED;

   return INIT_SUCCEEDED;
}

void OnTick()
{
   ma.Refresh();         // Must call Refresh() on each tick
   rsi.Refresh();
   macd.Refresh();

   double maValue    = ma.Main(0);      // Main(index) - 0 = current bar
   double maValue1   = ma.Main(1);      // Previous bar

   double rsiValue   = rsi.Main(0);

   double macdMain   = macd.Main(0);    // MACD main line
   double macdSignal = macd.Signal(0);  // MACD signal line

   // For Bollinger Bands
   // CiBands bands;
   // bands.Create(_Symbol, PERIOD_CURRENT, 20, 0, 2.0, PRICE_CLOSE);
   // bands.Refresh();
   // double upper  = bands.Upper(0);
   // double base   = bands.Base(0);
   // double lower  = bands.Lower(0);
}
```

### Other Standard Library Classes

| Class | Include | Purpose |
|-------|---------|---------|
| `CObject` | `<Object.mqh>` | Base class for all standard library objects. |
| `CArrayInt` | `<Arrays/ArrayInt.mqh>` | Dynamic array of int. |
| `CArrayDouble` | `<Arrays/ArrayDouble.mqh>` | Dynamic array of double. |
| `CArrayString` | `<Arrays/ArrayString.mqh>` | Dynamic array of string. |
| `CArrayObj` | `<Arrays/ArrayObj.mqh>` | Dynamic array of CObject pointers. |
| `CList` | `<Arrays/List.mqh>` | Doubly-linked list of CObject pointers. |
| `CString` | `<Strings/String.mqh>` | String wrapper with utility methods. |
| `CFile` | `<Files/File.mqh>` | File operations (base class for CFileTxt, CFileBin). |
| `CCanvas` | `<Canvas/Canvas.mqh>` | Pixel-level drawing on charts (for custom UIs). |

---

## Key Enumerations

### ENUM_TRADE_REQUEST_ACTIONS

| Value | Description |
|-------|-------------|
| `TRADE_ACTION_DEAL` | Place a market order (immediate execution). |
| `TRADE_ACTION_PENDING` | Place a pending order. |
| `TRADE_ACTION_SLTP` | Modify SL/TP of an existing position. |
| `TRADE_ACTION_MODIFY` | Modify parameters of a pending order. |
| `TRADE_ACTION_REMOVE` | Delete a pending order. |
| `TRADE_ACTION_CLOSE_BY` | Close a position by an opposite one (hedging only). |

### ENUM_ORDER_TYPE

| Value | Description |
|-------|-------------|
| `ORDER_TYPE_BUY` | Market buy order. |
| `ORDER_TYPE_SELL` | Market sell order. |
| `ORDER_TYPE_BUY_LIMIT` | Pending buy limit (below market). |
| `ORDER_TYPE_SELL_LIMIT` | Pending sell limit (above market). |
| `ORDER_TYPE_BUY_STOP` | Pending buy stop (above market). |
| `ORDER_TYPE_SELL_STOP` | Pending sell stop (below market). |
| `ORDER_TYPE_BUY_STOP_LIMIT` | Pending buy stop-limit. When price reaches stop, a buy limit is placed. |
| `ORDER_TYPE_SELL_STOP_LIMIT` | Pending sell stop-limit. When price reaches stop, a sell limit is placed. |

### ENUM_POSITION_TYPE

| Value | Description |
|-------|-------------|
| `POSITION_TYPE_BUY` | Long position. |
| `POSITION_TYPE_SELL` | Short position. |

### ENUM_ORDER_TYPE_FILLING

| Value | Description |
|-------|-------------|
| `ORDER_FILLING_FOK` | Fill or Kill. Entire volume must be filled or order is cancelled. |
| `ORDER_FILLING_IOC` | Immediate or Cancel. Fill what's available, cancel remainder. |
| `ORDER_FILLING_RETURN` | Return. Used for market-making; partial fills return remainder as pending. |

**Important**: Always detect the correct filling mode per symbol:

```mql5
ENUM_ORDER_TYPE_FILLING GetFillingMode(string symbol)
{
   long fillMode = SymbolInfoInteger(symbol, SYMBOL_FILLING_MODE);

   if((fillMode & SYMBOL_FILLING_FOK) == SYMBOL_FILLING_FOK)
      return ORDER_FILLING_FOK;

   if((fillMode & SYMBOL_FILLING_IOC) == SYMBOL_FILLING_IOC)
      return ORDER_FILLING_IOC;

   return ORDER_FILLING_RETURN;
}
```

Or simply use `CTrade::SetTypeFillingBySymbol(_Symbol)`.

### ENUM_SYMBOL_INFO_DOUBLE (Selected)

| Value | Description |
|-------|-------------|
| `SYMBOL_BID` | Current Bid price. |
| `SYMBOL_ASK` | Current Ask price. |
| `SYMBOL_POINT` | Point value (e.g., 0.00001 for 5-digit). |
| `SYMBOL_TRADE_TICK_VALUE` | Value of one tick in account currency. |
| `SYMBOL_TRADE_TICK_SIZE` | Minimum price change. |
| `SYMBOL_TRADE_CONTRACT_SIZE` | Contract size (e.g., 100000 for standard forex lot). |
| `SYMBOL_VOLUME_MIN` | Minimum volume for a deal (e.g., 0.01). |
| `SYMBOL_VOLUME_MAX` | Maximum volume for a deal. |
| `SYMBOL_VOLUME_STEP` | Volume step (e.g., 0.01). |
| `SYMBOL_TRADE_STOPS_LEVEL` | Minimum distance in points for SL/TP from current price. |
| `SYMBOL_TRADE_FREEZE_LEVEL` | Distance in points within which order modification/deletion is frozen. |

### ENUM_ACCOUNT_INFO_DOUBLE (Selected)

| Value | Description |
|-------|-------------|
| `ACCOUNT_BALANCE` | Account balance. |
| `ACCOUNT_EQUITY` | Account equity. |
| `ACCOUNT_PROFIT` | Current floating profit/loss. |
| `ACCOUNT_MARGIN` | Margin currently used. |
| `ACCOUNT_MARGIN_FREE` | Free margin available. |
| `ACCOUNT_MARGIN_LEVEL` | Margin level as percentage (Equity / Margin * 100). |

---

## MQL5-Specific Features

### Database (SQLite)

MQL5 has built-in SQLite database support for persistent structured storage.

```mql5
int db = INVALID_HANDLE;

int OnInit()
{
   // Open or create database in the common files folder
   db = DatabaseOpen("my_ea_data.sqlite", DATABASE_OPEN_READWRITE | DATABASE_OPEN_CREATE |
                     DATABASE_OPEN_COMMON);
   if(db == INVALID_HANDLE)
   {
      Print("DB open failed: ", GetLastError());
      return INIT_FAILED;
   }

   // Create table
   if(!DatabaseExecute(db,
      "CREATE TABLE IF NOT EXISTS trades ("
      "id INTEGER PRIMARY KEY AUTOINCREMENT,"
      "symbol TEXT NOT NULL,"
      "type INTEGER,"
      "volume REAL,"
      "open_price REAL,"
      "close_price REAL,"
      "profit REAL,"
      "open_time TEXT,"
      "close_time TEXT)"))
   {
      Print("CREATE TABLE failed: ", GetLastError());
   }

   return INIT_SUCCEEDED;
}

void InsertTrade(string symbol, int type, double volume, double price)
{
   string sql = StringFormat(
      "INSERT INTO trades (symbol, type, volume, open_price, open_time) "
      "VALUES ('%s', %d, %.2f, %.5f, '%s')",
      symbol, type, volume, price, TimeToString(TimeCurrent()));

   if(!DatabaseExecute(db, sql))
      Print("INSERT failed: ", GetLastError());
}

void ReadTrades()
{
   int request = DatabasePrepare(db, "SELECT id, symbol, profit FROM trades WHERE profit > 0");
   if(request == INVALID_HANDLE)
   {
      Print("Prepare failed: ", GetLastError());
      return;
   }

   int    id;
   string symbol;
   double profit;

   while(DatabaseRead(request))
   {
      DatabaseColumnInteger(request, 0, id);
      DatabaseColumnText(request, 1, symbol);
      DatabaseColumnDouble(request, 2, profit);
      PrintFormat("Trade #%d: %s, Profit=%.2f", id, symbol, profit);
   }

   DatabaseFinalize(request);
}

// Transactions for batch operations
void BatchInsert()
{
   DatabaseTransactionBegin(db);

   for(int i = 0; i < 100; i++)
   {
      if(!DatabaseExecute(db, StringFormat("INSERT INTO trades (symbol) VALUES ('ITEM_%d')", i)))
      {
         DatabaseTransactionRollback(db);
         return;
      }
   }

   DatabaseTransactionCommit(db);
}

void OnDeinit(const int reason)
{
   if(db != INVALID_HANDLE)
      DatabaseClose(db);
}
```

### Network Sockets

Available only in EAs and scripts (NOT indicators). Allows raw TCP and TLS connections.

```mql5
int OnInit()
{
   // Create socket
   int socket = SocketCreate();
   if(socket == INVALID_HANDLE)
   {
      Print("SocketCreate failed: ", GetLastError());
      return INIT_FAILED;
   }

   // Connect to server (TCP)
   if(!SocketConnect(socket, "api.example.com", 443, 5000))  // host, port, timeout_ms
   {
      Print("SocketConnect failed: ", GetLastError());
      SocketClose(socket);
      return INIT_FAILED;
   }

   // TLS handshake for HTTPS
   if(!SocketTlsHandshake(socket, "api.example.com"))
   {
      Print("TLS handshake failed: ", GetLastError());
      SocketClose(socket);
      return INIT_FAILED;
   }

   // Build HTTP request
   string request = "GET /data HTTP/1.1\r\n"
                    "Host: api.example.com\r\n"
                    "Connection: close\r\n"
                    "\r\n";

   // Send via TLS
   uchar reqData[];
   StringToCharArray(request, reqData, 0, WHOLE_ARRAY, CP_UTF8);
   int sent = SocketTlsSend(socket, reqData, ArraySize(reqData) - 1);  // exclude null terminator

   if(sent <= 0)
   {
      Print("SocketTlsSend failed: ", GetLastError());
      SocketClose(socket);
      return INIT_FAILED;
   }

   // Read response
   uchar response[];
   string result = "";
   uint timeout = 5000;

   do
   {
      uint len = SocketTlsReadAvailable(socket, response, timeout);
      if(len > 0)
         result += CharArrayToString(response, 0, WHOLE_ARRAY, CP_UTF8);
   }
   while(SocketTlsRead(socket, response, 1024, timeout) > 0);

   Print("Response: ", result);

   SocketClose(socket);
   return INIT_SUCCEEDED;
}

// For non-TLS (plain TCP):
// SocketSend(socket, data, dataLen);
// SocketRead(socket, response, responseLen, timeout);
```

### Resources

Embed external files (images, sounds, data) directly into the compiled EX5 file.

```mql5
// Embed resource at compile time
#resource "\\Images\\logo.bmp"          // Relative to source file
#resource "\\Sounds\\alert.wav"
#resource "\\Files\\config.txt"

void OnInit()
{
   // Access resource as "::Images\\logo.bmp"
   ObjectCreate(0, "Logo", OBJ_BITMAP_LABEL, 0, 0, 0);
   ObjectSetString(0, "Logo", OBJPROP_BMPFILE, "::Images\\logo.bmp");

   // Play sound from resource
   PlaySound("::Sounds\\alert.wav");

   // Create dynamic resource from pixel data
   uint pixels[];
   int width = 100, height = 100;
   ArrayResize(pixels, width * height);

   for(int i = 0; i < ArraySize(pixels); i++)
      pixels[i] = ColorToARGB(clrBlue, 200);

   ResourceCreate("::DynImage", pixels, width, height, 0, 0, width, COLOR_FORMAT_ARGB_NORMALIZE);

   // Read an image resource
   uint data[];
   int w, h;
   ResourceReadImage("::Images\\logo.bmp", data, w, h);

   // Save resource to file
   ResourceSave("::Images\\logo.bmp", "SavedFiles\\logo.bmp");

   // Free dynamic resource
   ResourceFree("::DynImage");
}
```

### OpenCL

GPU-accelerated parallel computing for intensive calculations (optimization, neural networks, etc.).

```mql5
void RunOpenCL()
{
   // 1. Create context (0 = default device)
   int clContext = CLContextCreate(CL_USE_GPU_ONLY);
   if(clContext == INVALID_HANDLE)
   {
      Print("CLContextCreate failed");
      return;
   }

   // 2. Create program from OpenCL kernel source
   string kernelSource =
      "__kernel void multiply(__global double *a, __global double *b, __global double *c, int n) {"
      "   int i = get_global_id(0);"
      "   if(i < n) c[i] = a[i] * b[i];"
      "}";

   string buildLog;
   int clProgram = CLProgramCreate(clContext, kernelSource, buildLog);
   if(clProgram == INVALID_HANDLE)
   {
      Print("CLProgramCreate failed: ", buildLog);
      CLContextFree(clContext);
      return;
   }

   // 3. Create kernel
   int clKernel = CLKernelCreate(clProgram, "multiply");

   // 4. Prepare data
   int n = 1000;
   double a[], b[], c[];
   ArrayResize(a, n);
   ArrayResize(b, n);
   ArrayResize(c, n);

   for(int i = 0; i < n; i++) { a[i] = i * 1.5; b[i] = i * 2.0; }

   // 5. Create buffers and write data
   int bufA = CLBufferCreate(clContext, n * sizeof(double), CL_MEM_READ_ONLY);
   int bufB = CLBufferCreate(clContext, n * sizeof(double), CL_MEM_READ_ONLY);
   int bufC = CLBufferCreate(clContext, n * sizeof(double), CL_MEM_WRITE_ONLY);

   CLBufferWrite(bufA, a);
   CLBufferWrite(bufB, b);

   // 6. Set kernel arguments
   CLSetKernelArgMem(clKernel, 0, bufA);
   CLSetKernelArgMem(clKernel, 1, bufB);
   CLSetKernelArgMem(clKernel, 2, bufC);
   CLSetKernelArg(clKernel, 3, n);

   // 7. Execute
   uint globalWorkSize[1] = {(uint)n};
   CLExecute(clKernel, 1, 0, globalWorkSize);

   // 8. Read results
   CLBufferRead(bufC, c);

   // 9. Cleanup
   CLBufferFree(bufA);
   CLBufferFree(bufB);
   CLBufferFree(bufC);
   CLKernelFree(clKernel);
   CLProgramFree(clProgram);
   CLContextFree(clContext);
}
```

### WebRequest (MQL5 Variant)

Two function signatures. URL must be whitelisted in Tools > Options > Expert Advisors. Only available in EAs and scripts, NOT indicators or Strategy Tester.

```mql5
// Form 1: Custom headers
void WebRequestWithHeaders()
{
   string url     = "https://api.example.com/webhook";
   string headers = "Content-Type: application/json\r\n"
                    "Authorization: Bearer YOUR_TOKEN\r\n";
   int    timeout = 5000;

   // Build JSON body
   string jsonBody = StringFormat(
      "{\"symbol\":\"%s\",\"price\":%.5f,\"action\":\"BUY\"}",
      _Symbol, SymbolInfoDouble(_Symbol, SYMBOL_ASK));

   char   postData[];
   char   resultData[];
   string resultHeaders;

   StringToCharArray(jsonBody, postData, 0, WHOLE_ARRAY, CP_UTF8);
   // Remove null terminator
   ArrayResize(postData, ArraySize(postData) - 1);

   int statusCode = WebRequest(
      "POST",              // string method
      url,                 // string url
      headers,             // string headers
      timeout,             // int timeout
      postData,            // char &data[]
      resultData,          // char &result[]
      resultHeaders        // string &result_headers
   );

   if(statusCode == -1)
   {
      Print("WebRequest error: ", GetLastError());
      Print("Add URL to allowed list: Tools > Options > Expert Advisors");
      return;
   }

   string response = CharArrayToString(resultData, 0, WHOLE_ARRAY, CP_UTF8);
   PrintFormat("HTTP %d: %s", statusCode, response);
}

// Form 2: Simple headers as string (cookie, referer)
void WebRequestSimple()
{
   string cookie = "", headers = "", response = "";
   char   postData[], resultData[];
   string resultHeaders;

   int statusCode = WebRequest(
      "GET",                              // method
      "https://api.example.com/data",     // url
      cookie,                             // cookie
      "",                                 // referer
      5000,                               // timeout
      postData,                           // data (empty for GET)
      0,                                  // data_size
      resultData,                         // result
      resultHeaders                       // result_headers
   );

   if(statusCode == 200)
   {
      response = CharArrayToString(resultData, 0, WHOLE_ARRAY, CP_UTF8);
      Print("Response: ", response);
   }
}
```

---

## MQL5 vs MQL4 Summary Table

| Feature | MQL4 | MQL5 |
|---------|------|------|
| **OOP** | Limited (classes added later) | Full OOP: classes, inheritance, interfaces, templates, operator overloading |
| **Account model** | Hedging only | Netting + Hedging |
| **Trade model** | Orders only (`OrderSend`, `OrderModify`, `OrderClose`) | Orders + Deals + Positions (`CTrade`, `OrderSend` with `MqlTradeRequest`) |
| **Standard Library** | Minimal | Comprehensive (Trade, Indicators, Arrays, Canvas, etc.) |
| **Event handlers** | `OnInit`, `OnDeinit`, `OnTick`, `OnTimer`, `OnChartEvent`, `OnCalculate`, `OnTester` | All MQL4 handlers + `OnTrade`, `OnTradeTransaction`, `OnBookEvent`, `OnTesterInit/Pass/Deinit` |
| **Indicator buffers** | Max 8 | Max 512 |
| **Draw styles** | 6 basic (LINE, SECTION, HISTOGRAM, ARROW, ZIGZAG, NONE) | 18 styles including color variants (DRAW_COLOR_LINE, DRAW_FILLING, DRAW_BARS, DRAW_CANDLES, etc.) |
| **OpenCL** | Not available | Full OpenCL support for GPU computing |
| **SQLite database** | Not available | Built-in `DatabaseOpen`, `DatabaseExecute`, `DatabasePrepare`, etc. |
| **Network sockets** | Not available | TCP + TLS via `SocketCreate`, `SocketConnect`, `SocketTlsHandshake`, etc. |
| **WebRequest** | `WebRequest()` (same function) | `WebRequest()` with two signatures (custom headers or simple) |
| **Multi-currency testing** | Limited | Full multi-symbol/multi-timeframe backtesting |
| **Cloud optimization** | Not available | MQL5 Cloud Network for distributed optimization |
| **Resources** | `#resource` for images | `#resource` for any file type + `ResourceCreate`/`ResourceReadImage`/`ResourceSave`/`ResourceFree` |
| **Custom optimization** | `OnTester()` returns double | `OnTester()` + `OnTesterInit/Pass/Deinit` + `FrameFirst/FrameNext/FrameAdd` for custom criteria and frame communication |
| **Timeseries access** | `iClose()`, `iOpen()`, etc. with shift | `CopyClose()`, `CopyOpen()`, etc. copying into arrays, or `iClose()` with MQL4-compatible syntax |
| **Indicator access** | `iMA()`, `iRSI()` return value directly | `iMA()`, `iRSI()` return handle (int). Use `CopyBuffer()` to get values, or use indicator classes. |

### Indicator Handle Pattern (MQL5)

```mql5
int maHandle;
double maBuffer[];

int OnInit()
{
   maHandle = iMA(_Symbol, PERIOD_CURRENT, 20, 0, MODE_SMA, PRICE_CLOSE);
   if(maHandle == INVALID_HANDLE) return INIT_FAILED;

   ArraySetAsSeries(maBuffer, true);  // Index 0 = newest bar
   return INIT_SUCCEEDED;
}

void OnTick()
{
   if(CopyBuffer(maHandle, 0, 0, 3, maBuffer) < 3) return;
   // maBuffer[0] = current bar MA value
   // maBuffer[1] = previous bar MA value
   // maBuffer[2] = two bars ago
}

void OnDeinit(const int reason)
{
   IndicatorRelease(maHandle);  // Free the indicator handle
}
```
