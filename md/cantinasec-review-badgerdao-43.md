# [Partial liquidation and leaving bad debt undistributed can be optimal for liquidators to maximize incentives received from other CDPs](https://github.com/cantinasec/review-badgerdao/issues/43)

> state: **open** opened by: **dmitriia** on: **10/5/2023**

**Context:** [LiquidationLibrary.sol#L539-L554](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/LiquidationLibrary.sol#L539-L554)

**Description:** Liquidators have incentives to avoid fully liquidating the bad CDPs until the very end of their liquidation sequence. This way they leave bad debt undistributed and maximize ICRs of the best liquidable CDPs, receiving more liquidation incentives, which linearly depend on the ICR when `MCR > ICR > LICR`.

On the same grounds as the issue ["Liquidator can receive outsized incentives by aiming the best liquidable CDPs first"](https://github.com/cantinasec/review-badgerdao/issues/42), it is also profitable for a liquidator to perform only partial liquidation of the worst CDP containing bad debt to receive `3%` of the maximal debt that can be burned partially, leaving only minimal collateral and the full bad debt part:

```solidity
function _calculateSurplusAndCap(
    // ...
)
    private
    view
    returns (uint256 cappedColPortion, uint256 collSurplus, uint256 debtToRedistribute)
{
    // Calculate liquidation incentive for liquidator:
    // If ICR is less than 103%: give away 103% worth of collateral to liquidator, i.e., repaidDebt * 103% / price
    // If ICR is more than 103%: give away min(ICR, 110%) worth of collateral to liquidator, i.e., repaidDebt * min(ICR, 110%) / price
    uint256 _incentiveColl;
    if (_ICR > LICR) {
        _incentiveColl = (_totalDebtToBurn * (_ICR > MCR ? MCR : _ICR)) / _price;
    } else {
        if (_fullLiquidation) {
            // for full liquidation, there would be some bad debt to redistribute
            _incentiveColl = collateral.getPooledEthByShares(_totalColToSend);
            uint256 _debtToRepay = (_incentiveColl * _price) / LICR;
            debtToRedistribute = _debtToRepay < _totalDebtToBurn
                ? _totalDebtToBurn - _debtToRepay
                : 0;
        } else {
            // for partial liquidation, new ICR would deteriorate
            // since we give more incentive (103%) than current _ICR allowed
            // See the line below
            _incentiveColl = (_totalDebtToBurn * LICR) / _price;
        }
    }
    cappedColPortion = collateral.getSharesByPooledEth(_incentiveColl);
    cappedColPortion = cappedColPortion < _totalColToSend ? cappedColPortion : _totalColToSend;
    collSurplus = (cappedColPortion == _totalColToSend) ? 0 : _totalColToSend - cappedColPortion;
}
```

There is a minimal debt/collatertal requirement in `partiallyLiquidate()`, that forms a boundary condition for the strategy, i.e. one has to leave enough debt at CDP to satisfy `MIN_NET_COLL` based condition:

```solidity
function _requirePartialLiqDebtSize(
    uint256 _partialDebt,
    uint256 _entireDebt,
    uint256 _price
) internal view {
    require(
        // See the line below
        (_partialDebt + _convertDebtDenominationToBtc(MIN_NET_COLL, _price)) <= _entireDebt,
        "LiquidationLibrary: Partial debt liquidated must be less than total debt"
    );
}
```

The correspondence between liquidator reward obtainable on full liquidation and the profit from keeping bad debt undistributed and ICRs being higher depends on CDP sizes and given big enough CDPs liquidator reward can not matter that much and be forfeited until the end. I.e. risk of losing it if someone else performs full liquidation, if liquidations are performed non-atomically, can be small compared to indirect profit obtainable from keeping bad debt intact.

Impact: similarly liquidation incentives will be maximized in excess of protocol logic and actual pool state, not only due to good ones being liquidated first, but also due to bad ones being liquidated only partially.

Per medium likelihood and impact setting the severity to be medium.

**Recommendation:** It looks like requiring that CDP being liquidated has to be the worst one is needed, as otherwise there be a hidden bad debt part remaining.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/43#issuecomment-1749081968) on: **10/5/2023**

Has been a while but I remember modelling Partial Liquidations / 3% fixed premium bad debt

https://github.com/GalloDaSballo/Cdp-Demo/blob/main/scripts/simple_debt_reabsorption.py

https://github.com/GalloDaSballo/Cdp-Demo/blob/main/scripts/insolvency_cascade.py
---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/43#issuecomment-1749161068) on: **10/5/2023**

IMO partial liquidation is designed to target "BIG" CDP. 

Considering the liquidation itself is highly competitive in DeFi, if the CDP is big enough then it means the incentive should be also quite attractive for potential liquidators: from MCR(`110%`) all the way down to LICR(`103%`). It could be a really hard choice for reasonable liquidators to just wait until last minute to bear the risk losing incentive all (others might liquidate first).

If the ICR already goes down below LICR(`103%`), it doesn't matter now if it is a full liquidation or partial liquidation, the incentive is always 3%. Unless the liquidator could guarantee it will win the competition of liquidation on other CDPs, otherwise the best strategy might be trying to extract the most incentive it could with current liquidation, i.e., full liquidation is a better choice than partial liquidation in terms of total incentive size. "Seize the day(CDP)"

For the suggestion of "CDP being liquidated has to be the worst one is needed", my two cents is here https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1750010805 
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/43#issuecomment-1752111663) on: **10/8/2023**

I don't believe that bad debt can be influnced by whether you perform a partial or a full liquidation, barring some rounding

Based on the CR of the CDP, and a 3% premium

Coll and Debt will be burned at 3% discount until all Coll is expired

Meaning the bad debt should be (formula for sure wrong due to compounding)
The 3% of the collRedeemed + the remaining debt that had no coll left


---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/43#issuecomment-1752116324) on: **10/8/2023**

```python
Simulate Total Liquidation with Bad debt
Coll is always 100
No price means 1 coll = 1 debt for price



DEBT 100.0
cr 100.0
DEBT 100.0
COLL 100
DEBT 2.9126213592233086



DEBT 111.11111111111111
cr 90.0
DEBT 111.11111111111111
COLL 100
DEBT 14.023732470334423



DEBT 125.0
cr 80.0
DEBT 125.0
COLL 100
DEBT 27.91262135922331



DEBT 142.85714285714286
cr 70.0
DEBT 142.85714285714286
COLL 100
DEBT 45.76976421636617
```
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/43#issuecomment-1752116384) on: **10/8/2023**

```python 
Simulate Partial Liquidation with Bad debt
Coll is always 100
No price means 1 coll = 1 debt for price



DEBT 100.0
cr 100.0
DEBT 100.0
COLL 100
DEBT 90.29126213592232
COLL 90.0
DEBT 81.55339805825243
COLL 81.0
DEBT 73.68932038834951
COLL 72.9
DEBT 66.6116504854369
COLL 65.61
DEBT 60.24174757281554
COLL 59.049
DEBT 54.508834951456315
COLL 53.1441
DEBT 49.34921359223301
COLL 47.82969
DEBT 44.70555436893204
COLL 43.046721
DEBT 40.526261067961165
COLL 38.7420489
DEBT 36.76489709708738
COLL 34.86784401
DEBT 33.37966952330097
COLL 31.381059608999998
DEBT 30.3329647068932
COLL 28.2429536481
DEBT 27.59093037212621
COLL 25.41865828329
DEBT 25.123099470835918
COLL 22.876792454961
DEBT 22.902051659674655
COLL 20.589113209464898
DEBT 20.903108629629518
COLL 18.53020188851841
DEBT 19.104059902588897
COLL 16.67718169966657
DEBT 17.484916048252337
COLL 15.009463529699913
DEBT 16.027686579349435
COLL 13.50851717672992
DEBT 14.716180057336821
COLL 12.157665459056929
DEBT 13.535824187525469
COLL 10.941898913151237
DEBT 12.473503904695251
COLL 9.847709021836113
DEBT 11.517415650148056
COLL 8.862938119652503
DEBT 10.65693622105558
COLL 7.976644307687252
DEBT 9.88250473487235
COLL 7.178979876918527
DEBT 9.185516397307445
COLL 6.461081889226675
DEBT 8.55822689349903
COLL 5.814973700304007
DEBT 7.993666340071457
COLL 5.233476330273606
DEBT 7.485561841986641
COLL 4.710128697246246
DEBT 7.028267793710307
COLL 4.239115827521621
DEBT 6.616703150261606
COLL 3.815204244769459
DEBT 6.246294971157775
COLL 3.4336838202925133
DEBT 5.912927609964328
COLL 3.090315438263262
DEBT 5.612896984890225
COLL 2.781283894436936
DEBT 5.342869422323532
COLL 2.5031555049932424
DEBT 5.099844616013509
COLL 2.2528399544939184
DEBT 4.8811222903344875
COLL 2.0275559590445265
DEBT 4.684272197223368



DEBT 111.11111111111111
cr 90.0
DEBT 111.11111111111111
COLL 100
DEBT 101.40237324703344
COLL 90.0
DEBT 92.66450916936354
COLL 81.0
DEBT 84.80043149946063
COLL 72.9
DEBT 77.72276159654801
COLL 65.61
DEBT 71.35285868392666
COLL 59.049
DEBT 65.61994606256744
COLL 53.1441
DEBT 60.460324703344135
COLL 47.82969
DEBT 55.81666548004316
COLL 43.046721
DEBT 51.63737217907229
COLL 38.7420489
DEBT 47.8760082081985
COLL 34.86784401
DEBT 44.49078063441209
COLL 31.381059608999998
DEBT 41.44407581800432
COLL 28.2429536481
DEBT 38.70204148323733
COLL 25.41865828329
DEBT 36.23421058194704
COLL 22.876792454961
DEBT 34.013162770785776
COLL 20.589113209464898
DEBT 32.01421974074064
COLL 18.53020188851841
DEBT 30.215171013700022
COLL 16.67718169966657
DEBT 28.596027159363462
COLL 15.009463529699913
DEBT 27.138797690460557
COLL 13.50851717672992
DEBT 25.827291168447942
COLL 12.157665459056929
DEBT 24.646935298636592
COLL 10.941898913151237
DEBT 23.584615015806374
COLL 9.847709021836113
DEBT 22.628526761259177
COLL 8.862938119652503
DEBT 21.768047332166702
COLL 7.976644307687252
DEBT 20.993615845983474
COLL 7.178979876918527
DEBT 20.296627508418567
COLL 6.461081889226675
DEBT 19.66933800461015
COLL 5.814973700304007
DEBT 19.104777451182578
COLL 5.233476330273606
DEBT 18.596672953097762
COLL 4.710128697246246
DEBT 18.139378904821427
COLL 4.239115827521621
DEBT 17.727814261372725
COLL 3.815204244769459
DEBT 17.357406082268895
COLL 3.4336838202925133
DEBT 17.024038721075446
COLL 3.090315438263262
DEBT 16.72400809600134
COLL 2.781283894436936
DEBT 16.45398053343465
COLL 2.5031555049932424
DEBT 16.210955727124624
COLL 2.2528399544939184
DEBT 15.992233401445603
COLL 2.0275559590445265
DEBT 15.795383308334484



DEBT 125.0
cr 80.0
DEBT 125.0
COLL 100
DEBT 115.29126213592232
COLL 90.0
DEBT 106.55339805825243
COLL 81.0
DEBT 98.68932038834951
COLL 72.9
DEBT 91.6116504854369
COLL 65.61
DEBT 85.24174757281554
COLL 59.049
DEBT 79.50883495145632
COLL 53.1441
DEBT 74.34921359223301
COLL 47.82969
DEBT 69.70555436893204
COLL 43.046721
DEBT 65.52626106796117
COLL 38.7420489
DEBT 61.764897097087385
COLL 34.86784401
DEBT 58.379669523300976
COLL 31.381059608999998
DEBT 55.33296470689321
COLL 28.2429536481
DEBT 52.590930372126216
COLL 25.41865828329
DEBT 50.123099470835925
COLL 22.876792454961
DEBT 47.90205165967466
COLL 20.589113209464898
DEBT 45.90310862962953
COLL 18.53020188851841
DEBT 44.10405990258891
COLL 16.67718169966657
DEBT 42.484916048252344
COLL 15.009463529699913
DEBT 41.02768657934944
COLL 13.50851717672992
DEBT 39.71618005733683
COLL 12.157665459056929
DEBT 38.53582418752548
COLL 10.941898913151237
DEBT 37.473503904695264
COLL 9.847709021836113
DEBT 36.51741565014807
COLL 8.862938119652503
DEBT 35.65693622105559
COLL 7.976644307687252
DEBT 34.88250473487236
COLL 7.178979876918527
DEBT 34.18551639730746
COLL 6.461081889226675
DEBT 33.55822689349905
COLL 5.814973700304007
DEBT 32.99366634007147
COLL 5.233476330273606
DEBT 32.48556184198665
COLL 4.710128697246246
DEBT 32.02826779371032
COLL 4.239115827521621
DEBT 31.616703150261618
COLL 3.815204244769459
DEBT 31.246294971157788
COLL 3.4336838202925133
DEBT 30.91292760996434
COLL 3.090315438263262
DEBT 30.612896984890234
COLL 2.781283894436936
DEBT 30.342869422323542
COLL 2.5031555049932424
DEBT 30.099844616013517
COLL 2.2528399544939184
DEBT 29.881122290334496
COLL 2.0275559590445265
DEBT 29.684272197223375



DEBT 142.85714285714286
cr 70.0
DEBT 142.85714285714286
COLL 100
DEBT 133.14840499306518
COLL 90.0
DEBT 124.41054091539529
COLL 81.0
DEBT 116.54646324549238
COLL 72.9
DEBT 109.46879334257976
COLL 65.61
DEBT 103.0988904299584
COLL 59.049
DEBT 97.36597780859918
COLL 53.1441
DEBT 92.20635644937587
COLL 47.82969
DEBT 87.5626972260749
COLL 43.046721
DEBT 83.38340392510403
COLL 38.7420489
DEBT 79.62203995423025
COLL 34.86784401
DEBT 76.23681238044384
COLL 31.381059608999998
DEBT 73.19010756403607
COLL 28.2429536481
DEBT 70.44807322926908
COLL 25.41865828329
DEBT 67.9802423279788
COLL 22.876792454961
DEBT 65.75919451681754
COLL 20.589113209464898
DEBT 63.76025148677241
COLL 18.53020188851841
DEBT 61.96120275973179
COLL 16.67718169966657
DEBT 60.342058905395234
COLL 15.009463529699913
DEBT 58.88482943649233
COLL 13.50851717672992
DEBT 57.57332291447972
COLL 12.157665459056929
DEBT 56.39296704466837
COLL 10.941898913151237
DEBT 55.33064676183815
COLL 9.847709021836113
DEBT 54.374558507290956
COLL 8.862938119652503
DEBT 53.51407907819848
COLL 7.976644307687252
DEBT 52.73964759201525
COLL 7.178979876918527
DEBT 52.04265925445035
COLL 6.461081889226675
DEBT 51.415369750641936
COLL 5.814973700304007
DEBT 50.85080919721436
COLL 5.233476330273606
DEBT 50.34270469912954
COLL 4.710128697246246
DEBT 49.88541065085321
COLL 4.239115827521621
DEBT 49.47384600740451
COLL 3.815204244769459
DEBT 49.10343782830068
COLL 3.4336838202925133
DEBT 48.77007046710723
COLL 3.090315438263262
DEBT 48.47003984203313
COLL 2.781283894436936
DEBT 48.200012279466435
COLL 2.5031555049932424
DEBT 47.95698747315641
COLL 2.2528399544939184
DEBT 47.738265147477385
COLL 2.0275559590445265
DEBT 47.54141505436627
```
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/43#issuecomment-1752116672) on: **10/8/2023**

Source 

```python

MAX_BPS = 10_000

def cr(debt, coll):
    return coll / debt * 100

def getDebtByCr(coll, cr_bps):
   return coll / cr_bps * MAX_BPS

LICR = 103_00


def max(a, b):
   if(a > b):
      return a
   return b

def min(a, b):
   if(a > b):
      return b
   return a

def full_liq(COLL):
  ## burn all
  return [COLL, COLL / LICR * MAX_BPS]

def partial_liq(COLL, percent):
   return [COLL * percent / 100, COLL * percent / 100 / LICR * MAX_BPS]

def loop_full(cr_bps):
  COLL = 100
  DEBT = getDebtByCr(COLL, cr_bps)
  print("DEBT", DEBT)
  print("cr", cr(DEBT, COLL))

  ## Liquidate Partially based on premium
  while(COLL > 2):
     [sub_coll, sub_debt] = partial_liq(COLL, 10)
    #  [sub_coll, sub_debt] = full_liq(COLL)
     print("DEBT", DEBT)
     print("COLL", COLL)
     DEBT -= sub_debt
     COLL -= sub_coll
  
  print("DEBT", DEBT)
     



CRS = [
   100_00,
   90_00,
   80_00,
   70_00
]

def main():
   print("Simulate Total Liquidation with Bad debt")
   print("Coll is always 100")
   print("No price means 1 coll = 1 debt for price")
   for cr in CRS:
    print("")
    print("")
    print("")
    loop_full(cr)
  

   



main()

```
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/43#issuecomment-1752438375) on: **10/9/2023**

This adapted script shows that there is no tangible difference for Partial Liquidations and Full Liquidations:
```python

MAX_BPS = 10_000

def cr(debt, coll):
    return coll / debt * 100

def getDebtByCr(coll, cr_bps):
   return coll / cr_bps * MAX_BPS

LICR = 103_00


def max(a, b):
   if(a > b):
      return a
   return b

def min(a, b):
   if(a > b):
      return b
   return a

def full_liq(COLL):
  ## burn all
  return [COLL, COLL / LICR * MAX_BPS]

def partial_liq(COLL, percent):
   return [COLL * percent / 100, COLL * percent / 100 / LICR * MAX_BPS]

def loop_full(cr_bps):
  COLL = 100
  DEBT = getDebtByCr(COLL, cr_bps)

  ## Liquidate Partially based on premium
  while(COLL > 2):
      print("DEBT", DEBT)
      print("COLL", COLL)
      # [sub_coll, sub_debt] = partial_liq(COLL, 10)
      [sub_coll, sub_debt] = full_liq(COLL)
      print("DEBT", DEBT)
      print("COLL", COLL)
      DEBT -= sub_debt
      COLL -= sub_coll

      if(COLL > 0):
         [sub_coll, sub_debt] = full_liq(COLL)
      
         DEBT -= sub_debt
         COLL -= sub_coll
  print("DEBT", DEBT)
     



CRS = [
   100_00,
   90_00,
   80_00,
   70_00
]

def main():
   print("Simulate Total Liquidation with Bad debt")
   print("Coll is always 100")
   print("No price means 1 coll = 1 debt for price")
   for cr in CRS:
    print("")
    print("")
    print("")
    loop_full(cr)
  

   



main()

```

The finding is valid in saying that `not triggering a redistribution` is profitable to the liquidator, basically a strategy to apply #44 

Still looking into that one

