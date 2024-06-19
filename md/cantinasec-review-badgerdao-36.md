# [Consider refactoring `SortedCdps._getCdpsOf` to simply the code and save gas](https://github.com/cantinasec/review-badgerdao/issues/36)

> state: **open** opened by: **StErMi** on: **9/26/2023**

**Context:** [SortedCdps.sol#L247-L249](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/SortedCdps.sol#L247-L249), [SortedCdps.sol#L274-L282](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/SortedCdps.sol#L274-L282)

**Description:**

The `_getCdpsOf(address owner, bytes32 startNodeId, uint maxNodes, uint maxArraySize)` internal function can only be called by `getCdpsOf(address owner)` and `function getCdpsOf(address owner, bytes32 startNodeId, uint maxNodes)` that have the following code:

```solidity
function getCdpsOf(address owner) external view override returns (bytes32[] memory) {
    uint _ownedCount = _cdpCountOf(owner, dummyId, 0);
    return _getCdpsOf(owner, dummyId, 0, _ownedCount);
}

function getCdpsOf(
    address owner,
    bytes32 startNodeId,
    uint maxNodes
) external view override returns (bytes32[] memory) {
    uint _ownedCount = _cdpCountOf(owner, startNodeId, maxNodes);
    return _getCdpsOf(owner, startNodeId, maxNodes, _ownedCount);
}
```

The `owner`, `startNodeId` and `maxNodes` parameters value are the same passed to both the `_cdpCountOf` and `_getCdpsOf` functions:

- `owner` is the owner of the CDPs.
- `startNodeId` is the CDP ID from which it needs to start iterating from (in a pagination logic it can be seen as the `offset` concept).
- `maxNodes` is the max number of iterations that the logic should do on the list (in a pagination logic it can be seen as the `limit` concept).

The `_cdpCountOf` function iterates over the list starting from `startNodeId` and iterates at max `maxNodes` times. It will return the number of CDPs owned by the `onwer` parameter.

Because the parameters passed to those functions are the same and because they iterate over the same "slice" of the list and for the same max number of iterations: 

- `maxArraySize` will always be `<= maxNodes` (you cannot find more matches than the number of iterations). This will always be true when `maxNodes > 0`.
- `_cdpRetrieved` will always be equal to `maxArraySize`.

Because of this, the `_getCdpsOf` logic can be simplified as following:

```diff
  function _getCdpsOf(
      address owner,
      bytes32 startNodeId,
      uint maxNodes,
      uint maxArraySize
  ) internal view returns (bytes32[] memory) {
      if (maxArraySize == 0) {
          return new bytes32[](0);
      }

      // Two-pass strategy, halving the amount of Cdps we can process before relying on pagination or off-chain methods
-     bytes32[] memory userCdps = new bytes32[](
-        (maxNodes > 0 && maxNodes < maxArraySize) ? maxNodes : maxArraySize
-     );
+     bytes32[] memory userCdps = new bytes32[](maxArraySize);

      uint i = 0;
      uint _cdpRetrieved;

      // walk the list, until we get to the index
      // start at the given node or from the tail of list
      bytes32 _currentCdpId = (startNodeId == dummyId ? data.tail : startNodeId);

      while (_currentCdpId != dummyId) {
          // if the current Cdp is owned by specified owner
          if (getOwnerAddress(_currentCdpId) == owner) {
              userCdps[_cdpRetrieved] = _currentCdpId;
              ++_cdpRetrieved;
          }
          ++i;

          // move to the next Cdp in the list
          _currentCdpId = data.nodes[_currentCdpId].prevId;

          // cut the run if we exceed expected iterations through the loop
          if (maxNodes > 0 && i >= maxNodes) {
              break;
          }
      }

      // if CDP number retrieved not equal to expected then we make a new copy
-     if (_cdpRetrieved > 0 && _cdpRetrieved != userCdps.length) {
-         bytes32[] memory _copyUserCdps = new bytes32[](_cdpRetrieved);
-         for (uint i = 0; i < _cdpRetrieved; ++i) {
-             require(userCdps[i] != dummyId, "SortedCdps: invalid CDP retrieved by getCdpsOf()");
-             _copyUserCdps[i] = userCdps[i];
-         }
-         userCdps = _copyUserCdps;
-     }

      return userCdps;
  }
```

Note that `_cdpRetrieved` can't be equal to `0` because for that to be true it would mean that `maxArraySize` have to be equal to zero. This case is already handled in the very first check performed by `_getCdpsOf`.

**Recommendation:** Badger should consider:
- Reviewing and simplifying the code of the `SortedCdps_getCdpsOf` function.
- Adding more test and fuzzing tests to validate the proposed refactoring and possible edge cases.
- replacing all the `uint` types with the `uint256` to adhere to the Solidity Style guide.

**BadgerDao:** Fixed as suggested in [PR 658](https://github.com/Badger-Finance/ebtc/pull/658).

**Cantina:** The recommendations have been implemented in [PR 658](https://github.com/Badger-Finance/ebtc/pull/658).

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/36#issuecomment-1736810767) on: **9/27/2023**

fixed as suggested in PR https://github.com/Badger-Finance/ebtc/pull/658
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/36#issuecomment-1736847499) on: **9/27/2023**

The recommendations have been implemented in the PR https://github.com/Badger-Finance/ebtc/pull/658
---
> from: [**luksgrin**](https://github.com/cantinasec/review-badgerdao/issues/36#issuecomment-1787768264) on: **10/31/2023**

For the report, will mark as gas optimisation and remove the informational tag
