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

From `/crave-devspaces/PixelOS`, first run a **vendor preflight** that does not start compiling:

~~~bash
crave run --projectID 82 --no-patch --detached -- "curl -fLSs --retry 3 https://raw.githubusercontent.com/asxcaz/pos16-zephyr/2eaece0f93e02664a1bcd3819c3196cce8cadf62/build.sh -o /tmp/pos16-zephyr-build.sh && POS16_CHECK_ONLY=1 bash /tmp/pos16-zephyr-build.sh"
~~~

This syncs only the five device/dependency paths, verifies the device tree and prebuilt kernel files, then clones the documented Vendor tree from GitLab into `vendor/oplus/zephyr` if that directory does not already exist. By default it uses the GitLab project's default branch; set `POS16_VENDOR_BRANCH` to pin a specific branch. If the directory already exists but lacks either required makefile, the script stops rather than overwriting it. With `POS16_CHECK_ONLY=1`, it reports the Vendor file count and exits **before** changing product files, cloning IMS, applying patches, or compiling. The check verifies the two required makefiles, not every referenced proprietary blob.

If that check passes, submit the actual build:

~~~bash
crave run --projectID 82 --no-patch --detached -- "curl -fLSs --retry 3 https://raw.githubusercontent.com/asxcaz/pos16-zephyr/2eaece0f93e02664a1bcd3819c3196cce8cadf62/build.sh -o /tmp/pos16-zephyr-build.sh && bash /tmp/pos16-zephyr-build.sh"
~~~

Both commands deliberately avoid `--clean`, `repo init`, and full PixelOS source initialization.

## Source branches used

- PixelOS base: sixteen-qpr2 (already provisioned by the Crave PixelOS project)
- Device tree: oplus-mt6895/android_device_oplus_zephyr, lineage-23.2
- Prebuilt kernel: V1NAY007/android_device_oplus_zephyr-kernel, upstream-test-5.10.260
- MediaTek hardware: oplus-mt6895/android_hardware_mediatek, lineage-23.2
- OPlus hardware: oplus-mt6895/android_hardware_oplus, lineage-23.2
- MediaTek vendor sepolicy: oplus-mt6895/android_device_mediatek_sepolicy_vndr, lineage-23.2
- Vendor tree: [NINJAHUNTER1357/proprietary_vendor_oplus_zephyr](https://gitlab.com/NINJAHUNTER1357/proprietary_vendor_oplus_zephyr), GitLab default branch unless `POS16_VENDOR_BRANCH` is set

The device tree BoardConfig uses device/oplus/zephyr-kernel as a prebuilt kernel and sets TARGET_FORCE_PREBUILT_KERNEL. Therefore this script uses the supplied 5.10.260 Image.gz and MT6895 DTB; it does not build android_kernel_oplus_mt6895 from source. If changing to a source-built kernel later, use that kernel repository's lineage-24.0 branch, not the deprecated 23.2 branch.

## Proprietary vendor tree

The device tree requires these files:

- `vendor/oplus/zephyr/zephyr-vendor.mk`
- `vendor/oplus/zephyr/BoardConfigVendor.mk`

The organization README lists the Vendor tree at [NINJAHUNTER1357/proprietary_vendor_oplus_zephyr](https://gitlab.com/NINJAHUNTER1357/proprietary_vendor_oplus_zephyr). The build script now clones that repository into `vendor/oplus/zephyr` automatically when the directory is absent. It uses the GitLab project's default branch unless `POS16_VENDOR_BRANCH` is set explicitly.

If `vendor/oplus/zephyr` already exists and contains both required makefiles, the script reuses it. If the directory exists but either file is missing, the script stops and refuses to overwrite it; repair or move that directory before retrying.

Do not commit proprietary blobs, firmware images, signing keys, or credentials to this public build-helper repository. The Vendor tree is fetched into the build workspace, not copied into this repository.

## What build.sh does

1. Locates the existing PixelOS source root and fails if no initialized .repo tree exists.
2. Writes a local manifest for the zephyr device tree, 5.10.260 prebuilt kernel, and required open-source dependencies.
3. Runs targeted incremental repo sync only for the five device/dependency paths, without --force-sync or --clean.
4. Fetches the documented Vendor tree from GitLab when needed and checks the two required makefiles before doing any expensive work.
5. Creates a PixelOS custom_zephyr product definition from the existing device configuration.
6. Uses small GitHub .patch downloads for optional compatibility changes instead of fetching entire framework commit history. The GED-KPI Surface patch is intentionally skipped because it conflicts with the Android 16 QPR2 Surface API and needs a proper port.
7. Clones the Android 16 QPR2 MediaTek IMS project, then runs lunch custom_zephyr-bp4a-userdebug and m pixelos.
8. Lists zip/image artifacts in out/target/product/zephyr when the build completes.

## Verification status

GitHub Actions (`.github/workflows/validate.yml`) checks Bash syntax, ShellCheck, the embedded manifest branches/paths, and safe refusal when no Crave source checkout exists. These are shallow checks only: the workflow does not sync Android sources or build a ROM. An actual ROM build has not been run from this repository: that requires a live Crave build job and the compatible proprietary vendor files.
