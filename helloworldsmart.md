---
timezone: UTC+8
---
# helloworldsmart

1. 自我介绍 Michael，希望自己按時學習
2. 你认为你会完成本次残酷学习吗？   會的
3. 你的联系方式（推荐 Telegram

## Notes

<!-- Content_START -->

### 2025.05.14

# [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) Introduction

EIP-7702 is an Ethereum Improvement Proposal introduced by Vitalik Buterin in 2024. Its goal is to improve the user experience and security related to **Account Abstraction**, particularly addressing the challenges of transforming **Externally Owned Accounts (EOAs)**.

---

### Core Concept

EIP-7702 proposes a new approach that allows a traditional EOA (e.g., a regular Ethereum wallet address) to **temporarily act as a smart contract account within a specific transaction**, enabling it to leverage account abstraction functionalities (such as custom verification logic, alternative gas payment methods, batch transactions, etc.).

---

### How It Works

EIP-7702 introduces:

#### `tx.accountCode` Field

A new transaction field that specifies the contract code the user wants their address to temporarily execute during the transaction.

* This code is deployed to the address before transaction execution.
* Once the transaction is completed, the code is automatically removed, and the address reverts to a pure EOA state.

---

### Advantages over EIP-4337

| Feature                                     | EIP-4337                            | EIP-7702                            |
| ------------------------------------------- | ----------------------------------- | ----------------------------------- |
| Model                                       | Full simulation on Layer 2          | Modifies L1 native EOA behavior     |
| Requires Paymaster / Bundler                | Yes                                 | Not necessarily                     |
| User Sovereignty                            | Contract account created by factory | User retains their original address |
| Support for Cold Wallets / External Signers | More complex                        | Simpler                             |
| Miner-Friendliness                          | Poor (non-native)                   | Native transaction support          |

---

### 📌 Use Cases

* Users can use their existing EOA wallet address to temporarily perform actions like:

  * Multi-step transactions (e.g., DEX swap + NFT minting)
  * Custom gas payment logic
  * ZK proof-based signature replacement
* After the transaction, the address reverts back to a normal EOA without becoming a permanent smart contract account.



### 2025.05.15

<!-- Content_END -->
