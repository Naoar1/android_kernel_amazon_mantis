# AIS FSM 詳細導讀（MTK Wi‑Fi / gen4 / Mantis）

> 目的：你正在學 driver code，這份專門帶你讀 `ais_fsm.c`。  
> 重點：**白話 + 具體函式 + 具體欄位 + 具體事件**。  
> 範圍：`drivers/misc/mediatek/connectivity/wlan/gen4/mgmt/ais_fsm.c` 與 `include/mgmt/ais_fsm.h`。

---

## 1) AIS FSM 是什麼？

AIS = **Infrastructure STA** 的連線狀態機（也兼顧部分 Ad-hoc 與 remain_on_channel）。

你可以把它想成 Wi‑Fi 站台模式的大腦，負責：

1. 何時掃描（scan）
2. 何時找 AP（search）
3. 何時要求頻道權（channel grant）
4. 何時開始 join/auth/assoc
5. 何時宣告連上 / 斷線 / 重連

AIS 自己不直接做所有硬體事，會透過 mailbox 叫其他模組：

- SCN（scan）
- CNM（channel manager）
- SAA（auth/assoc）
- Roaming FSM

---

## 2) 先看 header：狀態、請求、資料結構

檔案：`include/mgmt/ais_fsm.h`

## 2.1 狀態列舉 `ENUM_AIS_STATE_T`

共 15 個主要 state：

- `IDLE`
- `SEARCH`
- `SCAN`
- `ONLINE_SCAN`
- `LOOKING_FOR`
- `WAIT_FOR_NEXT_SCAN`
- `REQ_CHANNEL_JOIN`
- `JOIN`
- `JOIN_FAILURE`
- `IBSS_ALONE`
- `IBSS_MERGE`
- `NORMAL_TR`
- `DISCONNECTING`
- `REQ_REMAIN_ON_CHANNEL`
- `REMAIN_ON_CHANNEL`

對應 debug 字串在 `ais_fsm.c` 開頭 `apucDebugAisState[]`。

## 2.2 請求列舉 `ENUM_AIS_REQUEST_TYPE_T`

AIS 內部排隊請求（pending list）主要有：

- `AIS_REQUEST_SCAN`
- `AIS_REQUEST_RECONNECT`
- `AIS_REQUEST_ROAMING_SEARCH`
- `AIS_REQUEST_ROAMING_CONNECT`
- `AIS_REQUEST_REMAIN_ON_CHANNEL`

## 2.3 核心資料 `AIS_FSM_INFO_T`

你最該關注的欄位：

- `eCurrentState` / `ePreviousState`
- `fgIsChannelRequested` / `fgIsChannelGranted`
- `ucSeqNumOfReqMsg`（join 相關序號）
- `ucSeqNumOfChReq`（channel request 序號）
- `ucSeqNumOfScanReq`（scan 序號）
- `ucConnTrialCount`（連線重試次數）
- `prTargetBssDesc`（目前鎖定的 AP）
- `prTargetStaRec`（join 目標 sta record）
- `rPendingReqList`（等待中的請求）
- 各種 timer：
  - `rBGScanTimer`
  - `rScanDoneTimer`
  - `rJoinTimeoutTimer`
  - `rChannelTimeoutTimer`
  - `rDeauthDoneTimer`

---

## 3) AIS 初始化（一定先看）

## 3.1 `aisInitializeConnectionSettings()`

檔案：`ais_fsm.c`（前段）

設定預設連線策略：

- `eOPMode = NET_TYPE_INFRA`
- `eConnectionPolicy = CONNECT_BY_SSID_BEST_RSSI`
- `eAuthMode = AUTH_MODE_OPEN`
- `eEncStatus = ENUM_ENCRYPTION_DISABLED`
- `fgIsConnReqIssued = FALSE`
- `fgIsScanReqIssued = FALSE`

也會初始化 RSN 相關欄位與 roaming 預設。

## 3.2 `aisFsmInit()`

- 建立/初始化 AIS BSS info
- 初始 state：`IDLE`
- `ucSeqNumOfReqMsg/ChReq/ScanReq = 0`
- 初始化所有 timer callback

這一步做完，AIS 可以開始接收 scan/connect 等事件。

---

## 4) 核心引擎：`aisFsmSteps()`

函式：`aisFsmSteps()`（`ais_fsm.c:849`）

這是 AIS 的主 state engine。你看這個函式就能看懂 80% 行為。

它每次進來做三件事：

1. 記錄 `ePreviousState`
2. 印 transition log（`[old] -> [new]`）
3. 執行新 state 對應 entry action

而且它允許「連續 transition」：

- 用 `fgIsTransition` 控制 do-while 再跳下一個 state

---

## 5) 每個重要 state 進入時做什麼（具體）

## 5.1 `AIS_STATE_IDLE`

- 停 `rScanDoneTimer`
- 取 pending request（`aisFsmGetNextRequest`）
- 若有 connect 需求（`fgIsConnReqIssued`）：轉 `SEARCH`
- 若有 pending scan：轉 `SCAN`
- 若有 remain_on_channel：轉 `REQ_REMAIN_ON_CHANNEL`

## 5.2 `AIS_STATE_SEARCH`

- `scanSearchBssDescByPolicy()` 找候選 AP
- 找到：
  - 記錄 RSN cipher/AKM
  - `prTargetBssDesc = prBssDesc`
  - 轉 `REQ_CHANNEL_JOIN`
- 找不到：
  - 若 `fgTryScan` -> `LOOKING_FOR`
  - 否則 `aisFsmStateSearchAction(...)` 決定轉 `IDLE` 或 `WAIT_FOR_NEXT_SCAN`

## 5.3 `AIS_STATE_SCAN / ONLINE_SCAN / LOOKING_FOR`

- 組 `MSG_SCN_SCAN_REQ_V2`
- 填 `ucSeqNum = ++ucSeqNumOfScanReq`
- 決定 scan type：
  - wildcard / 指定 SSID
- 決定掃描 channel：
  - full / 2G4 / 5G / 指定 channel list
- `mboxSendMsg(... MID_AIS_SCN_SCAN_REQ_V2 ...)` 送給 SCN FSM

## 5.4 `AIS_STATE_REQ_CHANNEL_JOIN`

- 組 `MSG_CH_REQ_T`
- 填：目標 channel/band/sco/bw
- `mboxSendMsg(... MID_MNY_CNM_CH_REQ ...)` 向 CNM 要頻道權
- `fgIsChannelRequested = TRUE`

## 5.5 `AIS_STATE_JOIN`

- 呼叫 `aisFsmStateInit_JOIN(prTargetBssDesc)`
- 這裡會建 `STA_RECORD`、挑 auth algorithm、送 `MID_AIS_SAA_FSM_START`

## 5.6 `AIS_STATE_JOIN_FAILURE`

- `nicMediaJoinFailure(...)`
- 轉回 `IDLE`

## 5.7 `AIS_STATE_NORMAL_TR`

- 若 join channel 還在用（`fgIsInfraChannelFinished == FALSE`）先不動
- 否則依 pending request：
  - scan -> `ONLINE_SCAN`
  - roaming search -> `LOOKING_FOR`
  - roaming connect -> `SEARCH`
  - remain_on_channel -> `REQ_REMAIN_ON_CHANNEL`

## 5.8 `AIS_STATE_DISCONNECTING`

- 發 deauth frame
- 啟動 `rDeauthDoneTimer`

## 5.9 `AIS_STATE_REQ_REMAIN_ON_CHANNEL`

- 送 `MID_MNY_CNM_CH_REQ`
- 等 `aisFsmRunEventChGrant` 進下一步

## 5.10 `AIS_STATE_REMAIN_ON_CHANNEL`

- 保持 network active
- 等 cancel / timeout

---

## 6) 關鍵事件處理（你要逐個看）

## 6.1 Scan done：`aisFsmRunEventScanDone()`（`1527`）

流程：

1. 檢查 seq 是否等於 `ucSeqNumOfScanReq`
2. 停 `rScanDoneTimer`
3. 清 `fgIsScanReqIssued`
4. `kalScanDone(...)` 回報上層
5. 狀態轉移：
   - `SCAN` -> `IDLE`
   - `ONLINE_SCAN` -> `NORMAL_TR`（或 roaming update）
   - `LOOKING_FOR` -> `SEARCH`（或 roaming update）

## 6.2 Abort：`aisFsmRunEventAbort()`（`1618`）

- 讀 disconnect reason
- 設 `fgIsDisconnectedByNonRequest`
- 處理 roaming 特例（reassociation）
- 最後呼叫 `aisFsmStateAbort(...)`

## 6.3 Join complete：`aisFsmRunEventJoinComplete()`（`1834`）

- 檢查 state 是 `JOIN`
- 檢查 seq（`ucSeqNumOfReqMsg`）
- 呼叫 `aisFsmJoinCompleteAction()`

`aisFsmJoinCompleteAction()`（`1869`）是連線成敗分歧主點：

### 成功分支

- `aisChangeMediaState(CONNECTED)`
- `aisUpdateBssInfoForJOIN(...)`
- `cnmStaRecChangeState(... STA_STATE_3)`
- `aisIndicationOfMediaStateToHost(CONNECTED)`
- 狀態轉 `NORMAL_TR`

### 失敗分支

- retry 其他 auth type（`aisFsmStateInit_RetryJOIN`）
- 或增加 fail count 後：
  - `IDLE` / `WAIT_FOR_NEXT_SCAN` / `JOIN_FAILURE`

## 6.4 Channel grant：`aisFsmRunEventChGrant()`（`3377`）

- 若是 `REQ_CHANNEL_JOIN` + token match：
  - 設 join timeout timer
  - 轉 `JOIN`
  - `fgIsChannelGranted = TRUE`
- 若是 remain_on_channel：
  - 轉 `REMAIN_ON_CHANNEL`
  - 回報 `kalReadyOnChannel(...)`

## 6.5 Release channel：`aisFsmReleaseCh()`（`3460`）

- 若 requested/granted 任一為 true
- 送 `MID_MNY_CNM_CH_ABORT`
- 清 `fgIsChannelRequested/Granted`

---

## 7) timeout 處理（最容易出 bug）

## 7.1 Scan timeout：`aisFsmRunEventScanDoneTimeOut()`

- 強制 `kalScanDone(...)`
- `aisFsmStateAbort_SCAN()`（發 scan cancel）
- 依 state 轉回 `IDLE` 或 `NORMAL_TR`

## 7.2 Join timeout：`aisFsmRunEventJoinTimeout()`（`3068`）

- 在 `JOIN` state timeout：
  - `aisFsmStateAbort_JOIN()`
  - 增加 join failure count
  - 依條件轉 `SEARCH` / `WAIT_FOR_NEXT_SCAN` / `JOIN_FAILURE`

## 7.3 BG scan timeout：`aisFsmRunEventBGSleepTimeOut()`

- `WAIT_FOR_NEXT_SCAN` 到時轉 `LOOKING_FOR`

## 7.4 Channel timeout：`aisFsmRunEventChannelTimeout()`（`4017`）

- 釋放 channel
- 回報 `kalRemainOnChannelExpired(...)`
- 轉 `NORMAL_TR` 或 `IDLE`

---

## 8) Request queue（小但關鍵）

函式：

- `aisFsmInsertRequest()` (`3875`)
- `aisFsmIsRequestPending()` (`3814`)
- `aisFsmGetNextRequest()` (`3851`)
- `aisFsmFlushRequest()`

用途：

當前 state 不能立刻做某件事時，先排隊。  
例如在 `NORMAL_TR` 時收到 scan，可能先掛 `AIS_REQUEST_SCAN`，等合適時機處理。

---

## 9) 與其他模組的 mailbox 互動（重要 message）

在 `ais_fsm.c` 你會看到這些 message id：

- `MID_AIS_SCN_SCAN_REQ_V2`（要求 SCN 開始掃描）
- `MID_AIS_SCN_SCAN_CANCEL`（要求 SCN 取消掃描）
- `MID_MNY_CNM_CH_REQ`（向 CNM 要頻道）
- `MID_MNY_CNM_CH_ABORT`（向 CNM 還頻道）
- `MID_AIS_SAA_FSM_START`（叫 SAA 做 auth/assoc）
- `MID_AIS_SAA_FSM_ABORT`（中止 join）

這些就是 AIS 對外溝通骨幹。

---

## 10) 兩條最常用的導讀路徑

## A) Scan 路徑

1. `aisFsmScanRequestAdv()`
2. `aisFsmSteps()` 中 `AIS_STATE_SCAN/ONLINE_SCAN`
3. `aisFsmRunEventScanDone()`
4. `aisFsmRunEventScanDoneTimeOut()`（超時分支）

## B) Connect 路徑

1. `aisFsmStateSearchAction()` + `AIS_STATE_SEARCH`
2. `AIS_STATE_REQ_CHANNEL_JOIN` -> `aisFsmRunEventChGrant()`
3. `AIS_STATE_JOIN` -> `aisFsmStateInit_JOIN()`
4. `aisFsmRunEventJoinComplete()` / `aisFsmJoinCompleteAction()`
5. `aisIndicationOfMediaStateToHost()`

---

## 11) 讀碼時你要特別注意的旗標/欄位

- `fgIsConnReqIssued`：現在是否有連線需求
- `fgIsScanReqIssued`：是否已有 scan in-flight
- `fgIsDisconnectedByNonRequest`：是否非使用者主動斷線
- `fgIsInfraChannelFinished`：join channel 流程是否完成
- `ucConnTrialCount`：重試次數
- `ucSeqNumOfScanReq / ucSeqNumOfReqMsg / ucSeqNumOfChReq`：事件匹配關鍵

---

## 12) 實作層面常見坑

1. **seq mismatch**
   - 事件到了但被丟棄（「不是當前 request 的回應」）
2. **timer 與 state 不一致**
   - timeout callback 進來時 state 已切換，容易進錯分支
3. **pending request 沒清乾淨**
   - 導致重複 scan/connect 或奇怪 state 跳轉
4. **channel 未釋放**
   - 後續 join/roc 互相卡住

---

## 13) 建議的「實作級」閱讀順序

### 第 1 輪（看框架）

- `ais_fsm.h`（state/request/struct）
- `aisFsmInit`, `aisFsmSteps`

### 第 2 輪（看主流程）

- `aisFsmScanRequestAdv` + `aisFsmRunEventScanDone`
- `AIS_STATE_SEARCH/REQ_CHANNEL_JOIN/JOIN`
- `aisFsmRunEventJoinComplete`

### 第 3 輪（看錯誤/超時）

- `aisFsmRunEventJoinTimeout`
- `aisFsmRunEventScanDoneTimeOut`
- `aisFsmStateAbort_*`

### 第 4 輪（看額外功能）

- `remain_on_channel` 系列
- roaming 相關 `AIS_REQUEST_ROAMING_*`

---

## 14) 一句話總結

AIS FSM 的核心不是「連線 API」，而是：

**在多事件（scan/join/roaming/timeout）同時發生時，保證狀態一致、channel 一致、回報一致。**

你把 `aisFsmSteps + JoinComplete + Timeout + RequestQueue` 四塊吃透，就真的懂 AIS 了。
