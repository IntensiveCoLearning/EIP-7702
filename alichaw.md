---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# 你的名字

1. alichaw
2. Yes, will try my best
3. TG: alichen88

## Notes

<!-- Content_START -->

### 2025.05.14
- Smart EOA 概念理解
- EIP-7702 的目的
- 和 EIP-4337、帳戶抽象的關係
- Pectra

[Day1 學習筆記](https://medium.com/@alichen308/day-1-eip-7702-%E6%A6%82%E5%BF%B5%E8%88%87%E8%83%8C%E6%99%AF%E7%90%86%E8%A7%A3-1fb81d56793f)

### 2025.05.15
- 新交易類型：SET_CODE_TX_TYPE (0x04)
- 授權列表 `authorization_list` 格式與驗證流程
- 簽名 domain：MAGIC = 0x05
- Nonce 增量邏輯（tx.nonce 與 auth.nonce 的對應）

[Day2 學習筆記](https://medium.com/@alichen308/day-2-eip-7702-%E6%8A%80%E8%A1%93%E7%B5%90%E6%A7%8B%E8%88%87%E7%B0%BD%E7%AB%A0%E6%A9%9F%E5%88%B6-a76440fb415a)

### 2025.05.16
- 批量交易（批次轉帳、approve+swap）
- 氣費贊助（sponsored txs）
- 權限細分（session keys、子密鑰）
- Social recovery 與 ZK 恢復流程

[Day3 學習筆記](https://medium.com/@alichen308/day-3-eip-7702-%E6%87%89%E7%94%A8%E5%A0%B4%E6%99%AF%E8%88%87%E5%84%AA%E5%8B%A2%E5%AF%A6%E4%BE%8B-084888ebc08a)

### 2025.05.17
- 常見弱點總覽（Quantstamp、SlowMist 整理）：
    - tx.origin 假設破壞
    - initialize frontrun
    - storage collision
    - fake recharge 攻擊（CEX 對 smart EOA 未辨識）
    - replay delegation
- EIP-1271 簽名驗證可被繞過？

[Day4 學習筆記]([https://medium.com/p/7ac22d44f398/edit](https://medium.com/@alichen308/day-4-eip-7702%E5%AE%89%E5%85%A8%E8%88%87%E9%98%B2%E7%A6%A6-7ac22d44f398))
<!-- Content_END -->
