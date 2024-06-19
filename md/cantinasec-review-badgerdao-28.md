# [`PermitPositionManagerApproval` should use the correct type for the `status` parameter](https://github.com/cantinasec/review-badgerdao/issues/28)

> state: **open** opened by: **StErMi** on: **9/24/2023**

**Context:** [BorrowerOperations.sol#L27-L30](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/BorrowerOperations.sol#L27-L30)

**Description:**

The `status` parameter of the signature refers to the `enum PositionManagerApproval` value. The `Enum` type in Solidity can represent at max 256 values and are of type `uint8`.

Consider to be consistent with the real type value of the `enum PositionManagerApproval` type, and change the `_PERMIT_POSITION_MANAGER_TYPEHASH` accordingly.

**Recommendation:**

Badger should consider updating the `_PERMIT_POSITION_MANAGER_TYPEHASH` to use `uint8` for the `status` signature parameter to match the native type of the `enum PositionManagerApproval`.

**BadgerDao:** Agree with fixing this, nonce is `uint8`. Fixed in [commit 2ce7fe04 of `ebtc`](https://github.com/ebtc-protocol/ebtc/commit/2ce7fe044bc6452474c14ab0477dd709e4149de9). Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/28#issuecomment-1742933704) on: **10/2/2023**

Agree with fixing this, nonce is uint8
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/28#issuecomment-1742956959) on: **10/2/2023**

Fixed here: https://github.com/Badger-Finance/ebtc/pull/668/commits/2ce7fe044bc6452474c14ab0477dd709e4149de9
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/28#issuecomment-1744269566) on: **10/3/2023**

LGTM
