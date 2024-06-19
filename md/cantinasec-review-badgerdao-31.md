# [Possible confusion between borrower and delegatee when `CdpUpdated` events are emitted](https://github.com/cantinasec/review-badgerdao/issues/31)

> state: **open** opened by: **StErMi** on: **9/24/2023**

**Context:** [BorrowerOperations.sol#L466-L472](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/BorrowerOperations.sol#L466-L472), [BorrowerOperations.sol#L504C9-L504C9](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/BorrowerOperations.sol#L504C9-L504C9), [BorrowerOperations.sol#L531](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/BorrowerOperations.sol#L531), [BorrowerOperations.sol#L299](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/BorrowerOperations.sol#L299), [BorrowerOperations.sol#L366](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/BorrowerOperations.sol#L366)

**Description:**

The delegation system allows a delegate to execute three operations:
- Adjust CDP.
- Open CDP.
- Close CDP.

**Open CDP**

This operation executes :

```solidity
cdpManager.initializeCdp(
    _cdpId,
    vars.debt,
    _netCollAsShares,
    _liquidatorRewardShares,
    _borrower
);
```

That internally emit the event `CdpUpdated(_cdpId, _borrower, 0, 0, _debt, _coll, stake, CdpOperation.openCdp);`. In this case, the `_borrower` parameter of the event is the `_borrower` parameter passed as an input of the `_openCdp` function.

In this case, the event parameter seems to represent who's the final user that will own the CDP.

**Close CDP**

This operation executes :

```solidity
cdpManager.syncAccounting(_cdpId);
```

That internally **could** emit the event:

```solidity
emit CdpUpdated(
    _cdpId,
    ISortedCdps(sortedCdps).getOwnerAddress(_cdpId),
    prevDebt,
    prevColl,
    _newDebt,
    prevColl,
    Cdps[_cdpId].stake,
    CdpOperation.syncAccounting
);
```

In this case, the `_borrower` parameter of the event is **always** `ISortedCdps(sortedCdps).getOwnerAddress(_cdpId)`that represents the **owner** of the CDP and could be different compared to `msg.sender`. In this case, the event parameter represents the **owner** of the CDP.

At the end of the flow, the `closeCdp` function executes:

```solidity
cdpManager.closeCdp(_cdpId, msg.sender, debt, coll);
```

That internally emit the event `CdpUpdated(_cdpId, _borrower, _debt, _coll, 0, 0, 0, CdpOperation.closeCdp)`. In this case, the `_borrower` parameter of the event is **always** `msg.sender`. In this case, the event parameter seems to represent who has provided the funds to repay the debt and close the CDP and **not** the owner of the CDP.

**Adjust CDP**

This operation executes:

```solidity
cdpManager.syncAccounting(_cdpId);
```

That internally **could** emit the event: 

```solidity
emit CdpUpdated(
    _cdpId,
    ISortedCdps(sortedCdps).getOwnerAddress(_cdpId),
    prevDebt,
    prevColl,
    _newDebt,
    prevColl,
    Cdps[_cdpId].stake,
    CdpOperation.syncAccounting
);
```

In this case, the `_borrower` parameter of the event is **always** `ISortedCdps(sortedCdps).getOwnerAddress(_cdpId)`that represents the **owner** of the CDP and could be different compared to `msg.sender`. In this case, the event parameter represents the **owner** of the CDP.

At the end of the flow, the `_adjustCdpInternal` function executes:

```solidity
cdpManager.updateCdp(_cdpId, _borrower, vars.coll, vars.debt, vars.newColl, vars.newDebt);
```

That internally emits the event:

```solidity
emit CdpUpdated(
    _cdpId,
    _borrower,
    _debt,
    _coll,
    _newDebt,
    _newColl,
    stake,
    CdpOperation.adjustCdp
);
```

In this case, the `_borrower` parameter of the event is the **owner** of the CDP retrieved by `_adjustCdpInternal` at the start of the flow by executing `sortedCdps.getOwnerAddress(_cdpId)`. In this case, the event parameter represents the owner of the CDP, even if the one that is adjusting the CDP (`msg.sender`) could be different from the owner.

**Recommendation:**

As shown in the description above, the `_borrower` parameter of the event `CdpUpdated` does not assume always the same meaning, and it is not always the borrower (owner of the CDP).

Badger should consider refactoring those function to always pass to the event the owner of the CDP, or otherwise provide a specific explanation on why in some cases it assumes a value that is different from the CDP's owner.

**BadgerDAO:** A quick summary from above findings

| operation   | `_borrower` parameter in event `CdpUpdated`|
| --------       | ------- |
| openCdp    | [CDP owner]  |
| closeCdp    | `msg.sender` passed to `CdpManager.closeCdp()`|
| adjustCdp   | [CDP owner] |

Note that `CdpUpdated` emit by `CdpManager.syncAccounting()` **always** use [CDP owner] for `_borrower`.

Do we agree that to make it consistent by replacing `msg.sender` in `BorrowerOperations.closeCdp()` with [CDP owner]?

```diff
-- cdpManager.closeCdp(_cdpId, msg.sender, debt, coll);
++ cdpManager.closeCdp(_cdpId, _borrower, debt, coll);
```

`msg.sender` is better because we can get the `Owner` via theGraph binding in the event handler. Wisdom says just add both to the event.

Doing CDP owner in [PR 688](https://github.com/ebtc-protocol/ebtc/pull/688).

Acknowledged.

**Cantina:** Acknowledged.


### Comments

---
> from: [**rayeaster**](https://github.com/cantinasec/review-badgerdao/issues/31#issuecomment-1740340974) on: **9/29/2023**

a quick summary from above findings

| operation   | `_borrower` parameter in event `CdpUpdated`|
| --------       | ------- |
| openCdp    | [CDP owner]  |
| closeCdp    | `msg.sender` passed to `CdpManager.closeCdp()`|
| adjustCdp   | [CDP owner] |

Note that `CdpUpdated` emit by `CdpManager.syncAccounting()` **always** use [CDP owner] for `_borrower`

@dapp-whisperer do we agree that to make it consistent by replacing `msg.sender` in `BorrowerOperations.closeCdp()` with [CDP owner]?

```diff
-- cdpManager.closeCdp(_cdpId, msg.sender, debt, coll);
++ cdpManager.closeCdp(_cdpId, _borrower, debt, coll);
```

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/31#issuecomment-1742958352) on: **10/2/2023**

@dapp-whisperer should ask Basado here

2 cents are that msg.sender is better because we can get the Owner via theGraph binding in the event handler
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/31#issuecomment-1742958691) on: **10/2/2023**

Wisdom says just add both to the event
---
> from: [**dapp-whisperer**](https://github.com/cantinasec/review-badgerdao/issues/31#issuecomment-1758436251) on: **10/11/2023**

doing [CDP owner] 
