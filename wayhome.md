---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# wayhome

1. 自我介绍

还在学习中的 web3 开发者，目前刚学了一些 uniswap 和 solana 的开发，希望能通过此次共学对以太坊了解更深入一些。  

2. 你认为你会完成本次残酷学习吗？

会

3. 你的联系方式（推荐 Telegram）

t.me/wayhome

## Notes

<!-- Content_START -->

### 2025.05.23

## 其他相关提案

- 账户抽象化这个概念是在 2017 年2月 Vitalik Buterin 的 EIP-86 里首次提出。
- EIP-2938 提出的账户抽象化（Account Abstraction, AA）是一个可以让合约账户成为和外部账户一样的 “顶层” 账户的提案
- EIP-3074 主要的技术创新是向 以太坊虚拟机（EVM） 添加两个新的操作码：AUTH 和 AUTHCALL，它们旨在被称为“调用者”的智能合约使用。这些调用者获得了对授权他们的外部账户 EOA 的控制权，并可以代表他们进行调用
- ERC-4337  前面已讲



### 2025.05.22

## EIP-7702 安全影响 (https://x.com/threesigmaxyz/status/1924782348601872878)

最显著的安全影响是打破了 EOA 无法执行代码的假设。这一变化打破了基本的不变量：条件 tx.origin == msg.sender 不再保证当前执行上下文是由 EOA 直接发起的，而没有经过同一交易中的中间合约调用。这带来了间接的安全后果:

- 错误合同检测: 许多合约使用 extcodesize 或 code.length 来判断一个地址是否是合约。通过 EIP-7702，EOA 技术上拥有字节码（委托指示器），所以这些检查可能会错误地将 EOA 识别为合约，从而可能以意外的方式改变合约行为
- 接收 token 时出现意外的回滚： 此前，向 EOA 发送 ETH 或代币非常简单，因为 EOA 无法拒绝转账。通过 EIP-7702，如果 EOA 已委托代码，转账可能会触发委托合约的回退函数，这可能会消耗额外的 gas，甚至导致交易回滚。如果委托合约未实现 receive 或回退函数，原生转账将会回滚。
类似地，像 ERC-721（NFT）和 ERC-777 这样的代币标准尝试调用合约地址上的接收者钩子函数。委托的 EOA 应实现这些钩子，如果委托合约不支持它们，转账将会失败
- 破坏闪电贷保护： 合约有时使用 require(tx.origin == msg.sender)来防止中间合约在同一笔交易中操纵状态，以此防御闪电贷攻击。通过 EIP-7702，攻击者的 EOA 可以委托给合约，通过闪电贷借入资金，并且仍然可以通过此检查，因为 msg.sender 和 tx.origin 仍然是攻击者的 EOA 地址

## 透过EIP-7702看过去与未来 

- https://zhuanlan.zhihu.com/p/1905556549998453568

### 2025.05.21

## 账户抽象 (Account Abstraction)

核心概念是解耦用户身份与交易执行，传统账户模型用户必须持有私钥才能发起交易，而账户抽象后用户可通过多种方式验证身份并发起交易，从而实现更灵活安全易用的交易方式。

## Fatlane
一个支持 ERC-4337 的基础设施，提供 bundling 服务。通过聚合多个用户操作，提高交易效率并降低成本。

## MEV
MEV（Maximal Extractable Value）指的是从去中心化系统中可以提取的最大价值。这包括从交易排序和插入中获得的套利、三明治攻击和抢尾交易等利润，以及Gas 费用本身。MEV 在很大程度上是用户“意图”实现“最优执行”的对立面，可以看作是用户意图执行的一种偏差或税收.

在账户抽象和 ERC-4337 的框架下，MEV 成为了一个重要的考量因素。ERC-4337 通过引入一个替代内存池（Alt Mempool）实现了无需协议层修改的账户抽象。用户通过 UserOperation 对象来表达他们的交易意图。
这些 UserOperation 会被**捆绑者（Bundlers，这里指 ERC-4337 标准中的 Bundler，区别于传统 MEV 供应链中的 Bundler/搜索者）**收集并提交到以太坊网络，通常通过 EntryPoint 合约。

ERC-4337 的替代内存池设计旨在最大限度地提高许可性（permissionlessness）去中心化（decentralization），这对交易的活跃度和抗审查性很有利。
多个潜在的捆绑者会检测到 UserOperation。捆绑者是理性的参与者，他们会竞争处理这些 UserOperation，并根据能从中提取的价值来决定向验证者支付多少费用

## ERC-4337 Frontrunning 

为什么会发生 ERC-4337 Frontrunning？

1.UserOperations 的结构包含支付信息：UserOperation 结构（在版本 0.6 中）包含 maxFeePerGas 和 maxPriorityFeePerGas 等字段，用于指示中继操作需要支付的金额。
2.UserOperations 可能过度支付：为了确保交易在链上拥堵时不会停滞，UserOperations 通常会支付相对较高的 Gas 费用。例如，一个捆绑者可能支付了 0.0048 POL 的 Gas 费用，但从 UserOperation 中获得了 0.0091 POL 的返还，几乎是支付金额的两倍。
3.存在套利空间：UserOperation 中存在的过度支付创造了利润机会。当捆绑者支付的 Gas 费用低于 UserOperation 中的返还金额时，这种差额就成为了可提取的价值。

## ERC-4337 Frontrunning 的工作原理
其基本策略是找到那些以低于返还金额的成本进行中继的 UserOperations。然后，搜索者（在 MEV 领域被称为 searcher，不同于 ERC-4337 标准中的 Bundler）会以略高的 Gas 价格中继同一个 UserOperation。
如果搜索者的交易先于原始捆绑者的交易执行，搜索者将获得 UserOperation 的返还金额，从而赚取成本与返还之间的差价。如果后执行，交易将回滚，导致损失。
实现这一策略需要监控链上发送的 4337 Bundle，并能够查看待处理的交易。

## Frontrunning 机器人的挑战

1. 监控内存池：需要低延迟地获取待处理的 4337 交易流。作者最初利用 BlockNative Explorer 泄露的 API 密钥，后尝试 Bitquery 和 Alchemy 的 WebSocket 流但延迟较高。最终通过运行自己的节点（基于 reth）解决了低延迟问题。
2. 快速高效地发送交易：为了提高交易在同一区块中被包含的概率，需要尽快将 Frontrun 交易广播到尽可能多的 RPC。Merkle 的交易注入端点被认为是有效的秘密武器。
3. 管理 Nonce：交易 Nonce 需要连续且单调递增。快速发送多个交易可能导致网络未及时传播前一个 Nonce 的交易，导致后续交易被延迟或拒绝。作者通过轮换使用多个地址来增加单个地址的交易间隔，并使用 0 ETH 的自我转账交易来“解堵”被 Gas Price 急剧上涨卡死的地址。
4. 激烈的竞争：存在其他 Searcher 也在进行 ERC-4337 Frontrunning，导致激烈的竞价战。竞争对手可能拥有更优越的网络覆盖和快速反应能力。
   作者描述了与一个拥有出色执行能力的 Searcher B 的竞价过程。当作者提高 Gas Price 竞标时，Searcher B 能够迅速以更高价格替换交易或通过取消交易后重新发送来出价更高。
5. 对手的适应性：当作者尝试通过代理合约或改变函数选择器来隐藏自己的活动时，Searcher B 很快就适应了这些变化，表明他们可能正在深度分析所有交易的调用轨迹或监控作者的地址。作者最终承认在竞争中落败

### 2025.05.20

## ERC-4337 核心组件

- UserOperations： 代表用户想要执行的操作，可以包含一个或多个动作。它具有类似于传统以太坊交易的字段结构，但包含特定于 ERC-4337 的逻辑。
   
   可以被视为用户发送给他们的智能合约账户的“待办事项列表”。它包含了用户希望账户执行的一系列动作，例如转移资金、与智能合约交互或执行社交恢复。与传统以太坊交易不同，用户操作可以打包多个步骤，并在一个单一的操作中提交给 ERC-4337 系统进行处理。

- Bundlers： 负责将用户操作收集并打包成一个交易，然后提交给以太坊网络上的 EntryPoint 合约。捆绑者通常是验证者或 MEV 搜索者，并因处理用户操作而获得激励

   “捆绑者”（Bundlers）充当 ERC-4337 系统中的协调者。他们监控一个备用内存池（“Alt Mempool”），收集用户创建的 UserOperations。
   然后，他们将这些 UserOperations 打包成一个单一的交易，并将其提交给 EntryPoint 智能合约进行处理。
   捆绑者因其服务而获得激励，这确保了用户操作能够被提交到网络并得到执行。

- EntryPoint： 一个单一的智能合约，充当用户操作的网关。它负责验证、打包的用户操作，并协调这些操作在相应的智能合约账户中的执行。它也可以回滚失败的操作。

   “入口点”（EntryPoint）合约是 ERC-4337 架构的核心。它是一个所有捆绑者都交互的单点智能合约。
   EntryPoint 负责接收来自捆绑者的打包用户操作，验证这些操作，并协调在相应的智能合约账户中执行它们。它还负责处理 Gas 支付，并在操作失败时回滚事务。

- Contract Accounts： 用户控制的智能合约账户，它们可以接收和执行用户操作中定义的指令，并管理资产

- Paymaster (可选)： 一个可选的智能合约，可以代表用户支付交易费用（即赞助交易）

   ERC-4337 通过引入“支付者”（Paymaster）组件实现了第三方支付交易费用。支付者是一个可选的智能合约，它可以选择为用户的用户操作支付 Gas 费用。
   用户操作可以包含支付者合约的地址，该合约将同意偿还捆绑者处理该用户操作所需的 Gas 费用。
   这种机制使得应用程序、服务或甚至个人能够为用户赞助交易，从而提高用户体验，特别是对于不熟悉加密货币或不想直接处理 Gas 费用的用户。


- Aggregators (可选)： 一个可选的智能合约，帮助验证来自多个用户操作的签名


### 2025.05.19

## 与 ERC-4337 的兼容性

- EIP-7702 和 ERC-4337 都旨在推进账户抽象（Account Abstraction, AA），使外部拥有账户（EOA）能够拥有类似智能合约的功能
- EIP-7702 引入了一种新的交易类型 (0x04)，允许 EOA 执行临时的智能合约功能。这有效地将 EOAs 升级，使其功能更像智能合约钱包
- EIP-7702 被设计为与 ERC-4337 生态系统兼容且可组合。这种可组合性使得智能 EOAs 能够无缝集成到围绕 ERC-4337 构建的现有智能合约钱包和基础设施中
- 通过 EIP-7702 委托的地址可以直接指向现有的 ERC-4337 钱包代码
- 在 ERC-4337 基础设施中（例如通过 Bundler 和 EntryPoint 合约处理 UserOperation），启用了 EIP-7702 的 EOA 可以充当 UserOperation 的 sender 字段
- EIP-7702 被视为 EOA 轻松迁移到智能钱包的一种途径，同时保留 EOA 的属性并增加“超能力”功能（如批量交易、Gas 抽象、更新灵活性）
- 尽管兼容，但 EIP-7702 并不能完全取代 ERC-4337。它们解决的是不同的方面，并且可以共存
- EIP-7702 可以促进向未来的原生 AA 标准（如 RIP-7560，它包含了 ERC-4337 工具）的过渡

### 2025.05.18

## 安全注意事项

- Reentrancy Guard： 不要依赖 tx.origin 来防止重入攻击。

- Safe Initialization： 避免使用原子性 init() 调用；使用 initWithSig 以防止抢跑（frontrunning）。

- Use EntryPoint： 要求初始化必须通过 ERC-4337 的 EntryPoint 来调用，以增加安全性。

- Avoid Storage Collisions： 切换 delegation 合同时可能会因存储结构冲突而导致 bug。

- Phishing Risks： 只允许委托到不可变的、使用 CREATE2 部署的合约 —— 不要使用具有变形能力（metamorphic）的合约。

- Minimize Trusted Surface： 委托合约应尽量简洁且易于审计，以减少关键漏洞。

- Use Audited Contracts： 优先使用知名团队（如 Alchemy、Ambire、MetaMask、EF AA team）审计过的合约。

## 最佳实践

- Delegation Contract： 委托合约应遵循账户抽象（AA）标准，兼容 ERC-4337。

- Stay Permissionless： 不要硬编码 relayers；任何人都应该能转发交易，以实现抗审查性。

- Pick 4337 Bundlers： 使用版本 ≥ 0.8 的 EntryPoint 进行 gas 抽象处理。

- dApp Integration： 使用 ERC-5792 或 ERC-6900 与 dApp 集成。目前尚无标准方法供 dApps 直接请求 7702 授权签名。

- Avoid Lock-In： 应坚持使用开放、可互操作的标准，如 Alchemy 的 Modular Account。

- Preserve Privacy： 支持 ERC-20 gas 支付、session key、公有 mempools，以最大限度减少数据泄露。

- Use Proxies： 使用代理进行升级和模块化，无需为每次更改都重新授权 EIP-7702。

## 重要限制

- EOA的私钥依然至高无上的权能：私钥始终可以通过签署新交易来覆盖任何授权。这意味着无法实现真正的多重签名或时间锁功能。
- 没有部署的持久性：与拥有自己地址的已部署智能合约账户不同，EIP-7702的委托可以被覆盖。这意味着EOA仍然在本质上是一个具有智能功能的EOA，而不是一个真正的智能账户。
- 多链问题：默认情况下，EIP-7702的授权是链特定的，这意味着用户需要为每个链签署单独的授权。有一种变通方法，用户可以签署一个chain_id 设置为0的授权，这将在所有链上有效。然而，只有当用户的EOA在所有链上具有相同的nonce时，这种方法才有效，而在实践中这很少发生。这个限制可能导致在多链工作时出现同步问题。


### 2025.05.17

## 测试 EIP-7702 交易

1. 编写合约

- 批量处理执行逻辑

```solidity
/// @notice 代表批量调用中的单个调用。
struct Call {
    address to;
    uint256 value;
    bytes data;
}

/**
     * @notice 直接执行一批调用。
     * @dev 此函数旨在供智能账户本身（即 address(this)）调用合约时使用。它检查 msg.sender 是否为合约本身。
     * @param calls 包含目标地址、ETH 值和 calldata 的 Call 结构体数组。
     */
function execute(Call[] calldata calls) external payable {
    require(msg.sender == address(this), "Invalid authority");
    _executeBatch(calls);
}


/**
 * @notice 内部函数，用于执行批量调用。
 * 合约使用一个随机数（nonce）来防止重放攻击。每次成功执行后，随机数（nonce）都会递增。如果没有实现随机数
 *（nonce），攻击者可能会多次重放同一笔交易
 */
function _executeBatch(Call[] calldata calls) internal {
    uint256 currentNonce = nonce;
    nonce++;

    for (uint256 i = 0; i < calls.length; i++) {
        _executeCall(calls[i]);
    }

    emit BatchExecuted(currentNonce, calls);
}


    /**
     * @dev 内部函数，用于执行单个调用。
     * @param callItem 包含目标地址、价值和调用数据的 Call 结构体。
     */
function _executeCall(Call calldata callItem) internal {
    (bool success,) = callItem.to.call{value: callItem.value}(callItem.data);
    require(success, "Call reverted");
    emit CallExecuted(msg.sender, callItem.to, callItem.value, callItem.data);
}
```

- 签名验证逻辑

```solidity
bytes32 digest = keccak256(abi.encodePacked(nonce, encodedCalls));
require(ECDSA.recover(digest, signature) == msg.sender, "Invalid signature");
```

- 直接执行或赞助执行

```solidity
function execute(Call[] calldata calls) external payable {
    // The caller executes the calls directly
}
function execute(Call[] calldata calls, bytes calldata signature) external payable {
    // A sponsor executes the calls on behalf of the caller
}

```

2. 运行本地网络

```bash
anvil --hardfork prague
```

3.构建合约

```bash
forge install && forge build
```

4. 运行部署脚本

```bash
forge script ./script/BatchCallAndSponsor.s.sol --tc BatchCallAndSponsorScript --broadcast --rpc-url 127.0.0.1:8545
```

输出如下:

```bash
Chain 31337

Estimated gas price: 2.000000001 gwei

Estimated total gas used for script: 2718876

Estimated amount required: 0.005437752002718876 ETH

==========================

##### anvil-hardhat
✅  [Success] Hash: 0xf0c214e5c056c6815c8ba0df3b810ebc9a2496e48616ffd484455849b26e4fb1
Contract Address: 0x8464135c8F25Da09e49BC8782676a84730C318bC
Block: 1
Paid: 0.001049548001049548 ETH (1049548 gas * 1.000000001 gwei)


##### anvil-hardhat
✅  [Success] Hash: 0x6436bc2bfab0f3a1dc501f71963b79d41223db5c24d3a4db2fe2c3635e3d221d
Contract Address: 0x71C95911E9a5D330f4D621842EC243EE1343292e
Block: 2
Paid: 0.000810267154290925 ETH (916855 gas * 0.883746235 gwei)


##### anvil-hardhat
✅  [Success] Hash: 0x7d15797f21602e8b76d9210df673ee68bdc5850304553264f428c41f90086c86
Block: 3
Paid: 0.000053771380665105 ETH (68935 gas * 0.780030183 gwei)


##### anvil-hardhat
✅  [Success] Hash: 0x652de80051db83a07e0f64c03eddc01f51fb5a5a9aa3994aa15b98ce43bf7d4b
Block: 4
Paid: 0.000033296373116512 ETH (48752 gas * 0.682974506 gwei)

✅ Sequence #1 on anvil-hardhat | Total Paid: 0.00194688290912209 ETH (2084090 gas * avg 0.836687731 gwei)


==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
```



### 2025.05.16

交易流程:

```mermaid
sequenceDiagram
    participant 钱包客户端
    participant 智能账户
    participant 实现合约
    participant 赞助者（可选）

    Note over 钱包客户端: 步骤 1：生成授权签名
    钱包客户端->>钱包客户端: signAuthorization({ contractAddress })
    钱包客户端-->>智能账户: 针对实现合约字节码的签名授权

    Note over 智能账户: 步骤 2：临时分配字节码
    智能账户->>实现合约: 使用实现合约字节码
    智能账户-->>智能账户: 临时升级为智能合约

    Note over 智能账户: 步骤 3：构造交易
    智能账户->>智能账户: 构建交易
    智能账户->>智能账户: to = 智能账户地址
    智能账户->>智能账户: data = encodeFunctionData('execute', [calls])

    Note over 智能账户: 步骤 4：执行交易
    alt 直接执行
        智能账户->>智能账户: 发送包含授权列表的交易
    else 赞助执行
        赞助者（可选）->>赞助者（可选）: 使用相同的签名授权
        赞助者（可选）->>智能账户: 赞助者代表智能账户发送交易
    end

    Note over 智能账户: 步骤 5：恢复为普通 EOA
    智能账户->>智能账户: 恢复为原始 EOA 状态
  ```

### 2025.05.15

## 协议实现细节

1. EIP-7702 协议引入了一种新的交易类型，标识符为 0x04，称为 SET_CODE_TX_TYPE

2. 交易载荷 (TransactionPayload) EIP-7702 交易的载荷结构定义为 RLP 编码的一系列字段，包括 chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, destination, value, data, access_list, authorization_list, signature_y_parity, signature_r, signature_s

3. 授权列表 (authorization_list) EIP-7702 交易新增了一个 authorization_list 字段。这是一个列表类型，可以包含多个授权条目（tuple）。authorization_list 必须非空，确保每笔 EIP-7702 交易都明确指定了要授权的实现地址

4. 授权条目结构 authorization_list 中的每个授权条目包含六个字段：chain_id, address, nonce, y_parity, r, s。

  - chain_id: 指示此授权委托生效的链 ID。如果设置为 0，表示该签名允许在所有支持 EIP-7702 的链上进行重放，前提是账户的 Nonce 也匹配。
  - address: 将 EOA 委托给的目标智能合约地址。这代表委托的代码的地址。
  - nonce: 授权用户的 Nonce。包含此字段是为了避免重放问题。
  - y_parity, r, s: 委托用户对上述三个字段 (chain_id, address, nonce) 签名后获得的签名数据。这些类似于传统的 VRS 数据。

5. 授权签名过程 授权者（即 EOA）在签署授权数据时，需要先将 chain_id, address, nonce 进行 RLP 编码。然后，使用编码后的数据和魔术数字 0x05 进行 Keccak256 哈希运算。最后，使用授权者的私钥对该哈希数据进行签名，得到 y_parity, r, s。魔术数字 0x05 的作用是预分隔符，确保不同签名类型的结果不会冲突

6. 交易发起者与授权者 EIP-7702 交易的发起者和授权者可以不同。这使得实现交易赞助（由第三方支付 Gas 费用）成为可能

7. 交易预检 (preCheck) 交易被包含在区块中执行之前，区块提议者会先进行预检。
   - 检查交易的 destination 地址（即 to 字段）不能为空。EIP-7702 交易不能用于创建合约。
   - 检查 authorization_list 必须至少包含一个条目

8. 授权条目处理 在交易执行过程中，节点会遍历 authorization_list 中的每个授权条目并进行处理

  - 对于同一个授权用户签名的多个授权条目，只有最后一个有效条目会生效，前面的条目会被覆盖。
  - 处理过程包括验证授权条目（验证链 ID、Nonce、签名、恢复签名者地址）、将签名者地址添加到 accessed_addresses（EIP-2929 的概念），并验证签名者账户的代码是否为空或已委托，以及 Nonce 是否匹配。
  - 验证成功后，会增加签名者的 Nonce。
  - 设置账户代码：如果授权条目验证成功，授权者账户的代码会被设置为委托指示符：0xef0100 || address。其中 address 是授权条目中的目标智能合约地址。如果 address 为 0x00...00，则清除账户代码。
  - 委托指示符使用 EIP-3541 中禁止的 0xef 操作码作为前缀，确保与现有合约代码不冲突。0xef0100 的格式是为了与 EIP-3540 预留的 0xef00 区分，并为未来的 EIP-7702 升级预留空间。
  - 如果任何授权条目的检查失败，只会跳过该条目，不会导致整个交易失败。这有助于防止批量授权场景下的拒绝服务攻击.

9. Nonce 处理 EIP-7702 引入了新的 Nonce 增量路径

  - 交易 Nonce：首先验证交易 Nonce 是否与发起者的账户 Nonce 匹配，然后增加交易发起者的 Nonce。
  - 授权 Nonce：对于 authorization_list 中的每个授权条目，验证其 nonce 是否与签名账户的当前 Nonce 匹配。验证成功后，增加该签名账户的 Nonce.
  - 这意味着如果交易发起者和授权者是同一用户，他们的 Nonce 会在同一笔交易中增加两次（一次作为交易发起者，一次作为授权者）。由于节点先检查交易 Nonce，后检查授权者 Nonce，理论上授权者的 Nonce 会比交易 Nonce 多一

10. 代码执行

  - 一旦 EOA 的代码被设置为委托指示符，诸如 CALL, CALLCODE, DELEGATECALL, STATICCALL 等代码执行操作码会加载并执行由委托指示符指向的委托代码地址中的代码，并在智能 EOA 的上下文中执行。
  - 无效的委托（例如，指向没有代码的地址或预编译合约地址）将被忽略，并被视为没有代码。
  - CODESIZE 和 CODECOPY 操作码在委托执行期间返回的是委托代码的大小和内容，而不是委托指示符本身。而 EXTCODESIZE 和 EXTCODECOPY 等外部代码检查操作则作用于委托指示符本身，而不是遵循委托。
  - EIP-7702 禁止递归委托（即委托指示符指向另一个委托），节点只会遵循一级委托

11. Gas 成本

  - 新的交易类型的内在 Gas 成本基于 EIP-2930 计算，并额外增加了 PER_EMPTY_ACCOUNT_COST 乘以 authorization_list 的长度。PER_EMPTY_ACCOUNT_COST 是处理每个授权条目的成本，估计约为 12500 Gas。
  - 交易发起者支付所有授权条目的 Gas 费用，无论这些条目是否有效或重复。
  - 处理授权列表时，如果授权账户在状态中已经存在（即之前已委托过），会部分退还 Gas。这种先收取最大成本后退还的机制是为了避免在计算内在 Gas 时进行状态查找。
  - 代码读取指令访问冷账户（EIP-2929 概念）时，如果需要解析委托代码，会产生额外的 Gas 成本。

12. 与智能 EOA 的交互 与智能合约类似，通过将交易的 to 字段（即 destination）设置为智能 EOA 的地址，即可与其进行交互。

13. 绕过 EIP-3607 智能 EOA（具有有效委托指示符代码的 EOA）可以绕过 EIP-3607 的限制，允许其作为 tx.origin 发起交易。这被称为自我赞助，允许用户不依赖第三方基础设施使用 EIP-7702 的功能。

14. 存储 EIP-7702 委托不会清除现有存储。智能 EOA 的状态数据是存储在其自己的账户地址下的。这可能导致在重新委托到不同合约时发生存储冲突的风险。新的提案如 ERC-7201 提出了 namespace 标准来缓解此问题。用户在重新委托前应谨慎处理，或考虑使用专门的合约来清除旧的存储数据。

15. 构造函数 当委托代码到 EOA 时，委托目标合约的构造函数不会在 EOA 的上下文中执行。如果需要初始化数据，应考虑使用初始化模式，并在同一笔交易中完成委托和初始化调用，以避免抢跑风险.

这些实现细节展示了 EIP-7702 如何在协议层面赋予 EOA 新功能，同时也引入了一些需要开发者和用户注意的安全考虑和实践指南

### 2025.05.14

## EIP-7702 核心概念和目标

EIP-7702 是一项旨在改进以太坊用户交互方式的提案，特别关注增强外部拥有账户 (EOA) 的功能。其核心目标是模糊 EOA 和智能合约账户之间的界限，赋予 EOA 可编程性和可组合性

## 主要机制

- 批量处理
  
   一次交易，完成多项操作，告别重复确认

- 赞助交易

  他人代付Gas费，降低交易成本，惠及用户

- 权限降级

  精细化权限管理，保障账户安全


<!-- Content_END -->
