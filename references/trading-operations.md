# Trading Operations Reference

## Table of Contents

- [Order Management - MQL4](#order-management---mql4)
- [Order Management - MQL5](#order-management---mql5)
- [Risk Management](#risk-management)
- [Trailing Stop Implementations](#trailing-stop-implementations)
- [Trade Return Codes (MQL5)](#trade-return-codes-mql5)

---

## Order Management - MQL4

### OrderSend with Error Handling and Retry

```mql4
int SendOrderReliable(string symbol, int cmd, double lots, double price,
                      int slippage, double sl, double tp,
                      string comment, int magic, int maxRetries = 5)
{
   int ticket = -1;
   int digits = (int)MarketInfo(symbol, MODE_DIGITS);
   double point = MarketInfo(symbol, MODE_POINT);
   double stopLevel = MarketInfo(symbol, MODE_STOPLEVEL) * point;

   price = NormalizeDouble(price, digits);
   sl    = NormalizeDouble(sl, digits);
   tp    = NormalizeDouble(tp, digits);

   // Validate stop level distances
   if(sl != 0)
   {
      if(cmd == OP_BUY && (price - sl) < stopLevel)
      {
         Print("SL too close. Min distance: ", stopLevel / point, " points");
         return(-1);
      }
      if(cmd == OP_SELL && (sl - price) < stopLevel)
      {
         Print("SL too close. Min distance: ", stopLevel / point, " points");
         return(-1);
      }
   }

   for(int attempt = 0; attempt < maxRetries; attempt++)
   {
      if(attempt > 0)
      {
         int sleepMs = 1000 * (int)MathPow(2, attempt - 1); // 1s, 2s, 4s, 8s
         Sleep(sleepMs);
         RefreshRates();
         // Update price for market orders
         if(cmd == OP_BUY)  price = MarketInfo(symbol, MODE_ASK);
         if(cmd == OP_SELL) price = MarketInfo(symbol, MODE_BID);
         price = NormalizeDouble(price, digits);
      }

      ticket = OrderSend(symbol, cmd, lots, price, slippage, sl, tp, comment, magic, 0, clrNONE);

      if(ticket >= 0)
         return(ticket);

      int err = GetLastError();
      Print("OrderSend attempt ", attempt + 1, " failed. Error: ", err, " - ", ErrorDescription(err));

      // Retryable errors
      if(err == ERR_REQUOTE ||
         err == ERR_PRICE_CHANGED ||
         err == ERR_OFF_QUOTES ||
         err == ERR_SERVER_BUSY ||
         err == ERR_BROKER_BUSY ||
         err == ERR_TRADE_CONTEXT_BUSY ||
         err == ERR_TOO_MANY_REQUESTS ||
         err == ERR_TRADE_TIMEOUT ||
         err == ERR_NO_CONNECTION)
      {
         continue;
      }

      // Non-retryable errors - exit immediately
      break;
   }

   return(-1);
}
```

### Order Modification Pattern

```mql4
bool ModifyOrderReliable(int ticket, double sl, double tp)
{
   if(!OrderSelect(ticket, SELECT_BY_TICKET))
   {
      Print("Order ", ticket, " not found");
      return(false);
   }

   int digits = (int)MarketInfo(OrderSymbol(), MODE_DIGITS);
   sl = NormalizeDouble(sl, digits);
   tp = NormalizeDouble(tp, digits);

   // Skip if nothing changed
   if(MathAbs(OrderStopLoss() - sl) < Point / 2.0 &&
      MathAbs(OrderTakeProfit() - tp) < Point / 2.0)
      return(true);

   bool result = OrderModify(ticket, OrderOpenPrice(), sl, tp, OrderExpiration(), clrNONE);

   if(!result)
   {
      int err = GetLastError();
      Print("OrderModify failed for ticket ", ticket, ". Error: ", err, " - ", ErrorDescription(err));
   }

   return(result);
}
```

### Order Loop Patterns

**CRITICAL: Always use reverse loops when closing/deleting orders.** Closing an order shifts indices, so forward loops skip orders.

#### Close All Orders for a Symbol

```mql4
void CloseAllOrders(string symbol, int magic)
{
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!OrderSelect(i, SELECT_BY_POS, MODE_TRADES)) continue;
      if(OrderSymbol() != symbol) continue;
      if(OrderMagicNumber() != magic) continue;

      bool result = false;
      if(OrderType() == OP_BUY)
         result = OrderClose(OrderTicket(), OrderLots(), MarketInfo(symbol, MODE_BID), 3, clrNONE);
      else if(OrderType() == OP_SELL)
         result = OrderClose(OrderTicket(), OrderLots(), MarketInfo(symbol, MODE_ASK), 3, clrNONE);
      else
         result = OrderDelete(OrderTicket()); // Pending orders

      if(!result)
         Print("Failed to close order ", OrderTicket(), ". Error: ", GetLastError());
   }
}
```

#### Count Orders by Type and Magic Number

```mql4
int CountOrders(string symbol, int magic, int orderType = -1)
{
   int count = 0;
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!OrderSelect(i, SELECT_BY_POS, MODE_TRADES)) continue;
      if(OrderSymbol() != symbol) continue;
      if(OrderMagicNumber() != magic) continue;
      if(orderType >= 0 && OrderType() != orderType) continue;
      count++;
   }
   return(count);
}
```

#### Find Most Recent Order

```mql4
int FindLatestOrder(string symbol, int magic)
{
   int latestTicket = -1;
   datetime latestTime = 0;

   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!OrderSelect(i, SELECT_BY_POS, MODE_TRADES)) continue;
      if(OrderSymbol() != symbol) continue;
      if(OrderMagicNumber() != magic) continue;

      if(OrderOpenTime() > latestTime)
      {
         latestTime = OrderOpenTime();
         latestTicket = OrderTicket();
      }
   }
   return(latestTicket);
}
```

---

## Order Management - MQL5

### CTrade Setup

```mql5
#include <Trade/Trade.mqh>

CTrade trade;

int OnInit()
{
   trade.SetExpertMagicNumber(MagicNumber);
   trade.SetDeviationInPoints(Slippage);
   SetFillingPolicy(trade, _Symbol);
   return(INIT_SUCCEEDED);
}
```

### Filling Policy Detection

Brokers support different order filling modes. Setting the wrong filling type causes order rejection.

```mql5
void SetFillingPolicy(CTrade &trade, string symbol)
{
   long filling = SymbolInfoInteger(symbol, SYMBOL_FILLING_MODE);

   if((filling & SYMBOL_FILLING_FOK) != 0)
      trade.SetTypeFilling(ORDER_FILLING_FOK);
   else if((filling & SYMBOL_FILLING_IOC) != 0)
      trade.SetTypeFilling(ORDER_FILLING_IOC);
   else
      trade.SetTypeFilling(ORDER_FILLING_RETURN);
}
```

**Filling mode meanings:**
- **FOK (Fill or Kill):** Entire order must be filled at the requested price or better, otherwise it is cancelled entirely.
- **IOC (Immediate or Cancel):** Fill as much as possible immediately, cancel the remainder.
- **RETURN:** Partial fills allowed; unfilled portion remains as a live order. Common on exchange-traded instruments.

### Trade with Retry Logic (MQL5)

```mql5
class CTradeManager
{
private:
   CTrade   m_trade;
   int      m_maxRetries;
   int      m_magic;

   bool IsRetryable(uint retcode)
   {
      switch(retcode)
      {
         case TRADE_RETCODE_REQUOTE:          // 10004
         case TRADE_RETCODE_PRICE_CHANGED:    // 10020
         case TRADE_RETCODE_PRICE_OFF:        // 10021
         case TRADE_RETCODE_CONNECTION:        // 10031
         case TRADE_RETCODE_TIMEOUT:          // 10012
         case TRADE_RETCODE_TOO_MANY_REQUESTS:// 10024
            return(true);
         default:
            return(false);
      }
   }

public:
   CTradeManager(int magic, int slippage, int maxRetries = 5)
   {
      m_magic = magic;
      m_maxRetries = maxRetries;
      m_trade.SetExpertMagicNumber(magic);
      m_trade.SetDeviationInPoints(slippage);
   }

   bool Init(string symbol)
   {
      SetFillingPolicy(m_trade, symbol);
      return(true);
   }

   bool Buy(string symbol, double lots, double sl = 0, double tp = 0, string comment = "")
   {
      for(int attempt = 0; attempt < m_maxRetries; attempt++)
      {
         if(attempt > 0)
            Sleep(1000 * (int)MathPow(2, attempt - 1));

         double ask = SymbolInfoDouble(symbol, SYMBOL_ASK);

         if(m_trade.Buy(lots, symbol, ask, sl, tp, comment))
         {
            uint retcode = m_trade.ResultRetcode();
            if(retcode == TRADE_RETCODE_DONE || retcode == TRADE_RETCODE_PLACED)
               return(true);

            if(!IsRetryable(retcode))
            {
               Print("Buy failed (non-retryable): ", retcode, " - ", m_trade.ResultRetcodeDescription());
               return(false);
            }
            Print("Buy attempt ", attempt + 1, " retryable error: ", retcode);
         }
         else
         {
            uint retcode = m_trade.ResultRetcode();
            if(!IsRetryable(retcode))
            {
               Print("Buy failed (non-retryable): ", retcode, " - ", m_trade.ResultRetcodeDescription());
               return(false);
            }
            Print("Buy attempt ", attempt + 1, " retryable error: ", retcode);
         }
      }
      Print("Buy failed after ", m_maxRetries, " attempts");
      return(false);
   }

   bool Sell(string symbol, double lots, double sl = 0, double tp = 0, string comment = "")
   {
      for(int attempt = 0; attempt < m_maxRetries; attempt++)
      {
         if(attempt > 0)
            Sleep(1000 * (int)MathPow(2, attempt - 1));

         double bid = SymbolInfoDouble(symbol, SYMBOL_BID);

         if(m_trade.Sell(lots, symbol, bid, sl, tp, comment))
         {
            uint retcode = m_trade.ResultRetcode();
            if(retcode == TRADE_RETCODE_DONE || retcode == TRADE_RETCODE_PLACED)
               return(true);

            if(!IsRetryable(retcode))
            {
               Print("Sell failed (non-retryable): ", retcode, " - ", m_trade.ResultRetcodeDescription());
               return(false);
            }
            Print("Sell attempt ", attempt + 1, " retryable error: ", retcode);
         }
         else
         {
            uint retcode = m_trade.ResultRetcode();
            if(!IsRetryable(retcode))
            {
               Print("Sell failed (non-retryable): ", retcode, " - ", m_trade.ResultRetcodeDescription());
               return(false);
            }
            Print("Sell attempt ", attempt + 1, " retryable error: ", retcode);
         }
      }
      Print("Sell failed after ", m_maxRetries, " attempts");
      return(false);
   }

   CTrade* GetTrade() { return(&m_trade); }
};
```

### Position Loop Patterns (MQL5)

#### Iterate All Positions with Filtering

```mql5
void ProcessPositions(string symbol, int magic)
{
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;

      if(PositionGetString(POSITION_SYMBOL) != symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC) != magic) continue;

      double volume    = PositionGetDouble(POSITION_VOLUME);
      double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      double profit    = PositionGetDouble(POSITION_PROFIT);
      long   type      = PositionGetInteger(POSITION_TYPE);

      // Process position...
   }
}
```

#### Close All Positions for a Symbol

```mql5
void CloseAllPositions(string symbol, int magic)
{
   CTrade trade;
   trade.SetExpertMagicNumber(magic);
   SetFillingPolicy(trade, symbol);

   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC) != magic) continue;

      if(!trade.PositionClose(ticket))
         Print("Failed to close position ", ticket, ": ", trade.ResultRetcodeDescription());
   }
}
```

#### Count Positions by Type

```mql5
int CountPositions(string symbol, int magic, ENUM_POSITION_TYPE posType = -1)
{
   int count = 0;
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC) != magic) continue;
      if(posType >= 0 && (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE) != posType) continue;
      count++;
   }
   return(count);
}
```

---

## Risk Management

### Position Sizing Formulas

#### Fixed Lot

```mql5
double FixedLot(double lotSize, string symbol)
{
   double minLot  = SymbolInfoDouble(symbol, SYMBOL_VOLUME_MIN);
   double maxLot  = SymbolInfoDouble(symbol, SYMBOL_VOLUME_MAX);
   double lotStep = SymbolInfoDouble(symbol, SYMBOL_VOLUME_STEP);

   lotSize = MathFloor(lotSize / lotStep) * lotStep;
   lotSize = MathMax(minLot, MathMin(maxLot, lotSize));
   return(NormalizeDouble(lotSize, 2));
}
```

Simplest approach but ignores account growth and does not adapt to varying stop loss distances.

#### Percentage Risk (Recommended Default)

The core formula:

```
lots = (AccountBalance * RiskPercent / 100) / (SL_Points * TickValue / TickSize)
```

**MQL5 Implementation:**

```mql5
double CalcLotSize(string symbol, double riskPercent, double slPoints)
{
   if(slPoints <= 0)
   {
      Print("CalcLotSize: SL distance must be > 0");
      return(0);
   }

   double balance  = AccountInfoDouble(ACCOUNT_BALANCE);
   double tickVal  = SymbolInfoDouble(symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize = SymbolInfoDouble(symbol, SYMBOL_TRADE_TICK_SIZE);
   double minLot   = SymbolInfoDouble(symbol, SYMBOL_VOLUME_MIN);
   double maxLot   = SymbolInfoDouble(symbol, SYMBOL_VOLUME_MAX);
   double lotStep  = SymbolInfoDouble(symbol, SYMBOL_VOLUME_STEP);
   double point    = SymbolInfoDouble(symbol, SYMBOL_POINT);

   if(tickVal == 0 || tickSize == 0)
   {
      Print("CalcLotSize: Invalid tick info for ", symbol);
      return(minLot);
   }

   double riskMoney = balance * riskPercent / 100.0;
   double slValue   = slPoints * point * (tickVal / tickSize);
   double lots      = riskMoney / slValue;

   // Round down to lot step
   lots = MathFloor(lots / lotStep) * lotStep;

   // Clamp
   lots = MathMax(minLot, MathMin(maxLot, lots));

   // Verify margin is sufficient
   double margin = 0;
   if(!OrderCalcMargin(ORDER_TYPE_BUY, symbol, lots, SymbolInfoDouble(symbol, SYMBOL_ASK), margin))
   {
      Print("CalcLotSize: OrderCalcMargin failed");
      return(minLot);
   }

   if(margin > AccountInfoDouble(ACCOUNT_MARGIN_FREE))
   {
      Print("CalcLotSize: Insufficient free margin. Required: ", margin);
      // Reduce lots to fit available margin
      double freeMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE) * 0.95; // 5% safety buffer
      lots = lots * (freeMargin / margin);
      lots = MathFloor(lots / lotStep) * lotStep;
      lots = MathMax(minLot, lots);
   }

   return(NormalizeDouble(lots, 2));
}
```

**MQL4 Implementation:**

```mql4
double CalcLotSize_MQL4(string symbol, double riskPercent, double slPoints)
{
   if(slPoints <= 0)
   {
      Print("CalcLotSize: SL distance must be > 0");
      return(0);
   }

   double balance  = AccountBalance();
   double tickVal  = MarketInfo(symbol, MODE_TICKVALUE);
   double tickSize = MarketInfo(symbol, MODE_TICKSIZE);
   double minLot   = MarketInfo(symbol, MODE_MINLOT);
   double maxLot   = MarketInfo(symbol, MODE_MAXLOT);
   double lotStep  = MarketInfo(symbol, MODE_LOTSTEP);
   double point    = MarketInfo(symbol, MODE_POINT);

   if(tickVal == 0 || tickSize == 0)
   {
      Print("CalcLotSize: Invalid tick info for ", symbol);
      return(minLot);
   }

   double riskMoney = balance * riskPercent / 100.0;
   double slValue   = slPoints * point * (tickVal / tickSize);
   double lots      = riskMoney / slValue;

   lots = MathFloor(lots / lotStep) * lotStep;
   lots = MathMax(minLot, MathMin(maxLot, lots));

   // Verify margin
   double marginRequired = AccountFreeMarginCheck(symbol, OP_BUY, lots);
   if(marginRequired <= 0)
   {
      Print("CalcLotSize: Insufficient margin");
      return(minLot);
   }

   return(NormalizeDouble(lots, 2));
}
```

#### Kelly Criterion

```
kelly_fraction = (winRate * avgWin - (1 - winRate) * avgLoss) / avgWin
```

```mql5
double KellyLotSize(string symbol, double winRate, double avgWin, double avgLoss,
                    double slPoints, double kellyFraction = 0.25)
{
   // Full Kelly is too aggressive; use quarter-Kelly as maximum
   double kelly = (winRate * avgWin - (1.0 - winRate) * avgLoss) / avgWin;

   if(kelly <= 0)
   {
      Print("Kelly criterion negative - system has no edge");
      return(0);
   }

   double riskPercent = kelly * kellyFraction * 100.0;
   riskPercent = MathMin(riskPercent, 5.0); // Hard cap at 5%

   return(CalcLotSize(symbol, riskPercent, slPoints));
}
```

**WARNING:** Full Kelly sizing leads to extreme drawdowns in practice. Always use quarter-Kelly (kellyFraction = 0.25) or less. Even quarter-Kelly can produce 40-50% drawdowns. Most professional traders use eighth-Kelly or fixed fractional risk of 1-2%.

### Multi-Market Lot Calculation

Different instruments have different contract sizes and tick values:

| Market | Typical Contract Size | Notes |
|---|---|---|
| Forex | 100,000 units (1 standard lot) | Tick value varies by pair denomination |
| Indices (CFD) | Variable (often 1 or 10 per point) | Contract size varies by broker and index |
| Gold (XAUUSD) | 100 troy ounces (1 lot) | Large tick value per lot |
| Oil (crude) | 1,000 barrels (1 lot) | Varies by broker |
| Crypto | Variable | Bitcoin often 1 BTC per lot |

**Always use SymbolInfo functions to get accurate values per instrument:**

```mql5
void PrintSymbolTradeInfo(string symbol)
{
   double contractSize = SymbolInfoDouble(symbol, SYMBOL_TRADE_CONTRACT_SIZE);
   double tickValue    = SymbolInfoDouble(symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize     = SymbolInfoDouble(symbol, SYMBOL_TRADE_TICK_SIZE);
   double point        = SymbolInfoDouble(symbol, SYMBOL_POINT);
   double minLot       = SymbolInfoDouble(symbol, SYMBOL_VOLUME_MIN);
   double maxLot       = SymbolInfoDouble(symbol, SYMBOL_VOLUME_MAX);
   double lotStep      = SymbolInfoDouble(symbol, SYMBOL_VOLUME_STEP);
   int    digits       = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);

   PrintFormat("%s: Contract=%.2f TickVal=%.5f TickSize=%.5f Point=%.5f Digits=%d",
               symbol, contractSize, tickValue, tickSize, point, digits);
   PrintFormat("  MinLot=%.2f MaxLot=%.2f LotStep=%.2f", minLot, maxLot, lotStep);
}
```

**Key rule:** Never hardcode tick values or contract sizes. The `CalcLotSize` function above handles all markets correctly because it reads the values dynamically from the symbol properties.

### Drawdown Control

```mql5
class CDrawdownManager
{
private:
   double   m_startBalance;
   double   m_dailyStartBalance;
   double   m_maxDailyLossPercent;   // e.g., 3.0
   double   m_maxTotalDDPercent;     // e.g., 10.0
   datetime m_currentDay;
   bool     m_dailyLocked;
   bool     m_totalLocked;

public:
   CDrawdownManager(double maxDailyLoss = 3.0, double maxTotalDD = 10.0)
   {
      m_maxDailyLossPercent = maxDailyLoss;
      m_maxTotalDDPercent   = maxTotalDD;
      m_startBalance        = AccountInfoDouble(ACCOUNT_BALANCE);
      m_dailyStartBalance   = m_startBalance;
      m_currentDay          = iTime(_Symbol, PERIOD_D1, 0);
      m_dailyLocked         = false;
      m_totalLocked         = false;
   }

   void OnNewDay()
   {
      datetime today = iTime(_Symbol, PERIOD_D1, 0);
      if(today != m_currentDay)
      {
         m_currentDay = today;
         m_dailyStartBalance = AccountInfoDouble(ACCOUNT_BALANCE);
         m_dailyLocked = false;
         // Total lock is NOT reset on new day
      }
   }

   bool CanTrade()
   {
      OnNewDay();

      double equity = AccountInfoDouble(ACCOUNT_EQUITY);

      // Daily drawdown check
      double dailyLoss = (m_dailyStartBalance - equity) / m_dailyStartBalance * 100.0;
      if(dailyLoss >= m_maxDailyLossPercent)
      {
         if(!m_dailyLocked)
         {
            Print("Daily drawdown limit reached: ", DoubleToString(dailyLoss, 2), "%");
            m_dailyLocked = true;
         }
         return(false);
      }

      // Total drawdown check
      double totalDD = (m_startBalance - equity) / m_startBalance * 100.0;
      if(totalDD >= m_maxTotalDDPercent)
      {
         if(!m_totalLocked)
         {
            Print("Total drawdown limit reached: ", DoubleToString(totalDD, 2), "%");
            m_totalLocked = true;
         }
         return(false);
      }

      return(true);
   }

   void Reset()
   {
      m_startBalance = AccountInfoDouble(ACCOUNT_BALANCE);
      m_dailyStartBalance = m_startBalance;
      m_dailyLocked = false;
      m_totalLocked = false;
   }

   double GetDailyDD()
   {
      double equity = AccountInfoDouble(ACCOUNT_EQUITY);
      return((m_dailyStartBalance - equity) / m_dailyStartBalance * 100.0);
   }

   double GetTotalDD()
   {
      double equity = AccountInfoDouble(ACCOUNT_EQUITY);
      return((m_startBalance - equity) / m_startBalance * 100.0);
   }
};
```

### Margin Verification Before Trade

```mql5
bool HasSufficientMargin(string symbol, ENUM_ORDER_TYPE orderType, double lots, double safetyFactor = 0.9)
{
   double price = (orderType == ORDER_TYPE_BUY)
                  ? SymbolInfoDouble(symbol, SYMBOL_ASK)
                  : SymbolInfoDouble(symbol, SYMBOL_BID);

   double margin = 0;
   if(!OrderCalcMargin(orderType, symbol, lots, price, margin))
   {
      Print("OrderCalcMargin failed: ", GetLastError());
      return(false);
   }

   double freeMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);

   if(margin > freeMargin * safetyFactor)
   {
      PrintFormat("Insufficient margin. Required: %.2f, Available: %.2f (safety: %.0f%%)",
                  margin, freeMargin, safetyFactor * 100);
      return(false);
   }

   return(true);
}
```

**MQL4 equivalent:**

```mql4
bool HasSufficientMargin_MQL4(string symbol, int orderType, double lots)
{
   double freeMargin = AccountFreeMarginCheck(symbol, orderType, lots);

   if(freeMargin <= 0 || GetLastError() == ERR_NOT_ENOUGH_MONEY)
   {
      Print("Insufficient margin for ", lots, " lots on ", symbol);
      return(false);
   }

   return(true);
}
```

---

## Trailing Stop Implementations

### CTrailingManager (MQL5)

```mql5
#include <Trade/Trade.mqh>
#include <Trade/PositionInfo.mqh>
#include <Indicators/Trend.mqh>

class CTrailingManager
{
private:
   CTrade         m_trade;
   int            m_magic;

   bool ModifySL(ulong ticket, double newSL)
   {
      if(!PositionSelectByTicket(ticket))
         return(false);

      double currentSL = PositionGetDouble(POSITION_SL);
      double tp        = PositionGetDouble(POSITION_TP);
      string symbol    = PositionGetString(POSITION_SYMBOL);
      int    digits    = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);

      newSL = NormalizeDouble(newSL, digits);

      // Skip if SL hasn't changed meaningfully
      if(MathAbs(currentSL - newSL) < SymbolInfoDouble(symbol, SYMBOL_POINT))
         return(true);

      return(m_trade.PositionModify(ticket, newSL, tp));
   }

public:
   CTrailingManager(int magic, int slippage = 10)
   {
      m_magic = magic;
      m_trade.SetExpertMagicNumber(magic);
      m_trade.SetDeviationInPoints(slippage);
   }

   //--- Fixed Trailing Stop ---
   // trailPoints: distance in points to trail behind current price
   // activationPoints: minimum profit in points before trailing activates (0 = immediate)
   void TrailFixed(string symbol, double trailPoints, double activationPoints = 0)
   {
      double point = SymbolInfoDouble(symbol, SYMBOL_POINT);
      double stopLevel = SymbolInfoInteger(symbol, SYMBOL_TRADE_STOPS_LEVEL) * point;
      double trailDist = MathMax(trailPoints * point, stopLevel);

      for(int i = PositionsTotal() - 1; i >= 0; i--)
      {
         ulong ticket = PositionGetTicket(i);
         if(ticket == 0) continue;
         if(PositionGetString(POSITION_SYMBOL) != symbol) continue;
         if(PositionGetInteger(POSITION_MAGIC) != m_magic) continue;

         double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
         double currentSL = PositionGetDouble(POSITION_SL);
         long   type      = PositionGetInteger(POSITION_TYPE);

         if(type == POSITION_TYPE_BUY)
         {
            double bid = SymbolInfoDouble(symbol, SYMBOL_BID);
            double profitDist = bid - openPrice;

            if(activationPoints > 0 && profitDist < activationPoints * point)
               continue;

            double newSL = bid - trailDist;
            if(newSL > currentSL || currentSL == 0)
               ModifySL(ticket, newSL);
         }
         else if(type == POSITION_TYPE_SELL)
         {
            double ask = SymbolInfoDouble(symbol, SYMBOL_ASK);
            double profitDist = openPrice - ask;

            if(activationPoints > 0 && profitDist < activationPoints * point)
               continue;

            double newSL = ask + trailDist;
            if(newSL < currentSL || currentSL == 0)
               ModifySL(ticket, newSL);
         }
      }
   }

   //--- ATR-Based Trailing Stop ---
   // atrHandle: handle from iATR()
   // multiplier: ATR multiplier for trail distance (e.g., 2.0)
   void TrailATR(string symbol, int atrHandle, double multiplier = 2.0)
   {
      double atrBuffer[];
      ArraySetAsSeries(atrBuffer, true);

      if(CopyBuffer(atrHandle, 0, 1, 1, atrBuffer) != 1)
      {
         Print("TrailATR: Failed to copy ATR buffer");
         return;
      }

      double atrValue  = atrBuffer[0];
      double trailDist = atrValue * multiplier;
      double stopLevel = SymbolInfoInteger(symbol, SYMBOL_TRADE_STOPS_LEVEL) *
                         SymbolInfoDouble(symbol, SYMBOL_POINT);

      trailDist = MathMax(trailDist, stopLevel);

      for(int i = PositionsTotal() - 1; i >= 0; i--)
      {
         ulong ticket = PositionGetTicket(i);
         if(ticket == 0) continue;
         if(PositionGetString(POSITION_SYMBOL) != symbol) continue;
         if(PositionGetInteger(POSITION_MAGIC) != m_magic) continue;

         double currentSL = PositionGetDouble(POSITION_SL);
         long   type      = PositionGetInteger(POSITION_TYPE);

         if(type == POSITION_TYPE_BUY)
         {
            double bid  = SymbolInfoDouble(symbol, SYMBOL_BID);
            double newSL = bid - trailDist;

            if(newSL > currentSL || currentSL == 0)
               ModifySL(ticket, newSL);
         }
         else if(type == POSITION_TYPE_SELL)
         {
            double ask  = SymbolInfoDouble(symbol, SYMBOL_ASK);
            double newSL = ask + trailDist;

            if(newSL < currentSL || currentSL == 0)
               ModifySL(ticket, newSL);
         }
      }
   }

   //--- Parabolic SAR Trailing ---
   // sarHandle: handle from iSAR()
   // Uses bar[1] (last completed bar) for confirmed SAR value
   void TrailParabolicSAR(string symbol, int sarHandle)
   {
      double sarBuffer[];
      ArraySetAsSeries(sarBuffer, true);

      // Use bar index 1 (last closed bar) for confirmed value
      if(CopyBuffer(sarHandle, 0, 1, 1, sarBuffer) != 1)
      {
         Print("TrailSAR: Failed to copy SAR buffer");
         return;
      }

      double sarValue  = sarBuffer[0];
      double stopLevel = SymbolInfoInteger(symbol, SYMBOL_TRADE_STOPS_LEVEL) *
                         SymbolInfoDouble(symbol, SYMBOL_POINT);

      for(int i = PositionsTotal() - 1; i >= 0; i--)
      {
         ulong ticket = PositionGetTicket(i);
         if(ticket == 0) continue;
         if(PositionGetString(POSITION_SYMBOL) != symbol) continue;
         if(PositionGetInteger(POSITION_MAGIC) != m_magic) continue;

         double currentSL = PositionGetDouble(POSITION_SL);
         double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
         long   type      = PositionGetInteger(POSITION_TYPE);

         if(type == POSITION_TYPE_BUY)
         {
            double bid = SymbolInfoDouble(symbol, SYMBOL_BID);

            // SAR must be below price for buy (confirming uptrend)
            if(sarValue >= bid) continue;

            // Ensure minimum stop level distance
            if((bid - sarValue) < stopLevel) continue;

            // Only move SL forward
            if(sarValue > currentSL || currentSL == 0)
               ModifySL(ticket, sarValue);
         }
         else if(type == POSITION_TYPE_SELL)
         {
            double ask = SymbolInfoDouble(symbol, SYMBOL_ASK);

            // SAR must be above price for sell (confirming downtrend)
            if(sarValue <= ask) continue;

            // Ensure minimum stop level distance
            if((sarValue - ask) < stopLevel) continue;

            // Only move SL closer to entry (lower for sells)
            if(sarValue < currentSL || currentSL == 0)
               ModifySL(ticket, sarValue);
         }
      }
   }

   //--- Break-Even Logic ---
   // activationPoints: profit in points before moving to break-even
   // lockInPoints: points above entry to lock in (0 = exact entry, positive = small profit)
   void ManageBreakEven(string symbol, double activationPoints, double lockInPoints = 0)
   {
      double point = SymbolInfoDouble(symbol, SYMBOL_POINT);

      for(int i = PositionsTotal() - 1; i >= 0; i--)
      {
         ulong ticket = PositionGetTicket(i);
         if(ticket == 0) continue;
         if(PositionGetString(POSITION_SYMBOL) != symbol) continue;
         if(PositionGetInteger(POSITION_MAGIC) != m_magic) continue;

         double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
         double currentSL = PositionGetDouble(POSITION_SL);
         long   type      = PositionGetInteger(POSITION_TYPE);

         if(type == POSITION_TYPE_BUY)
         {
            double bid  = SymbolInfoDouble(symbol, SYMBOL_BID);
            double beSL = openPrice + lockInPoints * point;

            // Check if profit threshold reached and SL not already at or above BE
            if((bid - openPrice) >= activationPoints * point)
            {
               if(currentSL < beSL)
                  ModifySL(ticket, beSL);
            }
         }
         else if(type == POSITION_TYPE_SELL)
         {
            double ask  = SymbolInfoDouble(symbol, SYMBOL_ASK);
            double beSL = openPrice - lockInPoints * point;

            if((openPrice - ask) >= activationPoints * point)
            {
               if(currentSL > beSL || currentSL == 0)
                  ModifySL(ticket, beSL);
            }
         }
      }
   }
};
```

**Usage example:**

```mql5
// In global scope
CTrailingManager *trailMgr;
int atrHandle;
int sarHandle;

int OnInit()
{
   trailMgr  = new CTrailingManager(MagicNumber, Slippage);
   atrHandle = iATR(_Symbol, PERIOD_CURRENT, 14);
   sarHandle = iSAR(_Symbol, PERIOD_CURRENT, 0.02, 0.2);
   return(INIT_SUCCEEDED);
}

void OnTick()
{
   // Choose one trailing method:
   // trailMgr.TrailFixed(_Symbol, 50, 30);
   // trailMgr.TrailATR(_Symbol, atrHandle, 2.0);
   // trailMgr.TrailParabolicSAR(_Symbol, sarHandle);

   // Break-even can be combined with any trailing method
   trailMgr.ManageBreakEven(_Symbol, 30, 5); // Activate at 30pts profit, lock 5pts
}

void OnDeinit(const int reason)
{
   delete trailMgr;
   IndicatorRelease(atrHandle);
   IndicatorRelease(sarHandle);
}
```

### Fixed Trailing Stop (MQL4 Version)

```mql4
void TrailFixedMQL4(string symbol, int magic, double trailPoints)
{
   double point = MarketInfo(symbol, MODE_POINT);
   int    digits = (int)MarketInfo(symbol, MODE_DIGITS);
   double stopLevel = MarketInfo(symbol, MODE_STOPLEVEL) * point;
   double trailDist = MathMax(trailPoints * point, stopLevel);

   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(!OrderSelect(i, SELECT_BY_POS, MODE_TRADES)) continue;
      if(OrderSymbol() != symbol) continue;
      if(OrderMagicNumber() != magic) continue;

      if(OrderType() == OP_BUY)
      {
         double bid  = MarketInfo(symbol, MODE_BID);
         double newSL = NormalizeDouble(bid - trailDist, digits);

         if(newSL > OrderStopLoss() || OrderStopLoss() == 0)
            OrderModify(OrderTicket(), OrderOpenPrice(), newSL, OrderTakeProfit(), 0, clrNONE);
      }
      else if(OrderType() == OP_SELL)
      {
         double ask  = MarketInfo(symbol, MODE_ASK);
         double newSL = NormalizeDouble(ask + trailDist, digits);

         if(newSL < OrderStopLoss() || OrderStopLoss() == 0)
            OrderModify(OrderTicket(), OrderOpenPrice(), newSL, OrderTakeProfit(), 0, clrNONE);
      }
   }
}
```

---

## Trade Return Codes (MQL5)

| Constant | Value | Meaning |
|---|---|---|
| `TRADE_RETCODE_REQUOTE` | 10004 | Requote - price changed during processing |
| `TRADE_RETCODE_REJECT` | 10006 | Request rejected by server |
| `TRADE_RETCODE_CANCEL` | 10007 | Request cancelled by trader |
| `TRADE_RETCODE_PLACED` | 10008 | Order placed (pending) |
| `TRADE_RETCODE_DONE` | 10009 | Request completed successfully |
| `TRADE_RETCODE_DONE_PARTIAL` | 10010 | Request partially completed |
| `TRADE_RETCODE_TIMEOUT` | 10012 | Request timed out |
| `TRADE_RETCODE_INVALID` | 10013 | Invalid request parameters |
| `TRADE_RETCODE_INVALID_VOLUME` | 10014 | Invalid volume (lot size) |
| `TRADE_RETCODE_INVALID_PRICE` | 10015 | Invalid price |
| `TRADE_RETCODE_INVALID_STOPS` | 10016 | Invalid SL/TP levels |
| `TRADE_RETCODE_TRADE_DISABLED` | 10017 | Trading disabled for this symbol |
| `TRADE_RETCODE_MARKET_CLOSED` | 10018 | Market is closed |
| `TRADE_RETCODE_NO_MONEY` | 10019 | Insufficient funds |
| `TRADE_RETCODE_PRICE_CHANGED` | 10020 | Price changed since request |
| `TRADE_RETCODE_PRICE_OFF` | 10021 | No quotes available |
| `TRADE_RETCODE_INVALID_EXPIRATION` | 10022 | Invalid order expiration |
| `TRADE_RETCODE_ORDER_CHANGED` | 10023 | Order state changed |
| `TRADE_RETCODE_TOO_MANY_REQUESTS` | 10024 | Too many requests |
| `TRADE_RETCODE_CONNECTION` | 10031 | No connection to trade server |

**Handling guidance:**

- **Success:** 10008 and 10009 indicate the trade was accepted. Always check for both.
- **Retryable:** 10004, 10012, 10020, 10021, 10024, 10031 can be retried with backoff.
- **Fix and retry:** 10014 (adjust volume), 10015 (refresh price), 10016 (adjust stops).
- **Fatal:** 10006, 10013, 10017, 10018, 10019 usually require user intervention or logic changes.

### Interpreting Trade Results

```mql5
void LogTradeResult(CTrade &trade)
{
   uint retcode = trade.ResultRetcode();

   switch(retcode)
   {
      case TRADE_RETCODE_DONE:
      case TRADE_RETCODE_PLACED:
         PrintFormat("Trade OK: Order #%d, Deal #%d, Volume=%.2f, Price=%.5f",
                     trade.ResultOrder(), trade.ResultDeal(),
                     trade.ResultVolume(), trade.ResultPrice());
         break;

      case TRADE_RETCODE_DONE_PARTIAL:
         PrintFormat("Partial fill: Volume=%.2f of requested. Order #%d",
                     trade.ResultVolume(), trade.ResultOrder());
         break;

      default:
         PrintFormat("Trade FAILED: retcode=%u (%s), Comment: %s",
                     retcode, trade.ResultRetcodeDescription(),
                     trade.ResultComment());
         break;
   }
}
```
