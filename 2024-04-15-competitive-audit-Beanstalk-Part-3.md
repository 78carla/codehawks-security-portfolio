# Beanstalk Part 3 - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)


- ## Low Risk Findings
    - ### [L-01. Missing validation for ```totalUsdNeeded``` in ```LibUnripe::getPenalizedUnderlying``` can lead to the ```urBean``` chopping block ](#L-01)
    - ### [L-02. ```LibUnripe::getTotalRecapitalizedPercent``` returns wrong ```recapitalizedPercent``` if ```totalUsdNeeded``` is 0](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Beanstalk

### Dates: May 6th, 2024 - May 20th, 2024

[See more contest details here](https://www.codehawks.com/contests/clvo5kwin00078k6jhhjobn22)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 0
   - Low: 2



		


# Low Risk Findings

## <a id='L-01'></a>L-01. Missing validation for ```totalUsdNeeded``` in ```LibUnripe::getPenalizedUnderlying``` can lead to the ```urBean``` chopping block             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-Beanstalk-3/blob/662d26f12ee219ee92dc485c06e01a4cb5ee8dfb/protocol/contracts/libraries/LibUnripe.sol#L167

https://github.com/Cyfrin/2024-05-Beanstalk-3/blob/662d26f12ee219ee92dc485c06e01a4cb5ee8dfb/protocol/contracts/beanstalk/barn/UnripeFacet.sol#L89-L93

https://github.com/Cyfrin/2024-05-Beanstalk-3/blob/662d26f12ee219ee92dc485c06e01a4cb5ee8dfb/protocol/contracts/libraries/LibChop.sol#L33

## Summary
The ```LibUnripe::getPenalizedUnderlying()``` calculates the penalized amount of Ripe Tokens corresponding to the amount of Unripe Tokens that are chopped, according to the current chop rate into the protocol. When the Beanstalk is fully recapitalized, the ```totalUsdNeeded``` variable becomes ```0```. In the possible scenario where all ```urLP``` is chopped before ```urBeans```, the division by zero error causes the transaction to revert preventing users from chopping ```urBean``` into Ripe Bean.

## Vulnerability Details
```solidity
    function getPenalizedUnderlying(
        address unripeToken,
        uint256 amount,
        uint256 supply
    ) internal view returns (uint256 redeem) {
        require(isUnripe(unripeToken), "not vesting");
        AppStorage storage s = LibAppStorage.diamondStorage();
     
	    uint256 totalUsdNeeded = unripeToken == C.UNRIPE_LP ? LibFertilizer.getTotalRecapDollarsNeeded(supply) 
            : LibFertilizer.getTotalRecapDollarsNeeded();
       
        uint256 underlyingAmount = s.u[unripeToken].balanceOfUnderlying;
@>        redeem = underlyingAmount.mul(s.recapitalized).div(totalUsdNeeded).mul(amount).div(supply);
       
        if(redeem > underlyingAmount) redeem = underlyingAmount;
    }
```

## Impact
When the Beanstalk is fully recapitalized the ```urToken``` holders should be able to redeem the ripe underlying assets at a 1:1 rate. This can't happen in the scenario where all ```urLP``` is chopped before ```urBeans```. 

Considering the scenario where the recapitalization is completed and all ```urLP``` is chopped before ```urBeans```:  the ```totalUsdNeeded``` variable is 0  (the ```LibFertilizer.getTotalRecapDollarsNeeded()``` return 0 because ```C.unripeLP().totalSupply()``` is 0). 
The ```LibUnripe::getPenalizedUnderlying()```will perform a divion by zero error causing the transaction to revert preventing users from chopping ```urBean``` into Ripe Bean. The  ```urBean``` chop into Ripe Bean can't happen and the funds are stuck.

Impact: high because funds are directly at risk.

Likelihood: low because all ```urLP``` should be chopped before ```urBeans```. 

## Tools Used
Manual review

## Recommendations
To handle this scenario, appropriate checks should be added to ensure that in the case of full recapitalization the users can redeem at the new chop rate also in the case where all ```urLP``` is chopped before ```urBeans```.
## <a id='L-02'></a>L-02. ```LibUnripe::getTotalRecapitalizedPercent``` returns wrong ```recapitalizedPercent``` if ```totalUsdNeeded``` is 0            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-05-Beanstalk-3/blob/662d26f12ee219ee92dc485c06e01a4cb5ee8dfb/protocol/contracts/libraries/LibUnripe.sol#L181

## Summary
The ```LibUnripe::getTotalRecapitalizedPercent``` is designed to returns the total percentage that beanstalk has recapitalized (```recapitalizedPercent```). This calculation is based on the ratio of the amount recapitalized (```s.recapitalized```) to the total dollar amount needed to recapitalize Beanstalk (```totalUsdNeeded```). The function contains a conditional statement for handling the scenario where if ```totalUsdNeeded``` is equal to 0 returns 0. 

In the case of ```totalUsdNeeded``` is equal to 0, meaning the recapitalization is completed, the ```recapitalizedPercent``` should be 100% but the function returns 0. 

As indicated in the natspec comment "@dev this is calculated by the ratio of s.recapitalized and the total dollars the barnraise needs to raise returns the same precision as `getRecapPaidPercentAmount` (100% recapitalized = 1e6)". So the ```if``` statement in the function should return ```1e6``` (100% recapitalized).

## Vulnerability Details
```solidity
 function getTotalRecapitalizedPercent() internal view returns (uint256 recapitalizedPercent) {
        AppStorage storage s = LibAppStorage.diamondStorage();
        uint256 totalUsdNeeded = LibFertilizer.getTotalRecapDollarsNeeded();
@>        if(totalUsdNeeded == 0) return 0;
        return s.recapitalized.mul(DECIMALS).div(totalUsdNeeded);
    }
```

## Impact
The ```LibUnripe::getTotalRecapitalizedPercent``` is designed to returns the total percentage that beanstalk has recapitalized (```recapitalizedPercent```). In the case of the total dollar amount needed to recapitalize Beanstalk is 0 (```totalUsdNeeded==0```), the ```recapitalizedPercent``` should be 100% (all recapitalized) and not 0 as returned by the ```if``` statement into the function. This function is called in several get functions (```UnripeFacet::getLockedBeansUnderlyingUnripeBean```, ```UnripeFacet::getPercentPenalty```, etc), returning a wrong result in the case of ```totalUsdNeeded==0```).


## Tools Used
Manual review

## Recommendations
```diff
 function getTotalRecapitalizedPercent() internal view returns (uint256 recapitalizedPercent) {
        AppStorage storage s = LibAppStorage.diamondStorage();
        uint256 totalUsdNeeded = LibFertilizer.getTotalRecapDollarsNeeded();
-        if(totalUsdNeeded == 0) return 0;
+       if(totalUsdNeeded == 0) return 1e6;
        return s.recapitalized.mul(DECIMALS).div(totalUsdNeeded);
    }
```



