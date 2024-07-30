# Sparkn  - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. The organizer can be a winner - potential token steal](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeFox Inc.

### Dates: Aug 21st, 2023 - Aug 29th, 2023

[See more contest details here](https://codehawks.cyfrin.io/c/2023-08-sparkn)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 0



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. The organizer can be a winner - potential token steal            

### Relevant GitHub Links

https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/Distributor.sol

## Summary
The organizer address can be added as a winner.

## Vulnerability Details
The winner address can be anyone (also the organizer address). The organizer has the power to distribute the prize including also the winner addresses. So the organizer can add his/her address as a solo winner and steal all the funds of the contest.

## Impact
In the described vulnerability the steal of the funds is limited to one contest but it could involve a huge amount of money (it depends on how many funds have been collected for the specific contest). Another important aspect to consider is the loss of trust in the protocol. The stealing of money (few or many) leads to a loss of trust with a consequence of loss of users.

## Tools Used
Manual

## Recommendations
Add an if condition for excluding the organizer address in the distribution function.





