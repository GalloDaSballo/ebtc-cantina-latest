# [Governance can to end or extend the grace period even when a grace period is already ongoing](https://github.com/cantinasec/review-badgerdao/issues/30)

> state: **open** opened by: **StErMi** on: **9/24/2023**

**Context:** [CdpManagerStorage.sol#L112-L124](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/CdpManagerStorage.sol#L112-L124)

**Description:**

The `setGracePeriod` function allows an authed user to update the `recoveryModeGracePeriod` state variables that directly influence in real time the status of the grace period.

By allowing the user to change the duration of the grace period even when a grace period has already started, it allows the authed user to end the grace period earlier than expected or extend it more than it was expected once triggered.

- Scenario 1: `recoveryModeGracePeriod = 60 minutes`, `lastGracePeriodStartTimestamp = 30 minutes ago`. If `setGracePeriod(5 minutes)` is executed, it allows the authed user to end the ongoing grace period before expected.
- Scenario 2: `recoveryModeGracePeriod = 60 minutes`, `lastGracePeriodStartTimestamp = 59 minutes ago`. If `setGracePeriod(200 minutes)` is executed, it allows the authed user to extend the ongoing grace period even if it would have ended in `1 minute`.

**Recommendation:**

If Badger considers the behavior correct, it should:
1) document this behavior in the `setGracePeriod` natspec documentation
2) document this behavior on their website/UI to let the user know of this possibility

If this behavior is not expected and there is an ongoing grace period event, the `newGracePeriod` duration should be saved in a temporary state variable and used only once the current grace period is ended (in order to not directly influence the ongoing grace period duration).
If no grace period has started yet but would be triggered by `syncGlobalAccountingAndGracePeriod`executed in `setGracePeriod`, Badger should probably consider updating `recoveryModeGracePeriod` before executing `syncGlobalAccountingAndGracePeriod` that otherwise would trigger the new grace period with the "old" duration.

**BadgerDao:** I believe we should acknowledge this as it's a race condition that requires multiple external scenarios to happen.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/30#issuecomment-1742933049) on: **10/2/2023**

I believe we should ack this as it's a race condition that requires multiple external scenarios to happen
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/30#issuecomment-1744271730) on: **10/3/2023**

Ack
