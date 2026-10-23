# Signum Improvement Proposals (SIPs)

Signum Improvement Proposals (SIPs) describe standards for the Signum platform, including core protocol specifications, client APIs, and contract standards.

**Before you initiate a pull request**, please read the [SIP-Basics](./SIP/sip-basic.md) process document.
There is a [template SIP here](./SIP/sip-template.md). 

If you use your email address, that address must be the one publicly shown on [your GitHub profile](https://github.com/settings/profile).

## Project Goal

The Signum Improvement Proposals repository exists as a place to share concrete proposals with potential users of the proposal and the Signum community at large.

## SIP status terms

-   **Idea** - An idea that is pre-draft. This is not tracked within the SIP Repository.
-   **Draft** - The first formally tracked stage of an SIP in development. 
-   **Review** - An SIP Author marks an SIP as ready for and requesting Peer Review.
-   **Last Call** - This is the final review window for an SIP before moving to FINAL. An SIP editor will assign Last Call status and set a review end date (`last-call-deadline`), typically 14 days later. If this period results in necessary normative changes it will revert the SIP to Review.
-   **Final** - This SIP represents the final standard. A Final SIP exists in a state of finality and should only be updated to correct errata and add non-normative clarifications.
-   **Stagnant** - Any SIP in Draft or Review if inactive for a period of 6 months or greater is moved to Stagnant. A SIP may be resurrected from this state by Authors or SIP Editors through moving it back to Draft.
-   **Withdrawn** - The SIP Author(s) have withdrawn the proposed SIP. This state has finality and can no longer be resurrected using this SIP number. If the idea is pursued at later date it is considered a new proposal.
-   **Living** - A special status for SIPs that are designed to be continually updated and not reach a state of finality. This includes most notably SIP-Basics.

## SIP Types

SIPs are separated into a number of types:

### Standard Track

#### Core

Improvements requiring a consensus fork 

#### Interface

Includes improvements around client API/RPC specifications and standards, and also certain language-level standards like method names and contract ABIs.

#### SRC

Application-level standards and conventions, including contract standards such as smart contracts  

### Informational 

Describes a Signum design issue, or provides general guidelines or information to the Signum community, but does not propose a new feature. Informational SIPs do not necessarily represent Signum community consensus or a recommendation, so users and implementers are free to ignore Informational SIPs or follow their advice.

## Signum Improvement Proposals

| SIP                          | Type  | Category  | Title                                                          |  Status | Implemented|
|------------------------------| ----------- |-----------|----------------------------------------------------------------| ------- |----|
| [Basic](./SIP/sip-basic.md ) | Informational| -         | SIP Purpose and Guidelines                                     | Living| |
| [1](./SIP/sip-1.md )         | Standard  | Interface | Dynamic BRS Node Capabilities                                  | Final   | yes|
| [2](./SIP/sip-2.md )         | Standard  | Core      | Quadruple Block Size                                           | Final   | yes|
| [3](./SIP/sip-3.md )         | Standard  | Core      | Variable Slot-based Fees                                       | Final   | yes|
| [4](./SIP/sip-4.md )         | Standard  | Core      | Multi-Out Transactions                                         | Final   | yes|
| [5](./SIP/sip-5.md )         | Standard  | Core      | PoC2                                                           | Final   | yes|
| [6](./SIP/sip-6.md )         | Standard  | Interface | Unconfirmed Tx Queue Optimizations                             | Final   | yes|
| [7](./SIP/sip-7.md )         | Standard  | Interface | Differential Unconfirmed Tx Propagation                        | Final   |yes|
| [8](./SIP/sip-8.md )         | Informational| -         | Define Tx and Balance "Dust"                                   | Stagnant   |no |
| [9](./SIP/sip-9.md )         | Informational| -         | To-All Transactions                                            | Stagnant   |no |
| [10](./SIP/sip-10.md )       | Standard| Core      | Anchor Real-World Data in Blockchain                           | Stagnant   |no|
| [11](./SIP/sip-11.md )       | Standard| Core      | Tethered Assets                                                | Stagnant   |no|
| [12](./SIP/sip-12.md )       | Informational| -         | SFS                                                            | Stagnant   |no|
| [13](./SIP/sip-13.md )       | Standard| Interface | Suggested Transaction Fees                                     | Final|yes|
| [14](./SIP/sip-14.md )       | Standard| Interface | Signum Actions through Deeplink QR-codes                       | Final|yes|
| [15](./SIP/sip-15.md )       | Standard| Core      | RWFDS-enabled FEE_QUANT introspection and adjustment           | Draft|no|
| [16](./SIP/sip-16.md )       | Informational| -         | PoC2.X16 - A New Optimized Plot File Format                    | Final|yes|
| [17](./SIP/sip-17.md )       | Standard| Interface | Differential UT Propagation in push & pull                     | Final|yes|
| [18](./SIP/sip-18.md )       | Informational| -         | Cross-Platform Wallet UI                                       | Final|yes|
| [19](./SIP/sip-19.md )       | Standard| Interface | Incoming Multi-Out Tracking                                    | Final|yes|
| [20](./SIP/sip-20.md )       | Standard| Core      | Updated AT fees                                                | Final|yes|
| [21](./SIP/sip-21.md )       | Standard| Core      | Adjustment for Asset-Issuance fee                              | Final|yes|
| [22](./SIP/sip-22.md )       | Standard| SRC       | Deep Link Specification                                        | Final|yes|
| [23](./SIP/sip-23.md )       | Standard| Core      | Enforce slot fees                                              | Final|yes|
| [24](./SIP/sip-24.md )       | Standard| Core      | New deadline algorithm based on a logarithm transformation     | Final|yes|
| [25](./SIP/sip-25.md )       | Standard| Core      | Minor Network Changes                                          | Final|yes|
| [26](./SIP/sip-26.md )       | Standard| Interface | Extended Reed Solomon Address Format                           | Final|yes|
| [27](./SIP/sip-27.md )       | Standard| Core      | Proof of Commitment (PoC+) consensus                           | Final|yes|
| [28](./SIP/sip-28.md )       | Standard| Core      | Updated AT max steps and AT API                                | Final|yes|
| [29](./SIP/sip-29.md )       | Standard| Core      | Mininum mining incentives                                      | Final|yes|
| [30](./SIP/sip-30.md )       | Standard| Core      | Green Contracts                                                | Final|yes|
| [31](./SIP/sip-31.md )       | Standard| Core      | New fee framwork                                               | Final|yes|
| [32](./SIP/sip-32.md )       | Standard| Core      | PoC+ polishing                                                 | Final|yes|
| [33](./SIP/sip-33.md )       | Standard| Core      | Smart Tokens                                                   | Final|yes|
| [34](./SIP/sip-34.md )       | Standard| Core      | Round minimum fee to 2 digits                                  | Final|yes|
| [35](./SIP/sip-35.md )       | Standard| Core      | Cashback                                                       | Final|yes|
| [36](./SIP/sip-36.md )       | Standard| Core      | Distribution of node-related fees (smart contract and similar) | Final|yes|
| [37](./SIP/sip-37.md )       | Standard| Core      | General smart contract upgrade                                 | Final|yes|
| [38](./SIP/sip-38.md )       | Standard| Core      | Map support for smart contracts                                | Final|yes|
| [39](./SIP/sip-39.md )       | Standard| Core      | Smart token support for smart contracts                        | Final|yes|
| [40](./SIP/sip-40.md )       | Standard| SRC       | Signum NFT specifications                                      | Final|yes|
| [41](./SIP/sip-41.md )       | Standard| Core      | Smart Tokens Distribution Changes                              | Final|yes|
| [42](./SIP/sip-42.md )       | Standard| Core      | Subscription behaviour enhancement                             | Final|no|
| [43](./SIP/sip-43.md )       | Standard| Core      | Introduce Smart Token Owner                                    | Final|yes|
| [44](./SIP/sip-44.md )       | Standard| SRC       | Descriptor                                                     | Final|yes|
| [45](./SIP/sip-45.md )       | Standard| SRC       | Staking Contracts                                              | Final|yes|
| [46](./SIP/sip-46.md )       | Standard| SRC      | Secure solo mining setup                                      | Final|yes|
| [47](./SIP/sip-47.md )       | Standard| SRC       | URI Resolution (SNS)                                           | Review|no|
| [48](./SIP/sip-48.md )       | Standard| Core       | Modernize the Alias frameworks                              | Final|yes|
| [49](./SIP/sip-49.md )       | Standard| SRC     | Liquidity Pool Contracts                           | Final|yes|
| [50](./SIP/sip-50.md )       | Standard| SRC     | WebSocket API                           | Review|yes|
| [51](./SIP/sip-51.md )       | Standard| Core     | Lending Commitment                         | Draft|no|
| [53](./SIP/sip-53.md )       | Standard| Core     | Controllable transaction bundling for ATs  | Draft|no|
| [54](./SIP/sip-54.md )       | Standard| Core     | Delete map entries in smart contracts      | Draft|no|
| [55](./SIP/sip-55.md )       | Standard| Core     | Raise smart contract size limits           | Draft|no|


