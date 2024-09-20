# Tadle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Missing access control in ```SystemConfig::updateReferrerInfo``` leads to unauthorized updating of the referrer info](#H-01)

- ## Low Risk Findings
    - ### [L-01. Incorrect Validation of ```eachTradeTax``` and ```collateralRate``` in ```PreMarktes::createOffer``` ](#L-01)
    - ### [L-02. Missing access control in ```CapitalPool::approve``` enables any address to approve unlimited amounts](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Tadle

### Dates: Aug 5th, 2024 - Aug 12th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-tadle)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Missing access control in ```SystemConfig::updateReferrerInfo``` leads to unauthorized updating of the referrer info            



## Summary

The `SystemConfig::updateReferrerInfo` allows updating the referrer rate without any access control restrictions. This means any user interacting with the contract can modify the referrer rate, altering the economic incentives within the system.

Link: <https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/SystemConfig.sol#L41C4-L80C6>

## Vulnerability Details

The absence of access control mechanisms, such as the `onlyOwner` modifier on the `updateReferrerInfo` function exposes the contract to unauthorized modifications of critical parameters. This oversight enables any address to call the function and change the referrer rate, bypassing intended governance or administrative controls.

```solidity
 function updateReferrerInfo(
        address _referrer,
        uint256 _referrerRate,
        uint256 _authorityRate
@>    ) external {
        if (_msgSender() == _referrer) {
            revert InvalidReferrer(_referrer);
        }

       ... omitted code

    }
```

## Impact

Malicious actors can exploit this flaw to set referrer rates to their advantage, disrupting the intended economic model of the platform.

## Tools Used

Manual review

## Recommendations

Implement an access control adding the `onlyOwner` modifier.

```diff
- function updateReferrerInfo(address _referrer, uint256 _referrerRate, uint256 _authorityRate) external {
+ function updateReferrerInfo(address _referrer, uint256 _referrerRate, uint256 _authorityRate) external onlyOwner{
```

    


# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect Validation of ```eachTradeTax``` and ```collateralRate``` in ```PreMarktes::createOffer```             



## Summary

The `PreMarktes::createOffer` contains validation checks for the `eachTradeTax` and `collateralRate` parameters. The NatSpec comments indicate that eachTradeTax must be less than 100% and collateralRate must be more than 100%, with a decimal scaler of 10000 (link: <https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L42-L43>).
However, the actual implementation uses `>` and `<` operators, which do not align with the specified requirements.

## Vulnerability Details

The current implementation uses the following conditions (<https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L49C8-L55C10>):

```solidity
function createOffer(CreateOfferParams calldata params) external payable {
        /**
         * @dev points and amount must be greater than 0
         * @dev eachTradeTax must be less than 100%, decimal scaler is 10000
         * @dev collateralRate must be more than 100%, decimal scaler is 10000
         */
        if (params.points == 0x0 || params.amount == 0x0) {
            revert Errors.AmountIsZero();
        }

@>      if (params.eachTradeTax > Constants.EACH_TRADE_TAX_DECIMAL_SCALER) {
            revert InvalidEachTradeTaxRate();
        }

@>      if (params.collateralRate < Constants.COLLATERAL_RATE_DECIMAL_SCALER) {
            revert InvalidCollateralRate();
        }
... omitted code
}
```

These conditions allow `eachTradeTax` to be exactly 100% and `collateralRate` to be exactly 100%, which contradicts the NatSpec comments.

## Impact

This discrepancy can lead to unintended behavior where offers with `eachTradeTax` equal to 100% and `collateralRate` equal to 100% are accepted, causing financial losses within the marketplace.

## Tools Used

Manual review

## Recommendations

Update the validation conditions:

```diff
- if (params.eachTradeTax > Constants.EACH_TRADE_TAX_DECIMAL_SCALER) {
+ if (params.eachTradeTax >= Constants.EACH_TRADE_TAX_DECIMAL_SCALER) {
            revert InvalidEachTradeTaxRate();
        }

- if (params.collateralRate < Constants.COLLATERAL_RATE_DECIMAL_SCALER) {
+ if (params.collateralRate <= Constants.COLLATERAL_RATE_DECIMAL_SCALER) {
            revert InvalidCollateralRate();
        }
```

## <a id='L-02'></a>L-02. Missing access control in ```CapitalPool::approve``` enables any address to approve unlimited amounts            



## Summary

The `CapitalPool::approve` function is intended to allow only the designated token manager to approve tokens (as indicated in the natspec here <https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/interfaces/ICapitalPool.sol#L11>). However, the current implementation lacks proper access control mechanisms, enabling any address to call this function and approve unlimited amounts of tokens without restriction.

Link: <https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/CapitalPool.sol#L24C5-L39C6>

## Vulnerability Details

```solidity
@> function approve(address tokenAddr) external {
        address tokenManager = tadleFactory.relatedContracts(
            RelatedContractLibraries.TOKEN_MANAGER
        );
        (bool success, ) = tokenAddr.call(
            abi.encodeWithSelector(
                APPROVE_SELECTOR,
                tokenManager,
                type(uint256).max
            )
        );

        if (!success) {
            revert ApproveFailed();
        }
    }
```

## Impact

The absence of an access control mechanism, such as a modifier that verifies the caller's identity, means that the approve function does not enforce its intended restriction to the token manager only. This oversight allows unauthorized users to execute the function.

An attacker exploiting this vulnerability can approve tokens from the contract to any address, including their own, without needing authorization from the token manager and  transfer the funds.

Copy and paste this test in PreMarket.t.sol

Run forge test --match-test test\_notOnlyTokenManager

```solidity
function test_notOnlyTokenManager() public {
        vm.prank(user);
        capitalPool.approve(address(mockUSDCToken));
        vm.stopPrank();
    }

Log:
⠊] Compiling...
[⠑] Compiling 1 files with Solc 0.8.25
[⠘] Solc 0.8.25 finished in 2.77s
Compiler run successful!

Ran 1 test for test/PreMarkets.t.sol:PreMarketsTest
[PASS] test_notOnlyTokenManager() (gas: 53764)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.69ms (69.29µs CPU time)

Ran 1 test suite in 283.01ms (3.69ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual review

## Recommendations

Implement a modifier to enforce access control for the approve function, ensuring that only the designated token manager can invoke it.

```diff
+ modifier onlyTokenManager() {
+    address tokenManager = tadleFactory.relatedContracts(
+        RelatedContractLibraries.TOKEN_MANAGER
+    );
+    require(msg.sender == tokenManager, "Caller is not the token manager");
+    _;
+ }

- function approve(address tokenAddr) external {
+ function approve(address tokenAddr) external onlyTokenManager {

... omitted code
}
```



