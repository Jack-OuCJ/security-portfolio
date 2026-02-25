---
title: Curvy Puppet Audit Report
author: Jack
date: January 22, 2026
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

### [H-1] Read-only reentrancy via Curve `get_virtual_price()` allows artificial inflation of LP token valuation to trigger false liquidations and steal collateral

**Description:** `CurvyPuppetLending` calculates the value of its LP token-denominated debt using Curve's `get_virtual_price()`:

```solidity
function getBorrowValue(uint256 borrowAmount) public view returns (uint256) {
    // ⚠️ Relies on Curve's get_virtual_price() which is inconsistent mid-execution
    return oracle.getPrice(ETH).value.mulWadDown(curvePool.get_virtual_price()).mulWadDown(borrowAmount);
}
```

Curve's `get_virtual_price()` returns a value derived from the pool's current invariant and total LP supply. During Curve's `remove_liquidity()` execution, the pool sends ETH to the caller's `receive()` function **before** updating all internal state variables. At this intermediate point, `get_virtual_price()` returns a transiently **inflated** value because token balances have been partially reduced while the invariant has not yet been recalculated.

An attacker can exploit this state inconsistency using a read-only reentrancy pattern:

1. Call `curvePool.remove_liquidity(...)` to initiate liquidity removal.
2. When Curve sends ETH to the attacker's `receive()` function during its execution, reenter `CurvyPuppetLending::liquidate()`.
3. Inside the reentered liquidation, `getBorrowValue()` reads the transiently inflated `get_virtual_price()`, making healthy positions appear undercollateralized.
4. The attacker liquidates these artificially unhealthy positions and seizes all DVT collateral.

`CurvyPuppetLending` has `nonReentrant` protection on `liquidate()`, but this only prevents reentrancy into the lending contract itself — it does not prevent external contracts from reading Curve's inconsistent state during Curve's own execution.

**Impact:** An attacker can falsely trigger liquidation of all borrower positions during Curve's liquidity removal execution window, seizing all deposited DVT collateral. In the challenge scenario, 3 borrowers with 2,500 DVT collateral each (7,500 DVT total) are vulnerable.

**Proof of Concept:**

```solidity
contract Attacker {
    bool liquidating;

    function attack() external {
        // 1. Add large liquidity to Curve pool (requires flashloans for capital)
        curvePool.add_liquidity{value: largeAmount}([ethAmount, stethAmount], 0);

        // 2. Remove liquidity — triggers ETH transfer to receive()
        curvePool.remove_liquidity(lpBalance - 1, [uint256(0), uint256(0)]);
    }

    receive() external payable {
        // 3. During Curve's ETH transfer, get_virtual_price() is transiently inflated
        if (msg.sender == address(curvePool) && !liquidating) {
            liquidating = true;
            // 4. Liquidate all positions — they appear undercollateralized due to inflated LP price
            for (uint i = 0; i < users.length; i++) {
                lending.liquidate(users[i]);
                // Attacker receives 2,500 DVT collateral per liquidation
            }
        }
    }
}
```

**Recommended Mitigation:**

1. **Do not use `get_virtual_price()` as a real-time price oracle (recommended):** This function is explicitly documented by Curve as unsafe for use in on-chain price feeds due to the read-only reentrancy vulnerability. Use a time-averaged LP token price instead, or a Chainlink LP token price feed.

2. **Implement a reentrancy lock that covers the read:** Use a custom lock that prevents `liquidate()` from being called while any Curve pool is in an active execution context. Curve provides a `claim_admin_fees()` call that can be used to detect in-progress pool state changes.

3. **Add Curve's recommended mitigation:** Before reading `get_virtual_price()`, call an empty exchange or other Curve function that forces state consistency, or check that `curvePool.is_killed()` returns false.

4. **Use separate oracle sources for LP token prices:** Price LP tokens using the underlying asset prices from independent oracles (e.g., Chainlink ETH/USD + stETH/USD) rather than reading from the Curve pool itself.

[high]: https://img.shields.io/badge/-High-red "High"
[medium]: https://img.shields.io/badge/-Medium-orange "Medium"
[low]: https://img.shields.io/badge/-Low-yellow "Low"
[informational]: https://img.shields.io/badge/-Informational-blue "Informational"
