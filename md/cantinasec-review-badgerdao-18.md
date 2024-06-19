# [EIP-3156 requires `flashFee()` and `maxFlashLoan()` to accommodate their logic to `flashLoansPaused` flag](https://github.com/cantinasec/review-badgerdao/issues/18)

> state: **open** opened by: **dmitriia** on: **8/11/2023**

**Context:** [BorrowerOperations.sol#L816-L828](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L816-L828), [ActivePool.sol#L311-L332](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/ActivePool.sol#L311-L332)

**Description:** Per [ERC-3156](https://eips.ethereum.org/EIPS/eip-3156#lender-specification) `flashFee()` can revert, while `maxFlashLoan()` can return `0` when `flashLoansPaused == true`:


> The maxFlashLoan function MUST return the maximum loan possible for token. If a token is not currently supported maxFlashLoan MUST return 0, instead of reverting.

>The flashFee function MUST return the fee charged for a loan of amount token. If the token is not supported flashFee MUST revert.


Now the flag is ignored in the logic:

- [BorrowerOperations.sol#L816-L828](https://github.com/Badger-Finance/ebtc/blob/1967a223c7948faf3a5711bd485061f51449c404/packages/contracts/contracts/BorrowerOperations.sol#L816-L828)

```solidity
    function flashFee(address token, uint256 amount) public view override returns (uint256) {
        require(token == address(ebtcToken), "BorrowerOperations: EBTC Only");

        return (amount * feeBps) / MAX_BPS;
    }

    /// @dev Max flashloan, exclusively in ETH equals to the current balance
    function maxFlashLoan(address token) public view override returns (uint256) {
        if (token != address(ebtcToken)) {
            return 0;
        }
        return type(uint112).max;
    }
```

On the same grounds as eBTC, `flashFee()` and `maxFlashLoan()` need to change their behavior when `flashLoansPaused == true` for ActivePool's stETH flash loans.

Impact: in both cases flash loan logic do not comply with EIP-3156 when `flashLoansPaused` is on, which can break the integrations.

**Recommendation:** `flashFee()` can revert, while `maxFlashLoan()` can return `0` when `flashLoansPaused == true` in both cases.

**BadgerDao:** Addressed in [PR 574](https://github.com/ebtc-protocol/ebtc/pull/574).

**Cantina:** The recommendations have been implemented in [PR 574](https://github.com/ebtc-protocol/ebtc/pull/574).

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/18#issuecomment-1677139373) on: **8/14/2023**

https://github.com/Badger-Finance/ebtc/pull/574
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/18#issuecomment-1679019995) on: **8/15/2023**

The recommendations have been implemented in the PR https://github.com/Badger-Finance/ebtc/pull/574

@GalloDaSballo ping me when you merge the PR, so I can update the issue body+status
---
> from: [**dapp-whisperer**](https://github.com/cantinasec/review-badgerdao/issues/18#issuecomment-1683561651) on: **8/18/2023**

we're merging @StErMi 
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/18#issuecomment-1729694334) on: **9/21/2023**

Fix looks ok
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/18#issuecomment-1729776681) on: **9/21/2023**

LGTM
