# Beanstalk Part 2 - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
  - ### [M-01. `LibWstethEthOracle::getWstethEthPrice` returns wrong `wstETH/ETH` price in some conditions impacting system operations](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Beanstalk

### Dates: Apr 1st, 2024 - Apr 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clu7665bs0001fmt5yahc8tyh)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- Medium: 1

# Medium Risk Findings

## <a id='M-01'></a>M-01. `LibWstethEthOracle::getWstethEthPrice` returns wrong `wstETH/ETH` price in some conditions impacting system operations

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-beanstalk-2/blob/a3d702c2e108cac6ebdf2416906cbca73c83ec99/protocol/contracts/libraries/Oracle/LibWstethEthOracle.sol#L35-L37

https://github.com/Cyfrin/2024-04-beanstalk-2/blob/a3d702c2e108cac6ebdf2416906cbca73c83ec99/protocol/contracts/libraries/Oracle/LibWstethEthOracle.sol#L93-L98

## Summary

The `LibWstethEthOracle::getWstethEthPrice` function is designed in the system to compute the `wstETH/ETH` price . On the top of the `LibWstethEthOracle` contract a detailed NatSpec describes the price computation logic. Reported here for clarity: "It then computes a wstETH:ETH price by taking the minimum of (3) and either the average of (1) and (2) if (1) and (2) are within `MAX_DIFFERENCE` from each other or (1)."

According to the NatSpec, the contract should compute the `wstETH:ETH` price by taking the minimum of the the redemption value or the average of the Chainlink and Uniswap oracle prices if their percent difference is within a specified threshold (`MAX_DIFFERENCE`) or the Chainlink oracle price if the percent difference exceeds this threshold. However, the actual implementation does not handle the scenario where the percent difference exceeds `MAX_DIFFERENCE`. Consequently, users interacts with the system, such as minting fertilizer tokens, using inaccurate price data.

## Vulnerability Details

The `LibWstethEthOracle::getWstethEthPrice` lacks explicit handling for scenarios where the percent difference between the Chainlink and Uniswap oracle prices is greater then `MAX_DIFFERENCE`. This omission leads to situations where the contract does not default to the Chainlink price as intended, affecting the accuracy and reliability of the `wstETH:ETH` price computation.

```solidity
/**
 * @title Wsteth Eth Oracle Library
 * @author brendan
 * @notice Computes the wstETH:ETH price.
 * @dev
 * The oracle reads from 4 data sources:
 * a. wstETH:stETH Redemption Rate: (0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0)
 * b. stETH:ETH Chainlink Oracle: (0x86392dC19c0b719886221c78AB11eb8Cf5c52812)
 * c. wstETH:ETH Uniswap Pool: (0x109830a1AAaD605BbF02a9dFA7B0B92EC2FB7dAa)
 * d. stETH:ETH Redemption: (1:1)
 *
 * It then computes the wstETH:ETH price in 3 ways:
 * 1. wstETH -> ETH via Chainlink: a * b
 * 2. wstETH -> ETH via wstETH:ETH Uniswap Pool: c * 1
 * 3. wstETH -> ETH via stETH redemption: a * d
 *
@> * It then computes a wstETH:ETH price by taking the minimum of (3) and either the average of (1) and (2)
@> * if (1) and (2) are within `MAX_DIFFERENCE` from each other or (1).
**/


    function getWstethEthPrice(uint256 lookback) internal view returns (uint256 wstethEthPrice) {

        uint256 chainlinkPrice = lookback == 0 ?
            LibChainlinkOracle.getPrice(WSTETH_ETH_CHAINLINK_PRICE_AGGREGATOR, LibChainlinkOracle.FOUR_DAY_TIMEOUT) :
            LibChainlinkOracle.getTwap(WSTETH_ETH_CHAINLINK_PRICE_AGGREGATOR, LibChainlinkOracle.FOUR_DAY_TIMEOUT, lookback);

        // Check if the chainlink price is broken or frozen.
        if (chainlinkPrice == 0) return 0;

        uint256 stethPerWsteth = IWsteth(C.WSTETH).stEthPerToken();

        chainlinkPrice = chainlinkPrice.mul(stethPerWsteth).div(CHAINLINK_DENOMINATOR);


        // Uniswap V3 only supports a uint32 lookback.
        if (lookback > type(uint32).max) return 0;
        uint256 uniswapPrice = LibUniswapOracle.getTwap(
            lookback == 0 ? LibUniswapOracle.FIFTEEN_MINUTES :
            uint32(lookback),
            WSTETH_ETH_UNIV3_01_POOL, C.WSTETH, C.WETH, ONE
        );

        // Check if the uniswapPrice oracle fails.
        if (uniswapPrice == 0) return 0;

@>        if (LibOracleHelpers.getPercentDifference(chainlinkPrice, uniswapPrice) < MAX_DIFFERENCE) {
@>           wstethEthPrice = chainlinkPrice.add(uniswapPrice).div(AVERAGE_DENOMINATOR);
@>            if (wstethEthPrice > stethPerWsteth) wstethEthPrice = stethPerWsteth;
@>            wstethEthPrice = wstethEthPrice.div(PRECISION_DENOMINATOR);
        }
    }
}
```

## Impact

The absence of the missing return of the Chainlink oracle price in scenarios of significant price discrepancy between the Chainlink and Uniswap oracles (`LibOracleHelpers.getPercentDifference(chainlinkPrice, uniswapPrice) > MAX_DIFFERENCE`) can lead to a scenario where the contract uses an average price that does not accurately reflect market conditions. The smart contract will operate with an inaccurate `wstETH:ETH` price, impacting operations dependent on this price. This could result in financial losses for users and undermine the integrity of the system.

For example, in the beanstalk system, the `FertilizerFacet::mintFertilizer` function relies on the `LibWstethEthOracle::getWstethEthPrice` to fetch the `wstETH:ETH` price from. This price is crucial for calculating the amount of Fertilizer tokens that can be acquired with the provided `tokenAmountIn`. However, if this function returns an inaccurate price, it would not reflect the actual price of the asset. Consequently, users could continue to mint fertilizer tokens using this inaccurate price data, leading to transactions occurring at incorrect prices.

## Tools Used

Manual review

## Recommendations

Modify the `LibWstethEthOracle::getWstethEthPrice` function to include explicit logic for handling the case where the percent difference between the Chainlink and Uniswap prices is greater then `MAX_DIFFERENCE`.

```diff
if (LibOracleHelpers.getPercentDifference(chainlinkPrice, uniswapPrice) < MAX_DIFFERENCE) {
            wstethEthPrice = chainlinkPrice.add(uniswapPrice).div(AVERAGE_DENOMINATOR);
+      } else {
+        wstethEthPrice = chainlinkPrice;
+    }
            if (wstethEthPrice > stethPerWsteth) wstethEthPrice = stethPerWsteth;
            wstethEthPrice = wstethEthPrice.div(PRECISION_DENOMINATOR);
-        }
```
