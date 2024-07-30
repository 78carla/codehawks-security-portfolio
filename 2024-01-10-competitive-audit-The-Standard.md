# The Standard - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings

  - ### [H-01. Access Control `distributeAssets::LiquidationPool` can be called by anyone](#H-01)

- ## Low Risk Findings
  - ### [L-01. Precision loss when calculating `LiquidationPool::distributeAssets`](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2023-12-the-standard)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 0
- Low: 1

# High Risk Findings

## <a id='H-01'></a>H-01. Access Control `distributeAssets::LiquidationPool` can be called by anyone

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L205-L240

## Summary

`distributeAssets::LiquidationPool` can be called by anyone and isn't restricted to `onlyOwner`. The rewards in terms of fee and assets portion are managed by the `LiquidationPoolManager` calling the distributeFees() and the runLiquidation(). A malicious actor could call this function with a large number of assets and a low collateral rate acquiring more rewards.

## Vulnerability Details

```javascript

function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
        consolidatePendingStakes();
        (,int256 priceEurUsd,,,) = Chainlink.AggregatorV3Interface(eurUsd).latestRoundData();
        uint256 stakeTotal = getStakeTotal();
        uint256 burnEuros;
        uint256 nativePurchased;
        for (uint256 j = 0; j < holders.length; j++) {
            Position memory _position = positions[holders[j]];
            uint256 _positionStake = stake(_position);
            if (_positionStake > 0) {
                for (uint256 i = 0; i < _assets.length; i++) {
                    ILiquidationPoolManager.Asset memory asset = _assets[i];
                    if (asset.amount > 0) {
                        (,int256 assetPriceUsd,,,) = Chainlink.AggregatorV3Interface(asset.token.clAddr).latestRoundData();
                        uint256 _portion = asset.amount * _positionStake / stakeTotal;
                        uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
                            * _hundredPC / _collateralRate;
                        if (costInEuros > _position.EUROs) {
                            _portion = _portion * _position.EUROs / costInEuros;
                            costInEuros = _position.EUROs;
                        }
                        _position.EUROs -= costInEuros;
                        rewards[abi.encodePacked(_position.holder, asset.token.symbol)] += _portion;
                        burnEuros += costInEuros;
                        if (asset.token.addr == address(0)) {
                            nativePurchased += _portion;
                        } else {
                            IERC20(asset.token.addr).safeTransferFrom(manager, address(this), _portion);
                        }
                    }
                }
            }
            positions[holders[j]] = _position;
        }
        if (burnEuros > 0) IEUROs(EUROs).burn(address(this), burnEuros);
        returnUnpurchasedNative(_assets, nativePurchased);
    }
```

## Impact

A malicious actor could call this function with a large number of assets and a low collateral rate, effectively giving themselves a disproportionately large share of the assets. Or they could call this function with a high collateral rate, causing the cost of the assets to be inflated and potentially leading to financial loss for the holders.

## Tools Used

Manual review

## Recommendations

Add the `onlyManager` modifier.

```diff
- function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
+ function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable onlyManager{

        consolidatePendingStakes();
        (,int256 priceEurUsd,,,) = Chainlink.AggregatorV3Interface(eurUsd).latestRoundData();
        uint256 stakeTotal = getStakeTotal();
        uint256 burnEuros;
        uint256 nativePurchased;
        ...
        returnUnpurchasedNative(_assets, nativePurchased);
    }
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Precision loss when calculating `LiquidationPool::distributeAssets`

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L220

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L219

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L223

## Summary

The calculation of `_portion` and `costInEuros` in the `LiquidationPool.distributeAssets` suffers from a rounding down issue, resulting in a precision loss that can be improved. This can lead to a wrong portion of distributed assets to the holders.

## Vulnerability Details

Division before multiplication can lead to rounding down issue since Solidity has no fixed-point numbers. Consider the calculation of the:

- `_portion` in the `LiquidationPool.distributeAssets`, the function does the division (by `uint256(priceEurUsd)` before the multiplication (`_hundredPC`). Hence, the computed result can suffer from the rounding down issue, resulting in a small precision loss.
  https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L220C11-L220C11
- `costInEuros` in the `LiquidationPool.distributeAssets`, the function does the division (by `stakeTotal`) before the multiplication (`_position.EUROs `). Hence, the computed result can suffer from the rounding down issue, resulting in a small precision loss.
  https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L219
  https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L223

```javascript
function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
        consolidatePendingStakes();
        (,int256 priceEurUsd,,,) = Chainlink.AggregatorV3Interface(eurUsd).latestRoundData();
        uint256 stakeTotal = getStakeTotal();
        uint256 burnEuros;
        uint256 nativePurchased;
        for (uint256 j = 0; j < holders.length; j++) {
            Position memory _position = positions[holders[j]];
            uint256 _positionStake = stake(_position);
            if (_positionStake > 0) {
                for (uint256 i = 0; i < _assets.length; i++) {
                    ILiquidationPoolManager.Asset memory asset = _assets[i];
                    if (asset.amount > 0) {
                        (,int256 assetPriceUsd,,,) = Chainlink.AggregatorV3Interface(asset.token.clAddr).latestRoundData();
@>                        uint256 _portion = asset.amount * _positionStake / stakeTotal;
@>                        uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
                            * _hundredPC / _collateralRate;
                        if (costInEuros > _position.EUROs) {
@>                            _portion = _portion * _position.EUROs / costInEuros;
                            costInEuros = _position.EUROs;
                        }
                        _position.EUROs -= costInEuros;
                        rewards[abi.encodePacked(_position.holder, asset.token.symbol)] += _portion;
                        burnEuros += costInEuros;
                        if (asset.token.addr == address(0)) {
                            nativePurchased += _portion;
                        } else {
                            IERC20(asset.token.addr).safeTransferFrom(manager, address(this), _portion);
                        }
                    }
                }
            }
            positions[holders[j]] = _position;
        }
        if (burnEuros > 0) IEUROs(EUROs).burn(address(this), burnEuros);
        returnUnpurchasedNative(_assets, nativePurchased);
    }
```

## Impact

The outcome derived from the `LiquidationPool.distributeAssets` computation may be susceptible to rounding down issues. This issue carries a medium impact, given that two instances of precision loss occur within the same function. Additionally, the denominators are variable rather than constant, compromising the accuracy of the calculated asset. Nonetheless, there is a potential avenue for enhancing the calculation to mitigate precision loss, as outlined in the Recommendations section.

## Tools Used

Slither - static analysis framework.

## Recommendations

```javascript
function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
        ...
                    ILiquidationPoolManager.Asset memory asset = _assets[i];
                    if (asset.amount > 0) {
                        (,int256 assetPriceUsd,,,) = Chainlink.AggregatorV3Interface(asset.token.clAddr).latestRoundData();
                         uint256 _portion = asset.amount * _positionStake / stakeTotal;
-                        uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
                            * _hundredPC / _collateralRate;
+                    uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) * _hundredPC
                      / (uint256(priceEurUsd) * _collateralRate);
                        if (costInEuros > _position.EUROs) {
-                            _portion = _portion * _position.EUROs / costInEuros;
+                            _portion = (asset.amount * _positionStake * _position.EUROs) / stakeTotal / costInEuros;
                            costInEuros = _position.EUROs;
                        }
                        _position.EUROs -= costInEuros;
                        rewards[abi.encodePacked(_position.holder, asset.token.symbol)] += _portion;
                        burnEuros += costInEuros;
                        if (asset.token.addr == address(0)) {
                            nativePurchased += _portion;
                        } else {
                            IERC20(asset.token.addr).safeTransferFrom(manager, address(this), _portion);
                        }
                    }
                }
            }
            positions[holders[j]] = _position;
        }
        if (burnEuros > 0) IEUROs(EUROs).burn(address(this), burnEuros);
        returnUnpurchasedNative(_assets, nativePurchased);
    }
}

```
