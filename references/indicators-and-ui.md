# Indicators, UI Panels & Scripts

## Table of Contents

- [Custom Indicators - MQL4](#custom-indicators---mql4)
- [Custom Indicators - MQL5](#custom-indicators---mql5)
- [Indicator Handle System (MQL5)](#indicator-handle-system-mql5)
- [Graphical Objects](#graphical-objects)
- [UI Panels](#ui-panels)
- [Scripts](#scripts)
- [Chart Operations](#chart-operations)

---

## Custom Indicators - MQL4

### Properties

```mql4
#property strict
#property indicator_chart_window       // or indicator_separate_window
#property indicator_buffers 2          // visual buffers (max 8)
#property indicator_color1  clrRed
#property indicator_color2  clrBlue
#property indicator_width1  2
#property indicator_style1  STYLE_SOLID
#property indicator_minimum 0          // for separate window
#property indicator_maximum 100
#property indicator_level1  30         // horizontal levels
#property indicator_level2  70
```

### Buffer Setup

- `IndicatorBuffers(n)` sets **total** buffer count (visual + calculation)
- `#property indicator_buffers` sets **visual** buffer count
- If you need extra calculation buffers: `IndicatorBuffers(visual + extra)`
- Max 8 buffers in MQL4

```mql4
double Buffer1[], Buffer2[], CalcBuffer[];

int OnInit()
{
   IndicatorBuffers(3);  // 2 visual + 1 calculation
   SetIndexBuffer(0, Buffer1);
   SetIndexBuffer(1, Buffer2);
   SetIndexBuffer(2, CalcBuffer);

   SetIndexStyle(0, DRAW_LINE, STYLE_SOLID, 2, clrRed);
   SetIndexStyle(1, DRAW_HISTOGRAM, STYLE_SOLID, 1, clrBlue);

   SetIndexLabel(0, "Main Line");
   SetIndexLabel(1, "Histogram");

   SetIndexDrawBegin(0, 14);  // skip first 14 bars
   SetIndexEmptyValue(0, EMPTY_VALUE);

   IndicatorShortName("My Indicator");
   return(INIT_SUCCEEDED);
}
```

**Draw types (MQL4):** `DRAW_LINE`, `DRAW_HISTOGRAM`, `DRAW_ARROW`, `DRAW_NONE`, `DRAW_SECTION`, `DRAW_ZIGZAG`

### OnCalculate Template (MQL4)

```mql4
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
   // Efficient recalculation: only process new bars
   int limit = rates_total - prev_calculated;
   if(prev_calculated > 0) limit++;  // recheck last bar

   // MQL4 default: index 0 = oldest bar (non-series)
   // If you want series order (0 = newest):
   // ArraySetAsSeries(close, true);
   // ArraySetAsSeries(Buffer1, true);

   for(int i = limit - 1; i >= 0; i--)
   {
      // Calculate from oldest to newest
      int pos = rates_total - 1 - i;  // convert to non-series index
      Buffer1[pos] = close[pos];      // your calculation here
   }

   return(rates_total);
}
```

### Multi-Timeframe Indicator (MQL4)

```mql4
// Read indicator values from other timeframes
double htfMA = iMA(Symbol(), PERIOD_H4, 20, 0, MODE_SMA, PRICE_CLOSE, 0);

// Read custom indicator from other timeframe
double val = iCustom(Symbol(), PERIOD_H1, "MyIndicator", param1, param2, bufferIndex, shift);

// Shift bar alignment: find H4 bar that corresponds to current M15 bar
int htfShift = iBarShift(Symbol(), PERIOD_H4, Time[i]);
```

**Gotcha:** MTF indicators may repaint if higher timeframe bar is still forming.

---

## Custom Indicators - MQL5

### Properties and Buffers

```mql5
#property indicator_chart_window
#property indicator_buffers 3          // total buffers (up to 512)
#property indicator_plots   2          // visual plots

// Plot 1 configuration
#property indicator_label1  "Main"
#property indicator_type1   DRAW_LINE
#property indicator_color1  clrBlue
#property indicator_width1  2
#property indicator_style1  STYLE_SOLID

// Plot 2 configuration
#property indicator_label2  "Signal"
#property indicator_type2   DRAW_LINE
#property indicator_color2  clrRed
#property indicator_width2  1
```

### Buffers vs Plots

| Concept | What it is | Count rule |
|---------|-----------|------------|
| `indicator_buffers` | Total arrays (data + color + calc) | Must be >= plots |
| `indicator_plots` | Number of visual drawings | What user sees |
| Buffer types | `INDICATOR_DATA`, `INDICATOR_COLOR_INDEX`, `INDICATOR_CALCULATIONS` | Set in OnInit |

```mql5
double MainBuffer[], SignalBuffer[], CalcBuffer[];

int OnInit()
{
   // Map buffers to indices
   SetIndexBuffer(0, MainBuffer,   INDICATOR_DATA);
   SetIndexBuffer(1, SignalBuffer,  INDICATOR_DATA);
   SetIndexBuffer(2, CalcBuffer,    INDICATOR_CALCULATIONS);

   // Configure plots programmatically (alternative to #property)
   PlotIndexSetInteger(0, PLOT_DRAW_TYPE, DRAW_LINE);
   PlotIndexSetInteger(0, PLOT_LINE_COLOR, clrBlue);
   PlotIndexSetInteger(0, PLOT_LINE_WIDTH, 2);
   PlotIndexSetInteger(0, PLOT_DRAW_BEGIN, 14);
   PlotIndexSetDouble(0, PLOT_EMPTY_VALUE, EMPTY_VALUE);
   PlotIndexSetString(0, PLOT_LABEL, "Main Line");

   // Arrow indicator
   // PlotIndexSetInteger(0, PLOT_ARROW, 233); // wingdings code

   IndicatorSetString(INDICATOR_SHORTNAME, "My Indicator");
   IndicatorSetInteger(INDICATOR_DIGITS, _Digits);

   return(INIT_SUCCEEDED);
}
```

### OnCalculate - Two Variants

**Full form** (receives all OHLCV data):
```mql5
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
   // Default: arrays are NOT series (index 0 = oldest)
   // To use series order:
   // ArraySetAsSeries(close, true);
   // ArraySetAsSeries(MainBuffer, true);

   int limit = (prev_calculated == 0) ? 0 : prev_calculated - 1;

   for(int i = limit; i < rates_total; i++)
   {
      MainBuffer[i] = close[i];  // your calculation
   }

   return(rates_total);
}
```

**Short form** (single data source - "Apply to" dropdown):
```mql5
int OnCalculate(const int rates_total,
                const int prev_calculated,
                const int begin,
                const double &price[])
{
   int limit = (prev_calculated == 0) ? begin : prev_calculated - 1;

   for(int i = limit; i < rates_total; i++)
   {
      MainBuffer[i] = price[i];
   }

   return(rates_total);
}
```

**Important:** You can only have ONE variant in your indicator. The short form is used when the indicator needs to be applied to different data sources (close, open, another indicator's output, etc.).

### Draw Styles Reference

| Style | Data Buffers | Color Buffer | Total | Description |
|-------|-------------|-------------|-------|-------------|
| `DRAW_NONE` | 1 | - | 1 | Hidden buffer |
| `DRAW_LINE` | 1 | - | 1 | Simple line |
| `DRAW_SECTION` | 1 | - | 1 | Line segments |
| `DRAW_HISTOGRAM` | 1 | - | 1 | Histogram from zero |
| `DRAW_HISTOGRAM2` | 2 | - | 2 | Histogram between two values |
| `DRAW_ARROW` | 1 | - | 1 | Arrow symbols |
| `DRAW_ZIGZAG` | 2 | - | 2 | Zigzag line |
| `DRAW_FILLING` | 2 | - | 2 | Filled area between lines |
| `DRAW_BARS` | 4 | - | 4 | OHLC bars |
| `DRAW_CANDLES` | 4 | - | 4 | Candlesticks |
| `DRAW_COLOR_LINE` | 1 | 1 | 2 | Multi-color line |
| `DRAW_COLOR_SECTION` | 1 | 1 | 2 | Multi-color sections |
| `DRAW_COLOR_HISTOGRAM` | 1 | 1 | 2 | Multi-color histogram |
| `DRAW_COLOR_HISTOGRAM2` | 2 | 1 | 3 | Multi-color histogram2 |
| `DRAW_COLOR_ARROW` | 1 | 1 | 2 | Multi-color arrows |
| `DRAW_COLOR_ZIGZAG` | 2 | 1 | 3 | Multi-color zigzag |
| `DRAW_COLOR_BARS` | 4 | 1 | 5 | Multi-color bars |
| `DRAW_COLOR_CANDLES` | 4 | 1 | 5 | Multi-color candles |

### Color Line Example (MQL5)

```mql5
#property indicator_separate_window
#property indicator_buffers 2
#property indicator_plots   1
#property indicator_type1   DRAW_COLOR_LINE
#property indicator_color1  clrGreen,clrYellow,clrRed  // color palette
#property indicator_width1  2

double ValueBuffer[];
double ColorBuffer[];

int OnInit()
{
   SetIndexBuffer(0, ValueBuffer, INDICATOR_DATA);
   SetIndexBuffer(1, ColorBuffer, INDICATOR_COLOR_INDEX);
   PlotIndexSetInteger(0, PLOT_COLOR_INDEXES, 3);  // 3 colors
   return(INIT_SUCCEEDED);
}

int OnCalculate(const int rates_total, const int prev_calculated,
                const datetime &time[], const double &open[],
                const double &high[], const double &low[],
                const double &close[], const long &tick_volume[],
                const long &volume[], const int &spread[])
{
   int limit = (prev_calculated == 0) ? 14 : prev_calculated - 1;

   for(int i = limit; i < rates_total; i++)
   {
      // Example: RSI-like value
      ValueBuffer[i] = /* your calculation */;

      // Set color index: 0=Green, 1=Yellow, 2=Red
      if(ValueBuffer[i] > 70)      ColorBuffer[i] = 2;  // Red (overbought)
      else if(ValueBuffer[i] < 30)  ColorBuffer[i] = 0;  // Green (oversold)
      else                           ColorBuffer[i] = 1;  // Yellow (neutral)
   }
   return(rates_total);
}
```

### PlotIndex Functions Reference

```mql5
// Integer properties
PlotIndexSetInteger(plotIndex, PLOT_DRAW_TYPE, DRAW_LINE);
PlotIndexSetInteger(plotIndex, PLOT_LINE_STYLE, STYLE_SOLID);
PlotIndexSetInteger(plotIndex, PLOT_LINE_WIDTH, 2);
PlotIndexSetInteger(plotIndex, PLOT_LINE_COLOR, clrBlue);
PlotIndexSetInteger(plotIndex, PLOT_COLOR_INDEXES, 3);
PlotIndexSetInteger(plotIndex, PLOT_ARROW, 233);       // wingdings code
PlotIndexSetInteger(plotIndex, PLOT_SHIFT, 0);
PlotIndexSetInteger(plotIndex, PLOT_DRAW_BEGIN, 14);
PlotIndexSetInteger(plotIndex, PLOT_SHOW_DATA, true);

// Double properties
PlotIndexSetDouble(plotIndex, PLOT_EMPTY_VALUE, EMPTY_VALUE);

// String properties
PlotIndexSetString(plotIndex, PLOT_LABEL, "My Line");

// Indicator-level settings
IndicatorSetString(INDICATOR_SHORTNAME, "Name (params)");
IndicatorSetInteger(INDICATOR_DIGITS, _Digits);
IndicatorSetInteger(INDICATOR_LEVELS, 2);
IndicatorSetDouble(INDICATOR_LEVELVALUE, 0, 30.0);
IndicatorSetDouble(INDICATOR_LEVELVALUE, 1, 70.0);
IndicatorSetInteger(INDICATOR_LEVELCOLOR, 0, clrSilver);
IndicatorSetInteger(INDICATOR_LEVELSTYLE, 0, STYLE_DOT);
IndicatorSetDouble(INDICATOR_MINIMUM, 0);
IndicatorSetDouble(INDICATOR_MAXIMUM, 100);
```

---

## Indicator Handle System (MQL5)

In MQL5, built-in indicators return **handles** (int). Use `CopyBuffer()` to retrieve data.

### Creating Handles

```mql5
// Create in OnInit - only once
int g_maHandle, g_rsiHandle, g_bbHandle;

int OnInit()
{
   g_maHandle  = iMA(_Symbol, PERIOD_CURRENT, 20, 0, MODE_SMA, PRICE_CLOSE);
   g_rsiHandle = iRSI(_Symbol, PERIOD_CURRENT, 14, PRICE_CLOSE);
   g_bbHandle  = iBands(_Symbol, PERIOD_CURRENT, 20, 0, 2.0, PRICE_CLOSE);

   if(g_maHandle == INVALID_HANDLE || g_rsiHandle == INVALID_HANDLE)
   {
      Print("Failed to create indicator handles");
      return(INIT_FAILED);
   }
   return(INIT_SUCCEEDED);
}

void OnDeinit(const int reason)
{
   IndicatorRelease(g_maHandle);
   IndicatorRelease(g_rsiHandle);
   IndicatorRelease(g_bbHandle);
}
```

### Reading Data from Handles

```mql5
double maBuffer[], rsiBuffer[];
double bbUpper[], bbMiddle[], bbLower[];

void OnTick()
{
   ArraySetAsSeries(maBuffer, true);
   ArraySetAsSeries(rsiBuffer, true);
   ArraySetAsSeries(bbUpper, true);
   ArraySetAsSeries(bbMiddle, true);
   ArraySetAsSeries(bbLower, true);

   // CopyBuffer(handle, bufferIndex, startPos, count, array)
   if(CopyBuffer(g_maHandle,  0, 0, 3, maBuffer)  < 3) return;
   if(CopyBuffer(g_rsiHandle, 0, 0, 3, rsiBuffer) < 3) return;

   // Bollinger has 3 buffers: 0=middle, 1=upper, 2=lower
   if(CopyBuffer(g_bbHandle, 0, 0, 3, bbMiddle) < 3) return;
   if(CopyBuffer(g_bbHandle, 1, 0, 3, bbUpper)  < 3) return;
   if(CopyBuffer(g_bbHandle, 2, 0, 3, bbLower)  < 3) return;

   // maBuffer[0] = current bar, maBuffer[1] = previous bar
   double currentMA  = maBuffer[0];
   double currentRSI = rsiBuffer[0];
}
```

### Custom Indicator Handle

```mql5
// iCustom(symbol, timeframe, "IndicatorName", param1, param2, ...)
int customHandle = iCustom(_Symbol, PERIOD_H1, "MyIndicator", 14, 2.0);

double buf[];
ArraySetAsSeries(buf, true);
CopyBuffer(customHandle, 0, 0, 10, buf);  // buffer 0, last 10 values
```

### Common Built-in Indicator Handles

| Function | Buffers | Notes |
|----------|---------|-------|
| `iMA()` | 0: MA value | MODE_SMA, MODE_EMA, MODE_SMMA, MODE_LWMA |
| `iRSI()` | 0: RSI value | |
| `iMACD()` | 0: main, 1: signal | |
| `iBands()` | 0: middle, 1: upper, 2: lower | |
| `iATR()` | 0: ATR value | |
| `iStochastic()` | 0: main, 1: signal | |
| `iADX()` | 0: ADX, 1: +DI, 2: -DI | |
| `iCCI()` | 0: CCI value | |
| `iSAR()` | 0: SAR value | |
| `iIchimoku()` | 0: tenkan, 1: kijun, 2: senkouA, 3: senkouB, 4: chikou | |

---

## Graphical Objects

### Common Object Types

| Type | Description | Anchor Points |
|------|-------------|---------------|
| `OBJ_HLINE` | Horizontal line | 1 (price only) |
| `OBJ_VLINE` | Vertical line | 1 (time only) |
| `OBJ_TREND` | Trend line | 2 |
| `OBJ_RECTANGLE` | Rectangle | 2 (corners) |
| `OBJ_TRIANGLE` | Triangle | 3 |
| `OBJ_ELLIPSE` | Ellipse | 2 |
| `OBJ_LABEL` | Text label (pixel coords) | 0 |
| `OBJ_TEXT` | Text on chart (price/time) | 1 |
| `OBJ_EDIT` | Editable text field | 0 |
| `OBJ_BUTTON` | Clickable button | 0 |
| `OBJ_BITMAP_LABEL` | Image (pixel coords) | 0 |
| `OBJ_RECTANGLE_LABEL` | Rectangle (pixel coords) | 0 |
| `OBJ_ARROW` | Arrow symbol | 1 |

### Creating Objects

```mql5
// Price-anchored objects (use time/price coordinates)
ObjectCreate(0, "myTrend", OBJ_TREND, 0, time1, price1, time2, price2);
ObjectCreate(0, "myHLine", OBJ_HLINE, 0, 0, priceLevel);

// Pixel-anchored objects (use XDISTANCE/YDISTANCE)
ObjectCreate(0, "myLabel", OBJ_LABEL, 0, 0, 0);
ObjectSetInteger(0, "myLabel", OBJPROP_XDISTANCE, 50);
ObjectSetInteger(0, "myLabel", OBJPROP_YDISTANCE, 50);
ObjectSetInteger(0, "myLabel", OBJPROP_CORNER, CORNER_LEFT_UPPER);
ObjectSetString(0, "myLabel", OBJPROP_TEXT, "Hello World");
ObjectSetString(0, "myLabel", OBJPROP_FONT, "Arial");
ObjectSetInteger(0, "myLabel", OBJPROP_FONTSIZE, 12);
ObjectSetInteger(0, "myLabel", OBJPROP_COLOR, clrWhite);
```

### Common Object Properties

```mql5
// Set properties
ObjectSetInteger(chartId, name, OBJPROP_COLOR, clrRed);
ObjectSetInteger(chartId, name, OBJPROP_WIDTH, 2);
ObjectSetInteger(chartId, name, OBJPROP_STYLE, STYLE_DASH);
ObjectSetInteger(chartId, name, OBJPROP_BACK, true);          // draw behind candles
ObjectSetInteger(chartId, name, OBJPROP_SELECTABLE, false);   // prevent user selection
ObjectSetInteger(chartId, name, OBJPROP_HIDDEN, true);        // hide from object list
ObjectSetInteger(chartId, name, OBJPROP_ZORDER, 10);          // z-order for click priority

// Get properties
long   colorVal = ObjectGetInteger(0, name, OBJPROP_COLOR);
double priceVal = ObjectGetDouble(0, name, OBJPROP_PRICE);
string textVal  = ObjectGetString(0, name, OBJPROP_TEXT);

// Delete objects
ObjectDelete(0, "myLabel");
ObjectsDeleteAll(0, "prefix_");   // delete by name prefix
ObjectsDeleteAll(0, 0, OBJ_LABEL);  // delete by type in subwindow 0
```

### Button Example

```mql5
void CreateButton(string name, int x, int y, int width, int height, string text)
{
   ObjectCreate(0, name, OBJ_BUTTON, 0, 0, 0);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, y);
   ObjectSetInteger(0, name, OBJPROP_XSIZE, width);
   ObjectSetInteger(0, name, OBJPROP_YSIZE, height);
   ObjectSetString(0, name, OBJPROP_TEXT, text);
   ObjectSetInteger(0, name, OBJPROP_COLOR, clrWhite);
   ObjectSetInteger(0, name, OBJPROP_BGCOLOR, clrDodgerBlue);
   ObjectSetInteger(0, name, OBJPROP_BORDER_COLOR, clrNONE);
   ObjectSetInteger(0, name, OBJPROP_FONTSIZE, 10);
   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
}

// In OnInit:
CreateButton("btnBuy", 10, 30, 100, 30, "BUY");
CreateButton("btnSell", 120, 30, 100, 30, "SELL");
```

### OnChartEvent Handler

```mql5
void OnChartEvent(const int id,
                  const long &lparam,
                  const double &dparam,
                  const string &sparam)
{
   // Button click
   if(id == CHARTEVENT_OBJECT_CLICK)
   {
      if(sparam == "btnBuy")
      {
         // Execute buy logic
         ObjectSetInteger(0, "btnBuy", OBJPROP_STATE, false);  // reset button
      }
      else if(sparam == "btnSell")
      {
         // Execute sell logic
         ObjectSetInteger(0, "btnSell", OBJPROP_STATE, false);
      }
   }

   // Mouse click on chart (not on object)
   if(id == CHARTEVENT_CLICK)
   {
      int x = (int)lparam;    // pixel X
      int y = (int)dparam;    // pixel Y
      // Convert to price/time:
      datetime time;
      double price;
      int subwindow;
      ChartXYToTimePrice(0, x, y, subwindow, time, price);
   }

   // Keyboard
   if(id == CHARTEVENT_KEYDOWN)
   {
      long keyCode = lparam;  // virtual key code
   }

   // Mouse move (requires ChartSetInteger(0, CHART_EVENT_MOUSE_MOVE, true))
   if(id == CHARTEVENT_MOUSE_MOVE)
   {
      int mouseX = (int)lparam;
      int mouseY = (int)dparam;
      // sparam contains mouse button flags
   }

   // Edit field changed
   if(id == CHARTEVENT_OBJECT_ENDEDIT)
   {
      string editValue = ObjectGetString(0, sparam, OBJPROP_TEXT);
   }

   // Custom events (from EventChartCustom)
   if(id >= CHARTEVENT_CUSTOM)
   {
      int customId = id - CHARTEVENT_CUSTOM;
   }
}
```

**Event Types:**

| Event | id | lparam | dparam | sparam |
|-------|----|--------|--------|--------|
| `CHARTEVENT_KEYDOWN` | 0 | key code | repeat count | key flags |
| `CHARTEVENT_MOUSE_MOVE` | 1 | x | y | flags |
| `CHARTEVENT_OBJECT_CREATE` | 2 | - | - | object name |
| `CHARTEVENT_OBJECT_CHANGE` | 3 | - | - | object name |
| `CHARTEVENT_OBJECT_DELETE` | 4 | - | - | object name |
| `CHARTEVENT_CLICK` | 5 | x | y | - |
| `CHARTEVENT_OBJECT_CLICK` | 6 | x | y | object name |
| `CHARTEVENT_OBJECT_DRAG` | 7 | - | - | object name |
| `CHARTEVENT_OBJECT_ENDEDIT` | 8 | - | - | object name |
| `CHARTEVENT_CHART_CHANGE` | 9 | - | - | - |
| `CHARTEVENT_CUSTOM+n` | 1000+n | lparam | dparam | sparam |

### Sending Custom Events

```mql5
// Send to own chart
EventChartCustom(0, 0, longParam, doubleParam, "stringParam");
// id in OnChartEvent will be CHARTEVENT_CUSTOM + 0 = 1000

// Send to another chart
long otherChartId = ChartFirst();
EventChartCustom(otherChartId, 1, 0, 0.0, "message");
```

---

## UI Panels

### Simple Panel with Objects (Lightweight)

```mql5
string g_prefix = "panel_";

void CreatePanel()
{
   int panelX = 10, panelY = 30;
   int panelW = 220, panelH = 180;

   // Background
   ObjectCreate(0, g_prefix + "bg", OBJ_RECTANGLE_LABEL, 0, 0, 0);
   ObjectSetInteger(0, g_prefix + "bg", OBJPROP_XDISTANCE, panelX);
   ObjectSetInteger(0, g_prefix + "bg", OBJPROP_YDISTANCE, panelY);
   ObjectSetInteger(0, g_prefix + "bg", OBJPROP_XSIZE, panelW);
   ObjectSetInteger(0, g_prefix + "bg", OBJPROP_YSIZE, panelH);
   ObjectSetInteger(0, g_prefix + "bg", OBJPROP_BGCOLOR, C'32,32,32');
   ObjectSetInteger(0, g_prefix + "bg", OBJPROP_BORDER_TYPE, BORDER_FLAT);
   ObjectSetInteger(0, g_prefix + "bg", OBJPROP_BORDER_COLOR, clrDimGray);
   ObjectSetInteger(0, g_prefix + "bg", OBJPROP_CORNER, CORNER_LEFT_UPPER);

   // Title label
   CreateLabel(g_prefix + "title", panelX + 10, panelY + 5, "Trade Panel", 11, clrWhite);

   // Data labels
   CreateLabel(g_prefix + "spread", panelX + 10, panelY + 30, "Spread: --", 9, clrSilver);
   CreateLabel(g_prefix + "profit", panelX + 10, panelY + 50, "P/L: --", 9, clrSilver);

   // Buttons
   CreateButton(g_prefix + "buy",  panelX + 10,  panelY + 80, 95, 30, "BUY");
   CreateButton(g_prefix + "sell", panelX + 115, panelY + 80, 95, 30, "SELL");

   ChartRedraw();
}

void CreateLabel(string name, int x, int y, string text, int fontSize, color clr)
{
   ObjectCreate(0, name, OBJ_LABEL, 0, 0, 0);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, y);
   ObjectSetString(0, name, OBJPROP_TEXT, text);
   ObjectSetString(0, name, OBJPROP_FONT, "Consolas");
   ObjectSetInteger(0, name, OBJPROP_FONTSIZE, fontSize);
   ObjectSetInteger(0, name, OBJPROP_COLOR, clr);
   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
}

void UpdatePanel()
{
   double spread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD) * _Point;
   ObjectSetString(0, g_prefix + "spread", OBJPROP_TEXT,
      StringFormat("Spread: %.1f pips", spread / _Point / 10));

   double profit = AccountInfoDouble(ACCOUNT_PROFIT);
   color profitClr = profit >= 0 ? clrLime : clrRed;
   ObjectSetString(0, g_prefix + "profit", OBJPROP_TEXT,
      StringFormat("P/L: %.2f", profit));
   ObjectSetInteger(0, g_prefix + "profit", OBJPROP_COLOR, profitClr);

   ChartRedraw();
}

void DestroyPanel()
{
   ObjectsDeleteAll(0, g_prefix);
}
```

### CAppDialog Panel (MQL5 Standard Library)

```mql5
#include <Controls/Dialog.mqh>
#include <Controls/Button.mqh>
#include <Controls/Label.mqh>
#include <Controls/Edit.mqh>
#include <Controls/ComboBox.mqh>
#include <Controls/SpinEdit.mqh>
#include <Controls/CheckBox.mqh>

class CTradePanel : public CAppDialog
{
private:
   CButton  m_btnBuy;
   CButton  m_btnSell;
   CButton  m_btnClose;
   CLabel   m_lblSpread;
   CEdit    m_editLots;
   CSpinEdit m_spinSL;
   CComboBox m_cmbSymbol;

public:
   bool Create(const long chart, const string name, const int subwin,
               const int x1, const int y1, const int x2, const int y2);

   void UpdateInfo();

   // Event handlers
   void OnClickBuy();
   void OnClickSell();
   void OnClickClose();

   // Event map
   virtual bool OnEvent(const int id, const long &lparam,
                        const double &dparam, const string &sparam);
};

// Event map macro
EVENT_MAP_BEGIN(CTradePanel)
   ON_EVENT(ON_CLICK, m_btnBuy,   OnClickBuy)
   ON_EVENT(ON_CLICK, m_btnSell,  OnClickSell)
   ON_EVENT(ON_CLICK, m_btnClose, OnClickClose)
EVENT_MAP_END(CAppDialog)

bool CTradePanel::Create(const long chart, const string name, const int subwin,
                          const int x1, const int y1, const int x2, const int y2)
{
   if(!CAppDialog::Create(chart, name, subwin, x1, y1, x2, y2))
      return false;

   // Create controls (coordinates relative to dialog client area)
   if(!m_lblSpread.Create(m_chart_id, m_name + "Spread", m_subwin, 10, 10, 200, 25))
      return false;
   m_lblSpread.Text("Spread: --");
   if(!Add(m_lblSpread)) return false;

   if(!m_editLots.Create(m_chart_id, m_name + "Lots", m_subwin, 10, 35, 100, 55))
      return false;
   m_editLots.Text("0.1");
   if(!Add(m_editLots)) return false;

   if(!m_btnBuy.Create(m_chart_id, m_name + "Buy", m_subwin, 10, 65, 95, 95))
      return false;
   m_btnBuy.Text("BUY");
   m_btnBuy.ColorBackground(clrForestGreen);
   if(!Add(m_btnBuy)) return false;

   if(!m_btnSell.Create(m_chart_id, m_name + "Sell", m_subwin, 105, 65, 190, 95))
      return false;
   m_btnSell.Text("SELL");
   m_btnSell.ColorBackground(clrCrimson);
   if(!Add(m_btnSell)) return false;

   if(!m_btnClose.Create(m_chart_id, m_name + "Close", m_subwin, 10, 105, 190, 130))
      return false;
   m_btnClose.Text("CLOSE ALL");
   if(!Add(m_btnClose)) return false;

   return true;
}

void CTradePanel::OnClickBuy()
{
   double lots = StringToDouble(m_editLots.Text());
   // Execute buy...
}

void CTradePanel::OnClickSell()
{
   double lots = StringToDouble(m_editLots.Text());
   // Execute sell...
}

void CTradePanel::OnClickClose()
{
   // Close all positions...
}

// In EA:
CTradePanel g_panel;

int OnInit()
{
   if(!g_panel.Create(0, "TradePanel", 0, 20, 20, 240, 190))
      return INIT_FAILED;
   g_panel.Run();
   return INIT_SUCCEEDED;
}

void OnDeinit(const int reason)
{
   g_panel.Destroy(reason);
}

void OnChartEvent(const int id, const long &lparam,
                  const double &dparam, const string &sparam)
{
   g_panel.ChartEvent(id, lparam, dparam, sparam);
}

void OnTick()
{
   g_panel.UpdateInfo();
}
```

### CCanvas (MQL5 Pixel Drawing)

```mql5
#include <Canvas/Canvas.mqh>

CCanvas g_canvas;

void CreateCanvas()
{
   g_canvas.CreateBitmapLabel("myPanel", 10, 10, 400, 300,
                               COLOR_FORMAT_ARGB_NORMALIZE);

   // Clear background
   g_canvas.Erase(ColorToARGB(C'30,30,30'));

   // Draw shapes
   g_canvas.FillRectangle(0, 0, 400, 30, ColorToARGB(clrDarkBlue));
   g_canvas.Rectangle(0, 0, 399, 299, ColorToARGB(clrGray));
   g_canvas.Line(0, 30, 400, 30, ColorToARGB(clrGray));
   g_canvas.FillCircle(200, 150, 50, ColorToARGB(clrDodgerBlue, 128));

   // Draw text
   g_canvas.FontSet("Arial", 14, FW_BOLD);
   g_canvas.TextOut(10, 5, "Dashboard", ColorToARGB(clrWhite));

   g_canvas.FontSet("Consolas", 11);
   g_canvas.TextOut(10, 40, "Balance: $10,000", ColorToARGB(clrLime));

   // Apply changes
   g_canvas.Update();
}

void DestroyCanvas()
{
   g_canvas.Destroy();
}
```

**CCanvas key methods:**
- `Erase(argb)` - Clear entire canvas
- `Pixel(x, y, argb)` - Single pixel
- `Line(x1, y1, x2, y2, argb)` - Line
- `Rectangle(x1, y1, x2, y2, argb)` - Rectangle outline
- `FillRectangle(x1, y1, x2, y2, argb)` - Filled rectangle
- `Circle(x, y, r, argb)` / `FillCircle()` - Circle
- `Triangle(x1,y1, x2,y2, x3,y3, argb)` / `FillTriangle()`
- `FontSet(name, size, flags)` - Set font
- `TextOut(x, y, text, argb)` - Draw text
- `Update()` - Apply all changes to screen

---

## Scripts

### MQL4 Script Template

```mql4
#property strict
#property show_inputs        // show input dialog before execution

input double Lots      = 0.1;
input int    MagicNum  = 12345;

void OnStart()
{
   // One-time execution
   Print("Script started");

   // Scripts can trade, modify objects, read files, etc.
   // Scripts run once and terminate
}
```

### MQL5 Script Template

```mql5
#property script_show_inputs

input double InpLots = 0.1;

void OnStart()
{
   // One-time execution
   Print("Script started");
}
```

### Close All Positions Script (MQL5)

```mql5
#property script_show_inputs

#include <Trade/Trade.mqh>

input string InpSymbol   = "";     // Symbol filter (empty = all)
input long   InpMagic    = 0;      // Magic filter (0 = all)
input bool   InpConfirm  = true;   // Ask confirmation

void OnStart()
{
   int total = PositionsTotal();
   if(total == 0)
   {
      Print("No open positions");
      return;
   }

   if(InpConfirm)
   {
      int answer = MessageBox(
         StringFormat("Close %d position(s)?", total),
         "Confirm", MB_YESNO | MB_ICONQUESTION);
      if(answer != IDYES) return;
   }

   CTrade trade;
   int closed = 0, errors = 0;

   // Reverse loop: closing changes indices
   for(int i = total - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;

      // Apply filters
      if(InpSymbol != "" && PositionGetString(POSITION_SYMBOL) != InpSymbol)
         continue;
      if(InpMagic != 0 && PositionGetInteger(POSITION_MAGIC) != InpMagic)
         continue;

      if(trade.PositionClose(ticket))
         closed++;
      else
      {
         errors++;
         Print("Failed to close #", ticket, ": ", trade.ResultRetcodeDescription());
      }
   }

   Print(StringFormat("Closed: %d, Errors: %d", closed, errors));
}
```

### Close All Orders Script (MQL4)

```mql4
#property strict
#property show_inputs

input int MagicFilter = 0;  // 0 = all

void OnStart()
{
   int closed = 0;

   // Close market orders (reverse loop!)
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!OrderSelect(i, SELECT_BY_POS, MODE_TRADES)) continue;
      if(MagicFilter != 0 && OrderMagicNumber() != MagicFilter) continue;

      bool result = false;
      if(OrderType() == OP_BUY)
         result = OrderClose(OrderTicket(), OrderLots(), MarketInfo(OrderSymbol(), MODE_BID), 3);
      else if(OrderType() == OP_SELL)
         result = OrderClose(OrderTicket(), OrderLots(), MarketInfo(OrderSymbol(), MODE_ASK), 3);
      else // pending order
         result = OrderDelete(OrderTicket());

      if(result) closed++;
      else Print("Failed: ", GetLastError());
   }
   Print("Closed: ", closed);
}
```

### Common Script Use Cases

- **Close all positions** by symbol/magic/direction
- **Export trade history** to CSV file
- **Place grid orders** (multiple pending orders at intervals)
- **Batch modify SL/TP** on all open positions
- **Account statistics** calculator and display
- **Delete all objects** by type or prefix
- **Symbol scanner** (check conditions across multiple symbols)
- **Lot calculator** (calculate position size from risk parameters)

---

## Chart Operations

### Chart Properties

```mql5
// Set properties
ChartSetInteger(0, CHART_MODE, CHART_CANDLES);         // CHART_BARS, CHART_LINE
ChartSetInteger(0, CHART_AUTOSCROLL, true);
ChartSetInteger(0, CHART_SHIFT, true);                  // right margin
ChartSetInteger(0, CHART_SHOW_GRID, false);
ChartSetInteger(0, CHART_SHOW_VOLUMES, CHART_VOLUME_HIDE);
ChartSetInteger(0, CHART_SHOW_TRADE_LEVELS, true);
ChartSetInteger(0, CHART_SHOW_DATE_SCALE, true);
ChartSetInteger(0, CHART_SHOW_PRICE_SCALE, true);
ChartSetInteger(0, CHART_EVENT_MOUSE_MOVE, true);       // enable mouse events
ChartSetInteger(0, CHART_CROSSHAIR_TOOL, true);

// Colors
ChartSetInteger(0, CHART_COLOR_BACKGROUND, clrBlack);
ChartSetInteger(0, CHART_COLOR_FOREGROUND, clrWhite);
ChartSetInteger(0, CHART_COLOR_GRID, clrDimGray);
ChartSetInteger(0, CHART_COLOR_CANDLE_BULL, clrLime);
ChartSetInteger(0, CHART_COLOR_CANDLE_BEAR, clrRed);
ChartSetInteger(0, CHART_COLOR_CHART_UP, clrLime);
ChartSetInteger(0, CHART_COLOR_CHART_DOWN, clrRed);
ChartSetInteger(0, CHART_COLOR_CHART_LINE, clrWhite);

// Get properties
long chartMode = ChartGetInteger(0, CHART_MODE);
int  chartW    = (int)ChartGetInteger(0, CHART_WIDTH_IN_PIXELS);
int  chartH    = (int)ChartGetInteger(0, CHART_HEIGHT_IN_PIXELS);
int  firstBar  = (int)ChartGetInteger(0, CHART_FIRST_VISIBLE_BAR);
int  visBars   = (int)ChartGetInteger(0, CHART_VISIBLE_BARS);

// Force redraw
ChartRedraw(0);
```

### Chart Navigation

```mql5
// Change symbol/timeframe
ChartSetSymbolPeriod(0, "EURUSD", PERIOD_H1);

// Navigate to specific position
ChartNavigate(0, CHART_END, 0);           // go to end
ChartNavigate(0, CHART_BEGIN, 0);         // go to beginning
ChartNavigate(0, CHART_CURRENT_POS, -50); // scroll left 50 bars

// Iterate all open charts
long chartId = ChartFirst();
while(chartId >= 0)
{
   string sym = ChartSymbol(chartId);
   ENUM_TIMEFRAMES tf = ChartPeriod(chartId);
   Print(sym, " ", EnumToString(tf));
   chartId = ChartNext(chartId);
}

// Open new chart
long newChart = ChartOpen("GBPUSD", PERIOD_M15);

// Coordinate conversion
datetime time;
double   price;
int      subwindow;
ChartXYToTimePrice(0, pixelX, pixelY, subwindow, time, price);

int x, y;
ChartTimePriceToXY(0, 0, time, price, x, y);
```

### Chart Templates

```mql5
// Apply template
ChartApplyTemplate(0, "MyTemplate.tpl");

// Save template
ChartSaveTemplate(0, "MyTemplate.tpl");

// Screenshot
ChartScreenShot(0, "screenshot.png", 1920, 1080, ALIGN_RIGHT);
```
