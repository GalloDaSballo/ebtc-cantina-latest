# [Bad debt accumulation index is reset on partial liquidation and is not reset on CDP increase, allowing for phantom bad debt creation](https://github.com/cantinasec/review-badgerdao/issues/44)

> state: **open** opened by: **dmitriia** on: **10/6/2023**

**Context:** [LiquidationLibrary.sol#L446](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/LiquidationLibrary.sol#L446), [CdpManager.sol#L913-L916](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/CdpManager.sol#L913-L916)

**Description:** Partial liquidation incorrectly resets `debtRedistributionIndex` of the CDP, while CDP update functionality should reset it when CDP's collateral was increased, but doesn't do it. This can create debt that doesn't correspond to any eBTC minted.

Pending bad debt accumulator is subject to rounding error, i.e. `pendingEBTCDebtReward` can routinely be zero for small CDPs:

```solidity
function _getPendingRedistributedDebt(
    bytes32 _cdpId
) internal view returns (uint256 pendingEBTCDebtReward) {
    Cdp storage cdp = Cdps[_cdpId];

    if (cdp.status != Status.active) {
        return 0;
    }

    uint256 rewardPerUnitStaked = systemDebtRedistributionIndex -
        debtRedistributionIndex[_cdpId];

    if (rewardPerUnitStaked > 0) {
        // See the line below
        pendingEBTCDebtReward = (cdp.stake * rewardPerUnitStaked) / DECIMAL_PRECISION;
    }
}
```

`_updateRedistributedDebtSnapshot()`, that updates the bad debt index, is now called by `_syncAccounting()` only when pending bad debt is positive, i.e. when accumulation exceeds this rounding error:

```solidity
function _syncAccounting(bytes32 _cdpId) internal {
    // See the line below
    (, uint _newDebt, , uint _pendingDebt) = _applyAccumulatedFeeSplit(_cdpId);

    // Update pending debts
    if (_pendingDebt > 0) { // See this line
        Cdp storage _cdp = Cdps[_cdpId];
        uint256 prevDebt = _cdp.debt;
        uint256 prevColl = _cdp.coll;

        // Apply pending rewards to cdp's state
        _cdp.debt = _newDebt;
        // See the line below
        _updateRedistributedDebtSnapshot(_cdpId);
        // ...
```

I.e. when there is a rounding to zero the implemented approach allows accumulation to happen until pending part becomes positive to correct for that rounding error over time. This can take long enough when CDP is small.

But the same update is called in `_partiallyReduceCdpDebt()` in isolation and unconditionally:

```solidity
function _partiallyReduceCdpDebt(
    bytes32 _cdpId,
    uint256 _partialDebt,
    uint256 _partialColl
) internal {
    Cdp storage _cdp = Cdps[_cdpId];

    uint256 _coll = _cdp.coll;
    uint256 _debt = _cdp.debt;

    _cdp.coll = _coll - _partialColl;
    _cdp.debt = _debt - _partialDebt;
    _updateStakeAndTotalStakes(_cdpId);
    // See the line below
    _updateRedistributedDebtSnapshot(_cdpId);
}
```

Notice that there was a `_syncAccounting()` state update at the beginning of the `partiallyLiquidate` call:

```solidity
// Single CDP liquidation function (partially).
function partiallyLiquidate(
    // ...
) external nonReentrantSelfAndBOps {
    _liquidateIndividualCdpSetup(_cdpId, _partialAmount, _upperPartialHint, _lowerPartialHint);
}

// Single CDP liquidation function.
function _liquidateIndividualCdpSetup(
    // ...
) internal {
    _requireCdpIsActive(_cdpId);
    // See the line below
    _syncAccounting(_cdpId);
```

Overall there can be 3 cases, it was either:
1. `_pendingDebt > 0` and `_updateRedistributedDebtSnapshot(_cdpId)` in `_partiallyReduceCdpDebt()` does nothing as it was called already.
2. Or `_pendingDebt == 0` and `debtRedistributionIndex[_cdpId] == systemDebtRedistributionIndex` and `_updateRedistributedDebtSnapshot(_cdpId)` also does nothing.
3. Or `_pendingDebt == 0` and `debtRedistributionIndex[_cdpId] < systemDebtRedistributionIndex`, i.e. there were a rounding of the pending part to zero. Since CDP stake can only fall as a result of partial liquidation, the rounding thereafter can only become worse.

I.e. in order to reduce the total rounding error the correct take here is to keep old `debtRedistributionIndex[_cdpId]`, so it be updated later on when pending be accrued (slower than before as stake was reduced) to some positive value.

Notice also that in the similar situation of stake reduction due to partial redemption there is no `debtRedistributionIndex` reset, which looks correct this way:

```solidity
function _redeemCollateralFromCdp(
    SingleRedemptionInputs memory _redeemColFromCdp
) internal returns (SingleRedemptionValues memory singleRedemption) {
    // ...

    if (newDebt == 0) {
        // No debt remains, close CDP
        // ...
    } else {
        // Debt remains, reinsert CDP
        uint256 newNICR = LiquityMath._computeNominalCR(newColl, newDebt);

        // ...

        Cdps[_redeemColFromCdp.cdpId].debt = newDebt;
        Cdps[_redeemColFromCdp.cdpId].coll = newColl;
        // See the line below
        _updateStakeAndTotalStakes(_redeemColFromCdp.cdpId);
        // ...
```

On the other side, when CDP stake is increased, such debt index reset should take place as otherwise the accumulated `systemDebtRedistributionIndex - debtRedistributionIndex[_cdpId]` difference will be applied to the whole new stake next time, which can result in a massive new phantom bad debt creation when small CDP became much bigger as a result of CDP update.

```solidity
function updateCdp(
    // ...
) external {
    _requireCallerIsBorrowerOperations();

    _setCdpCollShares(_cdpId, _newColl);
    _setCdpDebt(_cdpId, _newDebt);

    uint256 stake = _updateStakeAndTotalStakes(_cdpId);

    emit CdpUpdated(
    // ...
```

I.e. suppose CDP's `debtRedistributionIndex` wasn't updated due to rounding to zero for a long time, then this CDP was increased by $10^5$ times by an update via `adjustCdpWithColl()` $\rightarrow$ `_adjustCdpInternal()`, while `debtRedistributionIndex` was kept intact as `cdpManager.syncAccounting(_cdpId)` was called before increase, while after increase no debt accruals updates are being made, as `_setCdpDebt` only rewrites current CDP debt figure.

Impact: bad debt accumulation is written off within rounding error for a partially liquidated CDP, while new non-existent bad debt can be added on a substantial CDP size increase. The latter deteriorates the health of the CDP and creates eBTC debt that was never minted. In order for that to be material, the growth in CDP size should be massive, which on the one hand is a part of regular workflow, on the another doesn't frequently happen, so the probability here can be estimated as medium. In the same time this effect, being systematic, will accumulate over time across different CDPs.

The effect from one CDPs is as follows: if realistic increase can be seen at $10^5x$, as there is a minimal CDP size constraint, a maximum increase can be, as an example, from having minimal collateral to `200k ETH` or about `USD 330 mln` at current rates. The impact will be about the same number in eBTC terms, as `0.99999 * 1e5` (i.e. `0.99999` was rounded to `0` before the increase) or `99999 * 27600 / 1e18 = 3 / 10^9 USD`.

While this alone isn't material, there also considerations of this new debt not being representing any eBTC minted, so along with such new debt creation grows a chance of DOS from eBTC burn rejection, i.e. in absense of manual additional minting the last CDP might not be able to close (i.e. it sent its eBTC to the market so others can purchase and close, while it can't close as there is not enough eBTC in circulation). By itself this looks manageable.

Both parts in total have high likelihood and low impact, so setting overall severity to be medium.

**Recommendation:** Consider removing `_updateRedistributedDebtSnapshot()` call in `_partiallyReduceCdpDebt()`:

```diff
  function _partiallyReduceCdpDebt(
      // ...
  ) internal {
      // ...

      _cdp.coll = _coll - _partialColl;
      _cdp.debt = _debt - _partialDebt;
      _updateStakeAndTotalStakes(_cdpId);

-     _updateRedistributedDebtSnapshot(_cdpId);
  }
```

Calling `_updateRedistributedDebtSnapshot()` in `updateCdp()` on CDP increase in the same time can be advised (the situation doesn't look to be symmetrical, nothing offsets this growth, so it need to be controlled at the price of forcing some debt rounding), consider adding:

```diff
  function updateCdp(
      // ...
  ) external {
      _requireCallerIsBorrowerOperations();

      _setCdpCollShares(_cdpId, _newColl);
      _setCdpDebt(_cdpId, _newDebt);

      uint256 stake = _updateStakeAndTotalStakes(_cdpId);

+     if (_newColl > _coll) {
+         _updateRedistributedDebtSnapshot(_cdpId);
+     }

      emit CdpUpdated(
          _cdpId,
          _borrower,
          _debt,
          _coll,
          _newDebt,
          _newColl,
          stake,
          CdpOperation.adjustCdp
      );
  }
```

**BadgerDAO:** The related code change was introduced via advised fix in an [earlier audit](https://github.com/spearbit-audits/review-badgerdao/issues/72) to apply update on `debtRedistributionIndex[_cdpId]` **only if** calculated `pendingDebt` is positive instead of original Liquity code behavior, i.e., [direct `debtRedistributionIndex[_cdpId]` comparison against `systemDebtRedistributionIndex`]( https://github.com/liquity/dev/blob/main/packages/contracts/contracts/TroveManager.sol#L1060). 

Intend to agree with the finding on:
- The "phantom debt increase" due to this update skip of `debtRedistributionIndex[_cdpId]`,
- And unconditional update of `debtRedistributionIndex[_cdpId]` in partial liquidation could make it worse when the `pendingDebt` is rounding to zero.

**Cantina:** Yes, [the issue from the past audit](https://github.com/spearbit-audits/review-badgerdao/issues/72) had these implications we didn’t reflected at that moment.

An alternative to the suggested mitigation here is to roll back that change, resetting the index all the time even if nothing was added to the accumulator, forfeiting bad debt accumulation for small CDPs. I.e. it might be the case that some part of bad debt distribution will be void for all the small ones due to the rounding if updates be frequent enough. Something like `F({frequency of updates}, {share of min collateral CDPs in the system})` $\rightarrow$ `{total share of bad debt being written off}` can be modelled to understand whether that’s material.

Partial liquidation suggestion will still stand as no additional reset be needed in this case too.

**BadgerDAO:** The suggested fix is in [PR 694](https://github.com/Badger-Finance/ebtc/pull/694). Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/44#issuecomment-1751585574) on: **10/7/2023**

The related code change was introduced via advised fix in [earlier audit](https://github.com/spearbit-audits/review-badgerdao/issues/72) to apply update on `debtRedistributionIndex[_cdpId]` **only if** calculated `pendingDebt` is positive instead of original Liquity code behavior, i.e., [direct `debtRedistributionIndex[_cdpId]` comparison against `systemDebtRedistributionIndex`]( https://github.com/liquity/dev/blob/main/packages/contracts/contracts/TroveManager.sol#L1060). 

Intend to agree with @dmitriia finding on 

- the "phantom debt increase" due to this update skip of `debtRedistributionIndex[_cdpId]` & 
- unconditional update of `debtRedistributionIndex[_cdpId]` in partial liquidation could make it worse when the `pendingDebt` is rounding to zero
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/44#issuecomment-1751718447) on: **10/7/2023**

Yes, [#72](https://github.com/spearbit-audits/review-badgerdao/issues/72) had these implications we didn’t reflected at that moment.

An alternative to the suggested mitigation here is to roll back that change, resetting the index all the time even if nothing was added to the accumulator, forfeiting bad debt accumulation for small CDPs. I.e. it might be the case that some part of bad debt distribution will be void for all the small ones due to the rounding if updates be frequent enough. Something like `F({frequency of updates}, {share of min collateral CDPs in the system}) -> {total share of bad debt being written off}` can be modelled to understand whether that’s material.

Partial liquidation suggestion will still stand as no additional reset be needed in this case too.
---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/44#issuecomment-1761016114) on: **10/13/2023**

@dmitriia Hello ser, would you please take a look here for the fix as you suggested? Really appreciate
https://github.com/Badger-Finance/ebtc/pull/694
