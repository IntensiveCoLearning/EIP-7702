---
timezone: UTC+12
---

# Bruce Xu

1. 自我介绍：E 卫兵
2. 你认为你会完成本次残酷学习吗？：会
3. 你的联系方式（推荐 Telegram）：@brucexu_eth

## Notes

<!-- Content_START -->

## 2025.05.14

# https://eip.fun/eips/eip-7702

This EIP therefore focuses on adding short-term functionality improvements to EOAs which will allow UX improvements to permeate through the entire application stack.

三大用例（TODO 分别制作 Demo）：

- Batching：多个交易合并一笔
- Sponsorship：使用其他账号和 ERC-20 来为当前账号代付 gas
- Privilege de-escalation：授权 sub-keys 用于低安全性使用场景，这样可以减少签名的操作

# 2025.05.15

authorization_list = [[chain_id, address, nonce, y_parity, r, s], ...]

The authorization list is processed before the execution portion of the transaction begins, but after the sender's nonce is incremented.

EOA 的执行流程：

1. 发送 SET_CODE_TX_TYPE 0x04 + authorization_list 交易，包含授权的信息
2. EOA 对 authorization_list 签名 + 循环验证 pre-execution checks
3. 依次处理 `authorization_list` 中的每个授权条目：验证授权并将有效授权设置到对应 EOA 的代码区，使其指向一个委托合约地址（即设置 Delegation Indicator）。若同一 EOA 有多个授权，则以列表中最后一个有效授权为准。
4. `authorization_list` 处理完毕后，执行交易的主要操作（如 `data` 字段中定义的调用）。此时，被成功设置了 Delegation Indicator 的 EOA 将作为代理（proxy）执行其委托合约的代码。
5. 交易完成后，EOA 上设置的 Delegation Indicator 会持续存在，除非通过新的 EIP-7702 交易来更改或清除它。

Delegation Indicator：

- 设置到 EOA code 字段，告诉 EVM 用特殊的方式执行这个 code
  - 使用 0xef0100 开头，后面是 20 bytes 的以太坊地址，是一个智能合约，包含了当前 EOA 被调用时候，真正的执行逻辑
  - 存在 code 的 EOA，行为上像是代理合约
  - 当有交易跟 EOA 交互，EVM 会加载 indicator 指向的合约代码
  - 作为代理合约，被执行的合约代码使用 EOA 的 context，所以 msg.sender 和 msg.value 都是当前 EOA 的
- 通过 7702 设置了 Delegation Indicator 就会一直存在，除非被另一个 7702 tx 修改或者清理，不会自动消失，实现持久性

TODO Delegation Indicator 会有多个吗？还是单例？如果设置过多个 authorization tuple，会是什么效果？

# 2025.05.16

可以直接指向 ERC-4337 或者 RIP-7560，这样可以低成本使用。TODO 做一个 Demo。

## Self-sponsoring: allowing tx.origin to set code

tx.origin 始终是最初的 EOA，msg.sender 可能是根据调用递进的。所以之前用 tx.origin == msg.sender 来判断是否有中间调用以及当前调用者是不是一个 EOA。

关于合约里面这个判断逻辑的安全风险，EIP 上面做了简单的分析。

传统的基于 relayer 的 gas sponsorship 流程是这样的：

1. 用户打算对一个 dapp 交互，但是没有 ETH，签署这个交易意图，链下发送给 relayer
2. relayer（有 ETH 的 EOA），验证有效性和从用户的其他途径收费，然后组装发送 tx，支付 gas
3. 合约被调用，此时 tx.origin 和 msg.sender 是 relayer，但是合约需要有能力验证签名，将实际功能作用于实际意图发起者

基于 7702 的新的流程：

1. 用户发起 0x04 交易，将 EOA Code 指向委托合约逻辑，然后直接执行委托合约的逻辑
2. gas 还是需要自己支付，但是不需要 relayer 进行执行

7702 可以让 EOA 可以设置 code 到某些更复杂的 AA 账号，使用相应功能，但是第一步的 Delegation Indicator 的设置费用还是需要的，如果后面不被清理，这是一次性的费用。估计还是需要让钱包进行赞助。

TODO 这里 AA 钱包项目其实可以开发一个 gas airdrop 项目，如果确定要设置和使用当前 AA，则可以在这里领取 gas，同时用于设置 Delegation indicator 到当前 AA。

# 2025.05.17

This EIP breaks a few invariants:

- An account balance can only decrease as a result of a transaction originating from that account.
  - Once an account has been delegated, any call to the account may also cause the balance to decrease.
  - 之前的任何 balance 的变动，都是要 EOA 自身去支付 gas 和转账，有了 7702 之后，EOA 变成了 Smart EOA，外部可以调用这个 EOA，然后实际上调用了委托逻辑合约的代码，然后对 EOA 的余额进行操作
    - 好危险啊，TODO 做一个安全的案例，授权了不安全的委托逻辑合约，可以被外部调用，盗走测试币。如果一个 EOA 曾对某个（可能是恶意的）合约进行了 ERC-20 代币的无限额度授权（approve(spender, type(uint256).max)），然后该 EOA 又通过 EIP-7702 委托给了一个可以被外部调用的、能触发该 spender 合约 transferFrom 的委托逻辑，那么风险会进一步放大。
- An EOA nonce may not increase after transaction execution has begun.
  - Once an account has been delegated, the account may call a create operation during execution, causing the nonce to increase.
- tx.origin == msg.sender can only be true in the topmost frame of execution.
  - Once an account has been delegated, it can invoke multiple calls per transaction.

EIP-7702 本身不提供直接的风险缓解机制，它赋予了 EOA 这种强大的能力，同时也要求用户和开发者承担相应的责任，TODO 可以做一些基础设施：

- 只委托给受信任的、经过审计的合约：这是最重要的一点。用户在通过 EIP-7702 设置 Delegation Indicator 时，必须极端谨慎，确保其指向的合约地址是完全可信的、代码是安全的、并且经过了严格的第三方安全审计。
  - 合约审计查询和安全报告
- 最小权限原则：如果可能，委托的合约应该遵循最小权限原则，只暴露必要的功能，并对这些功能进行严格的访问控制。
  - 7702 合约授权权限解释工具 + 可能的潜在风险
- 使用成熟的、标准化的钱包实现：例如，如果 EOA 委托给一个符合 ERC-4337 标准的、经过良好测试的智能账户实现，那么可以依赖该标准和实现的安全性。
  - 4337 AABeat 的安全性测试标准
- 权限降级用例的谨慎设计：在实现“权限降级”（例如，创建一个只能与特定 DApp 交互的子密钥）这类用例时，委托逻辑合约需要被精心设计，以确保它不能被滥用来执行超出预期的操作。
  - 权限控制面板，支持精细化的自定义权限等，或者进行权限管理、撤销等
- 用户教育和钱包的责任：钱包提供商在支持 EIP-7702 时，有责任向用户清晰地解释这种委托操作的潜在风险，并提供工具来审查和管理委托。可能需要有明确的警告，告知用户他们正在将其账户的控制权（部分或全部）交给另一个合约。
  - 对 Delegation Indicator 进行 AI 自动化安全扫描和解析，并提示可能风险
- 及时撤销委托：如果用户怀疑委托的合约有问题，或者不再需要该委托，应尽快通过发送一个新的 EIP-7702 交易，将 Delegation Indicator 指向零地址（0x00...00）来清除委托，或指向一个已知的安全合约。
  - 急救合约，在调用真实合约的前面，增加一个急救合约 wrapper，可以实现快速的合约调用切断。这个合约可以由安全公司和团队进行管理，仅支持阻断交易的功能

# 2025.05.18

安全的风险还是很高的，EOA delegated to code 之后，任何人都可以发起对这个 EOA Code 的调用，可能会直接清零。所以相关系统可能需要多加注意，不能静态的计算当前 EOA 的余额信息等。

## https://docs.biconomy.io/eip7702/explained/

EIP-7702 bridges this gap by enabling EOAs to behave like smart accounts while maintaining their original addresses and established on-chain history.

## https://media.licdn.com/dms/image/v2/D4D22AQG03Xobor9mqQ/feedshare-shrink_2048_1536/B4DZaUVG9DG0Ao-/0/1746245285654?e=1749081600&v=beta&t=pmXk3YrFztt99aRZ_icekl4ENfAgiWLW0fWoc-HuCb0

需要注意的安全事项和详细的解释案例：

- Reentrancy Guard 重入攻击，不要在使用 tx.origin 进行保护
  - 重入攻击是 Contract A 调用 Contract B 的时候，没有结束之前，重新调用了 Contract A，如果 balance 设置不正确，就会被恶意抽走
  - 之前有一种方式是判断 tx.origin == msg.sender 来阻断重入攻击，因为在重入之后，Contract B 成为了 msg.sender，这是不应该出现的
  - TODO 模拟一个 EOA 调用自己的 Contract，以及 reentrancy attack 看看相关的 tx.origin 和 msg.sender 的内容

# 2025.05.19

- Safe Initialization: Avoid atomic init() calls – use initWithSig to prevent frontrunning.
  - frontrunning 就是抢跑，看到 mempool 里面的交易，用更高的 gas 去抢跑执行，所以在 init call 的时候可以抢先执行，将 initialize(ownerAddress) 等操作设置为自己
  - initWithSig 包括真正 owner 的 sig，然后合约里面进行验证
  - 在代理合约模式、工厂合约模式等等，通常使用额外的 initialize tx 来对合约进行初始化参数

TODO 搞一个 Demo，看看 7702 能不能完全 0 gas（第一笔通过 relayer 的方式）实现指向。

TODO 做一个 7702 小游戏，I need your help，跟朋友一起玩，可以生成一个求助链接，然后朋友帮忙代付 gas

# 2025.05.26

- Use EntryPoint: Require initialization to be called from ERC-4337's EntryPoint for added safety
  - EntryPoint 合约是 AA 的全局、单例智能合约，是 AA 账户的统一入口，负责验证、执行 UserOperations、Paymaster 等
  - 这一条建议要求在合约里面判断 initialization call 是从 EntryPoint 合约发出的，这样可以借用 EntryPoint 合约对安全性提供更多检查
  - TODO 做一个 demo，使用 EOA 调用 EntryPoint 创建一个 AA 账号，以及额外的安全漏洞 demo
- Avoid Storage Collisions: Switching delegation contracts can cause bugs if storage layouts conflict
  - 因为合约变量值的存储是按照 slot 位置进行的，所以切换了不同的合约，数据还在，可能产生读取敏感数据或者写入覆盖等问题
  - 最好使用命名空间、兼容布局或者清理存储等方式
- Phishing Risks: Only delegate to immutable, CREATE2-deployed contracts - never metamorphic ones
  - CREATE2 地址计算 address = keccak256(0xff + factory + salt + keccak256(initCode)) 保证了防止指向的合约代码替换
- Minimize Trusted Surface: Keep delegation contract logic minimal and auditable to avoid critical bugs
  - 使用必要的核心逻辑合约，不要用大而复杂的，聚焦在模块化的具体功能上面

一些最佳实践：

- Delegation contract should align with AA standards
- Stay Permissionless: Don't hardcode relayers; anyone should be able to relay & censorship resistance
- Pick 4337 Bundlers: Use EntryPoint >= 0.8 for gas abstraction
- dApp integration: Utilize ERC-5792 or ERC-6900, no standardized methond for dApps to request 7702 authorization signatures directly
- Avoid Lock-In: Stick to open, interoperable standards like Alchemy's Modular Account
- Preserve Privacy: Support ERC-20 gas payments, session keys, public mempools to minimize data exposure
- Use Proxies: Delegate to proxies for upgrades and modularity without requiring additional EIP-7702 authorizations for each change

# 2025.05.28

## ERC-4337 基础知识

用户发送 UserOperation 的交易，包含更多指令和附加数据等。发送到 mempool 节点上。大概结构如下：

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

mempool 里面的 bundler 会将 UserOperation 打包成真实交易。

Bundler 是一个 mempool，监听 UserOperation，将多个 UserOperation 打包成一个交易，然后提交给 EntryPoint 合约，通过抽取 gas fee 维持。

EntryPoint 是一个单例智能合约，验证和执行 UserOperation，通过 calldata 来执行，从智能合约账户中取 gas 来支付。

Paymaster 也是一个智能合约，处理 Gas 支付方式等，消除原生代币才能支付 gas 的限制，实现代付。

## https://blog.biconomy.io/a-comprehensive-eip-7702-guide-for-apps/

EIP-7702 aims to solve these user experience issues by allowing EOAs to delegate their execution to smart contracts, effectively giving them programmable capabilities without requiring users to migrate to entirely new wallets.

In simpler terms, your EOA can say: "When I receive transactions, run this smart contract code instead."

How It Works:

- A user signs a special authorization message from their EOA
- This authorization is included in a transaction
- When processed, the Ethereum network records that this EOA should delegate to a specific smart contract
- Future transactions to this EOA will execute the smart contract's code 注意，这个 tx 可以是外部发起对当前 EOA 的调用，这样跟一个 contract 没什么区别，但是调用的上下文是 EOA 的。所以如果 delegate 的合约地址有问题，就比较大了 TODO 做一个安全攻击的 example
- Importantly, the msg.sender in these transactions remains the EOA's address

This is different from a proxy contract because the delegation happens at the protocol level - there's no separate contract deployed for each user. Instead, the EOA appears to have code attached to it directly.

TODO 黑客松 Transaction Batching demo
TODO 黑客松 Gas Sponsorship demo
TODO 黑客松 Permission Management demo

几个限制：

- eoa 的 pk 还是最高权限，可以随时清理 Delegation
- eoa 的 Delegation 可以更换，所以这个不是真正的不可变合约
- 多个 chain 需要分别 sign authorization，除非 sign 的时候，chain_id 设置为 0，但是 nonce 每个 chain 是不一样的，所以还是没法用的。导致跨链一致性和互操作性是有问题的
- app 无法 control eoa 的 7702 delegation，这个是 wallet 端自己的限制，他们拒绝 app 发起的 tx 包含 auth。wallet 会让 eoa 指向自己的实现，但是这样 app 无法引导用户使用自己喜欢的 smart accounts，针对不同的 wallet 会产生不同的实现

TODO 7702 smart account marketplace，列出来不同的 AA 可以让大家选择，可以借用某种机甲风格作为概念，相当于钢铁侠的机甲或者高达，或许变成了某种游戏？比如使用不同的人物角色

其他相关的可以一起使用的 EIP、ERC：

- ERC-7710: Creates a standard interface for smart contracts to delegate permissions to other contracts. Think of it as an extension of the ERC-20 approve function but for arbitrary permissions.
- ERC-7715: Introduces a new JSON-RPC method called wallet_grantPermissions that applications can use to request permissions from wallets.
- This approach utilizes the wallet_sendCalls JSON-RPC method defined in ERC-5792

TODO https://blog.biconomy.io/a-comprehensive-eip-7702-guide-for-apps/ 这个加入黑客松

One, often overlooked, feature of EIP-7702 is that it enables developers to deploy new smart accounts for users up to 80% cheaper than before. It achieves this by leveraging the fact that an EIP-7702 proxy costs only 12,500 gas to set up. By using a few clever cryptographic tricks - such as Nick’s method and signature packing - developers can deploy fully featured smart accounts (with support for resource locks, time locks and multi sigs) - by using just EIP-7702.

TODO 使用 PREP 方法部署一个 companion account https://blog.biconomy.io/provably-rootless-eip7702-proxy/

Companion Account: An application-specific smart account that acts as an intermediary between users and applications.

对于 app dev 来说，不同的 wallet 会使用不同的实现，所以需要兼容主流的钱包的一些方法，而且不是所有 chain 都会支持，这样的话，岂不是会造成 app 端更加的混乱？

## https://hackmd.io/@colinlyguo/SyAZWMmr1x

TODO 一个 7702 权限管理平台，类似 Web3 的 Google Account，用户可以签约授权，然后方便外面的 Web2、Web3 网站使用 OIDC 来调用当前权限管理平台进行登录。统一了登录接口，然后避免用户泄露了私钥，然后支持授权。

<!-- Content_END -->
