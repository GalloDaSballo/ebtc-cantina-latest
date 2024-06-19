# [LeverageMacroBase's _doCheckValueType() condition looks to be incorrectly reverted](https://github.com/cantinasec/review-badgerdao/issues/19)

> state: **open** opened by: **dmitriia** on: **8/11/2023**

**Context:** [LeverageMacroBase.sol#L236-L247](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/LeverageMacroBase.sol#L236-L247)

**Description:** Given the usage, say for the `debt >= minDebt` check in:

- [LeverageMacroBase.sol#L175-L176](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/LeverageMacroBase.sol#L175-L176)

```solidity
_doCheckValueType(cdpInfo.debt, checkParams.expectedDebt);
_doCheckValueType(cdpInfo.coll, checkParams.expectedCollateral);
```

it looks like `_doCheckValueType()` needs to compare first argument against the second according to the operator type.

**Recommendation:** Consider making the conditions reverted:

- [LeverageMacroBase.sol#L236-L247](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/LeverageMacroBase.sol#L236-L247)

```diff
  /// @dev Assumes that
  ///     >= you prob use this one
  ///     <= if you don't need >= you go for lte
  ///     And if you really need eq, it's third
  function _doCheckValueType(uint256 valueToCheck, CheckValueAndType memory check) internal {
      if (check.operator == Operator.skip) {
          // Early return
          return;
      } else if (check.operator == Operator.gte) {
-         require(check.value >= valueToCheck, "!LeverageMacroReference: gte post check");
+         require(valueToCheck >= check.value, "!LeverageMacroBase: gte post check");
      } else if (check.operator == Operator.lte) {
-         require(check.value <= valueToCheck, "!LeverageMacroReference: let post check");
+         require(valueToCheck <= check.value, "!LeverageMacroBase: lte post check");
```

**BadgerDao:** Addressed in [PR 573](https://github.com/ebtc-protocol/ebtc/pull/573).

**Cantina:** The PR just switched the order of the inputs but has not changed the check (where the problem was). What the check should ensure is that the `valueToCheck` respects the validation and not the inverse.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/19#issuecomment-1676996396) on: **8/14/2023**

https://github.com/Badger-Finance/ebtc/pull/573
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/19#issuecomment-1729746685) on: **9/21/2023**

It looks like the fix didn't changed the logic
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/19#issuecomment-1729767407) on: **9/21/2023**

> It looks like the fix didn't changed the logic

Agree with what @dmitriia said. The PR just switched the order of the inputs but has not changed the check (where the problem was).

What the check should ensure is that the `valueToCheck` respects the validation and not the inverse.
