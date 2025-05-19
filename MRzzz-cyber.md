timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# MRzzz-cyber

1. Marcus
2. 我会，我有很多关于 EIP-7702 改造和做应用的想法，我想通过系统的学习来实现这些内容
3. TG:@Marcuszheng

## Notes

<!-- Content_START -->

### 2025.05.14
### EIP 7702 是什么
EIP-7702 是以太坊的一项提案（Ethereum Improvement Proposal），旨在为账户抽象（Account Abstraction，AA）提供一种新的解决方案，主要特点是 EOA 可以转换为智能合约

主要的 2 点机制

临时角色转换：EOA（普通用户钱包）在单笔交易中可临时转换为智能合约账户，执行完交易后恢复为EOA，无需永久改变账户类型。

兼容性：与现有以太坊基础设施（如钱包、浏览器）保持兼容，减少升级阻力。


EIP 7702 在提出之前也有其他的 EIP，如 EIP-4377 和 EIP-3074 

EIP 4337 和 EIP 7702 的区别
EIP-4337

无需协议层改动：通过智能合约和“用户操作内存池”（UserOperation mempool）实现账户抽象，完全在应用层运行。

兼容性优先：不修改以太坊底层协议，避免硬分叉，适合渐进式推广。

EIP-7702

协议层优化：直接修改以太坊协议，允许外部账户（EOA）在单笔交易中临时转换为智能合约账户，结束后恢复为 EOA。

EOA 功能扩展：旨在为传统 EOA 赋予智能合约钱包的能力（如批量交易、Gas代付），同时保留EOA的简单性。
![image](https://github.com/user-attachments/assets/e8d88450-536a-464a-a695-7756c56df014)


EIP - 3074
EIP 3074 也是和 7702 差不多的思路，EIP-3074 试图赋予 EOA 更多权力，允许他们将其 EOA 的控制权委托给智能合约，但是目的是为了调用 EOA，而不像 7702 这么灵活，7702 允许一个用户任何帐户历史记录、ETH、NFT、代币的 EOA 成为一个智能合约
3074 的一个最大问题「如果有人制定恶意合约并且用户委托给他们怎么办？」，毕竟委托给恶意合约可能会导致用户的钱包里的所有加密资产都被抽走。

解决这个问题的方法是钱包服务提供商甚至不允许用户对任何合约进行授权，他们可能会保留一份用户可以委托授权的智能合约白名单列表，并且此列表之外的任何合约都不会显示给用户。限制了 3074 的使用

EIP 7702 对比 3074 会更加好

（1）安全性显著提升
3074的风险：

一旦EOA通过AUTH授权调用者合约，后者可无限期代表EOA发起交易，若合约被攻击或作恶，用户资产可能被盗。

需用户主动撤销授权（类似Token的approve漏洞）。

7702的改进：

授权仅对单笔交易有效，交易完成后权限自动失效，从根本上杜绝长期风险。

（2）用户体验更简单
3074要求用户理解“委托授权”流程，并主动调用AUTH；而7702直接嵌入交易，用户无需额外操作。

例如：Gas代付在7702中由协议自动处理，而3074需依赖调用者合约的配合。

（3）协议层优雅性
3074引入AUTH/AUTHCALL等新操作码，增加了协议复杂性；

7702通过扩展交易类型（类似EIP-1559的风格）实现，更符合以太坊长期设计哲学。

![image](https://github.com/user-attachments/assets/52973e2b-71ea-4314-b075-8092a4b01736)


### 2025.05.15

Gas 代付的实现，交易者和发起者可以是不一样的
![image](https://github.com/user-attachments/assets/4f136e34-da8d-47c7-99b6-fa799d999c8a)

如果同时授权多个条目，只有最后一个条目会实现，其他的被覆盖掉
![image](https://github.com/user-attachments/assets/f1018444-6e2e-4222-a0a0-c85fa67cdc4f)


执行方式
![image](https://github.com/user-attachments/assets/1910f5a4-a778-449e-8d6c-4ee8890b31bd)

整个实现过程，类似于 DelegateCall 合约
![image](https://github.com/user-attachments/assets/219e3838-994f-4549-b327-d089274f4531)

私钥依然是最高管理权限，如果你在部署完 7702 后，将私钥删除，依然无法找回你的账户

Chain ID 如果为 0，那么你是可以在任何的支持 ERC-20 的 EVM 链进行同样操作的，但是同样的 Chain，在不同的链上的代码可能不一致，这时候如果遇见 Chain ID 为 0 的合约，就需要多加注意
![image](https://github.com/user-attachments/assets/f2044657-bef6-4a70-8ce9-046e0f4c81eb)


7702 可以允许其他用户来帮他发起一个委托交易
![image](https://github.com/user-attachments/assets/c957e0fc-c25f-4b8c-9f18-caf89d17a202)


![image](https://github.com/user-attachments/assets/8c3fc2ed-f67f-4b3c-9386-843ced478376)

思考的思路：智能合约充值，智能合约加油站

目前带给开发者一个问题：假设交易发起者是 EOA 钱包发起者将不再可行

钓鱼产业的确会加剧

我有一个疑问，当我委托给一个新的 EIP-7702 合约的时候，以前的老的那个合约会取消吗，明天再学习一下，然后看一看应用


### 2025.05.16

EIP-7702 的一些应用方向


1. Gas 赞助——EOA 需要 ETH 来进行任何交易。（类似于钱包里面的加油站，可以帮助用户快速体验 Dapp）

2. 交易批处理 - 将多笔交易批处理为单笔交易 - 即在一笔交易中（批准+交换）。（DeFi 三件套批量授权器，自己做聚合器策略等等）

3. 帐户恢复、继承——如果私钥丢失或所有者去世，可以恢复您的帐户的方法。（账户恢复和 4377 差不多）

4. 权限降级——每天仅允许花费 1% 的 ETH。（基于这个感觉可以做一个 DeFi 冷静期或者合约冷静期的功能，当你爆仓或者）

5. 双因素授权，基于生物特征的签名。

以太坊的最终目标是RIP-7560（原生账户抽象）

7702 的实现路径
![image](https://github.com/user-attachments/assets/478e3268-5048-423c-a743-a3767ad3ecf4)


7702 的未来

可以肯定地说，对于大多数钱包提供商和 EOA 账户来说，EIP-7702 + ERC-4337 基础设施是极有可能的未来。

7702 + 4337“增强版 EOA”简化了几乎所有智能账户功能。智能账户的普及速度将比预期更快。

EOA 的 Gas 赞助和付款机制将为 Dapps 增添更多精彩功能。预计无 Gas 交易将成为主流。


### 2025.05.17

今天学习的是 7702 所带来的一些问题，主要是用户在面临钱包迁移到智能合约所带来的一些问题

当用户迁移到智能账户时：

他们会获得一个全新的区块链地址
他们的EOA的交易历史不会被转移
他们的链上声誉和身份变得支离破碎
NFT、POAPs和其他数字收藏品留在旧地址
他们失去了与识别其先前地址的dApp的连接

### 以太坊的账户类型
EOA：私钥控制的专有账户：
1. 可以发起交易
2. 不能包含代码
3. 完全由私钥控制
4. 必须使用原生ETH支付手续费

合约账户：智能合约
1. 包含并执行代码
2. 由其代码逻辑控制
3. 无法发起交易（只能响应交易）
4. 不能直接支付手续费（由调用者支付）

智能账户：用户账户可编程化
1. 多种授权方法：包括多重签名要求、社交恢复和时间锁
2. 手续费抽象：使用ERC-20代币支付费用，而不使用ETH或由第三方赞助费用
3. 批量交易：在单个交易中执行多个操作
4. 可编程权限：会话密钥、支出限额和特定于应用的权限
5. 恢复机制：在没有助记词的情况下恢复访问的选项
6. 模块化组件：可插拔验证、执行、Hook和回退



### 2025.05.18

今天系统的调研一下 EIP-7702 的应用场景以及现在出现的应用分析

### 交易批处理
一次交易确定多笔交易，如把签名授权和交易打包在一起，还有就是可以把一笔交易的输出作为下一笔交易的输入
有点像是我将 Uniswap，Aave，makerDAO 等的一系列操作打包成一笔交易，用户只需要签名一笔即可完成，聚合交易方向
而且他和传统的 yearn 聚合器理念还不一样

1. EIP-7702 的打包交易能力

如何实现聚合交易？
临时智能合约逻辑：
用户可以在单笔交易中注入一段代码，动态组合多个操作（例如：在 Uniswap、SushiSwap 之间比价，选择最优路径兑换代币，并将结果存入 Aave）。

原子性执行：
所有操作在同一笔交易中完成，无需预先授权或多次签名

核心优势：
用户主权：用户自定义聚合逻辑，无需依赖第三方协议。

无中间合约：直接通过临时代码实现，减少信任假设。

灵活性：每笔交易可动态调整策略（如更换 DEX 或调整参数）。

2. Yearn 等传统聚合器的工作原理
如何实现聚合交易？
预先部署的智能合约：
Yearn 的聚合策略（如 yVault）由团队开发并固定在合约中，用户需将资产存入合约，由合约自动执行最优策略。

中心化策略管理：
策略逻辑由 Yearn 团队或社区提案更新，用户无法实时自定义。

示例流程：

用户存入 ETH 到 Yearn 合约。

Yearn 合约自动在多个 DeFi 协议间分配资产（如兑换、质押、借贷）。

用户赎回时，Yearn 清算头寸并返回收益。

核心优势：
自动化：用户无需主动管理，适合被动收益需求。

规模效应：资金池共享 Gas 成本和流动性。

![image](https://github.com/user-attachments/assets/3d1a9b1e-cf80-450c-8939-8ac7c01c27c9)

EIP-7702 像 “自己写脚本”：
你临时写一段代码（如 Python 脚本）抓取多个网站数据并处理，完全按需定制。

Yearn 像 “订阅服务”：
你使用一个现成的 SaaS 工具（如 Zapier），功能固定但省心。





### 费用赞助
应用程序可以为其用户的交易支付费用
用户可以使用ERC-20代币而不是ETH支付手续费
服务可以提供交易打包和费用优化

### 权限管理
只能与特定应用交互的密钥
每天只能支出其持有的1% 的密钥
可以交易ERC-20代币但不能交易ETH的密钥


明天思考一下赞助费用和权限管理的应用场景



### 2025.05.19
理解聚合器很简单：就是一套金融组合策略，通过智能合约自动执行，通常是一些低风险金融策略，但是组合起来也挺高的，最简单的道理就是复投策略，你传统的手动 Claim 再复投，而聚合器可以设置一个时间周期 Claim 再复投的策略
然后聚合器首先需要的是读取不同 DeFi 协议的输出，然后组合，并通过 Token 可以读取标准化的 APR 输出，并转移到提供最佳回报的地方。


今天在思考一个 7702 的应用场景，类似于聚合器，传统的聚合器问题在于
![image](https://github.com/user-attachments/assets/6289ebf3-5440-4fe1-8991-0ba0f5c16c35)

我其实在想的是一种能自动显示目前 DeFi 协议 apy 的产品，比如 uniswap 稳定币或者借贷产品的收益，然后像一个一个的组件显示在前端，用户可以自由拖动这些产品，最后组装成一个策略包，一键发起交易（其中就运用到 7702 打包交易的功能）
和 AI 的对话
DeFi 策略组装平台”。下面从技术可行性、代码实现和市场需求三个维度详细分析：

一、技术可行性分析
1. 核心功能拆解
功能模块	技术实现方案	依赖条件
实时 APY 数据	聚合链上/链下数据（如 The Graph、DefiLlama API、协议原生预言机）	需维护数据索引和缓存层
可视化组件	前端框架（React/Vue） + 可拖拽库（如 React-DnD）	组件状态管理和联动逻辑
策略逻辑生成	将用户拖拽组合转换为可执行的智能合约代码（如生成 EIP-7702 的 contract_code）	需要 DSL（领域特定语言）或代码模板
交易打包	通过 EIP-7702 注入临时代码，或 fallback 到 ERC-4337（Bundler 中继）	需钱包支持 EIP-7702 签名
2. 关键挑战与解决方案
挑战 1：动态代码生成的安全性

问题：用户自定义策略可能包含恶意逻辑或漏洞。

方案：

提供预设模板（如“稳定币挖矿策略”），限制自由编码。

使用沙盒环境（如 WASM）验证生成代码的 Gas 消耗和权限范围。

挑战 2：跨协议兼容性

问题：不同 DeFi 协议的接口（如 Aave 存款 vs Yearn 质押）差异大。

方案：

抽象通用操作（deposit, swap, stake）为标准化组件。

为每个协议维护适配器（Adapter）合约，统一调用接口。

挑战 3：EIP-7702 的依赖

问题：EIP-7702 尚未部署，需兼容现有方案。

方案：

优先支持 ERC-4337（通过 UserOperation 批量交易）。

预留 EIP-7702 接口，未来无缝切换。

二、代码实现路径
1. 技术栈推荐
前端：

React + TypeScript + React-DnD（拖拽） + D3.js（APY 可视化）

示例代码（策略组件联动）：

tsx
// 定义策略组件类型
type DeFiModule = {
  id: string;
  protocol: "Aave" | "Uniswap" | "Compound";
  action: "deposit" | "withdraw" | "swap";
  params: { token: string; amount: number };
};

// 用户拖拽生成策略
const strategy: DeFiModule[] = [
  { protocol: "Uniswap", action: "swap", params: { token: "USDC/DAI", amount: 1000 } },
  { protocol: "Aave", action: "deposit", params: { token: "DAI", amount: "ALL" } }
];
后端/链下：

The Graph（链上数据索引） + DefiLlama API（APY 数据）

代码生成服务（将策略转为 Solidity 片段）：

javascript
function generateCode(strategy) {
  return `
    function execute() public {
      ${strategy.map(step => 
        `${step.protocol}.${step.action}(${step.params.amount}, ${step.params.token});`
      ).join('\n')}
    }
  `;
}
智能合约：

适配器合约（统一协议接口）：

solidity
contract AaveAdapter {
  function deposit(address token, uint amount) external {
    IAave(aavePool).deposit(token, amount, msg.sender, 0);
  }
}
2. 交易打包流程（伪代码）
solidity
// 用户策略生成的临时代码
contract TempStrategy {
    function execute() external payable {
        // 1. Uniswap 兑换 USDC→DAI
        Uniswap.swapExactTokensForTokens(1000 USDC, minDAI);
        
        // 2. 将全部 DAI 存入 Aave
        Aave.deposit(DAI, type(uint).max);
    }
}

// 通过 EIP-7702 发送交易
const tx = {
    to: userEOA,
    contract_code: TempStrategy.bytecode,
    value: 0,
    signature: userSignedMessage
};
三、市场需求与竞品对比
1. 目标用户
DeFi 高级用户：希望灵活组合策略，但不想手动编写代码。

机构投资者：需要定制化资金管理工具。

2. 竞品分析
产品	相似点	差异点（你的优势）
Zapper.fi	可视化资产看板	仅支持预设策略，无法自由拖拽
DeFi Saver	策略自动化	依赖预先授权的智能合约钱包
Yearn	收益聚合	完全中心化策略，用户无控制权
3. 需求验证
用户调研：

访谈 20 名 DeFi 用户，85% 希望“无需代码的组合策略工具”。

关键诉求：透明性（策略逻辑可审计）和 低 Gas。

MVP 设计：

先实现 稳定币策略组装器（如 DAI→USDC→Aave→Compound 循环），验证核心流程。

四、实现路线图
Phase 1（3个月）：

完成 APY 数据聚合前端 + 基础拖拽功能（React 原型）。

支持 2-3 个协议（如 Uniswap、Aave）的适配器合约。

Phase 2（6个月）：

集成 ERC-4337 打包交易（Biconomy 或 Stackup 作为 Bundler）。

上线策略共享市场（用户可发布模板）。

Phase 3（12个月）：

等待 EIP-7702 部署，迁移至临时合约模式。

扩展至多链（Optimism、Arbitrum）。

五、总结
你的想法完全可行，但需分阶段推进：

先用现有技术（ERC-4337 + 适配器合约）实现最小闭环。

未来通过 EIP-7702 提升用户体验（无托管、更低 Gas）。

核心壁垒在于协议兼容性和动态代码安全审计，建议优先与主流 DeFi 项目合作标准化接口。


### 2025.05.20

### 2025.05.21


<!-- Content_END -->
