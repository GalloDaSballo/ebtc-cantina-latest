# [There is no way to invalidate issued permits for eBTC and position managers](https://github.com/cantinasec/review-badgerdao/issues/39)

> state: **open** opened by: **dmitriia** on: **9/28/2023**

**Context:** [EBTCToken.sol#L181-L203](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/EBTCToken.sol#L181-L203), [BorrowerOperations.sol#L637-L671](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/BorrowerOperations.sol#L637-L671)

**Description:** It's now impossible to invalidate previously issued permits other than use them for both eBTC token and position manager approvals. There is a deadline argument control, but it doesn't always provide enough flexibility. Both permits are material, for example position manager can close a CDP, obtaining all the collateral.

As an example vector, a permit issued with any long dated `_deadline` can render revoking the approval void:

```solidity
/// @notice Revoke a position manager from taking further actions on your Cdps
/// @notice Similar to approving tokens, approving a position manager allows _stealing of all positions_ if given to a malicious account.
function revokePositionManagerApproval(address _positionManager) external override {
    _setPositionManagerApproval(msg.sender, _positionManager, PositionManagerApproval.None);
}
```

I.e. if there is a permit it does not matter if approval was revoked, as it can be restored, for example front or back-running funds movements. 

**Recommendation:** Consider adding a nonce increasing function as a way to quickly invalidate current permits, e.g.:

```solidity
/// @notice Clears outstanding permits for the current nonce
function increaseNonce() external returns (uint256) {
    return ++_nonces[msg.sender];
}
```

**BadgerDao:** Agree with fixing this + adding to UI as a way to Revoke any delegation. Fixed as suggested in [PR 672](https://github.com/ebtc-protocol/ebtc/pull/672).

**Cantina:** Fix looks ok, `increasePermitNonce()` is added via `PermitNonce` parent.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/39#issuecomment-1742933414) on: **10/2/2023**

Agree with fixing this + adding to UI as a way to Revoke any delegation
---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/39#issuecomment-1744519273) on: **10/3/2023**

fixed as suggested in PR https://github.com/Badger-Finance/ebtc/pull/672
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/39#issuecomment-1745009420) on: **10/3/2023**

> fixed as suggested in PR [Badger-Finance/ebtc#672](https://github.com/Badger-Finance/ebtc/pull/672)

Fix looks ok, `increasePermitNonce()` is added via `PermitNonce` parent.
