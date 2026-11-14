# Open Mapple

Apple's open-source XNU kernel and an iOS user space on Android phones, built like an Android custom ROM:
a repo manifest, a tree per device, one build command, a zip for your recovery. A research project, not
affiliated with Apple, MediaTek, realme or any phone maker.

**Status: the repositories are being set up. Nothing is released and the build commands below don't work
yet.**

## Devices

| Device | Codename | SoC | State |
| --- | --- | --- | --- |
| realme 8i | `spaced` | MediaTek Helio G96 (MT6781), Mali-G57 MC2 | bring-up |

## Getting the source

```sh
mkdir open-mapple && cd open-mapple
repo init -u https://github.com/open-mapple/manifest -b main
repo sync
```

Clone over SSH (`git@github.com:open-mapple/manifest`) or HTTPS with a token; the manifest names the
other repositories relative to itself, so they come the same way. The optional dual boot is not in the
default checkout: `repo init ... -g default,splitboot`.

## Building

```sh
. build/envsetup.sh
lunch mapple_spaced
mka mapple
```

You need macOS with Xcode (for the kernel; Linux builds are a later goal), python3 and 7-Zip.

**Running the build downloads iOS 16.7.16 for the iPhone X from Apple's servers. Running `mka` is your
consent to that download:** it prints one line saying so and does not ask. The download is checked against a
pinned size and SHA-256. To use your own copy instead: `mka mapple IPSW=/path/to/the.ipsw`.

The result, `out/target/product/spaced/mapple-*-PERSONAL.zip`, contains iOS. It is for you and your phone
only: don't share it. Open Mapple's releases are source only.

## Installing

Flash the zip in TWRP or OrangeFox. By default iOS replaces Android: the zip writes `boot` and formats the
whole of `userdata` (your Android data is gone), after copying the touch and Wi-Fi firmware off the phone.
Back up first. For Android and iOS side by side, flash `mapple_splitboot`'s zip before it.

## Unbricking (realme 8i)

Fastboot doesn't work on this phone; MediaTek's boot ROM mode does. Keep your own backup of `boot`, `lk` and
the partition table before you flash anything.

- With mtkclient: power the phone off, hold both volume keys, plug it in, then write your backup back, for
  example `python mtk.py w boot boot.img`. On macOS mtkclient needs `sudo` and FUSE-T.
- If the phone hangs in LK: hold Power and both volume keys for 30-60 s with mtkclient waiting.
- After mtkclient, hold Power for 10-15 s to boot. To get Android back entirely, flash the stock ROM.
- Never flash an LK you haven't verified.

## Repositories

| Path | Repository | What |
| --- | --- | --- |
| `build/mapple` | mapple_build | envsetup, lunch, mka; the IPSW download; image and zip packaging |
| `vendor/mapple` | mapple_vendor | shared configuration, release tooling, credits |
| `kernel/mediatek/common` | xnu_kernel_mediatek_common | XNU for MediaTek phones |
| `kernel/mediatek/mt6781` | xnu_kernel_mediatek_mt6781 | the MT6781's layer of the kernel |
| `device/realme/spaced` | xnu_device_realme_spaced | the realme 8i |
| `mapple/loader` | mapple_loader | the boot loaders |
| `mapple/userland` | mapple_userland | programs on the phone |
| `mapple/gpu/mali` | mapple_gpu_mali | the Metal driver for Mali GPUs |
| `mapple/splitboot` | mapple_splitboot | the optional dual boot |
| `external/mesa` | Mesa | Panfrost and the Valhall compiler |

## Porting another phone

[docs/porting.md](docs/porting.md): from a MediaTek phone's open Android trees to a booting port, with `mapple_device_setup_scripts`.

## Credits

Open Mapple is built on Apple's XNU (APSL 2.0), HakiOS (oosexgang), ios-sweet (dipa-studio), ios-viva and
viva-gpu (Nicola Leone Ciardi), darwin-xnu-build (blacktop), BearSSL and Mesa. Every commit taken from another
project says where it came from; `vendor/mapple/CREDITS` and each repository's `NOTICE` give the licences.

## Licence

Each repository has its own `LICENSE` and `NOTICE`; this one is MIT.
