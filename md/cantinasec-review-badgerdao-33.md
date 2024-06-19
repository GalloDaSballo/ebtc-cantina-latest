# [Bulk Informational Issues relative to Release 0.4 PR](https://github.com/cantinasec/review-badgerdao/issues/33)

> state: **open** opened by: **StErMi** on: **9/24/2023**

**Context:** Global scope

**Description:** This issue includes a list of informational issues found and noted while reviewing the PR that would merge the [Release 0.4 PR](https://github.com/Badger-Finance/ebtc/pull/620) into the main branch.

The issues are not sorted by type or files.

1) Inside `CdpManagerStorage.getICR`  `currentETH` are not ETH but SHARES.
2) Inside `CdpManagerStorage.getNominalICR`  `currentETH` are not ETH but SHARES.
3) `LiquityMath` `_computeNominalCR` and `_computeCR` should specify if `_coll` is `stETH` or shares.
4) In general, there are other places where the term `collateral`/`coll` is used in a mixed way. Sometimes it's about `stETH` shares, sometimes it's about `stETH`. I think that the best thing to do is to:
	1) Not use anymore the term `coll`/`collateral` and always use `stETH` or `stETHShares`.
	2) Decide which is the collateral and use the other as the secondary term. So, for example, you define that `stETH` is the collateral and so by default when you see the term `coll` you know you are referring to `stETH` and everything else that is not `coll` or `stETH` should be explicitly defined as `stETHShares`.
5) missing full-natspec for functions like `_calcSyncedAccounting`, `_getSyncedCdpDebtAndRedistribution` and so on (I assume there are more where the natspec miss or is incomplete). Try to fully cover the functions with `@notice`, `@dev`, `@param` and `@return` when needed.
6) `syncGlobalAccounting` `@dev` comment is unfinished.
7) if a grace period starts, it won't be bound to the current value of `recoveryModeGracePeriod`. The caller could call `setGracePeriod` and change the grace period that has already started by making it smaller or bigger. See issue ["Governance can to end or extend the grace period even when a grace period is already ongoing"](https://github.com/cantinasec/review-badgerdao/issues/30).
8) Consider changing `recoveryModeGracePeriod` to `recoveryModeGracePeriodDuration`.
9) Refactor `CdpManagerStorage` to have constants, storage layouts and so on as the first thing and function after (with a proper order). Please follow the [Solidity Style guide](https://docs.soliditylang.org/en/latest/style-guide.html).
11) In `CdpManagerStorage` `stEthFeePerUnitIndex` was more meaningful before (`stFeePerUnitcdp`). Now it has lost the information that it's about CDP. Consider changing the name to be a reference to the single CDP index.
12) Same thing for `debtRedistributionIndex` in `CdpManagerStorage`(not that before was more meaningful). Consider renaming to something that is related to the CDP.
13) In `CdpManagerStorage` `lastEBTCDebtErrorRedistribution` has "lost" the `_` in the name, but `lastETHError_Redistribution` still has it. Use the same naming standard.
14) In `CdpManagerStorage` there are multiple occurrences of the variable `CdpIdsArrayLength` that is declared with the first letter uppercase. Follow the Solidity style guide and declare it with the first letter lowercase.
15) In `CdpManagerStorage` use the same order for the input parameters (both `stETH` index) for `_syncStEthIndex` and `_calcSyncedGlobalAccounting`. In those function, the order is inverted and could cause confusion. Try to always stick with the same order.
16) `CdpManagerStorage` `getSyncedICR` and `getSyncedTCR` can be declared `external` because are not used by any contract.
17) `CdpManagerStorage` `getNominalICR` and `getICR` uses the variable `currentETH` but inside that variable you don't have `ETH` or `stETH` but `stETHShares` returned by `getDebtAndCollShares`. Change the name of the variable accordingly.
18) In general, it seems that the name `EBTC` has been replaced in many places with the general term "Debt". For example, `getPendingEBTCDebtReward` has been renamed `getPendingRedistributedDebt`. There are still many instances in the whole codebase where variables and functions refer to `eBTC` instead of `debt` (same for `stETH`/`ETH` for `coll` or `collateral`). Badger should define a very strict standard for the naming convention and apply broadly to the whole codebase. Having a mixture of both creates confusion and could lead to errors.
19) In `CdpManagerStorage` `_getPendingRedistributedDebt` explicitly return set or return a value for `pendingEBTCDebtReward` because right now the `else` case is not handled and could lead to error/create confusion.
20) `IEBTCToken` `EBTCTokenBalanceUpdated` event can be removed because it's not used anywhere in the codebase.
21) In `HintHelpers` `_calculateCdpStateAfterPartialRedemption` the variable names should be updated to clarify when they store `stETH` or `shares`. 
22) In `LiquidationLibrary` use the same style to declare the revert message. Sometimes they start with "LiquidationLibrary: ", sometimes with "CdpManager:" and sometimes like in `_reInsertPartialLiquidation` they do not have the reference to the source at all.
23) Avoid using "magic number" `1e18` in `LiquidationLibrary._liquidateIndividualCdpSetupCDPInNormalMode`. Use `BaseMath.DECIMAL_PRECISION`.
24) Avoid using "magic number" `1e18` in `LiquidationLibrary._liquidateIndividualCdpSetupCDPInRecoveryMode`. Use `BaseMath.DECIMAL_PRECISION`.
25) In `LiquidationLibrary` the function is passing `stETH shares` and not `stETH` to the event `CdpLiquidated` for the parameter `_coll`. Please verify that it is implemented as intended.
26) In `LiquidationLibrary` to the event `CdpLiquidated` `_premiumToLiquidator` is defined as `_cappedColl > _debtToColl ? (_cappedColl - _debtToColl) : 0` where `_cappedColl` is calculated as `collateral.getPooledEthByShares(_cappedColPortion + _liquidatorReward)`.

The `_liquidatorReward` (the reward given to the liquidator as the gas stipend) should be outside the calculation of the `_cappedColl` and should be always added to the final value passed to the `_premiumToLiquidator` parameter. Here's an example:

```solidity
uint _cappedColl = collateral.getPooledEthByShares(_cappedColPortion);
uint256 premium = _cappedColl > _debtToColl ? (_cappedColl - _debtToColl) : 0
emit CdpLiquidated(
    _recoveryState.cdpId,
    _borrower,
    _totalDebtToBurn,
    _cappedColPortion,
    CdpOperation.liquidateInRecoveryMode,
    msg.sender,
    premium + collateral.getPooledEthByShares(_liquidatorReward)
);
```

27) `LiquidationSequencer` does not check if the grace period has passed (if we are in RM). See issue ["The liquidation logic is not always using the same requirement when the system is in RM"](https://github.com/cantinasec/review-badgerdao/issues/32).
28) `LiquidationSequencer` does not perform the same check done in `LiquidationLibrary` when we are in `RM`. Over there `_liquidateIndividualCdpSetup` require that `_ICR <= _TCR` instead in the helper `_canLiquidateInCurrentMode` check that `_icr < _TCR`. See issue ["The liquidation logic is not always using the same requirement when the system is in RM"](https://github.com/cantinasec/review-badgerdao/issues/32).
29) Update the revert message in `LiquidationSequencer` because now they say "LiquidationLibrary:".
30) Improvement: create a common library for the checks done by `LiquidationSequencer` and `LiquidationLibrary` to be sure to implement both of them correctly.
31) In `LiquidationLibrary` avoid using the direct value of the Cdp Status, use the Enum instead (example `_cdpStatus == 1`).
32) Remove `public` visibility from the `MultiCdpGetter` constructor (it's ignored by solidity).
33) In `ICdpManagerData` rename the `debtToOffset` attribute of the struct `LiquidationValues` to `debtToBurn`. The current name used is a concept related to `Liquity` and could create confusion in `eBTC`.
34) In `ICdpManagerData` rename the `totalDebtToOffset` attribute of the struct `LiquidationTotals` to `debtToBurn`. The current name used is a concept related to `Liquity` and could create confusion in `eBTC`.
35) In `CdpManager.redeemCollateral` the `require(redemptionsPaused == false, ...)` check can be moved at the very beginning of the function to save gas if the redemption process has been paused.
36) In `redeemCollateral()` `totals.totalETHDrawn` and `totals.ETHToSendToRedeemer` are in stETH **shares** and to be named accordingly.

**Recommendation:** Badger should consider solving the issues listed in the description section. The main suggestion would be to:
- Consolidate the renaming migration they have started to remove any possible confusion around the term collateral, `stETH` and `shares`.
- Remove references and nomenclatures that were relevant in the Liquity project but are not inside the eBTC context.
- Cover all the variables, functions and interfaces with full natspec support.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/33#issuecomment-1737528464) on: **9/27/2023**

Updated the listed issues with a couple of more points and modified some of them. I'll probably end up creating separate issues to regroup some of them.
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/33#issuecomment-1737592967) on: **9/27/2023**

Moved some specific points to the issue https://github.com/cantinasec/review-badgerdao/issues/38
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/33#issuecomment-1747494862) on: **10/4/2023**

36. In `redeemCollateral()` `totals.totalETHDrawn` and `totals.ETHToSendToRedeemer` are in stETH **shares** and to be named accordingly.
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/33#issuecomment-1748154243) on: **10/5/2023**

> 36. In `redeemCollateral()` `totals.totalETHDrawn` and `totals.ETHToSendToRedeemer` are in stETH **shares** and to be named accordingly.

I have added it to the list, @dmitriia feel free to edit my issue when you need it ;)
