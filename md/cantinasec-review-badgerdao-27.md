# [`LiquidationSequencer` is not synchronizing the platform when calculating the list of CDP](https://github.com/cantinasec/review-badgerdao/issues/27)

> state: **open** opened by: **StErMi** on: **9/24/2023**

**Context:** [LiquidationSequencer.sol#L35-L87](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/LiquidationSequencer.sol#L35-L87)

**Description:**

The current implementation of `LiquidationSequencer` is not executing the `syncGlobalAccountingAndGracePeriod` what would: 
- update the `stETH` indexes.
- claim the splitting fees if needed (and remove them from the share amount of collateral).
- start the Grace Period if needed.

By not doing that, the following values could be outdated and invalid (compared to the one later used by the `CdpManager.batchLiquidateCdps()`):
- TCR.
- Recover Mode/Normal Mode.
- CDP's ICR that is calculated by calling `getICR` (and not `getSyncedICR`).

Because of this, the list of liquidable CDP returned by the function could include or exclude CDPs that instead should be excluded or included.

**Recommendation:**

Badger should follow the same logic, behavior and checks implemented by the `LiquidationLibrary.batchLiquidateCdps` function.

**BadgerDao:** Fixed as suggested in [PR 653](https://github.com/ebtc-protocol/ebtc/pull/653).

**Cantina:** Seems good to me. Please also consider the comments done in issue ["The liquidation logic is not always using the same requirement when the system is in RM"](https://github.com/cantinasec/review-badgerdao/issues/32).

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/27#issuecomment-1733144240) on: **9/25/2023**

fixed as suggested in https://github.com/Badger-Finance/ebtc/pull/653
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/27#issuecomment-1733725914) on: **9/25/2023**

> fixed as suggested in [Badger-Finance/ebtc#653](https://github.com/Badger-Finance/ebtc/pull/653)

Seems good to me. Please also consider the comments I've done here https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1733711940
