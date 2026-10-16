# PixelOS 16 QPR2 for realme GT Neo 3 (zephyr)

This repository contains the build launcher for Crave's **existing PixelOS project**. It is not a ROM source mirror, and it does not contain proprietary vendor blobs.

## Launch the formal build

Use the already configured PixelOS Crave project (project ID `82`). From the existing Devspace:

~~~bash
cd /crave-devspaces/PixelOS
crave run --projectID 82 --no-patch --detached -- "curl -fLSs --retry 3 https://raw.githubusercontent.com/asxcaz/pos16-zephyr/main/build.sh -o /tmp/pos16-zephyr-build.sh && bash /tmp/pos16-zephyr-build.sh"
~~~

The script first verifies that the workspace already has an initialized `.repo` checkout. It then re-pins the **existing** repo client to the official PixelOS `sixteen-qpr2` manifest and performs an incremental sync of the projects described by that manifest plus the local zephyr manifest.

**This updates the existing checkout; it does not run `--clean`, `--force-sync`, or clone a new PixelOS source tree.** Existing Git objects and checkouts are reused, while projects that are behind their manifest revisions will be fetched/updated. Because the base source provided by Crave may be older, the full manifest sync is intentional; syncing only the device repos would leave the Android/PixelOS base stale.

Do not run `repo sync` or Android build commands directly in the interactive Devspace shell. Submit the build with `crave run` from inside the project.

## Source branches currently configured

- PixelOS manifest/base projects: `PixelOS-AOSP/android_manifest`, `sixteen-qpr2`
- Device tree: `oplus-mt6895/android_device_oplus_zephyr`, `lineage-23.2`
- Official prebuilt kernel tree: [NINJAHUNTER1357/android_device_oplus_zephyr-kernel](https://gitlab.com/NINJAHUNTER1357/android_device_oplus_zephyr-kernel), `los-23.2`
- MediaTek hardware: `oplus-mt6895/android_hardware_mediatek`, `lineage-23.2`
- OPlus hardware: `oplus-mt6895/android_hardware_oplus`, `lineage-23.2`
- MediaTek vendor sepolicy: `oplus-mt6895/android_device_mediatek_sepolicy_vndr`, `lineage-23.2`
- Vendor tree: [NINJAHUNTER1357/proprietary_vendor_oplus_zephyr](https://gitlab.com/NINJAHUNTER1357/proprietary_vendor_oplus_zephyr), `lineage-23.2`
- MediaTek IMS: [Adarsh0127-Elite/android_vendor_mediatek_ims](https://github.com/Adarsh0127-Elite/android_vendor_mediatek_ims), `android-16-qpr2`

The device tree configures a prebuilt kernel (`TARGET_FORCE_PREBUILT_KERNEL := true`); this script does not build the Linux kernel source. Before entering Android build setup it verifies the kernel image, MT6895 DTB, all three module load lists, every listed module, and the extra modules explicitly named by `BoardConfig.mk`.

## Vendor and IMS handling

The required vendor makefiles are:

- `vendor/oplus/zephyr/zephyr-vendor.mk`
- `vendor/oplus/zephyr/BoardConfigVendor.mk`

When `vendor/oplus/zephyr` is absent, the script shallow-clones the documented GitLab Vendor tree at `lineage-23.2`. It refuses to overwrite an existing partial vendor directory; `POS16_VENDOR_BRANCH` can override the default for testing another compatible branch. It also pins IMS to `android-16-qpr2` and checks that `vendor/mediatek/ims/ims.mk` exists.

Do not commit proprietary blobs, firmware images, signing keys, or credentials to this public build-helper repository.

## Patch behavior

- The Aperture MediaTek 60 FPS patch is checked for already-applied/applicable state.
- The GED-KPI Surface patch is deliberately skipped because it conflicts with PixelOS `sixteen-qpr2`'s Surface API and needs a real port.
- The MediaTek UDFPS dither patch is a child of the GED-KPI patch, so it is skipped with the parent instead of being attempted in isolation.
- The MediaTek UDFPS GHBM patch is checked against the actual base source at build time. If it does not apply, the script logs a warning and continues. That does not by itself block compilation, but fingerprint high-brightness/dimming behavior should be validated on a built ROM; a successful compile would not prove the feature works correctly.

## What the build does

1. Reuses the existing Crave PixelOS source root and updates the manifest selection to `sixteen-qpr2`.
2. Runs an incremental, non-destructive sync over the existing PixelOS source and zephyr-specific repositories.
3. Retrieves Vendor and verifies required makefiles.
4. Verifies the prebuilt kernel and its referenced module files, then creates the PixelOS `custom_zephyr` product config.
5. Retrieves the MediaTek IMS dependency and runs compatible optional patches.
6. Executes `lunch custom_zephyr-bp4a-userdebug` and the real ROM build command `m -j"$JOBS" pixelos`.
7. Lists any zip/image artifacts under `out/target/product/zephyr` when the build completes.

## Automated checks

GitHub Actions validates shell syntax, ShellCheck, the local manifest, target/build commands, and safe refusal without a Crave checkout. A second job shallow-clones relevant upstream repos (including LFS-backed Vendor and kernel artifacts) to verify branches/files and report whether framework patches apply to the current PixelOS branch. **This is a deep source/patch preflight, not a ROM compile.** A real ROM build still requires a Crave build job.
