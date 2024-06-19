# [Use the correct function "State Mutability" when needed](https://github.com/cantinasec/review-badgerdao/issues/16)

> state: **open** opened by: **StErMi** on: **8/10/2023**

**Context:** [LeverageMacroBase.sol#L334](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L334), [LeverageMacroBase.sol#L423](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L423), [LeverageMacroBase.sol#L438](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L438), [LeverageMacroDelegateTarget.sol#L63](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroDelegateTarget.sol#L63), [LeverageMacroReference.sol#L44](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroReference.sol#L44)

**Description:** Here's the list of the function that could be declared with a more specific and restrictive "State Mutability":

- `LeverageMacroBase.decodeFLData` can be declared as `pure`.
- `LeverageMacroBase._doSwapChecks` can be declared as `view`.
- `LeverageMacroBase._ensureNotSystem` can be declared as `view`.
- `LeverageMacroDelegateTarget.owner` can be declared as `view`.
- `LeverageMacroReference.owner` can be declared as `view`.

**Recommendation:** BadgerDAO should consider replacing the function "state mutability" property with the suggested one.

**BadgerDao:** Thinking of keeping it as is to ensure any odd future compatibility since the functions are "developer only".

**Cantina:** Could you elaborate it more? Those functions cannot be overridden because they are not `virtual` so I'm not understanding the "odd future compatibility" part.

**BadgerDAO:** Acknowledging as of now to avoid having to change this in the future.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/16#issuecomment-1677139979) on: **8/14/2023**

Thinking of keeping it as is to ensure any odd future compatibility since the functions are "developer only"
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/16#issuecomment-1677140221) on: **8/14/2023**

@dapp-whisperer wdyt?
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/16#issuecomment-1678997181) on: **8/15/2023**

> Thinking of keeping it as is to ensure any odd future compatibility since the functions are "developer only"

Could you elaborate it more? Those functions cannot be overridden because they are not `virtual` so I'm not understanding the "odd future compatibility" part
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/16#issuecomment-1729342786) on: **9/21/2023**

Acknowledging as of now to avoid having to change this in the future
