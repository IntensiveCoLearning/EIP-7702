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

<!-- Content_END -->
