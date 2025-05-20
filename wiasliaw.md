---
timezone: UTC+8
---


# wiasliaw

1. 自我介紹：技術上的 E 衛兵
2. 會完成殘酷學習嗎：會
3. tg@wiasliaw

## Notes

<!-- Content_START -->

### 2025.05.14

#### How it works

- 7702 透過 eip-2718 引入一個新的交易類型，允許 EOA 指定一個合約作為其程式碼。這樣使得 EOA 可以像是合約執行 evm bytecode 同時能保有發起交易的能力。

#### set code transaction

- set code transaction 的 type 為 0x04，主要多出一個 authorization_list 欄位
- 執行 set code transaction
    1. 先驗證 authorization_list 並處理 set code
    2. 再來執行原本的 transaction
- 交易內容如下：
    
    ```tsx
    const eip_7702_transaction = bytes.concat(
      // transaction type
    	"0x04",
    	// transaction payload
    	rlp([
    		chain_id,
    		nonce,
    		max_priority_fee_per_gas, // eip-1559
    		max_fee_per_gas, gas_limit, // eip-1559
    		to,
    		value,
    		data,
    		access_list, // eip-2930
    		authorization_list, // eip-7702
    		signature_y_parity,
    		signature_r,
    		signature_s
    	])
    );
    ```
    

#### authorization_list

- 要將一個 EOA 升級成 smart contract account 需要 EOA 簽名 authorization 的資料
    
    ```python
    authorization_list = [
    	[chain_id, address, nonce, y_parity, r, s],
    	...
    ]
    ```
    
- 對 authorization 的驗證流程如下
    1. 驗證 chain ID 是否為 0 或為當前鏈的 ID。這表示支援 cross-chain replay
    2. 驗證 nonce 是否小於 `2**64 - 1`。
    3. 驗證簽名 `authority = ecrecover(msg, y_parity, r, s)`
    4. 將 authority 加入 accessed_addresses，如 EIP-2929 所定義
    5. 驗證 `authority` 的代碼是否為空或已被委派
    6. 驗證 `authority` 的 nonce 是否等於 `nonce`
    7. 若 `authority` 非空，則向全域退款計數器增加 `PER_EMPTY_ACCOUNT_COST - PER_AUTH_BASE_COST` 的 gas
    8. 將 `authority` 的代碼設為 `0xfe0100 || address`
        - 如果 address 為 zero address 則清除 authority 的 code region，並將其 code hash 設置成 empty code hash
    9. 將 `authority` 的 nonce 增加 1
- 如果驗證中任一步驟失敗的話，則跳過當前的 authorization element，並開始處理下一個 authorization element
- 如果後續的 transaction revert，authorization 對 EOA 的寫入並不會 revert

#### delegation indicator

- 寫入到 `authority` 的資訊不是可以直接執行的 evm bytecode，而是一個指示器，用以指示 evm 要以不同的方式處理 code region
- `*CALL` 都是受影響的 evm opcode，當一讀到指示器時，會從 `address` 讀取其 code region
- 另外受影響的 opcode 還有 `CODESIZE` 和 `CODECOPY`，同樣也會從 `address` 讀取其 code region
- 但是 `EXTCODESIZE` `EXTCODECOPY` `EXTCODEHASH`則不受影響

#### Reference

- https://eips.ethereum.org/EIPS/eip-7702
- https://x.com/blainemalone/status/1893428082653962306

### 2025.05.15

#### 7702 對 UX 的影響

回顧 7702 之前的 account 分成 EOA 和 Smart Wallet Account。EOA 主要流程是對 target contract 發起交易，並為交易簽名。這樣的流程使得 EOA 需要先有一些 ETH 作為手續費才可以使用：

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F0e13fc79-68f6-4250-ac7b-770fd24029db%2Fac7d5225-2a67-49e0-8788-0ae5ccd3d210%2Fimage.png/size/w=2000?exp=1747293492&sig=jH2hf3EM29JNlwlWFWi-KMT7rPKO4VJgwJ5s-Km7YF4&id=1f4098e3-5fa6-8016-99f4-e6d7e4548602&table=block&userId=b83e83cc-a393-4f23-82e9-22582fd02e8d)

Smart Wallet Account 則是以 EOA 作為某個 Smart Contract Wallet 的 owner，Smart Contract Wallet 會驗證 msg.sender 或是簽名來執行或是呼叫其他的合約。Programmable Wallet 由此開始，可以在交易過程執行一些邏輯，例如多簽或是 batch transaction 等等：

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F0e13fc79-68f6-4250-ac7b-770fd24029db%2Fbd3fc784-08bb-422c-8bf7-bc82ae1ebd98%2Fimage.png/size/w=2000?exp=1747293581&sig=MtVUNMdLfSShyzVBveMOQDR3fKaWg7P9kvF1Bu64Uls&id=1f4098e3-5fa6-808c-90ff-d47e26697497&table=block&userId=b83e83cc-a393-4f23-82e9-22582fd02e8d)

7702 上線後，EOA 可以同時做到上述 EOA 和 Smart Contract Wallet 的兩種行為。EOA 可以有擴充的邏輯可以使用，現在也支援其他簽名演算法。將 EOA 擴充成 4337 進而擴充 Wallet 的用途：

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F0e13fc79-68f6-4250-ac7b-770fd24029db%2Fc93bc3cc-0794-4502-9b79-d0bf0e7d1617%2Fimage.png/size/w=2000?exp=1747293605&sig=CQyGE6ievAM0BLLk7BnYL86YWwp5Te-2-R4Lpc1fXss&id=1f4098e3-5fa6-80e2-9b9b-e67859af1c31&table=block&userId=b83e83cc-a393-4f23-82e9-22582fd02e8d)

### 2025.05.16

#### Best Practice

- Private Key Management
    - 透過 7702 委託後，雖然可以合約錢包實作的 social recovery 機制來取回權限，但是 EOA 還是有非常高的權限，像是撤銷 7702 委託，所以 Private Key 依舊要保管好
- Storage Management
    - 根據 7702，撤銷委託只會清除 code region 不會清除 contract storage。對錢包開發者來說，需要 storage 時需要特別注意 storage layout 的規劃
- Initialization Issue
    - 使用 7702 委託，只會更新 code region，並沒有額外函式可以為 code region 做初始化。
    - 在 7702 委託後，如果要做初始化，開發者應在初始化期間做檢查。
    - 如果是委託給 4337 錢包，確保是使用 0.8 的 EntryPoint (有實作兼容 7702)

### 2025.05.17

#### 4337 compliance

- 在啟用 7702 的鏈上，eth_sendUserOperation rpc endpoint 可以接受 7702 auth list 的資料。bundler 必須以 set code transaction 打包所有的 auth list
- 4337 建立 smart wallet account 的資料存在於 PackedUserOperation.initCode。initCode 資料如果以 7702 開頭並以 0x00 填充，則表示此 account 是使用 7702 部署。initCode 不會和 factory 合約互動。initCode 超過 20 bytes，其餘的資料會用於初始化
- https://eips.ethereum.org/EIPS/eip-4337#support-for-eip-7702-authorizations

### 2025.05.20

- 在 7702 的設計底下，由於 EOA 仍保留了直接發送交易的能力，所以其 private key 仍有非常大的權力
- 透過 Nick's Method 以構造出來的 ECDSA 簽名發送 7702 authorization 的資料讓一個沒有私鑰的 EOA 升級成 7702 set account，然後初始化一組 key 做管理。這樣就能削減 EOA 權限過大的問題。
- https://blog.biconomy.io/prep-deep-dive

<!-- Content_END -->
