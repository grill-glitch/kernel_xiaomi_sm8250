# LineageOS 23.2 base + KernelSU-Next + SUSFS v2.3.0 (alioth / SM8250)

## What this is

A working fork of `LineageOS/android_kernel_xiaomi_sm8250` branch `lineage-23.2` with:

1. **KernelSU-Next** (legacy branch, `8af3d4fec3`) integrated via `drivers/kernelsu` (manual-hook mode — `CONFIG_KSU_SYSCALL_TABLE_HOOK` is unsupported on 4.19 and explicitly disabled).
2. **SUSFS v2.3.0** (non-GKI variant) — the version embedded in crDroid's `android_kernel_xiaomi_sm8150` branch `16.0`, backported to this 4.19.325 tree.
3. Verified end-to-end on **Xiaomi alioth** running **crDroid 16.0** (`lineage_alioth-bp4a-userdebug`), boot slot `_b`.

Target device: `Xiaomi Mi 11 / Mi 11 Pro / Mi 11 Ultra` (SM8250 / kona, codename `alioth`).
Kernel base: `4.19.325` (cip131-stable).

## Repository layout

```
.
├── .gitignore                 # ignores out/, ksu-backup/, KernelSU-Next/, drivers/kernelsu
├── arch/arm64/configs/.../alioth.config     # CONFIG_KSU / CONFIG_KSU_SUSFS enabled
├── drivers/{Kconfig,Makefile}               # drivers/kernelsu glue (3 lines total)
├── fs/susfs.c                               # SUSFS v2.3.0 core
├── fs/{namei,namespace,stat,readdir,...}.c  # SUS_PATH/SUS_MOUNT/SUS_KSTAT/SUS_MAP hooks
├── include/linux/{susfs.h,susfs_def.h}      # SUSFS uapi + cmd ids
├── mm/{memory,memfd}.c                      # SUS_MAP / SUS_MEMFD hooks
└── kernel/{kallsyms,reboot,sys}.c           # HIDE_KSU_SUSFS_SYMBOLS / supercall dispatch
```

## Branch layout

- **`lineage-23.2-ksu-susfs-v230`** (default) — clean commit chain on top of `LineageOS/android_kernel_xiaomi_sm8250@71b13e62`.

Commit history (newest first):

```
d5c05907  alioth: integrate SUSFS v2.3.0 (non-GKI 4.19)               35 files
cb0013ef  alioth: enable CONFIG_KSU + CONFIG_KSU_SUSFS in defconfig   1 file
a72a8935  alioth: wire drivers/kernelsu build glue (Kconfig + Makefile) 2 files
2336a7f5  .gitignore: ignore build outputs and external KSU checkout   1 file
71b13e62  (lineage-23.2 base — camera: Fix tele5x OIS firmware download path)
```

Each commit is independently revertable.

## Build prerequisites

`KernelSU-Next/` is **not** tracked in this repo (it's a separate `git clone` of `KernelSU-Next/KernelSU-Next`):

```bash
# Required: populate KernelSU-Next/kernel before building.
git clone -b legacy https://github.com/KernelSU-Next/KernelSU-Next
ln -s ../KernelSU-Next/kernel drivers/kernelsu
```

Then build the usual AOSP way:

```bash
cd /path/to/crDroid
source build/envsetup.sh
lunch lineage_alioth-bp4a-userdebug
mka bootimage
```

Output: `out/target/product/alioth/boot.img` (~192 MB).

## Verified behaviour (real device)

Booted on `alioth` slot `_b`, sha256回读一致 with `out/target/product/alioth/boot.img`:

```
dmesg:    susfs is initialized! version: v2.3.0
ksud susfs support  -> Supported
ksud susfs version  -> v2.3.0
ksud susfs variant  -> NON-GKI
ksud susfs features -> 10 entries (CONFIG_KSU_SUSFS_SUS_{PATH,MOUNT,KSTAT,MAP} etc.)
dmesg:    susfs_sdcard_monitor_fn start monitoring /data/media/0/Android
```

The 17-CMD KernelSU-Next reboot(2) supercall dispatcher matches `crdroidandroid/android_kernel_xiaomi_sm8150@16.0` one-for-one.

## Known gap (not a regression)

`/data/adb/ksu/bin/ksu_susfs` from the v1.5.9 era is no longer ABI-compatible:
`struct st_susfs_sus_path` changed from `{ino; pathname; uid}` to `{pathname; err}`.
Read commands (`show version`, `show variant`, `show enabled_features`) still work;
write commands (`add_sus_path`, etc.) pass garbage where the pathname belongs.
Rebuild the tool against `include/linux/susfs.h` (24 `st_susfs_*` structs) to restore writes.

## Source provenance

SUSFS v2.3.0 was ported from
[`crdroidandroid/android_kernel_xiaomi_sm8150`](https://github.com/crdroidandroid/android_kernel_xiaomi_sm8150)
branch `16.0` (4.14.357 non-GKI reference). Cross-referenced against
[`yspbwx2010/kernel_xiaomi_sm8250_mod`](https://github.com/yspbwx2010/kernel_xiaomi_sm8250_mod) and
[`liyafe1997/kernel_xiaomi_sm8250_mod`](https://github.com/liyafe1997/kernel_xiaomi_sm8250_mod) —
neither modified v2.3.0 source (both froze at SUSFS v1.5.9); archaeology report:
`ksu-backup/feature-migration/stage1-archaeology-report.md` in the local checkout.