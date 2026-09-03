# device_itel_S666LN-kernel

Prebuilt kernel-adjacent artefacts for the itel RS4 (S666LN, MT6789 / Helio G99).

**This repo does NOT contain the kernel image.** The `Image` is built from source —
see `TARGET_KERNEL_SOURCE` in the device tree's `BoardConfig.mk`, which points at
`kernel/itel/S666LN` (Google GKI `android12-5.10-lts`, pinned, plus
`s666ln_defconfig`). Only artefacts with no source live here.

```
dtb/mt6789.dtb            200,368 B   MediaTek dt_table (magic d7b7ab1e), one dtb inside
dtbo.img                8,388,608 B
modules/vendor_boot/      206 .ko + modules.load / .load.recovery / .dep / .alias / .softdep
modules/vendor_dlkm/      198 .ko + modules.load / .dep / .alias / .softdep
```

## Provenance

The source of record is **stock itel firmware revision 28** (build `251212V1661`),
re-extracted from the images in `itel-RS4-S666LN-28.zip`, each CRC32-verified
against the zip's central directory before anything was read from it.

* `dtbo.img` — extracted from the stock zip, sha256 `fb908ee2…`
* `dtb/mt6789.dtb` — extracted from stock `vendor_boot.img`. Boot header v4 (GKI)
  carries no dtb, and the zip ships no standalone `dtb.img`, so the dt_table lives
  in vendor_boot's dtb section (offset = page + aligned vendor_ramdisk_size).
* `modules/vendor_boot/` — 206 modules from the stock `vendor_boot` ramdisk.
* `modules/vendor_dlkm/` — 198 modules from the stock `vendor_dlkm` image.

**Exactly two artefacts are ours rather than stock, both deliberate and both
listed here so the set can be audited without reading git history:**

| file | what it is |
|---|---|
| `modules/vendor_dlkm/mali_kbase_mt6789.ko` | Mali r54p1 kbase, built from Samsung open source against 5.10.268. vermagic `5.10.268-Riza-vanilla`. |
| `modules/vendor_boot/adaptive-ts.ko` | stock binary, one byte changed, to ungate the touchscreen in recovery. `0x6c74: 0x52806089 -> 0x52806009`. |

Everything else is byte-identical to revision 28. That is checkable directly:
extract the stock `vendor_dlkm.img` and `vendor_boot.img` and compare — 197/198
and 205/206 respectively, with the two rows above as the only differences.

🔴 **Provenance is a property to VERIFY, not to assert.** Every module in
revision 28 carries vermagic `5.10.237-android12-9-gf280f42a626b`; a module here
bearing any other vermagic, other than the two listed above, does not belong to
this revision and should be treated as a defect. Revision 28 is pinned
deliberately — the shipped identity is tied to incremental `251212V1661`, so
mixing revisions breaks the fingerprint the ROM presents.

Other trees for this device ship a **patched** dtbo and dtb; those are
deliberately not used, and stock is re-extracted instead.

## KMI

Every module here requires `module_layout = 0x7c24b32d`, and the **same** KMI covers
both sets — one kernel satisfies all 404. Verified against a from-source build of
5.10.268:

```
198 modules · 12,226 symbol refs · 10,347 matched · 0 CRC mismatches
206 modules ·  9,491 symbol refs ·  8,455 matched · 0 CRC mismatches
```

The device tree runs this check at build time (`kmi-check.py`, wired into
`droidcore` via `Android.mk`). A kernel that fails it builds fine and boots
nothing, so the gate is not optional.

⚠ The vendor_dlkm figure was 12,229 until 2026-09-03 and is now 12,226: the
symbol graph changed with the module set. Re-read it from a build rather than
carrying it forward — a stale count in the file people re-derive from is worse
than no count.

## Load order

`modules.load` is **not** the module list — both partitions ship more than they
load (206 ships / 171 loads; 198 ships / 177 loads), and recovery loads *more*
than normal boot (199 vs 171) because it needs touch, display and charging up
front. `BOARD_*_KERNEL_MODULES_LOAD` must be set explicitly; leaving it unset
makes the build default to the full list.

## Known stock defect

`subaf.ko` (vendor_dlkm) references `dw9781caf_init` / `_poweroff` / `_target`
from a DW9781C OIS driver that itel does not ship. It fails to load on stock too.
Not a regression, and not something to "fix" here.

## License

Blobs are itel's and MediaTek's. Repo scaffolding: Apache-2.0.
