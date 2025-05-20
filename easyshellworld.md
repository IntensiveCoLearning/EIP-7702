---
timezone: UTC+8
---

# none

无名的人阿！

>我是这路上 没名字的人
我没有新闻 没有人评论
要拼尽所有 换得普通的剧本
曲折辗转 不过谋生
我是离开 小镇上的人
是哭笑着 吃过饭的人
是赶路的人 是养家的人
是城市背景的 无声
我不过 想亲手触摸
弯过腰的每一刻
留下的 湿透的脚印 是不是值得
这哽咽 若你也相同
就是同路的朋友


坚持走完这一路

## Notes

<!-- Content_START -->

### 2025.05.14
#### EIP-7702缘起
* **why**提出要EIP-7702?
    * 传统EOAz账户只能发起交易，却无法执行合约逻辑，用户体验与智能合约钱包相去甚远。EIP‑7702 允许在单笔交易上下文中「暂时为 EOA 设置合约代码」，赋予其智能账户能力，从而简化UX。相较于此前的EIP‑3074引入新操作码导致生态碎片化，7702通过新增交易类型（SET_CODE_TX_TYPE）而非扩展 EVM 指令，兼容性更佳，不需额外调用代理合约，进一步对齐ERC‑4337路线图。
    * 兼容并衔接 ERC‑4337。EIP‑4337 已提出基于「用户操作（UserOp）」的抽象账户架构，而 EIP‑7702 可直接在现有 EOA 上执行 Account Abstraction，无需迁移新地址，并与已部署的智能合约钱包共存。
* **who**提出的EIP‑7702
    * EIP‑7702 的撰写者和提案人是 Vitalik Buterin，同时 Tin Erispe 和 Ayush Bherwani 也对该提案做出了贡献。
* EIP‑7702的核心目的
    * 智能账户能力：EOA 可临时执行合约逻辑，实现批量交易（batching）、Gas 赞助（sponsorship）、社交恢复（social recovery）等功能。
    * 简化钱包UX：无需预先持有 ETH、单签名即可完成多步操作，用户可享受类似 Web2 的流畅体验。
    * 安全可控：链 ID 绑定、Nonce 绑定、可撤销授权等多重防护，避免被永久锁定或跨链滥用。
* Pectra
    * EIP‑7702提案随 Ethereum 2025 年 5 月 7 日上线的 Pectra 升级一并激活，Pectra 是迄今为止最全面的一次硬分叉，包含共 11 项 EIP 标准，除 EIP‑7702 外，还包括企业级质押（EIP‑7251）、验证者可编程退出（EIP‑7002/6110）、Layer‑2 提效（EIP‑7691/7523）、以及若干预编译与历史哈希等安全性能优化提案。

| EIP 编号   | 名称                          | 功能简介                           |
| -------- | --------------------------- | ------------------------------ |
| EIP‑7702 | Smart Accounts for Everyone | 让 EOA 临时具备智能账户能力               |
| EIP‑7251 | Enterprise‑Grade Staking    | 单验证者质押上限从 32 ETH 提升至 2,048 ETH |
| EIP‑7002 | Execution‑Layer Exits       | 允许执行层发起验证者主动退出                 |
| EIP‑6110 | On‑Chain Deposit Handling   | 缩短验证者激活时间从 \~12h 到 \~13min     |
| EIP‑7691 | Increased Blobspace         | 区块中 Blobspace 翻倍，促进 L2 数据可用性   |
| EIP‑7523 | State Bloat Mitigation      | 清理空账户、防止无谓合约创建                 |
| EIP‑2537 | BLS Precompile              | 内建 BLS12‑381 操作，加速聚合签名         |
| EIP‑2935 | Historical Hashes           | 存储历史区块哈希，支持无状态客户端              |
| EIP‑7594 | Peer-to-Peer DAS            | P2P 数据可用性抽样，强化 L2 安全           |
| …        | …                           | 其他如安全与开发者体验优化等                 |

### 2025.05.15
#### EIP‑7702 交易数据格式
* 交易类型
```
TransactionType = 0x04（Set‑Code 交易）
```
* TransactionPayload
```
[
  chain_id,
  nonce,
  max_priority_fee_per_gas,
  max_fee_per_gas,
  gas_limit,
  destination,        // 不能为 null
  value,
  data,
  access_list,       // 同 EIP‑4844
  authorization_list,  
  signature_y_parity,
  signature_r,
  signature_s
]

```
* authorization_list
```
[ chain_id, address, nonce, y_parity, r, s ]
```

### 2025.05.16
在本地 Anvil 节点上，通过 Alloy 库发送 EIP-7702 类型的交易
```rust
use alloy::{
    eips::eip7702::Authorization,
    network::{EthereumWallet, TransactionBuilder, TransactionBuilder7702},
    node_bindings::Anvil,
    primitives::U256,
    providers::{Provider, ProviderBuilder},
    rpc::types::TransactionRequest,
    signers::{local::PrivateKeySigner, SignerSync},
    sol,
};
use eyre::Result;

// 使用 Solidity 合约代码生成 Rust 接口
sol!(
    #[allow(missing_docs)]
    #[sol(rpc, bytecode = "608080604052...")]
    contract Log {
        #[derive(Debug)]
        event Hello();
        event World();

        function emitHello() public {
            emit Hello();
        }

        function emitWorld() public {
            emit World();
        }
    }
);

#[tokio::main]
async fn main() -> Result<()> {
    // 启动本地 Anvil 节点，启用 Prague 硬分叉
    let anvil = Anvil::new().arg("--hardfork").arg("prague").try_spawn()?;

    // 创建两个用户，Alice 和 Bob
    let alice: PrivateKeySigner = anvil.keys()[0].clone().into();
    let bob: PrivateKeySigner = anvil.keys()[1].clone().into();

    // 使用 Bob 的钱包创建提供者
    let rpc_url = anvil.endpoint_url();
    let wallet = EthereumWallet::from(bob.clone());
    let provider = ProviderBuilder::new().wallet(wallet).on_http(rpc_url);

    // 部署 Alice 授权的合约
    let contract = Log::deploy(&provider).await?;

    // 创建授权对象，供 Alice 签名
    let authorization = Authorization {
        chain_id: U256::from(anvil.chain_id()),
        address: *contract.address(),
        nonce: provider.get_transaction_count(alice.address()).await?,
    };

    // Alice 对授权对象进行签名
    let signature = alice.sign_hash_sync(&authorization.signature_hash())?;
    let signed_authorization = authorization.into_signed(signature);

    // 准备调用合约的 calldata
    let call = contract.emitHello();
    let emit_hello_calldata = call.calldata().to_owned();

    // 构建交易请求
    let tx = TransactionRequest::default()
        .with_to(alice.address())
        .with_authorization_list(vec![signed_authorization])
        .with_input(emit_hello_calldata);

    // 发送交易并等待广播
    let pending_tx = provider.send_transaction(tx).await?;

    println!("Pending transaction... {}", pending_tx.tx_hash());

    // 等待交易被包含并获取收据
    let receipt = pending_tx.get_receipt().await?;

    println!(
        "Transaction included in block {}",
        receipt.block_number.expect("Failed to get block number")
    );

    assert!(receipt.status());
    assert_eq!(receipt.from, bob.address());
    assert_eq!(receipt.to, Some(alice.address()));
    assert_eq!(receipt.inner.logs().len(), 1);
    assert_eq!(receipt.inner.logs()[0].address(), alice.address());

    Ok(())
}

```

### 2025.05.17
* EIP-4337 与 EIP-7702 对比

* EIP-4337 通过高层内存池（mempool）实现账号抽象，使用 `UserOperation` 对象、Bundlers（打包者）、EntryPoint 合约和可选 Paymasters（支付代理），无需共识层更改。
* EIP-7702 通过新增 EIP-2718 “Set Code Transaction” 类型，使外部账户（EOA）在交易执行期间临时注入智能合约代码，交易后即丢弃代码。
* 在 EIP-4337 中，智能合约账户为永久部署的合约，存储自有代码和状态；而 EIP-7702 的代码注入仅在交易生命周期内有效，之后恢复为普通 EOA。
* EIP-4337 完全兼容现有 EVM 行为，只实现用户态内存池，无需共识层变更；EIP-7702 则需通过硬分叉（如 Pectra）在协议层新增交易类型和授权元组解读。
* EIP-4337 适合通过工厂合约离链创建智能合约钱包，并使用 Paymasters 赞助 Gas；而 EIP-7702 使传统 EOA（热/冷钱包）无需部署独立合约即可获得智能钱包特性。
* 两种标准都支持 Gas 抽象，但 EIP-4337 通过 Paymaster 合约正式化 Gas 赞助（接受 ETH 或 ERC-20 费用），而 EIP-7702 关注通过交易内代码注入提高 Gas 效率。
* 安全模型不同：EIP-4337 已成熟，提供多种插件（如多签、限额、社交恢复）及工厂合约审计模式；EIP-7702 则需确保授权元组安全，防止前置攻击、存储冲突和未授权升级。

* 支持的智能合约库

* **eth‑infinitism/account‑abstraction**：EIP-4337 官方参考实现，包含 EntryPoint、StakeManager、ValidationLogic 等核心合约，用于接收并执行 `UserOperation` 对象，并管理账户创建和操作验证。
* **@mev‑boost‑aa/contracts**：针对 MEV Boost 的 ERC-4337 智能合约套件，提供扩展 EntryPoint、Paymaster 和跨链 MEV 收益提取逻辑。
* **@tisaacontracts/contracts**：一组针对 EIP-4337 的 Solidity 合约，包括 Wallet 实现、Factory、EntryPoint 接口和基础 Paymaster 模板。
* **Biconomy SCW Contracts（bcnmy/scw‑contracts）**：Biconomy 智能合约钱包实现，包含 BaseSmartAccount.sol、Proxy.sol、SmartAccountFactory.sol、EntryPoint.sol，支持 ERC-4337 与 ERC-6900。
* **bcnmy/biconomy‑paymasters**：示例级 Paymaster 合约合集，涵盖验证、预付费和自定义资费规则，用于在链上为用户操作赞助 Gas，支持 ERC-20 计费和订阅模式。
* **Mirror‑Tang/Account‑abstraction‑coding‑security‑library**：聚焦 EIP-4337 安全性边界的测试库，提供多种攻击场景测试合约和常见漏洞检出脚本。
* **4337Mafia/awesome‑account‑abstraction**：精选资源仓库，包含多种 Solidity 合约实现（如 EIP-4337、EIP-2718、EIP-6900），并链接至各项目源码。
* **OpenZeppelin EIP‑4337 审计代码**：尽管不是独立库，但 OpenZeppelin 在其博客中审计并引用了 eth‑infinitism 合约，尤其是 EntryPoint 和 StakeManager。

* 支持的 SDK

* **@account‑abstraction（UserOp.js）**：EIP-4337 官方 JS 库，用于构建、模拟和发送 `UserOperation` 对象，与 EntryPoint 合约交互。
* **ZeroDev SDK**：支持 EIP-4337 和 EIP-7702，提供智能账户创建、Gas 赞助、批处理交易、代理调用和账户迁移等高级 API。
* **Biconomy SDK**：集成 EIP-4337 Paymasters 和 UserOperations，简化免 Gas 交易、元交易和多签流程的 dApp 接入。
* **Etherspot SDK**：TypeScript 库，简化 EIP-4337 打包、Paymaster 集成、交易赞助及跨链账号抽象。
* **Safe Core SDK（Gnosis Safe）**：JavaScript 库，部署和管理兼容 EIP-4337 的模块化智能账户，内置多签、会话密钥和安全策略插件。
* **Alchemy AA SDK v3.0**：全功能账户抽象 SDK，支持 EIP-4337 和 EIP-6900，提供会话密钥插件、电邮/通行证认证及 viem 客户端集成。
* **Stackup SDK（userop.js）**：开发者工具包，用于生成和打包 `UserOperation` 对象，与 EntryPoint 和 Paymaster 合约交互，并通过 Stackup 基础设施监控操作状态。
* **Argent & Argent X SDKs**：钱包库，内置 EIP-4337 账号抽象，支持社交恢复、每日限额、Gas 赞助及模块化安全功能。


### 2025.05.18
* **`authorizationList` 是什么**  
  * EIP‑7702 新增交易类型（类型号 `0x04`）的特有字段，用于承载离线签名的授权信息。  
  * 该列表至少包含一个元组，每个元组代表一次授权；若列表为空或元组格式不符合规范，整个交易将被视为无效。  

* **字段详解**  
  * **`chain_id`**  
    * 指定签名时使用的链 ID，用于防止重放攻击。  
    * 在执行交易时，节点会校验元组中的 `chain_id` 与交易对象中的 `chainId` 字段是否一致。  

  * **`address`**  
    * 代表希望注入并执行的合约部署地址，也称为“Delegate 合约”地址。  
    * 该地址必须对应一个已部署的合约，并且节点会在执行前将此地址写入 EOA 的代码槽中。  

  * **`nonce`**  
    * 用于标识该授权的顺序，防止同一授权被重复使用。  
    * 与普通交易的 `nonce` 类似，但独立于 EOA 的交易 `nonce`，专门用于授权管理。  

  * **签名部分：`y_parity`、`r`、`s`**  
    * 三者共同构成 ECDSA 签名的分离格式：  
      * `r`, `s`：签名的两大数值部分。  
      * `y_parity`：恢复签名时的恢复位（v 的奇偶性），用于精确还原签名者地址。  
    * 通过 `ecrecover` 恢复出签名者地址，确保只有合法 EOA 所有者的授权才会生效。  

* **验证流程**  
  * **格式校验**：节点检查 `authorizationList` 非空，且每个元组包含恰好 6 个元素。  
  * **边界检查**：元组中数值字段（如 `chain_id`, `nonce`, `y_parity`, `r`, `s`）需在 EIP‑7702 规定的范围内。  
  * **签名验证**：节点使用 `ecrecover` 对 `(chain_id, address, nonce)` 进行哈希运算并验证签名，恢复出的地址必须与交易目标 EOA 一致。  
  * **代码注入**：验证通过后，节点将按照 `0xef01 00 || address` 的格式，将合约地址写入 EOA 的代码槽中，使其在后续操作中表现如同合约账户。  
  * **执行与恢复**：交易执行完成后，节点恢复 EOA 原始代码槽（通常为空），保证账户状态不发生永久改变。  

* **错误情况与边界**  
  * **空列表**：`authorizationList` 长度为零时，交易直接失效。  
  * **签名不匹配**：若 `ecrecover` 恢复的地址与目标 EOA 不符，视为无效授权。  
  * **格式不符**：元组元素数量非 6 或字段类型错误均会导致交易失败。  

* **实践建议**  
  * **使用可靠库**：优先采用成熟工具（如 ethers.js v6+ 的 `signAuthorization`、`prepareTransaction`）来构建和管理 `authorizationList`，以避免编码错误。  
  * **严格审计**：仅对已审计的 Delegate 合约地址进行授权，防范恶意或漏洞合约注入风险。  
  * **权限管理**：对于多重签名场景，合理设置 `nonce` 和签名顺序，防止重放攻击或授权滥用。  

```
import { JsonRpcProvider } from "ethers";

const provider = new JsonRpcProvider("https://sepolia.infura.io/v3/YOUR_KEY");
const relayer = new Wallet(RELAYER_KEY, provider);

// 查询 nonce
const txCount = await provider.getTransactionCount(signer.address);  // 用户 EOA 的 nonce :contentReference[oaicite:4]{index=4}

// 构造 tx
const tx = {
  type: 0x04,
  chainId: chainId,
  nonce: txCount,
  maxPriorityFeePerGas: utils.parseUnits("2", "gwei"),
  maxFeePerGas: utils.parseUnits("50", "gwei"),
  gasLimit: 200000,
  to: signer.address,          // 通常目标设为自身，EIP‑7702 处理后会执行委托
  value: 0,
  data: "0x",                  // 可留空或携带调用 payload
  accessList: [],              // EIP‑2930 访问列表，可选
  authorizationList: [
    [chainId, delegateAddr, authNonce, yParity, r, s]
  ],                            // 我们生成的授权列表
};

// 通过第三方 Relayer（付 gas）发送
const receipt = await relayer.sendTransaction(tx);
console.log("交易哈希:", receipt.hash);
```
### 2025.05.19
* **摘要**：Viem 对 EIP‑7702 提供了原生支持，包括在文档与 API 中专门的扩展，使您能够生成并管理授权元组 `(chain_id, contract_address, nonce, y_parity, r, s)`，无需手动构造，即可在链上调用中使用授权列表。

* **EIP‑7702 扩展位置**：

  * 在官方文档的 “EIP‑7702” 专栏（版本 2.29.4）中详细介绍了该提案及其在 Viem 中的实现。
  * EIP‑7702 本身定义了一种新的以太坊交易类型（`0x04`），允许 EOA 将执行权限委托给智能合约，实现批量操作、燃气赞助以及权限委派。

* **核心动作（Actions）**：

  * **prepareAuthorization**：准备授权数据的哈希输入。
  * **signAuthorization**：签名授权并返回已封装的授权元组，直接可用于 `authorizationList`。

* **辅助工具（Utilities）**：

  * **hashAuthorization**：对授权数据进行哈希计算。
  * **recoverAuthorizationAddress**：从授权签名中恢复签名者地址。
  * **verifyAuthorization**：验证授权签名的合法性。

* **集成指南**：

  * **合约调用示例**：展示如何将 `authorizationList` 传递给客户端的 `execute`（或等效）方法，以在 EIP‑7702 上下文中调用合约函数。
  * **交易发送示例**：

    ```ts
    import { encodeFunctionData } from 'viem'
    import { privateKeyToAccount } from 'viem/accounts'
    import { walletClient } from './config'
    import { abi, contractAddress } from './contract'

    const eoa = privateKeyToAccount('0x...')
    const authorization = await walletClient.signAuthorization({
      account: eoa,
      contractAddress,
    })
    const hash = await walletClient.sendTransaction({
      to: eoa.address,
      data: encodeFunctionData({ abi, functionName: 'initialize' }),
      authorizationList: [authorization],
    })
    ```

    提交一个 EIP‑7702 交易，使 EOA 委托合约执行初始化逻辑。

* **批量与赞助示例**：

  * 准备多个授权以实现原子化多步流程（如 ERC‑20 批准与转账），并可由中继者代付燃气：

    ```ts
    const auth1 = await client.signAuthorization({ contractAddress: A })
    const auth2 = await client.signAuthorization({ contractAddress: B })
    const txHash = await client.execute({
      address: eoa.address,
      batches: [ /* ... */ ],
      authorizationList: [auth1, auth2],
    })
    ```

    可将多个操作捆绑为一个事务，同时保持私钥离线。

* **生态系统集成**：

  * **Wagmi**：已将 Viem 的 EIP‑7702 支持上游合并，提供 React Hooks。
  * **Foundry & QuickNode**：示例代码展示如何在本地分叉及主网中通过 Viem 客户端测试 EIP‑7702。

  ### 2025.05.20
  ![测试中](./images//none/1.PNG)








<!-- Content_END -->
