# [Grace period reset can be avoided by redeeming when switch to normal mode occurs due to market conditions and there are no or few CDPs between MCR and TCR](https://github.com/cantinasec/review-badgerdao/issues/40)

> state: **open** opened by: **dmitriia** on: **10/4/2023**

**Context:** [CdpManager.sol#L327-L354](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/CdpManager.sol#L327-L354)

**Description:** CdpManager's `redeemCollateral()` allows redeeming any `cdpID` as long as `MCR < TCR && getICR(sortedCdps.getNext(cdpID), _price) < MCR && getICR(cdpID, _price) >= MCR`. In a rare situation when such CDP has `ICR > TCR`, its redemption can lower TCR below CCR, and this will happen without triggering of the grace period, as the only `_syncGracePeriodForGivenValues()` call happens with the resulting state.

In other words, redemption logic allows starting with `TCR > CCR` state and ending with `TCR < CCR`, while grace period will be updated with the output `TCR < CCR` state only. When the system was in the RM state and normal mode state have been reached due to the external market conditions, and not yet being accounted in the system, `redeemCollateral()` can be used to silently drive the system back to RM without starting the corresponding grace period.

Impact: when recovery to normal mode transition takes place due to market conditions and not yet accounted in the system, while there are no or few CDPs with ICR between MCR and TCR, it can be possible to drive the system back to RM without starting new grace period. Technically it allows for liquidation of CDPs with `MCR <= ICR < TCR`, but immediately this set will be empty as all such CDPs will have to be redeemed. However, with further market movements, which can happen quickly enough, some CDPs might satisfy this condition. They will be immediately liquidable as grace period be deemed expired.

Per low likelihood and medium impact setting the severity to be low.

**Recommendation:** Since `redeemCollateral()` is unique in the regard that it allows its impact to be RM initiating, as a simplest approach consider adding initial state synchronization there, e.g.:

- [CdpManager.sol#L327-L354](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/CdpManager.sol#L327-L354)

```diff
  function redeemCollateral(
      // ...
  ) external override nonReentrantSelfAndBOps {
      RedemptionTotals memory totals;

      _requireValidMaxFeePercentage(_maxFeePercentage);

      _syncGlobalAccounting(); // Apply state, we will syncGracePeriod at end of function

      totals.price = priceFeed.fetchPrice();
      {
          (
              uint256 tcrAtStart,
              uint256 totalCollSharesAtStart,
              uint256 totalEBTCSupplyAtStart
          ) = _getTCRWithSystemDebtAndCollShares(totals.price);
          totals.tcrAtStart = tcrAtStart;
          totals.totalCollSharesAtStart = totalCollSharesAtStart;
          totals.totalEBTCSupplyAtStart = totalEBTCSupplyAtStart;
      }

+     // Notify current mode
+     if (totals.tcrAtStart < CCR) {
+         _startGracePeriod(totals.tcrAtStart);
+     } else {
+         _endGracePeriod(totals.tcrAtStart);
+     }

      _requireTCRoverMCR(totals.price, totals.tcrAtStart);
  // ...
```

**BadgerDao:** I may be missing something but synching is done at [CdpManager.sol#L463C9-L463C39](https://github.com/Badger-Finance/ebtc/blob/9f7556e654e7fa3ecac471de92a55a4115c6edb8/packages/contracts/contracts/CdpManager.sol#L463C9-L463C39).

**Cantina:** The issue is that there is only one sync, while the operation can make the system enter RM as redemption logic allows starting with `TCR > CCR` state and ending with `TCR < CCR`. So the grace period end trigger can be skipped as the `_syncGracePeriodForGivenValues()` sync in the end will see `TCR < CCR`. This way the [second] grace period can be avoided, and liquidations can happen shortly.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/40#issuecomment-1752108525) on: **10/8/2023**

I may be missing something but synching is done here: https://github.com/Badger-Finance/ebtc/blob/9f7556e654e7fa3ecac471de92a55a4115c6edb8/packages/contracts/contracts/CdpManager.sol#L463C9-L463C39
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/40#issuecomment-1752128114) on: **10/8/2023**

The issue is that there is only one sync, while the operation can make the system enter RM as
> redemption logic allows starting with `TCR > CCR` state and ending with `TCR < CCR`.

So the grace period end trigger can be skipped as the `_syncGracePeriodForGivenValues()` sync in the end will see `TCR < CCR`. This way the [second] grace period can be avoided, and liquidations can happen shortly.
