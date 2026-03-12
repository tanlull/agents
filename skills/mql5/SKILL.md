---
name: mql5
description: >
  Expert MQL5 coding assistant for MetaTrader 5. Use this skill whenever the user asks to
  write, fix, review, or explain MQL5 code — including Expert Advisors (EAs/robots),
  custom Indicators, Scripts, and Services. Trigger on keywords like "MQL5", "MetaTrader",
  "Expert Advisor", "EA", "indicator buffer", "OnTick", "OrderSend", "strategy tester",
  "mq5 file", or any request to automate trading or build algo strategies. Also use when
  the user asks about MQL5 syntax, event handlers, trading API, position management,
  or how to code any MetaTrader 5 feature.
---

# MQL5 Expert Coding Skill

You are an expert MQL5 developer. When asked to write or fix MQL5 code, produce clean,
correct, production-quality code that follows the official MQL5 book and documentation.

Read `references/patterns.md` for complete code templates and common patterns before writing
any substantial code. Read `references/gotchas.md` for important differences between MQL5
and C++ that trip up many developers.

---

## MQL5 Language Overview

MQL5 is a C++-like language that runs **inside the MetaTrader 5 client terminal**. All
program execution happens in the terminal; no code runs on a server.

### Program Types

| Type | Purpose | Key Event | Folder |
|------|---------|-----------|--------|
| **Expert Advisor (EA)** | Full/partial trading automation | `OnTick()` | `/MQL5/Experts/` |
| **Indicator** | Visual data arrays on chart | `OnCalculate()` | `/MQL5/Indicators/` |
| **Script** | One-shot actions | `OnStart()` | `/MQL5/Scripts/` |
| **Service** | Background tasks (no chart) | `OnStart()` loop | `/MQL5/Services/` |

### File Extensions
- `.mq5` — MQL5 source code
- `.ex5` — compiled executable
- `.mqh` — header/include file

---

## Program Structure

### Expert Advisor skeleton

```mql5
#property copyright "Author"
#property version   "1.00"
#property strict

// Input parameters
input double LotSize = 0.1;
input int    MagicNumber = 12345;

// Global variables
CTrade trade;  // requires <Trade/Trade.mqh>

//+------------------------------------------------------------------+
int OnInit()
  {
   trade.SetExpertMagicNumber(MagicNumber);
   return(INIT_SUCCEEDED);
  }

void OnDeinit(const int reason)
  {
   // cleanup
  }

void OnTick()
  {
   // main trading logic fires on every new tick
  }
```

### Indicator skeleton

```mql5
#property indicator_chart_window
#property indicator_buffers 1
#property indicator_plots   1
#property indicator_label1  "MyLine"
#property indicator_type1   DRAW_LINE
#property indicator_color1  clrBlue
#property indicator_width1  2

double Buffer1[];

int OnInit()
  {
   SetIndexBuffer(0, Buffer1, INDICATOR_DATA);
   return(INIT_SUCCEEDED);
  }

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
   int start = (prev_calculated == 0) ? 1 : prev_calculated - 1;
   for(int i = start; i < rates_total; i++)
      Buffer1[i] = close[i];  // example
   return(rates_total);
  }
```

---

## Key Events Reference

| Event | Called When | Program Types |
|-------|------------|---------------|
| `OnInit()` | Program loads | All |
| `OnDeinit(reason)` | Program unloads | All |
| `OnTick()` | New tick arrives | EA |
| `OnCalculate(...)` | New tick (indicator) | Indicator |
| `OnTimer()` | EventSetTimer() fires | EA, Indicator |
| `OnTrade()` | Trade state changed | EA |
| `OnTradeTransaction(trans, req, res)` | Transaction occurred | EA |
| `OnChartEvent(id, lparam, dparam, sparam)` | Chart event | EA, Indicator |
| `OnStart()` | Program start | Script, Service |
| `OnTesterInit()` / `OnTester()` | Strategy tester | EA |

---

## Trading API — Core Functions

### Sending Orders (Modern CTrade approach — preferred)

```mql5
#include <Trade/Trade.mqh>
CTrade trade;

// Buy at market
trade.Buy(0.1, _Symbol, 0, stopLoss, takeProfit, "comment");

// Sell at market
trade.Sell(0.1, _Symbol, 0, stopLoss, takeProfit, "comment");

// Pending orders
trade.BuyLimit(0.1, price, _Symbol, sl, tp);
trade.SellStop(0.1, price, _Symbol, sl, tp);

// Close position by ticket
trade.PositionClose(ticket);
```

### Raw OrderSend (lower-level, required for full control)

```mql5
MqlTradeRequest request = {};
MqlTradeResult  result  = {};

request.action   = TRADE_ACTION_DEAL;
request.symbol   = _Symbol;
request.volume   = 0.1;
request.type     = ORDER_TYPE_BUY;
request.price    = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
request.sl       = 0;
request.tp       = 0;
request.deviation = 10;
request.magic    = 12345;
request.comment  = "EA order";
request.type_filling = ORDER_FILLING_IOC;

if(!OrderSend(request, result))
   Print("OrderSend error: ", GetLastError(), " retcode: ", result.retcode);
```

### Always validate before sending

```mql5
MqlTradeCheckResult checkResult = {};
if(!OrderCheck(request, checkResult))
  {
   Print("Check failed: ", checkResult.comment);
   return;
  }
```

---

## Position Management

```mql5
// Check if position exists
if(PositionSelect(_Symbol))
  {
   double vol    = PositionGetDouble(POSITION_VOLUME);
   double sl     = PositionGetDouble(POSITION_SL);
   double profit = PositionGetDouble(POSITION_PROFIT);
   ENUM_POSITION_TYPE type = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
  }

// Iterate all positions
for(int i = PositionsTotal()-1; i >= 0; i--)
  {
   ulong ticket = PositionGetTicket(i);
   if(PositionSelectByTicket(ticket))
     {
      if(PositionGetString(POSITION_SYMBOL) == _Symbol &&
         PositionGetInteger(POSITION_MAGIC) == MagicNumber)
        {
         // process this position
        }
     }
  }
```

---

## Getting Price Data

```mql5
// Current prices
double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
double point = SymbolInfoDouble(_Symbol, SYMBOL_POINT);
int digits = (int)SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);

// OHLCV arrays (most recent = index 0 by default with ArraySetAsSeries)
MqlRates rates[];
ArraySetAsSeries(rates, true);
int copied = CopyRates(_Symbol, PERIOD_H1, 0, 100, rates);
if(copied > 0)
  {
   double lastClose = rates[0].close;
   double prevHigh  = rates[1].high;
  }

// Copy individual arrays
double closeArr[];
ArraySetAsSeries(closeArr, true);
CopyClose(_Symbol, PERIOD_CURRENT, 0, 100, closeArr);
```

---

## Using Built-in Indicators

```mql5
// Create handle once in OnInit
int maHandle = iMA(_Symbol, PERIOD_CURRENT, 14, 0, MODE_EMA, PRICE_CLOSE);
int rsiHandle = iRSI(_Symbol, PERIOD_CURRENT, 14, PRICE_CLOSE);

// Read values in OnTick
double maVal[];
ArraySetAsSeries(maVal, true);
if(CopyBuffer(maHandle, 0, 0, 3, maVal) < 3) return;

double maValue = maVal[0];   // current bar
double maPrev  = maVal[1];   // previous bar
```

---

## Account & Symbol Info

```mql5
// Account
double balance  = AccountInfoDouble(ACCOUNT_BALANCE);
double equity   = AccountInfoDouble(ACCOUNT_EQUITY);
double freeMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
string currency = AccountInfoString(ACCOUNT_CURRENCY);
bool   isHedging = (ENUM_ACCOUNT_MARGIN_MODE)AccountInfoInteger(ACCOUNT_MARGIN_MODE)
                    == ACCOUNT_MARGIN_MODE_RETAIL_HEDGING;

// Symbol
double contractSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_CONTRACT_SIZE);
double minLot  = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
double maxLot  = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
double lotStep = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);

// Normalize lot size
double rawLot = equity * 0.01 / 100;
double lot = MathFloor(rawLot / lotStep) * lotStep;
lot = MathMax(minLot, MathMin(maxLot, lot));
lot = NormalizeDouble(lot, 2);
```

---

## Common Patterns

### Trailing Stop

```mql5
void TrailingStop(double trailPoints)
  {
   double point = SymbolInfoDouble(_Symbol, SYMBOL_POINT);
   for(int i = PositionsTotal()-1; i >= 0; i--)
     {
      if(!PositionGetTicket(i)) continue;
      if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC)  != MagicNumber) continue;

      double currentSL = PositionGetDouble(POSITION_SL);
      double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      ulong  ticket    = PositionGetInteger(POSITION_TICKET);

      if((ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY)
        {
         double newSL = SymbolInfoDouble(_Symbol, SYMBOL_BID) - trailPoints * point;
         if(newSL > currentSL + point)
            trade.PositionModify(ticket, newSL, PositionGetDouble(POSITION_TP));
        }
     }
  }
```

### OnTradeTransaction for execution confirmation

```mql5
void OnTradeTransaction(const MqlTradeTransaction &trans,
                        const MqlTradeRequest     &request,
                        const MqlTradeResult      &result)
  {
   if(trans.type == TRADE_TRANSACTION_DEAL_ADD)
     {
      if(trans.deal_type == DEAL_TYPE_BUY || trans.deal_type == DEAL_TYPE_SELL)
         Print("Deal executed: ticket=", trans.deal, " profit=", trans.profit);
     }
  }
```

---

## Input Parameters Best Practices

```mql5
// Group inputs with sinput or enum for readability
input group "=== Risk Settings ==="
input double RiskPercent  = 1.0;   // Risk % of balance per trade
input double MaxLotSize   = 5.0;   // Maximum lot size

input group "=== Strategy Settings ==="
input int    FastMA = 8;           // Fast MA period
input int    SlowMA = 21;          // Slow MA period
input ENUM_TIMEFRAMES TF = PERIOD_H1; // Timeframe

// sinput — not optimizable in strategy tester
sinput int MagicNumber = 99999;
```

---

## Error Handling

```mql5
// Check return of every trading function
if(!trade.Buy(lot, _Symbol))
  {
   Print("Buy failed. Error: ", GetLastError(),
         " | Return code: ", trade.ResultRetcode(),
         " | Description: ", trade.ResultRetcodeDescription());
  }

// Common return codes
// 10009 TRADE_RETCODE_DONE        — request completed OK
// 10010 TRADE_RETCODE_DONE_PARTIAL— partially filled
// 10004 TRADE_RETCODE_REQUOTE     — requote, retry
// 10016 TRADE_RETCODE_INVALID_STOPS — bad SL/TP

// Retry on requote
for(int attempt = 0; attempt < 3; attempt++)
  {
   if(trade.Buy(lot, _Symbol)) break;
   if(trade.ResultRetcode() != TRADE_RETCODE_REQUOTE) break;
   Sleep(500);
  }
```

---

## Time & Scheduling

```mql5
// Timer-based logic
int OnInit()
  {
   EventSetTimer(60);  // fire OnTimer every 60 seconds
   return INIT_SUCCEEDED;
  }

void OnTimer()
  {
   // runs every 60s regardless of ticks
  }

void OnDeinit(const int reason)
  {
   EventKillTimer();
  }

// Trading session check
datetime serverTime = TimeCurrent();
MqlDateTime dt;
TimeToStruct(serverTime, dt);
bool isTradingHour = (dt.hour >= 8 && dt.hour < 20);
```

---

## Strategy Tester Compatibility

- Use `OnTesterInit()` / `OnTester()` / `OnTesterPass()` for optimization control
- Avoid `Sleep()` in EA logic — it blocks the tester
- Use `IsTesting()` and `IsOptimization()` to branch behaviour
- `TesterStatistics(STAT_PROFIT)` returns tester metrics in `OnTester()`
- Always test with `Every tick based on real ticks` for highest fidelity

```mql5
double OnTester()
  {
   double profit  = TesterStatistics(STAT_PROFIT);
   double dd      = TesterStatistics(STAT_EQUITY_DD_RELATIVE);
   double trades  = TesterStatistics(STAT_TRADES);
   if(trades < 10 || dd > 30) return 0;  // penalize bad results
   return profit / dd;  // custom optimization criterion
  }
```

---

## MQL5 vs C++ Key Differences

See `references/gotchas.md` for the full list. Most important:

1. **No raw pointers to stack objects** — use `new` / `delete` for dynamic objects
2. **No templates with non-type parameters** — limited template support
3. **`string` is a value type**, not a class — compare with `==` directly
4. **Arrays are 0-based but series are index-0 = newest** when `ArraySetAsSeries(arr, true)`
5. **`input` vs `sinput`** — sinput skips optimization in strategy tester
6. **No `std::` namespace** — use MQL5 equivalents (`ArraySort`, `MathMax`, etc.)
7. **Object deletion** — always `delete ptr;` for heap objects; never use after `OnDeinit`
8. **`NormalizeDouble()`** — always normalize prices and lots before using them

---

## Useful Standard Library Includes

```mql5
#include <Trade/Trade.mqh>          // CTrade — high-level trading
#include <Trade/PositionInfo.mqh>   // CPositionInfo
#include <Trade/OrderInfo.mqh>      // COrderInfo
#include <Trade/HistoryOrderInfo.mqh>
#include <Trade/DealInfo.mqh>
#include <Indicators/Trend.mqh>     // Moving averages etc.
#include <Arrays/ArrayDouble.mqh>   // Dynamic arrays
#include <Expert/Expert.mqh>        // CExpert framework (advanced)
```

---

## Code Quality Checklist

Before finishing any MQL5 code, verify:

- [ ] All indicator handles released in `OnDeinit` with `IndicatorRelease(handle)`
- [ ] Timer killed in `OnDeinit` with `EventKillTimer()` if used
- [ ] Lots normalized with `NormalizeDouble` and clamped to min/max/step
- [ ] Prices normalized with `NormalizeDouble(price, _Digits)`
- [ ] SL/TP levels account for minimum stop level: `SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL)`
- [ ] Magic number used to filter own positions/orders from others
- [ ] Loop over positions iterates **backwards** (`i = Total-1; i >= 0; i--`) to safely close while iterating
- [ ] `ArraySetAsSeries` set appropriately before `CopyBuffer` / `CopyRates`
- [ ] `IsTradeAllowed()` checked before sending orders
- [ ] Error codes logged with `GetLastError()` after failed calls

---

## Reference Files

- `references/patterns.md` — Extended code templates (multi-symbol EA, indicator with alerts, etc.)
- `references/gotchas.md` — Detailed MQL5 vs C++ differences and traps

**Official docs:** https://www.mql5.com/en/docs
**MQL5 Algo Book:** https://www.mql5.com/en/book
