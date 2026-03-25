# Connect 精讀（逐行註解版）

> 目標：從 `wpa_supplicant connect` 開始，逐行追到 `cfg80211_connect_result`。  
> 主要檔案：
> - `gl_cfg80211.c`
> - `gl_kal.c`
> - `wlan_oid.c`
> - `ais_fsm.c`

---

## 0) 全流程地圖

```text
wpa_supplicant connect
 -> mtk_cfg80211_connect()
 -> kalIoctl(wlanoidSetAuthMode / wlanoidSetEncryptionStatus / wlanoidSetConnect)
 -> main_thread OID dispatch
 -> wlanoidSetConnect()
 -> mboxSendMsg(MID_OID_AIS_FSM_JOIN_REQ)
 -> AIS FSM: SEARCH -> REQ_CHANNEL_JOIN -> JOIN
 -> SAA 回 JOIN_COMPLETE
 -> aisFsmJoinCompleteAction()
 -> kalIndicateStatusAndComplete(WLAN_STATUS_MEDIA_CONNECT)
 -> cfg80211_connect_result()
```

---

## 1) `mtk_cfg80211_connect()` 逐行註解（入口）

**檔案**：`gl_cfg80211.c`  
**函式**：`mtk_cfg80211_connect`（約 `900`）

> 這個函式很長，下面只標註「真正影響連線行為」的區塊。

### 1.1 基本驗證與模式設定

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 900 | 函式入口 | cfg80211 connect 的主入口。 |
| 909~920 | 參數/指標防呆 | 避免空指標。 |
| 922~930 | `wlanoidSetInfrastructureMode` | 先把 operation mode 設為 infrastructure。 |

### 1.2 認證/加密參數推導（非常重要）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 970~975 | 判斷 WPA version | WPA1/WPA2/none。 |
| 977~987 | 解析 `auth_type` | open/shared 或 auto switch。 |
| 989~1014 | pairwise cipher | 把 cfg80211 cipher 轉 driver 表示。 |
| 1016~1038 | group cipher | 同上。 |
| 1040~1084 | `akm_suites` -> `eAuthMode/u4AkmSuite` | 這裡決定 PSK/802.1X/WPA2 等。 |

### 1.3 IE 處理（WPS/RSN/HS20）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1098~1163 | parse `sme->ie` | 找 WPS、WPA/RSN、HS20 相關 IE。 |
| 1115~1119 | `wlanoidSetWSCAssocInfo` | 若帶 WPS IE，會下發到 driver。 |
| 1151~1160 | parse RSN capability | 包含 MFP capability。 |

### 1.4 寫回 auth/encryption OID

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1171~1174 | `wlanoidSetAuthMode` | 下發認證模式。 |
| 1188~1212 | 計算 `eEncStatus` 後 `wlanoidSetEncryptionStatus` | 下發加密模式。 |

### 1.5 最終 connect OID（關鍵）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1238~1241 | 填 `PARAM_CONNECT_T rNewSsid` | 放 center_freq/bssid/ssid/ssid_len。 |
| 1242~1243 | `kalIoctl(... wlanoidSetConnect ... )` | 真正送出 connect request。 |
| 1245~1248 | 失敗回 `-EINVAL` | connect 被 OID 層拒絕。 |

---

## 2) `kalIoctl` + `main_thread`（中繼層）

這段跟 scan 一樣，但 connect 會連續下好幾個 OID。

## 2.1 `kalIoctlTimeout()`

流程：

1. 寫 `OidEntry`
2. 設 `GLUE_FLAG_OID_BIT`
3. 喚醒 `main_thread`
4. 等 `rPendComp`

## 2.2 `main_thread()`

看到 `OID_BIT` 後：

- 呼叫 `wlanSetInformation()`
- 執行本次 handler（例如 `wlanoidSetConnect`）
- `complete()` 回 caller

=> 如果你看到 connect 卡住，第一件事就是確認 `complete()` 有沒有被做。

---

## 3) `wlanoidSetConnect()` 逐行註解（核心）

**檔案**：`wlan_oid.c`  
**位置**：`1042~1188`

### 3.1 入口驗證 + 建 message

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1042 | 函式入口 | OID 層 connect 入口。 |
| 1063~1069 | 長度與 ACPI 檢查 | 非 `sizeof(PARAM_CONNECT_T)` 或 D3 直接 fail。 |
| 1070~1076 | 配置 `MSG_AIS_ABORT_T` | 之後送 `MID_OID_AIS_FSM_JOIN_REQ` 到 AIS。 |

### 3.2 寫入 connection settings

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1086~1091 | 清空舊 SSID/BSSID/policy | 每次 connect 重建設定。 |
| 1092~1100 | 若有 SSID，設 policy=by SSID | 也會比較是不是同 SSID（reassoc 判斷）。 |
| 1101~1110 | 若有 BSSID，設 policy=by BSSID | 同時驗證 unicast MAC。 |
| 1112 | `u4FreqInKHz = center_freq` | 鎖頻連線會用到。 |

### 3.3 若目前已連線，決定 disconnect reason

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1116~1127 | 判斷 `fgEqualSsid/fgEqualBssid` | 同 ESS 走 reassociation，否則新連線。 |
| 1123 | `kalIndicateStatusAndComplete(DISCONNECT)` | 非同 SSID 先回報斷線。 |

### 3.4 最終送給 AIS FSM

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1170~1174 | 設 `fgIsConnReqIssued` | 標記有連線請求 in-flight。 |
| 1176~1180 | `fgDelayIndication` | reassoc 場景可延遲 disconnect indication。 |
| 1182 | `mboxSendMsg(... MID_OID_AIS_FSM_JOIN_REQ ...)` | 真正把 connect request 交給 AIS。 |
| 1187 | return success | OID 層完成。 |

---

## 4) AIS 連線核心：`JoinComplete` 路徑逐行註解

## 4.1 `aisFsmRunEventJoinComplete()`（1834~1867）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1851~1854 | state 必須是 JOIN + seq 必須 match | 防止過期事件污染。 |
| 1854 | 呼叫 `aisFsmJoinCompleteAction` | 成敗決策主體。 |
| 1860~1861 | 若 next state 不同就 `aisFsmSteps` | 真正狀態轉移。 |

## 4.2 `aisFsmJoinCompleteAction()` 成功分支（1894~1957）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1894 | `rJoinStatus == SUCCESS` | 連線成功分支。 |
| 1897 | `ucConnTrialCount = 0` | 重試計數歸零。 |
| 1911 | `aisChangeMediaState(CONNECTED)` | 先更新媒體狀態。 |
| 1924 | `aisUpdateBssInfoForJOIN` | 把 AP 參數寫入 BSS info。 |
| 1927 | `cnmStaRecChangeState(... STA_STATE_3)` | STA 記錄進最終連線態。 |
| 1937 | `aisIndicationOfMediaStateToHost(CONNECTED)` | 通知 host 已連上。 |
| 1956 | `eNextState = AIS_STATE_NORMAL_TR` | 進正常收發狀態。 |

## 4.3 `aisFsmJoinCompleteAction()` 失敗分支（1959~2013）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 1961 | `aisFsmStateInit_RetryJOIN` | 可重試就換 auth 再試。 |
| 1965 | `ucJoinFailureCount++` | 失敗統計。 |
| 1968 | `aisFsmReleaseCh` | 釋放頻道資源。 |
| 1971 | 停 join timeout timer | 清舊流程遺留。 |
| 1983~1990 | BSS join fail 次數超閾值 | 暫時降低此 BSS 優先。 |
| 2004~2007 | 若整體 join 超時 | 進 `AIS_STATE_JOIN_FAILURE`。 |
| 2010~2012 | 否則排 `AIS_REQUEST_RECONNECT` | 回 `IDLE` 等重試。 |

---

## 5) 成功回報給 user space：`kalIndicateStatusAndComplete`

**檔案**：`gl_kal.c`  
**位置**：`947~1071`（MEDIA_CONNECT 分支）

| 行號 | 原碼摘要 | 註解 |
|---|---|---|
| 970 | `case WLAN_STATUS_MEDIA_CONNECT` | 進連線成功回報分支。 |
| 975~976 | `wlanoidQueryBssid` + wext event | 先通知舊 WEXT。 |
| 979 | `netif_carrier_on(wlan0)` | 網卡 carrier 拉起。 |
| 983~989 | 查詢 SSID 並印成功 log | 你常見的連線成功印訊在這裡。 |
| 1008~1040 | 確保 cfg80211 BSS 存在 | 若沒有，臨時 `cfg80211_inform_bss` 建立。 |
| 1066~1070 | `cfg80211_connect_result(...)` | 最終給 wpa_supplicant 的成功通知。 |

---

## 6) 斷線補充（連線失敗常一起看）

`gl_kal.c` 的 `WLAN_STATUS_MEDIA_DISCONNECT` 分支（1076 起）會做：

- `netif_carrier_off`
- `cfg80211_disconnected(...)`

若你看到「連上又立刻掉」，這段要一起看。

---

## 7) 實戰排障 checklist（connect）

1. `mtk_cfg80211_connect` 有沒有進？
2. `wlanoidSetAuthMode/EncryptionStatus/Connect` 有沒有都成功？
3. `wlanoidSetConnect` 是否 `mboxSendMsg(MID_OID_AIS_FSM_JOIN_REQ)` 成功？
4. AIS 是否到 `JOIN`？
5. `aisFsmRunEventJoinComplete` 有沒有進？seq match 嗎？
6. 成功後 `cfg80211_connect_result` 有沒有打到？

---

## 8) 一句話收斂

Connect 不是單一函式，而是「一串設定 + 一個狀態機交易」：

- 前半（cfg80211/OID）在準備連線條件
- 中段（AIS/SAA）在真正握手
- 後段（kal/cfg80211）在對 user space 宣告結果

你把這三段切開看，就不會迷路。
