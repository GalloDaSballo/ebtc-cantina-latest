# [[BCCR] CDP closing can be used for whale sniping](https://github.com/cantinasec/review-badgerdao/issues/23)

> state: **open** opened by: **dmitriia** on: **8/11/2023**

**Context:** [BorrowerOperations.sol#L451-L475](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#L451-L475)

**Description:** After BCCR introduction on CDP closing only `newTCR > CCR` is required, a whale sniper like scenario looks to be possible with an attacker preliminary opening two big CDPs (each can be a set of CDPs for granularity): first a `good` well capitalized one, at `goodICR >> BCCR`, and then one `bad`, at `badICR = MCR + epsilon1`, which is possible as `good` first one drags TCR up above BCCR.

Now, on observing any event that decreases the TCR, e.g. slashing (rare) or big enough oracle reading downtick (frequent enough), the attacker can front run the event transaction and remove `good` CDP (some combination of them), so that `newTCR = CCR + epsilon2`, and the event transaction triggers RM, which attacker back-runs with target liquidation.

Overall this requires CDP prepositioning and index/price update transaction sandwiching. Since keeping the said CDPs has low risk, the total cost for the attacker is gas and the cost of the capital involved. It can be viable given big enough expected profit up to `10%` of the whale collateral.

- [BorrowerOperations.sol#L451-L475](https://github.com/Badger-Finance/ebtc/blob/46f8df604fe7e8b57cb0ac5ec2aca7530e87bfe0/packages/contracts/contracts/BorrowerOperations.sol#L451-L475)

```solidity
function closeCdp(bytes32 _cdpId) external override {
    // ...

    uint newTCR = _getNewTCRFromCdpChange(
        collateral.getPooledEthByShares(coll),
        false,
        debt,
        false,
        price
    );
    // See the line below
    _requireNewTCRisAboveCCR(newTCR);

    cdpManager.removeStake(_cdpId);
    // ...
```

Impact: protocol manipulation in order to liquidate a specific CDP that isn't liquidatable by itself is possible via `closeCdp()`.

Per medium likelihood and high principal funds loss impact for CDP owner setting the severity to be high.

**Recommendation:** Since after BCCR introduction CDP adjusting is controlled at `135` TCR, while closing is at `125` TCR anyway (i.e. it's not a free exit), it might be reasonably to consider controlling it at `130` as a part of BCCR introduction. I.e. on the one hand it's not desirable to leave the surface open, on the another adding the very same control as for opening/adjusting might be too tight as closing is both more user sensitive operation and the cost and complexity of the attack is higher in this case.


**BadgerDao:** Agree with the finding, we have mitigated by adding a Grace Period (15 minutes) before allowing Recovery Mode to trigger liquidations (see [CdpManagerStorage.sol#L21-L124](https://github.com/ebtc-protocol/ebtc/blob/1b0a073075484670f24947e0f455031144d3a1e0/packages/contracts/contracts/CdpManagerStorage.sol#L21-L124)).

**Cantina:** Overall 15 minutes seems short as the only window for target whale borrower reaction in the whale sniping scenario. In the same time it is the period for which attacker bears the risk, so when this period is short it's both narrow window for possible target reaction and low enough risk for the attacker.

On the other hand, since infinite grace period means no liquidations in RM while `ICR >= MCR`, the max grace period is also due for `setGracePeriod()`.

The changes shown in [CdpManagerStorage.sol#L21-L124](https://github.com/ebtc-protocol/ebtc/blob/1b0a073075484670f24947e0f455031144d3a1e0/packages/contracts/contracts/CdpManagerStorage.sol#L21-L124) look ok. Somewhat higher lower boundary and an introduction of the upper one are suggested, i.e. the grace period can be bounded to `[1 hour, 2 days]`, as an example.

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.


### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/23#issuecomment-1680152988) on: **8/16/2023**

## Proper Attack

    They can always do it

    -> Raise Up (so that they allow it to be safe to raise down)
    -> Raise down (so that it's at RM + ε)

Example:
```python
SAFE_COLL 300
SAFE_DEBT 150.0
risky_coll 770
risky_debt 700.0
get_cr(total_coll, total_debt) 125.88235294117646
To Raise to 135 add:
Extra Coll for Setup 40 ## This is the extra cost of setup + the capital for attack which is part of the Liquity Model
```


## Script

```python
def get_debt(coll, cr):
    return coll / cr * 100

def get_cr(coll, debt):
    return coll / debt * 100

SAFE_COLL = 300
SAFE_CR = 200
SAFE_DEBT = get_debt(SAFE_COLL, SAFE_CR)
print("SAFE_COLL", SAFE_COLL)
print("SAFE_DEBT", SAFE_DEBT)

RISKY_CR = 110
SYSTEM_CR = 126
BUFFER_CR = 135

def main():
    total_coll = SAFE_COLL
    total_debt = SAFE_DEBT
    risky_coll = 0
    risky_debt = 0


    ## Get attacker CR
    while get_cr(total_coll, total_debt) > SYSTEM_CR:
        risky_coll += 10
        risky_debt = get_debt(risky_coll, RISKY_CR)

        total_coll = SAFE_COLL + risky_coll
        total_debt = SAFE_DEBT + risky_debt

    print("risky_coll", risky_coll)
    print("risky_debt", risky_debt)
    print("get_cr(total_coll, total_debt)", get_cr(total_coll, total_debt))

    print("To Raise to 135 add:")
    ## We allow COLL ONLY
    ## Just deposit Coll until new CR is 135 instead of 125/126
    safe_coll = 0
    while (get_cr(total_coll, total_debt) < BUFFER_CR):
        safe_coll += 10

        total_coll = total_coll + safe_coll
    
    print("Extra Coll for Setup", safe_coll)

main()
```
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/23#issuecomment-1680250785) on: **8/16/2023**

The above shows that as demonstrated by the finding the mitigation is insufficient
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/23#issuecomment-1729341653) on: **9/21/2023**

Agree with the finding, we have mitigated by adding a Grace Period (15 minutes) before allowing Recovery Mode to trigger liquidations
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/23#issuecomment-1729349568) on: **9/21/2023**

https://github.com/Badger-Finance/ebtc/blob/1b0a073075484670f24947e0f455031144d3a1e0/packages/contracts/contracts/CdpManagerStorage.sol#L21-L124
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/23#issuecomment-1729809183) on: **9/21/2023**

Still looking into the code, but overall `15 minutes` seems short as the only window for target whale borrower reaction in the whale sniping scenario. In the same time it is the period for which attacker bears the risk, so when this period is short it's both narrow window for possible target reaction and low enough risk for the attacker. 
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/23#issuecomment-1730170238) on: **9/21/2023**

On the other hand, since infinite grace period means no liquidations in RM while `ICR >= MCR`, the max grace period is also due for `setGracePeriod()`.
---
> from: [**dmitriia**](https://github.com/cantinasec/review-badgerdao/issues/23#issuecomment-1730175986) on: **9/21/2023**

> https://github.com/Badger-Finance/ebtc/blob/1b0a073075484670f24947e0f455031144d3a1e0/packages/contracts/contracts/CdpManagerStorage.sol#L21-L124

Looks ok.

Somewhat higher lower boundary and an introduction of the upper one are suggested, i.e. the grace period can be bounded to `[1 hour, 2 days]`, as an example.
