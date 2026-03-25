# Android Kernel（Mantis）新手版導覽

> 適合：只有 Linux 基礎、不是核心開發者的人

---

## 先講結論：這個專案到底是什麼？

這個 repo 是 **Fire TV Stick 4K（代號 mantis）的 Linux 核心原始碼**。  
你可以把它想成「裝置的作業系統引擎 + 所有硬體驅動總包」。

它不是一般 App 專案，而是非常底層的系統程式。

---

## 你可以把整個專案想成一台車

- `arch/`：引擎啟動流程（車怎麼發動）
- `drivers/`：各零件驅動（方向盤、煞車、冷氣、螢幕）
- `include/`：零件規格書（共同標準）
- `mm/`：記憶體管理（油路）
- `fs/`：檔案系統（行李箱）
- `net/`：網路（通訊系統）

對你最重要的是：
1. `defconfig`（功能開關）
2. `dts/dtsi`（硬體接線圖）
3. `drivers`（實際執行）

---

## 這台設備開機時，核心在做什麼？（白話流程）

1. **開機程式（bootloader）把 kernel + 硬體描述丟進記憶體**
2. `head.S` 開始執行（最早期啟動）
3. `start_kernel()` 建立記憶體、排程、中斷
4. 讀取硬體樹（Device Tree）
5. 根據 Device Tree 去載入對應驅動
6. 驅動開始接管硬體（Wi-Fi、BT、USB、HDMI、Thermal...）

---

## Mantis 最重要的 4 個檔案（先看這裡就夠）

## 1) `arch/arm64/configs/mantis_defconfig`

這是「功能開關總表」。  
哪些驅動要編進核心、哪些安全功能要開，這裡決定。

## 2) `arch/arm64/boot/dts/mediatek/mt8695.dtsi`

這是 SoC 通用硬體描述（MT8695 平台共用）。  
像是 CPU、USB 控制器、GPU、I2C、HDMI 這些基礎硬體節點。

## 3) `arch/arm64/boot/dts/mediatek/mantis.dts`

這是「這台板子自己的接線圖」。  
同一顆 SoC，不同產品腳位/電源/GPIO 會不同，這裡定義。

## 4) `arch/arm64/boot/dts/mediatek/mantis_thermal_zones.dtsi`

這是散熱策略規則表：
- 幾度開始降頻
- CPU/GPU/Wi-Fi 怎麼降
- 幾度進入 critical

---

## 你最關心的 Amazon 客製功能（白話）

## A. Board ID（板號）

檔案：`drivers/misc/amazon/board_id.c`

作用：
- 讀 3 個 GPIO 腳位
- 算出這台是什麼板版本（PVT/EVT/...）
- 給其他驅動參考

為什麼重要：
- 同一裝置不同板版，可能熱參數/硬體細節不同

## B. IDME（工廠資訊）

檔案：`drivers/misc/idme.c`

作用：
- 從 `/idme/*` 讀工廠參數（板號、bootmode、flags 等）
- 透過 `/proc/idme/*` 提供給系統使用

為什麼重要：
- 很多行為靠它判斷（校正值、模式、功能開關）

## C. USB 充電型態

檔案：`drivers/misc/amazon/usb_charge_type_sysfs.c`

作用：
- 在 `/sys/amazon/usb_charge_type` 提供充電型態給 userspace

## D. 遙控器 / 鍵盤客製

- `drivers/hid/hid-ftv-bleremote.c`：藍牙遙控器
- `drivers/hid/hid-aspen.c`：Amazon 鍵盤/觸控板

---

## 散熱（Thermal）完整流程，超白話版

你可以把它想成「溫控中樞」：

1. **感測器讀溫度**  
   `lab126_ts_bts.c`（熱敏電阻）

2. **把多顆感測器混成一個更穩定的溫度**  
   `virtual_sensor_thermal.c`（offset/alpha/weight）

3. **根據策略決定要限制誰**  
   `mantis_thermal_zones.dtsi`（trip points + cooling maps）

4. **實際執行降頻/降性能**  
   `amzn_thermal_cooling.c` + `wifi_cooling.c` 等

一句話：
> 讀溫度 → 算綜合溫度 → 比對門檻 → 降 CPU/GPU/Wi-Fi

---

## 這份 repo 為什麼看起來那麼大？

因為它是 **整棵 kernel 樹**，不是只有 mantis：
- 含大量其他平台（x86、powerpc、media 各種驅動）
- 真正跟 mantis 直接相關的是其中一小部分

所以閱讀策略要對：
1. 先看 `mantis_defconfig`
2. 再看 `mantis.dts` / `mt8695.dtsi`
3. 再沿著功能看對應 driver

---

## 你如果只會 Linux，怎麼快速定位問題？

## 情境 1：開機失敗 / 起不來

先看：
- `mantis_defconfig`
- `mantis.dts`
- `mt8695.dtsi`

重點：bootargs、記憶體、console、關鍵裝置 `status="okay"`

## 情境 2：過熱 / 卡頓 / 降速太早

先看：
- `mantis_thermal_zones.dtsi`
- `virtual_sensor_thermal.c`
- `lab126_ts_bts.c`
- `amzn_thermal_cooling.c`

## 情境 3：Wi-Fi 異常

先看：
- `mantis.dts` 的 `mmc2` / `wifi` 節點
- `drivers/misc/mediatek/connectivity/wlan/...`
- 若牽涉 MAC：`idme.c` 與讀 `/proc/idme/mac_addr` 的路徑

## 情境 4：遙控器按鍵不對

先看：
- `hid-ftv-bleremote.c`
- `hid-aspen.c`

---

## 你可以先不用碰的區域

- 大部分非 ARM64 平台（`arch/x86`, `arch/powerpc` ...）
- 與 mantis 無關的 media/PCI 驅動

這些不是無用，而是「目前你這個目標不優先」。

---

## 一頁重點（最短版）

1. 這是 Fire TV Stick 4K 的 vendor kernel（Linux 4.4.120）
2. 看懂它先抓三件事：
   - `mantis_defconfig`（功能開關）
   - `mantis.dts`（板級接線）
   - `drivers/*`（實作）
3. thermal 是最有 Amazon 客製深度的一條鏈：
   - thermistor → virtual sensor → thermal policy → cooling action
4. IDME/Board-ID 是裝置個體差異的關鍵資料來源

---

如果你要，我下一份可以直接做成「圖解版」：
- 開機時序圖（start_kernel 到 driver probe）
- Thermal 時序圖（溫度如何變成降頻）
- 檔案對照圖（你點哪裡就知道看哪個檔）
