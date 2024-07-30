# TempleGold - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Account abstraction wallet can't transfer cross-chain their TempleGold](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: TempleDAO

### Dates: Jul 4th, 2024 - Jul 11th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-templegold)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Account abstraction wallet can't transfer cross-chain their TempleGold            



## Summary

The `TempleGold::send` assumes that the sender's address (`msg.sender`) will always match the intended recipient address (`_to`) for allowing the TempleGold transfer cross-chain. This assumption fails when users use account abstraction wallet that  has a different address across different chains for the same account. Consequently, a strict comparison between `msg.sender` and `_to` using simple address equality leads to unintended transaction reverts, even when the user is effectively performing a self-operation across chains.

Link to Github: <https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/TempleGold.sol#L281C5-L311C6>

## Vulnerability Details

The vulnerability lies in the logic that enforces `msg.sender != _to` without accounting for the possibility of address variation across chains for the same user. In scenarios where a user's account abstraction wallet has different addresses on different chains, this check incorrectly interprets legitimate self-transfers or controlled operations as unauthorized attempts, leading to transaction failures.

```solidity
function send(
        SendParam calldata _sendParam,
        MessagingFee calldata _fee,
        address _refundAddress
    ) external payable virtual override(IOFT, OFTCore) returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt) {
        if (_sendParam.composeMsg.length > 0) { revert CannotCompose(); }
        /// cast bytes32 to address
        address _to = _sendParam.to.bytes32ToAddress();
        /// @dev user can cross-chain transfer to self
@>        if (msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }

        // @dev Applies the token transfers regarding this send() operation.
        // - amountSentLD is the amount in local decimals that was ACTUALLY sent/debited from the sender.
        // - amountReceivedLD is the amount in local decimals that will be received/credited to the recipient on the remote OFT instance.
        (uint256 amountSentLD, uint256 amountReceivedLD) = _debit(
            msg.sender,
            _sendParam.amountLD,
            _sendParam.minAmountLD,
            _sendParam.dstEid
        );

        // @dev Builds the options and OFT message to quote in the endpoint.
        (bytes memory message, bytes memory options) = _buildMsgAndOptions(_sendParam, amountReceivedLD);

        // @dev Sends the message to the LayerZero endpoint and returns the LayerZero msg receipt.
        msgReceipt = _lzSend(_sendParam.dstEid, message, options, _fee, _refundAddress);
        // @dev Formulate the OFT receipt.
        oftReceipt = OFTReceipt(amountSentLD, amountReceivedLD);

        emit OFTSent(msgReceipt.guid, _sendParam.dstEid, msg.sender, amountSentLD, amountReceivedLD);
    }
```

## Impact

Users with account abstraction wallets have a different address across different chains for the same account, so if someone using an account abstraction wallet can't transfer cross-chain their TempleGold asset.

## Tools Used

Manual review.

## Recommendations

Give the user the option to pass a destination address different from the sender address. Eventually, pass in the warning for account abstraction wallet holders not to pass the same wallet.

    





