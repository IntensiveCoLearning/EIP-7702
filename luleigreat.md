---
timezone: UTC+8
---



# Jerry

1. 自我介绍：我是一名区块链开发者，之前做过以太坊，OP相关的开发，建设及维护一条OP-Stack L2公链有近一年时间，目前已关停
2. 你认为你会完成本次残酷学习吗？：会
3. 你的联系方式（推荐 Telegram）：Telegram 用户名：@Jerry3721

## Notes

<!-- Content_START -->

### 2025.05.14
#### EIP-7702：从EOA到智能账户
**钱包能力提升**：
- 批量交易：如"approve"与"swap"可以在一笔交易中执行
- Gas捐赠：允许他人代支付交易手续费
- 身份认证：多种安全硬件可以用来验证账户身份，而不是只能通过私钥签名
- 消费限额：可以设置一个应用可以消耗多少token，或者一天最多消费多少token，提高账户安全性
- 恢复机制：当忘私钥时，可以通过多个不同选项来保护资产，而不是只能重新建立一个新账户

**用法**：

要使用EIP-7702,EOA需要签署授权交易，指向它想要执行其代码的特定委托地址，设置好后，账户获得了代理合约账户的代码能力（批量执行、Gas捐赠、授权逻辑等）。由于选择一个代理目标会交出很大控制权，EIP-7702会强制做以下检查：
- ChainID
- 账户nonce值 
- 撤销机制：可取消代理设置，或者设置一个新的代理


### 2025.05.15
学习了[EIP-7702：深入探讨智能EOA及其实施示例](https://learnblockchain.cn/article/13256)，发现还有很多新的知识点需要去了解，笔记如下：

1. EIP-7702引入新的交易类型 `SET_CODE_TX_TYPE` ( 0x04)，交易中有个`authorization_list`，是个授权列表:
```
authorization_list=[[chain_id,address,nonce,y_parity,r,s],…]
```
最终实现的效果是：
- 从负载（chain_id，address,nonce）和签名（y_parity，r,s）恢复出来的EOA地址设置address字段的地址为委托地址
- nonce值要与签名恢复出的EOA地址最新nonce一致，且每次授权后，nonce会+1
- 如果委托地址没被授权，会消耗25000的Gas成本，如果已授权则会部分退款12500
2. 撤销：用户可以利用 EIP-7702 修改授权地址。如果将 address 字段设置为 0x0000000000000000000000000000000000000000，则以往的授权将被撤销。这将清除账户的代码并将账户的代码哈希重置为空哈希。
3. 重新委托时，通过ERC-7201避免了存储位置的冲突
4. 查询交易状态，可以使用`eth_getTransactionReceipt`，查询当前授权，可以通过`eth_getCode`查询code，前缀为"0xef01"开头代表已设置代理地址
5. 指令集修改，如果账户授权了代理，以下指令会：
- EXTCODECOPY 仅返回0xef01
- EXTCODESIZE 仅返回2
- EXTCODEHASH 仅计算0xef01的Keccak256哈希
- CALL,CALLCODE、DELEGATECALL、STATICCALL将从委托代码地址加载代码并在EOA的上下文中执行
6. 与智能EOA交互，类似于调用智能合约，只需要将to字段设置为智能EOA地址


#### 问题
1. 如果A要设置B为委托地址，那么是A来签名一个授权，指定B为委托地址，然后放到`authorization_list`中，A再对整个交易签名、发送？这样A在一个交易中签名了两次，文章中说可以用这种签名解耦实现授权交易的Gas代付，没明白怎么操作
2. 为什么授权交易中，是一个授权列表，而不是单个授权，什么场景下会有多个授权在一个交易中出现？
3. 文章中提到 ERC-7201通过全名空间解决多次委托时存储位置冲突的问题，ERC-7201已经在主网上线了是吗
4. 调用智能EOA，还得是智能EOA的私钥来签名交易吧？也就是tx.origin与交易中的to地址相同？
5. 具体怎么实现与ERC-4337的兼容


### 2025.05.16

今天通过观看[Slowmist对EIP-7702的代码解析](https://www.youtube.com/watch?v=uZTeYfYM6fM)视频，以及查询一些其它资料，解答了昨天一些疑惑：
1. 需要明确的是，每个账户只能设置一个代理地址，如果设置多个，后面的会覆盖前面，最终只有一个会生效
2. 授权列表有多个，是一种批量授权的场景（同时解决了Gas代付问题），比如A，B，C，D都要设置地址X为代理地址，但是A-D账户都没有ETH，那么可以分别签署授权，然后把签名及授权payload交给一个有ETH的账户，一起发送到链上，同时实现对4个账户的授权操作
3. ERC-7201是一种合约规范，像ERC-4337一样，是没有修改链上实现的，需要开发者自己通过合约实现，具体的还得再研究下[ERC-7201 Storage Namespaces Explained](https://www.rareskills.io/post/erc-7201)
4. EIP-7702还是需要与ERC-4337配合才能实现Gas代付、批量交易等功能，智能EOA只能取代ERC-4337中的钱包账户能力（EOA + CA）


#### 问题
1. 设置完代理合约地址后，智能EOA账户自己可以调用自己来修改合约状态存储，如果是其它账户调用智能EOA呢，会修改其它账户的状态存储还是智能EOA账户的
2. 社交恢复的具体实现
3. 通过其它方式进行身份认证（而非账户私钥）

### 2025.05.17

今天跑了一遍[viem上的7702示例](https://viem.sh/docs/eip7702)，加深了对EIP-7702的理解：
1. 如果签名授权后，账户要自己发送SetCode交易，那么需要注意nonce值问题（授权增加nonce，发交易又增加一次），失败交易参考：[sepolia](https://sepolia.etherscan.io/tx/0xff9e60c0f5b0ab0ac3055d9f7dcfe7fdb6b1005a00aa9ee8252aaeba19ae966b#authorizationlist)
```
    const authorization = await walletClient.signAuthorization({
        // account: eoa,    //这样是有问题的，会设置失败
        contractAddress, 
        executor: 'self'    //注意这一行，这样设置没问题
    });
    
    const hash = await walletClient.writeContract({
        abi,
        address: walletClient.account.address,
        authorizationList: [authorization],
        functionName: "initialize",
    })
```
2. 在发送SetCode做授权的同时，还可以在同一笔交易中做智能EOA的调用，见上面的代码(`initialize`)
3. 昨天的第一个问题，如果其它账户调用智能EOA账户成功（有权限），修改的也是智能EOA账户自己的状态

### 2025.5.18
今天又找了一些EIP-7702的资料，对升级有了更深刻的理解：
1. 关于批量执行操作，[这篇文章](https://learnblockchain.cn/article/11498)给出了一个自己实现批量上链逻辑：
```
struct Call {
    address to;
    uint256 value;
    bytes data;
}

function execute(Call[] calldata calls) external payable {
    //交易发送者为智能EOA账户，自己发起的多个操作打包，不需要签名验证
    require(msg.sender == address(this), "Invalid authority");
    _executeBatch(calls);
}

function _executeBatch(Call[] calldata calls) internal {
    uint256 currentNonce = nonce;
    nonce++;

    for (uint256 i = 0; i < calls.length; i++) {
        _executeCall(calls[i]);
    }

    emit BatchExecuted(currentNonce, calls);
}

function _executeCall(Call calldata callItem) internal {
    (bool success,) = callItem.to.call{value: callItem.value}(callItem.data);
    require(success, "Call reverted");
    emit CallExecuted(msg.sender, callItem.to, callItem.value, callItem.data);
}
```
2. Gas赞助，类似于ERC-4337中的实现：用户签名`UserOperation`发送给Bundler统一打包上链：
```
//直接上链
function execute(Call[] calldata calls) external payable {
    // 调用者直接执行调用
}
//赞助者在验证签名是由 EOA 签署后，代表调用者执行调用。
function execute(Call[] calldata calls, bytes calldata signature) external payable {
    byte32 encodedCalls = abi.encodePacked(calls);
    bytes32 digest = keccak256(abi.encodePacked(nonce, encodedCalls));
    //这里还原签名地址后可能与白名单进行比较，不一定是msg.sender
    require(ECDSA.recover(digest, signature) == msg.sender, "Invalid signature");
    // 赞助者代表调用者执行调用
}
```

### 2025.5.19

#### EIP-7702 带来的影响与机会
EIP-7702 是一项重大改变，对区块链行业中的各个参与方都带来了重大影响，也提供了许多新的机会。

**合约钱包服务商**

从提案本身的标题和实现机制中可以看出，EIP-7702 本身并不专门提供有关账户抽象（Account Abstraction, AA）的上层能力（如 gas 抽象，nonce 抽象, 签名抽象等），只是提供了将 EOA 转化成 Proxy 合约 的基础能力。因此该提案并不与现行的各类合约钱包基础设施（如 Safe, ERC-4337 等）相冲突。相反，EIP-7702 与现有的合约钱包可以几乎无缝融合。允许用户的 EOA 转变成 Safe 钱包，ERC-4337 钱包。从而集成合约提供的多签、gas 代付、批量交易、passkey 签名等等功能。合约钱包提供商如果能快速、顺滑地集成 EIP-7702，可以有效扩展用户群体，提升用户体验。

**DApp 开发者**
由于 EIP-7702 对 AA 钱包的用户体验上有所提升，DApp 开发者可以考虑更大范围的接入 AA 钱包，为用户与 DApp 交互提供更便捷安全的体验。如利用合约钱包的批量交易能力一次性完成多个 DeFi 操作。

另一方向 DApp 也可以考虑根据项目特性为用户定制开发 Delegation 合约，为用户提供专有的复杂合约交互功能。

**资产托管方**
对于交易所、资金托管方，通常需要管理大批量的地址作为其用户的充币地址，并定期进行资金归集。传统的资金归集模式中使用 EOA 地址作为充币地址，进行归集时需要向充币地址充值一定金额的 gas，并由充币地址向出金地址进行转账交易。整个资金过程涉及大量的交易发送，流程很长且需要支付高额的 gas 费用。历史上也出现过机构进行资金归集导致链上交易费用攀升，交易拥堵的情况。

EIP-7702 的出现可以将 EOA 地址无缝转化成合约钱包，由于 EIP-7702 的 gas 代付机制，不再需要 gas 充值交易的存在。同时可以利用合约钱包的可编程性，将多笔归集转账打包在单笔交易中，减少交易数量并节约 gas 消耗。最终提高归集效率，降低归集成本。

### 2025.5.20
今天查EIP-7702关于与ERC4337融合的资料，找到如下内容，明天仔细研究下：
- [eip7702.io](https://eip7702.io/examples) 有关于用智能EOA生成 `UserOperation`的示例
- [ZeroDev](https://docs.zerodev.app/) 提供了关于智能EOA与智能合约账户（ERC-4337）的工具包

### 2025.5.21
今天看了下ZeroDev的文档，笔记如下：
ZeroDev 通过以下方式改进 Web3 UX：
- 密钥抽象
    - 通过passkey或社交账户登录
    - 如果用户丢失登录信息，可通过passkey等方式恢复帐户
- Gas抽象
    - 为用户赞助Gas
    - 支持用户用ERC20代币支付gas
- 交易抽象
    - 批量发送交易
    - 通过session key 自动发交易（对AI Agent友好）
- 链抽象
    - 不需要跨链即可在任意链上花费代币
    - 与任何交易所都有出入口，甚至在 L2 上

ZeroDev 是 AA 中最值得信赖的解决方案，为 100 多个团队在 30 多个网络上的 400 多万个智能账户提供支持。

试了一下通过passkey生成账户，并由ZeroDev官方代付Gas的示例：https://passkey-demo.zerodev.app/， 记录如下：
- passkey 生成账户需要手机的解锁密码
- 每次给passkey账户签名，都需要输入电脑的解锁密码

### 2025.5.22
#### ZeroDev
1. 今天看了下ZeroDev的DashBoard，发现有些类似Alchamy或Infura，要做Gas赞助，是这样的：
- 在DashBoard创建Project
- 设置Gas赞助，会生成一个ChainID绑定的rpc地址
2. 跑了下[7702的示例](https://github.com/zerodevapp/zerodev-examples/tree/main/7702)，没跑通，报错了，通过错误发现ZeroDev的rpc地址是一个自己提供的服务地址，服务封装了一些接口，如`pm_getPaymasterStubData`
3. 决定先不去研究ZeroDev了，原因如下：
- ZeroDev对账户抽象的实现比较复杂，封装比较深，一时没研究明白
- Github上的star数量并不多
- 这种类似于Alchamy的服务形式，注定后期要收费才能盈利

#### 7702
**应用场景**
- 批量交易，如approve + swap，要比ERC4337的方案省很多Gas
- Gas用ERC20代付，解决用户没有ETH的情况下也能交易的问题
**安全风险**
- chainid设置为0的情况下，别人可在其它链上通过重放交易来给账户设置代理
- [目前已经出现欺诈场景](https://www.iyiou.com/briefing/202505201776633)，因为eoa最终签名的是个hash，通过钓鱼网站，可以做到签名内容与hash不一致，为用户设置一个欺诈代理，把资产转移

### 2025.5.23
#### 总结篇
EIP-7702学习告一段落，本来是定的5天学完，其实5天已经把细节都学差不多了，该体验的也体验了，后来加到10天，由于后面工作繁重，没能花太多精力在上面，所以内容水了很多。

今天翻了翻其它同学的笔记，发现牛人还是很多的，言归正传，个人认为7702未来的方向预测：
- Metamask，OKX等钱包会集成EIP-770，加一个设置授权地址的功能，安全措施要不要做呢？从产品角度，好像超出了钱包的能力，但是还是期待能有这方面的方案
- 之前的ERC-4337方案都会支持EIP-7702 钱包，但是也有可能出现更轻量的方案集成EIP-7702实现Gas代付逻辑
- zkLogin、passKey等方案会被广泛应用，用户私钥恢复、web2方式登录会更加普及
- 由于门槛降低，web3用户会出现一波增量


还有一些问题后面需要继续研究：
- 私钥社交恢复的具体实现
- erc20代付gas的代码研究
- 存储冲突的解决，具体怎么用ERC-7201？


<!-- Content_END -->
