# [`EBTCToken.transferFrom` decrease the allowance of `(owner, spender)` even when the allowance is set to `type(uint256).max`](https://github.com/cantinasec/review-badgerdao/issues/20)

> state: **open** opened by: **StErMi** on: **8/11/2023**

**Context:** [EBTCToken.sol#L142-L144](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/EBTCToken.sol#L142-L144)

**Description:** While it's not defined in the [EIP-20](https://eips.ethereum.org/EIPS/eip-20), it's a common implementation (see both OpenZeppelin and Solmate) that the `transferFrom` function of an `ERC20` token does not decrease the allowance of the spender when such allowance has been set to the max value `type(uint256).max`.

The current implementation of `EBTCToken` does not follow this logic and decrease it by the `amount` transferred from the `sender` to the `recipient`

```solidity
unchecked {
    _approve(sender, msg.sender, cachedAllowances - amount);
}
```

This behavior could create problems in contracts that are used to a more common behavior like the one used in OpenZeppelin/Solmate and have approved only once (without a way to update such value) the `EBTCToken`.
The result is that at some point in the future, the `transferFrom` operation will revert because the `spender` would not have enough allowance anymore.

A contract that is already keen to this problem is `LeverageMacroReference`, an immutable contract that executes [`ebtcToken.approve(_borrowerOperationsAddress, type(uint256).max);`](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroReference.sol#L39) only once when the constructor is executed.

At some point, an instance of the contract could not be able to perform operations like

- Adjusting the CDP (by repaying some `eBTC` debt).
- Close the CDP (by repaying all the CDP debt).
- Repaying the `eBTC` flashloaned `amount + fee`.

**Recommendation:** BadgerDAO should follow the same logic used by the OpenZeppelin/Solmate implementation of the ERC20 token: the `transferFrom` function should not update the `spender` allowance if such allowance is equal to `type(uint256).max`.

**BadgerDao:** Addressed in [PR 567](https://github.com/Badger-Finance/ebtc/pull/567).

**Cantina:** The recommendations have been implemented in [PR 567](https://github.com/Badger-Finance/ebtc/pull/567).
A suggestion I could make is to move the check approval/update approval part of the code **before** `_transfer(sender, recipient, amount);`.

From a logical/security point of the view `transferFrom` functions should follow these steps:

1) verify that the spender has enough allowance.
2) update the spender's allowance.
3) transfer the tokens from `sender` to `receiver`.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/20#issuecomment-1674917145) on: **8/11/2023**

https://github.com/Badger-Finance/ebtc/pull/567/files
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/20#issuecomment-1676046352) on: **8/12/2023**

The recommendations have been implemented in the PR https://github.com/Badger-Finance/ebtc/pull/567

@GalloDaSballo a suggestion I could make is to move the check approval/update approval part of the code **before** `_transfer(sender, recipient, amount);`.

From a logical/security point of the view `transferFrom` functions should follow these steps

1) verify that the spender has enough allowance
2) update the spender's allowance
3) transfer the tokens from `sender` to `receiver`
