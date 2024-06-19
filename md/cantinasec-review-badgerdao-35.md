# [`SortedCdps` view functions `getCdpsOf`, `cdpCountOf`, `cdpOfOwnerByIndex` and `cdpOfOwnerByIdx` could return more information useful for the next pagination](https://github.com/cantinasec/review-badgerdao/issues/35)

> state: **open** opened by: **StErMi** on: **9/26/2023**

**Context:** [SortedCdps.sol#L114-L171](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/SortedCdps.sol#L114-L171), [SortedCdps.sol#L173-L214](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/SortedCdps.sol#L173-L214), [SortedCdps.sol#L216-L285](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/SortedCdps.sol#L216-L285)

**Description:**

The `cdpOfOwnerByIndex`, `cdpOfOwnerByIdx`, `_cdpOfOwnerByIndex`, `cdpCountOf`, `_cdpCountOf`, `getCdpsOf` and `_getCdpsOf` functions (with the different overloaded versions) have been added to the `SortedCdps` contract as a replaced for the removal of the state variables that were containing such information.

These functions are meant to be used off-chain or on-chain, but by "lens" contracts and not directly by the core eBTC code.
Because of the nature of the sorted linked list and the high probability to revert because of Out Of Gas exception, those functions support a "paginated" logic.

The current version of the functions are not returning some information that could help the caller to prepare the needed parameters for the next iteration of the "paginated" query.

- Let's say that the sorted list contains 10k records and an integrator wants to know the number of CDPs owned by Alice.
- Given the high number of records, the integrator cannot specify `maxNodes = 0` otherwise the transaction will revert.
- Given that the integrator doesn't know the information about the first ID owned by Alice (the first CDP owned by Alice with the lower ICR of her CDPs) it needs to specify `startNodeId`.
- Note that the integrator must iterate the whole list because it does not know the number of CDPs owned by Alice (the information it's trying to retrieve) and in the worst-case scenario, the last of her CDPs could be the one positioned in the `head` (the one with higher ICR).

At this point, the integrator will call `cdpCountOf(alice, dummyId, bigNumberOfIterationThatWillNotRevert)`. Let's say that it returns `X`. At this point, the integrator has to query for the next page of the iteration, but he does not know which `startNodeId` to pass as the off-set of the page.

Note that `_cdpOfOwnerByIndex` should not revert when the `maxNodes` has been exceeded to correctly handle the paginated result information.

**Recommendation:** Badger should consider improving the returned values of the `cdpOfOwnerByIndex`, `cdpOfOwnerByIdx`, `_cdpOfOwnerByIndex`, `cdpCountOf`, `_cdpCountOf`, `getCdpsOf` and `_getCdpsOf` functions to allow the integrators to properly query the Sorted CDPs list in a paginated manner.

**BadgerDAO:** Fixed as suggested in [PR 671](https://github.com/Badger-Finance/ebtc/pull/671). Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/35#issuecomment-1744419044) on: **10/3/2023**

fixed as suggested in PR https://github.com/Badger-Finance/ebtc/pull/671
