# MTK Wi-Fi Driver Code Flow 與 Firmware / User Space 交互指南（Mantis / MT7668）

> 目標：用「可落地排障」角度，講清楚這份 kernel 中 MTK Wi‑Fi 驅動怎麼跑、Firmware 怎麼下載、User space 怎麼互動。  
> 對象：有 Linux 基礎，但不是 Wi‑Fi driver 開發者。

---

## 0. 範圍與對應版本

本指南對應 repo：`android_kernel_amazon_mantis`（Linux 4.4.120）。

Wi‑Fi 主要路徑：

- `drivers/misc/mediatek/connectivity/wlan/gen4/`
- HIF（SDIO）：`.../os/linux/hif/sdio/sdio.c`
- Glue（Linux 介面）：`.../os/linux/gl_init.c`, `gl_kal.c`, `gl_cfg80211.c`, `gl_wext*.c`, `gl_proc.c`
- Core（共用邏輯）：`.../common/wlan_lib.c`, `wlan_oid.c`
- Mgmt（連線狀態機）：`.../mgmt/ais_fsm.c`
- TX/RX/NIC command/event：`.../nic/nic_tx.c`, `nic_rx.c`, `nic_cmd_event.c`
- Chip（MT7668）：`.../chips/mt7668.c`

---

## 1. 一張圖先看懂：分層架構

```text
User space (wpa_supplicant / hostapd / iw / ifconfig / iwpriv)
   │
   ├─ nl80211/cfg80211 path
   │    gl_cfg80211.c  -> kalIoctl() -> wlan_oid.c -> AIS/QM/NIC -> FW
   │
   ├─ WEXT / private ioctl path (legacy)
   │    gl_wext.c / gl_wext_priv.c -> kalIoctl() -> wlan_oid.c -> FW
   │
   └─ /proc debug path
        gl_proc.c (/proc/net/wlan/*)

Kernel Wi-Fi stack
   ├─ Glue: gl_init.c / gl_kal.c
   ├─ Core: wlan_lib.c / wlan_oid.c
   ├─ Mgmt: ais_fsm.c
   ├─ NIC: nic_tx.c / nic_rx.c / nic_cmd_event.c
   └─ HIF: sdio.c

Hardware/Firmware
   └─ MT7668 N9/CR4 FW + patch (request_firmware + section download)
```

---

## 2. 驅動 Bring-up（從 module load 到網卡可用）

## 2.1 模組入口

- `gl_init.c`
  - `module_init(initWlan)` / `module_exit(exitWlan)` (`2893-2894`)
  - `initWlan()` (`2800`)：初始化 debug level、註冊 bus
  - `glRegisterBus(wlanProbe, wlanRemove)` 由 SDIO 層掛上 (`2836`)

## 2.2 Bus 與 probe

- `os/linux/hif/sdio/sdio.c`
  - `glRegisterBus()` (`700`)：保存 probe/remove callback
  - `glBusInit()`：註冊 `sdio_driver`
  - SDIO 裝置匹配後進入 `wlanProbe()`

- `gl_init.c`
  - `wlanProbe()` (`2203`)
  - 主要工作：
    1. `glBusInit()`
    2. `wlanNetCreate()`：建立 `wiphy + wireless_dev + net_device`
    3. `wlanAdapterStart()`：做 firmware 與 NIC 啟動
    4. 註冊 netdev + procfs
    5. 啟動 `main_thread/hif_thread/rx_thread`

## 2.3 wiphy / netdev 建立（Linux 介面層）

- `gl_init.c:wlanCreateWirelessDevice()` (`1458`)
  - `wiphy_new(&mtk_wlan_ops, ...)`
  - 設定 band/cipher/vendor commands
  - `wiphy_register()` (`1523`)

- `gl_init.c:wlanNetCreate()`
  - `alloc_netdev_mq()` 建立 `wlan0`
  - 綁定 `netdev_ops = wlan_netdev_ops`
  - `wireless_handlers = &wext_handler_def`
  - `ieee80211_ptr = wireless_dev`

---

## 3. Firmware 下載與初始化流程（重點）

## 3.1 入口（Adapter Start）

- `common/wlan_lib.c:wlanAdapterStart()` (`318`)

這個函數是整個 Wi‑Fi bring-up 的主控流程，包含：

1. Power ownership / 初始 reset
2. 讀 chip info / capability
3. 下載 patch + FW
4. 啟用 interrupt 與 TX/RX 路徑

## 3.2 Firmware 檔名策略

- `gl_kal.c:kalFirmwareImageMapping()` (`344`)
- `gl_kal.c:kalFirmwareOpen()` (`175`)
- `chips/mt7668.c:mt7668ConstructFirmwarePrio()` (`83`)

檔名會依 chip id + ECO 版本組候選清單，例如：

- `WIFI_RAM_CODE_MT7668_E1`
- `WIFI_RAM_CODE_MT7668_E1.bin`
- `WIFI_RAM_CODE_MT7668`
- `WIFI_RAM_CODE_MT7668.bin`

Patch 另外走：

- `mt7668_patch_e1_hdr.bin`（格式：`mt%x_patch_e%x_hdr.bin`）

## 3.3 Firmware 搜尋路徑

`kalFirmwareOpen()` 註解已明示（`gl_kal.c:187`）：

- Android 常見：`/etc/firmware`, `/vendor/firmware`, `/firmware/image`
- Linux 常見：`/lib/firmware`, `/lib/firmware/update`

內部使用 `request_firmware()`。

## 3.4 真正下載到晶片

- `wlanDownloadFW()` (`3553`)
- `wlanDownloadPatch()` (`3653`)
- `wlanImageSectionDownload()` (`3038`)
- `wlanImageSectionDownloadStatus()` (`3216`)

實務上是「分段下載 section + 查每段 status」。

---

## 4. Thread / Queue 執行模型（命令與資料怎麼流）

## 4.1 三個核心 thread

- `main_thread()` (`2834`)：處理 OID、TX request、timer、部分 RX
- `hif_thread()` (`2639`)：處理 HIF/interrupt 相關
- `rx_thread()` (`2733`)：把 RX packet 往 OS 網路棧上送

## 4.2 OID 走法（控制面）

- `kalIoctl()` 會設 `GLUE_FLAG_OID_BIT` (`2162`)
- `main_thread()` 中抓到 OID flag 後呼叫：
  - `wlanSetInformation()` / `wlanQueryInformation()` (`gl_kal.c:2971`)
- OID handler 實際在 `wlan_oid.c`

## 4.3 Data Tx 走法（資料面）

1. `wlanHardStartXmit()` (`gl_init.c:984`)
2. `kalHardStartXmit()` (`gl_kal.c:1413`)：分類封包、入 tx queue、喚醒事件
3. `kalProcessTxReq()` (`gl_kal.c:2513`)：處理 command + packet
4. `wlanProcessTxFrame()` (`wlan_lib.c:4256`)：標記封包屬性
5. `nicTxEnqueueMsdu()` (`nic_tx.c:2495`)：進入 NIC/QM

## 4.4 Data Rx 走法

1. IRQ/HIF 觸發
2. `nicRxProcessRFBs()` (`nic_rx.c:2728`) 消化 RX buffer
3. `rx_thread()` 把封包 `kalRxIndicateOnePkt()` 上送 Linux 網路棧

---

## 5. User space 交互（你最常碰到的）

## 5.1 主路徑：cfg80211 / nl80211（wpa_supplicant 現代路徑）

`gl_init.c` 中的 `mtk_wlan_ops`（`337`）把 nl80211 操作綁到 driver：

- `.scan = mtk_cfg80211_scan`
- `.connect = mtk_cfg80211_connect`
- `.disconnect = mtk_cfg80211_disconnect`
- `.add_key/.del_key/...` 等

對應函數：

- `mtk_cfg80211_scan()` (`gl_cfg80211.c:815`)
- `mtk_cfg80211_connect()` (`900`)
- `mtk_cfg80211_disconnect()` (`1295`)

### Scan 典型流

`wpa_supplicant/iw` -> `mtk_cfg80211_scan` -> `kalIoctl(wlanoidSetBssidListScanAdv)` (`wlan_oid.c:686`) -> `aisFsmScanRequestAdv()` (`ais_fsm.c:3290`) -> FW 掃描 -> `EVENT_ID_SCAN_DONE` -> `kalIndicateStatusAndComplete(WLAN_STATUS_SCAN_COMPLETE)` (`gl_kal.c:947` 分派) -> `cfg80211_scan_done`

### Connect 典型流

`wpa_supplicant` -> `mtk_cfg80211_connect` -> `kalIoctl(wlanoidSetConnect)` (`wlan_oid.c:1042`) -> `mboxSendMsg(MID_OID_AIS_FSM_JOIN_REQ)` -> AIS FSM (`ais_fsm.c`) -> firmware auth/assoc -> `aisFsmRunEventJoinComplete()` (`1834`) / `aisUpdateBssInfoForJOIN()` (`2436`) -> `kalIndicateStatusAndComplete(WLAN_STATUS_MEDIA_CONNECT)` -> `cfg80211_connect_result`

## 5.2 舊路徑：WEXT / private ioctl

- `gl_wext.c` / `gl_wext_priv.c`
- 仍可透過 `iwconfig/iwpriv` 或 Android private command 進行控制
- 例如 `gl_wext_priv.c` 有大量 `CMD_*`（如 COUNTRY/SETROAMMODE/DRIVER/SETSUSPENDMODE 等）

## 5.3 /proc 偵錯路徑

- root：`/proc/net/wlan`（`PROC_ROOT_NAME`）
- 典型節點（`gl_proc.c`）：
  - `mcr`
  - `driver`
  - `dbg_level`
  - `cfg`
  - `efuse_dump`
  - `csi_data`
  - `get_txpwr_tbl`

對應建立在 `procCreateFsEntry()` (`1298`)。

---

## 6. Firmware 與 User space 的「橋接點」

可以把它理解成三個橋：

1. **User space -> Driver API 橋**
   - nl80211/cfg80211 或 wext/private ioctl
2. **Driver 控制面 -> Firmware 橋**
   - `kalIoctl -> wlan_oid -> wlanSendSetQueryCmd`（command/event）
3. **Firmware 事件 -> User space 回報橋**
   - `nic_cmd_event.c` 解析事件
   - `kalIndicateStatusAndComplete()` -> cfg80211 回報（connect/disconnect/scan done）

---

## 7. 實際排障指南（可直接照做）

## 7.1 Firmware 相關問題（最常見）

症狀：probe fail、wlan0 起不來、`request_firmware` failed

先檢查：

1. 韌體檔是否存在（例如 `/lib/firmware/...` 或 Android firmware 路徑）
2. 檔名是否符合 MT7668 + ECO 命名規則
3. dmesg 是否有 `Request FW image ... failed`

## 7.2 Scan 失敗

看三段：

1. `mtk_cfg80211_scan` 有沒有被打到
2. `wlanoidSetBssidListScanAdv` 是否 return success
3. 是否有 `EVENT_ID_SCAN_DONE` 回來

## 7.3 連線失敗（AUTH/ASSOC）

看四段：

1. `mtk_cfg80211_connect`
2. `wlanoidSetConnect`
3. AIS FSM（`aisFsmRunEventJoinComplete`）
4. `cfg80211_connect_result` 回報內容

## 7.4 可做的最小觀測點

- `dmesg -w`（核心 log）
- `iw dev` / `iw dev wlan0 link`
- `wpa_cli -i wlan0 status`
- `/proc/net/wlan/driver`、`/proc/net/wlan/dbg_level`

---

## 8. 建議你先掌握的 12 個關鍵函數

1. `initWlan` (`gl_init.c:2800`)
2. `wlanProbe` (`gl_init.c:2203`)
3. `wlanAdapterStart` (`wlan_lib.c:318`)
4. `kalFirmwareOpen` (`gl_kal.c:175`)
5. `kalFirmwareImageMapping` (`gl_kal.c:344`)
6. `wlanDownloadFW` (`wlan_lib.c:3553`)
7. `wlanDownloadPatch` (`wlan_lib.c:3653`)
8. `main_thread` (`gl_kal.c:2834`)
9. `kalProcessTxReq` (`gl_kal.c:2513`)
10. `mtk_cfg80211_scan` (`gl_cfg80211.c:815`)
11. `mtk_cfg80211_connect` (`gl_cfg80211.c:900`)
12. `wlanoidSetConnect` (`wlan_oid.c:1042`)

---

## 9. 一句話總結

這顆 MTK Wi‑Fi driver 的核心設計是：

- **Linux 介面層（cfg80211/wext）** 接 user space 指令，
- 經 **OID + command/event queue** 轉成 firmware 可理解的命令，
- 再由 **NIC/HIF thread** 驅動實體 SDIO 收發，
- 最後把結果透過 **cfg80211 事件** 回報給 wpa_supplicant/系統。

你抓住這條主線，排障會快很多。
