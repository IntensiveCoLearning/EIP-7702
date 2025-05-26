---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# 你的名字

1. 自我介绍 York
2. 你认为你会完成本次残酷学习吗？ I will do my best.
3. 你的联系方式（推荐 Telegram）Telegram group -> York

## Notes

<!-- Content_START -->

### 2025.05.14
#### EIP-7702
Key Points
1. What is EIP-7702
2. How it works?
3. What is Set Code?
4. Difference between EIP-4337 and EIP-3074

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day01.md

### 2025.05.15
#### Security Risk of EIP-7702
Key Points
1. Private Key Leakage Risk
2. Delegated Contract Vulnerabilities Affect All Delegators
3. Storage Collision and Incompatibility Risk
4. Cross-Chain Replay Attack
5. Fake Deposit Risk
6. Account Conversion Breaks Security Assumptions
7. No Constructor Initialization Risk
8. Phishing and User Misguidance Risk

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day02.md


### 2025.05.16
#### Follow EIP-7702 sample code
Key Points
1. Prepare the environment
2. Sample Code Review
3. Local Testing and Deployment
4. Thoughts

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day03.md

### 2025.05.17
#### EIP-7702 potential vulnerabilities POC
Key Points
1. Dangers of EIP-7702
2. Proof of Concept (POC)
3. Metamask Smart Account

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day04.md

### 2025.05.18
#### EIP-7702 Modular Architecture with PREP
Key Points
1. What is PREP (Programmable Runtime EOA Proxy)  
2. Stateless delegation and modular logic  
3. Why modular Smart Accounts matter  

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day05.md

### 2025.05.19
#### Persistent Delegation and Multi-chain Use
Key Points
1. Moving from single-use to persistent delegation  
2. Cross-chain authorization with `chainId = 0`  
3. Smart Account evolution: 4337, 7702, and beyond 

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day06.md

### 2025.05.20
#### Future of EIP-7702 and Modular Account Ecosystem
Key Points
1. How EIP-7702 fits into ERC-6900 modular AA
2. Delegation as plug-in logic
3. ZyFi’s vision: infra-light, user-first abstraction

Notre Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day07.md

### 2025.05.21
#### Real-world EIP-7702 Application: Ambire Wallet
Key Points
1. How Ambire integrates EIP-7702 into its wallet
2. Gasless transactions and ERC-20 fee payment
3. Temporary smart account logic with no address change
4. Batch execution and transaction simulation
5. Recovery and session key support for secure UX

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day08.md


### 2025.05.22
#### Real-world EIP-7702: Privy and ZeroDev
Key Points 
1. Sign EIP-7702 authorizations with embedded wallets  
2. Enable gasless transactions with Paymaster via ZeroDev  
3. Configure custom wallet UI using PrivyProvider  
4. Create temporary smart accounts (7702 kernel) via SDK

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day09.md


### 2025.05.23
#### Advanced EIP-7702: Insecure Delegation Patterns in Practice
Key Points
1. Unprotected execute() calls drain delegated accounts
2. The Illusion of Guardians
3. Anyone Can Initialize: Insecure Setup of Delegated Contracts
4. Replayable Signatures: Initialization Can Be Cloned Across Chains
5. Upgrade and Storage Collision
6. Replay Attack: oneTimeSend Without Nonce Allows Unlimited Withdrawals
7. Safe Upgrade with Nonce and Slot Cleanup

Note Link: https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day10.md

<!-- Content_END -->
