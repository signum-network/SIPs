---
sip: 39
title: Smart token support for smart contracts
description: Allow to issue, mint, receive, and send assets (tokens)
author:  jjos, frank_the_tank
status: Review
Last Call: 2022-04-24
type: Standard
category: Core
created: 2022-04-17
---
# Abstract
Allow smart contracts to fully interact with Signum smart token [SIP-33](https://github.com/signum-network/SIPs/blob/master/SIP/sip-33.md).

## Specification

The following operations are extended:
 - `GET_AMOUNT_FOR_TX_IN_A = 0x0306`: returns the asset quantity for the asset id in A2 (or SIGNA if A2==0)
 - `SEND_TO_ADDRESS_IN_B = 0x0402`: sends the asset quantity for the asset id in B2 (or SIGNA if B2==0)

The following new operations are included:
 - `B_TO_ASSET_IDS_OF_TX_IN_A = 0x030d`: if A is a valid tx then B set to the asset ids received (terminated by 0)
 - `ISSUE_ASSET = 0x0409`: issue asset with name in A1..2 and dec. places in B1, returns the asset id
 - `MINT_ASSET = 0x040a`: mint B1 quantity of asset ID in B2
 - `DIST_TO_ASSET_HOLDERS = 0x040b`: distribute to asset holders; B1 is the minimum holding of asset ID in B2, A1 the signa amount to distribute, A3 the asset ID to distribute, A4 the asset quantity to distribute
 - `GET_ASSET_HOLDERS_COUNT = 0x040c`: returns the number of tokens holders with more than B1 holdings of asset ID in B2
 - `GET_ASSET_CIRCULATING = 0x040f`: returns the circulating supply for the asset ID in B2

## Backwards Compatibility  
This is a hard forking change, thus breaking compatibility with old fully-validating nodes.  
It should not be deployed without widespread consensus.

## Additional Adjustments 

### Transfer Multiple Tokens in a Single Transaction
In many applications it is interesting to send multiple tokens in a single transaction.
Thus, we are introducing a new transaction type that can send multiple tokens (up to 4 diferent tokens) along with Signa in a single transaction.
This new transaction type is also supported by smart contracts, see `B_TO_ASSET_IDS_OF_TX_IN_A` above.


## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
