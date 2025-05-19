---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# Garden

1. Hi 我是 Garden
2. 盡力完成
3. TG: @gardenxu

## Notes

<!-- Content_START -->

### 2025.05.14

### EIP-7702：概念入門與背景脈絡

#### 為什麼需要帳戶抽象？Account Abstraction 的動機

以太坊原生帳戶分為：
- EOA（Externally Owned Account）：傳統錢包，只有私鑰，不能內建邏輯。
- 合約帳戶（Contract Account）：有程式邏輯，可自訂執行規則，但部署後不易修改。

帳戶抽象（Account Abstraction）希望打破這個二分法：
- 讓 EOA 也能支援自訂邏輯（如多簽、限額、社交恢復）。
- 支援批次交易、自訂驗簽邏輯、由 DApp 代付 Gas 等功能。

#### EIP-4337 回顧與限制

EIP-4337（Account Abstraction via EntryPoint contract）為第一代帳戶抽象方案，其設計包含：
- **Bundler**：收集使用者操作（UserOperation）並統一送出。
- **EntryPoint** 合約：統一處理驗證與執行。
- 支援 `paymaster`（由他人代付 gas）。

但也存在限制：
- 需要獨立 mempool，不被所有客戶端支援。
- 執行流程複雜、學習曲線高。
- 不屬於 L1 協議層功能，執行效率與整合有限。

#### EIP-7702：核心概念與創新

EIP-7702 是 Vitalik 主導的新提案，核心為：
- 為 EOA 提供 **動態邏輯委派** 的能力。
- EOA 的 `code` 欄位可暫時設定為 `0xef0100 || address`，即指向某個邏輯合約（Delegation contract）。
- 在交易結束後，自動清除 `code` 欄位，回到原始狀態。

特色如下：
- 無需 Bundler，直接支援在 L1 上的帳戶抽象。
- 無需在帳戶地址重新部署合約，即可為 EOA 附加執行邏輯。
- 適合批次操作、gas 贊助、動態簽名驗證。

#### Type 4 新交易格式與比較

| 類型 | 名稱       | 特點                         | 對應 EIP    |
|------|------------|------------------------------|-------------|
| 0    | Legacy     | 最早期的交易格式             | 無          |
| 1    | EIP-2930   | 支援 `access list`           | Berlin      |
| 2    | EIP-1559   | 動態 gas 模型（Base Fee）    | London      |
| 4    | EIP-7702   | 動態附加 delegation 行為     | Pectra      |

Type 4 是 7702 引入的專屬格式，允許一次性附加 delegation code pointer 與 authorization list。

#### delegation pointer 的格式與意義

EIP-7702 使用的 delegation pointer：
```
0xef0100 || address
```
- `0xef0100` 是 prefix，代表這是一筆 delegation。
- `address` 是被委託的邏輯合約位址。
- 整體構成帳戶 `code` 欄位的內容，讓 EVM 執行時判斷要用 delegatecall 呼叫此地址。
- 此機制使得 EOA 能像 proxy 一樣執行外部邏輯，並保留自身的 storage context、msg.sender、msg.value

### 2025.05.15

### EIP-7702：Set Code Transaction

#### Set Code Transaction 概述

EIP-7702 引入了 Type 4 交易，一種新的交易格式，允許 EOA（Externally Owned Account）將帳戶行為委派給指定的邏輯合約。這是透過 `authorization_list` 欄位實現：該欄位中記錄的簽名授權資訊會讓帳戶的 `code` 欄位被設定為 `0xef0100 || address`，形成 delegation 指示器，使該帳戶具備如同 proxy 的行為能力。

#### Type 4 交易格式

Type 4 是基於 EIP-2718 的擴充格式，其交易結構如下：

```js
rlp([
  chain_id,
  nonce,
  max_priority_fee_per_gas,
  max_fee_per_gas,
  gas_limit,
  destination,
  value,
  data,
  access_list,
  authorization_list,
  signature_y_parity,
  signature_r,
  signature_s
])
```

其中 `authorization_list` 是由帳戶簽名的一組授權資料，用來指定委派邏輯：

```js
authorization_list = [
  [chain_id, address, nonce, y_parity, r, s],
  ...
]
```

#### 授權驗證與 Delegation 設定流程

當交易執行前，Ethereum 會依序處理 `authorization_list` 中的每一筆授權：

1. 驗證 `chain_id` 是否為 0 或等於當前鏈 ID。
2. 驗證 `nonce` 是否小於 2^64。
3. 使用 `ecrecover` 驗證簽章並取得授權帳戶 `authority`。
4. 驗證該帳戶的 `code` 欄位為空或已有 delegation。
5. 若條件符合，設定帳戶 `code = 0xef0100 || address`。
6. 若 `address == 0`，則代表清除 delegation，回復為一般 EOA。
7. 將 `authority` 的 nonce 增加 1，防止重放攻擊。

注意：即便交易邏輯 `revert`，上述 code 設定仍會生效不會被還原。

#### Delegation 指示器的 EVM 行為

當帳戶的 `code` 欄位為 `0xef0100 || address`（23 bytes），EVM 在執行該帳戶時會套用以下邏輯：

- CALL / CALLCODE / DELEGATECALL / STATICCALL 這些指令，會跳轉至 delegation contract 的邏輯執行。
- 執行邏輯時，仍保有原 EOA 的 `msg.sender`、`msg.value`、storage context。
- CODESIZE / CODECOPY 等會從 delegation contract 讀取實際邏輯。
- EXTCODESIZE / EXTCODECOPY / EXTCODEHASH 則仍讀取原帳戶自身資訊（僅看到 delegation indicator 的 23 bytes）。

此機制讓 EOA 擁有像 proxy 合約的能力，但無需部署 smart contract 即可享有抽象帳戶邏輯。

### 2025.05.16

### EIP-7702：交易流程解析（自我發起與贊助者）

EIP-7702 的核心在於讓 EOA 在不部署合約的前提下，藉由設定特殊 code delegation（`0xef0100 || address`），具備動態執行邏輯的能力。主要分為兩種交易流程：

---

#### 一、自我發起（Self-sponsored）

當 EOA 自身擁有 ETH 且能支付 gas 時，可主動發送 Type 4 交易完成 delegation 與執行邏輯：

1. **EOA 自行簽署授權並發送 Type 4 交易**
   - 使用者（sender）即 signer，簽署一筆 [chain_id, address, nonce] 的授權資料。
   - authorization 格式：`[chain_id, address, nonce, y_parity, r, s]`。
   - 簽章的 msg 為 `keccak256(0x05 || rlp(chain_id, address, nonce))`，ecrecover 還原出 authority（即該 EOA），其 code 欄位會被設為 delegation pointer。

2. **EVM 處理授權與 delegation 設定**
   - 驗證簽章正確、nonce 合理、帳戶 code 必須為空或可覆蓋 delegation。
   - 驗證通過後，將該帳戶 code 設為 `0xef0100 || address`，即指向 delegation contract。

3. **執行交易 data（call）**
   - 若交易目的地址即為 authority，則會觸發代理邏輯（執行 delegation contract 的邏輯，保留原 msg.sender、msg.value、storage）。
   - 若 call revert，則交易執行結果會回滾，但 code 設定不會還原。

---

#### 二、贊助者模式（Sponsored）

當 EOA 沒有 ETH 或希望他人代付 gas，可採用贊助者模式：

1. **使用者簽署離線授權資訊**
   - 被授權的 EOA 簽署 [chain_id, address, nonce]，產生簽章（signer 僅用於設定該 EOA 的 code）。
   - 簽章的 msg 為 `keccak256(0x05 || rlp(chain_id, address, nonce))`，ecrecover 還原出 authority（即被授權帳戶），其 code 欄位會被設為 delegation pointer。
   - 授權中的 nonce 必須匹配該 EOA 當前 nonce，確保每筆 delegation 只能設定一次，無法重放。

2. **Sponsor 組合並提交 Type 4 交易**
   - Sponsor 僅負責將授權清單（authorization_list）與 call data 組成交易並送出，不參與簽章。
   - 可同時包含多筆授權（多個 EOA），以及單一或批次 call data。
   - 簽名僅用於設定授權者帳戶的 code，而不是 signer 的帳戶。

3. **EVM 處理授權與 delegation 設定**
   - 對每筆授權資料，驗證簽章、nonce 是否正確，並檢查該帳戶 code 必須為空或 delegation indicator。
   - 若授權資料無效（簽章錯誤、nonce 不符、code 非空），則該筆授權被拒絕，略過不處理。
   - 驗證通過者，將其 code 設為 delegation pointer，並將 nonce 增加 1。

4. **執行交易主體 call**
   - 每筆 call data 會以對應的 delegated EOA 為上下文執行（透過 delegation contract）。
   - 若多個授權帳戶共用相同 call data，delegation contract 需根據 `msg.sender` 區分執行邏輯，否則可能導致混淆或資安問題。

5. **revert 與批次處理行為**
   - 若交易過程中任一筆 call 發生 revert，整體交易會回滾（atomicity preserved）。
   - 若僅 delegation 授權失敗（如簽章錯誤、nonce 不符、code 非空），則不影響其他授權與主體 call 的執行。

---

#### 驗證與安全性補充

- **簽章不可重複使用**：每筆 delegation 授權必須綁定唯一 nonce，只有 nonce 與帳戶當前 nonce 相符時才能設定 delegation。這確保 replay protection。
- **多筆交易含相同簽章會被拒絕**：若多筆交易嘗試重用相同簽章，因 nonce 不符或 code 非空，授權步驟會被拒絕。
- **簽章驗證方式**：`ecrecover(keccak256(0x05 || rlp(chain_id, address, nonce)), y_parity, r, s)` 取得 authority（即被授權帳戶）。

---

此兩種模式皆允許 EOA 在無需預先部署智能合約的前提下，實現一次性或動態委派自訂邏輯，並配合 L1 交易機制執行，為帳戶抽象提供更原生且彈性的架構。

<!-- Content_END -->
