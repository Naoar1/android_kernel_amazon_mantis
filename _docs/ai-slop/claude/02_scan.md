# MTK Wi-Fi Scan 流程參考

主要檔案：`gl_cfg80211.c`、`gl_kal.c`、`wlan_oid.c`、`ais_fsm.c`

---

## 0. 流程總覽

```
wpa_supplicant / iw 送出 NL80211_CMD_TRIGGER_SCAN
  → cfg80211 框架呼叫 .scan op
  → mtk_cfg80211_scan()          [gl_cfg80211.c:815]
      — 填充 PARAM_SCAN_REQUEST_ADV_T
      — kalIoctl(wlanoidSetBssidListScanAdv)
  → kalIoctlTimeout()            [gl_kal.c:2048]
      — 設 GLUE_FLAG_OID_BIT，喚醒 main_thread，睡眠等待 completion
  → main_thread 執行 wlanSetInformation()
  → wlanoidSetBssidListScanAdv() [wlan_oid.c:686]
      — 轉換參數，呼叫 aisFsmScanRequestAdv()
      — 啟動 rScanDoneTimer
  → aisFsmScanRequestAdv()       [ais_fsm.c:3290]
      — 依當前 AIS state 決定立即掃描或排入 pending queue
  → aisFsmSteps() 進入 AIS_STATE_SCAN 或 AIS_STATE_ONLINE_SCAN
      — 送 MID_AIS_SCN_SCAN_REQ_V2 給 SCN FSM
      — SCN FSM 透過 firmware command 觸發掃描
  → firmware 完成掃描，回送 EVENT_ID_SCAN_DONE
  → aisFsmRunEventScanDone()     [ais_fsm.c:1527]
      — 停 rScanDoneTimer，清 fgIsScanReqIssued
      — kalScanDone()
  → kalIndicateStatusAndComplete(WLAN_STATUS_SCAN_COMPLETE)  [gl_kal.c:947]
  → kalCfg80211ScanDone(prScanRequest, FALSE)
  → cfg80211_scan_done()         ← cfg80211 通知 wpa_supplicant 掃描完成
```

---

## 1. `mtk_cfg80211_scan()`（`gl_cfg80211.c:815`）

```c
int mtk_cfg80211_scan(struct wiphy *wiphy,
                      struct cfg80211_scan_request *request)
```

**參數：**

`wiphy`（`struct wiphy *`）：cfg80211 的無線硬體抽象物件，`wiphy_priv(wiphy)` 可取得 `GLUE_INFO_T *`。

`request`（`struct cfg80211_scan_request *`）：cfg80211 傳入的 scan 請求，由 `wpa_supplicant` 或 `iw` 透過 nl80211 下達。包含以下重要成員：

```c
struct cfg80211_scan_request {
    struct cfg80211_ssid *ssids;    /* 要掃描的 SSID 清單 */
    int                  n_ssids;   /* SSID 數量，0 = wildcard scan */
    const u8            *ie;        /* Probe Request 附帶的 IE */
    size_t               ie_len;
    u32                  n_channels;
    struct ieee80211_channel *channels[]; /* 指定掃描頻道，NULL 表示全頻道 */
    /* ... */
};
```

**行為（行 815~925）：**

**行 826~840**：取得 `prGlueInfo`（透過 `wiphy_priv(wiphy)` 或 `netdev_priv`），宣告 `PARAM_SCAN_REQUEST_ADV_T rScanRequest`（stack 上分配，見 `03_connect.md` 第 3.6 節的結構定義）。

**行 847~855**：檢查 `prGlueInfo->prScanRequest`。`prScanRequest` 儲存前一次 scan 請求的指標；非 NULL 代表前一次 scan 尚未完成（`cfg80211_scan_done()` 尚未被呼叫）。此時回傳 `-EBUSY`（resource busy），拒絕新 scan。

**行 860~875**：將 `request->ssids[]`（`struct cfg80211_ssid`，包含 ssid bytes 和 ssid_len）複製到 `rScanRequest.rSsid[]`（`PARAM_SSID_T`），並設定 `rScanRequest.u4SsidNum`。`n_ssids = 0` 時 `rScanRequest.u4SsidNum = 0`，driver 層會發出 wildcard Probe Request。

**行 876~885**：若 `request->ie` 非 NULL，設定 `rScanRequest.pucIE = request->ie`，`rScanRequest.u4IELength = request->ie_len`。這些 IE 會被附在 Probe Request frame 中。常見的附帶 IE：WPS IE（Wi-Fi Protected Setup，讓 AP 知道 STA 支援 WPS）、HT Capabilities IE、Extended Capabilities IE 等。

**行 886~902**：若 `request->n_channels > 0`，將 `request->channels[]`（`struct ieee80211_channel *`，包含 center_freq、band、flags 等）轉換為 `RF_CHANNEL_INFO_T`（包含 MTK 內部的頻段/頻道/頻寬表示）並填入 `rScanRequest.arChnlInfoList[]`，設定 `rScanRequest.ucChannelListNum`。若 `n_channels = 0`，`ucChannelListNum = 0`，firmware 掃描所有合法頻道。

**行 903~910**：呼叫 `kalIoctl(prGlueInfo, wlanoidSetBssidListScanAdv, &rScanRequest, sizeof(PARAM_SCAN_REQUEST_ADV_T), FALSE, FALSE, FALSE, &u4BufLen)`。第三個 `FALSE` 表示這是 set 操作（`fgRead = FALSE`）。

**行 911~918**：`kalIoctl` 成功後，將 `request` 指標儲存至 `prGlueInfo->prScanRequest`。此指標之後在 `kalIndicateStatusAndComplete()` 中傳給 `cfg80211_scan_done()`。**注意順序：** 儲存 `prScanRequest` 在 `kalIoctl` 成功後才做，若 OID 失敗則不儲存，避免留下懸空指標。

**行 920~925**：回傳 0（成功）給 cfg80211。此時 scan 尚未完成，只是「請求已受理」。

**回傳值：** 0 表示請求已送出；`-EBUSY` 表示前一次 scan 還在進行；`-EINVAL` 表示 `n_ssids` 超出上限或 `kalIoctl` 失敗。

---

## 2. `kalIoctlTimeout()`（`gl_kal.c:2048`）

見 `01_kernel_overview.md` 第 5.1 節。scan 路徑對應的呼叫：

```c
kalIoctl(prGlueInfo,
         wlanoidSetBssidListScanAdv,  /* OID handler */
         &rScanRequest,               /* PARAM_SCAN_REQUEST_ADV_T * */
         sizeof(PARAM_SCAN_REQUEST_ADV_T),
         FALSE,   /* fgRead = FALSE，這是 set 操作 */
         FALSE,   /* fgWaitResp */
         FALSE,   /* fgCmd */
         &u4BufLen);
```

---

## 3. `wlanoidSetBssidListScanAdv()`（`wlan_oid.c:686`）

```c
WLAN_STATUS wlanoidSetBssidListScanAdv(
    P_ADAPTER_T     prAdapter,
    PVOID           pvSetBuffer,   /* PARAM_SCAN_REQUEST_ADV_T * */
    UINT_32         u4SetBufferLen,
    PUINT_32        pu4SetInfoLen  /* output: sizeof(PARAM_SCAN_REQUEST_ADV_T) */
)
```

**行為（行 686~777）：**

**行 697~702**：ACPI 狀態檢查。若 `prAdapter->rAcpiState == ACPI_STATE_D3`（裝置已進入 D3 power state，意味著硬體已斷電），回傳 `WLAN_STATUS_ADAPTER_NOT_READY`。ACPI D3 是 ACPI 電源管理的最低功耗狀態，在 D3 狀態下無法存取硬體。

**行 704~710**：長度驗證（`u4SetBufferLen != sizeof(PARAM_SCAN_REQUEST_ADV_T)`）和空指標檢查。不符合時回傳 `WLAN_STATUS_INVALID_DATA`。

**行 712~716**：若 `prAdapter->fgIsRadioOff == TRUE`（Radio 已關閉，例如飛航模式），直接回傳 `WLAN_STATUS_SUCCESS` 而不啟動掃描。這是故意的行為：讓 OID 成功回傳，避免 caller 收到錯誤而困惑，但實際上不做任何事。

**行 718~736**：從 `PARAM_SCAN_REQUEST_ADV_T *prScanRequest` 提取資料：

```c
UINT_32 u4SsidNum = prScanRequest->u4SsidNum;
/* 複製多 SSID 到本地，最多 CFG_SCAN_SSID_MAX_NUM 個（通常 4 或 6） */
for (i = 0; i < u4SsidNum; i++) {
    rSsid[i] = prScanRequest->rSsid[i];
}
pucIE = prScanRequest->pucIE;
u4IELength = prScanRequest->u4IELength;
```

**行 755~773**：判斷是否為 online scan（裝置目前已連線）。若目前 state 為 `AIS_STATE_NORMAL_TR`（正常連線收發），根據頻道佔用情況決定是否立即 scan。呼叫路徑：

- online scan 允許：`aisFsmScanRequestAdv(prAdapter, u4SsidNum, rSsid, pucIE, u4IELength, ucChannelListNum, arChnlInfoList)`（行 765）
- 一般 scan：同上（行 757），但 AIS FSM 的當前 state 不同，`aisFsmScanRequestAdv` 內部處理邏輯不同

**行 774~775**：啟動 `rScanDoneTimer`。這是一個 watchdog timer，在 scan 完成事件遲遲不來時強制結束 scan 流程。Timer callback 是 `aisFsmRunEventScanDoneTimeOut()`。

**行 776**：回傳 `WLAN_STATUS_SUCCESS`，`main_thread` 接著呼叫 `complete()`，喚醒 `kalIoctlTimeout()` 的等待者。

---

## 4. `aisFsmScanRequestAdv()`（`ais_fsm.c:3290`）

```c
VOID aisFsmScanRequestAdv(
    P_ADAPTER_T         prAdapter,
    UINT_8              ucSsidNum,
    P_PARAM_SSID_T      prSsid,       /* SSID 清單 */
    PUINT_8             pucIE,
    UINT_32             u4IELength,
    UINT_8              ucChannelListNum,
    P_RF_CHANNEL_INFO_T prChnlInfoList
)
```

**行為（行 3290~3366）：**

**行 3310**：`if (!prAisFsmInfo->fgIsScanReqIssued)`。`fgIsScanReqIssued`（`BOOLEAN`，在 `AIS_FSM_INFO_T` 中）標記目前是否已有 scan 請求送出但尚未完成。此 flag 在 `aisFsmRunEventScanDone()` 中清除。若已有 scan in-flight，跳到行 3364 印 warning 並返回（丟棄新請求）。

**行 3313~3323**：複製 SSID 清單到 `prAisFsmInfo->ucScanSSIDNum` 和 `prAisFsmInfo->arScanSSID[]`。AIS FSM 保留自己的 scan 參數副本，因為原始的 `PARAM_SCAN_REQUEST_ADV_T` 是 `mtk_cfg80211_scan` 的 stack 變數，函式返回後即消失。

**行 3325~3330**：複製 IE buffer 到 `prAisFsmInfo->aucScanIEBuf[]`，設定 `prAisFsmInfo->u4ScanIELength`。

**行 3333~3339**：複製頻道清單到 `prAisFsmInfo->arScanChnlInfoList[]`，設定 `prAisFsmInfo->ucScanChannelListNum`（條件編譯，`CFG_SUPPORT_ROAMING_SKIP_ONE_AP`）。

**行 3341~3356**：若當前 AIS state 為 `AIS_STATE_NORMAL_TR`（已連線，正在正常收發）：

- **行 3343~3346**：若 `!prAisFsmInfo->fgIsInfraChannelFinished`（連線建立的 channel 流程尚未完成），呼叫 `aisFsmInsertRequest(AIS_REQUEST_SCAN)` 將 scan 請求排入 pending queue，返回。稍後當 `fgIsInfraChannelFinished` 變為 TRUE，AIS FSM 會從 pending queue 取出並處理。
- **行 3347~3355**：否則，呼叫 `aisFsmSteps(prAdapter, AIS_STATE_ONLINE_SCAN)` 直接進入 online scan state。

**行 3357~3359**：若當前 AIS state 為 `AIS_STATE_IDLE`，呼叫 `aisFsmSteps(prAdapter, AIS_STATE_SCAN)`。

**行 3360~3362**：其他 state（如 `JOIN`、`REQ_CHANNEL_JOIN` 等），呼叫 `aisFsmInsertRequest(AIS_REQUEST_SCAN)` 排入 pending queue，等當前操作完成後再處理。

---

## 5. AIS FSM：SCAN 和 ONLINE_SCAN state 的 entry action

`aisFsmSteps()` 進入 `AIS_STATE_SCAN`、`AIS_STATE_ONLINE_SCAN` 或 `AIS_STATE_LOOKING_FOR` 時（`ais_fsm.c:849`），執行相同的 scan 請求邏輯：

**組建 `MSG_SCN_SCAN_REQ_V2`：**

```c
/* prMsgScanReq 是 MSG_SCN_SCAN_REQ_V2_T，從 cnm memory pool 分配 */
prMsgScanReq->ucSeqNum = ++prAisFsmInfo->ucSeqNumOfScanReq;
```

`ucSeqNumOfScanReq` 是遞增的 sequence number，用於在 `aisFsmRunEventScanDone()` 中比對「這個 scan done event 是不是本次 scan 的回應」，防止舊的、延遲的 event 被誤處理。

**決定 scan type：**

- `prAisFsmInfo->ucScanSSIDNum == 0`：wildcard scan（發 Probe Request with empty SSID，所有 AP 都回應）
- `ucScanSSIDNum > 0`：directed scan（Probe Request 帶指定 SSID，只有符合的 AP 回應，速度較快）

**決定掃描頻道：**

- `ucScanChannelListNum == 0`：全頻道 scan（2.4G 全部 + 5G 全部，取決於 regulatory domain）
- `ucScanChannelListNum > 0`：只掃指定頻道（加快 scan 速度）

**發送 mailbox 訊息：**

```c
mboxSendMsg(prAdapter, MBOX_ID_0, MSG_SCN_SCAN_REQ_V2, MSG_SEND_METHOD_BUF);
```

`mboxSendMsg()` 將訊息放入 SCN FSM 的 mailbox，SCN FSM 收到後建立對應的 firmware command 並透過 SDIO 送出。

---

## 6. `aisFsmRunEventScanDone()`（`ais_fsm.c:1527`）

```c
VOID aisFsmRunEventScanDone(
    P_ADAPTER_T     prAdapter,
    P_MSG_HDR_T     prMsgHdr   /* MSG_SCN_SCAN_DONE_T，含 ucSeqNum */
)
```

Firmware 完成掃描後發送 `EVENT_ID_SCAN_DONE` event，`nic_cmd_event.c` 解析後呼叫此函式。

**行為（行 1527~1607）：**

**行 1549**：取出 firmware 回傳的 `ucSeqNumOfCompMsg`（completion message 的 seq number）。

**行 1554~1556**：seq match 檢查：

```c
if (prMsgScanDone->ucSeqNum != prAisFsmInfo->ucSeqNumOfScanReq) {
    /* 不是本次 scan 的回應，忽略 */
    return;
}
```

此檢查防止以下情況：scan 請求送出後，AIS FSM 因某種原因取消了 scan（例如收到 connect 請求），但 firmware 仍然完成掃描並發回 event。若不做 seq 比對，這個遲來的 event 會誤觸發 scan done 流程。

**行 1557**：`cnmTimerStopTimer(prAdapter, &prAisFsmInfo->rScanDoneTimer)`。停止 watchdog timer，正常完成不需要 timeout 強制結束。

**行 1559~1567**：若當前 state 為 `AIS_STATE_SCAN`（從 IDLE 出發的一般掃描）：

```c
prAisFsmInfo->fgIsScanReqIssued = FALSE;  /* 清除 in-flight 標記 */
/* 清除 IE buffer 長度 */
prAisFsmInfo->u4ScanIELength = 0;
kalScanDone(prAdapter->prGlueInfo, KAL_NETWORK_TYPE_AIS_INDEX, WLAN_STATUS_SUCCESS);
eNextState = AIS_STATE_IDLE;  /* 回到 IDLE */
fgIsTransition = TRUE;
```

`kalScanDone()` 最終呼叫 `kalIndicateStatusAndComplete(WLAN_STATUS_SCAN_COMPLETE)`。

**行 1573~1584**：若當前 state 為 `AIS_STATE_ONLINE_SCAN`（已連線中的掃描）：

```c
prAisFsmInfo->fgIsScanReqIssued = FALSE;
prAisFsmInfo->u4ScanIELength = 0;
kalScanDone(...);
/* 若有 pending roaming 請求，進 LOOKING_FOR；否則回 NORMAL_TR */
eNextState = AIS_STATE_NORMAL_TR;
```

**行 1591~1596**：若當前 state 為 `AIS_STATE_LOOKING_FOR`（漫遊搜尋中的掃描）：掃描結束後進 `AIS_STATE_SEARCH`，由 `SEARCH` state 分析掃描結果並決定是否切換 AP。

**行 1605~1606**：若 `fgIsTransition == TRUE`，呼叫 `aisFsmSteps(prAdapter, eNextState)` 觸發狀態遷移。

---

## 7. `kalIndicateStatusAndComplete()`：SCAN_COMPLETE 分支（`gl_kal.c:1132`）

```c
VOID kalIndicateStatusAndComplete(
    P_GLUE_INFO_T   prGlueInfo,
    WLAN_STATUS     eStatus,    /* WLAN_STATUS_SCAN_COMPLETE */
    PVOID           pvBuf,      /* 不使用 */
    UINT_32         u4BufLen
)
```

**行 1132**：進入 `case WLAN_STATUS_SCAN_COMPLETE:` 分支。

**行 1134**：`wext_indicate_wext_event(prGlueInfo, SIOCGIWSCAN, NULL, 0)` 發送舊的 WEXT `SIOCGIWSCAN` 事件，向後相容 `iwconfig` 等工具。

**行 1137~1142**：

```c
prScanRequest = prGlueInfo->prScanRequest;
prGlueInfo->prScanRequest = NULL;  /* 先清 NULL，再使用 */
```

**先清 NULL 的原因**：若先使用再清，在多核心環境下，另一個 CPU 可能在「使用」和「清 NULL」之間看到非 NULL 的 `prScanRequest`，並拒絕新的 scan 請求（回傳 `-EBUSY`）。先清 NULL 可以更早允許新的 scan 請求，並且避免重複 complete 的問題。

**行 1145~1146**：`kalCfg80211ScanDone(prScanRequest, FALSE)` — 第二個參數 `FALSE` 表示 scan 正常完成（非 aborted）。

`kalCfg80211ScanDone()` 呼叫 `cfg80211_scan_done(prScanRequest, FALSE)`，cfg80211 框架將掃描結果（由 firmware 透過 `cfg80211_inform_bss()` 逐一回報，在 scan 過程中已完成）通知給等待的 `wpa_supplicant`，後者可接著呼叫 `NL80211_CMD_GET_SCAN` 取得結果。

---

## 8. Scan Timeout 路徑：`aisFsmRunEventScanDoneTimeOut()`

當 `rScanDoneTimer` 到期（firmware scan done event 遲遲未來），此函式被呼叫：

**行為：**

1. `aisFsmStateAbort_SCAN()` — 向 SCN FSM 發 `MID_AIS_SCN_SCAN_CANCEL`，要求取消掃描
2. 強制呼叫 `kalScanDone()` — 即使沒有正常的 scan done event，也要通知 upper layer
3. 依當前 state 轉回 `AIS_STATE_IDLE` 或 `AIS_STATE_NORMAL_TR`
4. 清除 `fgIsScanReqIssued`，清空 `prGlueInfo->prScanRequest`（避免留下懸空指標）

---

## 9. 掃描結果如何回報（`cfg80211_inform_bss()`）

掃描結果不是在 scan done 時一次性回報，而是在 scan 進行中，每發現一個 AP，firmware 透過 event 通知 driver，driver 立即呼叫 `cfg80211_inform_bss()` 將 BSS 資訊（BSSID、SSID、頻道、RSSI、Beacon/Probe Response 原始 frame 等）加入 cfg80211 的 BSS 資料庫。`cfg80211_scan_done()` 只是通知「掃描流程結束」，`wpa_supplicant` 再從 cfg80211 的 BSS 資料庫讀取結果。

---

## 10. 常見問題診斷

### `-EBUSY` 回傳

`prGlueInfo->prScanRequest` 非 NULL。前一次 scan 流程沒有正常完成（`kalCfg80211ScanDone` 未被呼叫，或 `prGlueInfo->prScanRequest` 未被清 NULL）。

可能原因：
- `aisFsmRunEventScanDone()` seq mismatch，scan done event 被丟棄
- `rScanDoneTimer` timeout 前 `aisFsmRunEventScanDoneTimeOut()` 未觸發
- `main_thread` 已退出，`kalIoctl` 的 `complete()` 未被呼叫，前一次 scan 的 `kalIoctl` 卡住，`prScanRequest` 根本沒被設置（這種情況 `mtk_cfg80211_scan` 會卡在 `kalIoctl` 而非回 `-EBUSY`）

### 掃描請求成功但沒有結果

確認：
1. `wlanoidSetBssidListScanAdv()` 是否正常回傳（radio off 時會靜默返回成功）
2. `aisFsmScanRequestAdv()` 執行時的 AIS state — 若 state 不是 `IDLE` 或 `NORMAL_TR`，請求可能被排入 pending queue 但遲遲未處理
3. `aisFsmRunEventScanDone()` 是否收到 seq match 的 event — seq mismatch 代表 firmware 回應的是前一次 scan

### 掃描卡住不結束

`rScanDoneTimer` 應在 `wlanoidSetBssidListScanAdv()` 行 774~775 啟動。若 timer 沒有被停止（正常完成路徑）或觸發（timeout 路徑），表示 timer 本身有問題，或 `cnmTimerStartTimer()` 未被呼叫。

---

## 11. 關鍵旗標與欄位

| 欄位 | 所在結構 | 說明 |
|---|---|---|
| `prScanRequest` | `GLUE_INFO_T` | 進行中的 cfg80211 scan request；非 NULL 拒絕新 scan |
| `fgIsScanReqIssued` | `AIS_FSM_INFO_T` | 是否有 scan 已送出但未完成 |
| `ucSeqNumOfScanReq` | `AIS_FSM_INFO_T` | 本次 scan 的 seq number，用於比對 scan done event |
| `rScanDoneTimer` | `AIS_FSM_INFO_T` | scan watchdog timer，超時強制結束 |
| `fgIsInfraChannelFinished` | `AIS_FSM_INFO_T` | 連線建立的 channel 流程是否完成；FALSE 時 online scan 排隊等待 |
| `fgIsRadioOff` | `ADAPTER_T` | Radio 是否關閉；TRUE 時 scan 靜默返回成功 |
