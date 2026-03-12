# MQL5 Extended Code Patterns

Complete, reusable code patterns for common MQL5 scenarios.

---

## Pattern 1: Complete Moving Average Crossover EA

```mql5
#property copyright "MQL5 Skill Example"
#property version   "1.00"
#include <Trade/Trade.mqh>

input group "=== Strategy ==="
input int    FastPeriod = 9;
input int    SlowPeriod = 21;
input ENUM_MA_METHOD MAMethod = MODE_EMA;
input ENUM_APPLIED_PRICE MAPrice = PRICE_CLOSE;

input group "=== Risk ==="
input double LotSize      = 0.1;
input double StopLossPips = 50;
input double TakeProfitPips = 100;

input group "=== EA Settings ==="
sinput int    MagicNumber = 11111;

CTrade  trade;
int     fastHandle, slowHandle;
double  fastBuf[], slowBuf[];

int OnInit()
  {
   trade.SetExpertMagicNumber(MagicNumber);
   trade.SetDeviationInPoints(10);

   fastHandle = iMA(_Symbol, PERIOD_CURRENT, FastPeriod, 0, MAMethod, MAPrice);
   slowHandle = iMA(_Symbol, PERIOD_CURRENT, SlowPeriod, 0, MAMethod, MAPrice);

   if(fastHandle == INVALID_HANDLE || slowHandle == INVALID_HANDLE)
     { Print("Failed to create MA handles"); return INIT_FAILED; }

   ArraySetAsSeries(fastBuf, true);
   ArraySetAsSeries(slowBuf, true);
   return INIT_SUCCEEDED;
  }

void OnDeinit(const int reason)
  {
   IndicatorRelease(fastHandle);
   IndicatorRelease(slowHandle);
  }

void OnTick()
  {
   // Only act on new bar
   static datetime lastBar = 0;
   datetime currentBar = iTime(_Symbol, PERIOD_CURRENT, 0);
   if(currentBar == lastBar) return;
   lastBar = currentBar;

   if(CopyBuffer(fastHandle, 0, 0, 3, fastBuf) < 3) return;
   if(CopyBuffer(slowHandle, 0, 0, 3, slowBuf) < 3) return;

   bool bullCross = fastBuf[1] > slowBuf[1] && fastBuf[2] <= slowBuf[2];
   bool bearCross = fastBuf[1] < slowBuf[1] && fastBuf[2] >= slowBuf[2];

   int posCount = CountPositions();
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double point = SymbolInfoDouble(_Symbol, SYMBOL_POINT);

   if(bullCross && posCount == 0)
     {
      double sl = NormalizeDouble(ask - StopLossPips * 10 * point, _Digits);
      double tp = NormalizeDouble(ask + TakeProfitPips * 10 * point, _Digits);
      trade.Buy(LotSize, _Symbol, ask, sl, tp);
     }
   else if(bearCross && posCount == 0)
     {
      double sl = NormalizeDouble(bid + StopLossPips * 10 * point, _Digits);
      double tp = NormalizeDouble(bid - TakeProfitPips * 10 * point, _Digits);
      trade.Sell(LotSize, _Symbol, bid, sl, tp);
     }
  }

int CountPositions()
  {
   int count = 0;
   for(int i = PositionsTotal()-1; i >= 0; i--)
     {
      if(PositionGetTicket(i) &&
         PositionGetString(POSITION_SYMBOL) == _Symbol &&
         PositionGetInteger(POSITION_MAGIC) == MagicNumber)
         count++;
     }
   return count;
  }

double OnTester()
  {
   double profit = TesterStatistics(STAT_PROFIT);
   double dd     = TesterStatistics(STAT_EQUITY_DD_RELATIVE);
   double trades = TesterStatistics(STAT_TRADES);
   if(trades < 10 || dd > 40) return 0;
   return profit / MathMax(dd, 1);
  }
```

---

## Pattern 2: RSI Overbought/Oversold Indicator with Alerts

```mql5
#property indicator_separate_window
#property indicator_buffers 3
#property indicator_plots   1
#property indicator_label1  "RSI"
#property indicator_type1   DRAW_LINE
#property indicator_color1  clrDodgerBlue
#property indicator_width1  2

input int    RSIPeriod      = 14;
input double OverboughtLevel = 70;
input double OversoldLevel   = 30;
input bool   EnableAlerts    = true;

double rsiBuffer[];
double obLevel[], osLevel[];   // for reference lines

int rsiHandle;
datetime lastAlertTime = 0;

int OnInit()
  {
   SetIndexBuffer(0, rsiBuffer, INDICATOR_DATA);
   SetIndexBuffer(1, obLevel,   INDICATOR_CALCULATIONS);
   SetIndexBuffer(2, osLevel,   INDICATOR_CALCULATIONS);

   rsiHandle = iRSI(_Symbol, PERIOD_CURRENT, RSIPeriod, PRICE_CLOSE);
   if(rsiHandle == INVALID_HANDLE) return INIT_FAILED;

   IndicatorSetInteger(INDICATOR_DIGITS, 2);
   IndicatorSetDouble(INDICATOR_LEVELVALUE, 0, OverboughtLevel);
   IndicatorSetDouble(INDICATOR_LEVELVALUE, 1, OversoldLevel);
   IndicatorSetInteger(INDICATOR_LEVELS, 2);

   return INIT_SUCCEEDED;
  }

void OnDeinit(const int reason)
  {
   IndicatorRelease(rsiHandle);
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
   int toCopy = rates_total - prev_calculated + 1;
   if(CopyBuffer(rsiHandle, 0, 0, toCopy, rsiBuffer) <= 0) return prev_calculated;

   // Alert on cross into overbought/oversold (current vs previous)
   if(EnableAlerts && rates_total > 1 && time[rates_total-1] != lastAlertTime)
     {
      double cur  = rsiBuffer[rates_total-1];
      double prev = rsiBuffer[rates_total-2];
      if(prev < OverboughtLevel && cur >= OverboughtLevel)
        { Alert(_Symbol, " RSI entered overbought (", DoubleToString(cur,1), ")");
          lastAlertTime = time[rates_total-1]; }
      if(prev > OversoldLevel && cur <= OversoldLevel)
        { Alert(_Symbol, " RSI entered oversold (", DoubleToString(cur,1), ")");
          lastAlertTime = time[rates_total-1]; }
     }

   return rates_total;
  }
```

---

## Pattern 3: Risk-Based Lot Sizing Function

```mql5
// Returns lot size based on % risk of account balance
// stopDistancePoints: distance of stop loss in POINTS (not pips)
double CalcLotByRisk(double riskPercent, double stopDistancePoints, string sym = NULL)
  {
   if(sym == NULL) sym = _Symbol;

   double balance       = AccountInfoDouble(ACCOUNT_BALANCE);
   double riskAmount    = balance * riskPercent / 100.0;
   double tickSize      = SymbolInfoDouble(sym, SYMBOL_TRADE_TICK_SIZE);
   double tickValue     = SymbolInfoDouble(sym, SYMBOL_TRADE_TICK_VALUE);
   double contractSize  = SymbolInfoDouble(sym, SYMBOL_TRADE_CONTRACT_SIZE);
   double point         = SymbolInfoDouble(sym, SYMBOL_POINT);
   double minLot        = SymbolInfoDouble(sym, SYMBOL_VOLUME_MIN);
   double maxLot        = SymbolInfoDouble(sym, SYMBOL_VOLUME_MAX);
   double lotStep       = SymbolInfoDouble(sym, SYMBOL_VOLUME_STEP);

   if(tickValue == 0 || tickSize == 0 || stopDistancePoints == 0) return minLot;

   double stopInTicks   = stopDistancePoints * point / tickSize;
   double lot           = riskAmount / (stopInTicks * tickValue);

   lot = MathFloor(lot / lotStep) * lotStep;
   lot = MathMax(minLot, MathMin(maxLot, lot));
   return NormalizeDouble(lot, 2);
  }
```

---

## Pattern 4: Multi-Symbol EA (handles multiple instruments)

```mql5
#include <Trade/Trade.mqh>

input string Symbols     = "EURUSD,GBPUSD,USDJPY";
input int    MAPeriod    = 20;
sinput int   MagicNumber = 22222;

CTrade trade;
string g_symbols[];
int    g_handles[];

int OnInit()
  {
   trade.SetExpertMagicNumber(MagicNumber);
   int n = StringSplit(Symbols, ',', g_symbols);
   ArrayResize(g_handles, n);
   for(int i = 0; i < n; i++)
     {
      StringTrimLeft(g_symbols[i]);
      StringTrimRight(g_symbols[i]);
      g_handles[i] = iMA(g_symbols[i], PERIOD_CURRENT, MAPeriod, 0, MODE_SMA, PRICE_CLOSE);
      if(g_handles[i] == INVALID_HANDLE)
        { PrintFormat("Failed handle for %s", g_symbols[i]); return INIT_FAILED; }
     }
   return INIT_SUCCEEDED;
  }

void OnDeinit(const int reason)
  {
   for(int i = 0; i < ArraySize(g_handles); i++)
      IndicatorRelease(g_handles[i]);
  }

void OnTick()
  {
   for(int i = 0; i < ArraySize(g_symbols); i++)
      ProcessSymbol(g_symbols[i], g_handles[i]);
  }

void ProcessSymbol(string sym, int handle)
  {
   double buf[];
   ArraySetAsSeries(buf, true);
   if(CopyBuffer(handle, 0, 0, 2, buf) < 2) return;

   double ask = SymbolInfoDouble(sym, SYMBOL_ASK);
   double bid = SymbolInfoDouble(sym, SYMBOL_BID);

   bool hasPos = false;
   for(int i = PositionsTotal()-1; i >= 0; i--)
     {
      if(PositionGetTicket(i) &&
         PositionGetString(POSITION_SYMBOL) == sym &&
         PositionGetInteger(POSITION_MAGIC) == MagicNumber)
        { hasPos = true; break; }
     }

   if(!hasPos)
     {
      if(ask > buf[0])  // price above MA → buy
         trade.Buy(0.1, sym);
      else if(bid < buf[0])
         trade.Sell(0.1, sym);
     }
  }
```

---

## Pattern 5: Order History Analysis

```mql5
// Calculate statistics for EA's trades over a date range
void AnalyseHistory(datetime from, datetime to)
  {
   if(!HistorySelect(from, to)) return;

   int    totalDeals = HistoryDealsTotal();
   double totalProfit = 0;
   int    wins = 0, losses = 0;

   for(int i = 0; i < totalDeals; i++)
     {
      ulong ticket = HistoryDealGetTicket(i);
      if(ticket == 0) continue;
      if(HistoryDealGetInteger(ticket, DEAL_MAGIC) != MagicNumber) continue;
      if(HistoryDealGetInteger(ticket, DEAL_ENTRY) != DEAL_ENTRY_OUT) continue; // only close deals

      double profit = HistoryDealGetDouble(ticket, DEAL_PROFIT)
                    + HistoryDealGetDouble(ticket, DEAL_SWAP)
                    + HistoryDealGetDouble(ticket, DEAL_COMMISSION);
      totalProfit += profit;
      if(profit > 0) wins++;
      else if(profit < 0) losses++;
     }

   int total = wins + losses;
   double winRate = total > 0 ? (double)wins / total * 100 : 0;
   PrintFormat("Deals: %d | Profit: %.2f | Win rate: %.1f%%", total, totalProfit, winRate);
  }
```

---

## Pattern 6: Breakeven + Trailing Stop Manager

```mql5
// Call from OnTick to manage all EA positions
void ManagePositions(double breakevenPips, double trailPips)
  {
   double point = SymbolInfoDouble(_Symbol, SYMBOL_POINT);
   double be    = breakevenPips * 10 * point;
   double trail = trailPips * 10 * point;

   for(int i = PositionsTotal()-1; i >= 0; i--)
     {
      ulong ticket = PositionGetTicket(i);
      if(!ticket) continue;
      if(PositionGetString(POSITION_SYMBOL)  != _Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC)  != MagicNumber) continue;

      double open = PositionGetDouble(POSITION_PRICE_OPEN);
      double sl   = PositionGetDouble(POSITION_SL);
      double tp   = PositionGetDouble(POSITION_TP);
      ENUM_POSITION_TYPE type = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);

      if(type == POSITION_TYPE_BUY)
        {
         double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
         double newSL = sl;

         // Breakeven
         if(bid >= open + be && sl < open)
            newSL = NormalizeDouble(open + _Point, _Digits);

         // Trail
         double trailSL = NormalizeDouble(bid - trail, _Digits);
         if(trailSL > newSL)
            newSL = trailSL;

         if(newSL > sl + _Point)
            trade.PositionModify(ticket, newSL, tp);
        }
      else if(type == POSITION_TYPE_SELL)
        {
         double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
         double newSL = sl;

         if(ask <= open - be && (sl > open || sl == 0))
            newSL = NormalizeDouble(open - _Point, _Digits);

         double trailSL = NormalizeDouble(ask + trail, _Digits);
         if(sl == 0 || trailSL < newSL)
            newSL = trailSL;

         if(sl == 0 || newSL < sl - _Point)
            trade.PositionModify(ticket, newSL, tp);
        }
     }
  }
```

---

## Pattern 7: OnTradeTransaction-Based Order Tracking

```mql5
// Track when our EA's orders are filled and log details
void OnTradeTransaction(const MqlTradeTransaction &trans,
                        const MqlTradeRequest     &request,
                        const MqlTradeResult      &result)
  {
   // Only interested in deal additions (position open/close)
   if(trans.type != TRADE_TRANSACTION_DEAL_ADD) return;

   // Verify it belongs to our EA
   if(HistoryDealSelect(trans.deal))
     {
      if(HistoryDealGetInteger(trans.deal, DEAL_MAGIC) != MagicNumber) return;

      ENUM_DEAL_TYPE dealType = (ENUM_DEAL_TYPE)HistoryDealGetInteger(trans.deal, DEAL_TYPE);
      ENUM_DEAL_ENTRY entry   = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(trans.deal, DEAL_ENTRY);
      double price   = HistoryDealGetDouble(trans.deal, DEAL_PRICE);
      double volume  = HistoryDealGetDouble(trans.deal, DEAL_VOLUME);
      double profit  = HistoryDealGetDouble(trans.deal, DEAL_PROFIT);
      string symbol  = HistoryDealGetString(trans.deal, DEAL_SYMBOL);

      PrintFormat("[Deal %I64u] %s %s %s | price=%.5f vol=%.2f profit=%.2f",
                  trans.deal,
                  symbol,
                  EnumToString(dealType),
                  EnumToString(entry),
                  price, volume, profit);
     }
  }
```

---

## Pattern 8: Global Terminal Variables for Inter-EA Communication

```mql5
// EA 1: Writes signal
void WriteSignal(int signal)  // 1=buy, -1=sell, 0=flat
  {
   GlobalVariableSet("MySignal_EURUSD", signal);
  }

// EA 2: Reads signal
void ReadAndAct()
  {
   if(!GlobalVariableCheck("MySignal_EURUSD")) return;
   int signal = (int)GlobalVariableGet("MySignal_EURUSD");
   if(signal == 1)  { /* buy */ }
   if(signal == -1) { /* sell */ }
  }
```

---

## Pattern 9: Proper Tick Filter (Only Trade on New Bars)

```mql5
// In EA global scope
datetime g_lastBar = 0;

bool IsNewBar()
  {
   datetime currentBar = iTime(_Symbol, PERIOD_CURRENT, 0);
   if(currentBar != g_lastBar)
     { g_lastBar = currentBar; return true; }
   return false;
  }

void OnTick()
  {
   if(!IsNewBar()) return;  // only run logic once per bar
   // ... rest of strategy
  }
```

---

## Pattern 10: Pending Order with Expiry

```mql5
void PlaceBuyLimitWithExpiry(double price, double sl, double tp, int expiryMinutes)
  {
   MqlTradeRequest req = {};
   MqlTradeResult  res = {};

   req.action      = TRADE_ACTION_PENDING;
   req.symbol      = _Symbol;
   req.volume      = 0.1;
   req.type        = ORDER_TYPE_BUY_LIMIT;
   req.price       = NormalizeDouble(price, _Digits);
   req.sl          = NormalizeDouble(sl, _Digits);
   req.tp          = NormalizeDouble(tp, _Digits);
   req.expiration  = TimeCurrent() + expiryMinutes * 60;
   req.type_time   = ORDER_TIME_SPECIFIED;
   req.magic       = MagicNumber;
   req.deviation   = 10;
   req.comment     = "BuyLimit";

   if(!OrderSend(req, res))
      PrintFormat("Pending order failed: %d (%s)", res.retcode, res.comment);
   else
      PrintFormat("Pending order placed: ticket=%I64u", res.order);
  }
```
