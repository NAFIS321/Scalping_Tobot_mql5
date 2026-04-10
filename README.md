# Scalping_Tobot_mql5
main source code 

#property copyright "Automation fx"
#property link      "https://automationfx.in"
#property version   "1.06"
#property strict

#include <Trade\Trade.mqh>
#include <Trade\AccountInfo.mqh>
//#include <LotSize_Managment.mqh>
//#include <Time Filter.mqh>
#include <Trade\PositionInfo.mqh>

CTrade trade;
CPositionInfo posinfo;
CPositionInfo pos;

enum LotMode
{
   FixedLot  = 0,    // Use fixed lot size
   RiskLot   = 1,    // Use equity % risk with SL
   EquityLot = 2,    // Lot = Equity
};
enum ENUM_HOUR { h00=0, h01=1, h02=2, h03=3, h04=4, h05=5, h06=6, h07=7,
   h08=8, h09=9, h10=10, h11=11, h12=12, h13=13, h14=14, h15=15, h16=16,
   h17=17, h18=18, h19=19, h20=20, h21=21, h22=22, h23=23 };
enum ENUM_MINUTE {
   m00 = 0,  m05 = 5,  m10 = 10,  m15 = 15,  m20 = 20,  m25 = 25,
   m30 = 30, m35 = 35, m40 = 40,  m45 = 45,  m50 = 50,  m55 = 55
};

//--- EA Inputs
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
// EA Inputs
//+------------------------------------------------------------------+

input string   A_1          = "=============== Lot & Risk Settings ===============";
input LotMode  lotMode      = RiskLot;        // FixedLot / RiskLot
input double   lotValue     = 1.0;            // Lot Size or % Risk
input int      Stoploss     = 200;            // Stop Loss in points
input int      distance     = 100;            // Distance between orders in points
input ulong    magic_No     = 12345;          // Unique Magic Number

//--- Trailing Stop Settings
input string   A2              = "=============== Trailing Stop Settings ===============";
input bool     UseTraling_Stop = true;
input double   TrailStartPips  = 50;          // Start trailing after X pips profit
input double   TrailStepPips   = 100;         // Move SL every X pips profit
input double   TrailSLPips     = 100;         // SL always X pips behind current price

input string   A_3             = "=============== Trading Hours Settings ===============";
input bool     UseTradingHours = false;       // Limit trading hours
input ENUM_HOUR    TradingHourStart  = h15;
input ENUM_MINUTE  TradingMinStart   = m00;
input ENUM_HOUR    TradingHourEnd    = h18;
input ENUM_MINUTE  TradingMinEnd     = m30;

input string   A4          = "=============== Indicator Settings ===============";
input int      FASTMA      = 3;
input int      SLOWMA      = 8;
input int      ADX_Period  = 14;
input int      ADXstrenth  = 25;

input string   A5          = "=============== Day_Target ===============";

input double   DailyProfitTarget = 200.0;    // Target in account currency

//--- veriables
int  adxHandle;
int  handle1;
int  handle2;
bool   dailyTargetHit  = false;
datetime dayStartTime;
double profitClosed    = 0.0;

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
{
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   dt.hour = 0; dt.min = 0; dt.sec = 0;

   adxHandle  = iADX(_Symbol, PERIOD_CURRENT, ADX_Period);
   handle1    = iMA(_Symbol, PERIOD_CURRENT, FASTMA, 0, MODE_SMA, PRICE_CLOSE);
   handle2    = iMA(_Symbol, PERIOD_CURRENT, SLOWMA, 0, MODE_SMA, PRICE_CLOSE);
   profitClosed = CalcDailyClosedProfit();
   ChartSetInteger(0,CHART_SHOW_GRID,false);
   TesterHideIndicators(true);

   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{

}

//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
{
   trade.SetExpertMagicNumber(magic_No);
   //if (!IsNewBar()) return;
   // Restrict trading to within specified hours
   // 1️⃣ Reset daily counters if new broker day
   CheckAndResetDailyCounters();

   // 1️⃣ Skip if daily target already hit
   if (dailyTargetHit)
   {
      Comment("🚫 Daily target reached, trading paused until tomorrow.");
      return;
   }

   // 3️⃣ Check if daily target hit now
   if (CheckDailyTargetAndLock())
      return;

   if (UseTradingHours && !IsCurrentTimeInInterval(TradingHourStart, TradingMinStart, TradingHourEnd, TradingMinEnd))
      return;

   if (UseTraling_Stop)
   {
      StepWiseTrailingStop();
      Mirror_SL_to_OppositePending();
   }

   // 5️⃣ Get ADX value & display
   double adxValue  = GetADX(0);
   string adxMessage = " 📊 ADX : " + DoubleToString(adxValue, 2) +
                        "  Required > " + DoubleToString(ADXstrenth, 2);
   adxMessage += (adxValue <= ADXstrenth) ? "  ❌ Too Weak" : "  ✅ Strong";
   Comment(adxMessage);

   int Buyorder   = CountMarketOrders(ORDER_TYPE_BUY);
   int Sellorder  = CountMarketOrders(ORDER_TYPE_SELL);
   int buystop    = CountPendingOrders(ORDER_TYPE_BUY_STOP);
   int sellstop   = CountPendingOrders(ORDER_TYPE_SELL_STOP);

   double buyprice  = SymbolInfoDouble(_Symbol, SYMBOL_ASK) + distance * _Point;
   double sellprice = SymbolInfoDouble(_Symbol, SYMBOL_BID) - distance * _Point;

   double fastma = GetFastMA(FASTMA, 0);
   double slowma = GetSlowMA(SLOWMA, 0);

   // 9️⃣ Open orders if conditions met
   if (Buyorder == 0 && Sellorder == 0 && buystop == 0 && sellstop == 0 && adxValue > ADXstrenth)
   {
      if (fastma > slowma)
         Open_Buystop(buyprice);
      else if (fastma < slowma)
         Open_Sellstop(sellprice);
   }

   // 🔄 Modify pending orders
   double Ask        = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double Bid        = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double orderPrice = OrderGetDouble(ORDER_PRICE_OPEN);

   if (Buyorder == 0 && Sellorder == 0 && sellstop == 0 && Ask < orderPrice)
      modifybuystop();

   if (Buyorder == 0 && Sellorder == 0 && buystop == 0 && Bid > orderPrice)
      modifysellstop();

   double profitOpen = CalcOpenProfitByMagic();
   double profitDay  = profitOpen + profitClosed;

   // --- Show daily profit info ---
   Comment(
      "\nMagic: ", magic_No,
      "\nOpen Profit: ",   DoubleToString(profitOpen, 2), " ", AccountInfoString(ACCOUNT_CURRENCY),
      "\nClosed Profit: ", DoubleToString(profitClosed, 2), " ", AccountInfoString(ACCOUNT_CURRENCY),
      "\nDay Profit: ",    DoubleToString(profitDay, 2), " ", AccountInfoString(ACCOUNT_CURRENCY),
      "\nTarget Profit: ", DoubleToString(DailyProfitTarget, 2), " ", AccountInfoString(ACCOUNT_CURRENCY)
   );
}

//+------------------------------------------------------------------+
void Open_Buystop(double Buy_price)
{
   double sl  = Buy_price - Stoploss * _Point;
   // double tp = Buy_price + Takeprofit * _Point;
   double lot = GetLotSize(lotMode, lotValue, Stoploss);

   if (!trade.BuyStop(lot, Buy_price, _Symbol, sl, 0, ORDER_TIME_DAY, 0, "buystop"))
      Print("❌ Buystop failed: ", GetLastError());
}

void Open_Sellstop(double Sell_price)
{
   double sl  = Sell_price + Stoploss * _Point;
   //double tp = Sell_price - Takeprofit * _Point;
   double lot = GetLotSize(lotMode, lotValue, Stoploss);

   if (!trade.SellStop(lot, Sell_price, _Symbol, sl, 0, ORDER_TIME_DAY, 0, "sellstop"))
      Print("❌ Sellstop failed: ", GetLastError());
}

//+------------------------------------------------------------------+
int CountMarketOrders(int type)
{
   int count = 0;
   for (int i = 0; i < PositionsTotal(); i++)
   {
      if (PositionGetSymbol(i) == _Symbol &&
          PositionGetInteger(POSITION_TYPE)  == type &&
          PositionGetInteger(POSITION_MAGIC) == (long)magic_No)
         count++;
   }
   return count;
}

int CountPendingOrders(int type)
{
   int count = 0;
   for (int i = 0; i < OrdersTotal(); i++)
   {
      if (OrderGetTicket(i) &&
          OrderGetString(ORDER_SYMBOL)  == _Symbol &&
          OrderGetInteger(ORDER_TYPE)   == type &&
          OrderGetInteger(ORDER_MAGIC)  == (long)magic_No)
         count++;
   }
   return count;
}

//+------------------------------------------------------------------+
void StepWiseTrailingStop()
{
   int total = PositionsTotal();
   for (int i = total - 1; i >= 0; i--)
   {
      if (!posinfo.SelectByIndex(i)) continue;
      if (posinfo.Symbol() != _Symbol || posinfo.Magic() != (long)magic_No) continue;

      bool   isBuy        = (posinfo.PositionType() == POSITION_TYPE_BUY);
      double entryPrice   = posinfo.PriceOpen();
      double currentPrice = isBuy ? SymbolInfoDouble(_Symbol, SYMBOL_BID)
                                  : SymbolInfoDouble(_Symbol, SYMBOL_ASK);

      double profitPips = isBuy ? (currentPrice - entryPrice) / _Point
                                : (entryPrice - currentPrice) / _Point;

      // --- start trailing only after TrailStartPips
      if (profitPips < TrailStartPips) continue;

      // --- calculate desired stop
      double desiredSL = isBuy ? currentPrice - TrailSLPips * _Point
                               : currentPrice + TrailSLPips * _Point;

      double currentSL = posinfo.StopLoss();

      // --- Ensure stop only moves in the favorable direction
      if (isBuy)
      {
         if (desiredSL <= currentSL) continue; // don't move backwards
      }
      else
      {
         if (desiredSL >= currentSL && currentSL != 0.0) continue; // don't move backwards
      }

      // --- Check if move is large enough (stepwise)
      double slMoveDistance = isBuy ? (desiredSL - currentSL) / _Point
                                    : (currentSL - desiredSL) / _Point;

      if (currentSL == 0.0 || slMoveDistance >= TrailStepPips)
      {
         desiredSL = NormalizeDouble(desiredSL, _Digits);

         // --- check broker stop level
         double stopLevel = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * _Point;
         if (isBuy  && (currentPrice - desiredSL < stopLevel)) continue;
         if (!isBuy && (currentPrice - desiredSL < stopLevel)) continue;

         // --- try modify
         if (!trade.PositionModify(posinfo.Ticket(), desiredSL, posinfo.TakeProfit()))
         {
            Print("❌ Failed to trail ticket #", posinfo.Ticket(),
                  "  Error: ", GetLastError());
         }
         else
         {
         Print("✅ Trailed ticket #", posinfo.Ticket(),
                   "  New SL = ", DoubleToString(desiredSL, _Digits));
          }
       }
    }
}

//+------------------------------------------------------------------+

void Mirror_SL_to_OppositePending()
{
   // BUY position active → update/create SELL STOP
   if (CountMarketOrders(ORDER_TYPE_BUY) > 0)
   {
      double buy_sl = 0, lot = GetLotSize(lotMode, lotValue, Stoploss);
      for (int i = PositionsTotal() - 1; i >= 0; i--)
         if (posinfo.SelectByIndex(i) &&
             posinfo.Symbol() == _Symbol &&
             posinfo.PositionType() == POSITION_TYPE_BUY &&
             posinfo.Magic() == (long)magic_No)
         {
            buy_sl = posinfo.StopLoss(); break;
         }

      if (buy_sl == 0) return;

      bool found = false;
      for (int i = OrdersTotal() - 1; i >= 0; i--)
      {
         if (!OrderGetTicket(i)) continue;
         if (OrderGetString(ORDER_SYMBOL) == _Symbol &&
             OrderGetInteger(ORDER_TYPE) == ORDER_TYPE_SELL_STOP &&
             OrderGetInteger(ORDER_MAGIC) == (long)magic_No)
         {
            found = true;
            double oldPrice = OrderGetDouble(ORDER_PRICE_OPEN);
            if (MathAbs(oldPrice - buy_sl) > (_Point * 5))
            {
               double sl = buy_sl + Stoploss * _Point;
               //double tp = buy_sl - Takeprofit * _Point;
               ModifyPendingOrderToPrice(OrderGetTicket(i), buy_sl, sl, 0);
            }
         }
      }

      if (!found)
      {
         double sl = buy_sl + Stoploss * _Point;
         // double tp = buy_sl - Takeprofit * _Point;
         trade.SellStop(lot, buy_sl, _Symbol, sl, 0, ORDER_TIME_DAY, 0, "mirror sellstop");
      }
   }

   // SELL position active → update/create BUY STOP
   if (CountMarketOrders(ORDER_TYPE_SELL) > 0)
   {
      double sell_sl = 0, lot = GetLotSize(lotMode, lotValue, Stoploss);
      for (int i = PositionsTotal() - 1; i >= 0; i--)
         if (posinfo.SelectByIndex(i) &&
             posinfo.Symbol() == _Symbol &&
             posinfo.PositionType() == POSITION_TYPE_SELL &&
             posinfo.Magic() == (long)magic_No)
         {
            sell_sl = posinfo.StopLoss(); break;
         }

      if (sell_sl == 0) return;

      bool found = false;
      for (int i = OrdersTotal() - 1; i >= 0; i--)
      {
         if (!OrderGetTicket(i)) continue;
         if (OrderGetString(ORDER_SYMBOL) == _Symbol &&
             OrderGetInteger(ORDER_TYPE) == ORDER_TYPE_BUY_STOP &&
             OrderGetInteger(ORDER_MAGIC) == (long)magic_No)
         {
            found = true;
            double oldPrice = OrderGetDouble(ORDER_PRICE_OPEN);
            if (MathAbs(oldPrice - sell_sl) > (_Point * 5))
            {
               double sl = sell_sl - Stoploss * _Point;
               //double tp = sell_sl + Takeprofit * _Point;
               ModifyPendingOrderToPrice(OrderGetTicket(i), sell_sl, sl, 0);
            }
         }
      }

      if (!found)
      {
         double sl = sell_sl - Stoploss * _Point;
         // double tp = sell_sl + Takeprofit * _Point;
         trade.BuyStop(lot, sell_sl, _Symbol, sl, 0, ORDER_TIME_DAY, 0, "mirror buystop");
      }
   }
}

//+------------------------------------------------------------------+
bool ModifyPendingOrderToPrice(ulong ticket, double newPrice, double sl, double tp)
{
   MqlTradeRequest request;
   MqlTradeResult  result;
   ZeroMemory(request);
   ZeroMemory(result);

   request.action = TRADE_ACTION_MODIFY;
   request.order  = ticket;
   request.price  = NormalizeDouble(newPrice, _Digits);
   request.sl     = NormalizeDouble(sl, _Digits);
   request.tp     = NormalizeDouble(tp, _Digits);
   request.symbol = _Symbol;

   if (!OrderSend(request, result) || result.retcode != TRADE_RETCODE_DONE)
   {
      Print("❌ Modify failed: ", result.retcode);
      return false;
   }
   return true;
}

// --- Get Fast MA
//+------------------------------------------------------------------+
double GetFastMA(int period, int shift)
{
   double value[];
   if (CopyBuffer(handle1, 0, shift, 1, value) > 0)
      return value[0];
   return 0;
}

// --- Get Slow MA
//+------------------------------------------------------------------+
double GetSlowMA(int period, int shift)
{
   double value[];
   if (CopyBuffer(handle2, 0, shift, 1, value) > 0)
      return value[0];
   return 0;
}

// --- Get ADX Value
//+------------------------------------------------------------------+
double GetADX(int shift)
{
   double adxBuffer[];
   if (CopyBuffer(adxHandle, 0, shift, 1, adxBuffer) > 0)
      return adxBuffer[0];

   Print("⚠ Failed to get ADX value at shift ", shift);
   return 0;
}

void modifybuystop()
{
   double predefinedDistance = distance * _Point;
   double stopLevel          = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * _Point;
   double ask                = SymbolInfoDouble(_Symbol, SYMBOL_ASK);

   for (int i = 0; i < OrdersTotal(); i++)
   {
      ulong ticket = OrderGetTicket(i);
      if (OrderSelect(ticket))
      {
         if (OrderGetInteger(ORDER_MAGIC) == magic_No &&
             OrderGetString(ORDER_SYMBOL) == _Symbol &&
             OrderGetInteger(ORDER_TYPE) == ORDER_TYPE_BUY_STOP)
         {
            double currentPrice = OrderGetDouble(ORDER_PRICE_OPEN);
            double newPrice     = ask + predefinedDistance;

            // Move down only
            if (newPrice < currentPrice && (newPrice - ask) >= stopLevel)
            {
               double newSL = newPrice - Stoploss * _Point;

               if (!trade.OrderModify(ticket, newPrice, newSL, 0.0, ORDER_TIME_GTC, 0))
                  Print("❌ Failed to modify Buy Stop. Error: ", GetLastError());
               else
                  Print("✅ Buy Stop moved down to ", DoubleToString(newPrice, _Digits));
            }
         }
      }
   }
}

// --- Modify Sell Stop Orders (move up with price)
//+------------------------------------------------------------------+
void modifysellstop()
{
   double predefinedDistance = distance * _Point;
   double stopLevel          = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * _Point;
   double bid                = SymbolInfoDouble(_Symbol, SYMBOL_BID);

   for (int i = 0; i < OrdersTotal(); i++)
   {
      ulong ticket = OrderGetTicket(i);
      if (OrderSelect(ticket))
      {
         if (OrderGetInteger(ORDER_MAGIC) == magic_No &&
             OrderGetString(ORDER_SYMBOL) == _Symbol &&
             OrderGetInteger(ORDER_TYPE) == ORDER_TYPE_SELL_STOP)
         {
            double currentPrice = OrderGetDouble(ORDER_PRICE_OPEN);
            double newPrice     = bid - predefinedDistance;

            // Move up only
            if (newPrice > currentPrice && (bid - newPrice) >= stopLevel)
            {
               double newSL = newPrice + Stoploss * _Point;

               if (!trade.OrderModify(ticket, newPrice, newSL, 0.0, ORDER_TIME_GTC, 0))
                  Print("❌ Failed to modify Sell Stop. Error: ", GetLastError());
               else
                  Print("✅ Sell Stop moved up to ", DoubleToString(newPrice, _Digits));
            }
         }
      }
   }
}

//+------------------------------------------------------------------+
// Calculate today's closed profit (Magic Number filtered)           |
//+------------------------------------------------------------------+
double CalcDailyClosedProfit()
{
   double profit = 0.0;

   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   dt.hour = 0;
   dt.min  = 0;
   dt.sec  = 0;

   datetime timeDayStart = StructToTime(dt);

   // Select only today's deals
   HistorySelect(timeDayStart, TimeCurrent() + 100);

   for (int i = HistoryDealsTotal() - 1; i >= 0; i--)
   {
      ulong dealTicket = HistoryDealGetTicket(i);
      int   entry      = (int)HistoryDealGetInteger(dealTicket, DEAL_ENTRY);
      long  magic      = HistoryDealGetInteger(dealTicket, DEAL_MAGIC);

      // Only count this EA's closed trades
      if (entry == DEAL_ENTRY_OUT && magic == magic_No)
      {
         double dealProfit     = HistoryDealGetDouble(dealTicket, DEAL_PROFIT);
         double dealCommission = HistoryDealGetDouble(dealTicket, DEAL_COMMISSION);
         profit += (dealProfit + dealCommission);
      }
   }
   return profit;
}

//+------------------------------------------------------------------+
// Calculate open profit for this Magic Number                       |
//+------------------------------------------------------------------+
double CalcOpenProfitByMagic()
{
   double profit = 0.0;
   for (int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if (pos.SelectByIndex(i)) // Select position by index
      {
         if (PositionGetInteger(POSITION_MAGIC) == magic_No)
            profit += PositionGetDouble(POSITION_PROFIT);
      }
   }
   return profit;
}

//+------------------------------------------------------------------+
// OnTradeTransaction - Update closed profit on deal add             |
//+------------------------------------------------------------------+
void OnTradeTransaction(
   const MqlTradeTransaction &trans,
   const MqlTradeRequest     &request,
   const MqlTradeResult      &result
)
{
   if (trans.type == TRADE_TRANSACTION_DEAL_ADD)
   {
      profitClosed = CalcDailyClosedProfit();
   }
}

//---- Reset daily counters at broker midnight
void CheckAndResetDailyCounters()
{
   MqlDateTime now;
   TimeToStruct(TimeCurrent(), now);
   now.hour = 0; now.min = 0; now.sec = 0;
   datetime todayMidnight = StructToTime(now);

   if (dayStartTime != todayMidnight)
   {
      dayStartTime    = todayMidnight;
      dailyTargetHit  = false;
      profitClosed    = CalcDailyClosedProfit();
      Print("✅ Daily counters reset for new day: ", TimeToString(dayStartTime, TIME_DATE));
   }
}

//---- Close all positions and pending orders for this EA
void CloseAllOrdersByMagic(int magic)
{
   // Close all positions
   for (int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if (pos.SelectByIndex(i) && pos.Magic() == magic)
         trade.PositionClose(pos.Ticket());
   }

   // Close all pending orders
   for (int i = OrdersTotal() - 1; i >= 0; i--)
   {
      ulong ticket = OrderGetTicket(i);
      if (OrderSelect(ticket))
      {
         if (OrderGetInteger(ORDER_MAGIC) == magic &&
             OrderGetInteger(ORDER_TYPE) <= ORDER_TYPE_SELL_STOP)
         {
            trade.OrderDelete(ticket);
         }
      }
   }
   Print("✅ All positions & pending orders closed for magic: ", magic);
}

//---- Check daily target and handle lock
bool CheckDailyTargetAndLock()
{
   double profitOpen = CalcOpenProfitByMagic();
   double profitDay  = profitOpen + profitClosed;

   if (profitDay >= DailyProfitTarget)
   {
      dailyTargetHit = true;
      CloseAllOrdersByMagic(magic_No);
      Comment("✅ Daily target reached – trading paused until tomorrow.");
      return true; // Target hit
   }
   return false; // Target not hit
}

double GetLotSize(LotMode mode, double riskPercent, double slPoints)
{
   double lot    = 0.01; // default fallback
   double equity = AccountInfoDouble(ACCOUNT_EQUITY);
   double tickValue  = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize   = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   double minLot     = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double maxLot     = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double lotStep    = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);

   switch (mode)
   {
      case FixedLot:
         lot = riskPercent; // Here, riskPercent is interpreted as fixed lot size
         break;

      case RiskLot:
         if (tickValue > 0 && tickSize > 0 && slPoints > 0)
         {
            double riskAmount = equity * riskPercent / 100.0;
            lot = riskAmount / (slPoints * (tickValue / tickSize));
         }
         break;

      case EquityLot:
         lot = equity * riskPercent / 100.0; // Lot = % of equity
         break;
   }

   lot = MathMax(minLot, MathMin(maxLot, lot));
   lot = NormalizeDouble(MathFloor(lot / lotStep) * lotStep, (int)MathLog10(1.0 / lotStep));
   return lot;
}

bool IsCurrentTimeInInterval(ENUM_HOUR hStart, ENUM_MINUTE mStart, ENUM_HOUR hEnd, ENUM_MINUTE mEnd) {
   MqlDateTime now;
   TimeCurrent(now);
   int current = now.hour * 60 + now.min;
   int start   = hStart * 60 + mStart;
   int end     = hEnd   * 60 + mEnd;

   if (start == end && current == start) return true;
   if (start < end && current >= start && current <= end) return true;
   if (start > end && (current >= start || current <= end)) return true;

   return false;
}

int Hour() {
   MqlDateTime mTime;
   TimeCurrent(mTime);
   return mTime.hour;
}
// thank you //  for more info visit my website automationfx.in
