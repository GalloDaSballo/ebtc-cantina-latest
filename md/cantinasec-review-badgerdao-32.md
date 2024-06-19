# [The liquidation logic is not always using the same requirement when the system is in RM](https://github.com/cantinasec/review-badgerdao/issues/32)

> state: **open** opened by: **StErMi** on: **9/24/2023**

**Context:** [LiquidationLibrary.sol#L72-L75](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/LiquidationLibrary.sol#L72-L75), [LiquidationLibrary.sol#L719-L722](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/LiquidationLibrary.sol#L719-L722), [CdpManagerStorage.sol#L812-L820](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/CdpManagerStorage.sol#L812-L820), [LiquidationSequencer.sol#L89-L97](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/LiquidationSequencer.sol#L89-L97)

**Description:**

The liquidation process is handled by the `LiquidationLibrary` that can execute the liquidation from three different functions:

- `liquidate(...)` to liquidate fully a single CDP.
- `partiallyLiquidate(...)` to liquidate partially a single CDP.
- `batchLiquidateCdps(...)` to liquidate an array of CDP in bulk.

These functions are called via `delegatecall` by the `CdpManager` contract. The "single CDP liquidation" functions will allow the liquidation of a CDP when the system is in Recovery Mode **if and only if**:

- `TCR < CCR`.
- `_ICR <= _TCR`.
- `cachedLastGracePeriodStartTimestamp != UNSET_TIMESTAMP && block.timestamp > cachedLastGracePeriodStartTimestamp + recoveryModeGracePeriod`.

If we look at the logic of `batchLiquidateCdps` when the system is in Recovery Mode it will execute `_getTotalFromBatchLiquidate_RecoveryMode` that is allowing the liquidation of a CDP (when we are in recovery mode) **if and only if** `ICR < MCR || canLiquidateRecoveryMode(vars.ICR, _TCR) == true`.

If we look at `CdpManagerStorage.canLiquidateRecoveryMode`  we see that the logic is different compared to the one we have found in the single liquidation:

```solidity
function canLiquidateRecoveryMode(uint256 icr, uint256 tcr) public view returns (bool) {
    // ICR < TCR and we have waited enough
    uint128 cachedLastGracePeriodStartTimestamp = lastGracePeriodStartTimestamp;
    return
        icr < tcr &&
        cachedLastGracePeriodStartTimestamp != UNSET_TIMESTAMP &&
        block.timestamp > cachedLastGracePeriodStartTimestamp + recoveryModeGracePeriod;
}
```

In this case, the check is stricter compared to the single liquidation that allows the liquidation even when `ICR == TCR`.

**`LiquidationSequencer`**

This contract allows an external user to generate the list of liquidable CDP to be passed later to the `CdpManager.batchLiquidateCdps(...)` function as a parameter.
In this case, a CDP is added to the list of liquidable CDP only if `status == 1` (the CDP is open and active) and `_canLiquidateInCurrentMode(_recoveryModeAtStart, _icr, _TCR) == true`.

The logic of `_canLiquidateInCurrentMode` is also different compared to the one used by the batch liquidation and single liquidation we have seen before:

```solidity
function _canLiquidateInCurrentMode(
    bool _recovery,
    uint256 _icr,
    uint256 _TCR
) internal view returns (bool) {
    bool _liquidatable = _recovery ? (_icr < MCR || _icr < _TCR) : _icr < MCR;

    return _liquidatable;
}
```

If we are in Recover Mode, the CDP is liquidable if `ICR < MCR || ICR < TCR`. In this case, the logic does not check at all the Grace Period.

This means that if we are in Recover Mode, it could happen that some of the CDP that are selected by the `LiquidationSequencer` won't be liquidated when `CdpManager.batchLiquidateCdps(...)` is executed.

**Recommendation:**

Badger should enforce that the same check for the liquidation is executed in both the single (full and partial) and bulk liquidation. Badger should also update the logic of `LiquidationSequencer` to only select CDP that can be liquidated by the `CdpManager`.

**BadgerDao:** Fixed in [PR 654](https://github.com/Badger-Finance/ebtc/pull/654). A related PR for `LiquidationSequencer` is [PR 653](https://github.com/Badger-Finance/ebtc/pull/653).

**Cantina:** I don't understand why you don't want to include the check about the grace period inside the `LiquidationSequencer`. What's the reason? Could you elaborate? By not doing that, you are including in the array some CDP that should not be liquidated.

Also, right now the `LiquidationSequencer` is "wasting" gas because it's calculating every time the RM flag that is static for the whole loop. The same "waste" is done by re-calculating each time the `TCR` that is already calculated in `sequenceLiqToBatchLiqWithPrice`.

Probably the best thing to do (to offer to the integrator) would be to "simulate" the liquidation of those CDPs and include only which are the "correct" CDP to liquidate because while you liquidate each CDP the RM and TCR will change and some of the included CDPs could end up being not liquidable. I don't know how much doable (how much it makes sense for gas cost) to do it onchain, so probably you should just document this scenario?

I think the optimal way is to ultimately get rid of duplications and call the very same function in core logic and in all the helpers. Full simulation can be ok for one instrument, but an overkill for another, it can be differentiated provided with proper descriptions of the limitations.

**BadgerDAO**: Agree with your point that it might be the best for integrator to have an exact/accurate "simulation" but it entails more work at this final stage before launch (e.g., we need to write new low-level functions to calculate the TCR with delta-change values instead using directly current collateral & debt in `ActivePool` as shown in `LiquityBase._getTCRWithSystemDebtAndCollShares()`). 

Considering the original purpose of separating this sequential-liquidation feature from core `LiquidationLibrary` into a dedicated "peripheral" contract, we could document as suggested that it should be expected that the result might contain CDPs that may not be liquidatable (thus skipped in later liquidation execution). 

The LiquidationSequencer's initial role is as a replacement to the `liquidateCdps(n)` function that was removed, because we use it throughout the testing suite. It is a periphery helper contract, and is intended for off-chain use. The points brought up are great, and it makes sense that a better method of simulating could be developed. As it's a periphery helper, this can be developed at another time.

**Cantina:** I understand that because it won't perform actual liquidations (and update the state), the returned array could be bigger compared to the array of CDPs that will be actually liquidated. 

But it should try at his best to select only what can be liquidated by applying all the selection filters, and the Grace Period is one of them.

Given how skilled you are, it could probably also make sense to release a python/typescript script (or foundry one) that simply forks the chain and really simulates the liquidation to select just the list of CDPs that can be liquidated. 

**BadgerDAO**: Considering moving to `<=` sign. Acknowledged.

**Cantina:** Acknowledged.



### Comments

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1733303393) on: **9/25/2023**

fixed in https://github.com/Badger-Finance/ebtc/pull/654

a related PR for `LiquidationSequencer` is at https://github.com/Badger-Finance/ebtc/pull/653
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1733711940) on: **9/25/2023**

@rayeaster I don't understand why you don't want to include the check about the grace period inside the `LiquidationSequencer`. What's the reason? Could you elaborate? By not doing that, you are including in the array some CDP that should not be liquidated.

Also, right now the `LiquidationSequencer` is "wasting" gas because it's calculating every time the RM flag that is static for the whole loop. The same "waste" is done by re-calculating each time the `TCR` that is already calculated in `sequenceLiqToBatchLiqWithPrice`.

Probably the best thing to do (to offer to the integrator) would be to "simulate" the liquidation of those CDPs and include only which are the "correct" CDP to liquidate because while you liquidate each CDP the RM and TCR will change and some of the included CDPs could end up being not liquidable. I don't know how much doable (how much it makes sense for gas cost) to do it onchain, so probably you should just document this scenario?

@dmitriia what are your thoughts?
---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1734985508) on: **9/26/2023**

> @rayeaster I don't understand why you don't want to include the check about the grace period inside the `LiquidationSequencer`. What's the reason? Could you elaborate? By not doing that, you are including in the array some CDP that should not be liquidated.
> 
> Also, right now the `LiquidationSequencer` is "wasting" gas because it's calculating every time the RM flag that is static for the whole loop. The same "waste" is done by re-calculating each time the `TCR` that is already calculated in `sequenceLiqToBatchLiqWithPrice`.
> 
> Probably the best thing to do (to offer to the integrator) would be to "simulate" the liquidation of those CDPs and include only which are the "correct" CDP to liquidate because while you liquidate each CDP the RM and TCR will change and some of the included CDPs could end up being not liquidable. I don't know how much doable (how much it makes sense for gas cost) to do it onchain, so probably you should just document this scenario?
> 
> @dmitriia what are your thoughts?

@StErMi agree with your point that it might be the best for integrator to have an exact/accurate "simulation" but it entails more work at this final stage before launch (e.g., we need to write new low-level functions to calculate the TCR with delta-change values instead using directly current collateral & debt in `ActivePool` as shown in `LiquityBase._getTCRWithSystemDebtAndCollShares()`). 

Considering the original purpose of separating this sequential-liquidation feature from core `LiquidationLibrary` into a dedicated "peripheral" contract, we could document as suggested that it should be expected that the result might contain CDPs that may not be liquidatable (thus skipped in later liquidation execution). 

@dapp-whisperer what is your insight on this topic?
---
> from: [**dapp-whisperer**](https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1740023081) on: **9/28/2023**

> tl;dr: Let's acknowledge the limitations of this sequencer 

The LiquidationSequencer's initial role is as a replacement to the liquidateCdps(n) function that was removed, because we use it throughout the testing suite.

It is a periphery helper contract, and is intended for off-chain use.

Stermi brings up great points, and it makes sense that a better method of simulating could be developed.

As it's a periphery helper, this can be developed at another time.
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1740334702) on: **9/29/2023**

I understand that because it won't perform actual liquidations (and update the state), the returned array could be bigger compared to the array of CDPs that will be actually liquidated. 

But it should try at his best to select only what can be liquidated by applying all the selection filters, and the Grace Period is one of them.

Given how skilled you are, it could probably also make sense to release a python/typescript script (or foundry one) that simply forks the chain and really simulates the liquidation to select just the list of CDPs that can be liquidated. 
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1742961692) on: **10/2/2023**

Let's chat about this today @dapp-whisperer 
---
> from: [**dapp-whisperer**](https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1743242337) on: **10/2/2023**

considering moving to <= sign
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/32#issuecomment-1745229665) on: **10/3/2023**

> @rayeaster I don't understand why you don't want to include the check about the grace period inside the `LiquidationSequencer`. What's the reason? Could you elaborate?
> ... 
> @dmitriia what are your thoughts?

I think the optimal way is to ultimately get rid of duplications and call the very same function in core logic and in all the helpers. Full simulation can be ok for one instrument, but an overkill for another, it can be differentiated provided with proper descriptions of the limitations. 
