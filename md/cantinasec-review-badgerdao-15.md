# [Replace `.transfer`, `.transferFrom` and `.approve` with the corresponding ERC20 "safe" version of them](https://github.com/cantinasec/review-badgerdao/issues/15)

> state: **open** opened by: **StErMi** on: **8/10/2023**

**Context:** [LeverageMacroBase.sol#L226](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L226), [LeverageMacroBase.sol#L395-L398](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L395-L398), [LeverageMacroBase.sol#L417](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L417), [LeverageMacroReference.sol#L40-L41](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroReference.sol#L40-L41), [BorrowerOperations.sol#L533](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/BorrowerOperations.sol#L533), [ActivePool.sol#L276](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/ActivePool.sol#L276), [ActivePool.sol#L285](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/ActivePool.sol#L285), [ActivePool.sol#L288](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/ActivePool.sol#L288)

**Description:** While it's true that `stETH` is following the `ERC20` standard and the `approve`, `transfer` and `transferFrom` on their token always returns `true` or reverts it's recommended to anyway, use the corresponding "safe" version of those function to be bullet-proof for the future. Lido could decide at some point to upgrade their `stETH` token to an implementation that does not follow the `ERC20` standard anymore.

The suggestion is even more highly recommended when such operations are performed on an external and arbitrary `ERC20` token.

Note that the current implementation of `stETH` `.transferShares` and `.transferSharesFrom` do not return `bool` value, but instead the amount of `stETH` tokens that have been moved from the `sender` to the `recipient` account.

**Recommendation:** BadgerDAO should consider replacing the `.transfer`, `.transferFrom` and `.approve` calls with the corresponding ERC20 "safe" version of them.

**BadgerDAO:** Fixed in `LeverageProxy` (see [PR 565](https://github.com/ebtc-protocol/ebtc/pull/565)), but acknowledge the rest. Will follow up. Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/15#issuecomment-1674869341) on: **8/11/2023**

https://github.com/Badger-Finance/ebtc/pull/565

Fixed in LeverageProxy but ack in rest, will follow up
