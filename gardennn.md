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

### 2025.05.17

### EIP-7702：安全性考量

#### 常見風險與防範建議

- **避免使用 `tx.origin` 作為重入保護（Reentrancy Guard）**：
  EIP-7702 改變了 `msg.sender == tx.origin` 的判斷方式，建議使用 transient 儲存或標準鎖機制取代 `tx.origin`。

- **避免使用原子初始化（atomic init）**：
  原子化 `init()` 呼叫可能遭到前置攻擊（frontrunning），建議使用 `initWithSig` 搭配簽章驗證初始化邏輯。

- **初始化應透過 EntryPoint 呼叫以提升安全性**：
  若採用 4337-compatible 的架構，初始化邏輯應限制由 EntryPoint 呼叫以避免未授權配置。

- **避免 storage layout 衝突（Storage Collisions）**：
  切換 delegation contract 時應確保使用 ERC-7201 命名空間等機制，避免不同合約之間發生存儲欄位覆蓋。

- **避免委派至具動態行為的合約（如 upgradeable 或 selfdestruct）**：
  僅應委派至透過 `CREATE2` 部署、不可變的邏輯合約，防止釣魚與意外邏輯改變。

- **最小化信任基礎與攻擊面（Trusted Surface）**：
  Delegation contract 應保持邏輯簡單、功能封閉、可審計，避免動態 dispatch 或不必要的外部呼叫。

- **選用已審計與可信任的標準合約模組**：
  優先採用來自 Ambire、Alchemy、MetaMask 或 EF 等已知團隊維護並審計過的模組化合約設計。

### 2025.05.18

### EIP-7702：Delegation 合約的設計原則

#### 一、設計核心與部署原則

- 委派合約應透過 **CREATE2 部署**，保證 deterministic address，可預先驗證與靜態指向。
- 合約邏輯應明確不可變，**避免使用 upgradeable patterns 或具毀損性的 selfdestruct**。
- 每個 delegation contract 應對應特定業務目的，避免萬用 dispatch 結構。

#### 二、簽章驗證與使用者授權綁定

- 每筆簽章應與交易中實際使用的 **msg.sender、calldata、gas、value、nonce（或 salt）一致**。
- EIP-7702 的簽章驗證格式為：`ecrecover(keccak256(0x05 || rlp([chain_id, address, nonce])), y_parity, r, s)`
- 為提升互通性與安全性，推薦使用 **EIP-712 標準格式** 建立簽章內容。

#### 三、多使用者支援設計

- 若允許多帳戶共用 delegation contract，應設計適當資料結構以管理權限，例如：
  ```solidity
  mapping(address => Permission) public permissions;
  ```
- 可擴充設計如：
  - 每個 signer 可被限制僅能呼叫特定函式 selector
  - 支援每日限額、可互動的 target contract 白名單等

#### 四、meta 資訊與風控擴充欄位

- 建議加入簽章參數：
  - `purpose`：明確表示簽章用途（如 mint、transfer）
  - `chainId`：防止跨鏈重放攻擊
  - `deadline`：簽章過期時間
- 可考慮加上 session ID、防重放 hash、或交易 domain 作為額外驗證。

#### 五、安全與審計建議

- 每筆簽章只能用一次，nonce 應與帳戶當前 nonce 精準匹配，避免重複授權。
- delegation 合約實作應通過靜態分析工具（如 Slither）與形式驗證（如 MythX）。
- 請避免動態 dispatch 結構與過多 call、delegatecall，減少潛在攻擊面。
- 設計應符合最小信任原則與模組化（如 OpenZeppelin AccessControl）。

> EIP-7702 的 delegation contract 並非萬用 proxy，應精確對應特定交易型態或邏輯，實作時宜採 whitelist、限權、風控多層交叉驗證，並以低複雜度、高可審計為原則。

### 2025.05.19

### EIP-7702 × ERC-4337：整合實作與應用策略

EIP-7702 提供 L1 原生帳戶抽象功能，不依賴 EntryPoint，設計更輕量且與現有交易邏輯相容；而 ERC-4337 則透過 Bundler 與 EntryPoint 提供 L2 抽象化錢包架構。兩者可互補整合，形成混合型帳戶抽象框架。

#### 結合方式與應用邏輯

- **Delegation contract 可實作 `validateUserOp()` 與 `execute()`**，與 ERC-4337 的 EntryPoint 完整對接，實現兼容型帳戶抽象。
- **於 4337 wallet 中引入 delegation pattern**，可支援 fallback recovery、session key、限權交易等彈性機制。
- **EIP-7702 提供 fallback 機制**：若主驗簽失效，可改由 delegation contract 實作 `isValidSignature()` 驗證邏輯。
- **支援二階段授權架構**：簽名與執行行為可由不同 signer 操作，實現授權與使用責任分離。
- **UserOperation 可增設 `delegationProof` 欄位**，提供基於 EIP-7702 授權的額外驗證來源與風控依據。
- **混合模式可依需求分流**：批次處理交由 ERC-4337 處理，細緻操作則由 EIP-7702 進行個別授權。

#### 優勢總結

- **組合彈性高**：EIP-7702 可提供 L1 安全且可控的邏輯授權，ERC-4337 提供策略封裝與批次處理功能。
- **安全設計更進階**：fallback 機制讓帳戶擁有多層次驗簽途徑與備援邏輯。
- **未來可建構 Layered Account Abstraction 架構**：4337 為策略與排程層，7702 提供底層 delegation 邏輯與執行控制。

> EIP-7702 與 ERC-4337 並非對立，而是互補：前者強化 L1 執行安全性與簡潔性，後者擴展錢包行為與批次能力。未來帳戶抽象標準中，EIP-7702 有潛力成為 recovery、fallback、限權 session 的關鍵元件。

### 2025.05.20

### EIP-7702：交易建構與 Viem 實作

#### Viem 實作交易流程

透過 Viem 發送 EIP-7702 類型（Type 4）的 smart account 交易，流程如下：

1. 使用 `signAuthorization()` 對某個 delegation contract 產生簽章。
2. 建立交易時，將 `to` 設為 **EOA 自己的地址**。
3. `data` 為呼叫 implementation contract（即代理邏輯合約）的 `execute(calls)` 方法。
4. 帶入 `authorizationList`，確保授權生效。
5. 可由自己或 sponsor 代為送出交易（sponsored execution）。

---

#### Viem TypeScript 實作程式碼

```ts
import { createWalletClient, http, parseEther } from 'viem';
import { anvil } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';
import { eip7702Actions } from 'viem/experimental';
import { abi, contractAddress } from './contract'; // Assuming you have already deployed the contract and exported the ABI and contract address in a separate file

const account = privateKeyToAccount('0x...'); // EOA 的私鑰產生帳戶

const walletClient = createWalletClient({
  account,
  chain: anvil,
  transport: http(),
}).extend(eip7702Actions()); // 加入 EIP-7702 擴充功能

// Step 1: 產生對 implementation contract 的簽章授權
const authorization = await walletClient.signAuthorization({
  contractAddress,
});

// Step 2~4: 組裝與送出交易
const hash = await walletClient.sendTransaction({
  to: walletClient.account.address, // 設為 smart account 本身地址
  authorizationList: [authorization],
  data: encodeFunctionData({
    abi,
    functionName: 'execute',
    args: [
      [
        {
          to: '0xcb98643b8786950F0461f3B0edf99D88F274574D',
          value: parseEther('0.001'),
          data: '0x',
        },
        {
          to: '0xd2135CfB216b74109775236E36d4b433F1DF507B',
          value: parseEther('0.002'),
          data: '0x',
        },
      ],
    ],
  }),
});
```


#### Flow of EIP-7702 Transaction
![image](https://github.com/user-attachments/assets/5cf8e323-3ccb-475a-887e-1866b7866c1e)

<!-- Content_END -->
