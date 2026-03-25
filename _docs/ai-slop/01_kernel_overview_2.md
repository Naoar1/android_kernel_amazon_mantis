# android_kernel_amazon_mantis 專案 Code Flow 與檔案作用報告（詳細）

> Repo: `https://github.com/chaosmaster/android_kernel_amazon_mantis`  
> 位置：`/home/maroiahsu1999/xlclaw-workdir/kernel-mantis-analysis/android_kernel_amazon_mantis`  
> 版本：`8730cd63`（單一匯入 commit）  
> Kernel 基底：`Linux 4.4.120`（`Makefile`: VERSION=4, PATCHLEVEL=4, SUBLEVEL=120）

---

## 1) 我如何「逐一檢視」

這份 repo 是完整 kernel source tree（超大）：

- 總檔案數：**55,558**
- `drivers/`：22,333 檔
- `arch/`：16,338 檔

因此「逐檔逐行人工讀完」不具可行性；我採用兩層檢視法：

1. **全樹盤點**（所有資料夾/檔案結構、數量、層級分布）
2. **深度 code flow tracing**（Mantis 裝置實際會走到的 build/runtime 路徑）

並輸出兩份可機器驗證的清單：

- `REPORT_top_level_counts.tsv`（top-level 檔案數）
- `REPORT_level2_counts.tsv`（二層路徑檔案數）

---

## 2) 專案本質與定位

這份專案不是「只含 mantis 的小專案」，而是 **完整 Linux/Android kernel 樹 + Mediatek/Amazon 裝置客製化**。

### 2.1 可辨識的裝置/平台訊息

- SoC 族：`mediatek,mt8695`
- 裝置樹：`arch/arm64/boot/dts/mediatek/mantis.dts`
- defconfig：`arch/arm64/configs/mantis_defconfig`
- commit 訊息顯示來源：`Import from FireTVStick4K-6.2.6.5-20190808.tar.bz2`

結論：這是 **Fire TV Stick 4K (mantis) 的 vendor kernel drop** 型態。

---

## 3) 全專案檔案作用（按目錄）

### 3.1 Top-level 角色

- `arch/`：架構相關（ARM/ARM64/x86/...），啟動碼、MMU、IRQ、平台初始化
- `drivers/`：所有驅動（SoC、GPU、媒體、網路、儲存、輸入、熱管理）
- `include/`：核心與驅動共用 header
- `kernel/`：排程、同步、workqueue、time、sysctl 核心邏輯
- `mm/`：記憶體管理（頁框、slab、vmalloc、回收）
- `fs/`：VFS 與檔案系統
- `net/`：網路協定棧
- `security/`：SELinux/LSM
- `sound/`：ALSA 與音訊子系統
- `Documentation/`：上游文件
- `scripts/`：build/tool scripts（Kconfig、模組、打包工具）
- `android/configs/`：Android kernel config fragments

### 3.2 數量概覽（節錄）

- `drivers/gpu`、`drivers/net`、`drivers/media` 佔比極高
- `arch/arm`、`arch/arm64` 共存（同樹支援多平台）
- `drivers/misc/mediatek` 是 MTK 客製核心區

---

## 4) Build Flow（從 config 到輸出映像）

## 4.1 入口配置

- `arch/arm64/configs/mantis_defconfig`
  - 啟用 `CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE=y`
  - 指定 `CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE_NAMES="mediatek/mantis"`

## 4.2 產物組裝路徑

- `arch/arm64/Kconfig`
  - 選擇 `Image.gz-dtb` 或 `Image-dtb`
- `arch/arm64/Makefile`
  - 若 APPENDED_DTB_IMAGE=y，`KBUILD_IMAGE` 走 appended image
- `arch/arm64/boot/Makefile`
  - `Image.gz-dtb := Image.gz + DTB_OBJS`
- `arch/arm64/boot/dts/Makefile`
  - 以 `CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE_NAMES` 決定 DTB list

**結論**：Mantis 預設輸出是 **包含 DTB 的 kernel 映像**，而非純 `Image.gz`。

---

## 5) 開機 Runtime Code Flow（核心主線）

## 5.1 CPU 進入點

- `arch/arm64/kernel/head.S`
  - `_head -> stext`
  - 要求 `x0` 指向 FDT
  - 建頁表、切換到 EL1、跳入 C 世界

## 5.2 架構初始化

- `init/main.c:start_kernel()`
  - `setup_arch()`、記憶體初始化、IRQ/timer、scheduler、console
- `arch/arm64/kernel/setup.c:setup_arch()`
  - `setup_machine_fdt(__fdt_pointer)` + `early_init_dt_scan`
  - `unflatten_device_tree()`
  - `arch_initcall_sync(arm64_device_init)` -> `of_platform_populate(...)`

## 5.3 Driver initcall 階段

- `init/main.c:do_initcalls()` 依 level 呼叫
- `device_initcall / late_initcall / fs_initcall` 的 driver 在此 probe

---

## 6) Mantis 裝置專屬 Flow（重點）

## 6.1 DTS 分層

- `arch/arm64/boot/dts/mediatek/mantis.dts`
  - `#include "mt8695.dtsi"`
  - `#include "mantis_thermal_zones.dtsi"`

也就是：

- `mt8695.dtsi` = SoC 通用硬體定義
- `mantis.dts` = 板級覆寫/開啟節點
- `mantis_thermal_zones.dtsi` = 熱策略與 cooling map

## 6.2 板級啟用節點（mantis.dts）

已明確 `status = "okay"` 的重要節點：

- `&bt`, `&wifi`
- `&i2c0..3`
- `&mmc0`（eMMC）, `&mmc2`（SDIO，連 Wi-Fi）
- `&uart0`, `&uart1`
- `&ssusb`（OTG）

並新增 Amazon 節點：

- `board_id { compatible = "amazon,board-id"; ... }`
- `virtual-sensor { compatible = "amazon,virtual_sensor"; ... }`
- `amzn_thermal@0 { compatible = "amazon,amzn-thermal"; ... }`

## 6.3 Thermal 策略 flow（最關鍵）

### A) 溫度來源層

- `drivers/misc/mediatek/thermal/mt8695/lab126_ts_bts.c`
  - 讀 thermistor ADC
  - 依 board revision 調整 pull-up 參數（呼叫 `get_board_id()` 或 `idme_get_board_rev()`）
  - `virtual_sensor_dev_register(...)` 註冊至虛擬熱感測框架

### B) 虛擬感測器融合層

- `drivers/thermal/virtual_sensor_thermal.c`
  - probe `compatible="amazon,virtual_sensor"`
  - 讀 `vs_components` 的 `offset/alpha/weight`
  - `virtual_sensor_get_temp()` 對多 sensor 做平滑 + 加權融合
  - 把融合結果當 thermal sensor 提供給 thermal zone

### C) Cooling 裝置註冊層

- `drivers/misc/mediatek/thermal/mt8695/amzn_thermal_cooling.c`
  - probe `compatible="amazon,amzn-thermal"`
  - 按 DTS `cooling_devices` 註冊 cpufreq/cpucore/gpufreq/wifi cooling device

### D) 熱區策略層

- `arch/arm64/boot/dts/mediatek/mantis_thermal_zones.dtsi`
  - `board_thermal` + `soc_thermal`
  - 設 trip points（57~61°C / 90~110°C）
  - cooling-map 對應 CPU/GPU/Wi-Fi 節流級別

=> 這是一條完整的 Amazon 客製 thermal pipeline：
`thermistor -> virtual sensor -> thermal zone -> cooling map -> throttling`

---

## 7) IDME / Board ID / Amazon 系統資訊 Flow

## 7.1 Board ID

- `drivers/misc/amazon/board_id.c`
  - 透過 `id0/id1/id2` GPIO 組出 board_id
  - 匯出 `get_board_id()`
  - 提供 `/proc/board_id`

## 7.2 IDME

- `drivers/misc/idme.c`
  - 讀 DT path `/idme/*`（如 board_id, productid2, bootmode, alscal, dev_flags）
  - 生成 `/proc/idme/*`
  - 匯出 helper：
    - `idme_get_board_type()`
    - `idme_get_board_rev()`
    - `idme_get_bootmode()`
    - `idme_get_battery_info()`
    - `idme_get_dev_flags_value()`

### 重要觀察

`/idme` 節點不在 `mantis.dts` 明確宣告，代表多半由 bootloader/FDT fixup 注入。

## 7.3 USB charge type sysfs

- `drivers/misc/amazon/usb_charge_type_sysfs.c`
  - 解析 cmdline `usb_extconn=`
  - 建立 `/sys/amazon/usb_charge_type`

---

## 8) Connectivity / Input / Media 相關 Flow

## 8.1 Wi-Fi 控制與 MAC 來源

- DTS：`wifi` node (`mediatek,mt7668_wifi_ctrl`), `mmc2` SDIO 非 removable
- 驅動：`drivers/misc/mediatek/connectivity/wlan/gen4/...`
- `gl_kal.c` 有 `CONFIG_IDME` 路徑讀 `/proc/idme/mac_addr`，覆寫預設 MAC 來源

## 8.2 BLE Remote / Keyboard 客製 HID

- `drivers/hid/hid-ftv-bleremote.c`（語音/按鍵/音訊 ring buffer）
- `drivers/hid/hid-aspen.c`（Amazon BT keyboard/trackpad 客製 key mapping）
- `drivers/hid/Kconfig` + `drivers/hid/Makefile` 有對應 `CONFIG_HID_FTV_BLEREMOTE` / `CONFIG_HID_AMAZON_ASPEN`

## 8.3 GPU / HDMI / Audio

- `mt8695.dtsi` 定義：`clark` GPU、`mfg`、`hdmitx`、`afe` audio
- `mantis.dts` 再覆寫 pinctrl / regulator / status

---

## 9) 關鍵設定（mantis_defconfig）解讀

`arch/arm64/configs/mantis_defconfig` 中 Mantis 相關重點：

- Thermal pipeline：
  - `CONFIG_THERMAL_VIRTUAL_SENSOR=y`
  - `CONFIG_BTS_THERMISTOR=y`
  - `CONFIG_AMZN_THERMAL_COOLING=y`
- Amazon 平台資訊：
  - `CONFIG_IDME=y`
  - `CONFIG_USB_CH_TYPE_SYSFS=y`
  - `CONFIG_BOARD_ID=y`
- 輸入裝置：
  - `CONFIG_HID_FTV_BLEREMOTE=y`
  - `CONFIG_HID_AMAZON_ASPEN=y`
- Android/安全：
  - `CONFIG_ANDROID_BINDER_IPC=y`
  - `CONFIG_ASHMEM=y`
  - `CONFIG_ION=y`
  - `CONFIG_DM_VERITY=y`
  - `CONFIG_SECURITY_SELINUX=y`

另可見 `mantis_debug_defconfig` 與量產版差異：

- 開 `IKCONFIG_PROC`, `SLUB_DEBUG`, `FUNCTION_PROFILER`, `DMA_API_DEBUG` 等除錯選項

---

## 10) 檔案作用對照（Mantis 直觀對照表）

| 檔案 | 作用 | 在 flow 的位置 |
|---|---|---|
| `arch/arm64/configs/mantis_defconfig` | 目標 kernel config | Build 起點 |
| `arch/arm64/Kconfig` | appended dtb image 選項 | Build 規格 |
| `arch/arm64/boot/Makefile` | 產生 `Image.gz-dtb` | Build 輸出 |
| `arch/arm64/boot/dts/mediatek/mt8695.dtsi` | SoC 基礎硬體描述 | 平台底座 |
| `arch/arm64/boot/dts/mediatek/mantis.dts` | 板級覆寫、Amazon 節點 | 板級啟用 |
| `arch/arm64/boot/dts/mediatek/mantis_thermal_zones.dtsi` | thermal policy/trip/cooling map | 熱管理策略 |
| `drivers/misc/amazon/board_id.c` | GPIO 讀板號、輸出 `/proc/board_id` | board revision 判定 |
| `drivers/misc/idme.c` | `/proc/idme/*` + 匯出 IDME helper | SKU/校正/旗標資訊 |
| `drivers/misc/amazon/usb_charge_type_sysfs.c` | `/sys/amazon/usb_charge_type` | USB 充電型態資訊 |
| `drivers/misc/mediatek/thermal/mt8695/lab126_ts_bts.c` | thermistor 讀值 + board 對應修正 | thermal sensor source |
| `drivers/thermal/virtual_sensor_thermal.c` | 融合多 sensor 溫度 | virtual sensor |
| `drivers/misc/mediatek/thermal/mt8695/amzn_thermal_cooling.c` | 註冊 CPU/GPU/Wi-Fi cooling | thermal cooling layer |
| `drivers/thermal/wifi_cooling.c` | Wi-Fi cooling state 實作 | thermal actuator |
| `drivers/misc/mediatek/base/power/mt8695/mt8695-cpufreq.c` | CPU DVFS/cpufreq driver | 性能/功耗控制 |
| `drivers/hid/hid-ftv-bleremote.c` | Fire TV BLE remote 客製 HID | 使用者輸入 |
| `drivers/hid/hid-aspen.c` | Amazon Aspen 鍵盤/觸控板 | 使用者輸入 |
| `drivers/misc/mediatek/connectivity/wlan/gen4/os/linux/gl_kal.c` | Wi-Fi glue，含 IDME MAC 讀取 | 網路 bring-up |
| `arch/arm64/kernel/head.S` | AArch64 進入點 | Boot 最前段 |
| `arch/arm64/kernel/setup.c` | FDT/平台初始化、of populate | Boot 中段 |
| `init/main.c` | `start_kernel -> do_initcalls -> init` | 全核心主流程 |

---

## 11) 重要發現 / 風險點

1. **這是 vendor drop（單 commit 匯入）**，不是維護中多 commit 歷史；追 bug 時要更依賴檔內註解與 DTS/config。
2. `mantis.dts` 的 bootargs 含 `androidboot.selinux=permissive` 與 `initcall_debug=1`，明顯偏工程/調試取向。
3. `idme.c` 假設 `/idme` 節點存在；若 bootloader 未注入，相關 feature 可能退化。
4. thermal 鏈路是多層耦合（DTS 參數 + virtual sensor + cooler），調整 trip 或權重需整體驗證。

---

## 12) 你可直接延伸的下一步（若你要我接著做）

1. 幫你畫 **Mantis 實際 driver probe 時序圖**（initcall level + probe defer）
2. 幫你做 **thermal 參數反推表**（每個 trip 對應 CPU/GPU/Wi-Fi 具體 state）
3. 幫你做 **最小可編譯路徑**（只建 mantis 目標）與 build 指令腳本
4. 幫你比對 `mantis_defconfig` vs `mantis_debug_defconfig` 的可執行風險差異

---

## 附錄：盤點檔

- `REPORT_top_level_counts.tsv`
- `REPORT_level2_counts.tsv`

