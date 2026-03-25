# MTK Wi-Fi Connect 流程參考

主要檔案：`gl_cfg80211.c`、`gl_kal.c`、`wlan_oid.c`、`ais_fsm.c`

---

## 0. 流程總覽

```
wpa_supplicant 送出 NL80211_CMD_CONNECT
  → cfg80211 框架呼叫 .connect op
  → mtk_cfg80211_connect()           [gl_cfg80211.c:900]
      1. kalIoctl(wlanoidSetInfrastructureMode)
      2. 解析 sme->crypto → eAuthMode / eEncStatus / u4AkmSuite
      3. 解析 sme->ie → WPS/RSN/HS20 IE
      4. kalIoctl(wlanoidSetAuthMode)
      5. kalIoctl(wlanoidSetEncryptionStatus)
      6. kalIoctl(wlanoidSetConnect, PARAM_CONNECT_T)

  → wlanoidSetConnect()              [wlan_oid.c:1042]
      — 寫入 rConnSettings（SSID/BSSID/policy/freq）
      — 判斷是否為 re-association
      — 設 fgIsConnReqIssued = TRUE
      — mboxSendMsg(MID_OID_AIS_FSM_JOIN_REQ)

  → AIS FSM 收到 MID_OID_AIS_FSM_JOIN_REQ
      → aisFsmRunEventAbort()        [ais_fsm.c:1618]
      → IDLE → SEARCH → REQ_CHANNEL_JOIN → JOIN

  → SAA FSM 執行 auth + assoc
  → SAA 完成後發 MID_AIS_SAA_FSM_COMP

  → aisFsmRunEventJoinComplete()     [ais_fsm.c:1834]
      → aisFsmJoinCompleteAction()

      成功：aisIndicationOfMediaStateToHost(CONNECTED)
            → kalIndicateStatusAndComplete(WLAN_STATUS_MEDIA_CONNECT)
            → cfg80211_connect_result()

      失敗：retry 或 AIS_STATE_JOIN_FAILURE
```

---

## 1. `mtk_cfg80211_connect()`（`gl_cfg80211.c:900`）

```c
int mtk_cfg80211_connect(struct wiphy *wiphy,
                         struct net_device *ndev,
                         struct cfg80211_connect_params *sme)
```

**參數：**

`sme`（`struct cfg80211_connect_params *`，即 station management entity params）：cfg80211 傳入的連線參數，包含：

```c
struct cfg80211_connect_params {
    struct ieee80211_channel *channel;  /* 指定頻道，NULL 表示任意 */
    const u8    *bssid;                 /* 指定 BSSID，NULL 表示任意 */
    const u8    *ssid;
    size_t       ssid_len;
    enum nl80211_auth_type auth_type;   /* OPEN_SYSTEM / SHARED_KEY / AUTO */
    const u8    *ie;                    /* RSN IE、WPS IE 等 */
    size_t       ie_len;
    struct cfg80211_crypto_settings crypto;
    const u8    *key;                   /* WEP key，現代加密不用 */
    /* ... */
};

struct cfg80211_crypto_settings {
    u32 wpa_versions;      /* NL80211_WPA_VERSION_1 / _2 */
    u32 cipher_group;      /* 群組加密套件（CCMP/TKIP/WEP 等） */
    int n_ciphers_pairwise;
    u32 ciphers_pairwise[5]; /* 單播加密套件 */
    int n_akm_suites;
    u32 akm_suites[2];     /* WLAN_AKM_SUITE_PSK / _8021X 等 */
    /* ... */
};
```

**行為（行 900~1248）：**

### 1.1 基本驗證（行 909~930）

**行 909~920**：空指標防呆（`prGlueInfo`、`prAdapter`、`prNetDev` 等）。

**行 922~930**：`kalIoctl(prGlueInfo, wlanoidSetInfrastructureMode, ...)`。將裝置 operation mode 設為 `NET_TYPE_INFRA`（Infrastructure，即 STA 連 AP 模式）。這是必要的，因為 driver 也支援 Ad-hoc 模式（`NET_TYPE_IBSS`），每次 connect 前明確設定確保模式正確。

### 1.2 WPA 版本與認證模式解析（行 970~1084）

**行 970~975**：從 `sme->crypto.wpa_versions` 判斷 WPA 版本：

```c
if (sme->crypto.wpa_versions & NL80211_WPA_VERSION_2)
    /* WPA2/RSN */
else if (sme->crypto.wpa_versions & NL80211_WPA_VERSION_1)
    /* WPA */
else
    /* 無 WPA（開放或 WEP） */
```

**行 977~987**：解析 `sme->auth_type`（`enum nl80211_auth_type`）：

| `nl80211_auth_type` 值 | 含義 | driver 行為 |
|---|---|---|
| `NL80211_AUTHTYPE_OPEN_SYSTEM` | 開放系統認證（802.11 標準 auth） | 設 `AUTH_MODE_OPEN` |
| `NL80211_AUTHTYPE_SHARED_KEY` | 共享金鑰認證（僅用於 WEP） | 設 `AUTH_MODE_SHARED` |
| `NL80211_AUTHTYPE_AUTOMATIC` | 自動選擇 | driver 嘗試 open，失敗再試 shared |

**行 989~1014**：解析 pairwise cipher（`sme->crypto.ciphers_pairwise[]`）。Pairwise cipher 是 STA 與 AP 之間單播資料的加密套件。

| `WLAN_CIPHER_SUITE_*` | `eEncStatus` |
|---|---|
| `WLAN_CIPHER_SUITE_CCMP`（AES-CCMP） | `ENUM_ENCRYPTION3_ENABLED` |
| `WLAN_CIPHER_SUITE_TKIP` | `ENUM_ENCRYPTION2_ENABLED` |
| `WLAN_CIPHER_SUITE_WEP40` / `_WEP104` | `ENUM_ENCRYPTION1_ENABLED` |
| 無 | `ENUM_ENCRYPTION_DISABLED` |

**行 1016~1038**：解析 group cipher（`sme->crypto.cipher_group`）。Group cipher 是廣播/多播資料的加密套件，通常與 pairwise cipher 相同或更低安全等級。

**行 1040~1084**：解析 AKM suites（`sme->crypto.akm_suites[]`）。AKM（Authentication and Key Management）定義認證方式和金鑰產生方式。

| `WLAN_AKM_SUITE_*` | `eAuthMode` |
|---|---|
| `WLAN_AKM_SUITE_PSK`（Pre-Shared Key） | `AUTH_MODE_WPA2_PSK` 或 `AUTH_MODE_WPA_PSK` |
| `WLAN_AKM_SUITE_8021X`（Enterprise） | `AUTH_MODE_WPA2` 或 `AUTH_MODE_WPA` |
| `WLAN_AKM_SUITE_FT_PSK`（Fast BSS Transition + PSK） | `AUTH_MODE_WPA2_FT_PSK` |
| 無（開放/WEP） | `AUTH_MODE_OPEN` 或 `AUTH_MODE_AUTO_SWITCH` |

### 1.3 IE 解析（行 1098~1163）

`sme->ie` 是 `wpa_supplicant` 附帶的 raw IE bytes，可能包含多個 IE，依 Element ID 識別。

**行 1098~1163**：逐一解析 `sme->ie` 中的 IE：

- **WPS IE**（Element ID 221，OUI `00:50:F2:04`）：行 1115~1119，呼叫 `kalIoctl(wlanoidSetWSCAssocInfo, ...)` 將 WPS IE 內容傳給 driver。WPS（Wi-Fi Protected Setup）允許不輸入密碼的便捷配對。
- **RSN IE**（Element ID 48，Robust Security Network）：行 1151~1160，解析 RSN Capabilities 欄位，取出 MFP（Management Frame Protection）capability bits。MFP 保護 deauthentication 和 disassociation 等管理幀不被偽造。
- **HS20 IE**（Element ID 221，OUI `50:6F:9A:10`，Hotspot 2.0）和 **Interworking IE**（Element ID 107）：條件編譯（`CFG_SUPPORT_PASSPOINT`），為 Hotspot 2.0 自動連線使用。

### 1.4 下發 auth / encryption OID（行 1171~1212）

**行 1171~1174**：`kalIoctl(prGlueInfo, wlanoidSetAuthMode, &eAuthMode, sizeof(ENUM_AUTH_MODE_T), ...)`。

`wlanoidSetAuthMode()` 將 `eAuthMode` 寫入 `prAdapter->rWifiVar.rConnSettings.eAuthMode`，並根據 auth mode 同步更新相關的 RSN/RSNA 設定（例如設定 PMKID list、清除舊的 key 等）。

**行 1188~1212**：計算最終的 `eEncStatus`（根據 pairwise/group cipher 的組合），然後 `kalIoctl(prGlueInfo, wlanoidSetEncryptionStatus, &eEncStatus, sizeof(ENUM_ENCRYPTION_TYPE_T), ...)`。

`wlanoidSetEncryptionStatus()` 將 `eEncStatus` 寫入 `prAdapter->rWifiVar.rConnSettings.eEncStatus`，並設定相關的加密相關暫存器初始值。

### 1.5 最終 connect OID（行 1238~1248）

**行 1238~1241**：填充 `PARAM_CONNECT_T rNewSsid`（行號附近的命名有些誤導性，`rNewSsid` 實際是 connect 請求的參數容器，不只包含 SSID）：

```c
PARAM_CONNECT_T rNewSsid;
rNewSsid.u4CenterFreq = sme->channel ?
    sme->channel->center_freq : 0;          /* MHz，0 表示任意頻率 */
rNewSsid.pucBssid = (UINT_8 *)sme->bssid;  /* 可為 NULL */
rNewSsid.pucSsid  = (UINT_8 *)sme->ssid;
rNewSsid.u4SsidLen = sme->ssid_len;
```

**行 1242~1243**：

```c
rStatus = kalIoctl(prGlueInfo,
                   wlanoidSetConnect,
                   &rNewSsid,
                   sizeof(PARAM_CONNECT_T),
                   FALSE, FALSE, TRUE,  /* fgCmd = TRUE：走 CMD path */
                   &u4BufLen);
```

**行 1245~1248**：若 `rStatus != WLAN_STATUS_SUCCESS`，回傳 `-EINVAL`。

---

## 2. `wlanoidSetConnect()`（`wlan_oid.c:1042`）

```c
WLAN_STATUS wlanoidSetConnect(
    P_ADAPTER_T     prAdapter,
    PVOID           pvSetBuffer,   /* PARAM_CONNECT_T * */
    UINT_32         u4SetBufferLen,
    PUINT_32        pu4SetInfoLen
)
```

**行為（行 1042~1187）：**

### 2.1 入口驗證（行 1063~1076）

**行 1063~1069**：長度驗證（`u4SetBufferLen != sizeof(PARAM_CONNECT_T)`）和 ACPI D3 檢查。ACPI D3 狀態下回傳 `WLAN_STATUS_ADAPTER_NOT_READY`。

**行 1070~1076**：分配 `MSG_AIS_ABORT_T` 訊息物件（從 cnm memory pool，`cnmMemAlloc()`）。`MSG_AIS_ABORT_T` 是用於通知 AIS FSM 的訊息結構，message ID 為 `MID_OID_AIS_FSM_JOIN_REQ`。

```c
typedef struct _MSG_AIS_ABORT_T {
    MSG_HDR_T       rMsgHdr;    /* 通用訊息頭，含 message ID */
    UINT_8          ucReasonOfDisconnect;
    BOOLEAN         fgDelayIndication;
} MSG_AIS_ABORT_T;
```

### 2.2 寫入 connection settings（行 1086~1112）

**行 1086~1091**：清空舊的連線設定（`aucSSID`、`aucBSSID`、`eConnectionPolicy`）。每次 connect 都重建，不保留前次設定。

**行 1092~1100**：若 `prNewSsid->u4SsidLen > 0`（SSID 非空）：

```c
COPY_SSID(prConnSettings->aucSSID, prConnSettings->ucSSIDLen,
          prNewSsid->pucSsid, prNewSsid->u4SsidLen);
prConnSettings->eConnectionPolicy = CONNECT_BY_SSID_BEST_RSSI;
```

同時比較新舊 SSID 是否相同（`EQUAL_SSID`），結果存入 `fgEqualSsid`，用於後續 re-association 判斷。

**行 1101~1110**：若 `prNewSsid->pucBssid != NULL`（指定 BSSID）：

```c
COPY_MAC_ADDR(prConnSettings->aucBSSID, prNewSsid->pucBssid);
/* 驗證是否為 unicast MAC（第一個 byte 最低位為 0） */
if (!IS_UCAST_MAC_ADDR(prNewSsid->pucBssid)) {
    return WLAN_STATUS_INVALID_DATA;
}
prConnSettings->eConnectionPolicy = CONNECT_BY_BSSID;
prConnSettings->fgIsConnByBssidIssued = TRUE;
```

**行 1112**：

```c
prConnSettings->u4FreqInKHz = prNewSsid->u4CenterFreq * 1000;
/* sme->channel->center_freq 是 MHz，轉換為 KHz */
```

### 2.3 已連線時的處理（行 1116~1127）

若目前 `prAisBssInfo->eConnectionState == MEDIA_STATE_CONNECTED`（已連線）：

**行 1116~1122**：比較新舊 SSID 是否相同（`fgEqualSsid`）以及新舊 BSSID 是否相同（`fgEqualBssid`）。

- 若 SSID 相同（同一 ESS，Extended Service Set）：走 re-association 路徑，`prMsgAisAbort->fgDelayIndication = TRUE`（延遲回報斷線事件，避免 user space 看到短暫的斷線再連線）
- 若 SSID 不同（切換到新網路）：

  **行 1123**：`kalIndicateStatusAndComplete(prGlueInfo, WLAN_STATUS_MEDIA_DISCONNECT, NULL, 0)` — 先通知 user space 斷線（因為要連新的網路），再進行新連線。

### 2.4 送給 AIS FSM（行 1170~1182）

**行 1170~1174**：

```c
prConnSettings->fgIsConnReqIssued = TRUE;
```

標記有 connect 請求 in-flight。AIS FSM 在 `AIS_STATE_IDLE` 進入時檢查此 flag，若為 TRUE 則進入 `AIS_STATE_SEARCH`。

**行 1176~1180**：

```c
prMsgAisAbort->fgDelayIndication = fgDelayIndication;
prMsgAisAbort->ucReasonOfDisconnect = DISCONNECT_REASON_CODE_REASSOCIATION;
/* 或 DISCONNECT_REASON_CODE_NEW_CONNECTION，依情況 */
```

**行 1182**：

```c
mboxSendMsg(prAdapter, MBOX_ID_0,
            (P_MSG_HDR_T)prMsgAisAbort,
            MSG_SEND_METHOD_BUF);
```

`mboxSendMsg()` 將訊息放入 AIS FSM 的 mailbox queue。AIS FSM 透過 `aisFsmRunEventAbort()` 處理 `MID_OID_AIS_FSM_JOIN_REQ` 訊息。

**行 1187**：回傳 `WLAN_STATUS_SUCCESS`，`main_thread` 呼叫 `complete()`，`kalIoctl` 返回。此時 connect 流程尚未完成，只是「請求已交給 AIS FSM」。

---

## 3. AIS FSM 接收 connect 請求：`aisFsmRunEventAbort()`（`ais_fsm.c:1618`）

```c
VOID aisFsmRunEventAbort(
    P_ADAPTER_T prAdapter,
    P_MSG_HDR_T prMsgHdr   /* MSG_AIS_ABORT_T */
)
```

**行 1634~1638**：取出 `MSG_AIS_ABORT_T`，讀取 `ucReasonOfDisconnect` 和 `fgDelayIndication`，然後 `cnmMemFree()` 釋放訊息記憶體。

**行 1644**：記錄 `prAisFsmInfo->rJoinReqTime = kalGetTimeTick()`，用於後續 join timeout 計算。

**行 1647~1652**：

```c
if (ucReasonOfDisconnect != DISCONNECT_REASON_CODE_REASSOCIATION)
    prAisFsmInfo->fgIsDisconnectedByNonRequest = FALSE;
```

`fgIsDisconnectedByNonRequest` 標記是否為非使用者主動觸發的斷線（例如收到 AP 的 deauthentication frame）。connect 請求不屬於這種情況，故清為 FALSE。

**行 1655~1666**：若 `ucReasonOfDisconnect == DISCONNECT_REASON_CODE_REASSOCIATION`（同 ESS 的重新連線），不做完整的中止流程，只清空特定 pending 請求。

**行 1669~1670**：

```c
aisFsmFlushRequest(prAdapter, AIS_REQUEST_RECONNECT);
aisFsmInsertRequest(prAdapter, AIS_REQUEST_RECONNECT);
```

先清除舊的 reconnect 請求（若有），再插入新的。確保 pending queue 中不會有重複的 reconnect 請求。

**行 1672~1676**：若當前 state 不是 `AIS_STATE_DISCONNECTING`，呼叫 `aisFsmStateAbort()`，中止當前進行中的操作（如 scan、join 等），然後透過 `aisFsmSteps()` 驅動狀態機到 `AIS_STATE_IDLE`，再從 `IDLE` 處理 pending 的 reconnect 請求進入 `AIS_STATE_SEARCH`。

---

## 4. AIS FSM 連線段

### 4.1 `AIS_STATE_SEARCH` entry action

`aisFsmSteps()` 進入 `AIS_STATE_SEARCH` 時（`ais_fsm.c:980`）：

**`scanSearchBssDescByPolicy()`**：在 driver 維護的 BSS 描述符資料庫中，依 `rConnSettings` 的條件（SSID / BSSID / 頻率）搜尋候選 AP。

BSS 資料庫由之前的 scan 結果建立（每個 `cfg80211_inform_bss()` 呼叫對應一個 `BSS_DESC_T`）。`BSS_DESC_T` 包含：
- `aucBSSID`：AP 的 MAC 位址
- `aucSSID` / `ucSSIDLen`：AP 廣播的 SSID
- `ucChannelNum` / `eBand`：頻道和頻段
- `i2RSSIlast`：最近一次量測的 RSSI（Received Signal Strength Indicator，接收訊號強度）
- `u2CapInfo`：Capabilities Information（是否支援 privacy、是否為 AP 等）
- `rIeContainer`：原始 Beacon/Probe Response IE 資料

找到候選 AP 後：
```c
prAisFsmInfo->prTargetBssDesc = prBssDesc;
```

若找不到：
- `fgTryScan = TRUE`（`prConnSettings->fgIsScanReqIssued` 未設）→ 進入 `AIS_STATE_LOOKING_FOR`（掃描找指定 SSID）
- 超過重試次數 → 進入 `AIS_STATE_JOIN_FAILURE` 或 `AIS_STATE_WAIT_FOR_NEXT_SCAN`

### 4.2 `AIS_STATE_REQ_CHANNEL_JOIN` entry action

找到目標 AP 後，AIS FSM 需要向 CNM（Coexistence and Non-concurrent Management）申請頻道使用權。CNM 負責管理 Wi-Fi 各個功能模組（STA 連線、P2P、TDLS 等）的頻道使用，避免衝突。

```c
/* 組建 MSG_CH_REQ_T */
prMsgChReq->ucTokenID = ++prAisFsmInfo->ucSeqNumOfChReq;
prMsgChReq->eReqType = CH_REQ_TYPE_JOIN;
prMsgChReq->ucPrimaryChannel = prTargetBssDesc->ucChannelNum;
prMsgChReq->eBand = prTargetBssDesc->eBand;
/* ... */
mboxSendMsg(prAdapter, MBOX_ID_0, MSG_MNY_CNM_CH_REQ, ...);
prAisFsmInfo->fgIsChannelRequested = TRUE;
```

`ucTokenID` 用於在 `aisFsmRunEventChGrant()` 中驗證這個 grant 是否對應本次請求（與 scan 的 seq number 機制類似）。

### 4.3 `aisFsmRunEventChGrant()`（`ais_fsm.c:3377`）

CNM 核可頻道請求後發送此事件。

**行為：**

```c
if (eCurrentState == AIS_STATE_REQ_CHANNEL_JOIN &&
    prMsgChGrant->ucTokenID == prAisFsmInfo->ucSeqNumOfChReq) {

    prAisFsmInfo->fgIsChannelGranted = TRUE;

    /* 啟動 join timeout timer */
    cnmTimerStartTimer(prAdapter, &prAisFsmInfo->rJoinTimeoutTimer,
                       AIS_JOIN_CH_GRANT_INTERVAL);

    /* 進入 JOIN state */
    aisFsmSteps(prAdapter, AIS_STATE_JOIN);
}
```

`AIS_JOIN_CH_GRANT_INTERVAL` 是 join 操作的最長允許時間（auth + assoc 的總 timeout）。

### 4.4 `AIS_STATE_JOIN` entry action：`aisFsmStateInit_JOIN()`

```c
VOID aisFsmStateInit_JOIN(
    P_ADAPTER_T     prAdapter,
    P_BSS_DESC_T    prBssDesc    /* prAisFsmInfo->prTargetBssDesc */
)
```

**行為：**

1. 從 STA record pool 分配或取得一個 `STA_RECORD_T`（`cnmGetStaRecByAddress()`），代表目標 AP 的站台記錄。`STA_RECORD_T` 儲存與特定 STA/AP 的連線狀態（序號、加密金鑰、QoS 設定等）。

2. 設定 `prStaRec->ucAuthAlgNum`（authentication algorithm number）：
   - Open System：0x00
   - Shared Key（WEP）：0x01
   - FT（Fast Transition）：0x02

3. 設定 `prAisFsmInfo->ucSeqNumOfReqMsg = cnmIncreaseTokenId(prAdapter)` — 本次 join 的 seq number。

4. 組建 `MSG_SAA_FSM_START_T` 並送給 SAA FSM：
   ```c
   mboxSendMsg(prAdapter, MBOX_ID_0, MID_AIS_SAA_FSM_START, ...);
   ```

SAA（Station Association Arbiter）FSM 負責實際執行 802.11 auth frame 和 association request/response 的交換。SAA 完成後發 `MID_AIS_SAA_FSM_COMP` 給 AIS。

---

## 5. `aisFsmRunEventJoinComplete()`（`ais_fsm.c:1834`）

```c
VOID aisFsmRunEventJoinComplete(
    P_ADAPTER_T prAdapter,
    P_MSG_HDR_T prMsgHdr  /* MSG_SAA_FSM_COMP_T，含 rJoinStatus 和 assoc response */
)
```

**行 1851~1854**：

```c
if (prAisFsmInfo->eCurrentState != AIS_STATE_JOIN)
    /* 忽略（state 已切換，這個 event 已過時） */

if (prMsgJoinComp->ucSeqNum != prAisFsmInfo->ucSeqNumOfReqMsg)
    /* seq mismatch，忽略 */
```

`ucSeqNum` 比對防止以下情況：AIS 在等待 SAA 完成時，因某種原因（例如收到 disconnect 請求）離開了 `JOIN` state，隨後 SAA 仍然完成並回傳 event，但此 event 已不屬於當前 join 流程。

**行 1854**：`eNextState = aisFsmJoinCompleteAction(prAdapter, prMsgJoinComp)`

**行 1860~1861**：若 `eNextState != eCurrentState`，呼叫 `aisFsmSteps(prAdapter, eNextState)` 驅動狀態轉換。

**行 1863~1866**：釋放 assoc response frame buffer 和訊息記憶體（`cnmMemFree()`）。

---

## 6. `aisFsmJoinCompleteAction()`（`ais_fsm.c:1869`）

```c
ENUM_AIS_STATE_T aisFsmJoinCompleteAction(
    P_ADAPTER_T             prAdapter,
    P_MSG_SAA_FSM_COMP_T    prMsgJoinComp
)
```

`MSG_SAA_FSM_COMP_T` 包含：
- `rJoinStatus`（`WLAN_STATUS`）：`WLAN_STATUS_SUCCESS` 或失敗碼
- `prStaRec`（`P_STA_RECORD_T`）：join 完成的 STA record
- `prSwRfb`（`P_SW_RFB_T`）：Association Response frame 的 buffer（`SW_RFB_T` 是 MTK 的 RX frame buffer 描述符）

### 6.1 成功分支（行 1894~1957）

**行 1894**：進入條件：`prMsgJoinComp->rJoinStatus == WLAN_STATUS_SUCCESS`

**行 1897**：`prAisFsmInfo->ucConnTrialCount = 0` — 重置重試計數器。

**行 1911**：`aisChangeMediaState(prAdapter, MEDIA_STATE_CONNECTED)` — 更新 `prAisBssInfo->eConnectionState`，標記邏輯上已連線。

**行 1924**：`aisUpdateBssInfoForJOIN(prAdapter, prStaRec, prSwRfb)` — 從 Association Response frame 更新 BSS info，包含：
- AP 指定的 Association ID（AID）
- 協商的 capability（HT/VHT/QoS 等）
- 計算 DTIM（Delivery Traffic Indication Message）interval（用於 PSM 省電模式）

**行 1927**：`cnmStaRecChangeState(prAdapter, prStaRec, STA_STATE_3)` — 將 STA record 的狀態設為 `STA_STATE_3`（已認證且已關聯，可正常收發資料）。802.11 定義了三個狀態：State 1（未認證未關聯）、State 2（已認證未關聯）、State 3（已認證且已關聯）。

**行 1930~1932**：`nicUpdateRSSI(prAdapter, ...)` — 更新 RSSI 量測值到 `prAisBssInfo->i2RSSIlast`。

**行 1937**：`aisIndicationOfMediaStateToHost(prAdapter, MEDIA_STATE_CONNECTED, FALSE)` — 通知 host 已連線，觸發 `kalIndicateStatusAndComplete(WLAN_STATUS_MEDIA_CONNECT, ...)`（見第 7 節）。

**行 1956**：`return AIS_STATE_NORMAL_TR` — 進入正常傳輸狀態。

### 6.2 失敗分支（行 1959~2013）

**行 1961**：`aisFsmStateInit_RetryJOIN(prAdapter, prStaRec)` — 若失敗原因允許重試（例如 auth algorithm mismatch，可嘗試換一種 auth 方式），重新啟動 join。若重試成功啟動，回傳 `AIS_STATE_JOIN`（保持在 JOIN state）。

**行 1965**：`prStaRec->ucJoinFailureCount++`

**行 1968**：`aisFsmReleaseCh(prAdapter)` — 釋放從 CNM 借用的頻道，清除 `fgIsChannelRequested` 和 `fgIsChannelGranted`，送 `MID_MNY_CNM_CH_ABORT` 給 CNM。

**行 1971**：`cnmTimerStopTimer(prAdapter, &prAisFsmInfo->rJoinTimeoutTimer)` — 停止 join timeout timer。

**行 1983~1990**：若此 BSS 的 join 失敗次數超過 `AIS_BSS_JOIN_FAIL_COUNT_LIMIT`（通常 2~3 次），在 `prBssDesc->fgIsJoinFailed = TRUE` 標記，並設定 `rJoinFailTime`（之後的 scan/connect 會暫時跳過此 BSS，等 `AIS_BSS_JOIN_FAIL_TIMEOUT` 後再試）。

**行 2004~2007**：若 join 整體時間（`kalGetTimeTick() - prAisFsmInfo->rJoinReqTime`）超過 `AIS_JOIN_TIMEOUT`：

```c
return AIS_STATE_JOIN_FAILURE;
/* JOIN_FAILURE state entry action 會呼叫 nicMediaJoinFailure()，
   觸發 kalIndicateStatusAndComplete(WLAN_STATUS_JOIN_FAILURE) */
```

**行 2010~2012**：否則：

```c
aisFsmInsertRequest(prAdapter, AIS_REQUEST_RECONNECT);
return AIS_STATE_IDLE;
/* IDLE state 處理 pending 的 RECONNECT 請求，進入 SEARCH 再試 */
```

---

## 7. 成功回報 user space：`WLAN_STATUS_MEDIA_CONNECT`（`gl_kal.c:947`）

```c
VOID kalIndicateStatusAndComplete(
    P_GLUE_INFO_T   prGlueInfo,
    WLAN_STATUS     eStatus,    /* WLAN_STATUS_MEDIA_CONNECT */
    PVOID           pvBuf,
    UINT_32         u4BufLen
)
```

進入 `case WLAN_STATUS_MEDIA_CONNECT:` 分支（行 970）：

**行 975~976**：`wlanoidQueryBssid()` 查詢目前連線的 BSSID，並發送 WEXT `SIOCGIWAP` 事件（向後相容）。

**行 979**：`netif_carrier_on(prGlueInfo->prDevHandler)` — 通知 Linux 網路子系統 carrier（實體連接）已建立。這會觸發系統中等待 carrier 的服務（如 DHCP client）啟動。`netif_carrier_on()` / `netif_carrier_off()` 是 Linux 網路驅動的標準介面，用於通知 link state 變化。

**行 983~989**：`wlanoidQuerySsid()` 查詢已連線的 SSID，用於印 log（常見的 "connected to SSID xxx" 訊息在此）。

**行 1008~1040**：確認 cfg80211 BSS 資料庫中存在目前連線的 AP 資訊。若不存在（可能因為 scan 很久之前做的，BSS 已被 cfg80211 從資料庫清除），呼叫 `cfg80211_inform_bss()` 重新建立，確保 `cfg80211_connect_result()` 可以找到有效的 BSS 物件。

**行 1066~1070**：

```c
cfg80211_connect_result(prGlueInfo->prDevHandler,  /* net_device */
                        prBssDesc->aucBSSID,        /* AP 的 BSSID */
                        pucReqIe, u4ReqIeLength,    /* Association Request IEs */
                        pucRspIe, u4RspIeLength,    /* Association Response IEs */
                        WLAN_STATUS_SUCCESS,
                        GFP_KERNEL);
```

`cfg80211_connect_result()` 是 cfg80211 提供的函式，它：
1. 通知 nl80211 框架連線成功
2. nl80211 框架透過 netlink 發送 `NL80211_CMD_CONNECT` event
3. `wpa_supplicant` 接收此 event，開始後續的 DHCP、4-way handshake（WPA2-PSK）或 EAP（802.1X）等流程

---

## 8. 斷線路徑

### 8.1 主動斷線（user space 發起）

```
mtk_cfg80211_disconnect()  [gl_cfg80211.c:1295]
  → kalIoctl(wlanoidSetDisassociate)
  → wlanoidSetDisassociate()  [wlan_oid.c:7814]
      — 清除 prConnSettings->fgIsConnReqIssued = FALSE
      — 送 MID_OID_AIS_FSM_JOIN_REQ，ucReasonOfDisconnect = DISCONNECT_REASON_CODE_NEW_CONNECTION
      — 若目前已連線：kalIndicateStatusAndComplete(WLAN_STATUS_MEDIA_DISCONNECT_LOCALLY)
  → AIS FSM 進入 DISCONNECTING state → 發 deauth frame → aisFsmDisconnect()
  → kalIndicateStatusAndComplete(WLAN_STATUS_MEDIA_DISCONNECT)
  → cfg80211_disconnected(prDevHandler, reasonCode, ...)
```

### 8.2 `WLAN_STATUS_MEDIA_DISCONNECT` 分支（`gl_kal.c:1076`）

**行 1076**：進入 `case WLAN_STATUS_MEDIA_DISCONNECT:` 分支。

**行 1078~1080**：`netif_carrier_off(prGlueInfo->prDevHandler)` — 通知 Linux 網路子系統 carrier 已失去，停止 DHCP renewal 等。

**行 1083~1090**：`cfg80211_disconnected(prDevHandler, u16ReasonCode, NULL, 0, FALSE, GFP_KERNEL)`。最後的 `FALSE` 表示這是 locally generated（本地發起的）斷線，若為 `TRUE` 代表收到 AP 發的 deauth/disassoc。`u16ReasonCode` 是 802.11 斷線原因碼（例如 3 = Deauthenticated because sending STA is leaving）。

---

## 9. 關鍵旗標與欄位

| 欄位 | 所在結構 | 說明 |
|---|---|---|
| `fgIsConnReqIssued` | `CONNECTION_SETTINGS_T` | 是否有 connect 請求 in-flight；AIS IDLE state 檢查此值 |
| `fgIsConnByBssidIssued` | `CONNECTION_SETTINGS_T` | 是否為指定 BSSID 連線 |
| `eConnectionPolicy` | `CONNECTION_SETTINGS_T` | `CONNECT_BY_SSID_BEST_RSSI` 或 `CONNECT_BY_BSSID` |
| `u4FreqInKHz` | `CONNECTION_SETTINGS_T` | 鎖頻連線頻率（KHz） |
| `eAuthMode` | `CONNECTION_SETTINGS_T` | 認證模式（由 `wlanoidSetAuthMode` 寫入） |
| `eEncStatus` | `CONNECTION_SETTINGS_T` | 加密模式（由 `wlanoidSetEncryptionStatus` 寫入） |
| `ucConnTrialCount` | `AIS_FSM_INFO_T` | 連線重試次數，成功後歸零 |
| `ucSeqNumOfReqMsg` | `AIS_FSM_INFO_T` | join 請求 seq number，用於比對 join complete event |
| `fgIsChannelRequested` | `AIS_FSM_INFO_T` | 是否已向 CNM 申請頻道 |
| `fgIsChannelGranted` | `AIS_FSM_INFO_T` | 是否已取得 CNM 頻道授權 |
| `prTargetBssDesc` | `AIS_FSM_INFO_T` | SEARCH state 找到的目標 AP BSS 描述符 |
| `rJoinReqTime` | `AIS_FSM_INFO_T` | connect 請求發出時間（`kalGetTimeTick()`），用於整體 timeout 計算 |
| `fgIsDisconnectedByNonRequest` | `AIS_FSM_INFO_T` | 是否為 AP 主動斷線（deauth/disassoc），非 user space 請求 |

---

## 10. 常見問題診斷

### connect 送出後沒有 `cfg80211_connect_result`

確認 `aisFsmRunEventJoinComplete()` 是否被呼叫。此函式由 SAA FSM 完成時觸發，SAA FSM 未完成代表 auth 或 assoc 失敗。

常見原因：
- Auth 被 AP 拒絕（錯誤的 auth algorithm，例如 AP 只接受 SAE，但 driver 送 Open System auth）
- Assoc 被 AP 拒絕（capability mismatch、RSN IE 不正確等）
- `rJoinTimeoutTimer` 超時但未呼叫 `aisFsmRunEventJoinTimeout`

### 可以掃描但不能連線

`eAuthMode` 或 `eEncStatus` 與 AP 的設定不符。`wlanoidSetAuthMode` 和 `wlanoidSetEncryptionStatus` 的呼叫順序和值需與 AP 的 RSN IE 吻合。具體比對 `prConnSettings->eAuthMode` 與 AP BSS 的 `prBssDesc->rIeContainer` 中的 RSN IE。

### connect 後 `cfg80211_disconnected` 立即出現

連線後立即斷線。可能原因：
- 4-way handshake 失敗（PSK 不正確），AP 發 deauth，driver 收到後呼叫 `cfg80211_disconnected`
- RSN IE mismatch 導致 AP 在 assoc 完成後立即拒絕

查看 `u16ReasonCode` 值（802.11 斷線原因碼）可縮小範圍。
