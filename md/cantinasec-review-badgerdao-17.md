# [Leverage contracts should fetch the protocol contract addresses from `cdpManager` instead of being passed as constructor parameter ](https://github.com/cantinasec/review-badgerdao/issues/17)

> state: **open** opened by: **StErMi** on: **8/10/2023**

**Context:** [LeverageMacroBase.sol#L51-L68](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroBase.sol#L51-L68), [LeverageMacroDelegateTarget.sol#L41-L60](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroDelegateTarget.sol#L41-L60), [LeverageMacroFactory.sol#L21-L35](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroFactory.sol#L21-L35), [LeverageMacroReference.sol#L17-L42](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/LeverageMacroReference.sol#L17-L42)

**Description:** All the `Leverage*` contracts (`LeverageMacroBase`, `LeverageMacroDelegateTarget`, `LeverageMacroFactory` and `LeverageMacroReference`) are initializing the `immutable` variables of `borrowerOperations`, `activePool`, `cdpManager`, `ebtcToken`, `sortedCdps` and `stETH` with the corresponding value passed down from the contract's `constructor`.

To reduce the human error and the possibility that the wrong value is passed to such `constructor`, BadgerDAO could consider to only passing the `cdpManager` contract into the `constructor` and then gathering all the other addresses by querying the `cdpManager` itself.

- `borrowerOperations` can be fetched by calling `cdpManager.borrowerOperationsAddress()`.
- `activePool` can be fetched by calling `cdpManager.activePool()`.
- `ebtcToken` can be fetched by calling `cdpManager.ebtcToken()`.
- `sortedCdps` can be fetched by calling `cdpManager.sortedCdps()`.
- `stETH` can be fetched by calling `cdpManager.collateral()`.

**Recommendation:** BadgerDAO should consider gathering the addresses needed to initialize `borrowerOperations`, `activePool`, `ebtcToken`, `sortedCdps` and `stETH` from the `cdpManager` instead of passing them down as `constructor` inputs.

This would reduce the possibility of human error or of a misconfiguration of the `Leverage*` contracts (`LeverageMacroBase`, `LeverageMacroDelegateTarget`, `LeverageMacroFactory` and `LeverageMacroReference`).

**BadgerDao:** Acknowledging as the system is immutable, there's a chance of making a mistake initially but I don't think there's a risk.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/17#issuecomment-1674901754) on: **8/11/2023**

Acknowledging as the system is immutable, there's a chance of making a mistake initially but I don't think there's a risk 
