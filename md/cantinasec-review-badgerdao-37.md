# [`CRLens` contract considerations and improvements](https://github.com/cantinasec/review-badgerdao/issues/37)

> state: **open** opened by: **StErMi** on: **9/27/2023**

**Context:** [CRLens.sol](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/CRLens.sol)

**Description/Recommendation:**

The `CRLens` contract is a utility contract that allows external entities to query the CDP Manager. 

1) Create a `ICRLens` interface that will be inherited by `CRLens` and that can be used by external integrators to build on top or query `CRLens`.
2) Add full natspec support to `CRLens` (or the `ICRLens` interface). Many functions have only a partial natspec support.
3) Consider renaming the function by replacing "real" with "synched". The `CdpManager` have already adopted this term to identify those functions that synch the system and CDP indexes, apply fee split and update the CDP state.
4) Delete the `price` variable inside `getRealNICR` because is never used.
5) Consider returning a `bool` instead of a `uint256` in `quoteCheckRecoveryMode`. The recovery mode is a `true/false` value, and it could be confusing for the integrators to receive a `uint256` value that can only assume `0/1`. The function could be refactored like this:

```diff
- function quoteCheckRecoveryMode() external returns (uint256) {
+ function quoteCheckRecoveryMode() external returns (bool) {
      try this.getCheckRecoveryMode(true) {} catch (bytes memory reason) {
-          return parseRevertReason(reason);
+          return parseRevertReason(reason) == 1;
      }
  }
```
6) Consider removing the `_priceFeed` dependency from the `CRLens.constructor`. The Price Feed in the `CdpManager` is an immutable variable and can be directly retrieved from it by executing `cdpManager.priceFeed()`. Here's an example of a possible refactoring:

```diff
- constructor(address _cdpManager, address _priceFeed) {
+ constructor(address _cdpManager) {
      cdpManager = ICdpManager(_cdpManager);
-     priceFeed = IPriceFeed(_priceFeed);
+     priceFeed = cdpManager.priceFeed();
  }
```
7) Consider further documenting the usage of `CRLens`. While it's true that this contract will probably be used only by integrators or advanced users, the documentation about the proper usage (and the revert mechanism) could be further improved and better explained.

**BadgerDao:** Not fixing as the `CRLens` is mostly a tool to debug.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/37#issuecomment-1742935781) on: **10/2/2023**

Nofixing as the CRLens is mostly a tool to debug
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/37#issuecomment-1744270260) on: **10/3/2023**

Ack
