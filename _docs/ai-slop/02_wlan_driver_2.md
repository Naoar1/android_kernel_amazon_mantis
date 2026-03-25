# MTK Wi‑Fi Driver 淺顯但具體版（Mantis / MT7668）

> 你現在在學 driver code，所以這版會保留細節，但用白話說。  
> 目標：你可以「照著函式一路追下去」，不是只看概念。

---

## 0) 先講人話：這個 driver 在做什麼？

一句話：

**把 user space（wpa_supplicant/iw）的 Wi‑Fi 指令，翻譯成 firmware 命令；再把 firmware 回來的結果，翻回 Linux 事件。**

你可以把它當成 3 層：

1. **入口層**（Linux API）
   - `cfg80211/nl80211`（主流）
   - `wext/private ioctl`（舊路）
2. **翻譯層**（MTK glue + OID + FSM）
3. **硬體層**（NIC + SDIO + MT7668 firmware）

---

## 1) 你先認得 4 個核心資料結構（很重要）

## 1.1 `GLUE_INFO_T`（Linux glue 的大總管）

檔案：`os/linux/include/gl_os.h`（`struct _GLUE_INFO_T`）

常用欄位（你追 log 時會一直遇到）：

- `prDevHandler`：`wlan0` 的 net_device
- `prAdapter`：指向內核核心結構 `ADAPTER_T`
- `ulFlag`：執行緒事件旗標（OID/TXREQ/RX/HALT）
- `OidEntry`：當前 OID 請求內容
- `prScanRequest`：目前 cfg80211 scan request
- `main_thread`, `hif_thread`, `rx_thread`
- `rCmdQueue`, `rTxQueue`
- `u4ReadyFlag`（卡是否 ready）

## 1.2 `ADAPTER_T`（Wi‑Fi 核心狀態）

前置宣告在 `include/typedef.h`，主要定義分散在 `include/nic/adapter.h`。

你先記住：
- `rWifiVar.rConnSettings`：連線參數（SSID/BSSID/Auth/Enc）
- `prAisBssInfo`：目前 station 連線 BSS 狀態

## 1.3 `CONNECTION_SETTINGS_T`

檔案：`include/nic/adapter.h`

關鍵欄位：
- `aucSSID`, `ucSSIDLen`
- `aucBSSID`
- `eConnectionPolicy`（by SSID / by BSSID）
- `eAuthMode`, `eEncStatus`
- `fgIsConnReqIssued`

## 1.4 `CMD_INFO_T`

檔案：`include/nic/cmd_buf.h`

它是「送給 firmware 的命令封包描述」：
- `ucCID`（command id）
- `fgSetQuery` / `fgNeedResp` / `fgIsOid`
- `pucInfoBuffer`、`u2InfoBufLen`
- `pfCmdDoneHandler` / `pfCmdTimeoutHandler`

---

## 2) 從開機到 wlan0 ready：逐步追（含函式）

## Step A：模組入口

檔案：`gl_init.c`

- `module_init(initWlan)` (`2893`)
- `initWlan()` (`2800`)
  - 註冊 bus callback：`glRegisterBus(wlanProbe, wlanRemove)` (`2836`)

## Step B：SDIO 層偵測裝置

檔案：`hif/sdio/sdio.c`

- `glRegisterBus()` (`700`)：真正做 `sdio_register_driver()`
- SDIO match 後進 `mtk_sdio_probe()` (`427`)
  - `sdio_enable_func(func)`
  - 呼叫 `pfWlanProbe(...)`（也就是上層的 `wlanProbe`）

## Step C：wlanProbe 主流程

檔案：`gl_init.c:wlanProbe()` (`2203`)

你在這裡會看到：

1. `glBusInit()`：SDIO block size/IRQ 等初始化
2. `wlanCreateWirelessDevice()`：
   - `wiphy_new(&mtk_wlan_ops, ...)` (`1458`)
   - `wiphy_register(...)` (`1523`)
3. `wlanNetCreate()`：
   - `alloc_netdev_mq()` 建立 `wlan0`
   - `netdev_ops = &wlan_netdev_ops`
   - `wireless_handlers = &wext_handler_def`
   - `ieee80211_ptr = prWdev`
4. `wlanAdapterStart()`（`wlan_lib.c:318`）
   - 會進入 firmware/patch 下載
5. 起 thread：
   - `main_thread` / `hif_thread` / `rx_thread`
6. `wlanoidQueryCurrentAddr` 拿 MAC，設 `u4ReadyFlag=1`
7. `wlanNetRegister()` 註冊網卡
8. `procCreateFsEntry()` 建 `/proc/net/wlan/*`

---

## 3) Firmware 下載：具體怎麼跑

## 3.1 入口函式

- `wlanAdapterStart()` (`wlan_lib.c:318`)
- 下載函式：
  - `wlanDownloadPatch()` (`3653`)
  - `wlanDownloadFW()` (`3553`)
  - `wlanImageSectionDownload()` (`3038`)

## 3.2 檔名如何決定

檔案：`chips/mt7668.c`, `gl_kal.c`

- `mt7668ConstructFirmwarePrio()` (`83`) 依 chip id + ECO 生成候選檔名
- `kalFirmwareImageMapping()` (`344`) 做候選組合
- `kalFirmwareOpen()` (`175`) 逐個 `request_firmware()` 嘗試

`config.h` 內基底名稱：
- `CFG_FW_FILENAME = "WIFI_RAM_CODE"`
- `CFG_CR4_FW_FILENAME = "WIFI_RAM_CODE2"`

patch 命名可見：
- `mt%x_patch_e%x_hdr.bin`

## 3.3 Firmware 搜尋路徑（程式註解直接寫）

`kalFirmwareOpen()` 註解說明：

- Android: `/etc/firmware`, `/vendor/firmware`, `/firmware/image`
- Linux: `/lib/firmware`, `/lib/firmware/update`

## 3.4 找不到 FW 的典型症狀

- `wlanProbe` fail
- `wlan0` 沒起來
- dmesg 出現 `Request FW image ... failed`

---

## 4) thread 與旗標：主迴圈怎麼工作

## 4.1 事件旗標（重點）

檔案：`gl_os.h`

常見 bit：
- `GLUE_FLAG_OID_BIT` (2)
- `GLUE_FLAG_TXREQ_BIT` (4)
- `GLUE_FLAG_RX_BIT` (10)
- `GLUE_FLAG_HALT_BIT` (0)

## 4.2 `main_thread()` 實際做什麼

檔案：`gl_kal.c:2834`

主流程：
1. `wait_event_interruptible(waitq, ulFlag 有事)`
2. 若 `OID_BIT`：
   - 取 `OidEntry`
   - `wlanSetInformation()` 或 `wlanQueryInformation()`
   - `complete(&rPendComp)` 喚醒 ioctl caller
3. 若 `TXREQ_BIT`：`kalProcessTxReq()`
4. 若 `RX_BIT`：`nicRxProcessRFBs()`（多執行緒模式）

這就是 driver 內部「事件驅動迴圈」。

---

## 5) OID 機制：最重要的翻譯層

## 5.1 `kalIoctl()` 的實際流程

檔案：`gl_kal.c:kalIoctlTimeout()`

具體步驟：

1. 先上鎖（`g_halt_sem` + `ioctl_sem`）
2. 把 OID 填進 `prGlueInfo->OidEntry`
3. 設 `GLUE_FLAG_OID_BIT`
4. `wake_up_interruptible(waitq)` 喚醒 `main_thread`
5. 呼叫端 `wait_for_completion(rPendComp)` 等結果
6. `main_thread` 做完後 `complete()` 回來

這就是 user request 到 driver 核心邏輯的同步橋樑。

---

## 6) Scan 路徑（用真實函式一路追）

## 6.1 cfg80211 入口

- `mtk_cfg80211_scan()` (`gl_cfg80211.c:815`)

它做了哪些「具體」事情：
- 讀 `request->n_ssids`
- 複製 `request->ssids[]` 到 `PARAM_SCAN_REQUEST_ADV_T`
- 帶上 `request->ie` / `ie_len`
- 若有指定 channel，轉成 driver 內格式
- 呼叫 `kalIoctl(... wlanoidSetBssidListScanAdv ... )`

## 6.2 OID 到 FSM

- `wlanoidSetBssidListScanAdv()` (`wlan_oid.c:686`)
  - 轉參數
  - 呼叫 `aisFsmScanRequestAdv()` (`ais_fsm.c:3290`)
  - 啟動 scan done timer

## 6.3 回報到 user space

- firmware 回 scan done event
- `kalIndicateStatusAndComplete(WLAN_STATUS_SCAN_COMPLETE)` (`gl_kal.c:947`)
- 內部呼叫 `kalCfg80211ScanDone(prScanRequest, FALSE)`
- user space 收到 scan 完成

---

## 7) Connect 路徑（具體到欄位）

## 7.1 cfg80211 connect 入口

- `mtk_cfg80211_connect()` (`gl_cfg80211.c:900`)

它不是只「一個 connect call」而已，會先做完整前置：

1. `wlanoidSetInfrastructureMode`
2. 解析 `sme->crypto.akm_suites[]` 決定 `eAuthMode`
3. 解析 cipher 決定 `eEncStatus`
4. 若有 IE（WPS/HS20...）分別下 OID
5. `wlanoidSetAuthMode`
6. `wlanoidSetEncryptionStatus`
7. 最後 `wlanoidSetConnect`

## 7.2 `wlanoidSetConnect()` 真正在做的事

檔案：`wlan_oid.c:1042`

具體重點：
- 驗證參數長度（`sizeof(PARAM_CONNECT_T)`）
- 寫入 `rConnSettings`：
  - `aucSSID`, `aucBSSID`
  - `eConnectionPolicy`（by SSID / by BSSID）
  - `u4FreqInKHz`
- 判斷是否 re-association
- 建 `MSG_AIS_ABORT_T`
- `mboxSendMsg(... MID_OID_AIS_FSM_JOIN_REQ ...)`

=> 這裡不是直接連 AP，而是把「連線請求」丟進 AIS 狀態機。

## 7.3 連上後如何回報

AIS 完成 join 後：
- `aisFsmRunEventJoinComplete()` (`ais_fsm.c:1834`)
- `aisUpdateBssInfoForJOIN()` (`2436`)

最後在 glue 層：
- `kalIndicateStatusAndComplete(WLAN_STATUS_MEDIA_CONNECT)`
- `cfg80211_connect_result(...)` 回報給 user space

---

## 8) Disconnect 路徑（實際）

- `mtk_cfg80211_disconnect()` (`gl_cfg80211.c:1295`)
- 呼叫：`kalIoctl(... wlanoidSetDisassociate ...)`
- `wlanoidSetDisassociate()` (`wlan_oid.c:7814`)
  - 清掉 `fgIsConnReqIssued`
  - 送 `MID_OID_AIS_FSM_JOIN_REQ`（abort/disconnect）
  - 若原本已連線，呼叫 `kalIndicateStatusAndComplete(...DISCONNECT_LOCALLY...)`
- glue 層最終 `cfg80211_disconnected(...)`

---

## 9) 資料封包（TX/RX）路徑

## TX（送出）

1. `wlanHardStartXmit()` (`gl_init.c:984`)
2. `kalHardStartXmit()` (`gl_kal.c:1413`)：分類封包、入 queue
3. `kalProcessTxReq()` (`2513`)：處理 tx queue + cmd queue
4. `wlanProcessTxFrame()` (`wlan_lib.c:4256`)：封包 metadata
5. `nicTxEnqueueMsdu()` (`nic_tx.c:2495`)

## RX（接收）

1. HIF / interrupt
2. `nicRxProcessRFBs()` (`nic_rx.c:2728`)
3. `rx_thread` / `kalRxIndicateOnePkt()` 上送 Linux

---

## 10) User space 三種互動方式（你真的會用到）

## 10.1 主路徑：cfg80211 / nl80211

- 來源：wpa_supplicant / iw
- 對應：`mtk_cfg80211_*`
- 優點：標準化、主流、問題最好排

## 10.2 舊路徑：WEXT / private cmd

- 檔案：`gl_wext.c`, `gl_wext_priv.c`
- `priv_driver_cmds()` (`10781`)
- 支援很多命令字串（例：`SETFWPATH`, `COUNTRY`, `SETROAMMODE`, `SET_FWLOG`...）

## 10.3 /proc debug

- 檔案：`gl_proc.c`
- root：`/proc/net/wlan`（`PROC_ROOT_NAME`）
- 常用節點：`driver`, `dbg_level`, `mcr`, `cfg`...
- `procDriverCmdWrite()` 會把字串丟給 `priv_driver_cmds()`

---

## 11) 你可以直接照抄的排障腳本（新手友善）

## 11.1 看 firmware 問題

```bash
dmesg -T | grep -Ei "wlan|firmware|mtk|mt7668|request fw|patch"
```

觀察重點：
- 有沒有 `Request FW image ... failed`
- `wlanProbe` 最終是 success 還是 fail

## 11.2 看 scan/connect 流程有沒有走完

```bash
dmesg -T | grep -Ei "scan|connect|disconnect|join|assoc|deauth|cfg80211"
```

## 11.3 看 /proc driver/debug

```bash
ls /proc/net/wlan
cat /proc/net/wlan/driver
cat /proc/net/wlan/dbg_level
```

## 11.4 看 user space 狀態

```bash
iw dev
iw dev wlan0 link
wpa_cli -i wlan0 status
```

---

## 12) 你讀碼時可以用的「最短路徑」

如果你現在只想吃透「連線」：

1. `mtk_cfg80211_connect`（入口）
2. `wlanoidSetConnect`（翻譯）
3. `aisFsmRunEventJoinComplete`（FSM完成）
4. `kalIndicateStatusAndComplete`（回報）
5. `cfg80211_connect_result`（user space看到成功）

這五個點打通，你就真正懂這顆 driver 的核心交互了。

---

## 13) 黑話翻譯表（最後再看）

- **cfg80211**：Linux 標準 Wi‑Fi API 框架
- **nl80211**：user space 跟 cfg80211 溝通的 netlink 協定
- **OID**：driver 內部控制命令介面（你可當成 API）
- **AIS FSM**：Station 模式連線狀態機
- **HIF**：Host Interface（這裡是 SDIO）
- **RFB/MSDU**：接收/傳送封包控制結構

---

如果你要，我下一份可以做「**逐行帶讀 Connect**」：
我會用「每 10~20 行做一段白話註解」方式，從 `mtk_cfg80211_connect()` 一路走到 `cfg80211_connect_result()`，讓你可以邊對照邊學。
