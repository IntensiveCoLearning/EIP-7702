---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区

# chesley666

1. 自我介绍：web2测试工程师转web3
2. 你认为你会完成本次残酷学习吗？会
3. 你的联系方式（推荐 Telegram）：@rorozn_OvQ_

## Notes

<!-- Content_START -->

### 2025.05.14  
1. 解读[EIP7702]([https://eips.ethereum.org/EIPS/eip-7702])提案 [视频资料](https://www.youtube.com/watch?v=uZTeYfYM6fM)/[文字版](https://slowmist.medium.com/in-depth-discussion-on-eip-7702-and-best-practices-968b6f57c0d5)
- EOA: externally owned account外部拥有账户  
- EIP7702：  
钱包将指向链上的智能合约，功能上类似starknet账户    
引入一种新的交易类型（0x04），允许账户自行设置和委托代码。它不再将完整的合约代码直接存储在账户中，而是存储一个**委托指示符**，该指示符由一个特定的前缀加上一个地址 ( (0xef0100 || address))组成。这个指针指向实际智能合约代码在链上的位置。  
7702交易的destination不能为空：不能是创建合约的交易。  

- 用途：  
让EOA账户拥有合约账户那样灵活的能力
保持原有地址不变，兼容现有工具
L1层面支持多签、社交恢复、
gas替付等能力：可以使用代币或者第三方等结算gas
批量合约调用：比如把approve和swap通过合约打包成1比交易，可以减少gas
会话session：比如定期转账或者授权dapp定投等

- 数据结构：  
授权列表：与1559对比：增加委托列表，至少1条 [chain_id授权的链, address目标合约地址, nonce, y_parity签名, r, s]   
同一条链只有最后1条生效  
**chainid为0**的话，在所有支持7702的连上重放/授权，务必先确定每条链都是安全的  
通过tx==msg.sender来检查交易是否合法的合约，需要重新设计安全检查，因为有了7702之后，交易发起者不一定是实际用户  

- 与 EIP-4337 比较：  
    - EIP-7702 和 ERC-4337 都旨在推进账户抽象（Account Abstraction, AA），使外部拥有账户（EOA）能够拥有类似智能合约的功能.  
    - EIP-7702 集成更精简，而 EIP-4337 提供更全面的账户抽象模型，两者可针对不同场景共存。7702无需中转合约调用，相比4337多次合约调用，bundler打包，消耗更少的gas。  
    - 7702需要通过硬分叉，节点、rpc、钱包都需要升级才能支持，而4337目前已经支持。
    - [参考](https://mirror.xyz/0x9FFC14AB8754E4De3b0C763F58564D60f935Ad6F/eiLgBj9iPFmy4s4bmjY2jvEW_7g8YxYMQaHvqm9Xw_o)  

### 2025.05.15  
2. 安全相关: [8类安全问题](https://github.com/Yorkchung/EIP-7702-Learning/blob/main/Day02.md#security-risk-of-eip-7702)
- 存储冲突：修改授权合约时，要确保修改前后合约存储的slot内容对应一致，[slot相关介绍](https://mixbytes.io/blog/collisions-solidity-storage-layouts)，或者利用ERC7201，使用[存储命名空间](https://www.rareskills.io/post/erc-7201)来避免冲突
- 骗gas的爆破？【待实践】
- 老的链上合约，有些使用require(msg.sender == tx.origin)来检验交易是否合法，在7702升级之后需要升级合约，因为存在7702的账户在和这种老合约交互的时候，tx.origin并不一定是真实用户的地址
3. 链上数据浏览
- [链上智能账户统计](https://www.bundlebear.com/eip7702-overview/all)
- [okx的代理合约](https://etherscan.io/address/0x80296FF8D1ED46f8e3C7992664D13B833504c2Bb)
```solidity
/**
* @dev Modifier to make a function callable by the account itself or EOA address under 7702
*/

modifier onlySelf() {
    if (msg.sender != address(this)) revert Errors.NotFromSelf();
    _;
}
```
- [设置7702的EOA](https://holesky.etherscan.io/tx/0x29252bf527155a29fc0df3a2eb7f5259564f5ee7a15792ba4e2ca59318080182) 对比 [取消设置的EOA](https://holesky.etherscan.io/tx/0xd410d2d2a2ad19dc82a19435faa9c19279fa5b96985988daad5d40d1a8ee2269#authorizationlist)(代理合约地址0x0..0，注意和chainid=0区分)  
取消代理2：直接把 code hash 重設為空值，真正回歸純 EOA

### 2025.05.16
3. 协议实现细节：[交易流程](https://github.com/IntensiveCoLearning/EIP-7702/blob/main/wayhome.md#20250516)
4. [gas成本为什么是12500](https://github.com/IntensiveCoLearning/EIP-7702/blob/main/universe-ron.md#2-gas-%E5%B8%B3%E6%9C%AC---%E7%82%BA%E4%BD%95%E8%A6%81%E6%94%B6-12-500)

### 2025.05.17  
[跟着大佬学习](https://github.com/IntensiveCoLearning/EIP-7702/blob/main/easyshellworld.md)  
5. Pectra升级  
IP‑7702提案随 Ethereum 2025 年 5 月 7 日上线的 Pectra 升级一并激活，Pectra 是迄今为止最全面的一次硬分叉，包含共 11 项 EIP 标准。  
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

  
6. 从本地发起7702交易(rust)  
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

### 2025.05.18
7. demo方向  
- [**7702Defender**](https://www.hackquest.io/zh-cn/projects/ETH-Beijing-2025-7702Defender): 谷歌浏览器插件，自动识别交易风险  
- **链上跟单分成系统**：比如一个小虾米A账户订阅鲸鱼B账户，如果赚到钱了，那么通过代理合约自动分成，虾米70%，鲸鱼30%  
- **按周期/区块高度的订阅付费**：  
- **0gas账户**：钱包A有ETH作为gas，钱包B没有，钱包A发送EIP-7702交易，设置钱包B的授权合约，前提是需要签名验证的rsv参数。[测试代码](https://sepolia.etherscan.io/tx/0x18ac3032d0a11d9c5f7e9b4cde0b99ce68a0bae44442fbf955c059182ba5fe35#authorizationlist)
- **社交**：用户在链上发帖、点赞、打赏，平台或广告商替其付费
- **交易返佣／补贴**：币安、OKX 等中心化平台可在链上做“免 gas 交易”
- **DAO治理免gas投票**
- **游戏**：先玩后付，免费铸造，活动期间免费mint
- **gas token 二级市场**


### 2025.05.19
8. [测试7702交易](https://github.com/IntensiveCoLearning/EIP-7702/blob/main/wayhome.md#%E6%B5%8B%E8%AF%95-eip-7702-%E4%BA%A4%E6%98%93)  
- 合约，批量处理逻辑
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
- 签名验证
```solidity
bytes32 digest = keccak256(abi.encodePacked(nonce, encodedCalls));
require(ECDSA.recover(digest, signature) == msg.sender, "Invalid signature");
```
- 执行 和 赞助方执行
```solidity
function execute(Call[] calldata calls) external payable {
    // The caller executes the calls directly
}
function execute(Call[] calldata calls, bytes calldata signature) external payable {
    // A sponsor executes the calls on behalf of the caller
}
```
- 运行脚本
```bash
anvil --hardfork prague
forge install && forge build
forge script ./script/BatchCallAndSponsor.s.sol --tc BatchCallAndSponsorScript --broadcast --rpc-url 127.0.0.1:8545
```
### 2025.05.20
9. 代付gas场景测试(源自群友)
```solidity
import { privateKeyToAccount } from 'viem/accounts'
import { createWalletClient, http } from 'viem'
import { sepolia } from 'viem/chains'

const walletA = createWalletClient({
  account: privateKeyToAccount('0x...'), // 有ETH的钱包
  chain: sepolia,
  transport: http(),
})

const walletB = privateKeyToAccount('0x...') // 无ETH的钱包
const delegatedContract = '0x80296F...2Bb'   // 共享的授权合约

// Step 1: A 为 B 签署授权绑定 delegatedContract
const authorization = await walletA.signAuthorization({
  account: walletB,
  contractAddress: delegatedContract,
})

// Step 2: A 发起交易，帮 B 完成代码设置 + 初始化调用
const hash = await walletA.sendTransaction({
  authorizationList: [authorization],
  data: '0x8129fc1c', // initialize() 函数选择器
  to: walletB.address,
})

```
b钱包没有eth，a代发7702交易并设定b的合约逻辑；合约中msg.sender是b而不是a；  
这使得 tx.origin == msg.sender 不再能判断eoa发起人，因为7702的msg.sender是b，而不是现实世界的交易发起者a

### 2025.05.21

### 2025.05.22

### 2025.05.23

<!-- Content_END -->
