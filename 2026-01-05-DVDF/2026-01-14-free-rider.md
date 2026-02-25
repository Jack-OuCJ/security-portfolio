---
title: Free Rider Audit Report
author: Jack
date: January 14, 2026
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---


Prepared by: [Jack](https://github.com/Jack-OuCJ)

# Table of Contents
- [Severity Classification](#severity-classification)
- [Summary](#summary)
- [Findings](#findings)
  - [High](#high)

# Severity Classification

| Severity         | Description |
| ---------------- | ----------- |
| ![high]          | A **directly** exploitable security vulnerability that leads to stolen/lost/locked/compromised assets or catastrophic denial of service. |
| ![medium]        | Assets not at direct risk, but the function of the protocol or its availability could be impacted. |
| ![low]           | A violation of common best practices or incorrect usage of primitives, which may not currently have a major impact on security. |
| ![informational] | Non-critical comments, recommendations or potential optimizations, not relevant to security. |

# Summary

| Severity         | Number of issues found |
| ---------------- | ---------------------- |
| ![high]          | 2                      |
| ![medium]        | 0                      |
| ![low]           | 0                      |
| ![informational] | 0                      |
| Total            | 2                      |

# Findings

## High

### [H-1] `buyMany()` validates `msg.value` against single-item price instead of total price, allowing batch purchase of N NFTs for the cost of one

**Description:** `FreeRiderNFTMarketplace::buyMany()` iterates over token IDs and calls `_buyOne()` for each. Inside `_buyOne()`, the payment check compares `msg.value` (the total ETH sent once for the entire transaction) against the price of a single token:

```solidity
function buyMany(uint256[] calldata tokenIds) external payable nonReentrant {
    for (uint256 i = 0; i < tokenIds.length; ++i) {
        _buyOne(tokenIds[i]);
    }
}

function _buyOne(uint256 tokenId) private {
    uint256 priceToPay = offers[tokenId];
    // ⚠️ msg.value is fixed for the entire transaction — this check passes for every iteration
    if (msg.value < priceToPay) {
        revert InsufficientPayment();
    }
    ...
}
```

`msg.value` does not change between loop iterations. An attacker can buy 6 NFTs at 15 ETH each (total 90 ETH) by sending only 15 ETH, since each `_buyOne` call independently passes the check.

**Impact:** An attacker can acquire all 6 listed NFTs for the price of just one (15 ETH), causing a loss of 75 ETH in missing seller revenue and the unintended transfer of 5 NFTs that were not paid for.

**Proof of Concept:**

```solidity
// Send only 15 ETH but buy 6 NFTs — each _buyOne check sees 15 >= 15
uint256[] memory ids = new uint256[](6);
for (uint256 i; i < 6; i++) ids[i] = i;
marketplace.buyMany{value: 15 ether}(ids); // 6 NFTs acquired for 15 ETH
```

**Recommended Mitigation:**

Validate the total payment against the sum of all item prices before processing individual purchases:

```diff
function buyMany(uint256[] calldata tokenIds) external payable nonReentrant {
+   uint256 totalPrice;
+   for (uint256 i = 0; i < tokenIds.length; ++i) {
+       totalPrice += offers[tokenIds[i]];
+   }
+   if (msg.value < totalPrice) revert InsufficientPayment();
    for (uint256 i = 0; i < tokenIds.length; ++i) {
        _buyOne(tokenIds[i]);
    }
}
```

---

### [H-2] `_buyOne()` reads `ownerOf()` after transferring NFT, sending seller payment to the buyer instead of the seller

**Description:** In `_buyOne()`, the NFT is transferred to `msg.sender` before the seller's address is looked up via `ownerOf()`. Since transfer updates ownership, `ownerOf()` now returns `msg.sender` (the buyer), so the payment is sent back to the buyer rather than to the original seller:

```solidity
function _buyOne(uint256 tokenId) private {
    uint256 priceToPay = offers[tokenId];
    ...
    DamnValuableNFT _token = token;

    // Step 1: Transfer NFT — ownership changes to msg.sender
    _token.safeTransferFrom(_token.ownerOf(tokenId), msg.sender, tokenId);

    // Step 2: ⚠️ ownerOf now returns msg.sender (buyer), not original seller
    payable(_token.ownerOf(tokenId)).sendValue(priceToPay);

    emit TokenBought(msg.sender, tokenId, priceToPay);
}
```

This means the marketplace pays the buyer for their own purchase — sellers receive nothing. Combined with `[H-1]`, an attacker can obtain all NFTs while the marketplace actually refunds them the single 15 ETH payment they sent.

**Impact:** Sellers receive zero ETH for their NFTs. The marketplace re-pays ETH to buyers, meaning the protocol suffers a double loss: NFTs leave the marketplace and ETH is refunded to the attacker simultaneously. In the challenge scenario, the attacker nets both all 6 NFTs and the 45 ETH bounty.

**Proof of Concept:**

```solidity
// Attacker sends 15 ETH, gets ETH back + 6 NFTs
// Market balance decreases by 15 ETH (refund to buyer) but 0 ETH goes to sellers
```

**Recommended Mitigation:**

Cache the seller address **before** the NFT transfer, then send payment to the cached address:

```diff
function _buyOne(uint256 tokenId) private {
    uint256 priceToPay = offers[tokenId];
    if (msg.value < priceToPay) revert InsufficientPayment();

+   address seller = token.ownerOf(tokenId); // ✅ Cache seller before transfer
    --offersCount;

    DamnValuableNFT _token = token;
    _token.safeTransferFrom(seller, msg.sender, tokenId);

-   payable(_token.ownerOf(tokenId)).sendValue(priceToPay);
+   payable(seller).sendValue(priceToPay); // ✅ Pay the real seller
    emit TokenBought(msg.sender, tokenId, priceToPay);
}
```

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
