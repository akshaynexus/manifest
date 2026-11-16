# Open Mapple manifest

The [repo](https://gerrit.googlesource.com/git-repo) manifest for [Open Mapple](https://open-mapple.pages.dev):
it checks out every repository the build needs, at the right place in one tree.

## Get the source

```sh
mkdir open-mapple && cd open-mapple
repo init -u https://github.com/open-mapple/manifest -b main
repo sync
```

The manifest names the other repositories relative to itself, so they come over the same protocol you used
here (HTTPS or SSH).

## Groups

| Group | What it adds | How |
| --- | --- | --- |
| `default` | everything needed to build a phone | always |
| `gpu` | the Metal driver and the pinned Mesa release | part of `default` |
| `splitboot` | the dual boot installer | `repo init ... -g default,splitboot` |

## The tree

| Path | Repository | What it holds |
| --- | --- | --- |
| `build/mapple` | mapple_build | `envsetup`, `lunch`, `mka`, the iOS download, image and zip packaging |
| `vendor/mapple` | mapple_vendor | shared configuration, release tooling, credits |
| `kernel/mediatek/common` | xnu_kernel_mediatek_common | XNU for MediaTek phones |
| `kernel/mediatek/mt6781` | xnu_kernel_mediatek_mt6781 | the MT6781 layer of the kernel |
| `device/realme/spaced` | xnu_device_realme_spaced | the realme 8i |
| `mapple/loader` | mapple_loader | the boot loader, shim and boot menu |
| `mapple/userland` | mapple_userland | programs that run on the phone |
| `mapple/manager` | mapple_manager | Mapple, the app on the phone |
| `mapple/gpu/mali` | mapple_gpu_mali | the Metal driver for Mali GPUs |
| `mapple/splitboot` | mapple_splitboot | SplitBoot, the dual boot (group `splitboot`) |
| `external/mesa` | Mesa | Panfrost and the Valhall compiler, at a pinned tag |

## Build

```sh
. build/envsetup.sh
lunch mapple_spaced_single-userdebug
mka mapple
```

What you need, the products and every target are in the [build guide](https://open-mapple.pages.dev/build).
To install a release instead, see the [install guide](https://open-mapple.pages.dev/install).

## Licence

This repository is MIT. Every other repository has its own `LICENSE` and `NOTICE`; the
[credits page](https://open-mapple.pages.dev/credits) lists the projects Open Mapple builds on.
