# MQL4 Language Reference

Complete reference for MetaQuotes Language 4 (MQL4), used to develop Expert Advisors, custom indicators, scripts, and libraries for the MetaTrader 4 trading platform.

## Table of Contents

- [Data Types](#data-types)
- [Variables](#variables)
- [Operators](#operators)
- [Arrays](#arrays)
- [Strings](#strings)
- [Program Types](#program-types)
- [Predefined Variables](#predefined-variables)
- [Technical Indicator Functions](#technical-indicator-functions)
- [Order Management](#order-management)
- [Market Information](#market-information)
- [Account Functions](#account-functions)
- [Preprocessor Directives](#preprocessor-directives)
- [Error Handling](#error-handling)
- [Common Gotchas](#common-gotchas)
- [File Operations](#file-operations)
- [WebRequest](#webrequest)
- [Utility Functions](#utility-functions)
- [Global Terminal Variables](#global-terminal-variables)

---

## Data Types

### Integer Types

| Type | Size | Range |
|------|------|-------|
| `char` | 1 byte | -128 to 127 |
| `uchar` | 1 byte | 0 to 255 |
| `short` | 2 bytes | -32,768 to 32,767 |
| `ushort` | 2 bytes | 0 to 65,535 |
| `int` | 4 bytes | -2,147,483,648 to 2,147,483,647 |
| `uint` | 4 bytes | 0 to 4,294,967,295 |
| `long` | 8 bytes | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |
| `ulong` | 8 bytes | 0 to 18,446,744,073,709,551,615 |

### Floating-Point Types

| Type | Size | Significant Digits | Range |
|------|------|---------------------|-------|
| `float` | 4 bytes | 7 | 1.175e-38 to 3.402e+38 |
| `double` | 8 bytes | 15-16 | 2.225e-308 to 1.797e+308 |

### Other Types

| Type | Description | Example |
|------|-------------|---------|
| `bool` | Boolean | `true`, `false` |
| `string` | Character string | `"Hello World"` |
| `datetime` | Date and time (seconds since 1970.01.01) | `D'2024.01.15'`, `D'2024.01.15 10:30:00'` |
| `color` | RGB color | `clrRed`, `clrBlue`, `C'128,128,128'`, `0x00FF00` |

```mql4
// Type examples
int    count      = 10;
double price      = 1.23456;
bool   isActive   = true;
string name       = "EURUSD";
datetime startTime = D'2024.01.15 08:00:00';
color  arrowColor = clrRed;
color  custom     = C'128,128,128';
```

### Enumerations

```mql4
enum ENUM_SIGNAL
{
   SIGNAL_NONE = 0,   // No signal
   SIGNAL_BUY  = 1,   // Buy signal
   SIGNAL_SELL = -1    // Sell signal
};

ENUM_SIGNAL signal = SIGNAL_BUY;
```

### Structures

```mql4
struct TradeSetup
{
   string symbol;
   int    direction;
   double entryPrice;
   double stopLoss;
   double takeProfit;
   double lotSize;
};

TradeSetup setup;
setup.symbol     = "EURUSD";
setup.direction  = OP_BUY;
setup.entryPrice = Ask;
setup.stopLoss   = Ask - 50 * Point;
setup.takeProfit = Ask + 100 * Point;
setup.lotSize    = 0.1;
```

### Type Casting

```mql4
double price = 1.23456;
int    pips  = (int)(price * 10000);     // Explicit cast
string text  = DoubleToString(price, 5); // Function conversion
```

---

## Variables

### Local Variables

Declared inside a function. Exist only during function execution.

```mql4
void CalculateSignal()
{
   int period = 14;          // Local variable
   double value = iRSI(NULL, 0, period, PRICE_CLOSE, 0);
}
```

### Global Variables

Declared outside any function. Accessible from all functions in the file. Initialized once when the program loads.

```mql4
double g_lotSize = 0.1;     // Global variable (file scope)
int    g_magicNumber = 12345;

int OnInit()
{
   g_lotSize = 0.2;         // Accessible here
   return INIT_SUCCEEDED;
}

void OnTick()
{
   double lots = g_lotSize; // Accessible here too
}
```

### Static Variables

Retain their value between function calls. Initialized only once.

```mql4
void OnTick()
{
   static int tickCount = 0;         // Initialized once
   static datetime lastBarTime = 0;

   tickCount++;

   if(Time[0] != lastBarTime)
   {
      lastBarTime = Time[0];
      // New bar logic here
   }
}
```

### Input Variables

User-configurable parameters. Appear in the EA/indicator properties dialog. Read-only at runtime.

```mql4
input int    InpMAPeriod   = 14;     // MA Period
input double InpLotSize    = 0.1;    // Lot Size
input string InpComment    = "MyEA"; // Order Comment
input bool   InpUseFilter  = true;   // Use Trend Filter
input ENUM_MA_METHOD InpMAMethod = MODE_SMA; // MA Method
```

### Extern Variables

Similar to `input` but can be modified at runtime. Legacy; prefer `input`.

```mql4
extern int    ExtPeriod  = 20;    // Period
extern double ExtFactor  = 1.5;   // Factor

void OnTick()
{
   ExtPeriod = 30; // Allowed (unlike input variables)
}
```

---

## Operators

### Arithmetic Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `+` | Addition | `a + b` |
| `-` | Subtraction | `a - b` |
| `*` | Multiplication | `a * b` |
| `/` | Division | `a / b` |
| `%` | Modulo | `a % b` |
| `++` | Increment | `a++` or `++a` |
| `--` | Decrement | `a--` or `--a` |

### Assignment Operators

| Operator | Description | Equivalent |
|----------|-------------|------------|
| `=` | Assign | `a = b` |
| `+=` | Add and assign | `a = a + b` |
| `-=` | Subtract and assign | `a = a - b` |
| `*=` | Multiply and assign | `a = a * b` |
| `/=` | Divide and assign | `a = a / b` |
| `%=` | Modulo and assign | `a = a % b` |

### Comparison Operators

| Operator | Description |
|----------|-------------|
| `==` | Equal to |
| `!=` | Not equal to |
| `<` | Less than |
| `>` | Greater than |
| `<=` | Less than or equal to |
| `>=` | Greater than or equal to |

### Logical Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `&&` | Logical AND | `a > 0 && b > 0` |
| `\|\|` | Logical OR | `a > 0 \|\| b > 0` |
| `!` | Logical NOT | `!isActive` |

### Bitwise Operators

| Operator | Description |
|----------|-------------|
| `&` | Bitwise AND |
| `\|` | Bitwise OR |
| `^` | Bitwise XOR |
| `~` | Bitwise NOT |
| `<<` | Left shift |
| `>>` | Right shift |

### Ternary Operator

```mql4
double lots = (AccountBalance() > 10000) ? 0.2 : 0.1;
string direction = (signal == SIGNAL_BUY) ? "BUY" : "SELL";
```

---

## Arrays

### Static Arrays

Fixed size, determined at compile time.

```mql4
double prices[100];
int    values[10] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
string symbols[3] = {"EURUSD", "GBPUSD", "USDJPY"};
```

### Dynamic Arrays

Size determined at runtime. Must be resized with `ArrayResize()`.

```mql4
double buffer[];
ArrayResize(buffer, 100);
buffer[0] = 1.23456;
```

### Multi-Dimensional Arrays

```mql4
double matrix[3][4];        // Static 3x4 matrix
double grid[][5];            // Dynamic first dimension, fixed second
ArrayResize(grid, 10);       // Now 10x5
```

### Array Functions

```mql4
// int ArrayResize(array&, int new_size, int reserve_size=0)
// Resizes a dynamic array. Returns new size or -1 on failure.
double data[];
int newSize = ArrayResize(data, 200);

// int ArraySize(const array&)
// Returns the total number of elements.
int count = ArraySize(data); // 200

// int ArrayCopy(dst&, const src, int dst_start=0, int src_start=0, int count=WHOLE_ARRAY)
// Copies elements from one array to another.
double source[5] = {1.0, 2.0, 3.0, 4.0, 5.0};
double dest[];
ArrayResize(dest, 5);
ArrayCopy(dest, source);

// bool ArraySort(array&, int count=WHOLE_ARRAY, int start=0, int direction=MODE_ASCEND)
// Sorts a numeric array. direction: MODE_ASCEND or MODE_DESCEND.
double values[5] = {3.0, 1.0, 4.0, 1.5, 2.0};
ArraySort(values);  // {1.0, 1.5, 2.0, 3.0, 4.0}

// void ArrayInitialize(array&, double value)
// Fills all elements with the given value.
double arr[100];
ArrayInitialize(arr, 0.0);

// int ArrayMaximum(const array&, int count=WHOLE_ARRAY, int start=0)
// Returns the index of the maximum value.
int maxIdx = ArrayMaximum(values);

// int ArrayMinimum(const array&, int count=WHOLE_ARRAY, int start=0)
// Returns the index of the minimum value.
int minIdx = ArrayMinimum(values);

// void ArrayFree(array&)
// Frees memory of a dynamic array and sets size to 0.
ArrayFree(data);

// bool ArraySetAsSeries(array&, bool flag)
// Sets reverse indexing (index 0 = last element). Used for indicator buffers.
double buffer[];
ArraySetAsSeries(buffer, true); // Now buffer[0] is the newest element
```

---

## Strings

### String Functions

```mql4
// int StringLen(string value)
int len = StringLen("Hello"); // 5

// string StringSubstr(string value, int start, int length=-1)
string sub = StringSubstr("EURUSD", 0, 3); // "EUR"

// int StringFind(string value, string match, int start=0)
// Returns position or -1 if not found.
int pos = StringFind("EURUSD", "USD"); // 3

// int StringReplace(string& str, string find, string replacement)
// Returns number of replacements made.
string text = "Hello World";
int count = StringReplace(text, "World", "MQL4"); // text = "Hello MQL4"

// string StringConcatenate(...)
// Concatenates multiple values. Faster than + operator for many strings.
string result = StringConcatenate("Price: ", DoubleToString(Ask, 5), " Time: ", TimeToString(TimeCurrent()));

// string StringToUpper(string value)
string upper = "eurusd";
StringToUpper(upper); // upper = "EURUSD"
// Note: StringToLower() also available

// int StringToInteger(string value)
int num = StringToInteger("42"); // 42

// double StringToDouble(string value)
double val = StringToDouble("1.23456"); // 1.23456

// string IntegerToString(long value, int str_len=0, ushort fill=' ')
string s1 = IntegerToString(42);      // "42"
string s2 = IntegerToString(5, 3, '0'); // "005"

// string DoubleToString(double value, int digits=8)
string s3 = DoubleToString(1.23456, 5); // "1.23456"
string s4 = DoubleToString(1.23456, 2); // "1.23"

// Additional useful conversions
// string TimeToString(datetime value, int mode=TIME_DATE|TIME_MINUTES)
// datetime StringToTime(string value)
// string EnumToString(enum_value)
```

### String Formatting with StringFormat

```mql4
// Works like C printf
string msg = StringFormat("Symbol: %s | Price: %.5f | Spread: %d",
                           _Symbol, Ask, (int)MarketInfo(_Symbol, MODE_SPREAD));
// "Symbol: EURUSD | Price: 1.23456 | Spread: 12"
```

---

## Program Types

### Expert Advisors (EAs)

Automated trading programs that run on charts and can place/modify/close orders.

```mql4
//+------------------------------------------------------------------+
//|                                                      MyExpert.mq4 |
//|                                            Copyright 2024, Author |
//+------------------------------------------------------------------+
#property copyright "Copyright 2024, Author"
#property link      "https://www.example.com"
#property version   "1.00"
#property strict

//--- Input parameters
input int    InpMAPeriod    = 14;        // MA Period
input double InpLotSize     = 0.1;       // Lot Size
input int    InpStopLoss    = 50;        // Stop Loss (points)
input int    InpTakeProfit  = 100;       // Take Profit (points)
input int    InpMagicNumber = 12345;     // Magic Number
input int    InpSlippage    = 3;         // Slippage (points)

//--- Global variables
double g_pipSize;
int    g_pipDigits;

//+------------------------------------------------------------------+
//| Expert initialization function                                     |
//| Called once when the EA is loaded onto a chart.                     |
//| Return: INIT_SUCCEEDED, INIT_FAILED,                               |
//|         INIT_PARAMETERS_INCORRECT, INIT_AGENT_NOT_SUITABLE         |
//+------------------------------------------------------------------+
int OnInit()
{
   // Detect broker digit mode
   if(Digits == 5 || Digits == 3)
   {
      g_pipSize  = Point * 10;
      g_pipDigits = 1;
   }
   else
   {
      g_pipSize  = Point;
      g_pipDigits = 0;
   }

   // Set up timer (optional)
   EventSetTimer(60); // Fire OnTimer() every 60 seconds

   Print("EA initialized on ", _Symbol, " | PipSize=", g_pipSize);
   return INIT_SUCCEEDED;
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                    |
//| Called when the EA is removed or the terminal is closed.            |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   EventKillTimer();

   string reasonText;
   switch(reason)
   {
      case REASON_REMOVE:     reasonText = "Removed from chart"; break;
      case REASON_RECOMPILE:  reasonText = "Recompiled"; break;
      case REASON_CHARTCHANGE: reasonText = "Symbol or period changed"; break;
      case REASON_CHARTCLOSE: reasonText = "Chart closed"; break;
      case REASON_PARAMETERS: reasonText = "Inputs changed"; break;
      case REASON_ACCOUNT:    reasonText = "Account changed"; break;
      case REASON_TEMPLATE:   reasonText = "Template applied"; break;
      case REASON_INITFAILED: reasonText = "OnInit failed"; break;
      case REASON_CLOSE:      reasonText = "Terminal closed"; break;
      default:                reasonText = "Unknown"; break;
   }
   Print("EA deinitialized. Reason: ", reasonText);
}

//+------------------------------------------------------------------+
//| Expert tick function                                                |
//| Called on every new tick (price change) for the chart symbol.       |
//+------------------------------------------------------------------+
void OnTick()
{
   // Check if trading is allowed
   if(!IsTradeAllowed()) return;
   if(!IsConnected()) return;

   // New bar detection
   static datetime lastBarTime = 0;
   if(Time[0] == lastBarTime) return;
   lastBarTime = Time[0];

   // Trading logic here
   double maValue = iMA(NULL, 0, InpMAPeriod, 0, MODE_SMA, PRICE_CLOSE, 1);

   if(Close[1] > maValue && CountOrders(OP_BUY) == 0)
   {
      OpenOrder(OP_BUY);
   }
   else if(Close[1] < maValue && CountOrders(OP_SELL) == 0)
   {
      OpenOrder(OP_SELL);
   }
}

//+------------------------------------------------------------------+
//| Timer function                                                      |
//| Called at the interval set by EventSetTimer().                      |
//+------------------------------------------------------------------+
void OnTimer()
{
   // Periodic tasks: cleanup, status updates, etc.
   Print("Timer event at ", TimeToString(TimeCurrent()));
}

//+------------------------------------------------------------------+
//| ChartEvent function                                                 |
//| Called when chart events occur (clicks, key presses, objects).      |
//+------------------------------------------------------------------+
void OnChartEvent(const int id,
                  const long &lparam,
                  const double &dparam,
                  const string &sparam)
{
   if(id == CHARTEVENT_CLICK)
   {
      Print("Chart clicked at x=", lparam, " y=", dparam);
   }
   else if(id == CHARTEVENT_KEYDOWN)
   {
      Print("Key pressed: ", lparam);
   }
   else if(id == CHARTEVENT_OBJECT_CLICK)
   {
      Print("Object clicked: ", sparam);
   }
}

//+------------------------------------------------------------------+
//| Count open orders by type for this EA                               |
//+------------------------------------------------------------------+
int CountOrders(int orderType)
{
   int count = 0;
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!OrderSelect(i, SELECT_BY_POS, MODE_TRADES)) continue;
      if(OrderSymbol() != _Symbol) continue;
      if(OrderMagicNumber() != InpMagicNumber) continue;
      if(OrderType() == orderType) count++;
   }
   return count;
}

//+------------------------------------------------------------------+
//| Open a market order                                                 |
//+------------------------------------------------------------------+
bool OpenOrder(int orderType)
{
   double price, sl, tp;

   RefreshRates();

   if(orderType == OP_BUY)
   {
      price = Ask;
      sl = (InpStopLoss > 0) ? NormalizeDouble(Ask - InpStopLoss * Point, Digits) : 0;
      tp = (InpTakeProfit > 0) ? NormalizeDouble(Ask + InpTakeProfit * Point, Digits) : 0;
   }
   else
   {
      price = Bid;
      sl = (InpStopLoss > 0) ? NormalizeDouble(Bid + InpStopLoss * Point, Digits) : 0;
      tp = (InpTakeProfit > 0) ? NormalizeDouble(Bid - InpTakeProfit * Point, Digits) : 0;
   }

   int ticket = OrderSend(_Symbol, orderType, InpLotSize, price, InpSlippage,
                           sl, tp, "MyEA", InpMagicNumber, 0, clrGreen);

   if(ticket < 0)
   {
      Print("OrderSend failed. Error: ", GetLastError());
      return false;
   }

   Print("Order opened. Ticket: ", ticket);
   return true;
}
```

#### Deinit Reason Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `REASON_PROGRAM` | 0 | EA called `ExpertRemove()` |
| `REASON_REMOVE` | 1 | Removed from chart |
| `REASON_RECOMPILE` | 2 | Recompiled |
| `REASON_CHARTCHANGE` | 3 | Symbol or period changed |
| `REASON_CHARTCLOSE` | 4 | Chart closed |
| `REASON_PARAMETERS` | 5 | Input parameters changed |
| `REASON_ACCOUNT` | 6 | Account changed |
| `REASON_TEMPLATE` | 7 | New template applied |
| `REASON_INITFAILED` | 8 | `OnInit()` returned non-zero |
| `REASON_CLOSE` | 9 | Terminal closed |

---

### Custom Indicators

Calculate and display values on charts. Cannot trade. Run in the interface thread (must be fast).

#### Property Directives for Indicators

```mql4
#property indicator_chart_window         // Draw on main chart
// OR
#property indicator_separate_window      // Draw in separate sub-window

#property indicator_buffers 2            // Number of visible plot buffers
#property indicator_color1 clrDodgerBlue // Color for buffer 0
#property indicator_color2 clrRed        // Color for buffer 1
#property indicator_width1 2             // Line width for buffer 0
#property indicator_width2 1             // Line width for buffer 1
#property indicator_style1 STYLE_SOLID   // Line style for buffer 0
#property indicator_style2 STYLE_DOT     // Line style for buffer 1

// For separate window indicators
#property indicator_minimum 0            // Minimum scale
#property indicator_maximum 100          // Maximum scale
#property indicator_level1 30            // Horizontal level line
#property indicator_level2 70            // Another level line
#property indicator_levelcolor clrGray   // Level line color
#property indicator_levelstyle STYLE_DOT // Level line style
```

**Important:** `#property indicator_buffers N` sets the number of **plotted** (visible) buffers. If you need additional calculation buffers that are not drawn, use `IndicatorBuffers(total)` in `OnInit()` to set the **total** number of buffers (plotted + calculation). The total must be >= the `#property indicator_buffers` value.

```mql4
#property indicator_buffers 2   // 2 visible plots

double PlotBuffer1[];   // Visible buffer 0
double PlotBuffer2[];   // Visible buffer 1
double CalcBuffer[];    // Hidden calculation buffer

int OnInit()
{
   IndicatorBuffers(3); // Total = 2 visible + 1 calculation

   SetIndexBuffer(0, PlotBuffer1);
   SetIndexBuffer(1, PlotBuffer2);
   SetIndexBuffer(2, CalcBuffer);  // Not plotted (index >= indicator_buffers)

   return INIT_SUCCEEDED;
}
```

#### Draw Styles

| Constant | Description |
|----------|-------------|
| `DRAW_LINE` | Simple line connecting buffer values |
| `DRAW_HISTOGRAM` | Vertical bars from zero line to buffer value |
| `DRAW_ARROW` | Symbols/arrows at buffer values |
| `DRAW_NONE` | Not drawn (for calculation buffers or when using `IndicatorBuffers`) |
| `DRAW_SECTION` | Line segments between non-empty values |
| `DRAW_ZIGZAG` | Zigzag line (requires two buffers) |

#### Full Indicator Template

```mql4
//+------------------------------------------------------------------+
//|                                                 MyIndicator.mq4   |
//+------------------------------------------------------------------+
#property copyright "Copyright 2024, Author"
#property link      "https://www.example.com"
#property version   "1.00"
#property strict

#property indicator_separate_window
#property indicator_buffers 2
#property indicator_color1  clrDodgerBlue
#property indicator_color2  clrRed
#property indicator_width1  2
#property indicator_width2  1
#property indicator_level1  30
#property indicator_level2  70
#property indicator_minimum 0
#property indicator_maximum 100

//--- Input parameters
input int InpPeriod = 14; // Period

//--- Indicator buffers
double MainBuffer[];
double SignalBuffer[];

//+------------------------------------------------------------------+
int OnInit()
{
   // Bind arrays to indicator buffers
   SetIndexBuffer(0, MainBuffer);
   SetIndexBuffer(1, SignalBuffer);

   // Set drawing styles
   SetIndexStyle(0, DRAW_LINE, STYLE_SOLID, 2);
   SetIndexStyle(1, DRAW_LINE, STYLE_DOT, 1);

   // Set labels (shown in Data Window)
   SetIndexLabel(0, "Main");
   SetIndexLabel(1, "Signal");

   // Set indicator name in sub-window
   IndicatorShortName("MyIndicator(" + IntegerToString(InpPeriod) + ")");

   // Set how many initial bars to skip
   SetIndexDrawBegin(0, InpPeriod);
   SetIndexDrawBegin(1, InpPeriod);

   return INIT_SUCCEEDED;
}

//+------------------------------------------------------------------+
//| OnCalculate - called on every tick and on history load             |
//|                                                                    |
//| rates_total   - total number of bars available                     |
//| prev_calculated - bars calculated on previous call (0 on first)    |
//| time[]        - bar open times                                     |
//| open[]        - bar open prices                                    |
//| high[]        - bar high prices                                    |
//| low[]         - bar low prices                                     |
//| close[]       - bar close prices                                   |
//| tick_volume[] - tick volumes                                       |
//| volume[]      - real volumes                                       |
//| spread[]      - spreads                                            |
//+------------------------------------------------------------------+
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
   // Insufficient bars
   if(rates_total < InpPeriod) return 0;

   // Determine starting index
   int start;
   if(prev_calculated == 0)
      start = InpPeriod;  // First calculation
   else
      start = prev_calculated - 1;  // Recalculate only new bars

   // Main calculation loop (oldest to newest)
   for(int i = start; i < rates_total; i++)
   {
      // Example: simple calculation
      double sum = 0;
      for(int j = 0; j < InpPeriod; j++)
         sum += close[i - j];

      MainBuffer[i]   = sum / InpPeriod;
      SignalBuffer[i]  = MainBuffer[i] * 0.95;
   }

   return rates_total; // Return for next prev_calculated
}
```

#### SetIndexBuffer / SetIndexStyle / SetIndexLabel

```mql4
// bool SetIndexBuffer(int index, double array[])
// Binds an array to an indicator buffer at the given index.
SetIndexBuffer(0, MainBuffer);

// void SetIndexStyle(int index, int type, int style=EMPTY, int width=EMPTY, color clr=CLR_NONE)
// type: DRAW_LINE, DRAW_HISTOGRAM, DRAW_ARROW, DRAW_NONE, etc.
// style: STYLE_SOLID, STYLE_DASH, STYLE_DOT, STYLE_DASHDOT, STYLE_DASHDOTDOT
SetIndexStyle(0, DRAW_LINE, STYLE_SOLID, 2, clrBlue);

// void SetIndexLabel(int index, string text)
// Sets the label shown in the Data Window. Use NULL to hide.
SetIndexLabel(0, "Fast MA");
SetIndexLabel(1, NULL); // Hidden from Data Window

// void SetIndexArrow(int index, int code)
// Sets the Wingdings symbol code for DRAW_ARROW style.
SetIndexStyle(2, DRAW_ARROW);
SetIndexArrow(2, 233); // Up arrow
SetIndexArrow(3, 234); // Down arrow

// void SetIndexDrawBegin(int index, int begin)
// Sets the bar number from which drawing starts.
SetIndexDrawBegin(0, InpPeriod);

// void SetIndexEmptyValue(int index, double value)
// Sets the value considered "empty" (not drawn). Default is EMPTY_VALUE.
SetIndexEmptyValue(0, 0.0);
```

---

### Scripts

Execute once and terminate. Can trade. Useful for one-time operations.

```mql4
//+------------------------------------------------------------------+
//|                                                    CloseAll.mq4   |
//+------------------------------------------------------------------+
#property copyright "Copyright 2024, Author"
#property link      "https://www.example.com"
#property version   "1.00"
#property strict
#property show_inputs  // Show input dialog before running

input int InpMagicNumber = 0; // Magic Number (0 = all)

//+------------------------------------------------------------------+
//| Script start function - called once when script is launched        |
//+------------------------------------------------------------------+
void OnStart()
{
   int closed = 0;

   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!OrderSelect(i, SELECT_BY_POS, MODE_TRADES)) continue;
      if(OrderSymbol() != _Symbol) continue;
      if(InpMagicNumber != 0 && OrderMagicNumber() != InpMagicNumber) continue;

      bool result = false;
      RefreshRates();

      if(OrderType() == OP_BUY)
         result = OrderClose(OrderTicket(), OrderLots(), Bid, 3, clrRed);
      else if(OrderType() == OP_SELL)
         result = OrderClose(OrderTicket(), OrderLots(), Ask, 3, clrRed);
      else
         result = OrderDelete(OrderTicket());

      if(result) closed++;
      else Print("Failed to close order #", OrderTicket(), " Error: ", GetLastError());
   }

   Print("Closed ", closed, " orders");
}
```

---

### Libraries

Reusable function collections. Cannot run independently.

```mql4
//+------------------------------------------------------------------+
//|                                                  TradeLib.mq4     |
//+------------------------------------------------------------------+
#property library
#property copyright "Copyright 2024, Author"
#property strict

//--- Exported function (callable from other programs)
double CalculateLotSize(double riskPercent, double stopLossPips) export
{
   double accountRisk = AccountBalance() * riskPercent / 100.0;
   double tickValue   = MarketInfo(Symbol(), MODE_TICKVALUE);
   double tickSize    = MarketInfo(Symbol(), MODE_TICKSIZE);
   double lotStep     = MarketInfo(Symbol(), MODE_LOTSTEP);
   double minLot      = MarketInfo(Symbol(), MODE_MINLOT);
   double maxLot      = MarketInfo(Symbol(), MODE_MAXLOT);

   if(tickValue == 0 || stopLossPips == 0) return minLot;

   double lots = accountRisk / (stopLossPips * (tickValue / tickSize));
   lots = MathFloor(lots / lotStep) * lotStep;
   lots = MathMax(minLot, MathMin(maxLot, lots));

   return NormalizeDouble(lots, 2);
}
```

#### Importing a Library

```mql4
// Import from compiled .ex4 library
#import "TradeLib.ex4"
   double CalculateLotSize(double riskPercent, double stopLossPips);
#import

// Import from Windows DLL
#import "user32.dll"
   int MessageBoxW(int hWnd, string text, string caption, int type);
#import

// Usage
void OnTick()
{
   double lots = CalculateLotSize(2.0, 50);
}
```

---

## Predefined Variables

These are built-in variables updated by the terminal. Available in all program types.

| Variable | Type | Description |
|----------|------|-------------|
| `Ask` | `double` | Current ask price (buy price) |
| `Bid` | `double` | Current bid price (sell price) |
| `Bars` | `int` | Number of bars on the current chart |
| `Point` | `double` | Point size (0.00001 for 5-digit, 0.0001 for 4-digit) |
| `Digits` | `int` | Number of decimal places (5 or 3 for 5-digit brokers) |
| `_Symbol` | `string` | Current chart symbol (same as `Symbol()`) |
| `_Period` | `int` | Current chart timeframe in minutes (same as `Period()`) |
| `Open[]` | `double` | Array of bar open prices |
| `High[]` | `double` | Array of bar high prices |
| `Low[]` | `double` | Array of bar low prices |
| `Close[]` | `double` | Array of bar close prices |
| `Time[]` | `datetime` | Array of bar open times |
| `Volume[]` | `long` | Array of bar tick volumes |

**Important:** The series arrays (`Open[]`, `High[]`, `Low[]`, `Close[]`, `Time[]`, `Volume[]`) use reverse indexing: **index 0 = the newest (current) bar**. Index 1 = previous bar, etc.

```mql4
double currentClose  = Close[0];   // Current bar's close
double previousHigh  = High[1];    // Previous bar's high
datetime barTime     = Time[0];    // Current bar's open time
double currentSpread = Ask - Bid;  // Current spread

// IMPORTANT: Call RefreshRates() before using Ask/Bid in order functions
// to ensure you have the latest prices.
RefreshRates();
double freshAsk = Ask;
double freshBid = Bid;
```

---

## Technical Indicator Functions

All built-in indicator functions follow the pattern:
`iFunction(symbol, timeframe, ...parameters, shift)`

- `symbol`: Use `NULL` or `_Symbol` for the current chart symbol.
- `timeframe`: Use `0` or `PERIOD_CURRENT` for the current chart timeframe.
- `shift`: Bar index (0 = current bar, 1 = previous bar, etc.).

### Moving Average - iMA

```mql4
double iMA(
   string symbol,        // Symbol name
   int    timeframe,     // Timeframe (PERIOD_*)
   int    period,        // Averaging period
   int    ma_shift,      // Horizontal shift (bars)
   int    ma_method,     // MA method (MODE_SMA, etc.)
   int    applied_price, // Applied price (PRICE_CLOSE, etc.)
   int    shift          // Bar index
);

// Example
double sma20 = iMA(NULL, 0, 20, 0, MODE_SMA, PRICE_CLOSE, 0);
double ema50 = iMA(NULL, PERIOD_H1, 50, 0, MODE_EMA, PRICE_CLOSE, 1);
```

### Relative Strength Index - iRSI

```mql4
double iRSI(
   string symbol,        // Symbol
   int    timeframe,     // Timeframe
   int    period,        // Averaging period
   int    applied_price, // Applied price
   int    shift          // Bar index
);

// Example
double rsi = iRSI(NULL, 0, 14, PRICE_CLOSE, 0);
if(rsi < 30) Print("Oversold");
if(rsi > 70) Print("Overbought");
```

### MACD - iMACD

```mql4
double iMACD(
   string symbol,          // Symbol
   int    timeframe,       // Timeframe
   int    fast_ema_period, // Fast EMA period (typically 12)
   int    slow_ema_period, // Slow EMA period (typically 26)
   int    signal_period,   // Signal line period (typically 9)
   int    applied_price,   // Applied price
   int    mode,            // Line index: MODE_MAIN or MODE_SIGNAL
   int    shift            // Bar index
);

// Example
double macdMain   = iMACD(NULL, 0, 12, 26, 9, PRICE_CLOSE, MODE_MAIN, 0);
double macdSignal = iMACD(NULL, 0, 12, 26, 9, PRICE_CLOSE, MODE_SIGNAL, 0);

if(macdMain > macdSignal) Print("Bullish crossover");
```

### Stochastic Oscillator - iStochastic

```mql4
double iStochastic(
   string symbol,      // Symbol
   int    timeframe,   // Timeframe
   int    Kperiod,     // %K period
   int    Dperiod,     // %D period (signal)
   int    slowing,     // Slowing
   int    method,      // MA method for smoothing
   int    price_field, // Price field: 0 = Low/High, 1 = Close/Close
   int    mode,        // MODE_MAIN (%K) or MODE_SIGNAL (%D)
   int    shift        // Bar index
);

// Example
double stochK = iStochastic(NULL, 0, 5, 3, 3, MODE_SMA, 0, MODE_MAIN, 0);
double stochD = iStochastic(NULL, 0, 5, 3, 3, MODE_SMA, 0, MODE_SIGNAL, 0);
```

### Bollinger Bands - iBands

```mql4
double iBands(
   string symbol,        // Symbol
   int    timeframe,     // Timeframe
   int    period,        // Averaging period
   double deviation,     // Standard deviation multiplier
   int    bands_shift,   // Horizontal shift (bars)
   int    applied_price, // Applied price
   int    mode,          // MODE_MAIN (middle), MODE_UPPER, MODE_LOWER
   int    shift          // Bar index
);

// Example
double upper  = iBands(NULL, 0, 20, 2.0, 0, PRICE_CLOSE, MODE_UPPER, 0);
double middle = iBands(NULL, 0, 20, 2.0, 0, PRICE_CLOSE, MODE_MAIN, 0);
double lower  = iBands(NULL, 0, 20, 2.0, 0, PRICE_CLOSE, MODE_LOWER, 0);
```

### Average True Range - iATR

```mql4
double iATR(
   string symbol,    // Symbol
   int    timeframe, // Timeframe
   int    period,    // Averaging period
   int    shift      // Bar index
);

// Example
double atr = iATR(NULL, 0, 14, 0);
double dynamicSL = NormalizeDouble(Ask - atr * 1.5, Digits);
```

### Custom Indicator - iCustom

```mql4
double iCustom(
   string symbol,      // Symbol
   int    timeframe,   // Timeframe
   string name,        // Custom indicator file name (without .ex4)
   ...                  // Indicator input parameters (in order)
   int    mode,        // Buffer index (0-based)
   int    shift        // Bar index
);

// Example: Call a custom indicator "SuperTrend" with inputs (10, 3.0)
double superTrend = iCustom(NULL, 0, "SuperTrend", 10, 3.0, 0, 0);

// The parameter order must match the indicator's input parameters exactly.
// mode corresponds to the SetIndexBuffer() index in the indicator.
```

### MA Method Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `MODE_SMA` | 0 | Simple Moving Average |
| `MODE_EMA` | 1 | Exponential Moving Average |
| `MODE_SMMA` | 2 | Smoothed Moving Average |
| `MODE_LWMA` | 3 | Linear Weighted Moving Average |

### Applied Price Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `PRICE_CLOSE` | 0 | Close price |
| `PRICE_OPEN` | 1 | Open price |
| `PRICE_HIGH` | 2 | High price |
| `PRICE_LOW` | 3 | Low price |
| `PRICE_MEDIAN` | 4 | (High + Low) / 2 |
| `PRICE_TYPICAL` | 5 | (High + Low + Close) / 3 |
| `PRICE_WEIGHTED` | 6 | (High + Low + Close + Close) / 4 |

### Timeframe Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `PERIOD_CURRENT` | 0 | Current chart timeframe |
| `PERIOD_M1` | 1 | 1 minute |
| `PERIOD_M5` | 5 | 5 minutes |
| `PERIOD_M15` | 15 | 15 minutes |
| `PERIOD_M30` | 30 | 30 minutes |
| `PERIOD_H1` | 60 | 1 hour |
| `PERIOD_H4` | 240 | 4 hours |
| `PERIOD_D1` | 1440 | Daily |
| `PERIOD_W1` | 10080 | Weekly |
| `PERIOD_MN1` | 43200 | Monthly |

---

## Order Management

### Order Types

| Constant | Value | Description |
|----------|-------|-------------|
| `OP_BUY` | 0 | Market buy order |
| `OP_SELL` | 1 | Market sell order |
| `OP_BUYLIMIT` | 2 | Buy limit pending order (below market) |
| `OP_SELLLIMIT` | 3 | Sell limit pending order (above market) |
| `OP_BUYSTOP` | 4 | Buy stop pending order (above market) |
| `OP_SELLSTOP` | 5 | Sell stop pending order (below market) |

### Trading Functions

```mql4
// OrderSend - Returns ticket on success, -1 on failure
int OrderSend(string symbol, int cmd, double volume, double price,
              int slippage, double stoploss, double takeprofit,
              string comment, int magic, datetime expiration, color arrow_color);

// OrderModify - Returns true on success
bool OrderModify(int ticket, double price, double stoploss,
                 double takeprofit, datetime expiration, color arrow_color);

// OrderClose - Close market order (can be partial)
bool OrderClose(int ticket, double lots, double price, int slippage, color Color);

// OrderDelete - Delete pending order only
bool OrderDelete(int ticket, color Color);

// OrderSelect - Must call before reading order properties
bool OrderSelect(int index, int select, int pool);
// select: SELECT_BY_POS or SELECT_BY_TICKET
// pool: MODE_TRADES or MODE_HISTORY

int OrdersTotal();         // Open + pending count
int OrdersHistoryTotal();  // Closed + deleted count
```

### Order Information Functions

After `OrderSelect()`, these return info about the selected order:

| Function | Return | Description |
|----------|--------|-------------|
| `OrderTicket()` | `int` | Ticket number |
| `OrderType()` | `int` | Order type (OP_BUY, etc.) |
| `OrderLots()` | `double` | Volume |
| `OrderOpenPrice()` | `double` | Open price |
| `OrderClosePrice()` | `double` | Close price (0 if open) |
| `OrderOpenTime()` | `datetime` | Open time |
| `OrderCloseTime()` | `datetime` | Close time (0 if open) |
| `OrderStopLoss()` | `double` | Stop loss |
| `OrderTakeProfit()` | `double` | Take profit |
| `OrderMagicNumber()` | `int` | Magic number |
| `OrderComment()` | `string` | Comment |
| `OrderSymbol()` | `string` | Symbol |
| `OrderProfit()` | `double` | Net profit |
| `OrderCommission()` | `double` | Commission |
| `OrderSwap()` | `double` | Swap |
| `OrderExpiration()` | `datetime` | Expiration |

> **Production patterns** (retry logic, order loops, close all, error handling): See [trading-operations.md](trading-operations.md)

---

## Market Information

### MarketInfo

```mql4
double MarketInfo(string symbol, int type);
```

| Constant | Description |
|----------|-------------|
| `MODE_SPREAD` | Spread in points |
| `MODE_STOPLEVEL` | Minimum stop level in points |
| `MODE_LOTSIZE` | Contract size (e.g., 100000 for forex) |
| `MODE_TICKVALUE` | Tick value in account currency |
| `MODE_TICKSIZE` | Tick size (minimum price change) |
| `MODE_SWAPLONG` | Swap for long positions |
| `MODE_SWAPSHORT` | Swap for short positions |
| `MODE_MINLOT` | Minimum lot size |
| `MODE_LOTSTEP` | Lot size step |
| `MODE_MAXLOT` | Maximum lot size |
| `MODE_MARGINREQUIRED` | Margin required for 1 lot |
| `MODE_DIGITS` | Number of decimal digits |
| `MODE_POINT` | Point size |
| `MODE_FREEZELEVEL` | Freeze distance in points (cannot modify orders within this distance) |

```mql4
double spread    = MarketInfo(_Symbol, MODE_SPREAD);
double stopLevel = MarketInfo(_Symbol, MODE_STOPLEVEL);
double minLot    = MarketInfo(_Symbol, MODE_MINLOT);
double lotStep   = MarketInfo(_Symbol, MODE_LOTSTEP);
double maxLot    = MarketInfo(_Symbol, MODE_MAXLOT);
double tickVal   = MarketInfo(_Symbol, MODE_TICKVALUE);
double tickSize  = MarketInfo(_Symbol, MODE_TICKSIZE);
double margin1   = MarketInfo(_Symbol, MODE_MARGINREQUIRED);
int    digits    = (int)MarketInfo(_Symbol, MODE_DIGITS);
double point     = MarketInfo(_Symbol, MODE_POINT);
```

### SymbolInfoDouble (Modern Replacement)

```mql4
double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
double tickSize  = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);

long spread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD);
long digits = SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);
long stopLevel = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL);
```

---

## Account Functions

| Function | Return Type | Description |
|----------|-------------|-------------|
| `AccountBalance()` | `double` | Account balance |
| `AccountEquity()` | `double` | Account equity (balance + floating P/L) |
| `AccountFreeMargin()` | `double` | Free margin available |
| `AccountMargin()` | `double` | Margin currently used |
| `AccountProfit()` | `double` | Total floating profit/loss |
| `AccountCurrency()` | `string` | Account currency (e.g., "USD") |
| `AccountNumber()` | `int` | Account number |
| `AccountCompany()` | `string` | Broker company name |
| `AccountLeverage()` | `int` | Account leverage (e.g., 100) |
| `AccountName()` | `string` | Account holder name |
| `AccountServer()` | `string` | Trade server name |
| `AccountStopoutLevel()` | `int` | Stop-out level (margin %) |
| `AccountStopoutMode()` | `int` | Stop-out mode (0=percent, 1=money) |
| `AccountInfoDouble(prop)` | `double` | Modern replacement |
| `AccountInfoInteger(prop)` | `long` | Modern replacement |
| `AccountInfoString(prop)` | `string` | Modern replacement |

```mql4
// Risk calculation example
double riskPercent = 2.0;
double balance = AccountBalance();
double riskAmount = balance * riskPercent / 100.0;

Print("Account: ", AccountNumber(), " at ", AccountCompany());
Print("Balance: ", balance, " ", AccountCurrency());
Print("Equity:  ", AccountEquity());
Print("Margin:  ", AccountMargin());
Print("Free:    ", AccountFreeMargin());
Print("Leverage: 1:", AccountLeverage());
```

---

## Preprocessor Directives

### #property

```mql4
// Common EA/indicator/script properties
#property copyright   "Copyright 2024, Author"
#property link        "https://www.example.com"
#property version     "1.00"
#property description "My Expert Advisor"
#property strict      // Enable strict compilation mode (recommended)

// Indicator-specific
#property indicator_chart_window
#property indicator_separate_window
#property indicator_buffers    3
#property indicator_color1     clrBlue
#property indicator_color2     clrRed
#property indicator_color3     clrGreen
#property indicator_width1     2
#property indicator_style1     STYLE_SOLID
#property indicator_minimum    0
#property indicator_maximum    100
#property indicator_level1     20
#property indicator_level2     80
#property indicator_levelcolor clrSilver

// Script-specific
#property show_inputs           // Show inputs dialog
#property show_confirm          // Show confirmation dialog

// Library-specific
#property library               // Mark as library
```

### #include

```mql4
// Include from MQL4/Include/ directory
#include <stdlib.mqh>           // Standard library (ErrorDescription, etc.)
#include <stderror.mqh>         // Error code constants

// Include from same directory as the source file
#include "MyHelpers.mqh"

// Include guards (manual, MQL4 has no #pragma once)
#ifndef MY_HELPERS_MQH
#define MY_HELPERS_MQH
// ... header content ...
#endif
```

### #define

```mql4
// Simple constants
#define EA_NAME    "SuperScalper"
#define EA_VERSION "2.1"
#define MAGIC      20240115

// Macros with parameters
#define MAX(a, b) ((a) > (b) ? (a) : (b))
#define MIN(a, b) ((a) < (b) ? (a) : (b))
#define ABS(x)    ((x) >= 0 ? (x) : -(x))
#define PIP(x)    ((x) * g_pipSize)

// Multi-line macro (use backslash)
#define LOG(msg) \
   Print(__FILE__, " Line ", __LINE__, ": ", msg)

// Predefined macros
// __FILE__     - Current file name
// __LINE__     - Current line number
// __FUNCTION__ - Current function name
// __DATETIME__ - Compilation date/time
// __MQLBUILD__ - MQL compiler build number
```

### #import

```mql4
// Import from compiled MQL4 library (.ex4)
#import "MyLibrary.ex4"
   double CalculateRisk(double percent, int stopPoints);
   int    CountSignals(string symbol, int timeframe);
#import

// Import from Windows DLL
#import "kernel32.dll"
   int GetTickCount();
   void Sleep(int milliseconds);
#import

#import "user32.dll"
   int MessageBoxW(int hWnd, string text, string caption, int type);
#import
```

---

## Error Handling

### Error Functions

```mql4
// int GetLastError()
// Returns the last error code and resets the internal error variable.
int err = GetLastError();

// void ResetLastError()
// Resets the error variable to ERR_NO_ERROR (0).
ResetLastError();

// int _LastError
// Predefined variable holding the last error code (NOT reset by reading).

// string ErrorDescription(int error_code)
// Requires: #include <stdlib.mqh>
#include <stdlib.mqh>
string desc = ErrorDescription(GetLastError());
```

### Trade Error Codes

| Constant | Value | Description |
|----------|-------|-------------|
| `ERR_NO_ERROR` | 0 | No error |
| `ERR_NO_RESULT` | 1 | No error but result unknown |
| `ERR_COMMON_ERROR` | 2 | Common error |
| `ERR_INVALID_TRADE_PARAMETERS` | 3 | Invalid trade parameters |
| `ERR_SERVER_BUSY` | 4 | Trade server is busy |
| `ERR_OLD_VERSION` | 5 | Old client terminal version |
| `ERR_NO_CONNECTION` | 6 | No connection to server |
| `ERR_NOT_ENOUGH_RIGHTS` | 7 | Insufficient rights |
| `ERR_TOO_FREQUENT_REQUESTS` | 8 | Too frequent requests |
| `ERR_MALFUNCTIONAL_TRADE` | 9 | Malfunctional trade operation |
| `ERR_ACCOUNT_DISABLED` | 64 | Account disabled |
| `ERR_INVALID_ACCOUNT` | 65 | Invalid account |
| `ERR_TRADE_TIMEOUT` | 128 | Trade timeout |
| `ERR_INVALID_PRICE` | 129 | Invalid price |
| `ERR_INVALID_STOPS` | 130 | Invalid stops |
| `ERR_INVALID_TRADE_VOLUME` | 131 | Invalid lot size |
| `ERR_MARKET_CLOSED` | 132 | Market closed |
| `ERR_TRADE_DISABLED` | 133 | Trading disabled |
| `ERR_NOT_ENOUGH_MONEY` | 134 | Not enough money |
| `ERR_PRICE_CHANGED` | 135 | Price changed (requote) |
| `ERR_OFF_QUOTES` | 136 | Off quotes |
| `ERR_BROKER_BUSY` | 137 | Broker busy |
| `ERR_REQUOTE` | 138 | Requote |
| `ERR_ORDER_LOCKED` | 139 | Order locked |
| `ERR_LONG_POSITIONS_ONLY_ALLOWED` | 140 | Long positions only |
| `ERR_TOO_MANY_REQUESTS` | 141 | Too many requests |
| `ERR_TRADE_MODIFY_DENIED` | 145 | Modification denied (too close to market) |
| `ERR_TRADE_CONTEXT_BUSY` | 146 | Trade context busy |
| `ERR_TRADE_EXPIRATION_DENIED` | 147 | Expiration denied by broker |
| `ERR_TRADE_TOO_MANY_ORDERS` | 148 | Too many open orders |
| `ERR_TRADE_HEDGE_PROHIBITED` | 149 | Hedging prohibited |
| `ERR_TRADE_PROHIBITED_BY_FIFO` | 150 | FIFO rule violation |

### Runtime Error Codes

| Constant | Value | Description |
|----------|-------|-------------|
| `ERR_NO_MQLERROR` | 4000 | No MQL error |
| `ERR_WRONG_FUNCTION_POINTER` | 4001 | Wrong function pointer |
| `ERR_ARRAY_INDEX_OUT_OF_RANGE` | 4002 | Array index out of range |
| `ERR_NO_MEMORY_FOR_CALL_STACK` | 4003 | No memory for function call stack |
| `ERR_RECURSIVE_STACK_OVERFLOW` | 4004 | Recursive stack overflow |
| `ERR_NOT_ENOUGH_STACK_FOR_PARAM` | 4005 | Not enough stack for parameter |
| `ERR_NO_MEMORY_FOR_PARAM_STRING` | 4006 | No memory for parameter string |
| `ERR_NO_MEMORY_FOR_TEMP_STRING` | 4007 | No memory for temp string |
| `ERR_NOT_INITIALIZED_STRING` | 4008 | Not initialized string |
| `ERR_NOT_INITIALIZED_ARRAYSTRING` | 4009 | Not initialized string array |
| `ERR_NO_MEMORY_FOR_ARRAYSTRING` | 4010 | No memory for string array |
| `ERR_TOO_LONG_STRING` | 4011 | String too long |
| `ERR_ZERO_DIVIDE` | 4013 | Division by zero |
| `ERR_UNKNOWN_COMMAND` | 4014 | Unknown command |
| `ERR_WRONG_JUMP` | 4015 | Wrong jump |
| `ERR_NOT_INITIALIZED_ARRAY` | 4016 | Array not initialized |
| `ERR_CUSTOM_INDICATOR_ERROR` | 4055 | Custom indicator error |
| `ERR_INCOMPATIBLE_ARRAYS` | 4056 | Incompatible arrays |
| `ERR_INVALID_FUNCTION_PARAMSCNT` | 4059 | Invalid function parameters count |
| `ERR_INVALID_FUNCTION_PARAMVALUE` | 4063 | Invalid function parameter value |
| `ERR_STRING_FUNCTION_INTERNAL` | 4062 | String function internal error |
| `ERR_ARRAY_AS_PARAMETER_EXPECTED` | 4065 | Array as parameter expected |
| `ERR_NO_ORDER_SELECTED` | 4105 | No order selected |
| `ERR_UNKNOWN_SYMBOL` | 4106 | Unknown symbol |
| `ERR_INVALID_PRICE_PARAM` | 4107 | Invalid price parameter |
| `ERR_INVALID_TICKET` | 4108 | Invalid ticket |
| `ERR_TRADE_NOT_ALLOWED` | 4109 | Trade not allowed (EA or AutoTrading disabled) |
| `ERR_LONGS_NOT_ALLOWED` | 4110 | Longs not allowed |
| `ERR_SHORTS_NOT_ALLOWED` | 4111 | Shorts not allowed |
| `ERR_OBJECT_ALREADY_EXISTS` | 4200 | Object already exists |
| `ERR_UNKNOWN_OBJECT_PROPERTY` | 4201 | Unknown object property |
| `ERR_OBJECT_DOES_NOT_EXIST` | 4202 | Object does not exist |
| `ERR_UNKNOWN_OBJECT_TYPE` | 4203 | Unknown object type |

### Robust Error Handling Pattern with Retry Logic

```mql4
#include <stdlib.mqh>

int SendOrderWithRetry(string symbol, int cmd, double volume, double price,
                       int slippage, double sl, double tp, string comment,
                       int magic, int maxRetries = 5)
{
   int ticket = -1;

   for(int attempt = 0; attempt < maxRetries; attempt++)
   {
      ResetLastError();
      RefreshRates();

      // Update price for market orders
      if(cmd == OP_BUY)  price = MarketInfo(symbol, MODE_ASK);
      if(cmd == OP_SELL) price = MarketInfo(symbol, MODE_BID);

      ticket = OrderSend(symbol, cmd, volume, price, slippage,
                          sl, tp, comment, magic, 0, clrGreen);

      if(ticket >= 0)
      {
         Print("Order sent successfully. Ticket: ", ticket);
         return ticket;
      }

      int err = GetLastError();
      Print("OrderSend attempt ", attempt + 1, " failed. Error ", err,
            ": ", ErrorDescription(err));

      switch(err)
      {
         case ERR_NO_ERROR:
            return ticket;

         // Retriable errors
         case ERR_SERVER_BUSY:
         case ERR_TRADE_TIMEOUT:
         case ERR_PRICE_CHANGED:
         case ERR_REQUOTE:
         case ERR_OFF_QUOTES:
         case ERR_BROKER_BUSY:
         case ERR_TRADE_CONTEXT_BUSY:
         case ERR_TOO_FREQUENT_REQUESTS:
         case ERR_TOO_MANY_REQUESTS:
            Sleep(1000 * (attempt + 1)); // Progressive delay
            break;

         // Fatal errors - stop retrying
         case ERR_INVALID_TRADE_PARAMETERS:
         case ERR_INVALID_STOPS:
         case ERR_INVALID_TRADE_VOLUME:
         case ERR_NOT_ENOUGH_MONEY:
         case ERR_TRADE_DISABLED:
         case ERR_MARKET_CLOSED:
         case ERR_ACCOUNT_DISABLED:
         case ERR_TRADE_NOT_ALLOWED:
            Print("Fatal trade error. Aborting.");
            return -1;

         default:
            Sleep(500);
            break;
      }
   }

   Print("OrderSend failed after ", maxRetries, " attempts");
   return -1;
}
```

---

## Common Gotchas

### 1. Double Comparison with NormalizeDouble

Floating-point numbers cannot be compared directly due to precision issues.

```mql4
// WRONG: Direct comparison
if(price1 == price2) // May fail due to floating-point precision

// CORRECT: Use NormalizeDouble
if(NormalizeDouble(price1 - price2, Digits) == 0)

// Or use a helper function
bool IsEqual(double a, double b, int digits)
{
   return NormalizeDouble(a - b, digits) == 0;
}

bool IsGreater(double a, double b, int digits)
{
   return NormalizeDouble(a - b, digits) > 0;
}

// IMPORTANT: Always NormalizeDouble prices before using in OrderSend/OrderModify
double sl = NormalizeDouble(Ask - 50 * Point, Digits);
double tp = NormalizeDouble(Ask + 100 * Point, Digits);
```

### 2. Four-Digit vs Five-Digit Broker Detection

Some brokers use 4-digit quotes (EURUSD = 1.2345), others use 5-digit (EURUSD = 1.23456). For JPY pairs, it is 2 vs 3 digits. You must handle this for pip calculations and slippage.

```mql4
double PipSize;
int    PipDigits;
int    SlippagePoints;

int OnInit()
{
   if(Digits == 5 || Digits == 3) // 5-digit broker
   {
      PipSize        = Point * 10;
      PipDigits      = Digits - 1;
      SlippagePoints = 30;   // 3.0 pips
   }
   else // 4-digit broker
   {
      PipSize        = Point;
      PipDigits      = Digits;
      SlippagePoints = 3;    // 3.0 pips
   }

   // Now use PipSize for calculations
   double stopLoss = 50 * PipSize; // 50 pips regardless of broker type
   return INIT_SUCCEEDED;
}
```

### 3. Slippage in Points, Not Pips

`OrderSend()` slippage parameter is always in **points** (the smallest price unit), not pips.

```mql4
// For a 5-digit broker where 1 pip = 10 points:
// If you want 3 pips slippage, use slippage = 30
int slippage = 3; // This is 3 POINTS = 0.3 pips on 5-digit!
// Correct for 3 pips:
int slippagePips = 3;
int slippagePoints = (Digits == 5 || Digits == 3) ? slippagePips * 10 : slippagePips;
```

### 4. RefreshRates() for Stale Prices

```mql4
// After long calculations or Sleep(), Ask/Bid may be stale
Sleep(5000);
RefreshRates(); // Update Ask, Bid, and predefined arrays
double freshAsk = Ask;
```

### 5. Stop Level Violations

The broker imposes a minimum distance for stop loss and take profit from the current price.

```mql4
double stopLevel = MarketInfo(_Symbol, MODE_STOPLEVEL) * Point;
double freezeLevel = MarketInfo(_Symbol, MODE_FREEZELEVEL) * Point;

// Ensure SL/TP are far enough from the current price
double sl = Ask - 50 * Point;
if(MathAbs(Ask - sl) < stopLevel)
{
   sl = NormalizeDouble(Ask - stopLevel - Point, Digits);
   Print("SL adjusted to meet stop level requirement");
}
```

### 6. Trade Context Busy (IsTradeAllowed)

Only one EA can send trade requests at a time in MT4.

```mql4
// Check if trading is allowed
if(!IsTradeAllowed())
{
   Print("Trade context is busy. Waiting...");
   // Wait for trade context
   int attempts = 0;
   while(!IsTradeAllowed() && attempts < 50)
   {
      Sleep(100);
      attempts++;
   }
   if(!IsTradeAllowed())
   {
      Print("Trade context still busy after waiting. Aborting.");
      return;
   }
}

// Also check these conditions
if(!IsConnected())      { Print("Not connected to server"); return; }
if(!IsTradeAllowed())   { Print("AutoTrading is disabled"); return; }
if(IsStopped())         { Print("EA is being stopped"); return; }
if(!IsExpertEnabled())  { Print("Expert Advisors disabled"); return; }
```

### 7. ECN Two-Step Order Placement

ECN/STP brokers often reject orders with SL/TP set at the time of opening. You must open the order first without SL/TP, then modify.

```mql4
bool OpenOrderECN(int type, double lots, double sl, double tp,
                   string comment, int magic)
{
   double price = (type == OP_BUY) ? Ask : Bid;

   // Step 1: Open order WITHOUT SL/TP
   int ticket = OrderSend(_Symbol, type, lots, price, 3,
                           0, 0, comment, magic, 0, clrGreen);

   if(ticket < 0)
   {
      Print("OrderSend failed: ", GetLastError());
      return false;
   }

   // Step 2: Modify to add SL/TP
   if(sl != 0 || tp != 0)
   {
      sl = NormalizeDouble(sl, Digits);
      tp = NormalizeDouble(tp, Digits);

      if(!OrderModify(ticket, price, sl, tp, 0, clrBlue))
      {
         Print("OrderModify failed: ", GetLastError(),
               " - Order is open but without SL/TP!");
      }
   }

   return true;
}
```

---

## File Operations

### File Modes

| Constant | Description |
|----------|-------------|
| `FILE_READ` | Open for reading |
| `FILE_WRITE` | Open for writing (creates/overwrites) |
| `FILE_BIN` | Binary mode |
| `FILE_CSV` | CSV mode (comma-separated) |
| `FILE_TXT` | Text mode |
| `FILE_COMMON` | Use common data folder (shared between terminals) |
| `FILE_ANSI` | ANSI encoding |
| `FILE_UNICODE` | Unicode encoding |

**Security:** File operations are sandboxed to `MQL4/Files/` directory (or `Terminal/Common/Files/` with `FILE_COMMON`). You cannot access files outside these directories.

### File Functions

```mql4
// int FileOpen(string filename, int flags, short delimiter=';', uint codepage=CP_ACP)
// Returns file handle (>= 0) or INVALID_HANDLE (-1) on failure.
int handle = FileOpen("data.csv", FILE_WRITE|FILE_CSV, ',');

// void FileClose(int handle)
FileClose(handle);

// uint FileWrite(int handle, ...)
// Writes data to a CSV or TXT file. Returns number of bytes written.
FileWrite(handle, "Symbol", "Price", "Time");

// string FileReadString(int handle, int length=0)
string line = FileReadString(handle);

// double FileReadDouble(int handle, int size=DOUBLE_VALUE)
double value = FileReadDouble(handle);

// int FileReadInteger(int handle, int size=INT_VALUE)
int num = FileReadInteger(handle);

// bool FileIsEnding(int handle)
while(!FileIsEnding(handle))
{
   string data = FileReadString(handle);
}

// bool FileDelete(string filename, int common_flag=0)
FileDelete("old_data.csv");

// bool FileIsExist(string filename, int common_flag=0)
if(FileIsExist("config.txt"))
{
   // Read config
}
```

### Write CSV File Example

```mql4
void SaveTradeHistory()
{
   int handle = FileOpen("trade_history.csv", FILE_WRITE|FILE_CSV, ',');
   if(handle == INVALID_HANDLE)
   {
      Print("FileOpen failed. Error: ", GetLastError());
      return;
   }

   // Header
   FileWrite(handle, "Ticket", "Symbol", "Type", "Lots",
             "OpenPrice", "ClosePrice", "Profit", "OpenTime", "CloseTime");

   // Data
   for(int i = OrdersHistoryTotal() - 1; i >= 0; i--)
   {
      if(!OrderSelect(i, SELECT_BY_POS, MODE_HISTORY)) continue;
      if(OrderSymbol() != _Symbol) continue;

      string typeStr = (OrderType() == OP_BUY) ? "BUY" : "SELL";

      FileWrite(handle,
                OrderTicket(),
                OrderSymbol(),
                typeStr,
                DoubleToString(OrderLots(), 2),
                DoubleToString(OrderOpenPrice(), Digits),
                DoubleToString(OrderClosePrice(), Digits),
                DoubleToString(OrderProfit(), 2),
                TimeToString(OrderOpenTime()),
                TimeToString(OrderCloseTime()));
   }

   FileClose(handle);
   Print("Trade history saved to MQL4/Files/trade_history.csv");
}
```

### Read CSV File Example

```mql4
void LoadSettings()
{
   if(!FileIsExist("settings.csv"))
   {
      Print("Settings file not found");
      return;
   }

   int handle = FileOpen("settings.csv", FILE_READ|FILE_CSV, ',');
   if(handle == INVALID_HANDLE)
   {
      Print("FileOpen failed: ", GetLastError());
      return;
   }

   while(!FileIsEnding(handle))
   {
      string key   = FileReadString(handle);
      string value = FileReadString(handle);
      Print("Setting: ", key, " = ", value);
   }

   FileClose(handle);
}
```

### Write/Read Binary File Example

```mql4
// Write binary
void SaveBuffer(double &buffer[], int count)
{
   int handle = FileOpen("buffer.bin", FILE_WRITE|FILE_BIN);
   if(handle == INVALID_HANDLE) return;

   FileWriteInteger(handle, count, INT_VALUE);
   for(int i = 0; i < count; i++)
      FileWriteDouble(handle, buffer[i], DOUBLE_VALUE);

   FileClose(handle);
}

// Read binary
void LoadBuffer(double &buffer[])
{
   int handle = FileOpen("buffer.bin", FILE_READ|FILE_BIN);
   if(handle == INVALID_HANDLE) return;

   int count = FileReadInteger(handle, INT_VALUE);
   ArrayResize(buffer, count);
   for(int i = 0; i < count; i++)
      buffer[i] = FileReadDouble(handle, DOUBLE_VALUE);

   FileClose(handle);
}
```

---

## WebRequest

Two function signatures. **Whitelist URL first:** Tools > Options > Expert Advisors. Only in EAs/Scripts (not indicators, not Strategy Tester).

```mql4
// Variant 1: Simple (cookie/referer)
int WebRequest(const string method, const string url,
               const string cookie, const string referer,
               int timeout, const char &data[], int data_size,
               char &result[], string &headers);

// Variant 2: Custom headers (recommended)
int WebRequest(const string method, const string url,
               const string headers, int timeout,
               const char &data[], char &result[], string &result_headers);

// Returns: HTTP status code (200, 404, etc.) or -1 on error.
```

**Key notes:**
- `StringToCharArray()` adds null terminator; subtract 1 from size for binary data
- Error 4060 = URL not whitelisted
- HTTPS supported and recommended

> **Full examples** (GET, POST/JSON, CHttpClient class, Node.js patterns, error handling): See [external-communication.md](external-communication.md)

---

## Utility Functions

### Mathematical Functions

```mql4
double MathAbs(double value);         // Absolute value
double MathCeil(double value);        // Round up
double MathFloor(double value);       // Round down
double MathRound(double value);       // Round to nearest
double MathMax(double a, double b);   // Maximum
double MathMin(double a, double b);   // Minimum
double MathPow(double base, double exp); // Power
double MathSqrt(double value);        // Square root
double MathLog(double value);         // Natural logarithm
double MathExp(double value);         // e^x
double MathMod(double a, double b);   // Modulo for doubles
int    MathRand();                    // Random number 0-32767
void   MathSrand(int seed);           // Seed random generator
double NormalizeDouble(double value, int digits); // Round to digits
```

### Date and Time Functions

```mql4
datetime TimeCurrent();               // Server time (last known)
datetime TimeLocal();                 // Local computer time
datetime TimeGMT();                   // GMT time
int      TimeYear(datetime t);        // Year
int      TimeMonth(datetime t);       // Month (1-12)
int      TimeDay(datetime t);         // Day of month (1-31)
int      TimeHour(datetime t);        // Hour (0-23)
int      TimeMinute(datetime t);      // Minute (0-59)
int      TimeSeconds(datetime t);     // Seconds (0-59)
int      TimeDayOfWeek(datetime t);   // Day of week (0=Sunday)
int      TimeDayOfYear(datetime t);   // Day of year (1-366)
string   TimeToString(datetime t, int mode=TIME_DATE|TIME_MINUTES);
datetime StringToTime(string value);

// Example: Trade only during London session
int hour = TimeHour(TimeCurrent());
if(hour >= 8 && hour < 16) // London hours
{
   // Trading allowed
}
```

### Chart Functions

```mql4
long  ChartID();                           // Current chart ID
bool  ChartSetInteger(long chart_id, int prop, long value);
bool  ChartSetDouble(long chart_id, int prop, double value);
bool  ChartSetString(long chart_id, int prop, string value);
void  ChartRedraw(long chart_id=0);        // Redraw chart

// Example: Add comment to chart
Comment("Balance: ", AccountBalance(), "\n",
        "Equity: ", AccountEquity(), "\n",
        "Spread: ", MarketInfo(_Symbol, MODE_SPREAD));
```

### Object Functions

```mql4
bool ObjectCreate(string name, int type, int window, datetime time1, double price1, ...);
bool ObjectDelete(string name);
bool ObjectSet(string name, int prop, double value);
bool ObjectSetText(string name, string text, int font_size, string font, color text_color);

// Example: Draw horizontal line
ObjectCreate("SupportLine", OBJ_HLINE, 0, 0, 1.23456);
ObjectSet("SupportLine", OBJPROP_COLOR, clrBlue);
ObjectSet("SupportLine", OBJPROP_WIDTH, 2);
ObjectSet("SupportLine", OBJPROP_STYLE, STYLE_DASH);
```

### Print and Alert

```mql4
Print("Message to Experts log");           // Experts tab in Terminal
Alert("Alert popup message");               // Pop-up dialog
Comment("On-chart comment");                // Displayed on chart
SendNotification("Push notification");      // Mobile push (if configured)
SendMail("Subject", "Email body");          // Email (if configured)
PlaySound("alert.wav");                     // Play sound file
```

### Miscellaneous

```mql4
void   Sleep(int milliseconds);            // Pause execution (not in indicators!)
bool   IsTesting();                        // Running in Strategy Tester?
bool   IsOptimization();                   // Running optimization?
bool   IsVisualMode();                     // Visual mode in tester?
bool   IsDemo();                           // Demo account?
bool   IsConnected();                      // Connected to server?
bool   IsTradeAllowed();                   // Trade context free?
bool   IsExpertEnabled();                  // Expert Advisors enabled?
bool   IsStopped();                        // EA being terminated?
bool   IsDllsAllowed();                    // DLL imports allowed?
string TerminalInfoString(int prop);       // Terminal info
string Symbol();                           // Current symbol (same as _Symbol)
int    Period();                           // Current period (same as _Period)
int    UninitializeReason();               // Deinitialization reason
void   ExpertRemove();                     // Remove EA from chart
```

---

## Global Terminal Variables

Not to be confused with global program variables. These are key-value pairs stored at the terminal level, shared between programs.

```mql4
// bool GlobalVariableSet(string name, double value)
// Creates or updates a global terminal variable.
GlobalVariableSet("LastTradeTime_EURUSD", (double)TimeCurrent());

// double GlobalVariableGet(string name)
// Returns the value. Returns 0 if not found (check with GlobalVariableCheck).
double lastTime = GlobalVariableGet("LastTradeTime_EURUSD");

// bool GlobalVariableCheck(string name)
// Returns true if the variable exists.
if(GlobalVariableCheck("LastTradeTime_EURUSD"))
{
   datetime t = (datetime)GlobalVariableGet("LastTradeTime_EURUSD");
}

// bool GlobalVariableDel(string name)
GlobalVariableDel("LastTradeTime_EURUSD");

// bool GlobalVariableSetOnCondition(string name, double new_value, double check_value)
// Atomic set: only sets if the current value matches check_value. Useful for locking.
// Returns true if the value was set.
if(GlobalVariableSetOnCondition("TradeLock", 1.0, 0.0))
{
   // We acquired the lock
   // ... trade ...
   GlobalVariableSet("TradeLock", 0.0); // Release lock
}
```
