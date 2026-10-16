# PixelOS 16 QPR2 for realme GT Neo 3 (zephyr)

This repository contains the build launcher for Crave's **already-synced PixelOS project**. It is not a ROM source mirror.

## Launch from the existing PixelOS Crave workspace

Your `crave list` shows PixelOS is project ID `82`. First create a snapclone workspace for that already-configured project, unless it already exists:

~~~bash
crave clone list
# Only if /crave-devspaces/PixelOS does not already exist:
crave clone create --projectID 82 /crave-devspaces/PixelOS
cd /crave-devspaces/PixelOS
~~~

The `clone create` operation creates a workspace snapshot of Crave's configured PixelOS project; it is not a manual `repo init`/full source download. Do not run `repo sync` or build commands directly in the Devspace shell. The build must be submitted with `crave run` from inside this project directory. Running `crave run` from `/crave-devspaces` itself causes `Could not get project root/url` because that parent directory is not a project clone.

Submit the build from `/crave-devspaces/PixelOS`:

~~~bash
crave run --projectID 82 --no-patch --detached -- "curl -fLSs --retry 3 https://raw.githubusercontent.com/asxcaz/pos16-zephyr/36f130dc3685b28f92c45833fd1eb680de3bec4d/build.sh -o /tmp/pos16-zephyr-build.sh && bash /tmp/pos16-zephyr-build.sh"
~~~

The command deliberately does not use `--clean`. The script refuses to run `repo init` or create a fresh PixelOS checkout. Inside the queued build job, it runs a targeted incremental `repo sync` for the five zephyr/device dependency paths only—not all PixelOS manifest projects—so it reuses Crave's pre-synced source tree.

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

They are not in the public device-tree repository. The build script checks for them immediately after the device dependencies sync and stops with an explicit error before downloading IMS or applying patches if they are absent. Extract the proprietary blobs from the compatible OnePlus 10R (pickle) stock firmware/device or stage a trusted vendor backup into the Crave build workspace before retrying.

Do not commit proprietary blobs, firmware images, signing keys, or credentials to this public repository. The script does not try to fabricate missing vendor files.

## What build.sh does

1. Locates the existing PixelOS source root and fails if no initialized .repo tree exists.
2. Writes a local manifest for the zephyr device tree, 5.10.260 prebuilt kernel, and required open-source dependencies.
3. Runs targeted incremental repo sync only for the five device/dependency paths, without --force-sync or --clean.
4. Checks for the proprietary vendor makefiles immediately after sync and exits before expensive work if they are missing.
5. Creates a PixelOS custom_zephyr product definition from the existing device configuration.
6. Uses small GitHub .patch downloads for optional compatibility changes instead of fetching entire framework commit history. The GED-KPI Surface patch is intentionally skipped because it conflicts with the Android 16 QPR2 Surface API and needs a proper port.
7. Clones the Android 16 QPR2 MediaTek IMS project, then runs lunch custom_zephyr-bp4a-userdebug and m pixelos.
8. Lists zip/image artifacts in out/target/product/zephyr when the build completes.

## Verification status

GitHub Actions (`.github/workflows/validate.yml`) checks Bash syntax, ShellCheck, the embedded manifest branches/paths, and safe refusal when no Crave source checkout exists. These are shallow checks only: the workflow does not sync Android sources or build a ROM. An actual ROM build has not been run from this repository: that requires a live Crave build job and the compatible proprietary vendor files.
