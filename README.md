# LineageOS 23.2 base + KernelSU-Next + SUSFS v2.3.0 + NetHunter capabilities

A working fork of `LineageOS/android_kernel_xiaomi_sm8250` branch `lineage-23.2` with
three layers of additions:

1. **KernelSU-Next** (legacy branch, `8af3d4fec3`) integrated via `drivers/kernelsu`
   (manual-hook mode — `CONFIG_KSU_SYSCALL_TABLE_HOOK` is unsupported on 4.19).
2. **SUSFS v2.3.0** (non-GKI variant) — the version embedded in
   `crdroidandroid/android_kernel_xiaomi_sm8150` branch `16.0`, backported to
   this 4.19.325 tree.
3. **Stage 1 + NetHunter capabilities** — USB serial adapters (CH341/FTDI/PL2303/...),
   USB-CAN (gs_usb/etc.), ZRAM LZ4HC, USB gadget functions (RNDIS/ECM/ACM),
   Bluetooth RFCOMM/BNEP/HIDP/HCIBTUSB, TC actions (CSUM/NAT/PEDIT), and USB
   1.1 / 2.0 host controller fallback.

Target device: `Xiaomi Mi 11 / Mi 11 Pro / Mi 11 Ultra` (SM8250 / kona,
codename `alioth`). Verified end-to-end on `alioth` running crDroid 16.0,
boot slot `_b`.

Kernel base: `4.19.325` (cip131-stable).

---

## Quick start

### Prerequisites

```bash
# KernelSU-Next/ is NOT tracked in this repo (independent git clone).
# Required: populate it before building.
cd kernel/xiaomi/sm8250
git clone -b legacy https://github.com/KernelSU-Next/KernelSU-Next
ln -s ../KernelSU-Next/kernel drivers/kernelsu
```

`KernelSU-Next/` and `drivers/kernelsu` are listed in `.gitignore` —
they are independent git checkouts maintained outside this repo.

### Build

```bash
cd /path/to/crDroid
source build/envsetup.sh
lunch lineage_alioth-bp4a-userdebug
mka bootimage
```

Output: `out/target/product/alioth/boot.img` (~192 MB).

### Flash (active slot only)

This repo uses an A/B device. **Always flash the currently active slot
(usually `_b`) — never flash the inactive slot. Flashing the inactive slot
with a non-bootable image leaves the device stuck in fastboot with no way
to recover remotely.**

```bash
# 1. Confirm current slot
fastboot getvar current-slot   # → b

# 2. Flash
fastboot flash boot_b boot.img

# 3. Reboot
fastboot reboot
```

For safe experimentation, use `fastboot boot boot.img` instead — it loads
the image into RAM and boots from it without writing to any partition.

---

## Branch layout

Default branch: **`lineage-23.2-ksu-susfs-v230`** (13 commits on top of
`71b13e62` LineageOS base).

### Commit history (newest first)

| SHA       | Subject |
| --------- | ------- |
| `1899485b` | enable USB 1.1 / 2.0 host controller fallback for NetHunter |
| `d856b713` | enable TC actions (CSUM/NAT/PEDIT) for NetHunter MITM tooling |
| `ef9207d7` | enable Bluetooth RFCOMM/BNEP/HIDP/HCIBTUSB for NetHunter |
| `612d9001` | enable USB_CONFIGFS_ACM for NetHunter USB serial gadget |
| `d08a7a36` | enable USB_CONFIGFS_ECM for NetHunter USB tethering |
| `b0469481` | enable USB_CONFIGFS_RNDIS for NetHunter USB tethering |
| `315e9483` | enable CRYPTO_LZ4HC for ZRAM |
| `7ff1601b` | enable USB-CAN adapters for SocketCAN |
| `404892d1` | enable USB serial adapters (CH341/FTDI/PL2303/OTI6858/TI/SPCP8X5/UPD78F0730) |
| `d5c05907` | integrate SUSFS v2.3.0 (non-GKI 4.19) |
| `cb0013ef` | enable CONFIG_KSU + CONFIG_KSU_SUSFS in defconfig |
| `a72a8935` | wire drivers/kernelsu build glue (Kconfig + Makefile) |
| `2336a7f5` | .gitignore: ignore build outputs and external KSU checkout |
| `71b13e62` | (LineageOS base — camera: Fix tele5x OIS firmware download path) |

Each commit is independently revertable with `git revert <sha>`.

---

## What's enabled

### Layer 1 — KernelSU + SUSFS

- **`CONFIG_KSU=y`** + **`CONFIG_KSU_MANUAL_HOOK=y`**
- **`CONFIG_KSU_SUSFS=y`** with all 10 user-selectable features
- SUSFS version: **`v2.3.0`**, variant **`NON-GKI`**
- KSU Manager: `com.rifsxd.ksunext` reports `SuSFS 版本 支持 | v2.3.0 (NON-GKI)`
- 17-CMD KSU-Next reboot(2) supercall dispatcher matching
  `crdroidandroid/android_kernel_xiaomi_sm8150@16.0` one-for-one
- `susfs_sdcard_monitor_fn` watches `/data/media/0/Android` via fsnotify

### Layer 2 — Stage 1 (USB Serial / USB-CAN / ZRAM)

- **USB serial adapters**: CH341, FTDI_SIO, PL2303, OTI6858, TI 3410/5052,
  SPCP8X5, UPD78F0730 (CP210X was already on). No source change — defconfig
  toggle only. QT2 (driver source absent in 4.19 mainline) is intentionally
  NOT enabled.
- **USB-CAN adapters**: gs_usb (CANable / candleLight), kvaser_usb,
  mcba_usb, ucan, 8dev_usb. **PEAK_USB is OFF** — its driver source
  (`drivers/net/can/usb/peak_usb/pcan_usb_pro.c:136`) trips `-Wvarargs` on
  4.19 + clang r563880c `-Werror` and has no defconfig-only fix.
- **ZRAM LZ4HC**: `CONFIG_CRYPTO_LZ4HC=y` — restores the `lz4hc` algorithm
  string in `/sys/block/zram0/comp_algorithm` (was silently missing
  despite `zcomp.c` registering the slot conditionally). 842 / LZ4K /
  LZ4KD / LZO-RLE are deferred (stage 2 territory).

### Layer 3 — NetHunter capabilities

- **USB Gadget** (CONFIGFS, kernel-registered):
  - `USB_CONFIGFS_RNDIS` (Windows USB tethering) — `rndis.rndis` exposed
  - `USB_CONFIGFS_ECM`   (Mac / Linux USB tethering) — `ecm.0` creatable
  - `USB_CONFIGFS_ACM`   (USB serial gadget) — `acm.0` creatable → `/dev/ttyGS*`
  - HID/F_FS/MASS_STORAGE/NCM/UAC1/UAC2/UVC/MIDI/MTP/PTP/ACC/AUDIO_SRC
    were already enabled by the parent fragments.
- **Bluetooth protocol stack extensions**:
  - `BT_RFCOMM` (Bluetooth serial profile)
  - `BT_BNEP`   (PAN profile — BNEP tethering)
  - `BT_HIDP`   (HID profile — Bluetooth keyboard/mouse)
  - `BT_HCIBTUSB` (USB-Bluetooth HCI transport for external BT adapters)
  - Internal QCA Bluetooth is untouched — these only add the external
    transport and the protocol layers above it.
- **TC actions for MITM**:
  - `NET_ACT_CSUM`   (update TCP/UDP checksum after packet edit)
  - `NET_ACT_NAT`    (NAT inside tc filter — replace src/dst IP:port)
  - `NET_ACT_PEDIT`  (byte-level packet editor)
  - These are the core of NetHunter on-device MITM tooling (sslstrip,
    netsed, tcprewrite, evilginx-style payload rewriting).
- **USB 1.1 / 2.0 host controller fallback**:
  - `USB_EHCI_HCD` + `USB_EHCI_HCD_PLATFORM`
  - `USB_OHCI_HCD` + `USB_OHCI_HCD_PLATFORM`
  - When alioth is put into USB-host mode (OTG + hub), older USB 1.1/2.0
    sniffers / rubber-ducky clones that predate xHCI work via these
    fallback controllers. xHCI is still the primary controller.

### Explicitly NOT enabled (deliberate decisions)

- `RTL8812AU` — driver source absent in 4.19 mainline; BLOCKED until
  out-of-tree (aircrack-ng) source is provided.
- `MAC80211` / `CFG80211_WEXT` — tightly coupled to the internal
  Qualcomm Wi-Fi chipset; modern NetHunter uses external USB adapters,
  not legacy wext monitor mode.
- `HID_QVR` — Windows Mixed Reality headset driver, irrelevant to
  NetHunter use cases.
- `NETLABEL` — conflicts with Android's mandatory SELinux MAC.
- `NFC_PN533/PN544`, `RFD_FTL/ULO`, `KSDRIVER` — old NetHunter / RFID
  / SDR stack components not used by modern NetHunter.
- `842 / LZ4K / LZ4KD / LZO-RLE` ZRAM backends — require broader zram
  / lib backports (stage 2).
- `CONFIGFS_F_PRINTER / F_TCM / F_LB_SS` — InfiniR itself does not
  enable these either.

---

## Verified behaviour (real device)

```
dmesg: susfs is initialized! version: v2.3.0
ksud susfs support  -> Supported
ksud susfs version  -> v2.3.0
ksud susfs variant  -> NON-GKI
ksud susfs features -> 10 entries (CONFIG_KSU_SUSFS_SUS_{PATH,MOUNT,KSTAT,MAP}, ...)
dmesg: susfs_sdcard_monitor_fn start monitoring /data/media/0/Android
ls /sys/block/zram0/comp_algorithm
  → [lzo] lz4 lz4hc zstd   (lz4hc restored)
ls /config/usb_gadget/g1/functions/
  → rndis.rndis, ncm.gs6, mass_storage.0, uac1.uac1, uac2.0, uvc.0, midi.gs5, ...
dmesg: Bluetooth: Core ver 2.22
dmesg: Bluetooth: RFCOMM socket layer initialized / RFCOMM ver 1.11
dmesg: usbcore: registered new interface driver btusb
nm vmlinux: tcf_csum_init, tcf_nat_init, tcf_pedit_init all present
nm vmlinux: ehci_hcd_init, ohci_hcd_mod_init, xhci_hcd_init all present
```

**panic / BUG / Oops check: 0** (dmesg grep returns 2-8 hits, all are
benign: `ramoops` / `reboot=panic_warm` kernel param / DRM debug strings).

---

## Known gaps (not regressions)

### v1.5.9-era userspace tool ABI mismatch

`/data/adb/ksu/bin/ksu_susfs` from the v1.5.9 era is no longer ABI-compatible:
`struct st_susfs_sus_path` changed from `{ino; pathname; uid}` to
`{pathname; err}` in v2.3.0.

- **Read commands still work**: `show version`, `show variant`,
  `show enabled_features` return correct values.
- **Write commands are broken**: `add_sus_path` etc. hand the kernel
  garbage where the pathname should be. Symptom in dmesg:
  ```
  susfs_add_sus_path] failed opening file ' <garbage> '
  CMD_SUSFS_ADD_SUS_PATH -> ret: -2
  ```
- **Fix**: rebuild the tool against `include/linux/susfs.h`
  (24 `st_susfs_*` structs).

### USB_CONFIGFS_F_ECM/ACM are exposed but not mkdir'd by Android HAL

`f_ecm.o` and `f_acm.o` ARE built into vmlinux and registered with
CONFIGFS (`mkdir /config/usb_gadget/g1/functions/ecm.0` succeeds with
the `ecm.0` directory appearing). However, the Android gadget HAL only
auto-creates the standard set (`mtp`, `ptp`, `adb`, `rndis`, `ncm`, ...).
NetHunter userspace creates `ecm.0` / `acm.0` on demand. To enable:

```bash
su -c "mkdir /config/usb_gadget/g1/functions/ecm.0"
su -c "mkdir /config/usb_gadget/g1/functions/acm.0"
```

---

## Source provenance

| Layer | Source | Reference |
| ----- | ------ | --------- |
| SUSFS v2.3.0 | `crdroidandroid/android_kernel_xiaomi_sm8150` branch `16.0` (4.14.357 non-GKI) | archaeology report: `ksu-backup/feature-migration/stage1-archaeology-report.md` |
| KernelSU-Next | `KernelSU-Next/KernelSU-Next` branch `legacy` @ `8af3d4fec3` | upstream repo |
| NetHunter audit | `raystef66/InfiniR_kernel_alioth` branch `16.0-alioth` | archaeology report: `ksu-backup/feature-migration/NETHUNTER_MIGRATION.md` |

Each commit's message includes the upstream reference SHA where applicable.

---

## Build outputs

The repository does NOT commit build outputs. Local artifacts live in
`out/target/product/alioth/obj/KERNEL_OBJ/` and are ignored.

Last verified `boot.img` (sha256 `fcba3d8f9f2ae33a…`) boots cleanly on
`alioth` slot `_b`. To rebuild: see **Quick start → Build** above.

---

## Contributing

Revert-friendly by design: every commit is a single logical change that
builds and boots independently. Open a PR against
`lineage-23.2-ksu-susfs-v230` with new commits layered on top.

Things explicitly out of scope:
- Wholesale InfiniR / ReSukiSU / NetHunter kernels merge
- Reverting or replacing KernelSU / SUSFS / KSU-Next with older variants
- Modifying vendor USB HAL, `init.rc`, or userspace NetHunter tooling
- Wholesale Linux subsystem backports (Bluetooth, cfg80211, USB, network)

---

## License

GPL-2.0 (kernel). See individual file headers for exceptions.

---

# 中文说明

针对 `alioth` (小米 11 / Pro / Ultra, SM8250) 的内核 fork，基于
LineageOS 23.2 lineage-23.2 分支。三层增量：

1. **KernelSU-Next** (legacy 分支) — `drivers/kernelsu` 软链接入，manual-hook 模式
   (4.19 不支持 `SYSCALL_TABLE_HOOK`)。
2. **SUSFS v2.3.0** (non-GKI) — 从 crDroid `android_kernel_xiaomi_sm8150`
   分支 `16.0` (4.14.357) backport 到本树 4.19.325。
3. **Stage 1 + NetHunter capability** — USB 串口适配器 (CH341/FTDI/PL2303 等)、
   USB-CAN (gs_usb 等)、ZRAM LZ4HC、USB gadget (RNDIS/ECM/ACM)、
   Bluetooth RFCOMM/BNEP/HIDP/HCIBTUSB、TC actions (CSUM/NAT/PEDIT)、
   USB 1.1/2.0 host 控制器 fallback。

## 编译与刷机

```bash
# 1. 克隆 KernelSU-Next (外部 git clone，不在本仓库内)
cd kernel/xiaomi/sm8250
git clone -b legacy https://github.com/KernelSU-Next/KernelSU-Next
ln -s ../KernelSU-Next/kernel drivers/kernelsu

# 2. 编译
cd /path/to/crDroid
source build/envsetup.sh
lunch lineage_alioth-bp4a-userdebug
mka bootimage

# 3. 刷机 (只刷当前 active slot，禁止刷非活动 slot)
fastboot getvar current-slot   # 确认是 b
fastboot flash boot_b out/target/product/alioth/boot.img
fastboot reboot
```

调试时用 `fastboot boot boot.img` 临时启动，不写任何分区。

## 验证

真机 `alioth` slot `_b`，已验证：
- SUSFS v2.3.0 / NON-GKI (10 features)
- ZRAM `lz4hc` 已可选 (`/sys/block/zram0/comp_algorithm`)
- USB Gadget functions 全注册 (RNDIS/NCM/MASS_STORAGE/UAC2/UVC/MIDI/...)
- Bluetooth RFCOMM/BNEP/HIDP/btusb 全部 init 成功
- TC actions 全部编入 (CSUM/NAT/PEDIT)
- USB EHCI/OHCI/xHCI 三套 host 控制器都 init
- 无 panic / BUG / Oops

详见上方英文版的 "Verified behaviour" 段。

## 已知缺口

- v1.5.9 时代的 `ksu_susfs` 工具**写命令不能用了**（struct ABI 不兼容）；
  读命令照常工作。
- USB Gadget ECM/ACM kernel 侧已注册但 Android HAL 默认不 mkdir，
  NetHunter userspace 需自行 `mkdir /config/usb_gadget/g1/functions/ecm.0`。
- RTL8812AU / MAC80211 monitor / 旧版 NetHunter 攻击工具链**未集成** — 见英文版
  "Explicitly NOT enabled" 段。

## 来源

完整 git 考古报告在 `ksu-backup/feature-migration/`：
- `stage1-archaeology-report.md` — Stage 1 (USB Serial / EROFS / CAN / ZRAM) 考古
- `NETHUNTER_MIGRATION.md` — NetHunter capability 调研与决策矩阵