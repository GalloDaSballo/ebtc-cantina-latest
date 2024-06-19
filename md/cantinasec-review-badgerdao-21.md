# [`LeverageMacroBase.sweepToCaller` could transfer `stETH shares` instead of "pure" `stETH` to avoid leaving dust into the contract](https://github.com/cantinasec/review-badgerdao/issues/21)

> state: **open** opened by: **StErMi** on: **8/11/2023**

**Context:** [LeverageMacroBase.sol#L226](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L226)

**Description:** From the Lido official documentation about `stETH/wstETH` integration guide, there is a specific paragraph about ["1-2 wei corner case"](https://docs.lido.fi/guides/steth-integration-guide#1-2-wei-corner-case).

> stETH balance calculation includes integer division, and there is a common case when the whole stETH balance can't be transferred from the account while leaving the last 1-2 wei on the sender's account. The same thing can actually happen at any transfer or deposit transaction. In the future, when the stETH/share rate will be greater, the error can become a bit bigger. To avoid it, one can use transferShares to be precise.

Lido itself suggests using `transferShares` instead of `transferFrom` to avoid this `1-2 wei corner case`.


**Recommendation:** BadgerDAO could consider using `stETH.transferShares` instead of `stETH.transfer` in the `LeverageMacroBase.sweepToCaller` execution. If BadgerDAO decides to chose so, it should also update `collateralBal` value properly get the correct share balance of the contract by executing `stETH.sharesOf(address(this));`.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/21#issuecomment-1674869883) on: **8/11/2023**

https://github.com/Badger-Finance/ebtc/compare/fix-steth-shares?expand=1
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/21#issuecomment-1674891803) on: **8/11/2023**

> [Badger-Finance/ebtc@`fix-steth-shares`?expand=1 (compare)](https://github.com/Badger-Finance/ebtc/compare/fix-steth-shares?expand=1)

Would you mind referencing all the instances? That link contains quite a few changes :D
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/21#issuecomment-1674916700) on: **8/11/2023**

https://github.com/Badger-Finance/ebtc/pull/568
