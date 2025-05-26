---
timezone: UTC+2
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# 你的名字

1. 自我介绍：我是 Zion，目前是一名合约开发者，熟悉 EVM 与 Solana 智能合约开发，基于跨链协议 LayerZero 开发了跨链 Token 与 Dex 合约。
2. 你认为你会完成本次残酷学习吗？I wish I can
3. 你的联系方式（推荐 Telegram）X: zlog_in

## Notes

<!-- Content_START -->

### 2025.05.14

#### EIP-7702 速览

* EIP-7702 目的是将 EOA 账号托管给合约账号，通过引入新的交易类型 `0x04` 实现授权托管与取消托管。
* `0x04` 交易结构： `[chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, destination, value, data, access_list, authorization_list,y_parity r, s]`
* 其中：`authorization_list = [[chain_id, address, nonce, y_parity r, s], ...]`。
* `authorization_list` 是一组 7702 签名数据，通过 `y_parityr,s` 获取 EOA Signer 地址，并将 `signer.code` 设置为 `0xef0100+address`, `address`  为目标合约地址，如果为〇地址，则为取消授权。
* `0x04` 交易其实是在 `0x03` (EIP-1559)基础上，挂载了一个额外的字段 `authorization_list` ，此字段内容将实现一组授权托管/取消托管操作；外部交易数据依然为一笔正常`0x03`交易，即可以为转账，或合约调用。`0x03` 部分与 `authorization_list` 可彼此独立
* 更详细内容可参见[此资料](https://docs.google.com/presentation/d/1CGHSzlRl-d8nxzPtAYF6Aq5wkCORvxg3SFs96TICAPg/edit?usp=sharing)

### 2025.05.15

#### EIP-7702 影响
* 在 EIP-7702 之前，以太坊 EOA 账号存在一些严格约束：只有私钥才能减少 EOA 账号的 ETH 余额；EOA 账号的 Code 与 Storage 为空；一笔 tx 最多只能将 EOA nonce + 1。
* 在 EIP-7702 之后，Smart EOA （被授权给智能合约的 EOA）余额可由合约代码控制；Smart EOA 的 Code 与 Storage 不再为空，其中 Storage 管理应遵循特定标准，例如[ERC-7201](https://eips.ethereum.org/EIPS/eip-7201)；一笔由 EOA signer 提交的 `0x04` 交易可将 signer nonce + 2。
* 在 EIP-7702 之后，Smart EOA 与 SC 之间界限模糊，仅凭之前的 `address.code.length > 0` 将无法区分，其判断逻辑应如下所示
```
     ┌────────────────────────┐                                 
     │address.code.length > 0 ├──────────────────► pure eoa     
     └───────────┬────────────┘          no                     
                 │                                              
                 │ yes                                          
                 │                                              
                 ▼                                              
┌────────────────────────────────────┐                          
│                                    │                          
│      address.code.legth == 23      ├──────────► smart contract
│ && address.code.hasPrefix 0xef0100 │   no                     
│                                    │                          
└────────────────┬───────────────────┘                          
                 │                                              
                 │ yes                                          
                 │                                              
                 ▼                                              
            smart eoa                                                                                     
```

### 2025.05.16

#### EIP-7702 细节
* 7702 签名过程：`cast wallet sign-auth address --private-key $PRIVATE_KEY --chain chain_id --nonce nonce ==> [chain_id, address, nonce, y_parity r, s]`
* 其中 `address` 地址可以是：一本智能合约地址，或者另外一个 Pure EOA（无效授权）；但不能是另外一个 Smart EOA。

### 2025.05.17

#### EIP-7702 风险
* 7702 签名由钱包工具支持，特别注意的是要严格检查 `chain_id` 与 `nonce`。如果 `chain_id == 0`，则意味该授权签名有可能被重复至其他 EVM 链上，前提是 nonce 刚好匹配。
* 在执行 7702 初次授权时，无法保证授权与 Smart EOA 存储状态的初始化同时发生，因此合约代码的初始化函数需要检查 `msg.sender == address(this)`，确保 Smart EOA 仅能通过私钥初始化。
* 在执行 7702 再授权时，应谨慎管理 Smart EOA 的存款空间，尽量避免存储冲突。最近实践是遵守ERC-7201标准，将存储空间离散化，以尽可能减少冲突风险。
* 因为 Smart EOA 可以调用自身合约接口，通过 `tx.origin == msg.sender` 来判断调用者否是为 EOA 将失效。
* 最大的风险是钓鱼攻击，即用户被诱导授权签名，将 EOA 授权给一本恶意合约，EOA 资产将有合约代码彻底控制。

### 2025.05.18
#### EIP-7702 展望
* 通过 EIP-7702 协议，传统私钥控制的 EOA 账号托管给智能合约代码。在不交出私钥的前提下，将 EOA 控制权转交给智能合约，同时保留私钥拥有撤销托管的能力。
* 通过特定合约的实现与链外组件的配合，可帮助用户实现无 gas 交互，无需用户支付 gas fee。
* 可选的 Authorization 包括 WebAuth、Passkey， 来辅助对 EOA 的访问鉴权，将降低链上交互的门槛。

### 2025.05.19
#### EIP-7702 应用场景
* 通过 EIP-7702，可以实现智能钱包的升级，将传统 EOA 钱包转变为可编程钱包，支持多签、社交恢复等高级功能。
* 在 DeFi 应用中，用户可以将 EOA 授权给特定的 DeFi 合约，实现自动化的投资策略和风险管理。
* 通过合约托管，可以实现更复杂的交易逻辑，如批量交易、条件触发交易等，提升用户体验。

### 2025.05.20
#### EIP-7702 实现建议
* 在开发支持 EIP-7702 的合约时，建议实现标准的接口，如 `IERC7702`，以便于其他合约进行交互和集成。
* 合约应该实现完善的权限管理机制，包括授权管理、撤销机制和紧急暂停功能。
* 建议在合约中实现事件日志，记录所有授权和撤销操作，便于链上追踪和审计。

### 2025.05.21
#### EIP-7702 安全影响
* EIP-7702 的长期安全影响主要体现在智能合约的权限管理上。通过将 EOA 的控制权委托给智能合约，实际上是将安全责任从用户转移到了合约代码上。
* 合约代码的质量和安全性将直接影响到用户资产的安全。因此，合约审计和形式化验证变得更加重要。
* 由于 Smart EOA 可以执行更复杂的操作，攻击面也随之扩大。合约开发者需要更加注重安全最佳实践，如重入攻击防护、权限检查等。
* 长期来看，可能会出现专门针对 Smart EOA 的新型攻击方式，需要持续关注和研究。

### 2025.05.22
#### EIP-7702 隐私影响
* EIP-7702 对用户隐私的影响主要体现在链上数据的可追踪性上。Smart EOA 的所有操作都会被记录在区块链上，增加了用户行为的可追踪性。
* 通过分析 Smart EOA 的调用模式，可能会泄露用户的交易习惯和偏好，这对隐私保护提出了新的挑战。
* 长期来看，可能需要开发新的隐私保护机制，如零知识证明或混币技术，来保护 Smart EOA 用户的隐私。
* 合约开发者需要在功能实现和隐私保护之间找到平衡点，避免过度收集用户数据。

### 2025.05.23
#### EIP-7702 便利性影响
* EIP-7702 的长期便利性影响主要体现在用户体验的改善上。通过智能合约托管，用户可以享受更丰富的功能和更便捷的操作。
* 无 gas 交易、批量操作、自动化策略等功能的实现，将大大降低用户使用门槛，提升用户体验。
* 长期来看，可能会出现更多创新的应用场景，如社交恢复、多签管理等，进一步简化用户操作。
* 随着工具和基础设施的完善，Smart EOA 的使用将变得更加简单和直观，推动更多用户采用。
<!-- Content_END --> 