# [Do not allow the `LeverageMacroBase` to override the `approval` for `eBTC/stETH` during the `_doSwap` execution](https://github.com/cantinasec/review-badgerdao/issues/25)

> state: **open** opened by: **StErMi** on: **8/11/2023**

**Context:** [LeverageMacroBase.sol#L395-L398](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L395-L398), [LeverageMacroBase.sol#L417](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L417)

**Description:** The `LeverageMacroReference` contracts that inherits from `LeverageMacroBase` executes some crucial approvals to allow the contract to repay the flashloan operations:

```solidity
// set allowance for flashloan lender/CDP open
ebtcToken.approve(_borrowerOperationsAddress, type(uint256).max);
stETH.approve(_borrowerOperationsAddress, type(uint256).max);
stETH.approve(_activePool, type(uint256).max);
```

Those approvals are done once and cannot be re-done by the `LeverageMacroReference`. 

The `LeverageMacroBase._doSwap` allows the caller to specify both an arbitrary `tokenForSwap` token, `addressForApprove` address and `exactApproveAmount` that could allow the caller to override those approvals done during the `LeverageMacroReference` constructor.

If such an event happens, the `LeverageMacroReference` contract would not be able to repay any flashloan operation. Given that there is no way to re-do those initial approvals and that the flashloan operation is a hard requirement for the `doOperation` execution, it would mean that the `LeverageMacroReference` would probably not be able to execute any operation that involves the overridden token approval.

**Recommendation:** BadgerDAO should consider reverting the `_doSwap` operation when

- `tokenForSwap == ebtcToken` and `addressForApprove == borrowerOperationsAddress`.
- `tokenForSwap == stETH` and `addressForApprove == borrowerOperationsAddress`.
- `tokenForSwap == stETH` and `addressForApprove == activePool`.

**BadgerDao:** Thinking this could be solved by granting temporary allowance just to repay the flashloan.

**Cantina:** It could be an option which probably would be more security proof compared to giving max allowance.
The approval should be done **after** the execution of `_handleOperation` in `onFlashLoan` so you know for sure that any "custom" approval done by the `_doSwaps` in `_handleOperation` does not override the one done during the flashloan to repay the flashloan as soon as the context move back to `BorrowOperations` or `ActivePool`.

Probably `_openCdpCallback`, `_closeCdpCallback` and `_adjustCdpCallback` have to also do custom approvals with the needed amounts of `eBTC` and `stETH` to perform the operation.

To recap; it could make sense but:

1) you are going to increase gas for FL and eBTC interaction (open, adjust, close) because you have more calls to `approve`.
2) probably it would need `approve(0)` at least for `stETH` because of rounding down after the transfer? (see [lido's "Account's stETH balance getting lower on 1 or 2 wei due to rounding down integer math" issue](https://github.com/lidofinance/lido-dao/issues/442)).

**BadgerDao:** Wouldn't you agree with adding the approvals here at the end of `onFlashLoan`? We'd have the exact amount (amount + fee) to approve and it's always executed at the end of the Flashloan.

Opted to add a simple function to reset approvals in the `LeverageMacroReference
` (see [PR 579](https://github.com/ebtc-protocol/ebtc/pull/579)).

Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/25#issuecomment-1677140845) on: **8/14/2023**

Thinking this could be solved by granting temporary allowance just to repay the flashloan
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/25#issuecomment-1679016166) on: **8/15/2023**

> Thinking this could be solved by granting temporary allowance just to repay the flashloan

It could be an option which probably would be more security proof compared to giving max allowance.
The approval should be done **after** the execution of `_handleOperation` in `onFlashLoan` so you know for sure that any "custom" approval done by the `_doSwaps` in `_handleOperation` does not override the one done during the flashloan to repay the flashloan as soon as the context move back to `BorrowOperations` or `ActivePool`.

Probably `_openCdpCallback`, `_closeCdpCallback` and `_adjustCdpCallback` have to also do custom approvals with the needed amounts of `eBTC` and `stETH` to perform the operation.

To recap: it could make sense but 

1) you are going to increase gas for FL and eBTC interaction (open, adjust, close) because you have more calls to `approve`
2) probably it would need `approve(0)` at least for `stETH` because of rounding down after the transfer? (see https://github.com/lidofinance/lido-dao/issues/442)
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/25#issuecomment-1679316074) on: **8/15/2023**

<img width="683" alt="Screenshot 2023-08-15 at 19 15 28" src="https://github.com/cantinasec/review-badgerdao/assets/13383782/e379fe29-544c-4762-a696-8ac1d1f40d57">

@StErMi wouldn't you agree with adding the approvals here at the end of `onFlashLoan`?
We'd have the exact amount (amount + fee) to approve and it's always executed at the end of the Flashloan

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/25#issuecomment-1679316878) on: **8/15/2023**

I guess we could also add approvals before each operation as well
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/25#issuecomment-1679317255) on: **8/15/2023**

We could also just add a "reset approvals" function just in case
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/25#issuecomment-1680079596) on: **8/16/2023**

Opted to add a simple function to reset approvals in the LeverageMacroReference
https://github.com/Badger-Finance/ebtc/pull/579

