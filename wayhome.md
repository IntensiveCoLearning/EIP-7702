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
