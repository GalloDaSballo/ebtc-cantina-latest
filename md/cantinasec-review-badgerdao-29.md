# [External entities could start/end the grace period, even when the grace period should not be started/ended](https://github.com/cantinasec/review-badgerdao/issues/29)

> state: **open** opened by: **StErMi** on: **9/24/2023**

**Context:** [CdpManagerStorage.sol#L77-L87](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/CdpManagerStorage.sol#L77-L87)

**Description:**

The `syncGracePeriod` in `CdpManagerStorage` allows anyone to synchronize the grace period based on the current system debt, collateral and price. The problem with the implementation is that it does not trigger the synchronization of the `stETH` index and the distribution of the splitting fees that could bring the system in `RM` or "normal mode" based on the new index value, the amount of fee split and the price of `stETH`:

```solidity
function syncGracePeriod() public {
    uint256 price = priceFeed.fetchPrice();
    uint256 tcr = _getTCR(price);
    bool isRecoveryMode = _checkRecoveryModeForTCR(tcr);

    if (isRecoveryMode) {
        _startGracePeriod(tcr);
    } else {
        _endGracePeriod(tcr);
    }
}
```

Based on the status of the system and the status of the grace period, calling the function could reset the grace period when it should not be resetted or start a grace period when it should not be started.

**Recommendation:**

Badger should remove the function and move the logic of it directly inside `syncGlobalAccountingAndGracePeriod` (the only function that is directly calling it).

External entities that want to only synchronize the grace period should be allowed to do so only via the execution of `syncGlobalAccountingAndGracePeriod`.


**BadgerDao:** Addressed in [PR 667](https://github.com/ebtc-protocol/ebtc/pull/667).

**Cantina:** The recommendations have been correctly implemented in [PR 667](https://github.com/ebtc-protocol/ebtc/pull/667).

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/29#issuecomment-1740326046) on: **9/29/2023**

PR is here https://github.com/Badger-Finance/ebtc/pull/667
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/29#issuecomment-1740330723) on: **9/29/2023**

The recommendations have been correctly implemented in the PR https://github.com/Badger-Finance/ebtc/pull/667
