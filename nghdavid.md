---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# 你的名字

1. David
2. Yes, will try my best
3. TG: nghdavid

## Notes

<!-- Content_START -->

### 2025.05.15
EIP-7702 是以太坊創辦人 Vitalik Buterin 於 2024 年提出的一項關鍵提案，旨在推動帳戶抽象（Account Abstraction）的發展，讓傳統的外部擁有帳戶（EOA）在不更換地址的情況下，獲得智能合約帳戶的靈活性與功能。

---

### EIP-7702 的核心機制

EIP-7702 引入了一種新的交易類型，允許 EOA 在單筆交易中臨時掛載智能合約代碼（`contract_code`），使其在交易期間具備智能合約的能力。交易結束後，該代碼會自動移除，EOA 恢復為原始狀態。

具體流程如下：

1. **設置代碼**：若 EOA 的合約代碼為空，則在交易開始時設置為提供的 `contract_code`。
2. **執行交易**：根據 `contract_code` 中定義的邏輯執行交易，通常包括 `verify`（驗證）和 `execute`（執行）兩個函數。
3. **移除代碼**：交易完成後，將 EOA 的合約代碼重置為空，恢復其原始狀態。

這一機制類似於 EIP-3074 中的 `AUTH` 和 `AUTHCALL` 操作碼，但 EIP-7702 不需引入新的操作碼，避免了對以太坊虛擬機（EVM）的永久性更改。

---

### 與其他提案的關係

* **EIP-3074**：EIP-7702 被視為 EIP-3074 的替代方案，提供了類似的功能，但通過交易類型實現，避免了新增操作碼帶來的潛在風險。
* **EIP-4337**：EIP-7702 與 EIP-4337 高度兼容，允許 EOA 利用現有的智能合約錢包代碼，實現如交易批處理、Gas 贊助等功能，促進了帳戶抽象的統一生態系統。
* **EIP-5003**：EIP-7702 為未來可能的永久性帳戶轉換（如 EIP-5003 提出的 EOA 永久轉換為智能合約帳戶）鋪平了道路。

---

### 實際應用與影響

EIP-7702 已被納入以太坊 2025 年的 Pectra 升級中，並受到社群廣泛支持。

**對用戶的好處**：

* **交易批處理**：可將多個操作合併為單筆交易，提升效率。
* **Gas 贊助**：允許第三方為用戶支付交易手續費，降低使用門檻。
* **無縫身份驗證**：支持多種簽名方式，如生物識別，提高安全性與便利性。
* **會話密鑰與限權帳戶**：實現臨時授權，提升操作靈活性。
* **靈活的安全策略**：支持多簽與社交恢復等安全機制。
* **跨鏈與模組化體驗**：支持跨多個網路使用同一套授權邏輯。
* **無需更換地址**：用戶可在不更換地址的情況下，享受智能合約帳戶的功能。

**對開發者的意義**：

* **錢包開發**：可在不改變用戶地址的情況下，提供智能帳戶功能。
* **智能合約/模組開發**：擴大智能合約錢包代碼的適用範圍，降低開發和維護成本。
* **DApp 集成**：可設計更複雜的交互流程，提升用戶體驗。

---


### 2025.05.16
*Security Risks*

- Lack of access control 

  If the delegate contract lacks proper access controls, attackers can execute arbitrary logic on behalf of the EOA, such as transferring the tokens out of the wallet.

- Initialization challenges 

  * Constructors: when you delegate code to an account, the constructor of the delegation designator contract does not execute in the context of the EOA
  * Front running and (re)initialization: If the contract uses an initialization pattern, users must make sure that the initialize function has proper access controls and that the contract can not be reinitialized by a malicious user. As a user, you’ll need to delegate and call the initialize function in the same transaction.

- Storage collisions
  
  Persistent state across redelegations and upgrades: Delegating code does not clear existing storage. When migrating from one delegation designator contract to another, users and developers must account for the old storage data to prevent storage collisions and unexpected behaviors.

### 2025.05.17
*Gas sponsorship*  
Any bundler/relayer can call this contract with correct calldata. EOA performs this required operation, while gas fees are incurred by relayer/bundler.

*Phishing attack of EIP-7702 wallet*  
A malicious EIP-7702 signature can drain your wallet in a single transaction.

*Transaction batching*  
Batch multiple transactions into single transaction

### 2025.05.18
*Account Abstraction*
- UserOperation
  ```
  struct UserOperation {
    address sender;
    uint256 nonce;
    bytes initCode;
    bytes callData;
    uint256 callGasLimit;
    uint256 verificationGasLimit;
    uint256 preVerificationGas;
    uint256 maxFeePerGas;
    uint256 maxPriorityFeePerGas;
    bytes paymasterAndData;
    bytes signature;
  }
  ```
  * 附加字段 - UserOperation 交易结构中的有一些新字段(如: paymasterAndData 等)
  * 另一个mempool - UserOperation 被发送到一个单独的mempool, 在那里捆绑器可以将 UserOperation 打包成真实的交易，并被包含在一个区块中。
  * 认证方式 - 对于一个传统交易, 认证总是通过一个私钥的签名来完成, 这个私钥对于一个给定的发起者来说永远不会改变。在UserOperation中,认证是可编程的。

- 捆绑器(Bundler)  
  捆绑器会监控一个专门为UserOperation建立的 mempool。捆绑器将多个UserOperation捆绑成一个交易, 并将该交易提交给入口点(EntryPoint)合约。捆绑器通过抽取部分 Gas 费用来获得补偿。
- 入口点(EntryPoint)
  EntryPoint是一个单例智能合约,用来接收来自捆绑器的交易,然后验证和执行UserOperation
- EntryPoint的执行过程是如何进行的?
  在执行过程中, EntryPoint合约通过使用UserOperation中指定的 calldata 来执行UserOperation, 并从智能合约账户中取钱给捆绑器报销合适的ETH来支付Gas
- Paymaster
  * Paymaster 是一个ERC-4337定义的智能合约,处理Gas 支付政策。支付政策为 Gas 的支付方式（例如，用什么货币）和由谁支付创造了灵活性，这消除了用户持有区块链原生代币才能与区块链交互的限制。
  * 允许应用程序开发人员使用其他ERC-20代币实现 Gas 支付
- 聚合器(Aggregator)
  * 聚合器是一个智能合约，它实现了一个支持聚合的签名方案
  * 通过将多个签名合并成一个签名, 聚合器有助于节省calldata成本

### 2025.05.19
*Important Limitations*

- The EOA's Private Key Remains All-Powerful  
  * The private key can always override any delegation by signing a new transaction
- No Deployment Permanence  
  * Unlike a deployed smart contract account with its own address, an EIP-7702 delegation can be overwritten
- Multichain Challenges
  * EIP-7702 authorizations are chain-specific, which means users would need to sign separate authorizations for each chain
- Controlled by wallets, not applications
  * Wallet providers have made it clear that they will reject transactions containing authorization fields from applications
  * Applications cannot directly use EIP-7702 to delegate user accounts to their preferred smart accounts
- ERC-7710/7715 Approach
  * ERC-7710: Creates a standard interface for smart contracts to delegate permissions to other contracts. Think of it as an extension of the ERC-20 approve function but for arbitrary permissions.
  * ERC-7715: Introduces a new JSON-RPC method called wallet_grantPermissions that applications can use to request permissions from wallets.
- Work flow
  * A wallet uses EIP-7702 to delegate the user's EOA to a smart contract that supports EIP-7710
  * An application requests permission via EIP-7715 to spend funds from the user's account
  * The wallet displays this request to the user
  * If approved, the application can now execute actions through the delegation
- Applications
  * Request direct permission for their contract to interact with the user's account, or
  * Deploy their own smart account (called a "companion account") that interacts with the user's account

### 2025.05.22
*BEST PRACTICES*
- Delegation contract should align with AA standards (ERC-4337 compatible)
- Stay Permissionless: Don’t hardcode relayers; anyone should be able to relay & censorship resistance
- Pick 4337 Bundlers: Use EntryPoint ≥ 0.8 for gas abstraction
- dApp Integration: Utilize ERC-5792 or ERC-6900, no standardized method for dApps to request 7702 authorization signatures directly
- Avoid Lock-In: Stick to open, interoperable standards like Alchemy’s Modular Account
- Preserve Privacy: Support ERC-20 gas payments, session keys, public mempools to minimize data exposure
- Use Proxies: Delegate to proxies for upgrades and modularity without requiring additional EIP-7702 authorizations for each change

<!-- Content_END -->
