# TR3600-Kwrt — Cudy TR3600 v1 自建固件（Kwrt / openwrt.ai 基底）

在 [kiddin9/Kwrt](https://github.com/kiddin9/Kwrt)（即 [openwrt.ai](https://openwrt.ai/) 的
線上編譯系統）之上，疊加**本倉庫自己的修正層**，用 GitHub Actions 編譯
Cudy TR3600 v1 的固件。

本倉庫**不 vendoring 上游**：workflow 在執行時 clone `kiddin9/Kwrt` 與
`openwrt/openwrt`，再把 `overlay/` 與 `files/` 疊上去。

---

## 為什麼要做這一版

目前機器上跑的固件是從 openwrt.ai 下載的 `Kwrt 25.12-SNAPSHOT 10.12.2026`
（kernel 6.12.112）。它在 TR3600 上有兩個實測出來的問題：

| 問題 | 根因 | 本倉庫的修正 |
|---|---|---|
| **風扇恆全速、完全不受控** | Kwrt 的 `devices/mediatek_filogic/patches/tr3600.patch` 把 pwm-fan 掛在 **PWM1 + `pwm1_pins`（groups = `pwm1_0`）**，而 `pwm1_0` 是 **PWM1@PIN7，與 JTAG 共用**。實機風扇在 **PWM0@PIN13**。掛錯時驅動照樣 bind、cooling device 照樣出現、`cur_state` 照樣會動（**會騙人**），但 PWM0 從未被 claim，停在 idle 準位 → 風扇全速且不受控。 | `zz-tr3600-fixes.patch`：改 `pwms = <&pwm 0 50000 0>` + `pinctrl-0 = <&pwm_pins>` |
| **風扇 idle 也在轉（>40 °C 就 50 %）** | `mt7987.dtsi` 把 **40 °C trip 對到 fan cooling level 1** → 只要 SoC 超過 40 °C 就 50 % 常轉（= 永遠）。 | 曲線移到 **68 / 78 / 88 °C** + `cooling-levels = <0 128 200 255>` |
| **USB 4G 網卡插上沒網路** | Kwrt **完全沒給這張卡建立介面**：沒有 `/lib/network/wwan/<vid>:<pid>` profile → `/etc/hotplug.d/usb/00_wwan.sh` 第一行測試就 `exit 0`；且映像裡**根本沒有 usbnet 系列 kmod**。 | `.config` 加 usbnet kmod；`uci-defaults` 建立 `network.tether`；`hotplug.d/net` 掛上實際 netdev 名 |
| **升完固件 LuCI 版面看起來不對** | Kwrt 的 nginx 對 `/luci-static/` **完全沒有送 `Cache-Control`** → 瀏覽器改用啟發式新鮮度（RFC 9111：約 `Last-Modified` 起算時間的 10 %，實測可達數小時不重驗）。而 `luci-theme-vantage` 的 cache-buster 綁的是**主題版本**：`cbi.js?v=1.0.4`、`vantage.css?v=1.0.4` —— 這兩個 URL **永遠不變**，即使 `luci-base` 換版也一樣。兩者相加 → 瀏覽器可能長時間跑上一版的 CSS/JS。 | `uci-defaults` 冪等地在 `location /luci-static` 內加 `add_header Cache-Control "no-cache";` |
| **內建 LAN 口（eth1）物理燒壞** | 對端短路（發燙的 USB 轉網口适配器）可能燒掉 RJ45/網變。軟體無法喚活死硬體，且 TR3600 硬體只有 2 個網口（eth0=WAN / eth1=LAN），沒有第 3 個可補。 | **軟體繞過**：`.config` 加 `kmod-usb-net-r8152` + `kmod-usb-net-asix`；`hotplug.d/net/30-tr3600-usb-lan` 把 RTL8153 / AX88179 這類**有線** USB 網卡橋進 `br-lan` 當 LAN 口（明確排除 4G tether 用的 `cdc_ether`/`rndis`/`qmi_wwan` 等，避免迴圈）。插上好的 dongle 即成一個可用的有線 LAN 口。 |
| **`target/linux failed to build`（mkimage 罵 `.its` syntax error）** | `CONFIG_TARGET_PER_DEVICE_ROOTFS=y` 讓 image machinery 為 target 裡**每一個** device 展開 recipe，包含我們沒選的。`target/linux/mediatek/image/filogic.mk:13` 對 Netgear 裝置呼叫 `gen_netgear_rootfs_node.sh …/root.squashfs$(if $(TARGET_PER_DEVICE_ROOTFS),+pkg=$(ROOTFS_ID/$(DEVICE_NAME)))`，而 `ROOTFS_ID/<dev>` **只有在 profile 選中時才被定義**（`include/image.mk:608-611`）→ `netgear_eax17` 拿到的路徑變成不存在的 `root.squashfs+pkg=` → `.its` 節點是空的 → `mkimage: FATAL ERROR: Unable to parse input tree`。 | ① `.config` 把 `CONFIG_TARGET_PER_DEVICE_ROOTFS` 改 **n**；② `01-tr3600-netgear-rootfs-guard.patch` 讓 `gen_netgear_rootfs_node.sh` 在 rootfs 映像不存在時 `exit 0`（不產生畸形節點） |

---

## 倉庫結構

```
overlay/devices/mediatek_filogic/
├── patches/zz-tr3600-fixes.patch   # DTS 修正（風扇通道 + 溫控曲線）
└── .config                          # 只編 cudy_tr3600-v1 + usbnet kmod

files/
├── uci-defaults/
│   ├── 99-tr3600-defaults      # 開機一次的板級預設（風扇 + tether + nginx 快取標頭）
│   └── 98-tr3600-extras        # 啟用 4G 自愈看門狗 + 寫入 /etc/config/tr3600 預設
├── etc/init.d/
│   └── tr3600-4g-wwatch        # 每分鐘排程 4G 看門狗（cron；同原廠 autoreboot 風格）
├── usr/sbin/
│   └── tr3600-4g-wwatch        # 4G 自愈階梯：重撥 → USB 重枚舉(usb3en/重 enumerate, 等價 usb_power_restart) → 選配重啟
└── hotplug.d/net/
    ├── 25-tr3600-usb-tether   # USB netdev 出現時指向 tether 介面（4G wan）
    └── 30-tr3600-usb-lan      # 有線 USB 網卡橋進 br-lan（內建 LAN 燒壞時的軟體繞過）

.github/workflows/build.yml              # GitHub Actions 建置線
```

### 兩個關鍵設計決定

1. **`zz-tr3600-fixes.patch` 是「後綴 patch」，不是取代 `tr3600.patch`。**
   Kwrt 的 patch 系列裡 `tr3000-mod.patch`、`25-platform.patch` 等會重寫
   `tr3600.patch` 也動到的共用清單（例如 `cudy,tr3000-mod` ↔
   `cudy,tr3000-v1-ubootmod`），所以直接取代 `tr3600.patch` 會 apply 失敗。
   本 patch 只動 **`tr3600.patch` 獨自建立的那個 DTS 檔**，檔名以 `zz-`
   開頭 → `sort -z` 排序後**最後**套用。

2. **板級檔案走 `files/`，不走 Kwrt 的 `devices/*/diy/`。**
   Kwrt 的 `cp -Rf ./diy/* ./` 在「來源目錄與目的目錄同名」時，
   BSD `cp` 會合併、GNU `cp` 會嵌套（`target/target/…`）—— 行為不一致。
   我們改成在 workflow 裡用 `install -D -m 0755` 明確安裝，
   順便保證 uci-defaults / hotplug 腳本的 **執行權限**正確。

---

## 怎麼用

1. 進 **Actions → Build Kwrt for Cudy TR3600 v1 → Run workflow**。
2. 參數：
   - `kwrt_ref`（預設 `25.12`）— Kwrt 建置系統的 ref
   - `openwrt_ref`（預設 `openwrt-25.12`）— 上游原始碼分支
   - `kwrt_action`（可留空）— 填 `nocache` 可跳過建置快取
   - `upload_imagebuilder`（預設 false）— 是否一併上傳 imagebuilder / initramfs
3. 編譯約 **1.5–3 小時**（Ubuntu runner，`make -j(nproc+1)`）。
4. 產物在該次 run 的 **Artifacts → `kwrt-mediatek_filogic-cudy_tr3600-v1`**。

### 刷機

只刷 **`…cudy_tr3600-v1-squashfs-sysupgrade.bin`**。

> ⚠️ `mediatek/filogic` 的 bin 目錄混著**別顆晶片**的 BL2 RAM loader，
> 刷錯會變磚。本 workflow 的 artifact 只收 `*tr3600*` + `sha256sums` +
> `profiles.json`，就是為了避開這個。

### 驗證修正有沒有生效（刷完在路由器上跑）

```sh
# 1. 風扇必須綁在 PWM0，不是 PWM1
grep -H . /sys/kernel/debug/pwm
#   要看到  pwm-0 (pwm-fan)   ← 對
#   若是    pwm-1 (pwm-fan)   ← 錯（Kwrt 原版）

# 2. 溫控曲線（第一個 active trip 應為 68 °C，不是 40 也不是 58）
cat /sys/class/thermal/thermal_zone0/trip_point_*_temp

# 3. cooling-levels = 0/128/200/255
cat /sys/class/thermal/cooling_device0/max_state     # 3

# 4. USB tether 介面存在且掛在 wan zone
uci show network.tether
uci show firewall | grep tether

# 5. LuCI 靜態資源要送 Cache-Control（修掉「升完固件版面不對」）
curl -s -D - -o /dev/null "http://127.0.0.1/luci-static/resources/cbi.js?v=1.0.4" \
  | grep -i cache-control          # 要看到 Cache-Control: no-cache
```

---

## 已知限制

- **`luci-app-fan` 會覆寫 DTS 的 trip。** `/usr/libexec/fan-control` 每次開機
  都會把「最低的那個 active trip」改寫成 `/etc/config/luci-fan` 的 `on_temp`
  （原廠預設 **58 °C**）。`files/uci-defaults/99-tr3600-defaults` 會把它改成
  **68 °C**，與 DTS 對齊；**若你在 LuCI 風扇頁手動改過 `on_temp`，DTS 的
  68 °C 就會被蓋掉**（這是預期行為，改回去即可）。
- **`trip_point_N_hyst` 一旦被用過就是唯讀** → `fan-control` 寫 hysteresis
  一定失敗（無害）。要改 hysteresis 只能改 DTS。
- **MT7990 PCIe 鏈路會掉 → kernel panic 自動重開。** 這是硬體層問題
  （AER `RxErr` → `SDES` Surprise Down → 驅動去讀已死的 BAR →
  synchronous external abort）。本版**未動** kernel cmdline
  （`chosen/bootargs` 仍是 `pci=pcie_bus_perf`）。可試的槓桿見下方。
- **USB 4G 網卡（Luat Air780EPM, `19d1:0001`）的 USB 鏈路本身臨界**：
  補上介面後掉線由 2–5 秒變成約 3 分鐘一次（每次斷約 8 秒 ≈ 4 % 停機），
  剩下的屬裝置端實體層問題（供電 / 訊號完整性）。
- **兩條高速鏈路（USB 2.0 與 PCIe）同時臨界 + 曾發生整機斷電重開
  → 強烈懷疑供電品質。** TR3600 是 USB-C PD 供電（5 V/3 A、9 V/2.2 A、
  12 V/1.6 A，最高 15 W）。**請確認用的是足瓦的 PD 電源**，這可能同時
  改善 PCIe panic 與 USB 掉線。

### 還沒試的 PCIe 槓桿

改 `target/linux/mediatek/dts/mt7987b-cudy-tr3600-v1.dts` 的
`chosen { bootargs = "… pci=pcie_bus_perf" }`：

- `pci=pcie_bus_safe` — 把 MPS 降到全鏈路最小值，對臨界鏈路較保守
- `pcie_ports=dpc-native` — 走原生 DPC（Downstream Port Containment），
  理論上可讓鏈路掉時被隔離而不是留下一個「已死但仍被讀」的裝置

（`pcie_aspm` 已排除：policy 本來就是 `performance`，即 ASPM 已關。）

---

## 授權與致謝

- 上游建置系統：**[kiddin9/Kwrt](https://github.com/kiddin9/Kwrt)**（MIT）與
  [openwrt/openwrt](https://github.com/openwrt/openwrt)（GPL-2.0）。
- 本倉庫只包含自己的修正層，不重散布上游程式碼。
- 風扇與溫控修正的數值是在**實機**上量出來的（各檔位強制 80 秒取 6 個樣本）。
