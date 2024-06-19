# [Outdated comments, error messages, naming across the codebase](https://github.com/cantinasec/review-badgerdao/issues/24)

> state: **open** opened by: **dmitriia** on: **8/11/2023**

**Context:** [CdpManagerStorage.sol#L160-L163](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/CdpManagerStorage.sol#L160-L163), [CdpManagerStorage.sol#L328-L331](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/CdpManagerStorage.sol#L328-L331), [CdpManagerStorage.sol#L500-L502](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/CdpManagerStorage.sol#L500-L502), [PriceFeed.sol#L783-L789](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/PriceFeed.sol#L783-L789), [LeverageMacroBase.sol#L26](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/LeverageMacroBase.sol#L26), [PriceFeed.sol#L789-L806](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/PriceFeed.sol#L789-L806), [BorrowerOperations.sol#L439-L442](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L439-L442), [BorrowerOperations.sol#L81-L92)](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L81-L92), [BorrowerOperations.sol#L162-L165](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L162-L165), [BorrowerOperations.sol#L176-L178](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L176-L178), [BorrowerOperations.sol#L189-L191](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L189-L191), [BorrowerOperations.sol#L219-L223](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L219-L223), [BorrowerOperations.sol#L605-L624](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L605-L624)

**Description:** Comments, error messages, naming can be corrected, per list below.

**Recommendation:** `_closeCdpWithoutRemovingSortedCdps()` error message:

- [CdpManagerStorage.sol#L160-L163](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/CdpManagerStorage.sol#L160-L163):

```diff
  require(
      closedStatus != Status.nonExistent && closedStatus != Status.active,
-     "CdpManagerStorage: close non-exist or non-active CDP!"
+     "CdpManagerStorage: close non-exist or active CDP!"
  );
```

Check is already done in the only caller above, `_closeCdpWithoutRemovingSortedCdps()`:

- [CdpManagerStorage.sol#L328-L331](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/CdpManagerStorage.sol#L328-L331):

```diff
- require(
-      cdpStatus != Status.nonExistent && cdpStatus != Status.active,
-     "CdpManagerStorage: remove non-exist or non-active CDP!"
- );
```

Naming:

- [CdpManagerStorage.sol#L500-L502](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/CdpManagerStorage.sol#L500-L502):

```diff
- function _requireMoreThanOneCdpInSystem(uint CdpOwnersArrayLength) internal view {
+ function _requireMoreThanOneCdpInSystem(uint CdpIdsArrayLength) internal view {
      require(
-         CdpOwnersArrayLength > 1 && sortedCdps.getSize() > 1,
+         CdpIdsArrayLength > 1 && sortedCdps.getSize() > 1,
          // ...
```

`_formatClAggregateAnswer()` description can substitute `stETH:BTC` feed that aren't used with `stETH:ETH` one in `_stEthEthAnswer` and `_stEthEthDecimals` params:

- [PriceFeed.sol#L783-L789](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/PriceFeed.sol#L783-L789):

```solidity
// @notice Returns the price of stETH:BTC in 18 decimals denomination
// @param _ethBtcAnswer CL price retrieve from ETH:BTC feed
// @param _stEthEthAnswer CL price retrieve from stETH:BTC feed
// @param _ethBtcDecimals ETH:BTC feed decimals
// @param _stEthEthDecimals stETH:BTC feed decimalss
// @return The aggregated calculated price for stETH:BTC
function _formatClAggregateAnswer(
    // ...
```

LeverageMacroBase can implement `IERC3156FlashBorrower`:

- [LeverageMacroBase.sol#L26](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/LeverageMacroBase.sol#L26)

```solidity
contract LeverageMacroBase {
    // ...
```

`_formatClAggregateAnswer()` logic can be simplified to:

```solidity
(uint256(_ethBtcAnswer) * uint256(_stEthEthAnswer) * LiquityMath.DECIMAL_PRECISION) /
10 ** (_stEthEthDecimals + _ethBtcDecimals)
```

Current implementation is a bit more prone to overflows (can break when bigger decimals number exceeds `30`):

- [PriceFeed.sol#L789-L806](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/PriceFeed.sol#L789-L806)

```solidity
function _formatClAggregateAnswer(
    int256 _ethBtcAnswer,
    int256 _stEthEthAnswer,
    uint8 _ethBtcDecimals,
    uint8 _stEthEthDecimals
) internal view returns (uint256) {
    uint256 _decimalDenominator = _stEthEthDecimals > _ethBtcDecimals
        ? _stEthEthDecimals
        : _ethBtcDecimals;
    uint256 _scaledDecimal = _stEthEthDecimals > _ethBtcDecimals
        ? 10 ** (_stEthEthDecimals - _ethBtcDecimals)
        : 10 ** (_ethBtcDecimals - _stEthEthDecimals);
    return
        (_scaledDecimal *
            uint256(_ethBtcAnswer) *
            uint256(_stEthEthAnswer) *
            LiquityMath.DECIMAL_PRECISION) / 10 ** (_decimalDenominator * 2);
}
```

These comments need to be removed or rewritten:

`closeCdp()` description:

- [BorrowerOperations.sol#L439-L442](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L439-L442):

```diff
  /**
- allows a borrower to repay all debt, withdraw all their collateral, and close their Cdp. Requires the borrower have a eBTC balance sufficient to repay their cdp's debt, excluding gas compensation - i.e. `(debt - 50)` eBTC.
+ allows a borrower to repay all debt, withdraw all their collateral, and close their Cdp.
  */
  function closeCdp(bytes32 _cdpId) external override {
      // ...
```

- [BorrowerOperations.sol#L81-L92](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L81-L92)

```solidity
constructor(
    // ...
) LiquityBase(_activePoolAddress, _priceFeedAddress, _collTokenAddress) {
    // This makes impossible to open a cdp with zero withdrawn EBTC
    // TODO: Re-evaluate this
    // ...
```

- [BorrowerOperations.sol#L162-L165](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L162-L165)

```solidity
/**
Withdraws `_collWithdrawal` amount of collateral from the caller’s Cdp. Executes only if the user has an active Cdp, the withdrawal would not pull the user’s Cdp below the minimum collateralization ratio, and the resulting total collateralization ratio of the system is above 150%.
*/
function withdrawColl(
    // ...
```

- [BorrowerOperations.sol#L176-L178](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L176-L178)

```solidity
// ...
Issues `_amount` of eBTC from the caller’s Cdp to the caller. Executes only if the Cdp's collateralization ratio would remain above the minimum, and the resulting total collateralization ratio is above 150%.
 */
function withdrawEBTC(
    // ...
```

- [BorrowerOperations.sol#L189-L191](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L189-L191)

```solidity
// ...
repay `_amount` of eBTC to the caller’s Cdp, subject to leaving 50 debt in the Cdp (which corresponds to the 50 eBTC gas compensation).
*/
function repayEBTC(
    // ...
```

- [BorrowerOperations.sol#L219-L223](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L219-L223)

```solidity
/**
enables a borrower to simultaneously change both their collateral and debt, subject to all the restrictions that apply to individual increases/decreases of each quantity with the following particularity: if the adjustment reduces the collateralization ratio of the Cdp, the function only executes if the resulting total collateralization ratio is above 150%. The borrower has to provide a `_maxFeePercentage` that he/she is willing to accept in case of a fee slippage, i.e. when a redemption transaction is processed first, driving up the issuance fee. The parameter is ignored if the debt is not increased with the transaction.
*/
// TODO optimization candidate
function adjustCdpWithColl(
// ...
```

- [BorrowerOperations.sol#L605-L624](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L605-L624)

```solidity
function _requireValidAdjustmentInCurrentMode(
    bool _isRecoveryMode,
    uint _collWithdrawal,
    bool _isDebtIncrease,
    LocalVariables_adjustCdp memory _vars
) internal view {
    /*
     *In Recovery Mode, only allow:
     *
     * - Pure collateral top-up
     * - Pure debt repayment
     * - Collateral top-up with debt repayment
     * - A debt increase combined with a collateral top-up which makes the
     * ICR >= 150% and improves the ICR (and by extension improves the TCR).
     *
     * In Normal Mode, ensure:
     *
     * - The new ICR is above MCR
     * - The adjustment won't pull the TCR below CCR
     */
```

**BadgerDao:** Some comments may still be off. Acknowledged.

**Cantina:** Acknowledged.


### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/24#issuecomment-1729340919) on: **9/21/2023**

Some comments may still be off
