# DittoETH - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## Medium Risk Findings
  - ### [M-01. Division before multiplication result in wrong price / unexpected behaviour](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Ditto

### Dates: Sep 8th, 2023 - Oct 9th, 2023

[See more contest details here](https://codehawks.cyfrin.io/c/2023-09-ditto)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 0
- Medium: 1
- Low: 0

# Medium Risk Findings

## <a id='M-01'></a>M-01. Division before multiplication result in wrong price / unexpected behaviour

### Relevant GitHub Links

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/libraries/LibOracle.sol

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/libraries/LibOrders.sol

## Summary

Multiplication is performed on the result of a division, which can lead to precision errors due to the truncation of decimal points in Solidity.

## Vulnerability Details

contracts/libraries/LibOracle.sol#85:

In LibOracle.baseOracleCircuitBreaker(uint256,uint80,int256,uint256,uint256) - line 85

```
twapPriceInEther = (twapPrice / Constants.DECIMAL_USDC) * 1000000000000000000
```

performs a multiplication on the result of a division.

contracts/libraries/LibOrders.sol#39-57:

In LibOrders.increaseSharesOnMatch(address,STypes.Order,MTypes.Match,uint88) - line 51

```
shares = eth * (timeTillMatch / 86400)
```

also performs a multiplication on the result of a division.

## Impact

These vulnerabilities can lead to incorrect calculations due to the loss of precision, which can have significant implications in a financial context (for example in the case of wrong "twapPriceInEther" calculation). The impact can range from minor discrepancies in value calculations to major financial losses, depending on the specific use case and the values involved.

## Tools Used

Slither

## Recommendations

To mitigate these vulnerabilities, consider rearranging the operations to perform multiplication before division. This can help to maintain precision and avoid potential rounding errors.
For example, can be used:

```
(twapPrice * 1000000000000000000) / Constants.DECIMAL_USDC
```

```
(eth * timeTillMatch) / 86400
```
