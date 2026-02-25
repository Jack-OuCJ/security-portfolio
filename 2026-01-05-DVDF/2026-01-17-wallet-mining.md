---
title: Wallet Mining Audit Report
author: Jack
date: January 17, 2026
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

### [H-1] `AuthorizerUpgradeable::init()` can be called by anyone at any time, allowing attackers to reset authorization and gain arbitrary deployment permissions

**Description:** `AuthorizerUpgradeable` is a UUPS-upgradeable contract that restricts who can deploy Safe wallets to specific addresses via `WalletDeployer::drop()`. Its `init()` function — which sets the authorized wards and deployment targets — lacks a one-time initialization guard:

```solidity
// AuthorizerUpgradeable.sol
function init(address[] memory _wards, address[] memory _aims) external {
    // ⚠️ No initializer modifier, no "already initialized" check
    for (uint256 i = 0; i < _wards.length; i++) {
        wards[_wards[i]][_aims[i]] = true;
    }
}
```

Any external caller can invoke `init()` at any time with arbitrary `wards` and `aims` arrays. This allows an attacker to authorize themselves to deploy a Safe proxy to any target address — including the `USER_DEPOSIT_ADDRESS` that holds 20,000,000 DVT.

**Impact:** An attacker can:
1. Call `init([attacker], [USER_DEPOSIT_ADDRESS])` to self-authorize.
2. Call `WalletDeployer::drop()` to deploy a Safe proxy to `USER_DEPOSIT_ADDRESS`.
3. Execute transactions from the newly deployed Safe to transfer all 20,000,000 DVT to an arbitrary address.

**Proof of Concept:**

```solidity
// 1. Re-initialize authorizer to grant attacker deployment rights
address[] memory wards = new address[](1); wards[0] = address(exploitContract);
address[] memory aims = new address[](1);  aims[0] = USER_DEPOSIT_ADDRESS;
authorizer.init(wards, aims);

// 2. Deploy Safe proxy to USER_DEPOSIT_ADDRESS via WalletDeployer
walletDeployer.drop(safeCopy, initializer, saltNonce);

// 3. Use Safe's execTransaction to transfer 20M DVT
safe.execTransaction(token, 0, transferCalldata, ..., userSignature);
```

**Recommended Mitigation:**

Protect `init()` with a standard OpenZeppelin `initializer` modifier to ensure it can only be called once:

```diff
+ import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

- function init(address[] memory _wards, address[] memory _aims) external {
+ function init(address[] memory _wards, address[] memory _aims) external initializer {
      for (uint256 i = 0; i < _wards.length; i++) {
          wards[_wards[i]][_aims[i]] = true;
      }
  }
```

Furthermore, ensure the proxy's `initialize` function is called immediately upon deployment (in the same transaction) to prevent front-running of the initialization.

---

### [H-2] Create2 address of Safe proxy is predictable and brute-forceable, enabling deployment to any pre-funded deposit address

**Description:** `WalletDeployer::drop()` uses `SafeProxyFactory::createProxyWithNonce()` which deploys proxies at deterministic Create2 addresses. The address is derived from:
- `deployer = proxyFactory`
- `salt = keccak256(keccak256(initializer), saltNonce)`
- `initCodeHash = keccak256(SafeProxy.creationCode ++ singletonAddress)`

All three inputs are either publicly known or controllable by the caller (via `saltNonce`). An attacker can compute off-chain which `saltNonce` value will produce a proxy at any desired address, including pre-funded addresses that hold tokens.

**Impact:** Combined with `[H-1]` (bypassing authorization checks), an attacker can deterministically land a Safe proxy at `USER_DEPOSIT_ADDRESS` by selecting the correct `saltNonce`, then control all tokens at that address.

**Proof of Concept:**

```solidity
// Off-chain: compute correct saltNonce
for (uint256 nonce = 0; ; nonce++) {
    bytes32 salt = keccak256(abi.encodePacked(keccak256(initializer), nonce));
    address predicted = vm.computeCreate2Address(salt, initCodeHash, address(factory));
    if (predicted == USER_DEPOSIT_ADDRESS) {
        correctNonce = nonce;
        break;
    }
}

// On-chain: deploy proxy to USER_DEPOSIT_ADDRESS
walletDeployer.drop(safeCopy, initializer, correctNonce);
```

**Recommended Mitigation:**

1. Do not pre-fund addresses based solely on expected Create2 deployment — the address is public and computable by anyone.
2. Restrict `WalletDeployer::drop()` to a narrow set of callers (e.g., only the specific ward registered per address), and ensure the authorization check is immutable (not re-initializable).
3. If pre-funding is required, only release funds after verifying the deployed contract's bytecode and initialization state match expected values.

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
