# [`LeverageMacroBase` does not allow the owner of the contract to perform CDP operations without performing a Flashloan](https://github.com/cantinasec/review-badgerdao/issues/22)

> state: **open** opened by: **StErMi** on: **8/11/2023**

**Context:** [LeverageMacroBase.sol#L161](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L161)

**Description:** The current implementation of `LeverageMacroBase` reverts if the `doOperation` does not execute a `stETH` or `eBTC` flashloan operation.

There could be cases for which the owner of the contract could want to be able to adjust or close the CDP without performing a flashloan.

**Recommendation:** BadgerDAO should consider allowing the caller of `LeverageMacroBase.doOperation` to perform the `LeverageMacroOperation operation` without executing a flashloan.

**BadgerDao:** Addressed in [PR 564](https://github.com/ebtc-protocol/ebtc/pull/564). Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/22#issuecomment-1674868014) on: **8/11/2023**

https://github.com/Badger-Finance/ebtc/pull/564
