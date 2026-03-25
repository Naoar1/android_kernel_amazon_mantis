# Mantis Kernel 與 MTK Wi-Fi Driver 架構參考

Repo: `android_kernel_amazon_mantis`  
裝置: Fire TV Stick 4K（代號 mantis）  
SoC: MediaTek MT8695  
Wi-Fi 晶片: MediaTek MT7668  
Kernel 版本: Linux 4.4.120  
Wi-Fi driver 路徑: `drivers/misc/mediatek/connectivity/wlan/gen4/`

---

## 1. Linux Wi-Fi 子系統背景

### 1.1 cfg80211 與 nl80211

Linux kernel 對 Wi-Fi 裝置提供兩層抽象：

**cfg80211**（`net/wireless/`）是 kernel 內部的 Wi-Fi 設定框架。任何 Wi-Fi driver 只要實作 `struct cfg80211_ops` 裡定義的 function pointer，就能接入標準的 Linux Wi-Fi 管理機制，不必自己實作掃描結果管理、BSS 資料庫、regulatory domain 等邏輯。這個框架在 2.6.22 引入，目的是取代每個 driver 各自實作的雜亂介面。

**nl80211**（`net/wireless/nl80211.c`）是 cfg80211 對 user space 暴露的 netlink 介面。Netlink 是 Linux 上 kernel 與 user space 之間傳遞結構化訊息的 socket 機制（`AF_NETLINK`）。nl80211 定義了一套訊息格式，`wpa_supplicant`、`iw` 等工具透過這個介面下達掃描、連線、斷線等指令，並接收事件（scan 完成、連線成功、斷線等）。

**WEXT（Wireless Extensions）** 是更舊的介面，透過 `ioctl` 傳遞 `struct iwreq`。仍存在於 driver 中，但已 deprecated，主要為了向後相容 `iwconfig` 等舊工具。

MTK driver 同時實作了兩者：
- `os/linux/gl_cfg80211.c`：實作 `struct cfg80211_ops`，對應 nl80211 路徑
- `os/linux/gl_wext.c` / `gl_wext_priv.c`：實作 WEXT 路徑

### 1.2 SDIO（Secure Digital Input Output）

MT7668 是一顆獨立的 Wi-Fi/BT combo 晶片，透過 SDIO 介面連接到 MT8695 SoC。SDIO 是 SD card 介面的功能擴充版本，允許連接非儲存裝置（Wi-Fi、BT、GPS 等）。在 Linux 中，SDIO 裝置由 `drivers/mmc/` 管理，driver 透過 `sdio_register_driver()` 註冊，匹配規則是 `sdio_device_id` 表中的 vendor/device ID。

MT7668 在 `mantis.dts` 中對應的是 `mmc2` 節點（SDIO 控制器），標記為 non-removable（不可熱插拔）。

### 1.3 OID（Object Identifier）

OID 是 MTK driver 內部的控制命令介面。這個命名來自 Microsoft NDIS（Network Driver Interface Specification），是 Windows 網路 driver 的標準介面。MTK 早期 driver 是從 Windows 平台移植過來的，因此保留了 OID 命名慣例，但在 Linux 上以不同機制實作（用 completion + waitqueue 而非 Windows 的 NDIS request）。

每個 OID 對應一個操作（如 `wlanoidSetConnect`、`wlanoidSetAuthMode`），由 `main_thread` 序列執行，保證 driver 核心邏輯的 thread safety。

### 1.4 FSM（Finite State Machine，有限狀態機）

Wi-Fi 連線是一個有明確狀態轉換的程序（idle → 掃描 → 找 AP → 要求頻道 → auth → assoc → 已連線），適合用狀態機建模。MTK driver 的 AIS FSM（`ais_fsm.c`）以明確的狀態列舉和 transition 函式實作這個狀態機，避免了用一堆 flag 拼湊出的隱式狀態機所帶來的難以維護問題。

---

## 2. 整體分層架構

```
User space
  wpa_supplicant / iw / iwconfig / iwpriv
       │
       │ nl80211（netlink AF_NETLINK）        WEXT（ioctl SIOCGIWSCAN 等）
       ▼
  Linux kernel: cfg80211 框架 / WEXT 框架
       │
       ▼
  MTK driver 入口層（os/linux/）
    gl_cfg80211.c   — struct cfg80211_ops 實作
    gl_wext.c       — WEXT ioctl 實作
    gl_wext_priv.c  — private ioctl（iwpriv）實作
    gl_proc.c       — /proc/net/wlan/* debug 節點
    gl_kal.c        — OS 抽象層（Kernel Abstraction Layer）
    gl_init.c       — 模組 init / probe / netdev 建立
       │
       │ OID（透過 kalIoctl → main_thread）
       ▼
  MTK driver 核心層（common/）
    wlan_lib.c      — 通用函式、firmware 下載、adapter 啟動
    wlan_oid.c      — 所有 OID handler
       │
       ▼
  MTK driver 管理層（mgmt/）
    ais_fsm.c       — AIS（Infrastructure STA）連線狀態機
    ais_fsm.h       — AIS 狀態/結構定義
    （其他：roaming FSM、SAA FSM、SCN 等）
       │
       ▼
  MTK driver NIC 層（nic/）
    nic_tx.c        — TX 封包入列、送出
    nic_rx.c        — RX 封包接收、分發
    nic_cmd_event.c — firmware command/event 解析
       │
       │ SDIO bus 傳輸
       ▼
  HIF 層（os/linux/hif/sdio/）
    sdio.c          — SDIO bus driver、probe、TX/RX DMA
       │
       ▼
  MT7668 硬體 + firmware（N9 + CR4 兩顆 MCU）
```

`gl_kal.c` 的 KAL（Kernel Abstraction Layer）是一層 OS 抽象，目的是讓 `common/`、`mgmt/`、`nic/` 等核心邏輯不直接呼叫 Linux API，而透過 `kal*` 函式間接呼叫，理論上可以在不同 OS 上移植核心邏輯。

---

## 3. 核心資料結構

### 3.1 `GLUE_INFO_T`（`os/linux/include/gl_os.h`）

```c
typedef struct _GLUE_INFO_T {
    struct net_device      *prDevHandler;
    struct ADAPTER         *prAdapter;
    unsigned long           ulFlag;
    OID_ENTRY_T             OidEntry;
    struct cfg80211_scan_request *prScanRequest;
    struct task_struct     *main_thread;
    struct task_struct     *hif_thread;
    struct task_struct     *rx_thread;
    wait_queue_head_t       waitq;
    struct completion       rPendComp;
    /* ... */
} GLUE_INFO_T;
```

`GLUE_INFO_T` 是 MTK driver 對 Linux OS 層的主要上下文結構。命名中 `GLUE` 指「黏合層」，把 Linux kernel 物件（`net_device`、`wiphy`、`task_struct` 等）和 MTK 核心物件（`ADAPTER_T`）黏合在一起。`_T` 後綴是 MTK 程式碼的 typedef 慣例，所有自定義型別都有這個後綴。

**欄位說明：**

`prDevHandler`（`struct net_device *`）：Linux 網路裝置物件，代表 `wlan0`。`struct net_device` 是 Linux 網路子系統的核心結構，包含 MAC 位址、統計資料、TX queue、以及 `netdev_ops`（send packet、change MTU 等 function pointer）。`pr` 前綴是 MTK 慣例，表示 pointer to struct。

`prAdapter`（`ADAPTER_T *`）：指向 MTK driver 的核心狀態結構 `ADAPTER_T`，包含所有 Wi-Fi 協定層狀態。`GLUE_INFO_T` 屬於 OS 層，`ADAPTER_T` 屬於獨立於 OS 的核心層，兩者分離是為了維持可移植性。

`ulFlag`（`unsigned long`）：執行緒事件旗標，使用 bit 操作（`set_bit`/`test_bit`）。每個 bit 代表一種待處理事件，`main_thread` 的 `wait_event_interruptible` 在任意 bit 被設置時喚醒。見第 4.1 節。

`OidEntry`（`OID_ENTRY_T`）：當前待執行的 OID 請求，包含 handler function pointer、參數 buffer、長度等。同一時間只有一個 OID 在執行，由 semaphore 保護。

`prScanRequest`（`struct cfg80211_scan_request *`）：當前進行中的 cfg80211 scan request 物件。cfg80211 framework 在呼叫 `.scan` op 時傳入這個指標；driver 在 scan 完成時將它傳回給 `cfg80211_scan_done()`。非 NULL 代表 scan in-flight，此時拒絕新的 scan 請求（回傳 `-EBUSY`）。

`main_thread` / `hif_thread` / `rx_thread`（`struct task_struct *`）：Linux 核心對 process/thread 的統一描述結構（Linux 不區分 process 和 thread，都是 `task_struct`）。儲存對應 kernel thread 的指標，用於 `kthread_stop()` 等操作。

`waitq`（`wait_queue_head_t`）：Linux wait queue，`main_thread` 在這上面等待事件（透過 `wait_event_interruptible`）。其他 thread 設置 `ulFlag` 後呼叫 `wake_up_interruptible(&waitq)` 喚醒 `main_thread`。

`rPendComp`（`struct completion`）：Linux completion 物件，用於同步 `kalIoctl` 呼叫端和 `main_thread`。呼叫端執行 `wait_for_completion()`，`main_thread` 完成 OID 後執行 `complete()`。這是 Linux kernel 中比 semaphore 更語義清晰的一次性同步原語。

### 3.2 `ADAPTER_T`（`include/nic/adapter.h`）

```c
typedef struct ADAPTER {
    WIFI_VAR_T          rWifiVar;
    BSS_INFO_T         *prAisBssInfo;
    AIS_FSM_INFO_T      rAisFsmInfo;
    /* 大量其他欄位 */
} ADAPTER_T;
```

獨立於 OS 的 Wi-Fi 核心狀態結構，代表整個 Wi-Fi adapter 的邏輯狀態。設計上不含任何 Linux 特定型別，以維持核心邏輯的可移植性。透過 `GLUE_INFO_T.prAdapter` 存取。

`rWifiVar`（`WIFI_VAR_T`）：Wi-Fi 各種設定與狀態變數的容器。`r` 前綴是 MTK 慣例，表示 struct 型別的非指標變數（相對於 pointer 的 `p` 前綴）。`rWifiVar.rConnSettings`（`CONNECTION_SETTINGS_T`）儲存目前的連線目標參數。

`prAisBssInfo`（`BSS_INFO_T *`）：目前 STA 連線所在 BSS（Basic Service Set）的資訊，包含 AP 的 BSSID、使用的頻道、連線狀態等。BSS 是 802.11 術語，一個 BSS 對應一個 AP 提供的無線網路單元。

`rAisFsmInfo`（`AIS_FSM_INFO_T`）：AIS FSM 的完整狀態，包含當前狀態、序號、timer 等。見 `04_ais_fsm.md`。

### 3.3 `CONNECTION_SETTINGS_T`（`include/nic/adapter.h`）

```c
typedef struct _CONNECTION_SETTINGS_T {
    UINT_8              aucSSID[ELEM_MAX_LEN_SSID];  /* 32 bytes */
    UINT_8              ucSSIDLen;
    UINT_8              aucBSSID[MAC_ADDR_LEN];      /* 6 bytes */
    ENUM_CONN_POLICY_T  eConnectionPolicy;
    ENUM_AUTH_MODE_T    eAuthMode;
    ENUM_ENCRYPTION_TYPE_T eEncStatus;
    BOOLEAN             fgIsConnReqIssued;
    BOOLEAN             fgIsConnByBssidIssued;
    UINT_32             u4FreqInKHz;
    /* ... */
} CONNECTION_SETTINGS_T;
```

儲存目前連線的目標參數，位於 `ADAPTER_T.rWifiVar.rConnSettings`。這些欄位由 `wlanoidSetConnect()` 寫入，由 AIS FSM 讀取作為搜尋目標。

`aucSSID` / `ucSSIDLen`：目標 SSID。`auc` 前綴表示 array of unsigned char。SSID 最長 32 bytes（`ELEM_MAX_LEN_SSID`），不一定是 null-terminated string，長度由 `ucSSIDLen` 給出。

`aucBSSID`：目標 AP 的 BSSID，即 AP radio 的 MAC 位址（6 bytes，`MAC_ADDR_LEN = 6`）。若 user space 指定 BSSID 連線，這裡非零。

`eConnectionPolicy`（`ENUM_CONN_POLICY_T`）：連線策略列舉。`e` 前綴是 MTK 慣例，表示 enum 型別。`CONNECT_BY_SSID_BEST_RSSI` 表示找所有符合 SSID 的 AP 中訊號最強的；`CONNECT_BY_BSSID` 表示只連指定 BSSID 的 AP。

`eAuthMode`（`ENUM_AUTH_MODE_T`）：認證模式，如 `AUTH_MODE_WPA2_PSK`、`AUTH_MODE_WPA2`（802.1X）、`AUTH_MODE_OPEN` 等。由 `mtk_cfg80211_connect()` 從 `sme->crypto.akm_suites` 解析而來。

`eEncStatus`（`ENUM_ENCRYPTION_TYPE_T`）：加密狀態，如 `ENUM_ENCRYPTION3_ENABLED`（CCMP/AES）、`ENUM_ENCRYPTION2_ENABLED`（TKIP）、`ENUM_ENCRYPTION1_ENABLED`（WEP）、`ENUM_ENCRYPTION_DISABLED`。

`fgIsConnReqIssued`（`BOOLEAN`）：`fg` 前綴是 MTK 慣例，表示 flag（bool）。標記目前是否有連線請求正在進行（in-flight）。AIS FSM 在 IDLE state 時檢查此欄位決定是否進入 SEARCH state。

`u4FreqInKHz`（`UINT_32`）：`u4` 前綴表示 unsigned 32-bit integer。若 user space 指定連線頻率，以 KHz 為單位儲存。0 表示不限頻率。

### 3.4 `CMD_INFO_T`（`include/nic/cmd_buf.h`）

```c
typedef struct _CMD_INFO_T {
    UINT_8                  ucCID;
    BOOLEAN                 fgSetQuery;
    BOOLEAN                 fgNeedResp;
    BOOLEAN                 fgIsOid;
    PUINT_8                 pucInfoBuffer;
    UINT_16                 u2InfoBufLen;
    PFN_CMD_DONE_HANDLER    pfCmdDoneHandler;
    PFN_CMD_TIMEOUT_HANDLER pfCmdTimeoutHandler;
    /* ... */
} CMD_INFO_T;
```

描述一個送往 firmware 的命令封包。與 `OID_ENTRY_T`（driver 層的 OID 請求）不同，`CMD_INFO_T` 是最終送到 MT7668 firmware 的命令描述符。

`ucCID`：Command ID，firmware 側定義的命令編號，例如 `CMD_ID_NETWORK_ATTACH`、`CMD_ID_SET_BSS_INFO` 等。`uc` 前綴表示 unsigned char（8-bit）。

`fgSetQuery`：TRUE 表示這是 set 命令（改變 firmware 狀態），FALSE 表示 query 命令（讀取 firmware 狀態）。

`fgNeedResp`：是否需要 firmware 回傳 response event。大多數 set 命令不需要，query 命令需要。

`pfCmdDoneHandler` / `pfCmdTimeoutHandler`：function pointer，命令完成或超時時的 callback。`PFN_` 前綴是 MTK 慣例，表示 pointer to function（typedef 的函式指標型別）。

### 3.5 `PARAM_CONNECT_T`（connect 請求的參數容器）

```c
typedef struct _PARAM_CONNECT_T {
    PUINT_8     pucSsid;       /* pointer to SSID bytes */
    UINT_32     u4SsidLen;
    PUINT_8     pucBssid;      /* pointer to 6-byte BSSID, or NULL */
    UINT_32     u4CenterFreq;  /* MHz, 0 = any */
} PARAM_CONNECT_T;
```

`PARAM_` 前綴從 NDIS 移植而來，表示這是在 OID 介面中傳遞的參數結構。這個結構是 `mtk_cfg80211_connect()` 打包連線目標資訊後，透過 `kalIoctl()` 傳給 `wlanoidSetConnect()` 的資料容器。

`pucSsid`（`PUINT_8`，即 `UINT_8 *`）：指向 SSID 字串的 pointer。`P` 前綴表示 pointer，`uc` 表示 unsigned char，合起來是 pointer to unsigned char array。

`pucBssid`：指向 BSSID（6 bytes MAC 位址）的 pointer。可為 NULL，表示不指定 AP。

`u4CenterFreq`：目標頻率，單位 MHz（注意：`CONNECTION_SETTINGS_T` 儲存時以 `u4FreqInKHz` 欄位轉為 KHz）。

### 3.6 `PARAM_SCAN_REQUEST_ADV_T`（scan 請求的參數容器）

```c
typedef struct _PARAM_SCAN_REQUEST_ADV_T {
    UINT_32           u4SsidNum;
    PARAM_SSID_T      rSsid[CFG_SCAN_SSID_MAX_NUM];
    PUINT_8           pucIE;
    UINT_32           u4IELength;
    UINT_8            ucChannelListNum;
    RF_CHANNEL_INFO_T arChnlInfoList[MAXIMUM_OPERATION_CHANNEL_LIST];
} PARAM_SCAN_REQUEST_ADV_T;
```

`ADV` 後綴表示 Advanced，相對於只有 BSSID 的舊版 `PARAM_SCAN_REQUEST_T`，這個版本支援多 SSID 和指定頻道清單。由 `mtk_cfg80211_scan()` 填入後傳給 `wlanoidSetBssidListScanAdv()`。

`u4SsidNum` / `rSsid[]`：要掃描的 SSID 清單。`PARAM_SSID_T` 定義為 `struct { UINT_8 aucSsid[32]; UINT_32 u4SsidLen; }`。空 SSID（wildcard scan）代表掃描所有 AP。

`pucIE` / `u4IELength`：Probe Request 要附帶的 Information Element，例如 WPS IE（Wi-Fi Protected Setup）、vendor-specific IE 等。IE 是 802.11 管理幀中的 TLV（Type-Length-Value）資料單元，用來在管理幀中攜帶各種擴充資訊。

`ucChannelListNum` / `arChnlInfoList[]`：指定掃描的頻道清單。0 表示掃描所有頻道。`RF_CHANNEL_INFO_T` 包含頻段（2.4G/5G）、頻道號碼、頻寬（20/40/80 MHz）等。

---

## 4. Thread 模型與事件旗標

MT7668 driver 使用三個 kernel thread 分工處理不同類型的工作。Kernel thread 是在 kernel space 執行的 thread，透過 `kthread_create()` 建立，不對應任何 user space process。

### 4.1 事件旗標（`gl_os.h`）

`GLUE_INFO_T.ulFlag` 是一個 `unsigned long`，每個 bit 代表一種待處理事件。以 `set_bit()`、`test_bit()`、`clear_bit()` 操作，這三個函式在 SMP 環境下是 atomic 的（使用 CPU 的 lock prefix 指令），不需要額外加鎖。

| 旗標名稱 | Bit | 設置位置 | 說明 |
|---|---|---|---|
| `GLUE_FLAG_HALT_BIT` | 0 | `glRemoveStation()` | driver 正在卸載，thread 收到後退出迴圈 |
| `GLUE_FLAG_OID_BIT` | 2 | `kalIoctlTimeout()` | 有 OID 請求待處理 |
| `GLUE_FLAG_TXREQ_BIT` | 4 | TX 路徑入列 | 有 TX 請求待送出 |
| `GLUE_FLAG_RX_BIT` | 10 | HIF 中斷 handler | 有 RX 資料待處理 |

### 4.2 `main_thread()`（`gl_kal.c:2834`）

```c
static int main_thread(void *data)
{
    while (TRUE) {
        wait_event_interruptible(prGlueInfo->waitq,
            prGlueInfo->ulFlag != 0);

        if (test_and_clear_bit(GLUE_FLAG_HALT_BIT, &prGlueInfo->ulFlag))
            break;

        if (test_and_clear_bit(GLUE_FLAG_OID_BIT, &prGlueInfo->ulFlag)) {
            wlanSetInformation() 或 wlanQueryInformation();
            complete(&prGlueInfo->rPendComp);
        }
        if (test_and_clear_bit(GLUE_FLAG_TXREQ_BIT, ...))
            kalProcessTxReq();
        if (test_and_clear_bit(GLUE_FLAG_RX_BIT, ...))
            nicRxProcessRFBs();
    }
}
```

`main_thread` 是 driver 的主控執行緒，序列處理所有控制面操作（OID）。序列執行保證 `ADAPTER_T` 的狀態不需要複雜的 locking，因為只有這一個 thread 在修改核心狀態。

`wait_event_interruptible()` 是 Linux kernel 的 sleep-and-wait 巨集，讓 thread 進入可中斷的睡眠狀態（`TASK_INTERRUPTIBLE`）直到條件成立或收到 signal。這比 busy-loop 節省 CPU，比 `wait_event()`（不可中斷睡眠 `TASK_UNINTERRUPTIBLE`）更能響應 signal（例如 `kthread_stop()` 發送的 `SIGKILL`）。

`test_and_clear_bit()` 是 atomic 操作，測試 bit 是否為 1 並同時清除它，回傳舊值。這避免了「test 後 clear 前又被別的 thread set」的 TOCTOU 問題。

### 4.3 `hif_thread()`（`gl_kal.c:2639`）

處理 SDIO 的底層 I/O 操作。SDIO 傳輸有自己的排程需求（DMA、interrupt acknowledge 等），從 `main_thread` 分離避免阻塞控制面操作。

### 4.4 `rx_thread()`（`gl_kal.c:2733`）

將已接收的封包從 driver 的 RX buffer 投遞到 Linux 網路棧（呼叫 `netif_rx()` 或等效路徑）。此操作可能因為 socket buffer 滿而短暫阻塞，獨立 thread 避免阻塞 `main_thread`。

---

## 5. OID 機制（kalIoctl → main_thread）

### 5.1 `kalIoctlTimeout()`（`gl_kal.c:2048`）

```c
WLAN_STATUS kalIoctlTimeout(
    P_GLUE_INFO_T           prGlueInfo,
    PFN_OID_HANDLER_FUNC    pfnOidHandler,  /* OID handler，如 wlanoidSetConnect */
    PVOID                   pvInfoBuf,      /* 參數 buffer，如 PARAM_CONNECT_T * */
    UINT_32                 u4InfoBufLen,   /* sizeof(PARAM_CONNECT_T) 等 */
    BOOLEAN                 fgRead,         /* TRUE=query, FALSE=set */
    BOOLEAN                 fgWaitResp,     /* 是否等 firmware response */
    BOOLEAN                 fgCmd,          /* 是否走 CMD path（非 OID） */
    PUINT_32                pu4QryInfoLen,  /* 實際資料長度（output，query 用） */
    UINT_32                 u4Timeout       /* ms */
);
```

**執行流程：**

1. 取得 `g_halt_sem` semaphore（避免 driver 卸載時仍有 OID 在執行）
2. 取得 `ioctl_sem`（序列化多個 `kalIoctl` 呼叫，同時只允許一個 OID 排隊）
3. 將所有參數填入 `prGlueInfo->OidEntry`
4. `set_bit(GLUE_FLAG_OID_BIT, &prGlueInfo->ulFlag)`
5. `wake_up_interruptible(&prGlueInfo->waitq)` 喚醒 `main_thread`
6. `wait_for_completion_timeout(&prGlueInfo->rPendComp, u4Timeout)` — 呼叫端睡眠等待
7. `main_thread` 執行完畢後呼叫 `complete(&prGlueInfo->rPendComp)`，喚醒呼叫端
8. 釋放 semaphore，回傳 OID 執行結果

`kalIoctl()` 是 `kalIoctlTimeout()` 的 wrapper，使用預設 timeout（`WLAN_OID_TIMEOUT_THRESHOLD`，通常 4 秒）。

**為何需要這個機制：** cfg80211 的 `.scan`、`.connect` 等 op 是從 kernel thread context（例如 `cfg80211_wq` workqueue）呼叫的，可以 sleep。但這些 op 不能直接修改 `ADAPTER_T`，因為 `main_thread` 是唯一被允許操作核心狀態的 thread。`kalIoctl` 提供了一個安全的跨 thread 同步橋樑。

**connect 卡住的診斷：** 若 `connect` 或 `scan` 沒有回應，`complete()` 未被呼叫是最常見原因，代表 `main_thread` 沒有執行到 `complete()`，可能是 OID handler 本身卡住（例如等 firmware response 超時），或 `main_thread` 已退出。

### 5.2 `wlanSetInformation()` / `wlanQueryInformation()`（`wlan_lib.c`）

`main_thread` 收到 `OID_BIT` 後，根據 `OidEntry.fgRead`：
- `fgRead = FALSE`（set）：呼叫 `wlanSetInformation()`
- `fgRead = TRUE`（query）：呼叫 `wlanQueryInformation()`

兩者都會最終呼叫 `OidEntry.pfnOidHandler`（如 `wlanoidSetConnect`）。這一層做了 ACPI 狀態檢查：若裝置在 D3 power state（已斷電），大多數 OID 會被拒絕並回傳 `WLAN_STATUS_ADAPTER_NOT_READY`。

---

## 6. Driver Bring-up 流程

### 6.1 模組入口（`gl_init.c`）

```c
module_init(initWlan);   /* 行 2893 */
module_exit(exitWlan);

static INT_32 initWlan(void)  /* 行 2800 */
{
    wlanDebugInit();
    glRegisterBus(wlanProbe, wlanRemove);  /* 行 2836 */
}
```

`module_init()` 是 Linux kernel 巨集，將函式標記為模組載入時執行的 init 函式。當 `insmod` 或 kernel 的 built-in init 載入 driver 時，kernel 呼叫這個函式。`glRegisterBus()` 將 `wlanProbe` 和 `wlanRemove` 儲存為 SDIO 層的 callback，但尚未真正呼叫它們，等 SDIO 匹配到 MT7668 device ID 才觸發。

### 6.2 SDIO 偵測與 probe（`hif/sdio/sdio.c`）

```c
/* sdio.c:700 */
INT_32 glRegisterBus(probe_card pfProbe, remove_card pfRemove)
{
    pfWlanProbe = pfProbe;
    pfWlanRemove = pfRemove;
    return sdio_register_driver(&mtk_sdio_driver);
}

/* SDIO 子系統匹配到 MT7668 device ID 後呼叫 */
static int mtk_sdio_probe(struct sdio_func *func,
                          const struct sdio_device_id *id)  /* 行 427 */
{
    sdio_enable_func(func);
    sdio_claim_host(func);
    /* ... SDIO 初始化 ... */
    pfWlanProbe((PVOID)func);
}
```

`struct sdio_func` 是 Linux SDIO 子系統的裝置描述符，包含 SDIO function number、vendor/device ID、block size 等。`sdio_enable_func()` 啟用 SDIO function（MT7668 的 Wi-Fi function）。`sdio_claim_host()` 取得 SDIO bus 的獨占存取權，SDIO bus 不支援並行存取，所有操作必須在 claim host 後進行。

### 6.3 `wlanProbe()`（`gl_init.c:2203`）

```c
static INT_32 wlanProbe(PVOID pvData)
/* pvData 是 struct sdio_func *，透過 void * 傳遞 */
```

**執行順序：**

```
1. glBusInit(pvData)
   — 設定 SDIO block size（通常 512 bytes，影響 DMA 效率）
   — 設定 SDIO bus clock 頻率
   — 啟用 SDIO interrupt（MT7668 透過 SDIO DAT1 引腳觸發中斷）

2. wlanCreateWirelessDevice()  [行 1458]
   — wiphy_new(&mtk_wlan_ops, sizeof(GLUE_INFO_T))
     wiphy（wireless physical device）是 cfg80211 的硬體抽象，
     一個 wiphy 對應一塊 Wi-Fi 硬體。
     GLUE_INFO_T 嵌在 wiphy 的 private data 區（wiphy_priv(wiphy)）
   — 填充 wiphy 屬性：支援的 band（2.4G/5G）、
     cipher suite、max scan SSIDs、vendor commands 等
   — wiphy_register(wiphy)  [行 1523]
     向 cfg80211 框架登記此 wiphy，之後 nl80211 才能操作它

3. wlanNetCreate()
   — alloc_netdev_mq(sizeof(NETDEV_PRIVATE_GLUE_INFO),
                     "wlan%d", NET_NAME_ENUM, ether_setup, 1)
     建立 net_device，名稱 wlan0，Ethernet 類型，1 個 TX queue
   — prNetDevOps = &wlan_netdev_ops
     綁定 netdev_ops（.ndo_start_xmit、.ndo_open、.ndo_stop 等）
   — prNetDev->wireless_handlers = &wext_handler_def
     綁定 WEXT handler table（向後相容 iwconfig）
   — prNetDev->ieee80211_ptr = prWdev
     將 net_device 與 wireless_dev（cfg80211 per-interface 結構）關聯

4. wlanAdapterStart()  [wlan_lib.c:318]
   — 下載 firmware 和 patch（見第 7 節）
   — 初始化 NIC TX/RX 路徑
   — 啟動 firmware（解除 N9/CR4 reset）

5. kthread_run(main_thread, ...)
   kthread_run(hif_thread, ...)
   kthread_run(rx_thread, ...)

6. wlanoidQueryCurrentAddr()
   向 firmware 查詢 MAC 位址，寫入 net_device.dev_addr

7. wlanNetRegister()
   — register_netdev(prNetDev)
     正式登記 wlan0，/sys/class/net/wlan0 出現
   — u4ReadyFlag = 1

8. procCreateFsEntry()  [行 1298]
   建立 /proc/net/wlan/* debug 節點
```

---

## 7. Firmware 下載流程

### 7.1 入口：`wlanAdapterStart()`（`wlan_lib.c:318`）

```
wlanAdapterStart()
  → 取得 power ownership（與 BT driver 協調，MT7668 是 combo 晶片）
  → 硬體 reset
  → 讀取 chip ID / ECO 版本（決定 firmware 版本）
  → wlanDownloadPatch()    [行 3653]  — 下載 ROM patch
  → wlanDownloadFW()       [行 3553]  — 下載主 firmware
  → 啟動 firmware（解除 N9/CR4 reset）
  → 等待 firmware ready 訊號
  → 初始化 TX/RX 路徑、command/event 機制
```

MT7668 有兩顆 MCU：N9（主要 Wi-Fi MCU，執行主 firmware）和 CR4（輔助 MCU，處理 offload 功能如 TCP ACK offload）。ROM patch 修正晶片 ROM 中的 bug，主 firmware 包含完整的 Wi-Fi 802.11 協定棧實作。

### 7.2 Firmware 檔名決策

**候選清單生成：** `chips/mt7668.c:mt7668ConstructFirmwarePrio()(行 83)` 依 chip ID 和 ECO 版本（晶片製程改版版本，從暫存器讀取）生成候選檔名清單：

```
WIFI_RAM_CODE_MT7668_E1
WIFI_RAM_CODE_MT7668_E1.bin
WIFI_RAM_CODE_MT7668
WIFI_RAM_CODE_MT7668.bin
```

Patch 檔名格式：`mt%x_patch_e%x_hdr.bin`，例如 `mt7668_patch_e1_hdr.bin`。

**搜尋路徑：** `gl_kal.c:kalFirmwareOpen()(行 175)` 對每個候選檔名呼叫 `request_firmware()`。`request_firmware()` 是 Linux kernel 的 firmware 載入 API，依序搜尋（`gl_kal.c:187` 註解明示）：

- Android：`/etc/firmware`、`/vendor/firmware`、`/firmware/image`
- Linux：`/lib/firmware`、`/lib/firmware/update`

**下載過程：** `wlanImageSectionDownload()(行 3038)` 將 firmware image 分段透過 SDIO 寫入 MT7668 的 RAM，每段下載後呼叫 `wlanImageSectionDownloadStatus()(行 3216)` 確認。分段下載是因為 SDIO DMA buffer 有大小限制，不能一次傳輸整個 firmware image。

---

## 8. TX / RX 資料路徑

### 8.1 TX 路徑

```
應用層送封包（write socket）
  → kernel 網路棧（TCP/IP 封裝成 sk_buff）
  → net_device.ndo_start_xmit 回呼
  → wlanHardStartXmit()  [gl_init.c:984]
  → kalHardStartXmit()   [gl_kal.c:1413]
      — 依 sk_buff 的 QoS 標記分類到 WMM Access Category
        （BE/BK/VI/VO，對應 802.11e EDCA）
      — 入列，set_bit(GLUE_FLAG_TXREQ_BIT)，wake_up
  → main_thread 收到 TXREQ_BIT
  → kalProcessTxReq()    [gl_kal.c:2513]
  → wlanProcessTxFrame() [wlan_lib.c:4256]
      — 標記封包屬性（加密、QoS tid 等）
  → nicTxEnqueueMsdu()   [nic_tx.c:2495]
      — 進入 QM（Queue Management）
      — 送往 HIF layer → SDIO DMA 送出
```

`sk_buff`（socket buffer）是 Linux 網路棧中封包的核心資料結構，包含封包資料（headroom + data + tailroom 三個區段）和描述 metadata（protocol、interface、timestamp 等）。

### 8.2 RX 路徑

```
SDIO 中斷觸發（MT7668 收到 802.11 frame，解密後 DMA 到 host RAM）
  → hif_thread 處理 SDIO interrupt
  → nicRxProcessRFBs()   [nic_rx.c:2728]
      — RFB（RX Frame Buffer）是 MTK 的 RX 封包描述符
      — 解析 RX descriptor（RSSI、channel、加密狀態等）
      — data frame → 往上送；management frame → 交給 mgmt 層
  → rx_thread
  → kalRxIndicateOnePkt()
      — 透過 netif_rx() 送入 Linux 網路棧
```

---

## 9. User Space 互動介面

### 9.1 cfg80211 ops（主路徑）

`gl_init.c:337` 定義 `mtk_wlan_ops`（`struct cfg80211_ops`），綁定 cfg80211 op 到 MTK 函式：

| cfg80211 op | MTK 函式 | 檔案:行號 |
|---|---|---|
| `.scan` | `mtk_cfg80211_scan` | `gl_cfg80211.c:815` |
| `.connect` | `mtk_cfg80211_connect` | `gl_cfg80211.c:900` |
| `.disconnect` | `mtk_cfg80211_disconnect` | `gl_cfg80211.c:1295` |
| `.add_key` | `mtk_cfg80211_add_key` | `gl_cfg80211.c` |
| `.del_key` | `mtk_cfg80211_del_key` | `gl_cfg80211.c` |
| `.get_station` | `mtk_cfg80211_get_station` | `gl_cfg80211.c` |

### 9.2 WEXT / private ioctl（舊路徑）

`gl_wext_priv.c:priv_driver_cmds()(行 10781)` 解析字串命令，透過同樣的 `kalIoctl` 機制執行。常見命令：`SETFWPATH`（覆寫 firmware 路徑）、`COUNTRY`（設定 regulatory domain）、`SETROAMMODE`（設定漫遊模式）、`SET_FWLOG`（設定 firmware log 層級）。

### 9.3 /proc debug 節點

`gl_proc.c:procCreateFsEntry()(行 1298)` 建立 `/proc/net/wlan/` 下的節點：

| 節點 | 用途 |
|---|---|
| `driver` | 字串命令（轉給 `priv_driver_cmds`） |
| `dbg_level` | 設定 debug log 層級 |
| `mcr` | 讀寫 MT7668 暫存器（MCR = MAC Control Register） |
| `cfg` | 設定參數 |
| `efuse_dump` | 讀取 eFuse（一次性可程式記憶體，儲存 MAC 位址、校正值等） |
| `csi_data` | Channel State Information（用於 beamforming 或 debug） |

---

## 10. Amazon 客製功能

### 10.1 Board ID（`drivers/misc/amazon/board_id.c`）

```c
static int amazon_board_id_probe(struct platform_device *pdev)
```

從 DTS `amazon,board-id` 節點讀取三個 GPIO 腳位（`id0`、`id1`、`id2`），計算 `board_id = id0 | (id1 << 1) | (id2 << 2)`（3-bit，最多 8 種板型）。匯出 `get_board_id()` 供其他 driver 呼叫（例如 `lab126_ts_bts.c` 依板版本調整 thermistor 參數）。在 `/proc/board_id` 提供 user space 存取。

DTS 節點（`mantis.dts:85`）：
```dts
board_id {
    compatible = "amazon,board-id";
    id-gpios = <&pio 14 0>, <&pio 15 0>, <&pio 16 0>;
};
```

### 10.2 IDME（`drivers/misc/idme.c`）

IDME（IDentification MEdia）是 Amazon 的工廠燒錄資訊機制。Bootloader 在開機時將工廠資料（序號、MAC 位址、校正值、bootmode 等）透過 Device Tree Blob fixup 動態注入 `/idme` DT 節點。`idme.c` 在 `fs_initcall(行 396)` 階段讀取這些節點並建立 `/proc/idme/*`。

`/idme` 節點不在 `mantis.dts` 靜態宣告，由 bootloader 動態注入。若 bootloader 未注入，`idme_init()` 找不到節點，所有 `idme_get_*()` 呼叫回傳預設值或失敗。

匯出函式：

| 函式 | 說明 |
|---|---|
| `idme_get_board_type()` | 板型（例如區分 EVT/DVT/PVT 工程階段） |
| `idme_get_board_rev()` | 板版本號 |
| `idme_get_bootmode()` | 開機模式（正常/工廠/recovery 等） |
| `idme_get_battery_info()` | 電池資訊（若有） |
| `idme_get_dev_flags_value()` | 裝置功能旗標 |

Wi-Fi MAC 位址：`gl_kal.c` 中 `CONFIG_IDME` 條件編譯，從 `/proc/idme/mac_addr` 讀取 MAC 位址並覆寫 firmware 預設值，確保每台裝置有唯一的 MAC。

### 10.3 Thermal Pipeline

```
thermistor（熱敏電阻，NTC 型，阻值隨溫度升高而降低）
  ↓
lab126_ts_bts.c
  — 讀取 MT8695 AUXADC（Auxiliary ADC Controller）的量測值
  — 依 board_id / board_rev 從查找表選對應的 pull-up 電阻參數
  — 將 ADC 值透過 Steinhart-Hart 公式轉換為攝氏溫度
  — 呼叫 virtual_sensor_dev_register() 向虛擬感測器框架登記

  ↓
virtual_sensor_thermal.c
  — 從 DTS vs_components 讀取每個感測器的 offset / alpha / weight
  — virtual_sensor_get_temp() 計算：
      filtered_temp = alpha * raw + (1 - alpha) * prev_filtered （指數低通濾波）
      contribution = (filtered_temp + offset) * weight
      virtual_temp = sum(contribution) / sum(weight)
  — 以 virtual_temp 作為 thermal zone 的溫度讀值

  ↓
mantis_thermal_zones.dtsi
  — board_thermal zone：trip points 57°C / 59°C / 61°C
  — soc_thermal zone：trip points 90°C / 100°C / 110°C
  — cooling-map 將每個 trip 對應到特定 cooling device 的 state

  ↓
amzn_thermal_cooling.c
  — 依 DTS cooling_devices 清單，呼叫對應的 cooling_register 函式
  — cpufreq cooling：透過 cpufreq driver 降低 CPU 最高允許頻率
  — gpufreq cooling：透過 GPU power domain 限制 GPU 頻率
  — wifi cooling（wifi_cooling.c）：限制 Wi-Fi TX 功率（降低晶片功耗）
```

**Probe defer 機制：** `virtual_sensor_thermal_probe()` 和 `amzn_thermal_probe()` 都需要 cpufreq 頻率表就緒才能完成初始化。若 `mt8695_cpufreq_probe()` 尚未執行完，這些函式回傳 `-EPROBE_DEFER`，kernel 的 deferred probe 機制會在稍後重新呼叫 probe。這是 Linux driver 處理初始化順序依賴的標準做法。

---

## 11. 開機時序：initcall 層級

Linux 4.4 的 initcall 機制（`include/linux/init.h`）將 driver init 函式依層級排序執行。各層級以 linker script 分組到不同的 ELF section（`.init.data` 的各子 section），`do_initcalls()`（`init/main.c:861`）依序呼叫。

| 層級 | 巨集 | 說明 |
|---|---|---|
| `early` | `early_initcall` | 極早期，console 等 |
| `core` | `core_initcall` | core 子系統（workqueue、RCU 等） |
| `subsys` | `subsys_initcall` | bus 子系統（PCI、SDIO 等） |
| `fs` | `fs_initcall` | 檔案系統、早期 driver |
| `device` | `device_initcall` / `module_init` | 大多數 driver |
| `late` | `late_initcall` | 依賴其他 driver 完成的初始化 |

Mantis 上的實際 driver probe 順序：

| 層級 | 函式 | 檔案 | DTS 對應 |
|---|---|---|---|
| `fs_initcall` | `idme_init()` | `drivers/misc/idme.c:396` | 由 bootloader 注入 |
| `device_initcall` | `mt8695_cpufreq_driver_init()` | `mt8695-cpufreq.c:750` | `mt8695.dtsi:30` |
| `device_initcall` | `board_id_driver` probe | `board_id.c:135` | `mantis.dts:85` |
| `late_initcall` | `virtual_sensor_thermal_init()` | `virtual_sensor_thermal.c:408` | `mantis.dts:114` |
| `late_initcall` | `ntc_bts_init()` / `gadc_thermal_init()` | `lab126_ts_bts.c:637,793` | `mantis.dts:94-113` |
| `late_initcall` | `amzn_thermal_platdrv_init()` | `amzn_thermal_cooling.c:279` | `mantis.dts:135` |

Wi-Fi driver（`initWlan`）使用 `module_init()`（等同 `device_initcall`），但實際的 SDIO probe（`mtk_sdio_probe`）由 SDIO 子系統在掃描 bus 時觸發，時間點在 `module_init` 之後。

---

## 12. Mantis `defconfig` 重點選項

```
# Build 輸出（附 DTB 的 kernel 映像）
CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE=y
CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE_NAMES="mediatek/mantis"

# Thermal pipeline
CONFIG_THERMAL_VIRTUAL_SENSOR=y
CONFIG_BTS_THERMISTOR=y
CONFIG_AMZN_THERMAL_COOLING=y

# Amazon 平台資訊
CONFIG_IDME=y
CONFIG_USB_CH_TYPE_SYSFS=y
CONFIG_BOARD_ID=y

# 輸入裝置
CONFIG_HID_FTV_BLEREMOTE=y
CONFIG_HID_AMAZON_ASPEN=y

# Android / 安全
CONFIG_ANDROID_BINDER_IPC=y
CONFIG_ASHMEM=y
CONFIG_ION=y
CONFIG_DM_VERITY=y
CONFIG_SECURITY_SELINUX=y
```

`mantis_debug_defconfig` 相較量產版多開：`IKCONFIG_PROC`（允許從 `/proc/config.gz` 讀取 kernel config）、`SLUB_DEBUG`（記憶體分配 debug）、`FUNCTION_PROFILER`、`DMA_API_DEBUG` 等。`bootargs` 含 `androidboot.selinux=permissive` 與 `initcall_debug=1`，後者讓 kernel 在 dmesg 印出每個 initcall 的執行時間。

---

## 13. 關鍵函式索引

| 函式 | 檔案 | 行號 |
|---|---|---|
| `initWlan` | `gl_init.c` | 2800 |
| `wlanProbe` | `gl_init.c` | 2203 |
| `wlanCreateWirelessDevice` | `gl_init.c` | 1458 |
| `wlanAdapterStart` | `wlan_lib.c` | 318 |
| `kalFirmwareOpen` | `gl_kal.c` | 175 |
| `kalFirmwareImageMapping` | `gl_kal.c` | 344 |
| `mt7668ConstructFirmwarePrio` | `chips/mt7668.c` | 83 |
| `wlanDownloadFW` | `wlan_lib.c` | 3553 |
| `wlanDownloadPatch` | `wlan_lib.c` | 3653 |
| `wlanImageSectionDownload` | `wlan_lib.c` | 3038 |
| `wlanImageSectionDownloadStatus` | `wlan_lib.c` | 3216 |
| `main_thread` | `gl_kal.c` | 2834 |
| `kalIoctlTimeout` | `gl_kal.c` | 2048 |
| `kalProcessTxReq` | `gl_kal.c` | 2513 |
| `wlanHardStartXmit` | `gl_init.c` | 984 |
| `kalHardStartXmit` | `gl_kal.c` | 1413 |
| `nicRxProcessRFBs` | `nic_rx.c` | 2728 |
| `nicTxEnqueueMsdu` | `nic_tx.c` | 2495 |
| `mtk_cfg80211_scan` | `gl_cfg80211.c` | 815 |
| `mtk_cfg80211_connect` | `gl_cfg80211.c` | 900 |
| `mtk_cfg80211_disconnect` | `gl_cfg80211.c` | 1295 |
| `wlanoidSetConnect` | `wlan_oid.c` | 1042 |
| `kalIndicateStatusAndComplete` | `gl_kal.c` | 947 |
| `procCreateFsEntry` | `gl_proc.c` | 1298 |
