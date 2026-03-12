# MQL5 vs C++ Gotchas & Traps

This file covers the differences between MQL5 and C++ that most commonly cause errors.

---

## 1. Strings

In C++ `std::string` is a class. In MQL5, `string` is a **built-in value type**.

```mql5
// CORRECT — compare strings with == directly
if(sym == "EURUSD") { ... }

// WRONG — StringCompare exists but == works fine
// No .c_str(), no .length() — use StringLen(s) instead
int len = StringLen(myStr);
string sub = StringSubstr(myStr, 0, 3);  // like substr
```

---

## 2. Arrays & Series Orientation

MQL5 arrays can be used in **two modes**:

- **Normal mode** (default): index 0 = oldest data
- **Series mode** (`ArraySetAsSeries(arr, true)`): index 0 = NEWEST data

Always set series mode explicitly when reading price data:

```mql5
double close[];
ArraySetAsSeries(close, true);
CopyClose(_Symbol, PERIOD_CURRENT, 0, 100, close);
// close[0] = current bar, close[1] = previous bar
```

Forgetting this causes your logic to read data backwards — a subtle bug.

---

## 3. No Pointer Arithmetic

MQL5 does not support raw pointer arithmetic like C++. You can use object pointers
with `new`/`delete` but cannot do `ptr++` or `ptr[n]` on them.

```mql5
// CORRECT
MyClass *obj = new MyClass();
obj.DoSomething();
delete obj;

// WRONG — no pointer arithmetic
// MyClass *arr = new MyClass[10];  // NO — use arrays instead
```

---

## 4. Templates Are Limited

MQL5 supports function templates but **not class templates with non-type parameters**.
Avoid complex template metaprogramming. The standard library uses templates for
collection classes but keep your own templates simple.

---

## 5. No Namespaces Like `std::`

MQL5 has no `std::` namespace. Use MQL5 equivalents:

| C++ | MQL5 |
|-----|------|
| `std::max(a, b)` | `MathMax(a, b)` |
| `std::min(a, b)` | `MathMin(a, b)` |
| `std::abs(x)` | `MathAbs(x)` |
| `std::sqrt(x)` | `MathSqrt(x)` |
| `std::sort(arr)` | `ArraySort(arr)` |
| `printf(...)` | `PrintFormat(...)` |
| `std::vector<double>` | `double arr[]; ArrayResize(arr, n);` |

---

## 6. NormalizeDouble is Mandatory

Never send raw floating-point prices or volumes. Always normalize:

```mql5
double price = NormalizeDouble(ask - 50 * _Point, _Digits);
double lot   = NormalizeDouble(rawLot, 2);  // usually 2 decimal places
```

Failing to normalize causes `TRADE_RETCODE_INVALID_VOLUME` or price errors.

---

## 7. Indicator Handles Must Be Released

Create indicator handles in `OnInit`, release in `OnDeinit`:

```mql5
int g_maHandle;

int OnInit()
  {
   g_maHandle = iMA(_Symbol, PERIOD_CURRENT, 14, 0, MODE_EMA, PRICE_CLOSE);
   if(g_maHandle == INVALID_HANDLE)
     { Print("iMA failed"); return INIT_FAILED; }
   return INIT_SUCCEEDED;
  }

void OnDeinit(const int reason)
  {
   IndicatorRelease(g_maHandle);
  }
```

---

## 8. The `#property strict` Difference

In MQL4, `#property strict` enables strict mode. In **MQL5 it is always strict** — the
directive exists but has no additional effect. Don't confuse MQL4 and MQL5 code patterns.

---

## 9. Position Types in Hedging vs Netting

MetaTrader 5 supports two account modes:

- **Netting** (most common for futures/stocks): one position per symbol; Buy + Sell = net offset
- **Hedging** (forex retail): multiple positions per symbol in both directions

Check the account mode and code accordingly:

```mql5
bool isHedging = ((ENUM_ACCOUNT_MARGIN_MODE)AccountInfoInteger(ACCOUNT_MARGIN_MODE)
                  == ACCOUNT_MARGIN_MODE_RETAIL_HEDGING);
```

---

## 10. Loop Direction When Closing Positions/Orders

Always iterate **backwards** when you may close/delete items in the loop body:

```mql5
// CORRECT — iterate backward to avoid skipping items
for(int i = PositionsTotal()-1; i >= 0; i--)
  {
   ulong ticket = PositionGetTicket(i);
   trade.PositionClose(ticket);
  }

// WRONG — closing while going forward can skip items
for(int i = 0; i < PositionsTotal(); i++)  // BUG if closing
  { ... }
```

---

## 11. `Sleep()` in EAs

`Sleep(ms)` pauses execution. This is **fine in Scripts and Services** but should be
avoided in Expert Advisors and Indicators because:

- It blocks the terminal's event queue
- The strategy tester cannot simulate it properly
- It causes the EA to miss ticks

Use `EventSetTimer()` + `OnTimer()` for time-delayed actions in EAs instead.

---

## 12. `input` vs `sinput` vs `extern`

- `input` — visible in EA properties, optimizable in strategy tester
- `sinput` — visible in properties, **NOT optimizable** (good for magic numbers, symbols)
- `extern` — MQL4 legacy; use `input` in MQL5

```mql5
sinput int MagicNumber = 12345;  // never optimize this
input  int FastPeriod  = 8;      // optimize this
```

---

## 13. Object Lifetime & `OnDeinit`

After `OnDeinit` is called, MQL5 destroys all static objects. If you use `new`/`delete`,
ensure you `delete` all heap objects inside `OnDeinit`. Accessing deleted objects causes
access violation errors.

---

## 14. `Print()` vs `PrintFormat()`

```mql5
Print("Value: ", someDouble);             // concatenation, fine for simple messages
PrintFormat("Value: %.5f", someDouble);   // like printf, better for formatted output
```

---

## 15. Minimum Stop Level

Always check the minimum stop distance before setting SL/TP:

```mql5
int stopLevel = (int)SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL);
double minStop = stopLevel * _Point;

double sl = bid - MathMax(desiredSLDistance, minStop + _Point);
```

Ignoring this causes `TRADE_RETCODE_INVALID_STOPS`.

---

## 16. CopyBuffer Index 0 is the CURRENT bar

When using `CopyBuffer` with `ArraySetAsSeries(buf, true)`:
- index 0 = current (possibly unfinished) bar
- index 1 = last completed bar

For strategies that trade only on closed bars, use index 1 as the signal bar.

---

## 17. `iCustom` and Custom Indicator Parameters

When calling a custom indicator:

```mql5
// All inputs must be passed in correct order after the timeframe
int handle = iCustom(_Symbol, PERIOD_CURRENT, "MyIndicator",
                     param1, param2);  // must match indicator's input order exactly
```

---

## 18. Thread Safety

MQL5 runs each program in its own thread. Do NOT share global data between two EA
instances on different charts via static variables — use files, global terminal variables
(`GlobalVariableSet`/`GlobalVariableGet`), or named pipes for inter-program communication.
