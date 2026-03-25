# MTK Wi‑Fi Scan 導讀版（逐步帶讀）

> 目標：讓你可以**從使用者下 scan 指令，一路追到 driver 與回報**。  
> 風格：淺顯，但每一步都有具體函式與檔案位置。

---

## 0) 你在追什麼？（一句話）

你要追的是這條線：

`wpa_supplicant / iw` 送出 scan  
→ `mtk_cfg80211_scan()` 接到  
→ `kalIoctl()` 排入 OID  
→ `wlanoidSetBssidListScanAdv()` 轉給 AIS FSM  
→ firmware 掃描完成回事件  
→ `cfg80211_scan_done()` 回 user space。

---

## 1) 入口：使用者空間怎麼觸發

常見觸發方式：

```bash
wpa_cli -i wlan0 scan
wpa_cli -i wlan0 scan_results
```

或：

```bash
iw dev wlan0 scan
```

這些最後都會走到 kernel 的 `cfg80211 ops`，對應 MTK 的：

- `mtk_cfg80211_scan()`
- 檔案：`drivers/misc/mediatek/connectivity/wlan/gen4/os/linux/gl_cfg80211.c`
- 行號：約 `815`

---

## 2) 第一步：`mtk_cfg80211_scan()` 做了什麼

函式：`mtk_cfg80211_scan(...)` (`gl_cfg80211.c:815`)

### 2.1 主要輸入

- `request->n_ssids`
- `request->ssids[]`
- `request->ie` / `request->ie_len`
- `request->n_channels` / `request->channels[]`（若有指定）

### 2.2 主要動作

1. 建立 `PARAM_SCAN_REQUEST_ADV_T rScanRequest`
2. 把 SSID 清單搬進 `rScanRequest.rSsid[]`
3. 把 IE 指標和長度放進 `rScanRequest.pucIE/u4IELength`
4. 若有指定 channel，轉成 driver 內部 channel 格式
5. 呼叫：
   ```c
   kalIoctl(prGlueInfo,
            wlanoidSetBssidListScanAdv,
            &rScanRequest,
            sizeof(PARAM_SCAN_REQUEST_ADV_T),
            FALSE, FALSE, FALSE,
            &u4BufLen)
   ```
6. 成功後把 `prGlueInfo->prScanRequest = request`（留給 scan done 回報時使用）

### 2.3 這邊常見失敗點

- `prGlueInfo->prScanRequest != NULL`：回 `-EBUSY`（前一次 scan 還沒結束）
- `request->n_ssids` 超過上限：回 `-EINVAL`
- `kalIoctl` 回失敗：回 `-EINVAL`

---

## 3) 第二步：`kalIoctl()` 怎麼把請求送進 driver 主執行緒

函式：
- `kalIoctl()`
- `kalIoctlTimeout()`
- 檔案：`gl_kal.c`（約 `2048` 起）

### 3.1 核心機制（你一定要懂）

`kalIoctlTimeout()` 不是直接做 scan，而是：

1. 把請求填進 `prGlueInfo->OidEntry`
2. 設 `GLUE_FLAG_OID_BIT`
3. `wake_up_interruptible(&waitq)` 喚醒 `main_thread`
4. 呼叫端 `wait_for_completion(&rPendComp)` 等結果

也就是：

- 呼叫端（cfg80211 thread）負責發 request
- `main_thread` 負責實際執行 OID handler

---

## 4) 第三步：`main_thread()` 收到 OID 後做什麼

函式：`main_thread()`

- 檔案：`gl_kal.c:2834`

### 4.1 重要旗標

`GLUE_FLAG_OID_BIT`（在 `gl_os.h` 定義）

### 4.2 執行流程

當 `main_thread` 看到 OID bit：

- 取 `prGlueInfo->OidEntry`
- 因為 scan 是 set 操作，所以走：
  - `wlanSetInformation(...)`
- 實際會呼叫到你傳進來的 handler：
  - `wlanoidSetBssidListScanAdv(...)`

做完後 `complete(&rPendComp)`，讓 `kalIoctlTimeout()` 返回。

---

## 5) 第四步：`wlanoidSetBssidListScanAdv()`（真正把 scan 交給 AIS）

函式：`wlanoidSetBssidListScanAdv()`

- 檔案：`common/wlan_oid.c`
- 行號：約 `686`

### 5.1 做的事（具體）

1. 檢查 ACPI 狀態（D3 會拒絕）
2. 檢查 `u4SetBufferLen == sizeof(PARAM_SCAN_REQUEST_ADV_T)`
3. `radio off` 時直接返回（不掃）
4. 把 `PARAM_SCAN_REQUEST_ADV_T` 轉成 AIS 可吃的格式
5. 呼叫 `aisFsmScanRequestAdv(...)`
6. 啟動 `rScanDoneTimer`（避免永遠等不到 scan done）

### 5.2 你要觀察的欄位

- `u4SsidNum`
- `rSsid[]`
- `pucIE` / `u4IELength`
- `ucChannelListNum` / `arChnlInfoList[]`

---

## 6) 第五步：AIS FSM 收到掃描請求後如何決策

函式：`aisFsmScanRequestAdv(...)`

- 檔案：`mgmt/ais_fsm.c`
- 行號：約 `3290`

### 6.1 它會先把參數存到 `rAisFsmInfo`

- `ucScanSSIDNum`
- `arScanSSID[]`
- `u4ScanIELength` + `aucScanIEBuf`
- `ucScanChannelListNum` + `arScanChnlInfoList`

### 6.2 再依目前 state 決定下一步

- `AIS_STATE_NORMAL_TR`：
  - 若 infra channel 還沒結束 -> 把 scan 請求暫存（pending）
  - 否則進 `AIS_STATE_ONLINE_SCAN`
- `AIS_STATE_IDLE`：進 `AIS_STATE_SCAN`
- 其他狀態：先 insert request，等時機

這就是為什麼有時你按 scan 不是立刻跑（driver 在保護當前連線流程）。

---

## 7) 第六步：掃描完成事件回來

函式：`aisFsmRunEventScanDone(...)`

- 檔案：`mgmt/ais_fsm.c`
- 行號：約 `1527`

### 7.1 它會做

1. 檢查 seq 是否符合當次 scan request
2. `cnmTimerStopTimer(rScanDoneTimer)`
3. 清 `fgIsScanReqIssued`
4. 清 IE buffer 長度
5. 呼叫 `kalScanDone(...)`
6. 依當前狀態切回 `AIS_STATE_IDLE` 或 `AIS_STATE_NORMAL_TR`

---

## 8) 第七步：回報給 cfg80211 / user space

函式：`kalIndicateStatusAndComplete(... WLAN_STATUS_SCAN_COMPLETE ...)`

- 檔案：`gl_kal.c:947`

這裡會：

1. 取出 `prGlueInfo->prScanRequest`
2. 設回 `NULL`（避免重複完成）
3. 呼叫 `kalCfg80211ScanDone(prScanRequest, FALSE)`

結果：`wpa_supplicant/iw` 收到 scan 完成。

---

## 9) 你可以直接照著看的「最短導讀路線」

請按這順序開檔：

1. `gl_cfg80211.c` 看 `mtk_cfg80211_scan`（入口）
2. `gl_kal.c` 看 `kalIoctlTimeout`（OID 進 queue）
3. `gl_kal.c` 看 `main_thread`（OID 被執行）
4. `wlan_oid.c` 看 `wlanoidSetBssidListScanAdv`（轉給 AIS）
5. `ais_fsm.c` 看 `aisFsmScanRequestAdv`（state 決策）
6. `ais_fsm.c` 看 `aisFsmRunEventScanDone`（完成事件）
7. `gl_kal.c` 看 `kalIndicateStatusAndComplete` 的 SCAN_COMPLETE case（回報出口）

---

## 10) 常見 scan 問題對照

## 問題 A：`-EBUSY`

原因：`prScanRequest` 還在，前一次 scan 沒完成。

## 問題 B：呼叫成功但沒結果

先查：
- `wlanoidSetBssidListScanAdv` 有沒有進
- `aisFsmScanRequestAdv` 有沒有進到 `AIS_STATE_SCAN/ONLINE_SCAN`
- `aisFsmRunEventScanDone` 有沒有被打到

## 問題 C：掃描卡住

查 `rScanDoneTimer` timeout 路徑：
- `aisFsmRunEventScanDoneTimeOut`（`ais_fsm.c`）

---

## 11) 建議你邊看邊做的小練習

1. 在 `mtk_cfg80211_scan` 把 `request->n_ssids` 和 `request->n_channels` 打 log
2. 在 `wlanoidSetBssidListScanAdv` 印出 `ucSsidNum/ucChannelListNum`
3. 在 `aisFsmRunEventScanDone` 印 `ucSeqNumOfCompMsg` 與 `ucSeqNumOfScanReq`

你會很快抓到 scan 整條路徑。
