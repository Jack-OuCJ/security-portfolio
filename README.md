# Smart Contract Security Portfolio

> **Author:** Jack | [GitHub](https://github.com/Jack-OuCJ)

A collection of smart contract security audit reports and DeFi challenge write-ups I have completed.

---

## Table of Contents

- [Audit Reports](#audit-reports)
- [Damn Vulnerable DeFi (DVDf) Challenges](#damn-vulnerable-defi-dvdf-challenges)
- [Skills & Focus Areas](#skills--focus-areas)

---

## Audit Reports

Private/competitive audits completed as part of security research training.

| Date | Protocol | Category | Report |
|------|----------|----------|--------|
| 2025-07-23 | PasswordStore | Access Control / Storage | [PDF](2025-07-23-passwordstore/2025-07-23-passwordstore-audit.pdf) |
| 2025-07-25 | PuppyRaffle | Randomness / Reentrancy | [PDF](2025-07-25-PuppyRaffle/2025-07-25-PuppyRaffle-Audit.pdf) |
| 2025-07-29 | TSwap | AMM / Price Manipulation | [PDF](2025-07-29-TSwap/2025-07-29-TSwap-Audit.pdf) |
| 2025-07-30 | ThunderLoan | Flash Loans / Logic Flaws | [PDF](2025-07-30-ThunderLoan/2025-07-30-ThunderLoan-Audit.pdf) |
| 2025-08-01 | L1BossBridge | Cross-chain Bridge Security | [PDF](2025-08-01-L1BossBridge/2025-08-01-L1BossBridge-Audit.pdf) |
| 2025-08-07 | Vault Guardian | Vault / ERC4626 / DeFi | [MD](2025-08-07-Vault-Guardian/2025-08-07-Vault-Guardian-Audit.md) |

---

## Damn Vulnerable DeFi (DVDf) Challenges

Write-ups for the [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/) challenge series.

| # | Challenge | Key Vulnerability | Write-up |
|---|-----------|-------------------|----------|
| 1 | Unstoppable | Flash loan / vault accounting | [MD](2026-01-05-DVDF/2026-01-05-unstoppable.md) |
| 2 | Naive Receiver | Unauthorized flash loan calls | [MD](2026-01-05-DVDF/2026-01-06-naive-receiver.md) |
| 3 | Truster | Arbitrary call in flash loan | [MD](2026-01-05-DVDF/2026-01-07-truster.md) |
| 4 | Side Entrance | Reentrancy via flash loan | [MD](2026-01-05-DVDF/2026-01-08-side-entrance.md) |
| 5 | The Rewarder | Reward manipulation via flash loan | [MD](2026-01-05-DVDF/2026-01-09-the-rewarder.md) |
| 6 | Selfie | Governance / flash loan attack | [MD](2026-01-05-DVDF/2026-01-10-selfie.md) |
| 7 | Compromised | Oracle price manipulation | [MD](2026-01-05-DVDF/2026-01-11-compromised.md) |
| 8 | Puppet | Uniswap v1 oracle manipulation | [MD](2026-01-05-DVDF/2026-01-12-puppet.md) |
| 9 | Puppet v2 | Uniswap v2 oracle manipulation | [MD](2026-01-05-DVDF/2026-01-13-puppet-v2.md) |
| 10 | Free Rider | NFT marketplace / flash loan | [MD](2026-01-05-DVDF/2026-01-14-free-rider.md) |
| 11 | Backdoor | Gnosis Safe proxy exploit | [MD](2026-01-05-DVDF/2026-01-15-backdoor.md) |
| 12 | Climber | Timelock privilege escalation | [MD](2026-01-05-DVDF/2026-01-16-climber.md) |
| 13 | Wallet Mining | CREATE2 address mining | [MD](2026-01-05-DVDF/2026-01-17-wallet-mining.md) |
| 14 | Puppet v3 | Uniswap v3 TWAP oracle attack | [MD](2026-01-05-DVDF/2026-01-18-puppet-v3.md) |
| 15 | ABI Smuggling | Calldata / selector bypass | [MD](2026-01-05-DVDF/2026-01-19-abi-smuggling.md) |
| 16 | Shards | ERC1155 / rounding exploit | [MD](2026-01-05-DVDF/2026-01-20-shards.md) |
| 17 | Withdrawal | L2→L1 bridge withdrawal exploit | [MD](2026-01-05-DVDF/2026-01-21-withdrawal.md) |
| 18 | Curvy Puppet | Curve Finance / liquidity attack | [MD](2026-01-05-DVDF/2026-01-22-curvy-puppet.md) |

---

## Skills & Focus Areas

- Solidity smart contract auditing
- DeFi protocol security (AMMs, flash loans, oracles, bridges)
- Common vulnerability patterns: reentrancy, access control, integer overflow, price manipulation
- Cross-chain bridge security
- Governance attack vectors
- Tools: Foundry, Slither, manual code review
