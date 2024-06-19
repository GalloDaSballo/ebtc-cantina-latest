# [Liquidator can receive outsized incentives by aiming the best liquidable CDPs first](https://github.com/cantinasec/review-badgerdao/issues/42)

> state: **open** opened by: **dmitriia** on: **10/5/2023**

**Context:** [LiquidationLibrary.sol#L505-L507](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/LiquidationLibrary.sol#L505-L507)

**Description:** While the issue not being limited to batch processing and is related to the overall protocol design, we first illustrate the batch liquidations case by showing that there stale ICRs can be used which enables the extraction of additional collateral incentives. LiquidationLibrary's `batchLiquidateCdps()` runs `_getTotalFromBatchLiquidate_RecoveryMode()` and `_getTotalsFromBatchLiquidate_NormalMode()` that process CDPs in an order supplied by liquidator. Those CDPs are synchronized with the current accrued fee split and bad debt values, while bad debt originated from the batch liquidations themself is not accounted for, i.e. after a liquidation resulted in creation of some bad debt, having `singleLiquidation.debtToRedistribute > 0`, all the ICR calculated for the subsequent CDPs using old accrual values become stale, i.e. somewhat oversized.

While TCR is tracked by updating `totalDebtToOffset`, `totalCollToSendToLiquidator`, `totalCollSurplus`, assuming that bad debt to be redistributed and so shouldn't be accounted for, the ICR of each CDP doesn't account for the fact that there is bad debt queued to be added to this CDP's debt. This is not equivalent to sequential liquidation of the same ids and allows for extracting of the additional collateral compensation by a liquidator as bigger ICR allows for bigger incentives portion than they can be provided with according to the protocol rules and given the actual state of the pool.

I.e. bad debt accounting happens after all the liquidations, so late CDPs are appear healthier than they are and provide liquidator more reward than they should, making the total liquidator reward bigger than it's due per protocol logic as part of the debt is hidden when reward is being calculated. In other words liquidation of many CDPs with bad debt artificially ignores the bad debt from within the batch, allowing for earning extra incentives compared to the sequential liquidation of the same ids that updates the state fully between each step.

Bad debt created is accumulated during subsequent liquidations in the totals structure:

```solidity
function _getLiquidationValuesRecoveryMode(
    // ...
) internal {
    LiquidationRecoveryModeLocals memory _recState = LiquidationRecoveryModeLocals(
        // ...
    );

    LiquidationRecoveryModeLocals
        memory _outputState = _liquidateIndividualCdpSetupCDPInRecoveryMode(_recState);

    singleLiquidation.entireCdpDebt = _outputState.totalDebtToBurn;
    singleLiquidation.debtToOffset = _outputState.totalDebtToBurn;
    singleLiquidation.totalCollToSendToLiquidator = _outputState.totalColToSend;
    singleLiquidation.collSurplus = _outputState.totalColSurplus;
    // See the line below
    singleLiquidation.debtToRedistribute = _outputState.totalDebtToRedistribute;
    singleLiquidation.collReward = _outputState.totalColReward;
}
```

```solidity
function _addLiquidationValuesToTotals(
    LiquidationTotals memory oldTotals,
    LiquidationValues memory singleLiquidation
) internal pure returns (LiquidationTotals memory newTotals) {
    // Tally all the values with their respective running totals
    newTotals.totalDebtInSequence =
        oldTotals.totalDebtInSequence +
        singleLiquidation.entireCdpDebt;
    newTotals.totalDebtToOffset = oldTotals.totalDebtToOffset + singleLiquidation.debtToOffset;
    newTotals.totalCollToSendToLiquidator =
        oldTotals.totalCollToSendToLiquidator +
        singleLiquidation.totalCollToSendToLiquidator;
    // See the line below
    newTotals.totalDebtToRedistribute =
        oldTotals.totalDebtToRedistribute +
        singleLiquidation.debtToRedistribute;
    // ...
```

```solidity
// Add liquidation values to their respective running totals
totals = _addLiquidationValuesToTotals(totals, singleLiquidation);
```

But it is applied to the pool state only after all the liquidations come through, in `_finalizeLiquidation()`:

```solidity
function _finalizeLiquidation(
    // ...
) internal {
    // update the staking and collateral snapshots
    _updateSystemSnapshotsExcludeCollRemainder(totalColToSend);

    emit Liquidation(totalDebtToBurn, totalColToSend, totalColReward);

    _syncGracePeriodForGivenValues(
        systemInitialCollShares - totalColToSend - totalColSurplus,
        systemInitialDebt - totalDebtToBurn,
        price
    );

    // redistribute debt if any
    if (totalDebtToRedistribute > 0) {
        // See the line below
        _redistributeDebt(totalDebtToRedistribute);
    }
    // ...
```

So liquidator incentive collateral can be overstated, being dependent linearly on stale ICR in the `_ICR > LICR` case, which appears bigger than it is as there is already generated, but still undistributed bad debt:

```solidity
function _calculateSurplusAndCap(
    uint256 _ICR,
    uint256 _price,
    uint256 _totalDebtToBurn,
    uint256 _totalColToSend,
    bool _fullLiquidation
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
        // See the line below
        _incentiveColl = (_totalDebtToBurn * (_ICR > MCR ? MCR : _ICR)) / _price;  // @audit _ICR is actually lower and that much incentive can't be given away
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
            _incentiveColl = (_totalDebtToBurn * LICR) / _price;
        }
    }
    cappedColPortion = collateral.getSharesByPooledEth(_incentiveColl);
    cappedColPortion = cappedColPortion < _totalColToSend ? cappedColPortion : _totalColToSend;
    collSurplus = (cappedColPortion == _totalColToSend) ? 0 : _totalColToSend - cappedColPortion;
}
```

Impact: liquidators can obtain extra collateral incentives with regard to the actual state of the pool and the stated `max(103%, min(ICR, 110%))` formula. In order for this to happen there should be CDPs with `ICR < LICR`, so bad debt can be generated, and also CDPs with `LICR < ICR < MCR`, so there is a collateral incentive obtainable linearly to the ICR of the CDP liquidated. Grace period introduction somewhat increases the odds of reaching this situation as during it some CDPs with `MCR < ICR < TCR` can become ones with `ICR < MCR`, as market quickly drops, for example, and it will generally be more CDPs available for liquidation during shorter period.

Per high total likelihood and low to medium impact setting the severity to be medium.

**Recommendation:** First, consider redistributing debt after each liquidation, e.g.:

```diff
  function _getTotalFromBatchLiquidate_RecoveryMode(
        // ...
  ) internal returns (LiquidationTotals memory totals) {
        // ...
      for (vars.i = _start; ; ) {
          vars.cdpId = _cdpArray[vars.i];
          // only for active cdps
          if (vars.cdpId != bytes32(0) && Cdps[vars.cdpId].status == Status.active) {
              vars.ICR = getICR(vars.cdpId, _price);

              if (
                  !vars.backToNormalMode &&
                  (vars.ICR < MCR || canLiquidateRecoveryMode(vars.ICR, _TCR))
              ) {
                  vars.price = _price;
                  _syncAccounting(vars.cdpId);
                  _getLiquidationValuesRecoveryMode(
                      _price,
                      vars.entireSystemDebt,
                      vars.entireSystemColl,
                      vars,
                      singleLiquidation,
                      sequenceLiq
                  );
+                 if (singleLiquidation.debtToRedistribute > 0) {
+                     _redistributeDebt(singleLiquidation.debtToRedistribute);
+                 }
                  // Update aggregate trackers
                  // ...
                  vars.backToNormalMode = _TCR < CCR ? false : true;
                  _liqFlags[vars.i] = true;
                  _liqCnt += 1;
              } else if (vars.backToNormalMode && vars.ICR < MCR) {
                  _syncAccounting(vars.cdpId);
                  _getLiquidationValuesNormalMode(
                      _price,
                      _TCR,
                      vars,
                      singleLiquidation,
                      sequenceLiq
                  );
+                 if (singleLiquidation.debtToRedistribute > 0) {
+                     _redistributeDebt(singleLiquidation.debtToRedistribute);
+                 }
                  // Add liquidation values to their respective running totals
                  totals = _addLiquidationValuesToTotals(totals, singleLiquidation);
                  _liqFlags[vars.i] = true;
                  _liqCnt += 1;
              }
              // ...
```

```diff
  function _getTotalsFromBatchLiquidate_NormalMode(
      // ...
  ) internal returns (LiquidationTotals memory totals) {
      // ...
      for (vars.i = _start; ; ) {
          vars.cdpId = _cdpArray[vars.i];
          // only for active cdps
          if (vars.cdpId != bytes32(0) && Cdps[vars.cdpId].status == Status.active) {
              vars.ICR = getICR(vars.cdpId, _price);

              if (vars.ICR < MCR) {
                  _syncAccounting(vars.cdpId);
                  _getLiquidationValuesNormalMode(
                      _price,
                      _TCR,
                      vars,
                      singleLiquidation,
                      sequenceLiq
                  );
+                 if (singleLiquidation.debtToRedistribute > 0) {
+                     _redistributeDebt(singleLiquidation.debtToRedistribute);
+                 }
                  // Add liquidation values to their respective running totals
                  totals = _addLiquidationValuesToTotals(totals, singleLiquidation);
                  _liqCnt += 1;
              }
          }
          // ...
```

```diff
  function _finalizeLiquidation(
      // ...
  ) internal {
      // update the staking and collateral snapshots
      _updateSystemSnapshotsExcludeCollRemainder(totalColToSend);

      emit Liquidation(totalDebtToBurn, totalColToSend, totalColReward);

      _syncGracePeriodForGivenValues(
          systemInitialCollShares - totalColToSend - totalColSurplus,
          systemInitialDebt - totalDebtToBurn,
          price
      );

-     // redistribute debt if any
-     if (totalDebtToRedistribute > 0) {
-         _redistributeDebt(totalDebtToRedistribute);
-     }
      // ...
```

It doesn't seem to be enough as a liquidator can simply arrange ids in `_cdpArray` argument of `batchLiquidateCdps()` so that all the good ones come first and all the bad ones come last, so ICRs of good ones not being reduced by bad debt redistribution coming from the bad ones.

This can be controlled by introducing a rolling ICR metric and demanding that liquidator supply `_cdpArray` with increasing ICRs, e.g. (for _getTotalFromBatchLiquidate_RecoveryMode it's the same):

```diff
  function _getTotalsFromBatchLiquidate_NormalMode(
        // ...
  ) internal returns (LiquidationTotals memory totals) {
        // ...
+     uint256 lastICR;
      for (vars.i = _start; ; ) {
          vars.cdpId = _cdpArray[vars.i];
          // only for active cdps
          if (vars.cdpId != bytes32(0) && Cdps[vars.cdpId].status == Status.active) {
              vars.ICR = getICR(vars.cdpId, _price);
+             require(vars.ICR >= lastICR, "LiquidationLibrary: ascending ICR order is required for CPDs");
+             lastICR = vars.ICR;
              if (vars.ICR < MCR) {
                   // ...
              }
          }
          // ...
```

This will not close the surface fully as a liquidator can still run a series of individual liquidations atomically in the same good first, bad last manner.

As a more complete approach in order to align design with incentives consider requiring that CDP being liquidated has to be the worst one, i.e. that `sortedCdps.getNext(liquidatedID) == sortedCdps.nonExistId()`. This will properly redistribute all the bad debt due and ensure that ICRs used for liquidator incentive calculations are not overstated.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1748985426) on: **10/5/2023**

Let's model this further before writing a fix, also worth discussing with Liquity
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1749081207) on: **10/5/2023**

Overall this is equivalent to saying that Current Liquidations are not Subject to Pending Bad Debt redistribution

I believe the most egregious case is how Bad Debt Redistribution could be avoided via front-running and closing / reducing stake, this is a further example of such a case which favours the Liquidator and unfavours Stakers

In terms of the size and impact of the redistribution that is dependent on a few factors which we should further model:
- Stake of Liquidatable CDPs (Stake % determine % of Bad Debt to receive)
- Ratio of Bad Debt

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1750010805) on: **10/6/2023**

This looks like a liquidation ordering issue: "Good" (debt redistribution free) first, "Bad" (generate debt redistribution) later. 

Every coin got two sides.

If we apply the suggestion of "CDP being liquidated has to be the worst one", it will remove the freedom for liquidators to choose CDPs, i.e., no flexibility and diversity in liquidation competition anymore. For example, if the "worst" one is a gigantic CDP, even with partial liquidation, it will still block the liquidations of other smaller risky CDPs. In a way, it might delay the recovery of the entire system to normal mode.

I intend to agree with @dmitriia opinion that adding a ICR ordering check for batch liquidation and apply debt redistribution inside the loop for each "bad" liquidated CDP could be a valid mitigation.

"run a series of individual liquidations" could cost liquidator more and increase uncertainty obviously 



---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1750185608) on: **10/6/2023**

Let's model this first via the following:

-> System at X% CR - (worst case IMO is 120% CR already insanely levered)
-> Bad Debt % (e.g. 109% vs 100% vs 95% vs 80% CR on Underwater CDPs)
-> Use Partial Liquidations with 3% premium, without redistribution
-> Compute total bad debt
-> Check Stake (since the bad debt is redistributed via stake)
-> See impact on those liquidations

Also note that since we have 3% fixed premium, the marginal additional bad debt is already guaranteed to the rest of stakers, so this must be modelled further

You can fork some of my scripts here to get started:
https://github.com/GalloDaSballo/Cdp-Demo/tree/main/scripts

Will send something next week
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1752112927) on: **10/8/2023**

Note on targeting CDPs, unless we enforce the sorting via the LL, then the recommendation above can be sidestepped by passing in one CDP at a time or a segment of the list that has lower risk
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1752475356) on: **10/9/2023**

Consideration 1:
The above doesn't apply to undercollateralized CDPs
For Undercollateralized CDPs, the Premium is fixed to 3%

If we simplify the system to Coll and Debt having the same price

Then a underwater CDP has 
Coll = 100
Debt > 100

For these CDPs, a 3% premium is
Coll / 103 = Debt Repaid
Coll withdrawn = 100%

If we were to add more Bad Debt to such Cdps, the Bad Debt wouldn't attack profitability of the Liquidator, that Bad Debt is already going to be redistributed

In other words: The Marginal increase in Bad Debt for Underwater CDPs is already going to be socialized

This excludes the worst case scenario

Thinking through non-underwater CDPs and premium for them
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1752479063) on: **10/9/2023**

From discussion in 43 we know that partial liquidations are basically equivalent, this means I used full liquidations to model the math

I think for maximum precision you'd need to do an integral for the proper math (liquidate CDP 1 -> Redistribute to Rest), but the below is a linear approximation in the excess


LOGS: CRS (100, 80, 50) Deeply underwater
System CR at 120

```python
BAD CR 100


total_coll 300
total_debt 250.0
underwater_coll 200
underwater_debt 200.0

sub_coll 200
sub_debt 194.1747572815534

bad_debt_to_redistribute 5.825242718446589
As % of debt to liquidate 2.9126213592232943
Skipped Debt 133.33333333333331
Skipped Debt as % of debt to redistribute 22.88888888888894
underwater_debt 200.0
underwater_coll 200
total_coll 300
As % coll 66.66666666666666
After all
new_total_coll 100
new_total_debt 55.82524271844659
cr of sys after 179.13043478260872


BAD CR 80


total_coll 180
total_debt 150.0
underwater_coll 80
underwater_debt 100.0

sub_coll 80
sub_debt 77.66990291262135

bad_debt_to_redistribute 22.330097087378647
As % of debt to liquidate 22.330097087378647
Skipped Debt 44.44444444444444
Skipped Debt as % of debt to redistribute 1.990338164251207
underwater_debt 100.0
underwater_coll 80
total_coll 180
As % coll 44.44444444444444
After all
new_total_coll 100
new_total_debt 72.33009708737865
cr of sys after 138.25503355704697


BAD CR 50


total_coll 129
total_debt 108.0
underwater_coll 29
underwater_debt 57.99999999999999

sub_coll 29
sub_debt 28.155339805825243

bad_debt_to_redistribute 29.84466019417475
As % of debt to liquidate 51.45631067961165
Skipped Debt 13.038759689922479
Skipped Debt as % of debt to redistribute 0.43688752376773443
underwater_debt 57.99999999999999
underwater_coll 29
total_coll 129
As % coll 22.48062015503876
After all
new_total_coll 100
new_total_debt 79.84466019417476
cr of sys after 125.2431906614786
```


Code

```python
## Bad Debt per Liquidation

## Each subsequent would have a stake %

## The Stake % is used to calculate the Debt Skipped by the Liquidator



def get_debt(coll, cr):
    return coll / cr * 100

def get_cr(coll, debt):
    return coll / debt * 100


## Figure out the cost of the swap
COST_SWAP = 30 ## Either Pool, or Mint Fee


SAFE_COLL = 100
SAFE_CR = 200
SAFE_DEBT = get_debt(SAFE_COLL, SAFE_CR)

SIM_CR = 120 ## 120 CR of the whole system


def generate_debt_and_coll_scenario(underwater_cr, target_system_cr, safe_coll, safe_debt):
    underwater_coll = 0
    underwater_debt = 0

    total_coll = safe_coll + underwater_coll
    total_debt = safe_debt + underwater_debt

    ## All of these are undercollateralized
    while get_cr(total_coll, total_debt) > target_system_cr:
        underwater_coll += 1
        underwater_debt = get_debt(underwater_coll, underwater_cr)

        total_coll = safe_coll + underwater_coll
        total_debt = safe_debt + underwater_debt

    return [
        underwater_coll,
        underwater_debt,
        total_coll,
        total_debt
    ]

    ## While the reality of chunking is slightly different
    ## A theoretical chunking is 100% of Liquidatable Debt / Stake of the total debt (give by CR)
    ## This is the absolute worst case, which is good to model worst impact

LICR = 103

def full_liq(COLL):
  ## burn all coll since we're capped on debt and not on coll
  return [COLL, COLL / LICR * 100]

def loop(underwater_cr):

  assert underwater_cr < 103 ## It's actually underwater

  [
        underwater_coll,
        underwater_debt,
        total_coll,
        total_debt
  ] = generate_debt_and_coll_scenario(underwater_cr, SIM_CR, SAFE_COLL, SAFE_DEBT)

  print("")
  print("")
  print("total_coll", total_coll)
  print("total_debt", total_debt)
  print("underwater_coll", underwater_coll)
  print("underwater_debt", underwater_debt)

  [sub_coll, sub_debt] = full_liq(underwater_coll)

  print("")
  print("sub_coll", sub_coll)
  print("sub_debt", sub_debt)
  print("")

  ## Now we check the bad debt
  bad_debt_to_redistribute = underwater_debt - sub_debt
  print("bad_debt_to_redistribute", bad_debt_to_redistribute)
  ## To Redistribute is the % of debt over all underwater
  print("As % of debt to liquidate", bad_debt_to_redistribute / underwater_debt * 100)

  ## Skipped is the debt 
  skipped_stake_ratio = (underwater_coll / total_coll)
  skipped_debt = underwater_debt * skipped_stake_ratio

  ## How much did we skip?
  print("Skipped Debt", skipped_debt)
  print("Skipped Debt as % of debt to redistribute", skipped_debt / bad_debt_to_redistribute)

  print("underwater_debt", underwater_debt)
  print("underwater_coll", underwater_coll)
  print("total_coll", total_coll)
  print("As % coll", underwater_coll / total_coll * 100)

  print("After all")

  new_total_coll = total_coll - sub_coll
  new_total_debt = total_debt - sub_debt

  print("new_total_coll", new_total_coll)
  print("new_total_debt", new_total_debt)

  print("cr of sys after", get_cr(new_total_coll, new_total_debt))



CRS = [
   100,
   80,
   50
]
def main():
  for bad_cr in CRS:
   print("")
   print("")
   print("BAD CR", bad_cr)
   loop(bad_cr)
  
main()
```
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1752504542) on: **10/9/2023**

After thinking about it

This is just another way of saying: Liquidators will liquidate Underwater CDPs last

Since those offer the lowest premium

This is factually true and has been modelled in conjunction with Risk DAO

The entirety of the discussion falls under the fact that Liquidators will liquidate riskiest CDPs last
And that’s a consequence of trying to minimise bad debt
We have modelled this in conjunction with risk Dao, to determine the proper balance
And 3% is the sweet spot from their simulations


——

Quick note on profitability:
From a tool I built: https://github.com/GalloDaSballo/pool-math

We can derive that both Curve and Balancer, with default settings, are able to swap at least the amount of a reserve before losing the 3%

This is further modelled through simulations in this report:
https://github.com/Risk-DAO/Reports/blob/main/eBTC.pdf


——

One way to further discuss the impact would be this idea of a simulation:

Bad Debt CDP with
1 Coll
Insane DEBT CDP -> highest debt possible

Redistribution Victim (skip)
120 CDP That drags the TCR down to 124.999 (max stake) -> highest Coll possible


Rest at 200 (safe users)


——

The above could help show the maximum impact

However, unless I missed something this should be cause for downgrading the odds as well as the severity of the issues as the redistribution being this way is a guarantee from any rational actor


---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1761532760) on: **10/13/2023**

I would classify this as the same as the risk of stakers of having to take on bad debt

Which is something we accept from using the liquity model

Would ask to reconsider severity given the explanation above, the impact and the risk profile the system already accepts
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/42#issuecomment-1761610521) on: **10/13/2023**

I tend to think that probability of the behavior described is high as it’s profitable and has zero cost for a liquidator.

It might be argued that the impact is low instead of medium, but it really depends on the state of the system. It can be correct to say that it’s low most of the times and can be medium when pool has substantial amount of bad debt.

So I updated the impact to `low to medium` and severity to `medium` this way. I do not think that the implementation should be left as it is though.

Simulation of the typical/most probable/going concern scenarios is generally not a proper measure of the associated risk. Corner cases can happen and should be accounted for.

Liquidation incentives in eBTC are higher than in Liquity, so it can’t be said that this phenomenon is the same and is just inherited.
