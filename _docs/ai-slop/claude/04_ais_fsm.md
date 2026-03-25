# AIS FSM 參考

檔案：`drivers/misc/mediatek/connectivity/wlan/gen4/mgmt/ais_fsm.c`  
Header：`include/mgmt/ais_fsm.h`

---

## 0. AIS FSM 的角色

AIS（Infrastructure STA）FSM 是 MTK Wi-Fi driver 中 Station 模式連線管理的核心狀態機。它本身不直接操作硬體，而是透過 mailbox 訊息系統協調以下模組：

| 模組 | Mailbox 訊息方向 | 負責工作 |
|---|---|---|
| SCN（Scan） | AIS → SCN | 執行 802.11 probe/beacon 掃描 |
| CNM（Coexistence and Non-concurrent Management） | AIS ↔ CNM | 管理頻道使用權（避免各功能搶佔同一頻道） |
| SAA（Station Association Arbiter） | AIS → SAA | 執行 802.11 auth/assoc frame 交換 |
| Roaming FSM | AIS ↔ Roaming | 漫遊決策 |

AIS FSM 的主要職責是：在多個非同步事件（scan done、channel grant、join complete、timeout、disconnect request 等）同時發生時，保證 driver 的狀態一致性，channel 資源不洩漏，以及向 user space 的回報準確有序。

---

## 1. 資料結構

### 1.1 `AIS_FSM_INFO_T`（`include/mgmt/ais_fsm.h`）

```c
typedef struct _AIS_FSM_INFO_T {
    /* 狀態 */
    ENUM_AIS_STATE_T    eCurrentState;
    ENUM_AIS_STATE_T    ePreviousState;

    /* Sequence numbers（事件匹配關鍵） */
    UINT_8  ucSeqNumOfReqMsg;   /* join 請求的 seq，比對 join complete event */
    UINT_8  ucSeqNumOfChReq;    /* channel request 的 token，比對 channel grant */
    UINT_8  ucSeqNumOfScanReq;  /* scan 請求的 seq，比對 scan done event */

    /* 頻道管理 */
    BOOLEAN fgIsChannelRequested;  /* 已向 CNM 發出頻道請求 */
    BOOLEAN fgIsChannelGranted;    /* CNM 已核准頻道 */
    BOOLEAN fgIsInfraChannelFinished; /* 連線建立的頻道流程是否完成 */

    /* 連線狀態 */
    BOOLEAN fgIsConnReqIssued;  /* 等同 CONNECTION_SETTINGS_T.fgIsConnReqIssued */
    BOOLEAN fgIsScanReqIssued;  /* 是否有 scan in-flight */
    BOOLEAN fgIsDisconnectedByNonRequest; /* 非 user space 觸發的斷線 */
    UINT_8  ucConnTrialCount;   /* 連線重試次數 */
    UINT_8  ucScanSSIDNum;
    PARAM_SSID_T arScanSSID[CFG_SCAN_SSID_MAX_NUM];
    UINT_32      u4ScanIELength;
    UINT_8       aucScanIEBuf[MAX_IE_LENGTH];
    UINT_8       ucScanChannelListNum;
    RF_CHANNEL_INFO_T arScanChnlInfoList[MAXIMUM_OPERATION_CHANNEL_LIST];

    /* 目標 AP */
    P_BSS_DESC_T    prTargetBssDesc;   /* SEARCH state 找到的 AP */
    P_STA_RECORD_T  prTargetStaRec;    /* join 目標的 STA record */

    /* 計時 */
    OS_SYSTIME  rJoinReqTime;  /* connect 請求發出的時間，用於整體 timeout 計算 */

    /* Pending request queue */
    LINK_T  rPendingReqList;   /* 排隊等待的請求（LINK_T 是 MTK 的雙向鏈結串列） */

    /* Timers */
    TIMER_T rBGScanTimer;          /* Background scan 週期 timer */
    TIMER_T rScanDoneTimer;        /* scan watchdog timer */
    TIMER_T rJoinTimeoutTimer;     /* join（auth+assoc）timeout timer */
    TIMER_T rChannelTimeoutTimer;  /* remain_on_channel timeout timer */
    TIMER_T rDeauthDoneTimer;      /* deauth 送出後等待完成的 timer */
} AIS_FSM_INFO_T;
```

`TIMER_T` 是 MTK 的軟體 timer 型別，使用 `cnmTimerStartTimer()` / `cnmTimerStopTimer()` 操作。軟體 timer 在 `main_thread` 中執行 callback（非 hard interrupt context），因此 callback 可以呼叫需要 sleep 的函式。

`LINK_T`（`{ P_LINK_ENTRY_T prNext; P_LINK_ENTRY_T prPrev; UINT_32 u4NumElem; }`）是 MTK 的雙向鏈結串列，`rPendingReqList` 儲存排隊等待的 `AIS_REQ_HDR_T` 物件。

### 1.2 `ENUM_AIS_STATE_T`（共 15 個）

```c
typedef enum _ENUM_AIS_STATE_T {
    AIS_STATE_IDLE = 0,
    AIS_STATE_SEARCH,
    AIS_STATE_SCAN,
    AIS_STATE_ONLINE_SCAN,
    AIS_STATE_LOOKING_FOR,
    AIS_STATE_WAIT_FOR_NEXT_SCAN,
    AIS_STATE_REQ_CHANNEL_JOIN,
    AIS_STATE_JOIN,
    AIS_STATE_JOIN_FAILURE,
    AIS_STATE_IBSS_ALONE,
    AIS_STATE_IBSS_MERGE,
    AIS_STATE_NORMAL_TR,
    AIS_STATE_DISCONNECTING,
    AIS_STATE_REQ_REMAIN_ON_CHANNEL,
    AIS_STATE_REMAIN_ON_CHANNEL,
    AIS_STATE_NUM
} ENUM_AIS_STATE_T;
```

各 state 的 debug 字串定義在 `ais_fsm.c` 開頭的 `apucDebugAisState[]` 陣列，可在 log 中看到如 `[AIS_STATE_JOIN]` 的字串。

### 1.3 `ENUM_AIS_REQUEST_TYPE_T`

```c
typedef enum _ENUM_AIS_REQUEST_TYPE_T {
    AIS_REQUEST_SCAN = 0,
    AIS_REQUEST_RECONNECT,
    AIS_REQUEST_ROAMING_SEARCH,
    AIS_REQUEST_ROAMING_CONNECT,
    AIS_REQUEST_REMAIN_ON_CHANNEL,
    AIS_REQUEST_NUM
} ENUM_AIS_REQUEST_TYPE_T;
```

這些請求被排入 `rPendingReqList`，在適當時機（通常是進入 `AIS_STATE_IDLE` 或 `AIS_STATE_NORMAL_TR`）被取出並處理。

---

## 2. 初始化

### 2.1 `aisInitializeConnectionSettings()`

在 `wlanProbe()` 流程中呼叫，設定連線參數的預設值：

```c
prConnSettings->eOPMode         = NET_TYPE_INFRA;
prConnSettings->eConnectionPolicy = CONNECT_BY_SSID_BEST_RSSI;
prConnSettings->eAuthMode       = AUTH_MODE_OPEN;
prConnSettings->eEncStatus      = ENUM_ENCRYPTION_DISABLED;
prConnSettings->fgIsConnReqIssued  = FALSE;
prConnSettings->fgIsScanReqIssued  = FALSE;
```

### 2.2 `aisFsmInit()`

```c
VOID aisFsmInit(P_ADAPTER_T prAdapter)
```

- 初始化 `prAisFsmInfo->eCurrentState = AIS_STATE_IDLE`
- 所有 seq number 歸零
- 初始化所有 timer（`cnmTimerInitTimer()`，傳入 callback function pointer）
- 初始化 `rPendingReqList`（`LINK_INITIALIZE`）

---

## 3. 核心引擎：`aisFsmSteps()`（`ais_fsm.c:849`）

```c
VOID aisFsmSteps(
    P_ADAPTER_T         prAdapter,
    ENUM_AIS_STATE_T    eNextState
)
```

所有 AIS state 切換都透過此函式。函式以 do-while 迴圈允許一次呼叫內的連續 state 切換（例如 `IDLE → SEARCH → REQ_CHANNEL_JOIN`），由 `fgIsTransition` 旗標控制是否繼續迴圈。

```c
do {
    fgIsTransition = FALSE;
    ePreviousState = prAisFsmInfo->eCurrentState;
    prAisFsmInfo->eCurrentState = eNextState;

    /* 印 "[ePreviousState] -> [eCurrentState]" log */
    DBGLOG(AIS, STATE, "[%s] -> [%s]\n",
           apucDebugAisState[ePreviousState],
           apucDebugAisState[prAisFsmInfo->eCurrentState]);

    /* 若 old state 不是 NORMAL_TR，保持 network active */
    if (ePreviousState != AIS_STATE_NORMAL_TR)
        nicActivateNetwork(prAdapter, ...);

    switch (prAisFsmInfo->eCurrentState) {
    case AIS_STATE_IDLE:     ...
    case AIS_STATE_SEARCH:   ...
    /* ... */
    }
} while (fgIsTransition);
```

**重要**：進入新 state 的 `switch case` 中，若要立即跳到下一個 state，設定 `eNextState = [新 state]; fgIsTransition = TRUE;` 即可。不要直接遞迴呼叫 `aisFsmSteps()`，會導致 stack 累積。

---

## 4. 各 State 的 Entry Action 詳細說明

### 4.1 `AIS_STATE_IDLE`（行 902~979）

**行 910~914**：

```c
cnmTimerStopTimer(prAdapter, &prAisFsmInfo->rScanDoneTimer);
prAisFsmInfo->fgIsScanning = FALSE;
```

**行 916~956**：`prAisReq = aisFsmGetNextRequest(prAdapter)` 從 `rPendingReqList` 取出下一個請求：

```c
switch (prAisReq->eReqType) {
case AIS_REQUEST_SCAN:
    eNextState = AIS_STATE_SCAN;
    fgIsTransition = TRUE;
    break;

case AIS_REQUEST_RECONNECT:
    if (prConnSettings->fgIsConnReqIssued) {
        eNextState = AIS_STATE_SEARCH;
        fgIsTransition = TRUE;
    }
    break;

case AIS_REQUEST_ROAMING_SEARCH:
case AIS_REQUEST_ROAMING_CONNECT:
    eNextState = AIS_STATE_SEARCH;
    fgIsTransition = TRUE;
    break;

case AIS_REQUEST_REMAIN_ON_CHANNEL:
    eNextState = AIS_STATE_REQ_REMAIN_ON_CHANNEL;
    fgIsTransition = TRUE;
    break;
}
cnmMemFree(prAdapter, prAisReq);  /* 取出後立即釋放 */
```

**行 957~971**：若 `rPendingReqList` 為空，但 `prConnSettings->fgIsConnReqIssued == TRUE`，仍進入 `SEARCH`（處理初次 connect 請求，此時 pending queue 為空，請求是直接由 `wlanoidSetConnect` 設定的）。

### 4.2 `AIS_STATE_SEARCH`（行 980~1086）

**行 983~985**：

```c
prBssDesc = scanSearchBssDescByPolicy(prAdapter, NETWORK_TYPE_AIS_INDEX);
```

`scanSearchBssDescByPolicy()` 遍歷 driver 的 BSS 資料庫（由歷次 scan 結果累積），依 `rConnSettings` 的 `eConnectionPolicy`、`aucSSID`、`aucBSSID`、`u4FreqInKHz` 篩選，對符合條件的 AP 依 RSSI 排序，回傳最佳候選者。

**行 986~1034**：

找到（`prBssDesc != NULL`）：
```c
prAisFsmInfo->prTargetBssDesc = prBssDesc;
/* 更新 RSN 相關欄位（AKM suite、pairwise cipher 等），供 SAA 使用 */
aisSetWpaRsneForTargetBss(prAdapter, prBssDesc);
eNextState = AIS_STATE_REQ_CHANNEL_JOIN;
fgIsTransition = TRUE;
```

找不到（`prBssDesc == NULL`）：
```c
aisFsmStateSearchAction(prAdapter, SEARCH_ACTION_SEARCH_RESULT_EMPTY);
/* aisFsmStateSearchAction 根據 ucConnTrialCount、fgTryScan 等決定：
   - 進 AIS_STATE_LOOKING_FOR（發起新 scan 找 SSID）
   - 進 AIS_STATE_WAIT_FOR_NEXT_SCAN（等一段時間再掃）
   - 進 AIS_STATE_JOIN_FAILURE（超過重試限制） */
```

### 4.3 `AIS_STATE_SCAN` / `AIS_STATE_ONLINE_SCAN` / `AIS_STATE_LOOKING_FOR`

這三個 state 的 entry action 都發起掃描，差別在於 scan 參數和完成後的目標 state：

| State | scan 目的 | 完成後目標 |
|---|---|---|
| `SCAN` | 從 IDLE 出發的一般 scan | `IDLE` |
| `ONLINE_SCAN` | 已連線時的 scan（`NL80211_CMD_TRIGGER_SCAN`） | `NORMAL_TR` |
| `LOOKING_FOR` | 漫遊搜尋或找不到目標 AP 時的指定 SSID scan | `SEARCH` |

```c
/* 分配 MSG_SCN_SCAN_REQ_V2_T */
prMsgScanReq->ucSeqNum = ++prAisFsmInfo->ucSeqNumOfScanReq;
prMsgScanReq->ucSSIDType = ...;   /* SCAN_REQ_SSID_WILDCARD 或 SCAN_REQ_SSID_SPECIFIED */
prMsgScanReq->ucChannelListNum = prAisFsmInfo->ucScanChannelListNum;
/* 複製 IE */
kalMemCopy(prMsgScanReq->aucIEBuf, prAisFsmInfo->aucScanIEBuf,
           prAisFsmInfo->u4ScanIELength);
prAisFsmInfo->fgIsScanReqIssued = TRUE;
mboxSendMsg(prAdapter, MBOX_ID_0, MID_AIS_SCN_SCAN_REQ_V2,
            MSG_SEND_METHOD_BUF);
```

### 4.4 `AIS_STATE_WAIT_FOR_NEXT_SCAN`

```c
cnmTimerStartTimer(prAdapter, &prAisFsmInfo->rBGScanTimer,
                   AIS_BG_SCAN_INTERVAL_MIN_SEC * 1000);
```

等待 `rBGScanTimer` 到期後，`aisFsmRunEventBGSleepTimeOut()` 觸發，進入 `AIS_STATE_LOOKING_FOR`。`AIS_BG_SCAN_INTERVAL_MIN_SEC` 通常為 1~2 秒，避免過於頻繁的掃描消耗電量。

### 4.5 `AIS_STATE_REQ_CHANNEL_JOIN`（行 1111~1158）

```c
/* 分配 MSG_CH_REQ_T，設定參數 */
prMsgChReq->ucTokenID      = ++prAisFsmInfo->ucSeqNumOfChReq;
prMsgChReq->eReqType       = CH_REQ_TYPE_JOIN;
prMsgChReq->ucPrimaryChannel = prBssDesc->ucChannelNum;
prMsgChReq->eBand          = prBssDesc->eBand;
prMsgChReq->eRfSco         = prBssDesc->eSco;   /* secondary channel offset */
prMsgChReq->eRfBw          = prBssDesc->eBandwidth;
prMsgChReq->u4MaxInterval  = AIS_JOIN_CH_REQUEST_INTERVAL; /* ms */

mboxSendMsg(prAdapter, MBOX_ID_0, MID_MNY_CNM_CH_REQ, MSG_SEND_METHOD_BUF);
prAisFsmInfo->fgIsChannelRequested = TRUE;
```

`eRfSco`（Secondary Channel Offset）和 `eRfBw`（RF Bandwidth）是 802.11n/ac 頻道綁定相關參數。40 MHz 模式需要知道第二個 20 MHz 頻道在主頻道的上方還是下方（`SCO_SCA` = above，`SCO_SCB` = below）。

### 4.6 `AIS_STATE_JOIN`（行 1163~1188）

```c
aisFsmStateInit_JOIN(prAdapter, prAisFsmInfo->prTargetBssDesc);
```

`aisFsmStateInit_JOIN()` 內部（見 `03_connect.md` 第 4.4 節）：分配/取得 STA record，設定 auth algorithm，送 `MID_AIS_SAA_FSM_START`。

### 4.7 `AIS_STATE_JOIN_FAILURE`

```c
nicMediaJoinFailure(prAdapter, NETWORK_TYPE_AIS_INDEX,
                    WLAN_STATUS_FAILURE);
/* nicMediaJoinFailure 呼叫 kalIndicateStatusAndComplete(WLAN_STATUS_JOIN_FAILURE) */
eNextState = AIS_STATE_IDLE;
fgIsTransition = TRUE;
```

### 4.8 `AIS_STATE_NORMAL_TR`（行 1285~1374）

```c
/* 保護連線建立中的頻道流程 */
if (!prAisFsmInfo->fgIsInfraChannelFinished)
    break;

/* 按優先序處理 pending 請求 */
prAisReq = aisFsmGetNextRequest(prAdapter);
if (prAisReq) {
    switch (prAisReq->eReqType) {
    case AIS_REQUEST_ROAMING_SEARCH:
        eNextState = AIS_STATE_LOOKING_FOR;  break;
    case AIS_REQUEST_ROAMING_CONNECT:
        eNextState = AIS_STATE_SEARCH;       break;
    case AIS_REQUEST_SCAN:
        eNextState = AIS_STATE_ONLINE_SCAN;  break;
    case AIS_REQUEST_REMAIN_ON_CHANNEL:
        eNextState = AIS_STATE_REQ_REMAIN_ON_CHANNEL; break;
    }
    fgIsTransition = TRUE;
    cnmMemFree(prAdapter, prAisReq);
}
```

Roaming 請求（`ROAMING_SEARCH`、`ROAMING_CONNECT`）優先於一般 scan，確保漫遊決策不被普通掃描延誤。

### 4.9 `AIS_STATE_DISCONNECTING`

```c
/* 若目前有 BSS 連線，發送 deauthentication frame */
if (prAisBssInfo->eConnectionState == MEDIA_STATE_CONNECTED) {
    /* 組建 deauth frame，設定 reason code */
    authSendDeauthFrame(prAdapter, prAisBssInfo, prStaRec,
                        NULL, REASON_CODE_DEAUTH_LEAVING_BSS, ...);
    cnmTimerStartTimer(prAdapter, &prAisFsmInfo->rDeauthDoneTimer,
                       AIS_DEAUTH_TIMEOUT_TIME);
}
```

等 deauth frame 送出確認（或 timer 超時），`aisFsmRunEventDeauthDone()` 再呼叫 `aisFsmDisconnect()` 完成斷線流程。

### 4.10 `AIS_STATE_REQ_REMAIN_ON_CHANNEL`

```c
/* 向 CNM 申請 remain on channel */
prMsgChReq->eReqType = CH_REQ_TYPE_ROC;   /* ROC = Remain On Channel */
prMsgChReq->ucTokenID = ++prAisFsmInfo->ucSeqNumOfChReq;
prMsgChReq->ucPrimaryChannel = prAisFsmInfo->rChReqInfo.ucChannelNum;
prMsgChReq->u4MaxInterval = prAisFsmInfo->rChReqInfo.u4DurationMs;
mboxSendMsg(..., MID_MNY_CNM_CH_REQ, ...);
```

Remain On Channel 是 cfg80211 提供的功能，讓 driver 停留在指定頻道一段時間（例如 P2P device discovery，或 `iw dev wlan0 remain-on-channel` 命令）。

### 4.11 `AIS_STATE_REMAIN_ON_CHANNEL`

```c
/* 通知 user space 已進入指定頻道 */
kalReadyOnChannel(prGlueInfo, prChGrant->u8Cookie,
                  prChGrant->eBand, prChGrant->eRfSco,
                  prChGrant->ucPrimaryChannel,
                  prChGrant->u4GrantInterval);
/* 啟動 channel timeout timer */
cnmTimerStartTimer(prAdapter, &prAisFsmInfo->rChannelTimeoutTimer,
                   prChGrant->u4GrantInterval);
```

---

## 5. 關鍵事件處理函式

### 5.1 `aisFsmRunEventScanDone()`（行 1527）

見 `02_scan.md` 第 6 節。

### 5.2 `aisFsmRunEventAbort()`（行 1618）

見 `03_connect.md` 第 3 節。

### 5.3 `aisFsmRunEventJoinComplete()`（行 1834）

見 `03_connect.md` 第 5 節。

### 5.4 `aisFsmRunEventChGrant()`（`ais_fsm.c:3377`）

```c
VOID aisFsmRunEventChGrant(
    P_ADAPTER_T prAdapter,
    P_MSG_HDR_T prMsgHdr   /* MSG_CH_GRANT_T */
)
```

CNM 核可頻道請求後觸發。

**行為：**

```c
if (prAisFsmInfo->ucSeqNumOfChReq != prMsgChGrant->ucTokenID) {
    /* token mismatch，已過時的 grant，還給 CNM */
    aisFsmReleaseCh(prAdapter);
    return;
}

if (eCurrentState == AIS_STATE_REQ_CHANNEL_JOIN) {
    prAisFsmInfo->fgIsChannelGranted = TRUE;
    cnmTimerStartTimer(prAdapter, &prAisFsmInfo->rJoinTimeoutTimer,
                       prMsgChGrant->u4GrantInterval);
    aisFsmSteps(prAdapter, AIS_STATE_JOIN);

} else if (eCurrentState == AIS_STATE_REQ_REMAIN_ON_CHANNEL) {
    prAisFsmInfo->fgIsChannelGranted = TRUE;
    aisFsmSteps(prAdapter, AIS_STATE_REMAIN_ON_CHANNEL);
    /* REMAIN_ON_CHANNEL entry action 呼叫 kalReadyOnChannel() */

} else {
    /* 不在預期 state，還給 CNM */
    aisFsmReleaseCh(prAdapter);
}
```

`u4GrantInterval` 是 CNM 實際核准的頻道使用時間（ms），可能小於請求的 `u4MaxInterval`（CNM 可能因其他原因縮短）。

### 5.5 `aisFsmReleaseCh()`（`ais_fsm.c:3460`）

```c
VOID aisFsmReleaseCh(P_ADAPTER_T prAdapter)
```

釋放目前持有的 CNM 頻道授權：

```c
if (prAisFsmInfo->fgIsChannelRequested || prAisFsmInfo->fgIsChannelGranted) {
    /* 組建 MSG_CH_ABORT_T */
    prMsgChAbort->ucTokenID = prAisFsmInfo->ucSeqNumOfChReq;
    mboxSendMsg(prAdapter, MBOX_ID_0, MID_MNY_CNM_CH_ABORT, MSG_SEND_METHOD_BUF);
    prAisFsmInfo->fgIsChannelRequested = FALSE;
    prAisFsmInfo->fgIsChannelGranted = FALSE;
}
```

此函式在多個中止路徑中呼叫，確保 CNM 頻道資源不洩漏。若不釋放，後續的 join/roc 請求可能永遠無法取得頻道授權。

---

## 6. Timeout 處理

### 6.1 `aisFsmRunEventScanDoneTimeOut()`

`rScanDoneTimer` 到期時觸發。

**行為：**

1. `aisFsmStateAbort_SCAN(prAdapter)` — 送 `MID_AIS_SCN_SCAN_CANCEL` 給 SCN FSM
2. 強制呼叫 `kalScanDone()` — 必須呼叫，否則 cfg80211 的 `prScanRequest` 永遠不會被清除，導致後續所有 scan 都回傳 `-EBUSY`
3. 清除 `prGlueInfo->prScanRequest = NULL`
4. `prAisFsmInfo->fgIsScanReqIssued = FALSE`
5. 依 current state 轉回 `AIS_STATE_IDLE` 或 `AIS_STATE_NORMAL_TR`

### 6.2 `aisFsmRunEventJoinTimeout()`（行 3068）

`rJoinTimeoutTimer` 到期時觸發（在 `AIS_STATE_JOIN` 進入時啟動）。

**行為：**

1. 若 current state 不是 `AIS_STATE_JOIN`：不處理（state 已切換）
2. `aisFsmStateAbort_JOIN(prAdapter)` — 送 `MID_AIS_SAA_FSM_ABORT` 給 SAA FSM，中止 auth/assoc
3. `prAisFsmInfo->ucConnTrialCount++`（`ucJoinFailureCount`）
4. `aisFsmReleaseCh(prAdapter)` — 釋放頻道
5. 依 trial count 和整體時間決定：
   - trial count 未超限且整體時間未超：`aisFsmInsertRequest(AIS_REQUEST_RECONNECT)`，進 `IDLE` 重試
   - 整體時間超過 `AIS_JOIN_TIMEOUT`：進 `AIS_STATE_JOIN_FAILURE`

### 6.3 `aisFsmRunEventBGSleepTimeOut()`

`rBGScanTimer` 到期（在 `AIS_STATE_WAIT_FOR_NEXT_SCAN` 啟動）：

```c
aisFsmSteps(prAdapter, AIS_STATE_LOOKING_FOR);
```

### 6.4 `aisFsmRunEventChannelTimeout()`（行 4017）

`rChannelTimeoutTimer` 到期（remain on channel 超時）：

```c
aisFsmReleaseCh(prAdapter);
kalRemainOnChannelExpired(prGlueInfo, ...);  /* 通知 user space */
/* 依 previous state 回到 NORMAL_TR 或 IDLE */
```

---

## 7. `aisFsmStateAbort()`（行 1690）

```c
VOID aisFsmStateAbort(
    P_ADAPTER_T prAdapter,
    UINT_8      ucReasonOfDisconnect,
    BOOLEAN     fgDelayIndication   /* 是否延遲 disconnect indication */
)
```

此函式從多處呼叫（`aisFsmRunEventAbort` 等），根據 `eCurrentState` 執行對應的中止動作：

| Current State | 中止動作 |
|---|---|
| `AIS_STATE_SCAN` | 送 `MID_AIS_SCN_SCAN_CANCEL`；將 scan 請求放回 pending queue（可能再次執行） |
| `AIS_STATE_ONLINE_SCAN` | 同上 |
| `AIS_STATE_REQ_CHANNEL_JOIN` | `aisFsmReleaseCh()`（還頻道給 CNM） |
| `AIS_STATE_JOIN` | `aisFsmStateAbort_JOIN()`（送 `MID_AIS_SAA_FSM_ABORT`）；`aisFsmReleaseCh()` |
| `AIS_STATE_LOOKING_FOR` | 送 `MID_AIS_SCN_SCAN_CANCEL` |
| `AIS_STATE_REQ_REMAIN_ON_CHANNEL` | `aisFsmReleaseCh()` |
| `AIS_STATE_REMAIN_ON_CHANNEL` | `aisFsmReleaseCh()`；停 `rChannelTimeoutTimer`；通知 user space |
| `AIS_STATE_NORMAL_TR`（已連線） | 視 `fgDelayIndication`：立即或延遲後進入 `DISCONNECTING`，或直接 `aisFsmDisconnect()` |

**行 1808~1817**（已連線時）：

```c
if (prAisBssInfo->eConnectionState == MEDIA_STATE_CONNECTED) {
    if (fgDelayIndication) {
        /* 發 deauth，進 DISCONNECTING */
        aisFsmSteps(prAdapter, AIS_STATE_DISCONNECTING);
        return;  /* disconnect indication 延後 */
    } else {
        /* 直接斷線 */
        aisFsmDisconnect(prAdapter, fgDelayIndication);
    }
}
```

**行 1819**：`aisFsmDisconnect(prAdapter, fgDelayIndication)` — 統一的斷線收尾：

```c
/* 清除 BSS 連線狀態 */
aisChangeMediaState(prAdapter, MEDIA_STATE_DISCONNECTED);
/* 通知 user space */
if (!fgDelayIndication) {
    kalIndicateStatusAndComplete(prGlueInfo,
        WLAN_STATUS_MEDIA_DISCONNECT, NULL, 0);
}
/* 清除各種連線相關資源 */
cnmStaRecChangeState(prAdapter, prStaRec, STA_STATE_1);
aisFsmSteps(prAdapter, AIS_STATE_IDLE);
```

---

## 8. Pending Request Queue 操作

### `aisFsmInsertRequest()`（行 3875）

```c
VOID aisFsmInsertRequest(
    P_ADAPTER_T             prAdapter,
    ENUM_AIS_REQUEST_TYPE_T eReqType
)
```

從 cnm memory pool 分配 `AIS_REQ_HDR_T`，設定 `eReqType`，插入 `rPendingReqList` 尾部（FIFO 順序）。

### `aisFsmGetNextRequest()`（行 3851）

```c
P_AIS_REQ_HDR_T aisFsmGetNextRequest(P_ADAPTER_T prAdapter)
```

從 `rPendingReqList` 頭部取出一個請求（FIFO）。取出後**不釋放記憶體**，呼叫端取出並處理後須呼叫 `cnmMemFree()` 釋放。

### `aisFsmIsRequestPending()`（行 3814）

```c
BOOLEAN aisFsmIsRequestPending(
    P_ADAPTER_T             prAdapter,
    ENUM_AIS_REQUEST_TYPE_T eReqType,
    BOOLEAN                 fgRemove   /* TRUE：找到後從 queue 移除 */
)
```

遍歷 `rPendingReqList` 尋找指定 type 的請求，`fgRemove = TRUE` 時找到後移除並釋放。

### `aisFsmFlushRequest()`

清空整個 `rPendingReqList`，釋放所有 `AIS_REQ_HDR_T`。在 driver 卸載或重大狀態重置時呼叫。

---

## 9. Mailbox 訊息 ID 速查

| Message ID | 方向 | 觸發位置 | 說明 |
|---|---|---|---|
| `MID_OID_AIS_FSM_JOIN_REQ` | OID → AIS | `wlanoidSetConnect()` / `wlanoidSetDisassociate()` | connect / disconnect / abort |
| `MID_AIS_SCN_SCAN_REQ_V2` | AIS → SCN | `aisFsmSteps()` 的 SCAN/ONLINE_SCAN/LOOKING_FOR entry | 啟動掃描 |
| `MID_AIS_SCN_SCAN_CANCEL` | AIS → SCN | `aisFsmStateAbort_SCAN()` | 取消掃描 |
| `MID_MNY_CNM_CH_REQ` | AIS → CNM | `aisFsmSteps()` 的 REQ_CHANNEL_JOIN/REQ_ROC entry | 申請頻道授權 |
| `MID_MNY_CNM_CH_ABORT` | AIS → CNM | `aisFsmReleaseCh()` | 歸還頻道 |
| `MID_AIS_SAA_FSM_START` | AIS → SAA | `aisFsmStateInit_JOIN()` | 啟動 auth/assoc |
| `MID_AIS_SAA_FSM_ABORT` | AIS → SAA | `aisFsmStateAbort_JOIN()` | 中止 auth/assoc |
| `MID_AIS_SAA_FSM_COMP` | SAA → AIS | SAA FSM 完成時 | join complete（觸發 `aisFsmRunEventJoinComplete`） |
| `MID_SCN_AIS_SCAN_DONE` | SCN → AIS | SCN FSM 收到 firmware scan done event | scan done（觸發 `aisFsmRunEventScanDone`） |
| `MID_CNM_AIS_CH_GRANT` | CNM → AIS | CNM 核准頻道請求時 | channel grant（觸發 `aisFsmRunEventChGrant`） |

---

## 10. 函式索引

| 函式 | 行號 |
|---|---|
| `aisFsmInit` | — |
| `aisInitializeConnectionSettings` | — |
| `aisFsmSteps` | 849 |
| `aisFsmStateSearchAction` | — |
| `aisFsmStateInit_JOIN` | — |
| `aisFsmScanRequestAdv` | 3290 |
| `aisFsmRunEventScanDone` | 1527 |
| `aisFsmRunEventScanDoneTimeOut` | — |
| `aisFsmRunEventAbort` | 1618 |
| `aisFsmStateAbort` | 1690 |
| `aisFsmStateAbort_JOIN` | — |
| `aisFsmStateAbort_SCAN` | — |
| `aisFsmDisconnect` | — |
| `aisFsmRunEventJoinComplete` | 1834 |
| `aisFsmJoinCompleteAction` | 1869 |
| `aisFsmRunEventJoinTimeout` | 3068 |
| `aisFsmRunEventChGrant` | 3377 |
| `aisFsmReleaseCh` | 3460 |
| `aisFsmRunEventBGSleepTimeOut` | — |
| `aisFsmRunEventChannelTimeout` | 4017 |
| `aisFsmInsertRequest` | 3875 |
| `aisFsmGetNextRequest` | 3851 |
| `aisFsmIsRequestPending` | 3814 |
| `aisFsmFlushRequest` | — |
| `aisIndicationOfMediaStateToHost` | — |
| `aisChangeMediaState` | — |
| `aisUpdateBssInfoForJOIN` | 2436 |
