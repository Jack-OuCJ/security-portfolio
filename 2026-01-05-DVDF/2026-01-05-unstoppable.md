---
title: Unstoppable Audit Report
author: Jack
date: January 05, 2026
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

### [H-1] ERC4626 invariant broken by direct token transfer causes permanent flashloan DoS

**Description:** `UnstoppableVault` enforces a strict invariant check inside `flashLoan()` before each loan is issued:

```solidity
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
```

`balanceBefore` is derived from `totalAssets()`, which directly reads the contract's live token balance (`asset.balanceOf(address(this))`). `convertToShares(totalSupply)` computes the expected share-to-asset ratio. When both sides of the equation are equal the check passes. However, anyone can call `token.transfer(address(vault), amount)` directly, bypassing the `deposit()` function. This increases `totalAssets()` without minting new shares, making the two sides permanently unequal.

```solidity
// UnstoppableVault.sol
function flashLoan(...) external returns (bool) {
    ...
    uint256 balanceBefore = totalAssets();
    // ⚠️ Strict equality check that can be broken by direct transfer
    if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
    ...
}
```

**Impact:** Any user who directly transfers even a single token unit to the vault permanently disables the free flashloan service. The `UnstoppableMonitor` detects the broken invariant and triggers ownership transfer, effectively locking the protocol's core function forever. There is no recovery path once the invariant is broken.

**Proof of Concept:**

```solidity
function test_unstoppable() public checkSolvedByPlayer {
    // Directly transfer tokens, bypassing deposit()
    token.transfer(address(vault), INITIAL_PLAYER_TOKEN_BALANCE);
    // Result: convertToShares(totalSupply) < totalAssets() → permanent DoS
}
```

Attack steps:
1. Attacker holds any amount of the vault's underlying token.
2. Attacker calls `token.transfer(address(vault), 1)` directly (not via `deposit()`).
3. `totalAssets()` increases but `totalSupply` does not change.
4. `convertToShares(totalSupply)` returns a value less than `totalAssets()`.
5. Every subsequent `flashLoan()` call reverts with `InvalidBalance`.
6. `UnstoppableMonitor.checkFlashLoan()` detects failure and escalates ownership, permanently killing the service.

**Recommended Mitigation:**

Remove the invariant check entirely. The security of the flashloan is already guaranteed by the repayment check (`safeTransferFrom` will revert if the borrower does not return funds). The share-to-asset ratio is not a necessary safety condition:

```diff
function flashLoan(...) external returns (bool) {
    if (amount == 0) revert InvalidAmount(0);
    if (address(asset) != _token) revert UnsupportedCurrency();
    uint256 balanceBefore = totalAssets();
-   if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
    ...
}
```

Alternatively, use an internal accounting variable instead of `balanceOf` to track assets, so external direct transfers do not affect the invariant.

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
