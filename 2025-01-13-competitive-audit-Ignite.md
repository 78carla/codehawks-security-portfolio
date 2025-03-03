# Ignite - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)


- ## Low Risk Findings
    - ### [L-01. `StakingContract::updatePriceFeed` can't update `maxPriceAges` for already `acceptedTokens` allowing stale price and causing `registerNode`  to revert](#L-01)
    - ### [L-02. Incorrect role validation in `ValidatorRevarder::unpause` prevents `ROLE_UNPAUSE` holder from unpausing the `ValidatorRevarder` contract](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Benqi

### Dates: Jan 13th, 2025 - Jan 27th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-01-benqi)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 0
- Low: 2



    


# Low Risk Findings

## <a id='L-01'></a>L-01. `StakingContract::updatePriceFeed` can't update `maxPriceAges` for already `acceptedTokens` allowing stale price and causing `registerNode`  to revert            



## Summary

The `StakingContract::updatePriceFeed` function allows updating only the `priceFeeds` for `acceptedTokens`, but does not update the `maxPriceAges`. Updating `maxPriceAges` is crucial if Chainlink changes the heartbeat duration for a price feed. However, this cannot be done via `StakingContract::updatePriceFeed`, leading to a scenario where outdated or unreliable price data might be accepted and incorrectly validated by the contract.

In contrast, the `Ignite::configurePriceFeed` function enables the correct updating of both `priceFeeds` and `maxPriceAges` for existing payment tokens (see link <https://github.com/Cyfrin/2025-01-benqi/blob/f24a5550694e5ff24b059334feeb387b3576ffc9/ignite/src/Ignite.sol#L853>). This ensures that if Chainlink updates the heartbeat, the `Ignite` contract can adjust its `maxPriceAges` accordingly, whereas the `StakingContract` cannot.

The issue arises when the `StakingContract` has a longer `maxPriceAge` than `Ignite`. In such cases, a price update might be acceptable for the `StakingContract` but not for `Ignite`. As a result, when `StakingContract::registerNode` calls `Ignite::registerWithPrevalidatedQiStake`, it will revert, preventing node registration (see <https://github.com/Cyfrin/2025-01-benqi/blob/f24a5550694e5ff24b059334feeb387b3576ffc9/zeeve/contracts/staking.sol#L441>,  <https://github.com/Cyfrin/2025-01-benqi/blob/f24a5550694e5ff24b059334feeb387b3576ffc9/ignite/src/Ignite.sol#L387>).
Operating with a stale price can disrupt the normal node registration process.

## Vulnerability Details

```solidity
  function updatePriceFeed(
        address token,
        address newPriceFeed
    ) external onlyRole(BENQI_ADMIN_ROLE) {
        require(acceptedTokens.contains(token), "Token not accepted"); // Check if the token is accepted
@>        address oldPriceFeed = address(priceFeeds[token]);
@>        _validateAndSetPriceFeed(token, newPriceFeed, maxPriceAges[token]);
        emit PriceFeedUpdated(token, oldPriceFeed, newPriceFeed);
    }
```

## Impact

In the `zeeve` folder add Foundry to the Hardhat project following this procedure: <https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry>

In the `zeeve/test` folder create a file named Audit.t.sol and copy and paste this:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Test, console2} from "forge-std/src/Test.sol";

import {StakingContract} from "../contracts/staking.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

interface IgniteInterface {
    function qiSlashPercentage() external view returns (uint256);
    function priceFeeds(address) external view returns (IPriceFeed);
    function maxPriceAges(address) external view returns (uint256);
    function configurePriceFeed(address token, address priceFeedAddress, uint256 maxPriceAge) external;
}

interface IPriceFeed {
    function latestRoundData()
        external
        view
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound);
}

contract Audit is Test {
    IERC20 public qiToken;

    StakingContract.ContractAddresses public contractAddresses;

    StakingContract public stakingContract;
    ERC1967Proxy public proxy;

    ERC1967Proxy public proxy2;

    //Staking contract variable
    address qiTokenAddress = 0xFFd31a26B7545243F430C0999d4BF11A93408a8C;
    address zeeveWallet = 0x6Ce78374dFf46B660E274d0b10E29890Eeb0167b;
    address igniteSmartContract = 0xF1652dc03Ee76F7b22AFc7FF1cD539Cf20d545D5;
    address joeRouterAddress = 0xd7f655E3376cE2D7A2b08fF01Eb3B1023191A901;

    address AVAX = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

    address qiPriceFeed = 0xFa8A82e96b527F65Bf1c00Bd299C2BE5BABB2FcF;
    address avaxPriceFeed = 0xDa82F8AE3935162a0225c39FC6aad5e22Bc28f6C;
    uint256 initialStakingAmount = 201 ether;
    uint256 initialHostingFee = 1 ether;

    address benqiSuperAdmin = makeAddr("benqiSuperAdmin");
    address benqiAdmin = makeAddr("benqiAdmin");
    address zeeveSuperAdmin = makeAddr("zeeveSuperAdmin");
    address zeeveAdmin = makeAddr("zeeveSuperAdmin");

    bytes32 public constant ZEEVE_SUPER_ADMIN_ROLE = keccak256("ZEEVE_SUPER_ADMIN_ROLE");
    bytes32 public constant ZEEVE_ADMIN_ROLE = keccak256("ZEEVE_ADMIN_ROLE");
    bytes32 public constant BENQI_SUPER_ADMIN_ROLE = keccak256("BENQI_SUPER_ADMIN_ROLE");
    bytes32 public constant BENQI_ADMIN_ROLE = keccak256("BENQI_ADMIN_ROLE");
    bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;

    //Ignite contract variables
    address avaxPriceFeedIgnite = 0x7dF6058dd1069998571497b8E3c0Eb13A8cb6a59;
    address qiPriceFeedIgnite = 0xF3f62E241bC33EF00C731D257F945e8645396Ced;

    address whale = 0xcA7B774A20c1512cDD7461956943C0f3cBcbd087;

    function setUp() public {
        qiToken = IERC20(qiTokenAddress);
        console2.log("qiToken: ", address(qiToken));

        contractAddresses = StakingContract.ContractAddresses(
            qiTokenAddress, avaxPriceFeed, qiPriceFeed, zeeveWallet, igniteSmartContract
        );

        stakingContract = new StakingContract();

        proxy = new ERC1967Proxy(
            address(stakingContract),
            abi.encodeWithSelector(
                StakingContract.initialize.selector,
                contractAddresses,
                benqiSuperAdmin,
                benqiAdmin,
                zeeveSuperAdmin,
                zeeveAdmin,
                initialStakingAmount,
                initialHostingFee,
                joeRouterAddress,
                432000,
                432000
            )
        );

        stakingContract = StakingContract(address(proxy));
        console2.log("stakingContract: ", address(stakingContract));

        vm.startPrank(benqiAdmin);
        stakingContract.setMaxSlippage(100);
        stakingContract.setSlippage(40);
        vm.stopPrank();

        console2.log("-------------------------------");
    }

    function test_cantRegisterERC20NodeWithStalePrice() public {
        //Ignite Contract
        address igniteDefaultAdmin = 0xcA7B774A20c1512cDD7461956943C0f3cBcbd087;
        IgniteInterface igniteContract = IgniteInterface(igniteSmartContract);

        vm.startPrank(igniteDefaultAdmin);

        //We simulate the Staking Contract stale price using another pricefeed address for the Ignite contract
        address newPriceFeed = 0x5c92bD486bB9A04a2b6a0CE1B794218a34c941D5;
        uint256 newmaxPriceAge = 36000;
        igniteContract.configurePriceFeed(address(qiTokenAddress), newPriceFeed, newmaxPriceAge);

        IPriceFeed qiPriceFeedAfter = igniteContract.priceFeeds(address(qiTokenAddress));
        assertEq(address(qiPriceFeedAfter), newPriceFeed);

        //Now we simulate a QI stake and a node rgister
        string memory nodeId = "node-27000";
        bytes memory blsKey = abi.encodePacked(vm.randomBytes(144));
        uint256 stakeIndex = 0;

        //Whale stakes QI for 2 weeks
        uint256 avaxStakeAmount = stakingContract.avaxStakeAmount();
        uint256 hostingFeeAmount = stakingContract.hostingFeeAvax();
        uint256 totalRequiredQi = stakingContract.convertAvaxToQI(avaxStakeAmount + hostingFeeAmount);

        vm.startPrank(whale);
        qiToken.approve(address(stakingContract), type(uint256).max);
        stakingContract.stakeWithERC20(1209600, totalRequiredQi, address(qiToken));

        StakingContract.StakeRecord[] memory stakingRecords = stakingContract.getStakeRecords(whale);
        assertGt(stakingRecords[0].amountStaked, 0);
        assertGt(stakingRecords[0].hostingFeePaid, 0);
        assertEq(stakingRecords[0].duration, 1209600);
        assertEq(stakingRecords[0].tokenType, address(qiToken));
        assertEq(uint256(stakingRecords[0].status), 1); //provisioning

        console2.log("amountStaked: ", stakingRecords[0].amountStaked);
        console2.log("hostingFeePaid: ", stakingRecords[0].hostingFeePaid);

        vm.stopPrank();

        uint256 stakingContractBalanceInitial = qiToken.balanceOf(address(stakingContract));
        uint256 zeeveWalletBalanceInitial = qiToken.balanceOf(zeeveWallet);
        console2.log("stakingContractBalanceInitial: ", stakingContractBalanceInitial);
        console2.log("zeeveWalletBalanceInitial: ", zeeveWalletBalanceInitial);

        //Zeeve admin try to register the node but it reverts
        vm.startPrank(zeeveAdmin);
        vm.expectRevert();
        stakingContract.registerNode(whale, nodeId, blsKey, stakeIndex);

        StakingContract.StakeRecord[] memory records = stakingContract.getStakeRecords(whale);
        assertGt(records[0].amountStaked, 0);
        assertGt(records[0].hostingFeePaid, 0);
        assertEq(records[0].duration, 1209600);
        assertEq(records[0].tokenType, address(qiToken));
        assertEq(uint256(records[0].status), 1); //already in provisioning

        uint256 stakingContractBalanceFinal = qiToken.balanceOf(address(stakingContract));
        uint256 zeeveWalletBalanceFinal = qiToken.balanceOf(zeeveWallet);
        console2.log("stakingContractBalanceFinal: ", stakingContractBalanceFinal);
        console2.log("zeeveWalletBalanceFinal: ", zeeveWalletBalanceFinal);

        assertEq(zeeveWalletBalanceInitial, zeeveWalletBalanceFinal);
        assertEq(stakingContractBalanceFinal, stakingContractBalanceInitial);

        vm.stopPrank();
    }

    function test_canUpdateMaxPricesAgesOnlyOnIgniteContract() public {
        //Ignite Contract
        address igniteDefaultAdmin = 0xcA7B774A20c1512cDD7461956943C0f3cBcbd087;
        IgniteInterface igniteContract = IgniteInterface(igniteSmartContract);

        vm.startPrank(igniteDefaultAdmin);

        IPriceFeed qiPriceFeedBefore = igniteContract.priceFeeds(address(qiTokenAddress));
        uint256 qiMaxPriceAgeBefore = igniteContract.maxPriceAges(address(qiTokenAddress));
        (, int256 qiIgnitepriceBefore,, uint256 updatedAtIgniteBefore,) = qiPriceFeedBefore.latestRoundData();
        console2.log("Ignite contract: ");
        console2.log("qiPriceFeedBefore: ", address(qiPriceFeedBefore));
        console2.log("qiMaxPriceAgeBefore: ", qiMaxPriceAgeBefore);
        console2.log("qiIgnitepriceBefore", qiIgnitepriceBefore);
        console2.log("---------------------------------");

        address newPriceFeed = 0x7dF6058dd1069998571497b8E3c0Eb13A8cb6a59;
        uint256 newmaxPriceAge = 36000; //10 hours - 36000
        igniteContract.configurePriceFeed(address(qiTokenAddress), newPriceFeed, newmaxPriceAge);

        IPriceFeed qiPriceFeedAfter = igniteContract.priceFeeds(address(qiTokenAddress));
        uint256 qiMaxPriceAgeAfter = igniteContract.maxPriceAges(address(qiTokenAddress));
        (, int256 qiIgnitepriceAfter,, uint256 updatedAtIgniteAfter,) = qiPriceFeedAfter.latestRoundData();
        console2.log("qiPriceFeedAfter: ", address(qiPriceFeedAfter));
        console2.log("qiMaxPriceAgeAfter: ", qiMaxPriceAgeAfter);
        console2.log("qiIgnitepriceAfter", qiIgnitepriceAfter);
        console2.log("-------------------------------");
        console2.log("-------------------------------");

        assertEq(address(qiPriceFeedAfter), newPriceFeed);
        assertEq(qiMaxPriceAgeAfter, newmaxPriceAge);

        // //Staking Contract
        AggregatorV3Interface qiPriceFeedSCBefore = stakingContract.priceFeeds(address(qiToken));
        uint256 qiMaxPriceAgeSCBefore = stakingContract.maxPriceAges(address(qiToken));
        (, int256 qiStakingpriceBefore,, uint256 updatedAtStakingBefore,) = qiPriceFeedSCBefore.latestRoundData();
        console2.log("Staking contract: ");
        console2.log("qiPriceFeedSCBefore: ", address(qiPriceFeedSCBefore));
        console2.log("qiMaxPriceAgeSCBefore: ", qiMaxPriceAgeSCBefore);
        console2.log("qiStakingpriceBefore", qiStakingpriceBefore);
        console2.log("-------------------------------");

        vm.startPrank(benqiAdmin);
        address newPriceFeedSC = 0x7dF6058dd1069998571497b8E3c0Eb13A8cb6a59;
        stakingContract.updatePriceFeed(address(qiToken), newPriceFeedSC);

        AggregatorV3Interface qiPriceFeedSCAfter = stakingContract.priceFeeds(address(qiToken));
        uint256 qiMaxPriceAgeSCAfter = stakingContract.maxPriceAges(address(qiToken));
        (, int256 qiStakingpriceAfter,, uint256 updatedAtStakingAfter,) = qiPriceFeedSCAfter.latestRoundData();
        console2.log("qiPriceFeedSCAfter: ", address(qiPriceFeedSCAfter));
        console2.log("qiMaxPriceAgeSCAfter: ", qiMaxPriceAgeSCAfter);
        console2.log("qiStakingpriceAfter", qiStakingpriceAfter);
        console2.log("-------------------------------");

        assertEq(address(qiPriceFeedSCAfter), newPriceFeedSC);
        assertEq(qiMaxPriceAgeSCAfter, qiMaxPriceAgeSCBefore);

        vm.stopPrank();
    }
}
```

Run the command: forge test --match-test test\_cantRegisterERC20NodeWithStalePrice --fork-url <https://avax-fuji.g.alchemy.com/v2/yourAPIKey> -vv --via-ir

Note: edit `yourAPIKey` with your Alchemy (or other provider) APIKey.

```solidity
Logs:
Ran 1 test for test/Audit.t.sol:Audit
[PASS] test_cantRegisterERC20NodeWithStalePrice() (gas: 444642)
Logs:
  qiToken:  0xFFd31a26B7545243F430C0999d4BF11A93408a8C
  stakingContract:  0x2e234DAe75C793f67A35089C9d99245E1C58470b
  -------------------------------
  amountStaked:  1206040828894402571
  hostingFeePaid:  6000203128827873
  stakingContractBalanceInitial:  1212041032023230444
  zeeveWalletBalanceInitial:  67924312038644785583
  stakingContractBalanceFinal:  1212041032023230444
  zeeveWalletBalanceFinal:  67924312038644785583

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.70s (9.05s CPU time)

Ran 1 test suite in 16.04s (14.70s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The test shows that if the Staking Contact uses a stale price the Ignite contract will revert and can't register the node.

Then run the command:
forge test --match-test test\_canUpdateMaxPricesAgesOnlyOnIgniteContract --fork-url <https://avax-fuji.g.alchemy.com/v2/yourAPIKey> -vv --via-ir

```solidity
Logs:
Ran 1 test for test/Audit.t.sol:Audit
[PASS] test_canUpdateMaxPricesAgesOnlyOnIgniteContract() (gas: 111606)
Logs:
  qiToken:  0xFFd31a26B7545243F430C0999d4BF11A93408a8C
  stakingContract:  0x2e234DAe75C793f67A35089C9d99245E1C58470b
  -------------------------------
  Ignite contract: 
  qiPriceFeedBefore:  0xFa8A82e96b527F65Bf1c00Bd299C2BE5BABB2FcF
  qiMaxPriceAgeBefore:  432000
  qiIgnitepriceBefore 3416551000000
  ---------------------------------
  qiPriceFeedAfter:  0x7dF6058dd1069998571497b8E3c0Eb13A8cb6a59
  qiMaxPriceAgeAfter:  36000
  qiIgnitepriceAfter 2735000000
  -------------------------------
  -------------------------------
  Staking contract: 
  qiPriceFeedSCBefore:  0xFa8A82e96b527F65Bf1c00Bd299C2BE5BABB2FcF
  qiMaxPriceAgeSCBefore:  432000
  qiStakingpriceBefore 3416551000000
  -------------------------------
  qiPriceFeedSCAfter:  0x7dF6058dd1069998571497b8E3c0Eb13A8cb6a59
  qiMaxPriceAgeSCAfter:  432000
  qiStakingpriceAfter 2735000000
  -------------------------------

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.62s (2.95s CPU time)
```

The test shows that it is possible to update the `maxPriceAge`  in the `Ignite` contract but it isn't possible in the `StakingContract`.

## Tools Used

Manual review.

## Recommendations

Add the `newMaxPriceAge` in the `updatePriceFeed` allowing the updating.

```Solidity
function updatePriceFeed(
        address token,
        address newPriceFeed
+     uint256 newMaxPriceAge
    ) external onlyRole(BENQI_ADMIN_ROLE) {
        require(acceptedTokens.contains(token), "Token not accepted"); 
        address oldPriceFeed = address(priceFeeds[token]);
  
+       uint oldPriceMaxAge = maxPriceAges[token];
+       maxPriceAges[token] = newMaxPriceAge;
  
        _validateAndSetPriceFeed(token, newPriceFeed, maxPriceAges[token]);
  
-      emit PriceFeedUpdated(token, oldPriceFeed, newPriceFeed);
+      emit PriceFeedUpdated(token, oldPriceFeed, newPriceFeed, oldMaxPriceAge, newMaxPriceAge);
    }
```

## <a id='L-02'></a>L-02. Incorrect role validation in `ValidatorRevarder::unpause` prevents `ROLE_UNPAUSE` holder from unpausing the `ValidatorRevarder` contract            



## Summary

The `ValidatorRevarder` contract implements two distinct roles: `ROLE_PAUSE` and `ROLE_UNPAUSE`, suggesting an intended separation of duties for pausing and unpausing operations. However, the `unpause()` function incorrectly checks for `ROLE_PAUSE` instead of `ROLE_UNPAUSE`, making the `ROLE_UNPAUSE` effectively useless. Accounts granted `ROLE_UNPAUSE` cannot unpause the contract despite having the designated role for this purpose.

## Vulnerability Details

The improper role check in `unpause()` means that accounts with only `ROLE_UNPAUSE` cannot unpause the contract and only accounts with `ROLE_PAUSE` can unpause the contract.

```Solidity
    bytes32 public constant ROLE_PAUSE = keccak256("ROLE_PAUSE");
 @> bytes32 public constant ROLE_UNPAUSE = keccak256("ROLE_UNPAUSE");

    function unpause() external {
@>      if (!hasRole(ROLE_PAUSE, msg.sender)) {
            revert Unauthorized();
        }

        _unpause();
    }

```

## Impact

Accounts specifically granted `ROLE_UNPAUSE` cannot perform the unpause of the contract. Anyone with the `ROLE_PAUSE` permission can call `unpause()`, even though they shouldn't have the ability to unpause the contract. This discrepancy can lead to unauthorized unpausing of the `ValidatorRewarder` contract.

In the `ignite` folder add Foundry to the project following this procedure: <https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry>

Create a file named ValidatorRewarder.t.sol and copy and paste this:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Test, console2} from "forge-std/src/Test.sol";

import {ValidatorRewarder} from "../src/ValidatorRewarder.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import "@openzeppelin/contracts-upgradeable/token/ERC20/IERC20Upgradeable.sol";

contract Validator is Test {
    ValidatorRewarder public validatorRewarderContract;
    ERC1967Proxy public proxy;

    address public qiTokenAddress = 0xFFd31a26B7545243F430C0999d4BF11A93408a8C;
    address public igniteSmartContract = 0xF1652dc03Ee76F7b22AFc7FF1cD539Cf20d545D5;

    address public AVAX = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

    address public withdrawAdmin = makeAddr("withdrawAdmin");
    address public pauseAdmin = makeAddr("pauseAdmin");
    address public unpauseAdmin = makeAddr("unpauseAdmin");

    bytes32 public constant ROLE_WITHDRAW = keccak256("ROLE_WITHDRAW");
    bytes32 public constant ROLE_PAUSE = keccak256("ROLE_PAUSE");
    bytes32 public constant ROLE_UNPAUSE = keccak256("ROLE_UNPAUSE");
    bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;

    address public whale = 0xcA7B774A20c1512cDD7461956943C0f3cBcbd087;
    address public igniteDefaultAdmin;

    function setUp() public {
        validatorRewarderContract = new ValidatorRewarder();

        proxy = new ERC1967Proxy(
            address(validatorRewarderContract),
            abi.encodeWithSelector(
                ValidatorRewarder.initialize.selector, qiTokenAddress, igniteSmartContract, 2500, address(this)
            )
        );

        validatorRewarderContract = ValidatorRewarder(address(proxy));
        console2.log("validatorRewarder: ", address(validatorRewarderContract));

        validatorRewarderContract.grantRole(ROLE_PAUSE, pauseAdmin);
        validatorRewarderContract.grantRole(ROLE_UNPAUSE, unpauseAdmin);
    }

    function test_unpauseRoleCantUnpauseTheValidatorRewarderContract() public {
        //ROLE_PAUSE pauses the contract
        vm.startPrank(pauseAdmin);
        validatorRewarderContract.pause();
        vm.stopPrank();

        assertEq(validatorRewarderContract.paused(), true);

        //ROLE_UNPAUSE tries to unpause the contract but can't
        vm.startPrank(unpauseAdmin);
        vm.expectRevert();
        validatorRewarderContract.unpause();
        vm.stopPrank();

        //ROLE_PAUSE can unpause the contract but should not
        vm.startPrank(pauseAdmin);
        validatorRewarderContract.unpause();
        vm.stopPrank();
        assertEq(validatorRewarderContract.paused(), false);
    }
  }
```

Run the command: forge test --match-test test\_unpauseRoleCantUnpauseTheValidatorRewarderContract --fork-url <https://avax-fuji.g.alchemy.com/v2/yourAPIKey> -vv --via-ir

Note: edit the `yourAPIKey` with your Alchemy (or other provider) before running the command.

```solidity
Logs:
Ran 1 test for tests/Validator.t.sol:Validator
[PASS] test_unpauseRoleCantUnpauseTheValidatorRewarderContract() (gas: 40788)
Logs:
  validatorRewarder:  0x2e234DAe75C793f67A35089C9d99245E1C58470b

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.48s (4.53ms CPU time)

Ran 1 test suite in 5.92s (4.48s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The test shows that the `unpauseAdmin` granted with the role `ROLE_UNPAUSE` can't unpause the contract but only the \`\`pauseAdmin`with the`ROLE\_PAUSE\` can.

## Tools Used

Manual review

## Recommendations

Modify the `unpause()` function to check for the correct role:

```diff
function unpause() external {
-    if (!hasRole(ROLE_PAUSE, msg.sender)) {
+    if (!hasRole(ROLE_UNPAUSE, msg.sender)) {
        revert Unauthorized();
    }
    _unpause();
}
```

Now add this test in the ValidatorRewarder.t.sol file

```solidity
function test_unpauseRoleCanUnpauseTheValidatorRewarderContract() public {
        //ROLE_PAUSE pauses the contract
        vm.startPrank(pauseAdmin);
        validatorRewarderContract.pause();
        vm.stopPrank();

        assertEq(validatorRewarderContract.paused(), true);

        //ROLE_UNPAUSE can correctly unpause the contract
        vm.startPrank(unpauseAdmin);
        validatorRewarderContract.unpause();
        vm.stopPrank();
        assertEq(validatorRewarderContract.paused(), false);
    }
```

Run the command: forge test --match-test test\_unpauseRoleCanUnpauseTheValidatorRewarderContract --fork-url <https://avax-fuji.g.alchemy.com/v2/yourAPIKey> -vv --via-ir

```solidity
Logs:
Ran 1 test for tests/Validator.t.sol:Validator
[PASS] test_unpauseRoleCanUnpauseTheValidatorRewarderContract() (gas: 38638)
Logs:
  validatorRewarder:  0x2e234DAe75C793f67A35089C9d99245E1C58470b

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.36s (2.40ms CPU time)
```

Now all works as should be.



