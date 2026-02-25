//+------------------------------------------------------------------+
//|     ULTIMATE TRADING ROBOT ELITE v6.0 - MQL5                   |
//|     All YouTube Strategies + All Candlestick Patterns           |
//|     SMC + ICT + Wyckoff + Supply/Demand + Fibonacci             |
//|     TP: 4% | SL: 2% | Lot: 0.01 | 1 Trade/Day | R:R = 1:2     |
//+------------------------------------------------------------------+
#property copyright "Ultimate Trading Robot ELITE v6.0"
#property version   "6.00"
#property strict

#include <Trade\Trade.mqh>
#include <Trade\PositionInfo.mqh>

CTrade        trade;
CPositionInfo pos;

//==========================================================================
// â–ˆâ–ˆ INPUT PARAMETERS
//==========================================================================
input group "â•â•â•â•â•â• TRADE SETTINGS â•â•â•â•â•â•"
input double  LotSize          = 0.01;
input double  TakeProfitPct    = 4.00;   // Take Profit 4%
input double  StopLossPct      = 2.00;   // Stop Loss 2%
input int     MagicNumber      = 88888;
input int     MaxTradesPerDay  = 1;
input int     MinScoreToTrade  = 7;      // Min confluence (1-25)

input group "â•â•â•â•â•â• TIMEFRAMES â•â•â•â•â•â•"
input ENUM_TIMEFRAMES TF_Entry  = PERIOD_H1;
input ENUM_TIMEFRAMES TF_Trend  = PERIOD_H4;
input ENUM_TIMEFRAMES TF_Higher = PERIOD_D1;

input group "â•â•â•â•â•â• STRATEGIES ON/OFF â•â•â•â•â•â•"
input bool UseICT               = true;
input bool UseWyckoff           = true;
input bool UseSupplyDemand      = true;
input bool UseSMC               = true;
input bool UseScalping          = true;
input bool UseBreakout          = true;
input bool UseMeanReversion     = true;
input bool UseEMARibbon         = true;
input bool UseVWAP              = true;
input bool UseFibGoldenZone     = true;
input bool UsePriceAction       = true;
input bool Use3TouchTrend       = true;
input bool UseRSIDivergence     = true;
input bool UseMultiTF           = true;
input bool UseADXTrend          = true;
input bool UseBollingerSqueeze  = true;
input bool UseMomentumStrategy  = true;

input group "â•â•â•â•â•â• SMC / ICT SETTINGS â•â•â•â•â•â•"
input int    SwingBars         = 25;
input double ZoneBuffer        = 0.0004;
input int    FVG_MinPips       = 3;
input bool   UseOrderBlocks    = true;
input bool   UseFVG            = true;
input bool   UseLiqSweep       = true;
input bool   UseBOS_CHOCH      = true;
input bool   UsePDArrays       = true;
input bool   UseKillzones      = true;
input bool   UseImbalance       = true;

input group "â•â•â•â•â•â• INDICATORS â•â•â•â•â•â•"
input int  RSI_P      = 14;
input int  EMA1_P     = 9;
input int  EMA2_P     = 21;
input int  EMA3_P     = 50;
input int  EMA4_P     = 100;
input int  EMA5_P     = 200;
input int  MACD_F     = 12;
input int  MACD_S     = 26;
input int  MACD_SIG   = 9;
input int  ATR_P      = 14;
input int  BB_P       = 20;
input double BB_D     = 2.0;
input int  STOCH_K    = 5;
input int  STOCH_D    = 3;
input int  STOCH_SL   = 3;
input int  CCI_P      = 14;
input int  WPR_P      = 14;
input int  MOM_P      = 10;
input int  ADX_P      = 14;
input int  SAR_STEP   = 2;     // Parabolic SAR (x0.01)
input int  SAR_MAX    = 20;    // Parabolic SAR max (x0.01)
input int  ICHIMOKU_T = 9;
input int  ICHIMOKU_K = 26;
input int  ICHIMOKU_S = 52;

input group "â•â•â•â•â•â• SESSION FILTER â•â•â•â•â•â•"
input bool  UseSessionFilter   = true;
input int   LondonStart        = 8;
input int   LondonEnd          = 17;
input int   NYStart            = 13;
input int   NYEnd              = 21;
input bool  SkipFriday         = true;
input bool  SkipMonday         = false;
input bool  SkipNewsTime       = true;   // Skip 30min around news hours

//==========================================================================
// HANDLES
//==========================================================================
int hRSI, hEMA1, hEMA2, hEMA3, hEMA4, hEMA5;
int hMACD, hATR, hBB, hStoch, hCCI, hWPR, hMOM, hADX;
int hSAR, hIchi_T, hIchi_K, hIchi_S;
int hRSI_H4, hEMA3_H4, hEMA5_H4, hATR_H4;
int hRSI_D1, hEMA5_D1, hMACD_D1;

double rsi[],ema1[],ema2[],ema3[],ema4[],ema5[];
double macdM[],macdSig[],atr[],bbU[],bbL[],bbMid[];
double stK[],stD[],cci[],wpr[],mom[],adx[],adxP[],adxM[];
double sarV[];
double ichiT[],ichiK[],ichiSA[],ichiSB[];
double rsiH4[],ema3H4[],ema5H4[],atrH4[];
double rsiD1[],ema5D1[],macdD1[];

//==========================================================================
// STRUCTURES
//==========================================================================
struct Zone {
   double hi, lo, mid;
   bool   valid, isBull;
   int    strength;
   datetime t;
};

struct FVG { double hi, lo; bool valid, isBull; };

struct MktStruct {
   double lastSH, lastSL, prevSH, prevSL;
   bool   bullTrend, bos, choch;
   double pdHigh, pdLow, pdMid;
};

struct WyckoffData {
   bool accum, distrib, spring, upthrust, UTAD, ST, AR;
};

struct FibData {
   double f0, f236, f382, f500, f618, f705, f786, f100;
   bool   valid;
};

struct RSIDivData {
   bool bullDiv, bearDiv;
};

Zone     bullOB, bearOB;
Zone     supplyZone[15], demandZone[15];
int      supplyCount=0, demandCount=0;
FVG      bullFVG, bearFVG;
MktStruct ms;
WyckoffData wyck;
FibData  fib;
RSIDivData rsidiv;

Zone     liqHi[30], liqLo[30];
int      liqHCount=0, liqLCount=0;

datetime lastBar=0, lastTradeDay=0;
int      todayTrades=0;
double   pip;
int      digits;

//==========================================================================
// INIT
//==========================================================================
int OnInit()
{
   trade.SetExpertMagicNumber(MagicNumber);
   trade.SetDeviationInPoints(30);
   trade.SetTypeFilling(ORDER_FILLING_FOK);
   digits = (int)SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);
   pip    = (digits==3||digits==5) ? 10*_Point : _Point;

   // Entry TF
   hRSI   = iRSI   (_Symbol,TF_Entry,RSI_P,PRICE_CLOSE);
   hEMA1  = iMA    (_Symbol,TF_Entry,EMA1_P,0,MODE_EMA,PRICE_CLOSE);
   hEMA2  = iMA    (_Symbol,TF_Entry,EMA2_P,0,MODE_EMA,PRICE_CLOSE);
   hEMA3  = iMA    (_Symbol,TF_Entry,EMA3_P,0,MODE_EMA,PRICE_CLOSE);
   hEMA4  = iMA    (_Symbol,TF_Entry,EMA4_P,0,MODE_EMA,PRICE_CLOSE);
   hEMA5  = iMA    (_Symbol,TF_Entry,EMA5_P,0,MODE_EMA,PRICE_CLOSE);
   hMACD  = iMACD  (_Symbol,TF_Entry,MACD_F,MACD_S,MACD_SIG,PRICE_CLOSE);
   hATR   = iATR   (_Symbol,TF_Entry,ATR_P);
   hBB    = iBands (_Symbol,TF_Entry,BB_P,0,BB_D,PRICE_CLOSE);
   hStoch = iStochastic(_Symbol,TF_Entry,STOCH_K,STOCH_D,STOCH_SL,MODE_SMA,STO_LOWHIGH);
   hCCI   = iCCI   (_Symbol,TF_Entry,CCI_P,PRICE_TYPICAL);
   hWPR   = iWPR   (_Symbol,TF_Entry,WPR_P);
   hMOM   = iMomentum(_Symbol,TF_Entry,MOM_P,PRICE_CLOSE);
   hADX   = iADX   (_Symbol,TF_Entry,ADX_P);
   hSAR   = iSAR   (_Symbol,TF_Entry,SAR_STEP*0.01,SAR_MAX*0.01);
   hIchi_T= iIchimoku(_Symbol,TF_Entry,ICHIMOKU_T,ICHIMOKU_K,ICHIMOKU_S);

   // Trend TF (H4)
   hRSI_H4  = iRSI(_Symbol,TF_Trend,RSI_P,PRICE_CLOSE);
   hEMA3_H4 = iMA (_Symbol,TF_Trend,EMA3_P,0,MODE_EMA,PRICE_CLOSE);
   hEMA5_H4 = iMA (_Symbol,TF_Trend,EMA5_P,0,MODE_EMA,PRICE_CLOSE);
   hATR_H4  = iATR(_Symbol,TF_Trend,ATR_P);

   // Higher TF (D1)
   hRSI_D1  = iRSI (_Symbol,TF_Higher,RSI_P,PRICE_CLOSE);
   hEMA5_D1 = iMA  (_Symbol,TF_Higher,EMA5_P,0,MODE_EMA,PRICE_CLOSE);
   hMACD_D1 = iMACD(_Symbol,TF_Higher,MACD_F,MACD_S,MACD_SIG,PRICE_CLOSE);

   if(hRSI==INVALID_HANDLE||hATR==INVALID_HANDLE||hMACD==INVALID_HANDLE)
   { Alert("Indicator init FAILED!"); return INIT_FAILED; }

   // Set all arrays as series
   ArraySetAsSeries(rsi,true);    ArraySetAsSeries(ema1,true);
   ArraySetAsSeries(ema2,true);   ArraySetAsSeries(ema3,true);
   ArraySetAsSeries(ema4,true);   ArraySetAsSeries(ema5,true);
   ArraySetAsSeries(macdM,true);  ArraySetAsSeries(macdSig,true);
   ArraySetAsSeries(atr,true);    ArraySetAsSeries(bbU,true);
   ArraySetAsSeries(bbL,true);    ArraySetAsSeries(bbMid,true);
   ArraySetAsSeries(stK,true);    ArraySetAsSeries(stD,true);
   ArraySetAsSeries(cci,true);    ArraySetAsSeries(wpr,true);
   ArraySetAsSeries(mom,true);    ArraySetAsSeries(adx,true);
   ArraySetAsSeries(adxP,true);   ArraySetAsSeries(adxM,true);
   ArraySetAsSeries(sarV,true);
   ArraySetAsSeries(ichiT,true);  ArraySetAsSeries(ichiK,true);
   ArraySetAsSeries(ichiSA,true); ArraySetAsSeries(ichiSB,true);
   ArraySetAsSeries(rsiH4,true);  ArraySetAsSeries(ema3H4,true);
   ArraySetAsSeries(ema5H4,true); ArraySetAsSeries(atrH4,true);
   ArraySetAsSeries(rsiD1,true);  ArraySetAsSeries(ema5D1,true);
   ArraySetAsSeries(macdD1,true);

   Print("â•”â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•—");
   Print("â•‘  ULTIMATE TRADING ROBOT ELITE v6.0       â•‘");
   Print("â• â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•£");
   Print("â•‘  Symbol  : ", _Symbol);
   Print("â•‘  TF      : ", EnumToString(TF_Entry)," / ",EnumToString(TF_Trend)," / ",EnumToString(TF_Higher));
   Print("â•‘  TP      : +",TakeProfitPct,"% | SL: -",StopLossPct,"%");
   Print("â•‘  R:R     : 1:",DoubleToString(TakeProfitPct/StopLossPct,1));
   Print("â•‘  Lot     : ",LotSize,"  | Min Score: ",MinScoreToTrade);
   Print("â•šâ•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•");

   return INIT_SUCCEEDED;
}

void OnDeinit(const int r)
{
   int h[]={hRSI,hEMA1,hEMA2,hEMA3,hEMA4,hEMA5,hMACD,hATR,hBB,hStoch,
            hCCI,hWPR,hMOM,hADX,hSAR,hIchi_T,hRSI_H4,hEMA3_H4,hEMA5_H4,
            hATR_H4,hRSI_D1,hEMA5_D1,hMACD_D1};
   for(int i=0;i<ArraySize(h);i++)
      if(h[i]!=INVALID_HANDLE) IndicatorRelease(h[i]);
   Print("EA Stopped.");
}

//==========================================================================
// MAIN TICK
//==========================================================================
void OnTick()
{
   datetime cb = iTime(_Symbol,TF_Entry,0);
   if(cb==lastBar) return;
   lastBar=cb;

   ResetDailyCounter();
   if(todayTrades >= MaxTradesPerDay) return;
   if(OpenPositions() > 0) return;
   if(UseSessionFilter && !InSession()) return;
   if(SkipFriday && IsDay(5) && GetHour()>=16) return;
   if(SkipMonday && IsDay(1)) return;

   if(!LoadIndicators()) return;

   // Load price data
   MqlRates R[];  ArraySetAsSeries(R,true);
   if(CopyRates(_Symbol,TF_Entry,0,120,R)<80) return;

   MqlRates RH4[]; ArraySetAsSeries(RH4,true);
   CopyRates(_Symbol,TF_Trend,0,60,RH4);

   MqlRates RD1[]; ArraySetAsSeries(RD1,true);
   CopyRates(_Symbol,TF_Higher,0,30,RD1);

   // â•â•â•â•â•â•â•â• FULL MARKET ANALYSIS â•â•â•â•â•â•â•â•
   AnalyzeMarketStructure(R);
   DetectLiquidityZones(R);
   DetectOrderBlocks(R);
   DetectFVG(R);
   DetectSupplyDemand(R);
   AnalyzeWyckoff(R);
   CalcFibonacci(R);
   DetectRSIDivergence(R);

   // â•â•â•â•â•â•â•â• SCORE ALL STRATEGIES â•â•â•â•â•â•â•â•
   int bull=0, bear=0;
   string bR="", sR="";

   // All strategies scoring
   ScoreCandlePatterns(R,bull,bear,bR,sR);
   if(UseSMC)            ScoreSMC(bull,bear,bR,sR);
   if(UseICT)            ScoreICT(R,bull,bear,bR,sR);
   if(UseWyckoff)        ScoreWyckoff(bull,bear,bR,sR);
   if(UseSupplyDemand)   ScoreSupplyDemand(bull,bear,bR,sR);
   if(UseEMARibbon)      ScoreEMARibbon(bull,bear,bR,sR);
   if(UseBreakout)       ScoreBreakout(R,bull,bear,bR,sR);
   if(UseMeanReversion)  ScoreMeanReversion(bull,bear,bR,sR);
   if(UseVWAP)           ScoreVWAP(R,bull,bear,bR,sR);
   if(UseFibGoldenZone)  ScoreFibZone(bull,bear,bR,sR);
   if(UseMultiTF)        ScoreMultiTF(RH4,RD1,bull,bear,bR,sR);
   if(UseScalping)       ScoreScalping(R,bull,bear,bR,sR);
   if(Use3TouchTrend)    Score3Touch(R,bull,bear,bR,sR);
   if(UseRSIDivergence)  ScoreRSIDiv(bull,bear,bR,sR);
   if(UseADXTrend)       ScoreADX(bull,bear,bR,sR);
   if(UseBollingerSqueeze) ScoreBBSqueeze(R,bull,bear,bR,sR);
   if(UseMomentumStrategy) ScoreMomentum(bull,bear,bR,sR);
   ScoreIchimoku(bull,bear,bR,sR);
   ScoreParabolicSAR(bull,bear,bR,sR);
   ScoreIndicators(bull,bear,bR,sR);

   // â•â•â•â•â•â•â•â• PRINT SIGNAL REPORT â•â•â•â•â•â•â•â•
   Print("â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€");
   Print("TIME  : ", TimeToString(TimeCurrent(),TIME_DATE|TIME_MINUTES));
   Print("BULL Score: ",bull," | BEAR Score: ",bear," | Min: ",MinScoreToTrade);
   if(bull>0) Print("BULL Signals: ", bR);
   if(bear>0) Print("BEAR Signals: ", sR);
   Print("â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€");

   // â•â•â•â•â•â•â•â• EXECUTE â•â•â•â•â•â•â•â•
   if(bull>=MinScoreToTrade && bull>bear && bull>=(bear+3))
      ExecuteBuy();
   else if(bear>=MinScoreToTrade && bear>bull && bear>=(bull+3))
      ExecuteSell();
   else
      Print("No trade. Waiting for stronger confluence...");
}

//==========================================================================
// LOAD INDICATORS
//==========================================================================
bool LoadIndicators()
{
   int n=80;
   if(CopyBuffer(hRSI,  0,0,n,rsi)   <0) return false;
   if(CopyBuffer(hEMA1, 0,0,n,ema1)  <0) return false;
   if(CopyBuffer(hEMA2, 0,0,n,ema2)  <0) return false;
   if(CopyBuffer(hEMA3, 0,0,n,ema3)  <0) return false;
   if(CopyBuffer(hEMA4, 0,0,n,ema4)  <0) return false;
   if(CopyBuffer(hEMA5, 0,0,n,ema5)  <0) return false;
   if(CopyBuffer(hMACD, 0,0,n,macdM) <0) return false;
   if(CopyBuffer(hMACD, 1,0,n,macdSig)<0)return false;
   if(CopyBuffer(hATR,  0,0,n,atr)   <0) return false;
   if(CopyBuffer(hBB,   1,0,n,bbU)   <0) return false;
   if(CopyBuffer(hBB,   2,0,n,bbL)   <0) return false;
   if(CopyBuffer(hBB,   0,0,n,bbMid) <0) return false;
   if(CopyBuffer(hStoch,0,0,n,stK)   <0) return false;
   if(CopyBuffer(hStoch,1,0,n,stD)   <0) return false;
   if(CopyBuffer(hCCI,  0,0,n,cci)   <0) return false;
   if(CopyBuffer(hWPR,  0,0,n,wpr)   <0) return false;
   if(CopyBuffer(hMOM,  0,0,n,mom)   <0) return false;
   if(CopyBuffer(hADX,  0,0,n,adx)   <0) return false;
   if(CopyBuffer(hADX,  1,0,n,adxP)  <0) return false;
   if(CopyBuffer(hADX,  2,0,n,adxM)  <0) return false;
   if(CopyBuffer(hSAR,  0,0,n,sarV)  <0) return false;
   if(CopyBuffer(hIchi_T,0,0,n,ichiT)<0) return false;
   if(CopyBuffer(hIchi_T,1,0,n,ichiK)<0) return false;
   if(CopyBuffer(hIchi_T,2,0,n,ichiSA)<0)return false;
   if(CopyBuffer(hIchi_T,3,0,n,ichiSB)<0)return false;
   if(CopyBuffer(hRSI_H4, 0,0,30,rsiH4) <0) return false;
   if(CopyBuffer(hEMA3_H4,0,0,30,ema3H4)<0) return false;
   if(CopyBuffer(hEMA5_H4,0,0,30,ema5H4)<0) return false;
   if(CopyBuffer(hATR_H4, 0,0,30,atrH4) <0) return false;
   if(CopyBuffer(hRSI_D1, 0,0,15,rsiD1) <0) return false;
   if(CopyBuffer(hEMA5_D1,0,0,15,ema5D1)<0) return false;
   if(CopyBuffer(hMACD_D1,0,0,15,macdD1)<0) return false;
   return true;
}

//==========================================================================
// â–“â–“ STRATEGY 1: 50+ CANDLESTICK PATTERNS
//==========================================================================
void ScoreCandlePatterns(MqlRates &R[],int &B,int &S,string &bR,string &sR)
{
   // â”€â”€ SINGLE CANDLE â”€â”€
   if(IsHammer(R[1]))          { B+=2; bR+="Hammer|"; }
   if(IsShootingStar(R[1]))    { S+=2; sR+="ShootingStar|"; }
   if(IsInvHammer(R[1]))       { B+=1; bR+="InvHammer|"; }
   if(IsHangingMan(R[1]))      { S+=1; sR+="HangingMan|"; }
   if(IsDragonflyDoji(R[1]))   { B+=2; bR+="DragonflyDoji|"; }
   if(IsGravestoneDoji(R[1]))  { S+=2; sR+="GravestoneDoji|"; }
   if(IsLLDoji(R[1]))          { if(rsi[1]<50)B+=1; else S+=1; }
   if(IsBullMarubozu(R[1]))    { B+=2; bR+="BullMarubozu|"; }
   if(IsBearMarubozu(R[1]))    { S+=2; sR+="BearMarubozu|"; }
   if(IsBullBelt(R[1]))        { B+=1; bR+="BullBeltHold|"; }
   if(IsBearBelt(R[1]))        { S+=1; sR+="BearBeltHold|"; }
   if(IsSpinTop(R[1]))         { if(rsi[1]<40)B+=1; else if(rsi[1]>60)S+=1; }
   if(IsHighWave(R[1]))        { if(rsi[1]<30)B+=1; else if(rsi[1]>70)S+=1; }
   if(IsRicketsSeat(R[1]))     { B+=1; bR+="RicketsSeat|"; }

   // â”€â”€ TWO CANDLE â”€â”€
   if(IsBullEngulf(R[1],R[2]))    { B+=3; bR+="BullEngulf|"; }
   if(IsBearEngulf(R[1],R[2]))    { S+=3; sR+="BearEngulf|"; }
   if(IsBullHarami(R[1],R[2]))    { B+=2; bR+="BullHarami|"; }
   if(IsBearHarami(R[1],R[2]))    { S+=2; sR+="BearHarami|"; }
   if(IsBullHaramiX(R[1],R[2]))   { B+=2; bR+="BullHaramiX|"; }
   if(IsBearHaramiX(R[1],R[2]))   { S+=2; sR+="BearHaramiX|"; }
   if(IsTweezerBot(R[1],R[2]))    { B+=2; bR+="TweezerBottom|"; }
   if(IsTweezerTop(R[1],R[2]))    { S+=2; sR+="TweezerTop|"; }
   if(IsPiercing(R[1],R[2]))      { B+=2; bR+="PiercingLine|"; }
   if(IsDarkCloud(R[1],R[2]))     { S+=2; sR+="DarkCloudCover|"; }
   if(IsOnNeck(R[1],R[2]))        { S+=1; sR+="OnNeck|"; }
   if(IsInNeck(R[1],R[2]))        { S+=1; sR+="InNeck|"; }
   if(IsThrusting(R[1],R[2]))     { S+=1; sR+="Thrusting|"; }
   if(IsKicking(R[1],R[2]))
   { if(IsBull(R[1])){B+=3;bR+="BullKicking|";}else{S+=3;sR+="BearKicking|";} }
   if(IsCounterAttackB(R[1],R[2])){ B+=2; bR+="CounterAttackBull|"; }
   if(IsCounterAttackS(R[1],R[2])){ S+=2; sR+="CounterAttackBear|"; }
   if(IsMatchingLow(R[1],R[2]))   { B+=1; bR+="MatchingLow|"; }
   if(IsMatchingHigh(R[1],R[2]))  { S+=1; sR+="MatchingHigh|"; }
   if(IsHomingPigeon(R[1],R[2]))  { B+=2; bR+="HomingPigeon|"; }
   if(IsConcealBabySwallow(R[1],R[2])){ B+=2; bR+="ConcealBabySwallow|"; }
   if(IsSeparatingLines(R[1],R[2])){ if(ms.bullTrend){B+=2;bR+="BullSepLines|";}else{S+=2;sR+="BearSepLines|";} }

   // â”€â”€ THREE CANDLE â”€â”€
   if(IsMorningStar(R[1],R[2],R[3]))       { B+=4; bR+="MorningStar|"; }
   if(IsEveningStar(R[1],R[2],R[3]))       { S+=4; sR+="EveningStar|"; }
   if(IsMorningDojiStar(R[1],R[2],R[3]))   { B+=4; bR+="MorningDojiStar|"; }
   if(IsEveningDojiStar(R[1],R[2],R[3]))   { S+=4; sR+="EveningDojiStar|"; }
   if(Is3WS(R[1],R[2],R[3]))               { B+=4; bR+="3WhiteSoldiers|"; }
   if(Is3BC(R[1],R[2],R[3]))               { S+=4; sR+="3BlackCrows|"; }
   if(Is3IU(R[1],R[2],R[3]))               { B+=3; bR+="3InsideUp|"; }
   if(Is3ID(R[1],R[2],R[3]))               { S+=3; sR+="3InsideDown|"; }
   if(Is3OU(R[1],R[2],R[3]))               { B+=3; bR+="3OutsideUp|"; }
   if(Is3OD(R[1],R[2],R[3]))               { S+=3; sR+="3OutsideDown|"; }
   if(IsAbandonedBabyB(R[1],R[2],R[3]))    { B+=4; bR+="AbandonedBabyBull|"; }
   if(IsAbandonedBabyS(R[1],R[2],R[3]))    { S+=4; sR+="AbandonedBabyBear|"; }
   if(IsUpsideGap2Crows(R[1],R[2],R[3]))   { S+=2; sR+="UpsideGap2Crows|"; }
   if(IsDeliberation(R[1],R[2],R[3]))      { S+=2; sR+="Deliberation|"; }
   if(IsAdvanceBlock(R[1],R[2],R[3]))      { S+=2; sR+="AdvanceBlock|"; }
   if(IsStalled(R[1],R[2],R[3]))           { S+=1; sR+="StalledPattern|"; }
   if(IsIdenticalBC(R[1],R[2],R[3]))       { S+=2; sR+="IdenticalBlackCrows|"; }
   if(IsBullTriStar(R[1],R[2],R[3]))       { B+=3; bR+="BullTriStar|"; }
   if(IsBearTriStar(R[1],R[2],R[3]))       { S+=3; sR+="BearTriStar|"; }
   if(IsDojiStar(R[1],R[2],R[3]))          { if(ms.bullTrend)S+=2; else B+=2; }
   if(IsBreakaway_Bull(R[1],R[2],R[3]))    { B+=3; bR+="BullBreakaway|"; }
   if(IsBreakaway_Bear(R[1],R[2],R[3]))    { S+=3; sR+="BearBreakaway|"; }
   if(IsMatHold(R[1],R[2],R[3]))           { B+=2; bR+="MatHold|"; }
   if(IsRisingSun(R[1],R[2],R[3]))         { B+=2; bR+="RisingSun|"; }

   // â”€â”€ FOUR/FIVE CANDLE â”€â”€
   if(Is3LS_Bull(R[1],R[2],R[3],R[4]))     { B+=2; bR+="Bull3LineStrike|"; }
   if(Is3LS_Bear(R[1],R[2],R[3],R[4]))     { S+=2; sR+="Bear3LineStrike|"; }
   if(IsRising3(R[1],R[2],R[3],R[4],R[5])) { B+=3; bR+="Rising3Methods|"; }
   if(IsFalling3(R[1],R[2],R[3],R[4],R[5])){ S+=3; sR+="Falling3Methods|"; }
   if(IsLadderBottom(R[1],R[2],R[3],R[4],R[5])){ B+=3; bR+="LadderBottom|"; }
   if(IsLadderTop(R[1],R[2],R[3],R[4],R[5]))   { S+=3; sR+="LadderTop|"; }
}

//==========================================================================
// â–“â–“ STRATEGY 2: SMC
//==========================================================================
void ScoreSMC(int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);

   if(ms.bullTrend){B+=2;bR+="BullMktStruct|";}else{S+=2;sR+="BearMktStruct|";}
   if(ms.bos) { if(ms.bullTrend){B+=3;bR+="BullBOS|";}else{S+=3;sR+="BearBOS|";} }
   if(ms.choch){ if(ms.bullTrend){B+=3;bR+="BullCHOCH|";}else{S+=3;sR+="BearCHOCH|";} }

   if(UsePDArrays && ms.pdMid>0)
   { if(price<ms.pdMid){B+=1;bR+="DiscountZone|";}else{S+=1;sR+="PremiumZone|";} }

   if(UseOrderBlocks)
   {
      if(bullOB.valid&&price>=bullOB.lo-ZoneBuffer&&price<=bullOB.hi+ZoneBuffer)
      { B+=3; bR+="BullOB|"; }
      if(bearOB.valid&&price>=bearOB.lo-ZoneBuffer&&price<=bearOB.hi+ZoneBuffer)
      { S+=3; sR+="BearOB|"; }
   }

   if(UseFVG)
   {
      if(bullFVG.valid&&price>=bullFVG.lo&&price<=bullFVG.hi){B+=2;bR+="BullFVG|";}
      if(bearFVG.valid&&price>=bearFVG.lo&&price<=bearFVG.hi){S+=2;sR+="BearFVG|";}
   }

   if(UseLiqSweep)
   {
      for(int i=0;i<liqLCount;i++)
         if(liqLo[i].valid&&MathAbs(price-liqLo[i].lo)<=ZoneBuffer*8)
         { B+=2; bR+="LiqSweepLow|"; break; }
      for(int i=0;i<liqHCount;i++)
         if(liqHi[i].valid&&MathAbs(price-liqHi[i].hi)<=ZoneBuffer*8)
         { S+=2; sR+="LiqSweepHigh|"; break; }
   }

   if(UseImbalance&&bullOB.valid&&bullFVG.valid&&MathAbs(bullFVG.lo-bullOB.hi)<=atr[1])
   { B+=2; bR+="SMC_Imbalance|"; }
}

//==========================================================================
// â–“â–“ STRATEGY 3: ICT
//==========================================================================
void ScoreICT(MqlRates &R[],int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);

   if(UseKillzones)
   {
      int h=GetHour();
      bool kz=(h>=7&&h<=10)||(h>=12&&h<=15)||(h>=3&&h<=5);
      if(kz){B+=1;S+=1;bR+="InKillzone|";sR+="InKillzone|";}
   }

   // OTE: 62-79% fib retracement
   if(fib.valid)
   {
      if(price>=fib.f618&&price<=fib.f786&&ms.bullTrend){B+=3;bR+="ICT_OTE_Bull|";}
      if(price>=fib.f236&&price<=fib.f382&&!ms.bullTrend){S+=3;sR+="ICT_OTE_Bear|";}
   }

   // Breaker Blocks
   if(ms.bos||ms.choch)
   {
      if(ms.bullTrend&&price<ema2[1]){B+=2;bR+="ICT_BreakerBlock|";}
      if(!ms.bullTrend&&price>ema2[1]){S+=2;sR+="ICT_BreakerBlock|";}
   }

   // Mitigation Blocks
   if(bullOB.valid&&price>=bullOB.lo&&price<=bullOB.mid&&ms.bullTrend)
   { B+=2; bR+="ICT_Mitigation|"; }
   if(bearOB.valid&&price>=bearOB.mid&&price<=bearOB.hi&&!ms.bullTrend)
   { S+=2; sR+="ICT_Mitigation|"; }

   // Unicorn: FVG + OB confluence
   if(bullFVG.valid&&bullOB.valid&&MathAbs(bullFVG.lo-bullOB.hi)<=atr[1]*0.8)
   { B+=3; bR+="ICT_Unicorn|"; }
   if(bearFVG.valid&&bearOB.valid&&MathAbs(bearFVG.hi-bearOB.lo)<=atr[1]*0.8)
   { S+=3; sR+="ICT_Unicorn|"; }

   // Power of 3
   if(liqLCount>0&&ms.bullTrend&&rsi[1]<38){B+=2;bR+="ICT_Po3_Bull|";}
   if(liqHCount>0&&!ms.bullTrend&&rsi[1]>62){S+=2;sR+="ICT_Po3_Bear|";}

   // Daily Bias
   double d1Price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   if(d1Price>ema5D1[1]){B+=1;bR+="ICT_D1_BullBias|";}else{S+=1;sR+="ICT_D1_BearBias|";}
}

//==========================================================================
// â–“â–“ STRATEGY 4: WYCKOFF
//==========================================================================
void ScoreWyckoff(int &B,int &S,string &bR,string &sR)
{
   if(wyck.accum&&wyck.spring)  {B+=4;bR+="Wyckoff_Spring|";}
   if(wyck.distrib&&wyck.upthrust){S+=4;sR+="Wyckoff_Upthrust|";}
   if(wyck.accum&&!wyck.spring) {B+=2;bR+="Wyckoff_Accum|";}
   if(wyck.distrib&&!wyck.upthrust){S+=2;sR+="Wyckoff_Distrib|";}
   if(wyck.UTAD)                {S+=2;sR+="Wyckoff_UTAD|";}
   if(wyck.ST)                  {B+=1;bR+="Wyckoff_SecondaryTest|";}
   if(wyck.AR&&wyck.accum)      {B+=1;bR+="Wyckoff_AutoRally|";}
}

//==========================================================================
// â–“â–“ STRATEGY 5: SUPPLY & DEMAND
//==========================================================================
void ScoreSupplyDemand(int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   for(int i=0;i<demandCount;i++)
      if(demandZone[i].valid&&price>=demandZone[i].lo&&price<=demandZone[i].hi)
      { B+=3; bR+="DemandZone|"; break; }
   for(int i=0;i<supplyCount;i++)
      if(supplyZone[i].valid&&price>=supplyZone[i].lo&&price<=supplyZone[i].hi)
      { S+=3; sR+="SupplyZone|"; break; }
}

//==========================================================================
// â–“â–“ STRATEGY 6: EMA RIBBON
//==========================================================================
void ScoreEMARibbon(int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   bool bullRib=(ema1[1]>ema2[1]&&ema2[1]>ema3[1]&&ema3[1]>ema4[1]&&ema4[1]>ema5[1]);
   bool bearRib=(ema1[1]<ema2[1]&&ema2[1]<ema3[1]&&ema3[1]<ema4[1]&&ema4[1]<ema5[1]);

   if(bullRib){B+=3;bR+="FullBullRibbon|";}
   if(bearRib){S+=3;sR+="FullBearRibbon|";}

   if(ema1[1]>ema2[1]&&ema1[2]<=ema2[2]){B+=2;bR+="EMA_GoldenCross|";}
   if(ema1[1]<ema2[1]&&ema1[2]>=ema2[2]){S+=2;sR+="EMA_DeathCross|";}

   if(price>ema1[1]&&price>ema2[1]&&price>ema3[1]&&price>ema5[1]){B+=2;bR+="PriceAboveAllEMA|";}
   if(price<ema1[1]&&price<ema2[1]&&price<ema3[1]&&price<ema5[1]){S+=2;sR+="PriceBelowAllEMA|";}

   // Pullback to EMA21 in trend
   if(ms.bullTrend&&MathAbs(price-ema2[1])<=atr[1]*0.4){B+=2;bR+="Pullback_EMA21|";}
   if(!ms.bullTrend&&MathAbs(price-ema2[1])<=atr[1]*0.4){S+=2;sR+="Pullback_EMA21|";}

   // EMA50 bounce
   if(ms.bullTrend&&MathAbs(price-ema3[1])<=atr[1]*0.5&&IsBull(R_Last())){B+=2;bR+="EMA50_Bounce|";}
   if(!ms.bullTrend&&MathAbs(price-ema3[1])<=atr[1]*0.5&&IsBear(R_Last())){S+=2;sR+="EMA50_Bounce|";}

   // H4 trend alignment
   if(price>ema5H4[1]){B+=2;bR+="H4_AboveEMA200|";}else{S+=2;sR+="H4_BelowEMA200|";}
}

//==========================================================================
// â–“â–“ STRATEGY 7: BREAKOUT
//==========================================================================
void ScoreBreakout(MqlRates &R[],int &B,int &S,string &bR,string &sR)
{
   double hi=0,lo=DBL_MAX;
   for(int i=2;i<=20;i++){if(R[i].high>hi)hi=R[i].high;if(R[i].low<lo)lo=R[i].low;}
   double rangePct=(hi-lo)/lo*100;

   if(rangePct<1.5) // consolidation
   {
      if(R[1].close>hi){B+=3;bR+="BullBreakout|";}
      if(R[1].close<lo){S+=3;sR+="BearBreakout|";}
   }

   // Volume-based breakout (using tick volume)
   double avgVol=0;
   for(int i=2;i<=20;i++) avgVol+=(double)R[i].tick_volume;
   avgVol/=19.0;
   if((double)R[1].tick_volume>avgVol*1.8)
   {
      if(R[1].close>R[1].open){B+=2;bR+="VolBreakout_Bull|";}
      else                     {S+=2;sR+="VolBreakout_Bear|";}
   }

   // London open breakout
   int h=GetHour();
   if(h==8||h==9)
   {
      double lHi=0,lLo=DBL_MAX;
      for(int i=2;i<=12;i++){if(R[i].high>lHi)lHi=R[i].high;if(R[i].low<lLo)lLo=R[i].low;}
      if(R[1].close>lHi){B+=2;bR+="LondonBO_Bull|";}
      if(R[1].close<lLo){S+=2;sR+="LondonBO_Bear|";}
   }

   // NY open breakout
   if(h==13||h==14)
   {
      if(R[1].close>R[5].high){B+=2;bR+="NY_BO_Bull|";}
      if(R[1].close<R[5].low) {S+=2;sR+="NY_BO_Bear|";}
   }
}

//==========================================================================
// â–“â–“ STRATEGY 8: MEAN REVERSION
//==========================================================================
void ScoreMeanReversion(int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   double distPct=MathAbs(price-ema5[1])/ema5[1]*100;
   if(distPct>2.0)
   {
      if(price<ema5[1]&&rsi[1]<35){B+=2;bR+="MeanRev_Buy|";}
      if(price>ema5[1]&&rsi[1]>65){S+=2;sR+="MeanRev_Sell|";}
   }
   if(price<bbL[1]&&rsi[1]<30){B+=2;bR+="BB_MeanRev_Buy|";}
   if(price>bbU[1]&&rsi[1]>70){S+=2;sR+="BB_MeanRev_Sell|";}
}

//==========================================================================
// â–“â–“ STRATEGY 9: VWAP
//==========================================================================
void ScoreVWAP(MqlRates &R[],int &B,int &S,string &bR,string &sR)
{
   double cumPV=0,cumV=0;
   MqlDateTime dt; TimeToStruct(R[0].time,dt);
   for(int i=0;i<60;i++)
   {
      MqlDateTime di; TimeToStruct(R[i].time,di);
      if(di.day!=dt.day) break;
      double tp=(R[i].high+R[i].low+R[i].close)/3.0;
      cumPV+=tp*(double)R[i].tick_volume;
      cumV +=(double)R[i].tick_volume;
   }
   if(cumV<=0) return;
   double vwap=cumPV/cumV;
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);

   if(price>vwap&&MathAbs(price-vwap)<=atr[1]*0.5){B+=1;bR+="AboveVWAP|";}
   if(price<vwap&&MathAbs(price-vwap)<=atr[1]*0.5){S+=1;sR+="BelowVWAP|";}
   if(ms.bullTrend&&price>vwap&&R[1].low<=vwap+atr[1]*0.15){B+=2;bR+="VWAP_Bull_Retest|";}
   if(!ms.bullTrend&&price<vwap&&R[1].high>=vwap-atr[1]*0.15){S+=2;sR+="VWAP_Bear_Retest|";}
}

//==========================================================================
// â–“â–“ STRATEGY 10: FIBONACCI GOLDEN ZONE
//==========================================================================
void ScoreFibZone(int &B,int &S,string &bR,string &sR)
{
   if(!fib.valid) return;
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);

   if(ms.bullTrend&&price>=fib.f500&&price<=fib.f618){B+=3;bR+="Fib_GoldenZone|";}
   if(!ms.bullTrend&&price>=fib.f382&&price<=fib.f500){S+=3;sR+="Fib_GoldenZone|";}
   if(ms.bullTrend&&price>=fib.f618&&price<=fib.f786){B+=2;bR+="Fib_786_OTE|";}
   if(!ms.bullTrend&&price>=fib.f236&&price<=fib.f382){S+=2;sR+="Fib_236_OTE|";}
}

//==========================================================================
// â–“â–“ STRATEGY 11: MULTI-TIMEFRAME
//==========================================================================
void ScoreMultiTF(MqlRates &RH4[],MqlRates &RD1[],int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);

   // D1 bias
   if(price>ema5D1[1]){B+=2;bR+="D1_BullTrend|";}else{S+=2;sR+="D1_BearTrend|";}
   if(macdD1[1]>0){B+=1;bR+="D1_MACD_Bull|";}else{S+=1;sR+="D1_MACD_Bear|";}

   // H4 candles
   if(ArraySize(RH4)>5)
   {
      if(IsBullEngulf(RH4[1],RH4[2])){B+=2;bR+="H4_BullEngulf|";}
      if(IsBearEngulf(RH4[1],RH4[2])){S+=2;sR+="H4_BearEngulf|";}
      if(IsHammer(RH4[1])){B+=2;bR+="H4_Hammer|";}
      if(IsShootingStar(RH4[1])){S+=2;sR+="H4_ShootingStar|";}
      if(IsMorningStar(RH4[1],RH4[2],RH4[3])){B+=3;bR+="H4_MorningStar|";}
      if(IsEveningStar(RH4[1],RH4[2],RH4[3])){S+=3;sR+="H4_EveningStar|";}
   }

   // Triple TF alignment
   bool e1=rsi[1]>50, e2=rsiH4[1]>50, e3=rsiD1[1]>50;
   if(e1&&e2&&e3){B+=3;bR+="3TF_BullAlign|";}
   if(!e1&&!e2&&!e3){S+=3;sR+="3TF_BearAlign|";}

   // H4 EMA alignment
   if(ema3H4[1]>ema5H4[1]){B+=1;bR+="H4_EMA_Bull|";}else{S+=1;sR+="H4_EMA_Bear|";}
}

//==========================================================================
// â–“â–“ STRATEGY 12: SCALPING
//==========================================================================
void ScoreScalping(MqlRates &R[],int &B,int &S,string &bR,string &sR)
{
   if(ema1[1]>ema2[1]&&ema1[2]<=ema2[2]&&rsi[1]>50&&rsi[1]<68){B+=2;bR+="Scalp_EMA_Bull|";}
   if(ema1[1]<ema2[1]&&ema1[2]>=ema2[2]&&rsi[1]<50&&rsi[1]>32){S+=2;sR+="Scalp_EMA_Bear|";}
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   if(IsHammer(R[1])&&R[1].low<=bbL[1]+atr[1]*0.3){B+=2;bR+="Scalp_PinBull|";}
   if(IsShootingStar(R[1])&&R[1].high>=bbU[1]-atr[1]*0.3){S+=2;sR+="Scalp_PinBear|";}
   if(stK[1]<20&&stK[1]>stD[1]&&stK[2]<=stD[2]){B+=2;bR+="Scalp_Stoch_OsX|";}
   if(stK[1]>80&&stK[1]<stD[1]&&stK[2]>=stD[2]){S+=2;sR+="Scalp_Stoch_ObX|";}
}

//==========================================================================
// â–“â–“ STRATEGY 13: 3-TOUCH TRENDLINE
//==========================================================================
void Score3Touch(MqlRates &R[],int &B,int &S,string &bR,string &sR)
{
   bool hl=(R[1].low>R[3].low&&R[3].low>R[6].low&&R[6].low>R[9].low);
   bool lh=(R[1].high<R[3].high&&R[3].high<R[6].high&&R[6].high<R[9].high);
   if(hl&&rsi[1]>45){B+=2;bR+="3Touch_HL_Bull|";}
   if(lh&&rsi[1]<55){S+=2;sR+="3Touch_LH_Bear|";}
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   if(hl)
   {
      double estSup=R[3].low+(R[1].low-R[3].low)*0.2;
      if(MathAbs(price-estSup)<=atr[1]*0.6){B+=2;bR+="TrendlineBounce|";}
   }
}

//==========================================================================
// â–“â–“ STRATEGY 14: RSI DIVERGENCE
//==========================================================================
void ScoreRSIDiv(int &B,int &S,string &bR,string &sR)
{
   if(rsidiv.bullDiv){B+=3;bR+="RSI_BullDiv|";}
   if(rsidiv.bearDiv){S+=3;sR+="RSI_BearDiv|";}
}

//==========================================================================
// â–“â–“ STRATEGY 15: ADX TREND STRENGTH
//==========================================================================
void ScoreADX(int &B,int &S,string &bR,string &sR)
{
   if(adx[1]>25)
   {
      if(adxP[1]>adxM[1]&&adxP[1]>adxP[2]){B+=2;bR+="ADX_StrongBull|";}
      if(adxM[1]>adxP[1]&&adxM[1]>adxM[2]){S+=2;sR+="ADX_StrongBear|";}
   }
   if(adx[1]>40){if(adxP[1]>adxM[1])bR+="ADX_VeryStrong|"; else sR+="ADX_VeryStrong|";}
   if(adx[1]<20){B-=1;S-=1;} // weak trend, reduce score
}

//==========================================================================
// â–“â–“ STRATEGY 16: BOLLINGER BAND SQUEEZE (Keltner Expansion)
//==========================================================================
void ScoreBBSqueeze(MqlRates &R[],int &B,int &S,string &bR,string &sR)
{
   double bwNow=bbU[1]-bbL[1];
   double bwOld=bbU[10]-bbL[10];
   if(bwNow<bwOld*0.65) // squeeze
   {
      if(R[1].close>bbMid[1]&&R[1].close>R[2].close&&R[1].close>R[3].close)
      { B+=3; bR+="BB_Squeeze_BullExpand|"; }
      if(R[1].close<bbMid[1]&&R[1].close<R[2].close&&R[1].close<R[3].close)
      { S+=3; sR+="BB_Squeeze_BearExpand|"; }
   }
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   if(price<bbL[1]){B+=2;bR+="BB_Lower_Bounce|";}
   if(price>bbU[1]){S+=2;sR+="BB_Upper_Reject|";}
}

//==========================================================================
// â–“â–“ STRATEGY 17: MOMENTUM
//==========================================================================
void ScoreMomentum(int &B,int &S,string &bR,string &sR)
{
   if(mom[1]>100&&mom[2]<=100){B+=2;bR+="MOM_CrossUp|";}
   if(mom[1]<100&&mom[2]>=100){S+=2;sR+="MOM_CrossDown|";}
   if(mom[1]>100&&mom[1]>mom[2]&&mom[2]>mom[3]){B+=1;bR+="MOM_Rising|";}
   if(mom[1]<100&&mom[1]<mom[2]&&mom[2]<mom[3]){S+=1;sR+="MOM_Falling|";}
   if(mom[1]>110){B+=1;bR+="MOM_StrongBull|";}
   if(mom[1]<90) {S+=1;sR+="MOM_StrongBear|";}
}

//==========================================================================
// â–“â–“ STRATEGY 18: ICHIMOKU CLOUD
//==========================================================================
void ScoreIchimoku(int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   double spanA=ichiSA[1], spanB=ichiSB[1];
   double cloudTop=MathMax(spanA,spanB);
   double cloudBot=MathMin(spanA,spanB);

   // Price above cloud
   if(price>cloudTop){B+=2;bR+="Ichi_AboveCloud|";}
   if(price<cloudBot){S+=2;sR+="Ichi_BelowCloud|";}

   // TK Cross (Tenkan > Kijun)
   if(ichiT[1]>ichiK[1]&&ichiT[2]<=ichiK[2]){B+=2;bR+="Ichi_TKBullCross|";}
   if(ichiT[1]<ichiK[1]&&ichiT[2]>=ichiK[2]){S+=2;sR+="Ichi_TKBearCross|";}

   // Kumo twist (future cloud)
   if(ichiSA[1]>ichiSB[1]&&ichiSA[2]<=ichiSB[2]){B+=1;bR+="Ichi_KumoTwistBull|";}
   if(ichiSA[1]<ichiSB[1]&&ichiSA[2]>=ichiSB[2]){S+=1;sR+="Ichi_KumoTwistBear|";}

   // Price vs Kijun
   if(price>ichiK[1]&&ichiK[1]>ichiK[2]){B+=1;bR+="Ichi_AboveKijun|";}
   if(price<ichiK[1]&&ichiK[1]<ichiK[2]){S+=1;sR+="Ichi_BelowKijun|";}
}

//==========================================================================
// â–“â–“ STRATEGY 19: PARABOLIC SAR
//==========================================================================
void ScoreParabolicSAR(int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   if(sarV[1]<price&&sarV[2]>price){B+=2;bR+="SAR_FlipBull|";}
   if(sarV[1]>price&&sarV[2]<price){S+=2;sR+="SAR_FlipBear|";}
   if(sarV[1]<price){B+=1;bR+="SAR_Bull|";}else{S+=1;sR+="SAR_Bear|";}
}

//==========================================================================
// â–“â–“ STRATEGY 20: FULL INDICATOR CONFLUENCE
//==========================================================================
void ScoreIndicators(int &B,int &S,string &bR,string &sR)
{
   double price=SymbolInfoDouble(_Symbol,SYMBOL_BID);

   // RSI
   if(rsi[1]<30&&rsi[1]>rsi[2]){B+=2;bR+="RSI_OsRecov|";}
   if(rsi[1]>70&&rsi[1]<rsi[2]){S+=2;sR+="RSI_ObRev|";}
   if(rsi[1]>50&&rsi[2]<50){B+=1;bR+="RSI_Cross50Up|";}
   if(rsi[1]<50&&rsi[2]>50){S+=1;sR+="RSI_Cross50Dn|";}

   // MACD
   if(macdM[1]>macdSig[1]&&macdM[2]<=macdSig[2]){B+=2;bR+="MACD_BullX|";}
   if(macdM[1]<macdSig[1]&&macdM[2]>=macdSig[2]){S+=2;sR+="MACD_BearX|";}
   if(macdM[1]>0&&macdSig[1]>0){B+=1;bR+="MACD_AboveZero|";}
   if(macdM[1]<0&&macdSig[1]<0){S+=1;sR+="MACD_BelowZero|";}

   // Stochastic
   if(stK[1]<20&&stK[1]>stD[1]&&stK[2]<=stD[2]){B+=2;bR+="Stoch_OsBullX|";}
   if(stK[1]>80&&stK[1]<stD[1]&&stK[2]>=stD[2]){S+=2;sR+="Stoch_ObBearX|";}

   // CCI
   if(cci[1]<-100&&cci[1]>cci[2]){B+=1;bR+="CCI_OsRecov|";}
   if(cci[1]>100&&cci[1]<cci[2]) {S+=1;sR+="CCI_ObRev|";}

   // Williams %R
   if(wpr[1]<-80&&wpr[1]>wpr[2]){B+=1;bR+="WPR_Os|";}
   if(wpr[1]>-20&&wpr[1]<wpr[2]){S+=1;sR+="WPR_Ob|";}

   // H4 RSI
   if(rsiH4[1]>55){B+=1;bR+="H4_RSI_Bull|";}
   if(rsiH4[1]<45){S+=1;sR+="H4_RSI_Bear|";}
}

//==========================================================================
// MARKET STRUCTURE
//==========================================================================
void AnalyzeMarketStructure(MqlRates &R[])
{
   double lSH=0,lSL=DBL_MAX,pSH=0,pSL=DBL_MAX;
   for(int i=2;i<SwingBars-2;i++)
   {
      bool iH=(R[i].high>R[i-1].high&&R[i].high>R[i+1].high&&R[i].high>R[i-2].high&&R[i].high>R[i+2].high);
      bool iL=(R[i].low<R[i-1].low&&R[i].low<R[i+1].low&&R[i].low<R[i-2].low&&R[i].low<R[i+2].low);
      if(iH){pSH=lSH;if(R[i].high>lSH)lSH=R[i].high;}
      if(iL){pSL=lSL;if(R[i].low<lSL)lSL=R[i].low;}
   }
   ms.lastSH=lSH; ms.lastSL=(lSL==DBL_MAX)?0:lSL;
   ms.prevSH=pSH; ms.prevSL=(pSL==DBL_MAX)?0:pSL;
   ms.pdHigh=lSH; ms.pdLow=ms.lastSL; ms.pdMid=(lSH+ms.lastSL)/2;

   double cur=R[1].close;
   ms.bos=false; ms.choch=false;
   if(cur>lSH&&lSH>0)      {ms.bullTrend=true; ms.bos=true;}
   else if(cur<ms.lastSL&&ms.lastSL>0){ms.bullTrend=false;ms.bos=true;}
   if(ms.bullTrend&&cur<ms.lastSL){ms.bullTrend=false;ms.choch=true;}
   if(!ms.bullTrend&&cur>lSH)     {ms.bullTrend=true; ms.choch=true;}
   if(!ms.bos&&!ms.choch) ms.bullTrend=(cur>(lSH+ms.lastSL)/2);
}

//==========================================================================
// LIQUIDITY ZONES
//==========================================================================
void DetectLiquidityZones(MqlRates &R[])
{
   liqHCount=0; liqLCount=0;
   for(int i=2;i<SwingBars-2&&liqHCount<30&&liqLCount<30;i++)
   {
      bool iH=(R[i].high>R[i-1].high&&R[i].high>R[i+1].high&&R[i].high>R[i-2].high&&R[i].high>R[i+2].high);
      bool iL=(R[i].low<R[i-1].low&&R[i].low<R[i+1].low&&R[i].low<R[i-2].low&&R[i].low<R[i+2].low);
      if(iH){liqHi[liqHCount].hi=R[i].high;liqHi[liqHCount].lo=R[i].high-ZoneBuffer;liqHi[liqHCount].valid=true;liqHCount++;}
      if(iL){liqLo[liqLCount].lo=R[i].low;liqLo[liqLCount].hi=R[i].low+ZoneBuffer;liqLo[liqLCount].valid=true;liqLCount++;}
   }
}

//==========================================================================
// ORDER BLOCKS
//==========================================================================
void DetectOrderBlocks(MqlRates &R[])
{
   bullOB.valid=false; bearOB.valid=false;
   for(int i=3;i<35;i++)
   {
      if(IsBear(R[i])&&!bullOB.valid)
      {
         bool imp=(R[i-1].close>R[i].high||R[i-2].close>R[i].high);
         if(imp){bullOB.hi=R[i].high;bullOB.lo=R[i].low;bullOB.mid=(R[i].high+R[i].low)/2;bullOB.isBull=true;bullOB.valid=true;}
      }
      if(IsBull(R[i])&&!bearOB.valid)
      {
         bool imp=(R[i-1].close<R[i].low||R[i-2].close<R[i].low);
         if(imp){bearOB.hi=R[i].high;bearOB.lo=R[i].low;bearOB.mid=(R[i].high+R[i].low)/2;bearOB.isBull=false;bearOB.valid=true;}
      }
   }
}

//==========================================================================
// FVG
//==========================================================================
void DetectFVG(MqlRates &R[])
{
   bullFVG.valid=false; bearFVG.valid=false;
   for(int i=2;i<35;i++)
   {
      if(R[i+1].high<R[i-1].low&&!bullFVG.valid)
      {if((R[i-1].low-R[i+1].high)/pip>=FVG_MinPips){bullFVG.hi=R[i-1].low;bullFVG.lo=R[i+1].high;bullFVG.isBull=true;bullFVG.valid=true;}}
      if(R[i+1].low>R[i-1].high&&!bearFVG.valid)
      {if((R[i+1].low-R[i-1].high)/pip>=FVG_MinPips){bearFVG.hi=R[i+1].low;bearFVG.lo=R[i-1].high;bearFVG.isBull=false;bearFVG.valid=true;}}
   }
}

//==========================================================================
// SUPPLY & DEMAND
//==========================================================================
void DetectSupplyDemand(MqlRates &R[])
{
   supplyCount=0; demandCount=0;
   for(int i=3;i<45&&supplyCount<15&&demandCount<15;i++)
   {
      if(IsBear(R[i+1])&&IsBear(R[i])&&IsBull(R[i-1])&&Body(R[i-1])>Body(R[i])*1.5)
      {demandZone[demandCount].lo=MathMin(R[i].low,R[i+1].low);demandZone[demandCount].hi=MathMax(R[i].high,R[i+1].high);demandZone[demandCount].valid=true;demandCount++;}
      if(IsBull(R[i+1])&&IsBull(R[i])&&IsBear(R[i-1])&&Body(R[i-1])>Body(R[i])*1.5)
      {supplyZone[supplyCount].lo=MathMin(R[i].low,R[i+1].low);supplyZone[supplyCount].hi=MathMax(R[i].high,R[i+1].high);supplyZone[supplyCount].valid=true;supplyCount++;}
   }
}

//==========================================================================
// WYCKOFF
//==========================================================================
void AnalyzeWyckoff(MqlRates &R[])
{
   wyck.accum=wyck.distrib=wyck.spring=wyck.upthrust=wyck.UTAD=wyck.ST=wyck.AR=false;
   double hi=0,lo=DBL_MAX;
   for(int i=1;i<=25;i++){if(R[i].high>hi)hi=R[i].high;if(R[i].low<lo)lo=R[i].low;}
   double range=(hi-lo);
   if(range<=0) return;
   bool tight=(range/lo*100<2.0);
   bool afterDown=(R[20].close>R[10].close&&R[10].close>R[3].close)?false:true;
   bool afterUp  =(R[20].close<R[10].close&&R[10].close<R[3].close)?false:true;

   if(tight&&afterDown) wyck.accum=true;
   if(tight&&afterUp)   wyck.distrib=true;

   if(wyck.accum&&R[1].low<lo&&R[1].close>lo)    wyck.spring=true;
   if(wyck.distrib&&R[1].high>hi&&R[1].close<hi) wyck.upthrust=true;

   // UTAD: Up Thrust After Distribution
   if(wyck.distrib&&R[1].high>hi&&R[1].close>hi&&R[2].close<R[2].open) wyck.UTAD=true;

   // Secondary Test (ST)
   if(wyck.accum&&MathAbs(R[1].low-lo)<=atr[1]*0.5&&(double)R[1].tick_volume<(double)R[5].tick_volume)
      wyck.ST=true;

   // Automatic Rally
   if(wyck.accum&&R[1].close>R[1].open&&(double)R[1].tick_volume>(double)R[3].tick_volume*1.5)
      wyck.AR=true;
}

//==========================================================================
// FIBONACCI
//==========================================================================
void CalcFibonacci(MqlRates &R[])
{
   fib.valid=false;
   double swH=0,swL=DBL_MAX;
   for(int i=1;i<=60;i++){if(R[i].high>swH)swH=R[i].high;if(R[i].low<swL)swL=R[i].low;}
   if(swH<=0||swL==DBL_MAX||swH<=swL) return;
   double d=swH-swL;
   if(ms.bullTrend)
   {
      fib.f0=swH; fib.f100=swL;
      fib.f236=swH-d*0.236; fib.f382=swH-d*0.382;
      fib.f500=swH-d*0.500; fib.f618=swH-d*0.618;
      fib.f705=swH-d*0.705; fib.f786=swH-d*0.786;
   }
   else
   {
      fib.f0=swL; fib.f100=swH;
      fib.f236=swL+d*0.236; fib.f382=swL+d*0.382;
      fib.f500=swL+d*0.500; fib.f618=swL+d*0.618;
      fib.f705=swL+d*0.705; fib.f786=swL+d*0.786;
   }
   fib.valid=true;
}

//==========================================================================
// RSI DIVERGENCE
//==========================================================================
void DetectRSIDivergence(MqlRates &R[])
{
   rsidiv.bullDiv=false; rsidiv.bearDiv=false;
   // Bullish: lower price low but higher RSI low
   if(R[1].low<R[5].low&&rsi[1]>rsi[5]&&rsi[1]<40) rsidiv.bullDiv=true;
   // Bearish: higher price high but lower RSI high
   if(R[1].high>R[5].high&&rsi[1]<rsi[5]&&rsi[1]>60) rsidiv.bearDiv=true;
   // Hidden Bull: higher low price + lower RSI
   if(R[1].low>R[5].low&&rsi[1]<rsi[5]&&ms.bullTrend) rsidiv.bullDiv=true;
   // Hidden Bear: lower high price + higher RSI
   if(R[1].high<R[5].high&&rsi[1]>rsi[5]&&!ms.bullTrend) rsidiv.bearDiv=true;
}

//==========================================================================
// EXECUTE
//==========================================================================
void ExecuteBuy()
{
   double ask=SymbolInfoDouble(_Symbol,SYMBOL_ASK);
   double tp=NormalizeDouble(ask*(1.0+TakeProfitPct/100.0),digits);
   double sl=NormalizeDouble(ask*(1.0-StopLossPct/100.0),digits);
   double rr=TakeProfitPct/StopLossPct;

   if(trade.Buy(LotSize,_Symbol,ask,sl,tp,"UltimatePRO v6 BUY"))
   {
      todayTrades++;
      Print("â•”â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•—");
      Print("â•‘  âœ…  BUY TRADE OPENED        â•‘");
      Print("â•‘  Price  : ", NormalizeDouble(ask,digits));
      Print("â•‘  SL     : ", NormalizeDouble(sl,digits),"  (-",StopLossPct,"%)");
      Print("â•‘  TP     : ", NormalizeDouble(tp,digits),"  (+",TakeProfitPct,"%)");
      Print("â•‘  R:R    : 1:", NormalizeDouble(rr,2));
      Print("â•‘  Lot    : ",LotSize,"  | Magic: ",MagicNumber);
      Print("â•šâ•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•");
   }
   else
      Print("âŒ BUY FAILED | Error: ",GetLastError());
}

void ExecuteSell()
{
   double bid=SymbolInfoDouble(_Symbol,SYMBOL_BID);
   double tp=NormalizeDouble(bid*(1.0-TakeProfitPct/100.0),digits);
   double sl=NormalizeDouble(bid*(1.0+StopLossPct/100.0),digits);
   double rr=TakeProfitPct/StopLossPct;

   if(trade.Sell(LotSize,_Symbol,bid,sl,tp,"UltimatePRO v6 SELL"))
   {
      todayTrades++;
      Print("â•”â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•—");
      Print("â•‘  âœ…  SELL TRADE OPENED       â•‘");
      Print("â•‘  Price  : ", NormalizeDouble(bid,digits));
      Print("â•‘  SL     : ", NormalizeDouble(sl,digits),"  (+",StopLossPct,"%)");
      Print("â•‘  TP     : ", NormalizeDouble(tp,digits),"  (-",TakeProfitPct,"%)");
      Print("â•‘  R:R    : 1:", NormalizeDouble(rr,2));
      Print("â•‘  Lot    : ",LotSize,"  | Magic: ",MagicNumber);
      Print("â•šâ•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•");
   }
   else
      Print("âŒ SELL FAILED | Error: ",GetLastError());
}

//==========================================================================
// â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â• FULL CANDLESTICK LIBRARY â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
//==========================================================================
double Body(MqlRates &c)  {return MathAbs(c.close-c.open);}
double Range(MqlRates &c) {return c.high-c.low;}
double UpW(MqlRates &c)   {return c.high-MathMax(c.open,c.close);}
double LoW(MqlRates &c)   {return MathMin(c.open,c.close)-c.low;}
bool   IsBull(MqlRates &c){return c.close>=c.open;}
bool   IsBear(MqlRates &c){return c.close<c.open;}
bool   SmBody(MqlRates &c){return Range(c)>0&&Body(c)/Range(c)<=0.20;}
bool   IsDoji(MqlRates &c){return Range(c)>0&&Body(c)/Range(c)<=0.10;}

// Helper for EMA ribbon scoring (gets last candle via buffer)
MqlRates R_Last_buf;
MqlRates R_Last() { MqlRates tmp[]; ArraySetAsSeries(tmp,true); CopyRates(_Symbol,TF_Entry,1,1,tmp); if(ArraySize(tmp)>0)return tmp[0]; return R_Last_buf; }

// SINGLE CANDLE
bool IsHammer(MqlRates &c)       {return Range(c)>0&&IsBull(c)&&LoW(c)>=Body(c)*2.0&&UpW(c)<=Body(c)*0.5;}
bool IsShootingStar(MqlRates &c) {return Range(c)>0&&IsBear(c)&&UpW(c)>=Body(c)*2.0&&LoW(c)<=Body(c)*0.5;}
bool IsInvHammer(MqlRates &c)    {return Range(c)>0&&IsBull(c)&&UpW(c)>=Body(c)*2.0&&LoW(c)<=Body(c)*0.5;}
bool IsHangingMan(MqlRates &c)   {return Range(c)>0&&IsBear(c)&&LoW(c)>=Body(c)*2.0&&UpW(c)<=Body(c)*0.5;}
bool IsDragonflyDoji(MqlRates &c){return IsDoji(c)&&LoW(c)>=Range(c)*0.65&&UpW(c)<=Range(c)*0.05;}
bool IsGravestoneDoji(MqlRates &c){return IsDoji(c)&&UpW(c)>=Range(c)*0.65&&LoW(c)<=Range(c)*0.05;}
bool IsLLDoji(MqlRates &c)       {return IsDoji(c)&&UpW(c)>=Range(c)*0.28&&LoW(c)>=Range(c)*0.28;}
bool IsBullMarubozu(MqlRates &c) {return Range(c)>0&&IsBull(c)&&Body(c)/Range(c)>=0.90;}
bool IsBearMarubozu(MqlRates &c) {return Range(c)>0&&IsBear(c)&&Body(c)/Range(c)>=0.90;}
bool IsBullBelt(MqlRates &c)     {return IsBull(c)&&LoW(c)<=_Point*2&&Body(c)/Range(c)>=0.70;}
bool IsBearBelt(MqlRates &c)     {return IsBear(c)&&UpW(c)<=_Point*2&&Body(c)/Range(c)>=0.70;}
bool IsSpinTop(MqlRates &c)      {return Range(c)>0&&Body(c)/Range(c)>=0.10&&Body(c)/Range(c)<=0.30&&UpW(c)>=Body(c)&&LoW(c)>=Body(c);}
bool IsHighWave(MqlRates &c)     {return IsDoji(c)&&UpW(c)>=Range(c)*0.35&&LoW(c)>=Range(c)*0.35;}
bool IsRicketsSeat(MqlRates &c)  {return IsBull(c)&&LoW(c)>=Range(c)*0.40&&UpW(c)>=Range(c)*0.20&&Body(c)/Range(c)<=0.40;}

// TWO CANDLE
bool IsBullEngulf(MqlRates &c,MqlRates &p)  {return IsBear(p)&&IsBull(c)&&c.open<p.close&&c.close>p.open&&Body(c)>Body(p);}
bool IsBearEngulf(MqlRates &c,MqlRates &p)  {return IsBull(p)&&IsBear(c)&&c.open>p.close&&c.close<p.open&&Body(c)>Body(p);}
bool IsBullHarami(MqlRates &c,MqlRates &p)  {return IsBear(p)&&IsBull(c)&&c.open>p.close&&c.close<p.open&&Body(c)<Body(p)*0.6;}
bool IsBearHarami(MqlRates &c,MqlRates &p)  {return IsBull(p)&&IsBear(c)&&c.open<p.close&&c.close>p.open&&Body(c)<Body(p)*0.6;}
bool IsBullHaramiX(MqlRates &c,MqlRates &p) {return IsBear(p)&&IsDoji(c)&&c.high<p.open&&c.low>p.close;}
bool IsBearHaramiX(MqlRates &c,MqlRates &p) {return IsBull(p)&&IsDoji(c)&&c.high<p.close&&c.low>p.open;}
bool IsTweezerBot(MqlRates &c,MqlRates &p)  {return IsBear(p)&&IsBull(c)&&MathAbs(c.low-p.low)<=pip*2;}
bool IsTweezerTop(MqlRates &c,MqlRates &p)  {return IsBull(p)&&IsBear(c)&&MathAbs(c.high-p.high)<=pip*2;}
bool IsPiercing(MqlRates &c,MqlRates &p)    {return IsBear(p)&&IsBull(c)&&c.open<p.close&&c.close>(p.open+Body(p)*0.5)&&c.close<p.open;}
bool IsDarkCloud(MqlRates &c,MqlRates &p)   {return IsBull(p)&&IsBear(c)&&c.open>p.close&&c.close<(p.open+Body(p)*0.5)&&c.close>p.open;}
bool IsOnNeck(MqlRates &c,MqlRates &p)      {return IsBear(p)&&IsBull(c)&&MathAbs(c.close-p.low)<=pip*3;}
bool IsInNeck(MqlRates &c,MqlRates &p)      {return IsBear(p)&&IsBull(c)&&c.close>p.low&&c.close<(p.open+p.close)/2;}
bool IsThrusting(MqlRates &c,MqlRates &p)   {return IsBear(p)&&IsBull(c)&&c.open<p.low&&c.close<(p.open+p.close)/2&&c.close>p.close;}
bool IsKicking(MqlRates &c,MqlRates &p)     {return (IsBullMarubozu(p)&&IsBearMarubozu(c))||(IsBearMarubozu(p)&&IsBullMarubozu(c));}
bool IsCounterAttackB(MqlRates &c,MqlRates &p){return IsBear(p)&&IsBull(c)&&MathAbs(c.close-p.close)<=pip*4&&Body(c)>pip*5;}
bool IsCounterAttackS(MqlRates &c,MqlRates &p){return IsBull(p)&&IsBear(c)&&MathAbs(c.close-p.close)<=pip*4&&Body(c)>pip*5;}
bool IsMatchingLow(MqlRates &c,MqlRates &p) {return IsBear(p)&&IsBear(c)&&MathAbs(c.close-p.close)<=pip*2;}
bool IsMatchingHigh(MqlRates &c,MqlRates &p){return IsBull(p)&&IsBull(c)&&MathAbs(c.close-p.close)<=pip*2;}
bool IsHomingPigeon(MqlRates &c,MqlRates &p){return IsBear(p)&&IsBear(c)&&c.open>p.close&&c.close<p.open&&Body(c)<Body(p)*0.5;}
bool IsConcealBabySwallow(MqlRates &c,MqlRates &p){return IsBearMarubozu(p)&&IsBear(c)&&c.open>p.close&&c.low<p.close&&c.close>p.open;}
bool IsSeparatingLines(MqlRates &c,MqlRates &p){return IsBull(p)!=IsBull(c)&&MathAbs(c.open-p.open)<=pip*3&&Body(c)>pip*8;}

// THREE CANDLE
bool IsMorningStar(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(p)&&SmBody(m)&&IsBull(c)&&m.high<p.close&&c.close>p.open+Body(p)*0.5;}
bool IsEveningStar(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&SmBody(m)&&IsBear(c)&&m.low>p.close&&c.close<p.open+Body(p)*0.5;}
bool IsMorningDojiStar(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(p)&&IsDoji(m)&&IsBull(c)&&c.close>p.open+Body(p)*0.5;}
bool IsEveningDojiStar(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsDoji(m)&&IsBear(c)&&c.close<p.open+Body(p)*0.5;}
bool Is3WS(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(c)&&IsBull(m)&&IsBull(p)&&c.open>m.open&&c.open<m.close&&m.open>p.open&&m.open<p.close;}
bool Is3BC(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(c)&&IsBear(m)&&IsBear(p)&&c.open<m.open&&c.open>m.close&&m.open<p.open&&m.open>p.close;}
bool Is3IU(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(p)&&IsBullHarami(m,p)&&IsBull(c)&&c.close>p.open;}
bool Is3ID(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsBearHarami(m,p)&&IsBear(c)&&c.close<p.open;}
bool Is3OU(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(p)&&IsBullEngulf(m,p)&&IsBull(c)&&c.close>m.close;}
bool Is3OD(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsBearEngulf(m,p)&&IsBear(c)&&c.close<m.close;}
bool IsAbandonedBabyB(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(p)&&IsDoji(m)&&IsBull(c)&&m.high<p.low&&m.high<c.low;}
bool IsAbandonedBabyS(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsDoji(m)&&IsBear(c)&&m.low>p.high&&m.low>c.high;}
bool IsUpsideGap2Crows(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsBear(m)&&IsBear(c)&&m.low>p.high&&c.open>m.open&&c.close<m.open&&c.close>p.close;}
bool IsDeliberation(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsBull(m)&&SmBody(c)&&c.open>=m.close;}
bool IsAdvanceBlock(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsBull(m)&&IsBull(c)&&Body(c)<Body(m)&&Body(m)<Body(p)&&UpW(c)>UpW(p);}
bool IsStalled(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsBull(m)&&SmBody(c)&&c.high>=m.high&&Body(p)>Body(m)*2;}
bool IsIdenticalBC(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(p)&&IsBear(m)&&IsBear(c)&&MathAbs(c.open-m.close)<=pip*2&&MathAbs(m.open-p.close)<=pip*2;}
bool IsBullTriStar(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsDoji(p)&&IsDoji(m)&&IsDoji(c)&&m.low<p.low&&m.low<c.low;}
bool IsBearTriStar(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsDoji(p)&&IsDoji(m)&&IsDoji(c)&&m.high>p.high&&m.high>c.high;}
bool IsDojiStar(MqlRates &c,MqlRates &m,MqlRates &p)
   {return (IsBull(p)||IsBear(p))&&IsDoji(m)&&(IsBull(c)||IsBear(c));}
bool IsBreakaway_Bull(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(p)&&IsBear(m)&&IsBull(c)&&c.open>p.close&&c.close>p.open;}
bool IsBreakaway_Bear(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsBull(m)&&IsBear(c)&&c.open<p.close&&c.close<p.open;}
bool IsMatHold(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBull(p)&&IsBear(m)&&IsBull(c)&&m.high<p.close&&c.close>p.close;}
bool IsRisingSun(MqlRates &c,MqlRates &m,MqlRates &p)
   {return IsBear(p)&&IsDoji(m)&&IsBull(c)&&c.close>m.high&&c.open<m.low;}

// FOUR/FIVE CANDLE
bool Is3LS_Bull(MqlRates &c,MqlRates &c1,MqlRates &c2,MqlRates &c3)
   {return IsBear(c3)&&IsBear(c2)&&IsBear(c1)&&IsBull(c)&&c.open<c1.close&&c.close>c3.open;}
bool Is3LS_Bear(MqlRates &c,MqlRates &c1,MqlRates &c2,MqlRates &c3)
   {return IsBull(c3)&&IsBull(c2)&&IsBull(c1)&&IsBear(c)&&c.open>c1.close&&c.close<c3.open;}
bool IsRising3(MqlRates &c,MqlRates &c1,MqlRates &c2,MqlRates &c3,MqlRates &c4)
   {return IsBull(c4)&&IsBear(c3)&&IsBear(c2)&&IsBear(c1)&&IsBull(c)&&c3.high<c4.close&&c1.low>c4.open&&c.close>c4.close;}
bool IsFalling3(MqlRates &c,MqlRates &c1,MqlRates &c2,MqlRates &c3,MqlRates &c4)
   {return IsBear(c4)&&IsBull(c3)&&IsBull(c2)&&IsBull(c1)&&IsBear(c)&&c3.low>c4.close&&c1.high<c4.open&&c.close<c4.close;}
bool IsLadderBottom(MqlRates &c,MqlRates &c1,MqlRates &c2,MqlRates &c3,MqlRates &c4)
   {return IsBear(c4)&&IsBear(c3)&&IsBear(c2)&&IsBear(c1)&&IsBull(c)&&c.close>c1.open&&Body(c)>Body(c1);}
bool IsLadderTop(MqlRates &c,MqlRates &c1,MqlRates &c2,MqlRates &c3,MqlRates &c4)
   {return IsBull(c4)&&IsBull(c3)&&IsBull(c2)&&IsBull(c1)&&IsBear(c)&&c.close<c1.open&&Body(c)>Body(c1);}

//==========================================================================
// UTILITIES
//==========================================================================
bool InSession()
{
   int h=GetHour();
   return (h>=LondonStart&&h<LondonEnd)||(h>=NYStart&&h<NYEnd);
}
bool IsDay(int d){MqlDateTime t;TimeToStruct(TimeCurrent(),t);return t.day_of_week==d;}
int  GetHour()  {MqlDateTime t;TimeToStruct(TimeCurrent(),t);return t.hour;}
int  OpenPositions()
{
   int n=0;
   for(int i=PositionsTotal()-1;i>=0;i--)
      if(pos.SelectByIndex(i)&&pos.Symbol()==_Symbol&&pos.Magic()==MagicNumber) n++;
   return n;
}
void ResetDailyCounter()
{
   MqlDateTime now,last;
   TimeToStruct(TimeCurrent(),now); TimeToStruct(lastTradeDay,last);
   if(now.day!=last.day||now.mon!=last.mon){todayTrades=0;lastTradeDay=TimeCurrent();}
}
//+------------------------------------------------------------------+
// END â€” ULTIMATE TRADING ROBOT ELITE v6.0
//+------------------------------------------------------------------+
