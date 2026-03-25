# AIS 精讀（逐行註解版）

> 檔案：`drivers/misc/mediatek/connectivity/wlan/gen4/mgmt/ais_fsm.c`  
> 對象：正在學 driver 的初/中階讀者  
> 方式：挑 AIS 核心函式做「逐行註解」

---

## 0. 先說明範圍

`ais_fsm.c` 本身超過 4,000 行，完整逐行會變成一本書。  
這份精讀用「**核心路徑逐行**」：

1. `aisFsmSteps()`（狀態機主引擎）
2. `aisFsmRunEventAbort()` + `aisFsmStateAbort()`（中止/重連主入口）
3. `aisFsmRunEventJoinComplete()` + `aisFsmJoinCompleteAction()`（連線成敗分岔）

你把這三段吃透，AIS 主要邏輯就懂了。

---

## 1) `aisFsmSteps()` 逐行註解（核心引擎）

**位置**：`ais_fsm.c:849`

### 1.1 入口段（849~909）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 849 | `VOID aisFsmSteps(...)` | AIS 的「主狀態切換函式」。幾乎所有 state 變化都經過這裡。 |
| 851~857 | 宣告 `prAisFsmInfo/prAisBssInfo/ePreviousState/fgIsTransition` | 先把 AIS 主要上下文抓出來。 |
| 859~861 | `ASSERT(prAdapter)` / 取 `rAisFsmInfo` | 防呆 + 取得狀態機實體。 |
| 863~868 | 若 old state 非 `NORMAL_TR`，把 network active | 讓 TX/RX queue 在狀態轉換期間保持可用。 |
| 871 | `do {` | 允許一次事件內連續多次 state transition。 |
| 872~873 | `ePreviousState = eCurrentState` | 保存舊狀態，給 log 和判斷用。 |
| 874 | `eCurrentState = eNextState` | 實際切換狀態。 |
| 877~890 | `DBGLOG` 印狀態遷移訊息 | 這是你 trace AIS 行為最有價值的 log。 |
| 893~894 | `fgIsTransition = FALSE` | 預設這輪不再跳下一個狀態。 |
| 897 | `switch (eCurrentState)` | 進入新狀態的 entry action。 |
| 902~909 | `AIS_STATE_IDLE` 分支前半 | 先停 scan done timer，避免舊 timer 干擾新流程。 |

### 1.2 `IDLE` 決策段（910~979）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 910~914 | `rScanDoneTimer` stop + `fgIsScanning = FALSE` | IDLE 代表當前沒有 scan in-flight。 |
| 916~919 | `prAisReq = aisFsmGetNextRequest(...)` | 取 pending queue 下一個請求。 |
| 920~923 | 若有 request，先印 log | 可觀察 queue 消化順序。 |
| 925~944 | `switch(prAisReq->eReqType)` | 按 request type 決定下一個 state。 |
| 927~932 | `AIS_REQUEST_SCAN -> AIS_STATE_SCAN` | 一般 scan 直接進 scan state。 |
| 933~937 | `AIS_REQUEST_RECONNECT` | 若連線需求存在，轉 `SEARCH`。 |
| 938~943 | roaming req | roaming 搜尋/連線請求也導到 `SEARCH`。 |
| 945~949 | remain_on_channel req | 轉 `REQ_REMAIN_ON_CHANNEL`。 |
| 952~956 | queue 項目處理完釋放記憶體 | `cnmMemFree` 回收 request 物件。 |
| 957~971 | 無 pending request 時的分支 | 若 `fgIsConnReqIssued` 為真，仍可直接走 `SEARCH`。 |
| 972~978 | `fgIsTransition = TRUE` | 告訴 do-while 還要再跑一輪切到新 state。 |

### 1.3 `SEARCH` 決策段（980~1086）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 983~985 | `scanSearchBssDescByPolicy` | 依 policy（SSID/BSSID/頻道）找候選 AP。 |
| 986~997 | 有找到 BSS 的處理 | 記錄目標 AP、更新 RSN 相關欄位。 |
| 998~1004 | `prTargetBssDesc = prBssDesc` | 之後 JOIN 都靠這個目標。 |
| 1006~1016 | 找不到目標 AP | 可能改走 `LOOKING_FOR` 或等待下一次 scan。 |
| 1020~1034 | `aisFsmStateSearchAction` | 搜尋失敗後的統一策略函式（很關鍵）。 |
| 1036~1086 | 轉移到 `REQ_CHANNEL_JOIN` 或其他狀態 | 最後若可 join，準備去要 channel privilege。 |

### 1.4 `REQ_CHANNEL_JOIN` / `JOIN` 核心段（1111~1188）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1111~1136 | 組 `MSG_CH_REQ_T` | 向 CNM 申請頻道權，才能做 auth/assoc。 |
| 1138~1147 | 填 token / band / ch / sco / max interval | 這些欄位決定要哪個頻道和使用時間。 |
| 1148~1154 | `mboxSendMsg(... MID_MNY_CNM_CH_REQ ...)` | 真正把請求送給 CNM FSM。 |
| 1155~1158 | `fgIsChannelRequested = TRUE` | 標記「我正在等頻道授權」。 |
| 1163~1188 | `AIS_STATE_JOIN` | 進入 join state，呼叫 `aisFsmStateInit_JOIN`。 |

### 1.5 `NORMAL_TR` 段（1285~1374）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1287~1294 | 若 infra channel 未完成先 break | 防止連線保護期內被其他工作打斷。 |
| 1297~1306 | 先看 roaming request | roaming 有較高優先序。 |
| 1307~1325 | 處理一般 pending request | scan / remain_on_channel 等。 |
| 1326~1371 | 若要轉新 state，`fgIsTransition=TRUE` | 讓 state engine 連續跳轉。 |

---

## 2) `Abort` 路徑逐行註解（斷線/重連核心）

## 2.1 `aisFsmRunEventAbort()`（1618~1677）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1618 | 函式入口 | AIS 接收到 abort/disconnect 事件。 |
| 1634~1638 | 讀 `MSG_AIS_ABORT_T` + free msg | 取出 reason 和 delay flag 後釋放訊息。 |
| 1644 | 記錄 `rJoinReqTime` | 之後 timeout/retry 會用到。 |
| 1647~1652 | 設 `fgIsDisconnectedByNonRequest` | 區分非使用者主動斷線（deauth/disassoc）。 |
| 1655~1666 | 特別處理 reassociation | 若是漫遊重連，不走一般中止流程。 |
| 1669~1670 | 先清舊 reconnect，再插入新 reconnect request | 保證 queue 裡 request 不重複。 |
| 1672~1676 | 若不在 DISCONNECTING，呼叫 `aisFsmStateAbort` | 真正中止目前工作。 |

## 2.2 `aisFsmStateAbort()`（1690~1823）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1708 | 記錄 disconnect reason 到 BSS info | 後續回報 userspace/roaming 決策會用。 |
| 1711~1803 | 按 current state 做中止動作 | 這段是大重點：不同 state 中止方式不同。 |
| 1725~1732 | 在 `SCAN` state abort | 取消 scan，並把 scan request 放回 queue（稍後重跑）。 |
| 1743~1749 | 在 `REQ_CHANNEL_JOIN` state abort | 先還 channel。 |
| 1751~1756 | 在 `JOIN` state abort | 呼叫 `aisFsmStateAbort_JOIN`。 |
| 1766~1775 | 在 `ONLINE_SCAN` state abort | 同樣取消 scan + request 回 queue。 |
| 1787~1798 | remain_on_channel state abort | 還 channel + 停 channel timeout timer。 |
| 1805~1817 | 若原本已 connected | 視情況進 `DISCONNECTING` 發 deauth，再走 normal abort。 |
| 1819 | `aisFsmDisconnect(...)` | 統一收尾，觸發 disconnect indication。 |

---

## 3) Join complete 路徑逐行註解（成功/失敗分岔）

## 3.1 `aisFsmRunEventJoinComplete()`（1834~1867）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1851~1854 | 檢查當前 state 是 JOIN 且 seq match | 防止舊事件/錯誤事件污染狀態機。 |
| 1854 | `eNextState = aisFsmJoinCompleteAction(...)` | 成敗決策都在下一個函式。 |
| 1860~1861 | 若 state 改變，呼叫 `aisFsmSteps` | 狀態機正式遷移。 |
| 1863~1866 | 回收 assoc rsp 與 msg | 清理記憶體。 |

## 3.2 `aisFsmJoinCompleteAction()`（1869~2018）

### 成功分支（1894~1957）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1894 | `rJoinStatus == SUCCESS` | 進成功分支。 |
| 1897 | `ucConnTrialCount = 0` | 重置重試次數。 |
| 1911 | `aisChangeMediaState(CONNECTED)` | 先改媒體狀態。 |
| 1924 | `aisUpdateBssInfoForJOIN` | 把 AP 參數寫進 BSS info。 |
| 1927 | `cnmStaRecChangeState(... STA_STATE_3)` | STA record 進已連線狀態。 |
| 1930~1932 | `nicUpdateRSSI` | 更新 RSSI。 |
| 1937 | `aisIndicationOfMediaStateToHost(CONNECTED)` | 對上層宣告連線成功。 |
| 1956 | `eNextState = AIS_STATE_NORMAL_TR` | 連線完成進正常傳輸狀態。 |

### 失敗分支（1959~2013）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1961 | `aisFsmStateInit_RetryJOIN` | 先嘗試另一種 auth 重試。 |
| 1965 | `prStaRec->ucJoinFailureCount++` | STA 失敗次數累加。 |
| 1968 | `aisFsmReleaseCh` | 釋放頻道資源。 |
| 1971 | 停 `rJoinTimeoutTimer` | 避免舊 timer 再觸發。 |
| 1983~1990 | BSS 失敗次數與暫時封鎖 | 避免一直撞同個壞 AP。 |
| 1996~1997 | 必要時釋放 STA rec | 清理。 |
| 2004~2007 | 若整體 join 超時 -> `JOIN_FAILURE` | 終止本次連線。 |
| 2010~2012 | 否則插入 `AIS_REQUEST_RECONNECT` -> `IDLE` | 回到 idle 等重試。 |

---

## 4) 你讀 AIS 時最該盯的 8 個欄位

1. `eCurrentState`
2. `fgIsScanReqIssued`
3. `fgIsConnReqIssued`
4. `fgIsChannelRequested`
5. `fgIsChannelGranted`
6. `ucSeqNumOfScanReq`
7. `ucSeqNumOfReqMsg`
8. `ucConnTrialCount`

只要這 8 個欄位你能在 log 裡跟到，AIS 行為基本就可預測。

---

## 5) 最短實戰排障路線（AIS）

1. 先看 state transition log（`[old] -> [new]`）
2. 若掃描卡住：看 `ScanDone seq` / `rScanDoneTimer`
3. 若連線卡住：看 `JOIN -> JoinComplete` 有沒有進
4. 若反覆重連：看 `ucConnTrialCount` + `AIS_REQUEST_RECONNECT` queue

---

## 6) 下一步建議

如果你要更進一步，我可以再給你一份：

- **AIS 事件時間線模板（可直接貼 dmesg）**  
  你只要貼 log，我可以幫你標出「哪一行是 state 轉折、哪一行是異常點」。
