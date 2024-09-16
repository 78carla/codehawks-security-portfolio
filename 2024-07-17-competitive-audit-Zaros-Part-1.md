# Zaros Part 1 - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)


- ## Low Risk Findings
    - ### [L-01. Referral Code Abuse in ```TradingAccountBranch::createTradingAccount``` allowing user creating multiple accounts using their own standard referral code](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Zaros

### Dates: Jul 17th, 2024 - Jul 31st, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-zaros)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 0
- Low: 1



    


# Low Risk Findings

## <a id='L-01'></a>L-01. Referral Code Abuse in ```TradingAccountBranch::createTradingAccount``` allowing user creating multiple accounts using their own standard referral code            



## Summary

The `TradingAccountBranch::createTradingAccount` function allows new users to specify a referral code during account creation. This code can either be a standard referral code (representing the Ethereum address of the referrer) or a custom referral code. The system rewards the referrer with a bonus for each new user who signs up using their referral code. However, the current implementation does not adequately prevent a single user from creating multiple accounts using their own standard referral code, potentially  the referral program to unjustly earn bonuses.

Link: <https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/TradingAccountBranch.sol#L229C4-L280C6>

## Vulnerability Details

The vulnerability stems from the lack of effective mechanisms to prevent a user from creating multiple accounts using their own standard referral code. While the contract checks for self-referrals (i.e., the referral code matches the creator's address), it does not prevent a user from creating multiple accounts and using their own referral code across these accounts to earn multiple bonuses.

```solidity
 function createTradingAccount(
        bytes memory referralCode,
        bool isCustomReferralCode
    )
        public
        virtual
        returns (uint128 tradingAccountId)
    {
        // fetch storage slot for global config
        GlobalConfiguration.Data storage globalConfiguration = GlobalConfiguration.load();

        // increment next account id & output
        tradingAccountId = ++globalConfiguration.nextAccountId;

        // get refrence to account nft token
        IAccountNFT tradingAccountToken = IAccountNFT(globalConfiguration.tradingAccountToken);

        // create account record
        TradingAccount.create(tradingAccountId, msg.sender);

        // mint nft token to account owner
        tradingAccountToken.mint(msg.sender, tradingAccountId);

        emit LogCreateTradingAccount(tradingAccountId, msg.sender);

        Referral.Data storage referral = Referral.load(msg.sender);

        if (referralCode.length != 0 && referral.referralCode.length == 0) {
            if (isCustomReferralCode) {
                CustomReferralConfiguration.Data storage customReferral =
                    CustomReferralConfiguration.load(string(referralCode));
                if (customReferral.referrer == address(0)) {
                    revert Errors.InvalidReferralCode();
                }
                referral.referralCode = referralCode;
                referral.isCustomReferralCode = true;
            } else {
                @> address referrer = abi.decode(referralCode, (address));

                @> if (referrer == msg.sender) {
                @>   revert Errors.InvalidReferralCode();
                }

                @> referral.referralCode = referralCode;
                @> referral.isCustomReferralCode = false;
            }

            emit LogReferralSet(msg.sender, referral.getReferrerAddress(), referralCode, isCustomReferralCode);
        }

        return tradingAccountId;
    }
```

## Impact

An attacker could exploit this vulnerability to create numerous accounts using their own referral code, earning the referral bonus for each new account created. Let's imagine a scenario assuming a referral bonus of 10 USD as a fee discount per new user (at the moment the Zaros team hasn't provided detailed info about the referral program). An attacker can create 100 accounts and amass significant rewards through fraudulent means (in our example 1000USDC in fee discount, the attacker can use to trade for free), depleting the referral program's budget and undermining its integrity.&#x20;

## Tools Used

Manual review.

## Recommendations

Considering implementing one or more mechanisms that limit the use of a referral code (for example: each account can only use a unique referral code once or apply rate limiting on account creations associated with referral codes) and/or implement stricter validation logic for referral codes (checks against patterns of abuse such as rapid account creations associated with a single referral code) and/or implement a KYC procedures during the account creation process to verify the identity of users.



