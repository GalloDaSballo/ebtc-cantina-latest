# [Chainlink `priceFeed.getRoundData(lastRoundId - 1)` could return a "false broken" answer, triggering the primary oracle broken path](https://github.com/cantinasec/review-badgerdao/issues/13)

> state: **closed** opened by: **StErMi** on: **8/10/2023**

**Context:** [PriceFeed.sol#L723-L753](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/PriceFeed.sol#L723-L753)

**Description:**

Useful links:

- [ETH/USD Proxy Price Feed](https://etherscan.deth.net/address/0x5f4ec3df9cbd43714fe2740f5e3616155c5b8419)
- [ETH/USD Price Aggregator used by the Proxy in this phase (phaseID 6)](https://etherscan.deth.net/address/0xE62B71cf983019BFf55bC83B48601ce8419650CC)
- [Official Chainlink Documentation about "Getting Historical Data"](https://docs.chain.link/data-feeds/historical-data)

When the `PriceFeed` contracts execute the `fetchPrice` function, it gathers the current Chainlink response and the previous one by executing 

```solidity
int256 ethBtcAnswer;
        int256 stEthEthAnswer;
        try ETH_BTC_CL_FEED.getRoundData(_currentRoundEthBtcId - 1) returns (
            uint80 roundId,
            int256 answer,
            uint256,
            /* startedAt */
            uint256 timestamp,
            uint80 /* answeredInRound */
        ) {
            ethBtcAnswer = answer;
            prevChainlinkResponse.roundEthBtcId = roundId;
            prevChainlinkResponse.timestampEthBtc = timestamp;
        } catch {
            // If call to Chainlink aggregator reverts, return a zero response with success = false
            return prevChainlinkResponse;
        }

        try STETH_ETH_CL_FEED.getRoundData(_currentRoundStEthEthId - 1) returns (
            uint80 roundId,
            int256 answer,
            uint256,
            /* startedAt */
            uint256 timestamp,
            uint80 /* answeredInRound */
        ) {
            stEthEthAnswer = answer;
            prevChainlinkResponse.roundStEthEthId = roundId;
            prevChainlinkResponse.timestampStEthEth = timestamp;
        } catch {
            // If call to Chainlink aggregator reverts, return a zero response with success = false
            return prevChainlinkResponse;
        }
```

The `roundID` returned by `latestRoundData` and `getRoundData` is not the `roundID` of the Price Aggregator, but is the `roundID` of the Price Feed Proxy that internally will call the aggregator. 

The "`proxyRoundId`" is a composed value that internally store two information:
- `phaseID`: which was the `phaseId` when the query was made. By knowing the phase, we can understand which is the aggregator that has been used
- `aggregatorRoundId`: which is the internal `roundId` from which the price comes from. This makes sense only for the aggregator used in the phase

There could be multiple cases where executing `priceFeed.getRoundData(currentProxyRoundId - 1)` will return an invalid answer
- the proxy has switched to a new `PriceAggregator` and the `phaseId` inside the Proxy has increased
- the `currentProxyRoundId - 1` would revert for underflow or return a wrong answer because `aggregatorRoundId` is equal to the first valid ID

If the `currentProxyRoundId - 1` produces an invalid `roundId`, the Price Feed Proxy will return an invalid answer, but this does not mean that the **real** previous data on the proxy does not exist or that a **real** error has occurred. 
In those cases, the `eBTC Price Feed` will interpret it as an exception and will not trust the Chainlink oracle, falling back to the `FallbackOracle` if available or returning the `lastGoodPrice` that, as we already mentioned in other issues, could be a **stale price**.


**Recommendation:**

Currently, there is not a clear way to easily and correctly retrieve the **safe** `previousRoundId`. 
BadgerDAO should monitor those reverts event to understand if the revert was **real** or because of a misscalculation of the `previousRoundId` 

### Comments

---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/13#issuecomment-1672806781) on: **8/10/2023**

Note: I'm planning to talk to a Chainlink dev about this issue, but the meeting is scheduled in a couple of weeks. 
---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/13#issuecomment-1674750371) on: **8/11/2023**

I'm thinking this is a valid latent risk, but it seems like this will cause DOS for one round if chainlink upgrades

It's worth figuring out if the DOS would be longer than a few hours and most importantly if the system would be able to recover from the DOS after the new set of RoundIDs are sequential
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/13#issuecomment-1674771518) on: **8/11/2023**

@GalloDaSballo I have moved the issue into the previous audit repository given that it was discovered and documented over there.

Issue: https://github.com/spearbit-audits/review-badgerdao/issues/114
GitHub Discussion I have created about `priceFeed.getRoundData`: https://github.com/spearbit-audits/review-badgerdao/discussions/110
