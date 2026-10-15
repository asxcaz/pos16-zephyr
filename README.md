# PixelOS 16 QPR2 for realme GT Neo 3 (zephyr)

This repository contains the build launcher for Crave's **already-synced PixelOS project**. It is not a ROM source mirror.

## Launch from the existing PixelOS Crave workspace

Run this in the Crave terminal/CLI associated with the PixelOS project, not in an empty checkout of this script repository:

~~~bash
crave run --no-patch -- "curl -fLSs --retry 3 https://raw.githubusercontent.com/asxcaz/pos16-zephyr/ded24d69cd5dceb42c0f21a8542c5f51e0a3d462/build.sh -o /tmp/pos16-zephyr-build.sh && bash /tmp/pos16-zephyr-build.sh"
~~~

The command deliberately does not use --clean. The script refuses to run repo init or create a fresh PixelOS checkout. It runs a targeted incremental repo sync for the five zephyr/device dependency paths only, not all PixelOS manifest projects, so it can reuse Crave's pre-synced source tree. It does not run repo sync inside the interactive Devspace.

## Source branches used

- PixelOS base: sixteen-qpr2 (already provisioned by the Crave PixelOS project)
- Device tree: oplus-mt6895/android_device_oplus_zephyr, lineage-23.2
- Prebuilt kernel: V1NAY007/android_device_oplus_zephyr-kernel, upstream-test-5.10.260
- MediaTek hardware: oplus-mt6895/android_hardware_mediatek, lineage-23.2
- OPlus hardware: oplus-mt6895/android_hardware_oplus, lineage-23.2
- MediaTek vendor sepolicy: oplus-mt6895/android_device_mediatek_sepolicy_vndr, lineage-23.2

The device tree BoardConfig uses device/oplus/zephyr-kernel as a prebuilt kernel and sets TARGET_FORCE_PREBUILT_KERNEL. Therefore this script uses the supplied 5.10.260 Image.gz and MT6895 DTB; it does not build android_kernel_oplus_mt6895 from source. If changing to a source-built kernel later, use that kernel repository's lineage-24.0 branch, not the deprecated 23.2 branch.

## Proprietary vendor files are required

The public device tree references both of these files:

- vendor/oplus/zephyr/zephyr-vendor.mk
- vendor/oplus/zephyr/BoardConfigVendor.mk

They are not in the public device-tree repository. The build script checks for them and stops with an explicit error if they are absent. Extract the proprietary blobs from the compatible OnePlus 10R (pickle) stock firmware/device or stage a trusted vendor backup into the Crave build workspace before retrying.

Do not commit proprietary blobs, firmware images, signing keys, or credentials to this public repository. The script does not try to fabricate missing vendor files.

## What build.sh does

1. Locates the existing PixelOS source root and fails if no initialized .repo tree exists.
2. Writes a local manifest for the zephyr device tree, 5.10.260 prebuilt kernel, and required open-source dependencies.
3. Runs targeted incremental repo sync only for the five device/dependency paths, without --force-sync or --clean.
4. Creates a PixelOS custom_zephyr product definition from the existing device configuration.
5. Clones the Android 16 QPR2 MediaTek IMS project and applies the specific compatibility commits referenced by the device tree, skipping commits already present.
6. Checks for proprietary vendor files, then runs lunch custom_zephyr-bp4a-userdebug and m pixelos.
7. Lists zip/image artifacts in out/target/product/zephyr when the build completes.

## Verification status

GitHub Actions (`.github/workflows/validate.yml`) checks Bash syntax, ShellCheck, the embedded manifest branches/paths, and safe refusal when no Crave source checkout exists. These are shallow checks only: the workflow does not sync Android sources or build a ROM. An actual ROM build has not been run from this repository: that requires a live Crave build job and the compatible proprietary vendor files.
