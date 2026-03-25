# Scan 精讀（逐行註解版）

> 目標：從 `wpa_cli scan` 開始，逐行追到 `cfg80211_scan_done`。  
> 主要檔案：
> - `gl_cfg80211.c`
> - `gl_kal.c`
> - `wlan_oid.c`
> - `ais_fsm.c`

---

## 0) 全流程地圖

```text
user space scan
 -> mtk_cfg80211_scan()
 -> kalIoctl(... wlanoidSetBssidListScanAdv ...)
 -> main_thread 處理 OID
 -> wlanoidSetBssidListScanAdv()
 -> aisFsmScanRequestAdv()
 -> AIS_STATE_SCAN / ONLINE_SCAN
 -> firmware 回 scan done event
 -> aisFsmRunEventScanDone()
 -> kalIndicateStatusAndComplete(WLAN_STATUS_SCAN_COMPLETE)
 -> kalCfg80211ScanDone()
```

---

## 1) `mtk_cfg80211_scan()` 逐行註解（入口）

**檔案**：`gl_cfg80211.c`  
**位置**：約 `815`

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 815 | `int mtk_cfg80211_scan(...)` | cfg80211 入口，所有 scan 都會來這裡。 |
| 826~840 | 取 `prGlueInfo`、宣告 `rScanRequest` | 建立要送給 OID 的參數容器。 |
| 847~855 | 若 `prScanRequest` 已存在，拒絕新 scan | 防止同時多次 scan（回 `-EBUSY`）。 |
| 860~875 | 搬 `request->ssids[]` 到 `rScanRequest.rSsid[]` | 把 cfg80211 格式轉為 MTK 內部格式。 |
| 876~885 | 存 `request->ie` / `ie_len` | 掃描要附帶的 IE（WPS/vendor IE 等）。 |
| 886~902 | 處理 channel list（若 cfg 指定） | 指定掃描頻段/頻道。 |
| 903~910 | 進 `kalIoctl(... wlanoidSetBssidListScanAdv ...)` | 把 scan request 丟進 OID 路徑。 |
| 911~918 | OID 成功後存 `prGlueInfo->prScanRequest=request` | 後面 scan done 回報要用。 |
| 920~925 | return 0 | 回 cfg80211「scan 已受理」。 |

### 這段你要特別看

- `prGlueInfo->prScanRequest` 何時設/清（scan 競態關鍵）
- `rScanRequest` 裡 SSID/IE/channel 是否符合預期

---

## 2) `kalIoctl` + `main_thread` 逐行註解（中繼）

### 2.1 `kalIoctlTimeout()`

**檔案**：`gl_kal.c`（約 `2048` 起）

| 步驟 | 具體動作 |
|---|---|
| A | 把 handler/參數長度/SetQuery 等寫入 `prGlueInfo->OidEntry` |
| B | 設 `GLUE_FLAG_OID_BIT` |
| C | `wake_up_interruptible(waitq)` 喚醒 `main_thread` |
| D | 呼叫端 `wait_for_completion(&rPendComp)` 阻塞等待 |
| E | `main_thread` 做完 OID 後 `complete()`，呼叫端返回 |

### 2.2 `main_thread()` OID 處理分支

**檔案**：`gl_kal.c:2834`

| 行為 | 解釋 |
|---|---|
| 看到 `GLUE_FLAG_OID_BIT` | 表示有控制命令待執行 |
| 取 `OidEntry` | 找出當前 handler（這次是 `wlanoidSetBssidListScanAdv`） |
| 執行 `wlanSetInformation` | set 類 OID 會走這條 |
| `complete(&rPendComp)` | 喚醒 `kalIoctlTimeout` 的等待者 |

---

## 3) `wlanoidSetBssidListScanAdv()` 逐行註解（OID 核心）

**檔案**：`wlan_oid.c`  
**位置**：`686~777`

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 686 | 函式入口 | Scan OID 真正執行點。 |
| 697~702 | ACPI D3 檢查 | 睡眠態不可掃描。 |
| 704~710 | 長度與空指標檢查 | 必須是 `sizeof(PARAM_SCAN_REQUEST_ADV_T)`。 |
| 712~716 | `fgIsRadioOff` 處理 | Radio off 時直接成功返回（不掃）。 |
| 718~736 | 解析 `prScanRequest`（SSID/IE） | 把 cfg 帶來的資料搬到本地變數。 |
| 720~732 | 複製多 SSID | 支援 multiple SSID scan。 |
| 755~773 | 判斷 online scan 條件 | connected 狀態下是否允許 scan。 |
| 757/765 | `aisFsmScanRequestAdv(...)` | 把請求交給 AIS FSM。 |
| 774~775 | `cnmTimerStartTimer(... rScanDoneTimer ...)` | 啟 scan 完成 watchdog。 |
| 776 | return success | OID 層受理完成。 |

---

## 4) `aisFsmScanRequestAdv()` 逐行註解（FSM 入口）

**檔案**：`ais_fsm.c`  
**位置**：`3290~3366`

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 3310 | `if (!fgIsScanReqIssued)` | 防重入：同時只能有一個 scan in-flight。 |
| 3313~3323 | 複製 SSID 到 `rAisFsmInfo` | AIS 自己保留 scan 參數。 |
| 3325~3330 | 複製 IE buffer | 供 scan request 帶出。 |
| 3333~3339 | 複製 channel list（條件編譯） | 指定頻道掃描。 |
| 3341~3356 | 當前 state=`NORMAL_TR` | 若連線流程可插入，走 `ONLINE_SCAN`。 |
| 3343~3346 | 若 infra channel 未完成 | 先排 `AIS_REQUEST_SCAN`，稍後再掃。 |
| 3357~3359 | 當前 state=`IDLE` | 直接進 `AIS_STATE_SCAN`。 |
| 3360~3362 | 其他 state | 把 scan 請求插入 pending queue。 |
| 3364 | 已有 scan in-flight | 丟棄新 scan，印 warning。 |

---

## 5) `aisFsmRunEventScanDone()` 逐行註解（完成事件）

**檔案**：`ais_fsm.c`  
**位置**：`1527~1607`

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1549 | `ucSeqNumOfCompMsg = ...` | 取 firmware 回來的 scan seq。 |
| 1554~1556 | seq mismatch 檢查 | 不是本次 scan 的事件就不處理。 |
| 1557 | 停 `rScanDoneTimer` | 正常完成時要停 watchdog。 |
| 1559~1567 | state=`AIS_STATE_SCAN` | 清 scan flag/IE，`kalScanDone`，轉 `IDLE`。 |
| 1573~1584 | state=`ONLINE_SCAN` | 清 flag，`kalScanDone`，通常回 `NORMAL_TR`。 |
| 1591~1596 | state=`LOOKING_FOR` | 掃描結束後轉 `SEARCH`（或 roaming update）。 |
| 1605~1606 | 若 state changed，呼叫 `aisFsmSteps` | 正式做狀態遷移。 |

---

## 6) 回到 user space：`WLAN_STATUS_SCAN_COMPLETE`

**檔案**：`gl_kal.c`  
**函式**：`kalIndicateStatusAndComplete()`  
**位置**：`1132~1147`

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1132 | `case WLAN_STATUS_SCAN_COMPLETE:` | 進 scan 完成分支。 |
| 1134 | `wext_indicate_wext_event(SIOCGIWSCAN)` | 舊 WEXT 事件。 |
| 1137~1142 | 取 `prScanRequest` 並立刻清空 | 防止 race（新 scan 進來時重複使用舊指標）。 |
| 1145~1146 | `kalCfg80211ScanDone(prScanRequest, FALSE)` | 正式回報 cfg80211，user space 收到完成。 |

---

## 7) 你應該怎麼用這份精讀

### 練習 1：看 scan 是否卡在 OID

在 `mtk_cfg80211_scan` 與 `wlanoidSetBssidListScanAdv` 各打一條 log，確認 request 有到 OID。

### 練習 2：看 scan 是否卡在 FSM

在 `aisFsmScanRequestAdv` 和 `aisFsmRunEventScanDone` 打 log，檢查 state 是否有進/出。

### 練習 3：看是否卡在回報層

在 `kalIndicateStatusAndComplete` 的 `SCAN_COMPLETE` case 打 log，確認 cfg80211 回報有發出。

---

## 8) 一句話收斂

Scan 失敗大多不是「掃描 API 壞了」，而是卡在三件事：

1. OID 沒完成（`kalIoctl/main_thread`）
2. FSM 沒進入可掃狀態（`fgIsScanReqIssued/state`）
3. 完成事件沒回來或 seq mismatch（`aisFsmRunEventScanDone`）
