# /e/OS (Android 16) for the Moto G8 Plus (amogus_doha)

This repo has everything needed to rebuild our unofficial **/e/OS 4.4
(Android 16)** ROM for the **Motorola Moto G8 Plus** (`doha`, XT2019-1/2,
Snapdragon 665). In the source tree the device is called **`amogus_doha`**.

/e/ doesn't support this phone officially. This build isn't made or
supported by the /e/ foundation, so please report problems here, not
to them.

It's the sibling of our LineageOS 23.2 ROM,
[`christiaan/lineage-23-amogus_doha`](https://github.com/christiaan/lineage-23-amogus_doha).
Both use the same device side: the community doha port by Aldair402 and
moto-common (device and vendor trees, blobs), and our 4.14.357 LTS kernel
([`christiaan/android_kernel_motorola_msm-4.14`](https://github.com/christiaan/android_kernel_motorola_msm-4.14),
branch `lts`). What differs is the base: /e/'s `v4.4-a16` branch, which
is LineageOS 23.0 (AOSP `android-16.0.0_r1`) with /e/'s changes and
their own monthly security merges. Their apps come with it: microG,
App Lounge, Advanced Privacy, Bliss Launcher, Open Camera, the /e/
Updater, /e/'s backup apps, plus Seedvault.

| Path | What it is |
| --- | --- |
| `default.xml` | A `repo` manifest that pins all 1156 projects to the exact commits the released build used: /e/'s `v4.4-a16` manifest, plus the doha projects from the LineageOS repo |
| `patches/` | Our fixes. `patches/<project path>.patch` applies to that project |
| `build.sh` | Syncs the sources, applies the patches and builds the zip |

## Requirements

- **x86-64 Linux** with Docker. The build runs inside the
  [`lineageos4microg/docker-lineage-cicd`](https://github.com/lineageos4microg/docker-lineage-cicd)
  image, which has all the build tools and `repo` installed. To build
  without Docker, install the
  [LineageOS build dependencies](https://wiki.lineageos.org/emulator#install-the-build-packages),
  `repo` and `git-lfs` first.
- **About 350 GB of free disk**: roughly 210 GB for the synced sources
  (including 3.4 GB of /e/ APKs in git LFS) and 130 GB for `out/`.
  ccache adds up to 30 GB more if you enable it. On btrfs (or XFS) you can
  start from a LineageOS 23.x checkout with `cp -a --reflink=always`; see
  the LineageOS repo's README for the steps and the pitfalls. Expect the
  sync to still download about 100 GB, because many /e/ projects differ
  from Lineage's.
- **32 GB RAM plus at least 32 GB swap.**
- Time. A clean build (empty ccache) took 2 h 25 min on 16 threads with
  30 GB RAM. With a warm ccache, a rebuild after a patch change takes
  8-15 minutes.

## Building

```sh
git clone https://github.com/christiaan/eos-a16-amogus_doha.git doha-eos
mkdir -p ~/doha/eos-src ~/doha/ccache-eos ~/doha/zips

docker run --rm -it \
  --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v "$PWD/doha-eos:/manifest:ro" \
  -v ~/doha/eos-src:/src \
  -v ~/doha/ccache-eos:/ccache -e CCACHE_DIR=/ccache -e CCACHE_SIZE=30G \
  -v ~/doha/zips:/zips -e ZIP_DIR=/zips \
  --entrypoint /manifest/build.sh \
  lineageos4microg/docker-lineage-cicd:latest /src
```

Run the container as your own user, as above. A tree synced as root
makes git refuse `/manifest` and the projects as "dubious ownership"
later. `/manifest` must be a real clone, not a `git worktree`.

`build.sh` works as described in the LineageOS repo's README: it syncs
the pinned manifest, applies `patches/`, and runs
`breakfast amogus_doha userdebug` and `mka target-files-package bacon`.
The options (`SKIP_SYNC=1`, `JOBS`, `SYNC_JOBS`, `CCACHE_SIZE`,
`ZIP_DIR`) are the same. Because `vendor/eos` exists in this tree, the
output is named the way /e/ names it:
`~/doha/zips/e-4.4.1-a16-<date>-UNOFFICIAL-amogus_doha.zip`, next to
`e-4.4.1-a16-<date>-UNOFFICIAL-amogus_doha-{boot,dtbo,vbmeta}.img` and a
`.sha256sum`.

The build is `userdebug` and signed with the AOSP test keys.

## Installing

You need an unlocked bootloader. The phone is A/B, and the recovery is
inside `boot`.

**Coming from LineageOS (our releases included) or any other ROM, you
have to factory reset.** Seedvault backups made on our LineageOS 23.2
can be restored on this build.

1. `fastboot flash boot e-4.4.1-a16-<date>-UNOFFICIAL-amogus_doha-boot.img`.
2. Boot into recovery, do a factory reset, then choose
   **Apply update → Apply from ADB** and run
   `adb sideload e-4.4.1-a16-<date>-UNOFFICIAL-amogus_doha.zip`.
3. When the install finishes, recovery asks whether you want to install
   an additional zip. Choose **No**, then **Reboot system now**. The
   first boot takes about 4 minutes.

To update a phone that already runs this ROM, turn on USB debugging,
authorize the computer, and run

```sh
adb reboot sideload-auto-reboot
adb wait-for-sideload
adb sideload e-4.4.1-a16-<date>-UNOFFICIAL-amogus_doha.zip
```

The notes in the LineageOS README apply here too. Recovery installs to
the inactive slot, the sideload slows down at around 47% (tapping the
screen speeds it up), and flashing `vbmeta` prints a harmless
`anti rollback downgrade` warning.

## Status

Tested on a Himax/FPC unit (XT2019-1): boot, SIM and LTE, calls, Wi-Fi,
touch, both cameras, speaker, vibration and reboot, with SELinux
enforcing. microG's self-check passes, including signature spoofing.
App Lounge, Advanced Privacy and the Updater open. Apps install through
Aurora Store.

Not tested yet: SMS, NFC, App Lounge with a normal (not supervised)
Google account, and the other hardware variants (FocalTech or Novatek
touch, Egistec fingerprint, XT2019-2 dual-SIM).

Known issues:
- App Lounge's anonymous login is refused by /e/'s token server
  (`403 Not allowed`). Other unofficial /e/ builds report the same.
- Supervised (Family Link) Google accounts don't work with microG.
- The /e/ Updater finds no updates; new builds are posted as releases here.

## The patches

Most patches are the same as in the LineageOS repo, and its README
explains each one. These differ.

Only in this repo, backports of fixes that LineageOS 23.2 already has:

| Patch | Source | Why |
| --- | --- | --- |
| `frameworks/opt/telephony` | AOSP `44e3dbf3d3` | 23.0 creates one UICC slot while doha's modem runs dual-SIM (`dsds`), so `com.android.phone` crash-loops and there's no SIM |
| `frameworks/native` | LineageOS `d8d3541220` | gpuservice aborts when the gpuWork BPF maps are missing, which they are on the 4.14 kernel, and crash-loops |

Changed for 23.0:

| Patch | Change | Why |
| --- | --- | --- |
| `build/soong` | backport of AOSP `8461cd858` (`prebuilt_lib`) and 23.2's fsgen mapping of `lib/` to it | 23.0's fsgen installs `PRODUCT_COPY_FILES` from `lib/` into `lib64`, giving 126 packaging conflicts |
| `device/lineage/sepolicy` | same `sched_energy_aware` removal, new context | the line sits elsewhere in 23.0's file |
| `packages/modules/Connectivity` | same BPF kernel-gate relaxation, rewritten | 23.0's gates have a different form |
| `device/motorola/common` | no `powerhal_flags-aconfig-cc` link | 23.0's libperfmgr predates the power HAL flags |
| `device/motorola/common` | doesn't request the platform `NfcNci` | on 23.0 it only builds inside the NFC APEX, which is installed anyway |

`system/core`, `device/lineage/sepolicy` and `packages/modules/Connectivity`
are /e/'s forks; the patches apply on top of them.
