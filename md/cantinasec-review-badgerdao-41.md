# [BorrowerOperations and CdpManager function names and error messages don't fully reflect the logic with regard to ordering required](https://github.com/cantinasec/review-badgerdao/issues/41)

> state: **open** opened by: **dmitriia** on: **10/4/2023**

**Context:** [BorrowerOperations.sol#L880](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/BorrowerOperations.sol#L880), [BorrowerOperations.sol#L786](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/BorrowerOperations.sol#L786), [BorrowerOperations.sol#L848](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/BorrowerOperations.sol#L848), [BorrowerOperations.sol#L855](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/BorrowerOperations.sol#L855), [BorrowerOperations.sol#L859](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/BorrowerOperations.sol#L859), [BorrowerOperations.sol#L866](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/BorrowerOperations.sol#L866), [CdpManager.sol#L747](https://github.com/Badger-Finance/ebtc/blob/feat/release-0.5/packages/contracts/contracts/CdpManager.sol#L747)

**Description:** Function names and error messages do not strictly correspond to the logic, per list below.

**Recommendation:** BorrowerOperations.sol:

Error messages can be updated, e.g.:

```diff
  function _requireAtLeastMinNetStEthBalance(uint256 _coll) internal pure {
      require(
          _coll >= MIN_NET_COLL,
-         "BorrowerOperations: Cdp's net coll must be greater than minimum"
+         "BorrowerOperations: Cdp's net coll must not fall below minimum"
      );
  }
```

and

```diff
  function _requireNoStEthBalanceDecrease(uint256 _stEthBalanceDecrease) internal pure {
      require(
          _stEthBalanceDecrease == 0,
-         "BorrowerOperations: Collateral withdrawal not permitted Recovery Mode"
+         "BorrowerOperations: Collateral withdrawal not permitted during Recovery Mode"
      );
  }
```

Function names:

`_requireICRisAboveMCR()` can be renamed to `_requireICRisNotBelowMCR`:

```solidity
function _requireICRisAboveMCR(uint256 _newICR) internal pure {
    require(
        _newICR >= MCR,
        "BorrowerOperations: An operation that would result in ICR < MCR is not permitted"
    );
}
```

`_requireICRisAboveCCR()` can be renamed to `_requireICRisNotBelowCCR`:

```solidity
function _requireICRisAboveCCR(uint256 _newICR) internal pure {
    require(_newICR >= CCR, "BorrowerOperations: Operation must leave cdp with ICR >= CCR");
}
```

`_requireNewICRisAboveOldICR()` can be renamed to `_requireNoDecreaseOfICR()`:

```solidity
function _requireNewICRisAboveOldICR(uint256 _newICR, uint256 _oldICR) internal pure {
    require(
        _newICR >= _oldICR,
        "BorrowerOperations: Cannot decrease your Cdp's ICR in Recovery Mode"
    );
}
```

`_requireNewTCRisAboveCCR()` can be renamed to `_requireNewTCRisNotBelowCCR()`:

```solidity
function _requireNewTCRisAboveCCR(uint256 _newTCR) internal pure {
    require(
        _newTCR >= CCR,
        "BorrowerOperations: An operation that would result in TCR < CCR is not permitted"
    );
}
```

`CdpManager.sol`:

`_requireTCRoverMCR()` can be renamed to `_requireTCRisNotBelowMCR()`:

```solidity
function _requireTCRoverMCR(uint256 _price, uint256 _TCR) internal view {
    require(_TCR >= MCR, "CdpManager: Cannot redeem when TCR < MCR");
}
```

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

