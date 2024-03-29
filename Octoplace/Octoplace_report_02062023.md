# Octoplace Security Audit Report



## INTRODUCTION

### ABOUT ODDSEQUENCE

OddSequence is a blockchain security team with broad experience in researching DeFi projects and performing security audits.

### DISCLAMER

The audit does not provide any guarantees or assurances regarding the code's usefulness, code safety, suitability of the business model, investment advice, endorsement of the platform or its products, regulatory framework for the business model, or any other statements regarding the contracts' fitness for purpose or their bug-free status. The audit documentation is intended for discussion purposes only. The information presented in this report is both confidential and privileged. By reading this report, you agree to maintain its confidentiality and not to copy, disclose, or distribute it without the Client's agreement. If you are not the intended recipient(s) of this document, please be aware that any disclosure, copying, or distribution of its contents is strictly prohibited.


### METHODOLOGY

A team of auditors is engaged in the audit process. The goal is to understand the projects's architercture and logic, detect typical vulnerabilities, detect unexpected behavior, and find other security issues. The security engineers conduct independent evaluations of the provided source code, following the methodology outlined below:

#### 1. Architecture assessment:

* Documentation review, code review, identifyng structure and purpose of code elements
* Research and study of the project architecture based on the source code

#### 2. Identifying unexpected behavior:

* Finding exploit scenarios and other issues with impact
* Proof of Concept (PoC) exploit writing

#### 3. Consolidation of findings in the report:

* Internal discussion of issues found, identifying issue classification based on their severity and likelyhood
* Selecting findings for the audit report


#### Finding Severity

Findings are catergorized in four levels: Critical, High, Medium, Low. This classification in the report is based on issue severity and likelihood.


## PROJECT AND SCOPE

### Overview

Octoplace smart contracts allow exchanging NFTs 1-to-1 based on listings and offers. One party can list an NFT for sale, other parties can offer their NFTs to buy the listed NFT. If the seller accepts an offer Octoplace makes an exchange and parties receive desired NFT.


### Project Summary

_ | _
--- | ---
Client             | Octoplace
Scope              | NFT exchange
Time          | 29.05.2023 - 02.06.2023
Auditors | 2

### Scope links

Commit | Note
--- | ---
c374bec00a1c7cef4adbd1d1e9d323ceed44a91b  | Start of the audit commit

All smart contract files and tests:
https://github.com/OCTOplace/octoplace-contracts/commit/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b



## FINDINGS

### Summary

Finding | Severity | Status | Description 
--- | --- | --- | ---
C1 | Critical | New | Anyone can spam the marketplace with fake offers and delete real bids
H1 | High | New | Extra native tokens paid for listing can be stuck on the contract
M1 | Medium | New | Potentially disabled getters in SwapData
M2 | Medium | New | Approval checks for NFTs are easy to bypass
L1 | Low | New | Superadmin transition does not follow two-step procedure

### Detailed description

### [C1] CRITICAL. Anyone can spam the marketplace with fake offers and delete real bids

`Octoplace.createMarketItemOfferTNT20()` function allows anyone to make offers for some listed NFT.
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/Octoplace.sol#L357


But offers do not require real payment with tokens - the function just checks that the allowance given to the marketplace is above the previous highest offer.
In this case, the highest bid is updated and the caller gets stored as an active bidder / winner
```
    idToMarketItem[itemId].highestOffer = offerPrice;
    idToMarketItem[itemId].bidder = payable(msg.sender);
```
In addition, previous successful bidder is deleted.

Then, when seller accepts offer (`acceptMarketItemOfferTNT20()`), the function checks that the active bidder still has approval given to the marketplace. If it is not, the contract just nulifies the storage for the winning bidder.
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/Octoplace.sol#L376-L391

This is the scenario anyone can exploit to break offers with no cost.
Anyone can:
1) Give high approval
2) Call createMarketItemOfferTNT20() with the new highest bid
3) Revoke approval

It can be made in one simple transaction. And all previous bidders will have to make new offers and seller will not be able to accept previous real bids.

#### Recommendation

We recommend taking real tokens from bidders and return them if a new highest bid is made.

---



### [H1] HIGH. Extra native tokens paid for listing can be stuck on the contract.

SwapNFT contract has the function createListing() which accepts some Native token as payment for listing.
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/SwapNFT.sol#L50

Then it checks that this amount is above txCharge.
```
require(msg.value >= txCharge, "Insufficient tfuel sent for txCharge");
```

Then it stores that a user paid txCharge.
```
listing.transactionCharge = txCharge;
```

When the deal is executed further by calling acceptOffer() the contract transfers native tokens stored in `listing.transactionCharge`.
So in case of `msg.value` sent above `txCharge`these native token can be stuck with no way to withdraw by anyone.
This is also probable because the admin can change the `txCharge`and other contracts/users can operate using the older txCharge, one which is higher, for instance.
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/SwapNFT.sol#L249C1-L255

#### Recommendation

The easiest solution is if `listing.transactionCharge`stores `msg.value`.

---


### [M1] MEDIUM. Potentially disabled getters in SwapData.

SwapData contract has getters that iterate through mappings `listings`, `swapOffers`, `trades`:
- `readAllListings()`
- `readAllOffers()`
- `readAllTrades()`
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/DataFlat.sol#L405-L443

They return the whole length of associated structs.
It is possible that the length turns so large, that it will be impossible to return due to the max gas limit. 
These functions are getters and are not used in the other write functions. We evaluate the severity of the finding high because anyone can spam `offers` with little cost and break the functions. We calculated that 3M gas limit is not enough to return even 180 offers.

#### Recommendation

If the functions are used for off-chain data extraction we recommend introducing start and end numbers to limit the length of return. For instance, one call to get structs from 0 to 50, a new call to get structs from 51 to 100, and so on.

---

### [M2] MEDIUM. Approval checks for NFTs are easy to bypass

SwapData in two functions for parties checks that a token owner gave an approval to SwapData contract.
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/SwapNFT.sol#L88-L95
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/SwapNFT.sol#L55-L63

First, it is easy to bypass this check: to give approval, call SwapData contract, to revoke approval. But still `acceptOffer()` would later require approvals from both parties to execute `transferFrom` and revert if something is wrong.
Second, the worst case is if an NFT contract does not revert on `transferFrom()` when there is no approval. It would parties to scam each other.

#### Recommendation

One of the options is to check in `acceptOffer()` that receivers of NFTs  turn owners of their new NFTs, and revert if someone is not.

---


### [L1] LOW. Superadmin transition does not follow two-step procedure

Octoplace has superadmin role, which can be transferred via `setSuperAdmin()`
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/Octoplace.sol#L187-L189

It changes the role to the inputted address without checks. 

#### Recommendation

The good practice is requiring non-zero address and introducing a new function for a new superadmin to accept the role.

---

### [L2] LOW. Typo breaking compilation

Octoplace contract cannot compile due to syntax error. ";" symbol is required at the end of the line.
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/Octoplace.sol#L539-L542
The same thing for the flat version.
- https://github.com/OCTOplace/octoplace-contracts/blob/c374bec00a1c7cef4adbd1d1e9d323ceed44a91b/contracts/Octoplace_FLAT.sol#L2121-L2123

#### Recommendation

To fix the syntax error