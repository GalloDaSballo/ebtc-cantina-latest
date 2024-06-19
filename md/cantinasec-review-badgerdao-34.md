# [Refactor `LiquidationSequencer` to save gas and be more readable](https://github.com/cantinasec/review-badgerdao/issues/34)

> state: **open** opened by: **StErMi** on: **9/26/2023**

**Context:** [LiquidationSequencer.sol](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/LiquidationSequencer.sol)

**Description:**

The `LiquidationSequencer` contract can be refactored to save gas and be more readable.  Before the refactoring (or at least keep them in mind), Badger should first solve the issues that have been already reported:
- ["The liquidation logic is not always using the same requirement when the system is in RM"](https://github.com/cantinasec/review-badgerdao/issues/32).
- ["`LiquidationSequencer` is not synchronizing the platform when calculating the list of CDP"](https://github.com/cantinasec/review-badgerdao/issues/27).

1) Avoid re-calculatin the `TCR`.

The `TCR` of the system is already calculated inside `sequenceLiqToBatchLiqWithPrice` but it's also later re-calculated by `_sequenceLiqToBatchLiq` (that is only called by `sequenceLiqToBatchLiqWithPrice`).

The value returned by `_getTCRWithSystemDebtAndCollShares` and `cdpManager.getTCR(_price)` is the same, given that `getTCR` will internally call `_getTCRWithSystemDebtAndCollShares`.

It would make sense to re-calculate each time the `TCR` if the contract was simulating the consequence (and the increase of TCR) of a liquidation, but that's not the case.

2) Refactor the two for-loops of `_sequenceLiqToBatchLiq`.

90% of the code inside those two for loops is the same, and such logic could be moved to common functions that will be called by the loops. Such a refactor would make the code much more readable.

3) Consider breaking early in the first for-loop of `_sequenceLiqToBatchLiq` if a CDP is not liquidable because of its ICR or because of RM and Grace Period.

If a CDP cannot be liquidated because of its `ICR` and the sorted list is correctly sorted, it means that the "getPrev()" (with higher `ICR`) won't be liquidable too, and the next "getPrev()" CDP won't be and so on. Badger should consider breaking the loop early instead of keeping iterating until the reach `i < _n && _cdpId != _first` condition.

4) Consider removing the `_cdpStatus == 1` check.

Theoretically, the `sortedCdps` list should only include **active** CDPs, if a non-active CDP is in that list it means that something is wrong in the overall implementation of the protocol. Badger should consider verifying such statement and remove the check.

5) Consider refactoring the whole second for-loop in `_sequenceLiqToBatchLiq`.

If the `sortedCdps` list is correctly sorted, and it does not contain any non-active CDP (like it should), the second for-loop of the function should only iterate to add the CDPs to the `_array` array.

6) Consider refactoring the `constructor` to only receive the `_cdpManagerAddress` address and retrieve all the other addresses (`activePool`, `priceFeed`, `collateralAddress`, `sortedCdps`) by querying directly the `cdpManager`.

**Recommendation:** Badger should consider applying the suggestions provided in the Description part of the issue.

**Important Note:** Many of those suggestions rely on the strong hypothesis that the `sortedCdps` list is correctly sorted and does not contain non-active CDPs.

**BadgerDao:** Please check [PR 653](https://github.com/ebtc-protocol/ebtc/pull/653) for this optimization as suggested.

**Cantina:** The code seems good (from the logic prospective) but I can make the following notes:

1) `_canLiquidateInCurrentMode` function can be removed, and the code integrated directly into `_checkCdpLiquidability`. The logic is tiny and easy to read and is only called by `_checkCdpLiquidability`. I think that calling functions (because of the jump and so on) introduces gas overhead.
2) `_recoveryMode` can be pre-computed directly in `_sequenceLiqToBatchLiq`. That value (given that you do not influence TCR by actually liquidating the CDP) won't change, and you can avoid to re-calculate it each time.
3) I would say that you can further simplify the logic by doing:
	- First loop check which is the higher ICR CDP to be liquidated, and you return the iterations done. After that CDP, all the other CDPs can't be liquidated (for the current state of the protocol).
	- The second loop just iterates CDPs from zero for X iterations (where X is the value returned by the first loop).
4) Suggestion: does it make sense to have a different flavor of the function where you specify both the `_n` parameter and the `cdpStartIdx` from where you want to start liquidating? For example: I would like to liquidate 10 CDPs starting from (included) the CDP with id XYZ.

**Important Note:** As I mentioned previously, all of this logic only works **if and only if** the `sortedCdps` contains **correctly sorted** CDPs and **active** CDPs (something that should be true for a correct functioning of the protocol).

**BadgerDAO:**
- `LeverageMacro*` and `SimplifiedDiamondLike` will be handled by the related [PR 654](https://github.com/Badger-Finance/ebtc/pull/654).
- Discussion [`stETH` share historical value](https://github.com/cantinasec/review-badgerdao/discussions/2) is fixed as suggested.
- `BorrowerOperations` is fixed as suggested by setting the iteration upper bound of the second iteration as the returned value from the first iteration.
- `CdpManager` might be a interesting feature and I suggest to address it in a dedicated separate task if necessary.

Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/34#issuecomment-1735388946) on: **9/26/2023**

please check following PR for this optimization as suggested 

https://github.com/Badger-Finance/ebtc/pull/653
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/34#issuecomment-1735482993) on: **9/26/2023**

The code seems good (from the logic prospective) but I can make the following notes:

1) `_canLiquidateInCurrentMode` function can be removed, and the code integrated directly into `_checkCdpLiquidability`. The logic is tiny and easy to read and is only called by `_checkCdpLiquidability`. I think that calling functions (because of the jump and so on) introduces gas overhead.
2) `_recoveryMode` can be pre-computed directly in `_sequenceLiqToBatchLiq`. That value (given that you do not influence TCR by actually liquidating the CDP) won't change, and you can avoid to re-calculate it each time
3) I would say that you can further simplify the logic by doing
	- First loop check which is the higher ICR CDP to be liquidated, and you return the iterations done. After that CDP, all the other CDPs can't be liquidated (for the current state of the protocol)
	- The second loop just iterates CDPs from zero for X iterations (where X is the value returned by the first loop)
4) Suggestion: does it make sense to have a different flavor of the function where you specify both the `_n` parameter and the `cdpStartIdx` from where you want to start liquidating? For example: I would like to liquidate 10 CDPs starting from (included) the CDP with id XYZ

⚠️ As I mentioned previously, all of this logic only works **if and only if** the `sortedCdps` contains **correctly sorted** CDPs and **active** CDPs (something that should be true for a correct functioning of the protocol)



---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/34#issuecomment-1736674019) on: **9/27/2023**

@StErMi PR is updated with respect to your notes. 

Note #1 will be handled by another related PR https://github.com/Badger-Finance/ebtc/pull/654/files#diff-307f35bf6928957226c09fdb313749bb5c94edea6f623f9860b92a83c211322b
Note #2 is fixed as suggested
Note #3 is fixed as suggested by setting the iteration upper bound of the second iteration as the returned value from the first iteration
Note #4 might be a interesting feature and I suggest to address it in a dedicated separate task if necessary
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/34#issuecomment-1736734147) on: **9/27/2023**

I'll wait for the final merge to review the whole code
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/34#issuecomment-1736862722) on: **9/27/2023**

@rayeaster I have added another point to the issue (see point 6) where I suggest a refactor of the `constructor`. All the addresses that you are passing to the constructor can be retrieved by querying directly the `cdpManager`. 


---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/34#issuecomment-1738257256) on: **9/28/2023**

There are optimization possibilities for `_sequenceLiqToBatchLiq` [outlined in the previous review](https://github.com/spearbit-audits/review-badgerdao/pull/4/files#r1259014036), they are still relevant as far as I see.
---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/34#issuecomment-1738352865) on: **9/28/2023**

> @rayeaster I have added another point to the issue (see point 6) where I suggest a refactor of the `constructor`. All the addresses that you are passing to the constructor can be retrieved by querying directly the `cdpManager`.

@dapp-whisperer what is your opinion on this topic? Thought current constructor implementation with all related dependent addresses passed in as separate arguments is a consistent match with what we use for other contracts using `EBTCDeployer`.
---
> from: [**luksgrin**](https://github.com/cantinasec/review-badgerdao/issues/34#issuecomment-1787771914) on: **10/31/2023**

Will be marked as gas optimization only, for the report
