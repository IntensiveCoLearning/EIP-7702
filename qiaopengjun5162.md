---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区

# 你的名字

1. 自我介绍
   我是一名Web3 开发者，对区块链技术非常感兴趣，希望能在本次残酷学习中学习到更多相关知识。
2. 你认为你会完成本次残酷学习吗？
   我相信只要我坚持不懈，努力学习，一定能够完成本次残酷学习。
3. 你的联系方式（推荐 Telegram）
   我的联系方式是 @Qiao4812，欢迎随时与我交流学习心得和问题。

## Notes

<!-- Content_START -->

### 2025.05.14

概述：EIP-7702 是以太坊的一项提案，允许外部拥有账户（EOAs）集成链上代码，推动账户抽象化，支持批量交易、多重签名等功能。然而，这也带来了开发者必须关注的重大安全风险。
EIP-7702 核心特性：
新交易类型：EOAs 可通过委托指示器（前缀+链上代码地址）将行为委托给智能合约。
授权列表：包含链 ID、委托合约地址、随机数和签名。
委托机制：支持批量操作或燃气费用赞助等高级功能，但 EOA 的私钥仍完全控制账户，若私钥泄露，账户将面临风险。
与 EIP-4337 比较：EIP-7702 集成更精简，而 EIP-4337 提供更全面的账户抽象模型，两者可针对不同场景共存。
安全风险：
访问控制问题：
若委托合约缺乏访问控制，攻击者可代表 EOA 执行任意逻辑，如转移代币。
示例：无访问控制的 doSomething 函数可被任何人调用，执行恶意交易。
初始化挑战：
委托时，合约构造函数不执行，可能导致行为预期错误。
需使用初始化模式（如代理合约），但必须设置访问控制，防止攻击者抢先调用或重新初始化。
存储冲突：
重新委托时，原存储数据不清除，可能导致冲突（如 bool 被误解为 uint），引发未定义行为或安全漏洞。
结论：EIP-7702 增强了以太坊账户模型的灵活性，但也扩大了攻击面。开发者需精心设计、彻底审计并强化访问控制，以确保安全实施。

### 2025.05.15

交易不能是创建合约的交易

在以太坊标准交易中，msg.To == nil 表示试图创建合约，而 EIP-7702 交易不能是创建合约的交易，因此禁止 msg.To == nil。

EIP-7702 交易必须包含至少一条授权条目，否则因空授权列表（ErrEmptyAuthList）而被拒绝。

如果 EIP-7702 交易的授权列表中包含同一个用户的多个授权条目，只有最后一个条目会生效，其余会被覆盖。

如果 EIP-7702 交易的授权条目关联的账户与发起者是同一账户，仍需两次签名：一次为交易签名，一次为授权条目签名。

### 2025.05.16

在 EIP-7702 交易中，msg.To == nil 表示试图创建合约被禁止，交易必须包含至少一条授权条目，若授权条目与发起者同账户需两次独立签名，且授权 nonce 与交易 nonce 通常独立，理论上无需比交易 nonce 多 1。

### 2025.05.17

在 EIP-7702 交易中，如果当前授权条目在执行过程中遇到错误（例如，签名无效或逻辑失败），它不会导致整个交易失败，而是会被跳过，EVM 会继续处理授权列表中的下一条条目。

在 EIP-7702 交易中，若某个授权条目执行时遇到错误，它会被跳过而不影响交易整体成功，EVM 会继续处理下一个授权条目。

### 2025.05.18

<https://github.com/ethereum/go-ethereum/blob/701df4baad3bbdb0bdf4c837f19d25cb07ffc3af/core/state_transition.go#L611>

```go
// Otherwise install delegation to auth.Address.
st.state.SetCode(authority, types.AddressToDelegation(auth.Address))

```

<https://github.com/ethereum/go-ethereum/blob/701df4baad3bbdb0bdf4c837f19d25cb07ffc3af/core/types/tx_setcode.go#L34>

<https://github.com/ethereum/go-ethereum/blob/701df4baad3bbdb0bdf4c837f19d25cb07ffc3af/core/types/tx_setcode.go#L45>

```go
// DelegationPrefix is used by code to denote the account is delegating to
// another account.
var DelegationPrefix = []byte{0xef, 0x01, 0x00}

// AddressToDelegation adds the delegation prefix to the specified address.
func AddressToDelegation(addr common.Address) []byte {
 return append(DelegationPrefix, addr.Bytes()...)
}
```

### 2025.05.19

Best Practices

- PrivateKey Management 私钥保护
- Multi-chain replay  chainId 0  相同的合约地址代码是否一定相同
- No initcode  初始化权限检查 避免抢跑
- Storage Management  插槽  ERC-7201   ERC-7779 验证 插槽
- False Top-up  假充值  检查状态
- Account Conversion  打破 msg.sender ==   tx.origin 安全检查
- Compatibility  兼容性  ERC-721 ERC-777
- Phishing 钓鱼风险

Remember: Not your keys, not your coins.

### 2025.05.20

如果某个授权条目在执行过程中因无效签名或逻辑失败等原因出错，该条目会被 EVM 跳过，而不会导致整个交易失败，EVM 会继续处理授权列表中的下一个条目。这种机制增强了交易的容错性和灵活性，适用于批量授权或账户抽象化场景，但开发者需注意 Gas 消耗和潜在的安全问题。若在测试 EIP-7702 交易时遇到 HTTP 502 错误，可能是 RPC 节点问题，建议切换提供商或检查节点状态。

### 2025.05.21
  
ERC-4337（账户抽象）通过智能合约钱包替代传统外部账户（EOA），优化区块链用户体验，核心组件包括UserOperation（可编程交易意图）、Bundler（打包交易）、EntryPoint（执行验证）、Paymaster（灵活支付Gas）和Aggregator（聚合签名）。它在不修改底层链的情况下运行，支持无Gas交易、多代币支付等功能，是以太坊生态的重要升级，此前基于EIP-2938和EIP-3074的探索。

### 2025.05.22

EIP-7702 是 Biconomy 博客于 2025 年 2 月 28 日由 Mislav Javor 发布的一篇文章中介绍的以太坊改进提案，旨在通过允许外部拥有账户（EOAs）委托执行给智能合约来提升用户体验，而无需迁移到新的智能账户。它解决了 dApp 交互中的摩擦问题，支持交易批量处理、燃气费赞助和权限管理，同时保持 EOA 的地址和兼容性。然而，EIP-7702 由钱包而非应用控制，可能导致不同钱包采用不同实现方式而产生碎片化。开发者可利用 ERC-7710、ERC-7715 和 ERC-5792 标准，或通过 Biconomy 的伴侣账户和 AbstractJS SDK 简化整合，大幅降低智能账户部署成本。尽管存在 EOA 私钥保留完全控制权和多链挑战等限制，EIP-7702 是迈向更用户友好区块链交互的重要一步。

### 2025.05.23

笔记内容

<!-- Content_END -->
