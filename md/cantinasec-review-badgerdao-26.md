# [[BCCR] Many TCR increasing operations will be blocked when TCR is between 125 and 135](https://github.com/cantinasec/review-badgerdao/issues/26)

> state: **open** opened by: **dmitriia** on: **8/11/2023**

**Context:** [BorrowerOperations.sol#L404](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#404), [BorrowerOperations.sol#L650](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#L650)

**Description:** It looks like after BCCR introduction some healthy TCR increasing openings and adjustments are now blocked.

If `oldTCR < newTCR`, but `ICR <= BCCR`, the operation can be denied if `TCR < BCCR`: 

- [BorrowerOperations.sol#L398-L413](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#L398-L413)

```solidity
if (isRecoveryMode) {
    _requireICRisAboveCCR(vars.ICR);
} else {
    _requireICRisAboveMCR(vars.ICR);
    uint newTCR = _getNewTCRFromCdpChange(vars.netColl, true, vars.debt, true, vars.price); // bools: coll increase, debt increase
    // See line below
    if (vars.ICR <= BUFFERED_CCR) {
        // Any open CDP is a debt increase so this check is safe

        // If you're dragging TCR toward buffer or RM, we add an extra check for TCR
        // Which forces you to raise TCR to 135+
        _requireNewTCRisAboveBufferedCCR(newTCR);
    } else {
        _requireNewTCRisAboveCCR(newTCR);
    }
}
```

Similarly, when it's ICR increasing collateral withdrawal coupled with debt repayment, or debt increasing coupled with collateral posting, i.e. when one adjusts, making position less risky in the process, it will be blocked when `newTCR < BCCR`: 

- [BorrowerOperations.sol#L634-L660](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#L634-L660)

```solidity
if (_isRecoveryMode) {
    // ...
} else {
    // if Normal Mode
    _requireICRisAboveMCR(_vars.newICR);
    _vars.newTCR = _getNewTCRFromCdpChange(
        // ...
    );
       // See line below
    if ((_isDebtIncrease || _collWithdrawal > 0) && _vars.newICR <= BUFFERED_CCR) {
        // Adding debt or reducing coll has negative impact on TCR, do a stricter check

        // If you're dragging TCR toward buffer or RM, we add an extra check for TCR
        // Which forces you to raise TCR to 135+
        _requireNewTCRisAboveBufferedCCR(_vars.newTCR);
    } else {
        // Other cases have a laxer check
        _requireNewTCRisAboveCCR(_vars.newTCR);
    }
}
```

Note, error message in these cases will incorrectly say about *TCR decreasing*:

- [BorrowerOperations.sol#L670-L675](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#L670-L675)

```solidity
function _requireNewTCRisAboveBufferedCCR(uint _newTCR) internal pure {
    require(
        _newTCR >= BUFFERED_CCR,
        // See line below
        "BorrowerOps: A TCR decreasing operation that would result in TCR < BUFFERED_CCR is not permitted"
    );
}
```

Previous to BCCR introduction this was true as both checks happen in non-RM state and if now it's RM the TCR has to be decreased indeed. But once logic becomes 2-tiered it not the case.

It looks like both conditions should involve `&& newTCR < oldTCR`, as if it's not then nothing bad is happening, the system health increases, just not by that much that it's desired, but it might be impossible to achieve for small CDPs in big TVL conditions, which isn't a good reason to block those as system health would have strictly improved.

Impact: many CPD opening and, mostly importantly, CDP adjustment operations that makes the system healthier by increasing TCR will be denied. This is an issue both from UX perspective and protocol stability as the cumulative impact here is that such operations, mostly coming from small CDPs (which will constitute the bigger percentage of all accounts over time along with TVL growth) will not be carried out, so there will be a downward pressure on TCR compared to the situation before the change. I.e. some operations will be carried over with bigger funds brought in, but this will be less and less possible for an average CDP owner over time, and the bigger share of operations will be just cancelled. This will result in lower TCR.

Per high likelihood and medium impact setting the severity to be high.

**Recommendation:** Consider requiring the BCCR only when where was a decrease of TCR as a result of the operation, which was the initial rationale for buffer introduction, for example:

- [BorrowerOperations.sol#L375-L413](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#L375-L413)

```diff
- bool isRecoveryMode = _checkRecoveryModeForTCR(_getTCR(vars.price));
+ uint oldTCR = _getTCR(vars.price);
+ bool isRecoveryMode = _checkRecoveryModeForTCR(oldTCR);

  vars.debt = _EBTCAmount;

  // Sanity check
  require(vars.netColl > 0, "BorrowerOperations: zero collateral for openCdp()!");

  uint _netCollAsShares = collateral.getSharesByPooledEth(vars.netColl);
  uint _liquidatorRewardShares = collateral.getSharesByPooledEth(LIQUIDATOR_REWARD);

  // ICR is based on the net coll, i.e. the requested coll amount - fixed liquidator incentive gas comp.
  vars.ICR = LiquityMath._computeCR(vars.netColl, vars.debt, vars.price);

  // NICR uses shares to normalize NICR across CDPs opened at different pooled ETH / shares ratios
  vars.NICR = LiquityMath._computeNominalCR(_netCollAsShares, vars.debt);

  /**
      In recovery move, ICR must be greater than CCR
      CCR > MCR (125% vs 110%)

      In normal mode, ICR must be greater thatn MCR
      Additionally, the new system TCR after the CDPs addition must be >CCR
  */
  if (isRecoveryMode) {
      _requireICRisAboveCCR(vars.ICR);
  } else {
      _requireICRisAboveMCR(vars.ICR);
      uint newTCR = _getNewTCRFromCdpChange(vars.netColl, true, vars.debt, true, vars.price); // bools: coll increase, debt increase

-     if (vars.ICR <= BUFFERED_CCR) {
+     // When new TCR is worse than before, it has to be above the buffer
+     if (vars.ICR <= BUFFERED_CCR && newTCR < oldTCR) {
          // Any open CDP is a debt increase so this check is safe

          // If you're dragging TCR toward buffer or RM, we add an extra check for TCR
          // Which forces you to raise TCR to 135+
          _requireNewTCRisAboveBufferedCCR(newTCR);
      } else {
          _requireNewTCRisAboveCCR(newTCR);
      }
  }
```

Similarly for `_requireValidAdjustmentInCurrentMode()`, assuming that `_vars.oldTCR` was populated before just as above via `_vars.oldTCR = _getTCR(vars.price)`:

- [BorrowerOperations.sol#L634-L660](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#L634-L660)

```diff
  if (_isRecoveryMode) {
      // ...
  } else {
      // if Normal Mode
      _requireICRisAboveMCR(_vars.newICR);
      _vars.newTCR = _getNewTCRFromCdpChange(
          // ...
      );
-     if ((_isDebtIncrease || _collWithdrawal > 0) && _vars.newICR <= BUFFERED_CCR) {
+     if ((_isDebtIncrease || _collWithdrawal > 0) && _vars.newTCR < _vars.oldTCR && _vars.newICR <= BUFFERED_CCR) {
          // Adding debt or reducing coll has negative impact on TCR, do a stricter check

          // If you're dragging TCR toward buffer or RM, we add an extra check for TCR
          // Which forces you to raise TCR to 135+
          _requireNewTCRisAboveBufferedCCR(_vars.newTCR);
      } else {
          // Other cases have a laxer check
          _requireNewTCRisAboveCCR(_vars.newTCR);
      }
  }
```

**BadgerDao:** Personally not as concerned with this fix being added until the Economic Risks are fully modelled, I think this can make some setups cheaper given issue ["[BCCR] CDP closing can be used for whale sniping"](https://github.com/cantinasec/review-badgerdao/issues/23).

The change in logic was reverted (see [BorrowerOperations.sol#L790-L846](https://github.com/Badger-Finance/ebtc/blob/1b0a073075484670f24947e0f455031144d3a1e0/packages/contracts/contracts/BorrowerOperations.sol#L790-L846)).

**Cantina:** Looks ok.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/26#issuecomment-1680151216) on: **8/16/2023**

Personally not as concerned with this fix being added until the Economic Risks are fully modelled, I think this can make some setups cheaper given #23
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/26#issuecomment-1729340636) on: **9/21/2023**

The change in logic was reverted
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/26#issuecomment-1729348853) on: **9/21/2023**

https://github.com/Badger-Finance/ebtc/blob/1b0a073075484670f24947e0f455031144d3a1e0/packages/contracts/contracts/BorrowerOperations.sol#L790-L846
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/26#issuecomment-1730122665) on: **9/21/2023**

Looks ok
