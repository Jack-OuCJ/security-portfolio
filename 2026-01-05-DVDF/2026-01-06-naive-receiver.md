---
title: Naive Receiver Audit Report
author: Jack
date: January 06, 2026
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

### [H-1] Missing authorization check in `flashLoan()` allows anyone to drain any receiver's funds via forced fee collection

**Description:** `NaiveReceiverPool::flashLoan()` does not verify that the caller is authorized to initiate a loan on behalf of the specified `receiver`. Any external account can trigger a flashloan targeting any deployed `FlashLoanReceiver` contract:

```solidity
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
    external
    returns (bool)
{
    if (token != address(weth)) revert UnsupportedCurrency();

    // ⚠️ No check that msg.sender == address(receiver) or authorized by receiver
    weth.transfer(address(receiver), amount);
    totalDeposits -= amount;

    if (receiver.onFlashLoan(msg.sender, address(weth), amount, FIXED_FEE, data) != CALLBACK_SUCCESS) {
        revert CallbackFailed();
    }

    uint256 amountWithFee = amount + FIXED_FEE;
    weth.transferFrom(address(receiver), address(this), amountWithFee);
    ...
}
```

Each flashloan charges a fixed fee of 1 WETH, even for zero-amount loans. Since `FlashLoanReceiver` automatically approves repayment in `onFlashLoan()`, an attacker can register fake loans with `amount = 0` and repeatedly drain the receiver's balance by paying only the 1 WETH fee per call.

**Impact:** An attacker can drain the entire WETH balance of any `FlashLoanReceiver` contract without the receiver's consent. In the challenge scenario, the receiver holds 10 WETH which can be emptied with 10 zero-amount flashloan calls.

**Proof of Concept:**

```solidity
// Drain 10 WETH from FlashLoanReceiver via 10 forced fee-collection loans
for (uint256 i = 0; i < 10; i++) {
    pool.flashLoan(receiver, address(weth), 0, "");
}
// receiver.balance == 0 after 10 calls
```

**Recommended Mitigation:** Add authorization control so that only the receiver itself (or a pre-approved operator) can initiate a flashloan on its behalf:

```diff
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
    external
    returns (bool)
{
+   if (msg.sender != address(receiver)) revert UnauthorizedFlashLoan();
    ...
}
```

---

### [H-2] `_msgSender()` reads from calldata tail enabling Multicall+Forwarder identity spoofing to drain pool deposits

**Description:** `NaiveReceiverPool` overrides `_msgSender()` to support EIP-2771 meta-transactions. When called from the trusted forwarder, it reads the "true sender" from the last 20 bytes of `msg.data`:

```solidity
function _msgSender() internal view override returns (address) {
    if (msg.sender == trustedForwarder && msg.data.length >= 20) {
        // ⚠️ Reads sender from calldata tail — attacker can craft arbitrary value
        return address(bytes20(msg.data[msg.data.length - 20:]));
    } else {
        return super._msgSender();
    }
}
```

The pool also inherits `Multicall`, which uses `delegatecall` to execute sub-calls. Because `delegatecall` preserves the original `msg.data`, an attacker can craft a `multicall` payload where the last 20 bytes of the calldata contain the pool deployer's address. When `withdraw()` is called inside the `multicall`, `_msgSender()` returns the deployer's address instead of the attacker's, granting unauthorized access to the deployer's full deposit balance.

**Impact:** An attacker can combine `Multicall` + `BasicForwarder` to forge the identity of the pool deployer and call `withdraw()` to steal all funds deposited under the deployer's account (1000 WETH in the challenge scenario).

**Proof of Concept:**

```solidity
// 1. Build withdraw calldata with deployer address appended to the tail
bytes memory withdrawCall = abi.encodePacked(
    abi.encodeCall(pool.withdraw, (POOL_BALANCE, payable(recovery))),
    bytes32(uint256(uint160(deployer))) // last 20 bytes → _msgSender() returns deployer
);

// 2. Pack as multicall entry and send via forwarder
bytes[] memory calls = new bytes[](1);
calls[0] = withdrawCall;
bytes memory multicallData = abi.encodeCall(pool.multicall, (calls));

// 3. Sign and execute via BasicForwarder
forwarder.execute(Request({from: player, target: pool, data: multicallData, ...}), signature);
```

**Recommended Mitigation:**

- For `_msgSender()`: validate that the context is actually a forwarder-initiated call and do not allow `delegatecall` paths to inherit the appended address. Alternatively, restrict sensitive operations to use `msg.sender` directly.
- For `Multicall`: replace `delegatecall` with `call` so that `msg.data` is not carried into sub-calls, or strip the appended address before each delegatecall.
- Apply a stricter check: only allow the appended-address interpretation when the call originated directly from the forwarder (not via internal delegatecall chains).

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
