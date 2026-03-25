# MTK Wi‑Fi Connect 導讀版（逐步帶讀）

> 目標：讓你可以從 `connect` 指令一路追到「真的連上」或「失敗回報」的每一段。

---

## 0) 先抓主線（不用背，先看懂）

`wpa_supplicant` 發 connect  
→ `mtk_cfg80211_connect()` 收到請求  
→ 多個 OID 先設定（mode/auth/encrypt/IE）  
→ `wlanoidSetConnect()` 把請求送進 AIS FSM  
→ AIS 做 join/auth/assoc  
→ 成功：`cfg80211_connect_result()`  
→ 失敗：重試/回到 idle/回報 disconnect。

---

## 1) 入口：`mtk_cfg80211_connect()` 做了哪些具體動作

檔案：`os/linux/gl_cfg80211.c`  
函式：`mtk_cfg80211_connect(...)`（約 `900`）

這個函式不是只做一件事，而是分段設定。

## 1.1 設定 operation mode

先呼叫：

- `wlanoidSetInfrastructureMode`

目的：把連線模式切到 infrastructure（station 連 AP 的模式）。

## 1.2 解析 AKM（認證套件）

它會讀 `sme->crypto.akm_suites[0]`，映射成 driver 的 `eAuthMode`，例如：

- `WLAN_AKM_SUITE_PSK` -> `AUTH_MODE_WPA2_PSK`
- `WLAN_AKM_SUITE_8021X` -> `AUTH_MODE_WPA2`

## 1.3 解析 cipher（加密）

它會組 `cipherGroup | cipherPairwise`，決定 `eEncStatus`：

- CCMP -> `ENUM_ENCRYPTION3_ENABLED`
- TKIP -> `ENUM_ENCRYPTION2_ENABLED`
- WEP -> `ENUM_ENCRYPTION1_ENABLED`

## 1.4 處理 IEs（很重要）

如果 `sme->ie` 不空，會抓出：

- WPS IE -> `wlanoidSetWSCAssocInfo`
-（條件編譯）HS20/Interworking/Roaming consortium 相關 IE
- 也會 parse RSN IE，看 MFP capability

## 1.5 下發 auth/encryption

- `wlanoidSetAuthMode`
- `wlanoidSetEncryptionStatus`

## 1.6 最終 connect request

打包 `PARAM_CONNECT_T`：

- `u4CenterFreq`
- `pucBssid`
- `pucSsid`
- `u4SsidLen`

呼叫：

- `kalIoctl(... wlanoidSetConnect ... )`

---

## 2) `kalIoctl` 這層到底做什麼

檔案：`os/linux/gl_kal.c`  
函式：`kalIoctlTimeout()`（約 `2048`）

和 scan 一樣，connect 也經過 OID queue：

1. 填 `prGlueInfo->OidEntry`
2. 設 `GLUE_FLAG_OID_BIT`
3. 喚醒 `main_thread`
4. 等 `rPendComp` completion

所以你看到 connect 卡住時，很多時候其實是卡在「OID 沒被完成」。

---

## 3) `wlanoidSetConnect()`：真正把連線交給 AIS FSM

檔案：`common/wlan_oid.c`  
函式：`wlanoidSetConnect(...)`（約 `1042`）

這個函式是 connect 核心轉接點。

## 3.1 重要檢查

- `u4SetBufferLen == sizeof(PARAM_CONNECT_T)`
- ACPI 不能是 D3
- 至少要有 SSID 或 BSSID

## 3.2 寫入 `rConnSettings`

重點欄位：

- `aucSSID / ucSSIDLen`
- `aucBSSID`
- `eConnectionPolicy`
  - `CONNECT_BY_SSID_BEST_RSSI`
  - `CONNECT_BY_BSSID`
- `fgIsConnByBssidIssued`
- `u4FreqInKHz`

## 3.3 處理「目前已連線」情況

它會判斷這次是不是 re-association（同 SSID/BSSID）或新連線。

## 3.4 送訊息給 AIS FSM

它會建立 `MSG_AIS_ABORT_T`，訊息 ID 用：

- `MID_OID_AIS_FSM_JOIN_REQ`

再 `mboxSendMsg(...)` 丟進 AIS mailbox。

重點：

> 到這一步才算「正式把連線請求交給連線狀態機」。

---

## 4) AIS FSM 連線段：成功與失敗在哪裡分歧

主要檔案：`mgmt/ais_fsm.c`

## 4.1 `aisFsmRunEventAbort()`

收到 OID connect/disconnect 相關訊息後會先走這裡（清 pending，決定是否重連）。

## 4.2 `aisFsmRunEventJoinComplete()`（約 `1834`）

SAA FSM 回 join 結果後，進這裡。

它會做：

- 檢查當前 state 必須是 `AIS_STATE_JOIN`
- 檢查 seq 是否匹配
- 呼叫 `aisFsmJoinCompleteAction()`

## 4.3 `aisFsmJoinCompleteAction()` 成功路徑

當 `rJoinStatus == WLAN_STATUS_SUCCESS`：

1. reset retry count
2. `aisChangeMediaState(... CONNECTED)`
3. `aisUpdateBssInfoForJOIN(...)`
4. `cnmStaRecChangeState(... STA_STATE_3)`
5. `nicUpdateRSSI(...)`
6. `aisIndicationOfMediaStateToHost(... CONNECTED ...)`
7. state 轉 `AIS_STATE_NORMAL_TR`

## 4.4 失敗路徑（常見）

- join 失敗次數累加
- release channel
- stop join timer
- 視情況重試或進 `JOIN_FAILURE` / `IDLE`

---

## 5) 連上後如何回報給 user space

檔案：`os/linux/gl_kal.c`  
函式：`kalIndicateStatusAndComplete()`（約 `947`）

當狀態是 `WLAN_STATUS_MEDIA_CONNECT`：

1. `wlanoidQueryBssid` / `wlanoidQuerySsid`
2. `netif_carrier_on(wlan0)`
3. 確保 cfg80211 BSS 存在（必要時 `cfg80211_inform_bss`）
4. 呼叫 `cfg80211_connect_result(...)`

這一刻 user space 才真正收到「connected」。

---

## 6) Disconnect 路徑（對照 connect）

入口：

- `mtk_cfg80211_disconnect()`（`gl_cfg80211.c:1295`）

中段：

- `wlanoidSetDisassociate()`（`wlan_oid.c:7814`）
  - 清 `fgIsConnReqIssued`
  - 送 `MID_OID_AIS_FSM_JOIN_REQ`（abort/disconnect）

回報：

- `kalIndicateStatusAndComplete(... MEDIA_DISCONNECT ...)`
- `cfg80211_disconnected(...)`

---

## 7) 你要學 connect 時，建議邊看邊記這些變數

在 `mtk_cfg80211_connect`：

- `eAuthMode`
- `eEncStatus`
- `u4AkmSuite`
- `rNewSsid`（其實是 connect 參數容器）

在 `wlanoidSetConnect`：

- `prConnSettings->eConnectionPolicy`
- `fgIsConnReqIssued`
- `fgIsConnByBssidIssued`
- `u4FreqInKHz`

在 `AIS FSM`：

- `eCurrentState`
- `ucSeqNumOfReqMsg`
- `ucConnTrialCount`

---

## 8) 實際排障（你可以直接照跑）

## 8.1 連不上：先看控制面

```bash
dmesg -T | grep -Ei "cfg80211|connect|join|assoc|auth|deauth|wlan"
```

你要確認：
- `mtk_cfg80211_connect` 有進來
- `wlanoidSetConnect` 有成功 return
- `aisFsmRunEventJoinComplete` 是否被觸發

## 8.2 連上又馬上掉

看：
- `cfg80211_connect_result` 後是否立即 `cfg80211_disconnected`
- `u2DeauthReason`（在 disconnect 路徑）

## 8.3 可以掃描但不能連

大多是：
- AKM/cipher 組合不匹配
- IE（WPS/RSN）帶入有問題
- FSM 在 retry/failure 路徑

重點看：
- `eAuthMode`、`eEncStatus`
- `wlanoidSetAuthMode`、`wlanoidSetEncryptionStatus`
- `aisFsmJoinCompleteAction` 的失敗分支

---

## 9) 最短帶讀順序（Connect）

請照這個順序讀：

1. `gl_cfg80211.c`：`mtk_cfg80211_connect`
2. `gl_kal.c`：`kalIoctlTimeout`
3. `wlan_oid.c`：`wlanoidSetConnect`
4. `ais_fsm.c`：`aisFsmRunEventJoinComplete` + `aisFsmJoinCompleteAction`
5. `gl_kal.c`：`kalIndicateStatusAndComplete` 的 MEDIA_CONNECT 分支

你會看到一條完整「進來 -> 翻譯 -> 執行 -> 回報」鏈。

---

## 10) 你學會這條線後就能做什麼

1. 判斷連線失敗是卡在 user->driver，還是 driver->firmware
2. 看懂 connect result 為什麼沒回來
3. 快速定位要改 cfg80211 還是 OID/FSM 層

這就是學 driver 最實用的第一步。
