# [Some event parameters can be declared as `indexed` to better monitor those events with monitoring tools](https://github.com/cantinasec/review-badgerdao/issues/38)

> state: **open** opened by: **StErMi** on: **9/27/2023**

**Context:** [ICdpManagerData.sol#L47](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/ICdpManagerData.sol#L47), [ICdpManagerData.sol#L56C3-L56C3](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/ICdpManagerData.sol#L56C3-L56C3), [ICdpManagerData.sol#L16](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/ICdpManagerData.sol#L16), [ICdpManagerData.sol#L29](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/ICdpManagerData.sol#L29), [ICollSurplusPool.sol#L9](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/ICollSurplusPool.sol#L9), [IERC3156FlashLender.sol#L8](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IERC3156FlashLender.sol#L8), [IERC3156FlashLender.sol#L9](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IERC3156FlashLender.sol#L9), [IERC3156FlashLender.sol#L10](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IERC3156FlashLender.sol#L10), [IFeeRecipient.sol#L8](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IFeeRecipient.sol#L8), [IPool.sol#L11](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IPool.sol#L11), [IPositionManagers.sol#L13-L14](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IPositionManagers.sol#L13-L14), [IBorrowerOperations.sol#L10](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IBorrowerOperations.sol#L10), [IBorrowerOperations.sol#L11](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IBorrowerOperations.sol#L11), [IPriceFeed.sol#L9](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IPriceFeed.sol#L9), [IPriceFeed.sol#L10](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.4/packages/contracts/contracts/Interfaces/IPriceFeed.sol#L10)

**Description:** Some events contain parameters that could be declared as `indexed`. By doing that, Badger can later use those parameters to filters those events by those parameters value via dApps and monitoring system:

- `ICdpManagerData` `event` `CdpLiquidated` can declare the `_liquidator` parameter as `indexed`.
- `ICdpManagerData` event `CdpPartiallyLiquidated` can declare the `_liquidator` parameter as `indexed.
- `ICdpManagerData` event `FeeRecipientAddressChanged` can declare the `_feeRecipientAddress` input as `indexed`.
- `ICollSurplusPool` event `CollSharesTransferred` can declare the `_to` parameter as `indexed`.
- `IERC3156FlashLender` event `FlashFeeSet` can declare the `_setter` parameter as `indexed`.
- `IERC3156FlashLender` event `MaxFlashFeeSet` can declare the `_setter` parameter as `indexed`.
- `IERC3156FlashLender` event `FlashLoansPaused` can declare the `_setter` parameter as `indexed`.
- `IFeeRecipient` event `CollSharesTransferred` can declare the `_account` parameter as `indexed`.
- `IPool` event `CollSharesTransferred` can declare the `_to` parameter as `indexed`.
- `IPositionManagers` event `PositionManagerApprovalSet` can declare the `_borrower` and `_positionManager` parameters as `indexed`.
- `IBorrowerOperations` event `FeeRecipientAddressChanged` can declare the `_feeRecipientAddress` input as `indexed`.
- `IBorrowerOperations` event `FlashLoanSuccess` can declare the `_receiver` and `_token` inputs as `indexed.
- `ICdpManagerData` event `Redemption` can declare the `_redeemer` input as `indexed`.
- `IPriceFeed` event `FallbackCallerChanged` can declare the `_oldFallbackCaller` and `_newFallbackCaller` inputs as `indexed`.
- `IPriceFeed` event `UnhealthyFallbackCaller` can declare the `_fallbackCaller` input as `indexed`.

**Recommendation:** Badger should consider renaming the event's parameters listed in the description as `indexed`.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/38#issuecomment-1742940430) on: **10/2/2023**

@dapp-whisperer let's ask Basado as well
