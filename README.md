# device_itel_S666LN-kernel

Prebuilt kernel-adjacent artefacts for the itel RS4 (S666LN, MT6789 / Helio G99).

**Branch `lineage-23.2`: this repo DOES carry the kernel image**, in `prebuilt/`. It is the
custom kernel built by the separate kernel project
([`itel-rs4-kernel`](https://github.com/riza220201/itel-rs4-kernel): Neutron clang, BORE, ntsync, and
`epoll_pwait2`, which Android 16's libgui needs and 5.10 lacks — without it BLASTBufferQueue spins on ENOSYS),
imported by the device tree's `import-kernel.sh`, which refuses the import unless the `Image.gz`
is byte-identical to the kernel inside the **published** boot image and the KMI gate passes on both
module sets below. With `prebuilt/{Image.gz, vmlinux.symvers, IMPORTED}` present (the normal case) the
ROM never compiles a kernel. Without `prebuilt/` at all, the device tree's `BoardConfig.mk` falls back,
loudly, to building plain GKI from `kernel/itel/S666LN`, so the tree still builds from public sources;
a partial `prebuilt/` stops the build. (On `lineage-20.0` the image was built from `kernel/itel/S666LN`
and `prebuilt/` was ignored.)

```
prebuilt/Image.gz          the kernel (published v10 vanilla: 5.10.272-Riza-vanilla)
prebuilt/vmlinux.symvers   its symbol CRCs, read by the build-time KMI gate
prebuilt/kernel.config     provenance only
prebuilt/IMPORTED          where it came from, which published image proved it, sha256 of the three
```

Everything else here has no source and comes from stock:

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

**Exactly three artefacts are ours rather than stock, all deliberate and all
listed here so the set can be audited without reading git history:**

| file | what it is |
|---|---|
| `modules/vendor_dlkm/mali_kbase_mt6789.ko` | Mali r54p1 kbase, built from Samsung open source (SM-A175F) against the imported v9 kernel, 5.10.272. vermagic `5.10.272-Riza-vanilla`, srcversion `EFC2461A110EA84728CF6A0` (unchanged since 5.10.260: same source, only the kernel moved). sha256 `c131c029…`. |
| `modules/vendor_dlkm/mtk_gpufreq_mt6789.ko` | MT6789 GPU frequency driver, built from Samsung open source (SM-A175F `gpufreq/v2`, `gpuppm` from `v2_legacy`) ported to revision 28's ABI, against 5.10.272, clang 12.0.5. **Stock behaviour, no overclock:** OPP table, limit/adjust tables and fp tables byte-identical to revision 28's module, OPP0 = 1,100,000 kHz, AVS on; 158 of 160 functions identical to stock (the other two: a `__LINE__` constant and one branch layout). Built by `~/itel-rs4-devicetree/tools/kernel-patches/gpufreq/build-gpufreq.sh` (versioned source, overclock code removed, deterministic); 161 of 161 functions identical to the first build of it (`5f44c321…`, only the embedded source path differed). sha256 `71b45e97…`. |
| `modules/vendor_boot/adaptive-ts.ko` | stock binary, one byte changed, to ungate the touchscreen in recovery. `0x6c74: 0x52806089 -> 0x52806009`. |

Everything else is byte-identical to revision 28. That is checkable directly:
extract the stock `vendor_dlkm.img` and `vendor_boot.img` and compare — 196/198
and 205/206 respectively, with the three rows above as the only differences.

🔴 **Provenance is a property to VERIFY, not to assert.** Every module in
revision 28 carries vermagic `5.10.237-android12-9-gf280f42a626b`; a module here
bearing any other vermagic, other than the two built ones listed above, does not belong to
this revision and should be treated as a defect. Revision 28 is pinned
deliberately — the shipped identity is tied to incremental `251212V1661`, so
mixing revisions breaks the fingerprint the ROM presents.

Other trees for this device ship a **patched** dtbo and dtb; those are
deliberately not used, and stock is re-extracted instead.

## KMI

Every module here requires `module_layout = 0x7c24b32d`, and the **same** KMI covers
both sets — one kernel satisfies all 404. Verified against the imported v10 kernel
(`prebuilt/vmlinux.symvers`, 5.10.272), with both rebuilt GPU modules in place (2026-11-15; v9 gave
the same counts on 2026-11-13):

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
