---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# 你的名字

1. Billy
2. Yes, will try my best
3. TG: ounaiewfo

## Notes

<!-- Content_START -->

### 2025.05.14

### 2025.05.15

### 2025.05.16

EIP-7702 是為了讓傳統 EOA具備智慧帳戶能力而提出的提案，概念是透過一種特殊的Delegation Designator（0xef0100 || address），讓 EOA 指向外部的智能合約。

這讓 EOA 能有更多功能，包括：
- 批次交易執行：可在單一交易中完成多個操作，提高效率與用戶體驗。
- Gas Support：允許第三方支付交易費用（如 Dapp 為新用戶贊助 Gas）。
- 多簽與替代簽章邏輯：可將驗證邏輯改為社交恢復、多重簽章等形式。

此外，此提案也與帳戶抽象化目標一致，讓以太坊未來的錢包可以自由定義驗證機制與控制權限架構。但要注意以下安全問題：

- Delegated若未妥善設計存取控制，將導致任意執行邏輯（Access Control Risk）
- Constructor可被front run，導致帳戶被接管
- 多次委派間若未考慮Storage Collision，會出現邏輯漏洞
- 雜湊與簽名機制若不嚴謹，可能導致 replay 攻擊或錯誤授權（Replay / Verification Risk）

### 2025.05.17

1. 閱讀 https://hackmd.io/@colinlyguo/SyAZWMmr1x

```
contract Proxy {
    address immutable implementation;

    constructor(address impl) {
        implementation = impl;
    }

    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

2.Idea發想

Session Key可以使用在DeFi登入，減少User friction並增加交易速度
結合access control限制session key使用情境

例如Game Fi如果需要短時間內多次跟合約互動特定function，就可以使用


### 2025.05.18


<!-- Content_END -->
