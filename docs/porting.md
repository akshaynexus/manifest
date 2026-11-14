# Porting a MediaTek phone

How a MediaTek phone goes from `mapple_device_setup_scripts`' brief to a booting port, in the order the realme 8i (`spaced`,
MT6781) needed it. It is one phone's path: every number below is spaced's and says so, none carries to another phone unread.
Expect a step this page does not know.

The method is HakiOS's: its `docs/mtk-porting.md` (the Infinix Hot 20) and `docs/from-a-clone.md` are the model for the order
of work, the run cycle and the list of what to read off the phone first. Open Mapple differs in its loader, shim and device-tree
builder. Read HakiOS's pages as well.

## What you need

- an unlocked bootloader, root in Android, and a way back that does not depend on the partition you write: MediaTek's boot ROM
  mode with mtkclient (spaced: fastboot does not work; on macOS mtkclient needs `sudo` and FUSE-T)
- your own backup of `boot`, `lk` and the partition table, before anything is flashed
- the phone's open Android trees (device and kernel) and its own tree and GPT, read from the running phone
- a port writes two things: `boot` (the loader chained in front of Android's kernel) and one partition of its own, `xnusys`,
  carved from `userdata`. The carve is done on files first and written only when you say so. Never flash an LK you have not verified
- nothing of Apple's, of the vendor's (no firmware, no image, no device tree off the phone) or personal (serials, Wi-Fi names)
  goes into a repository; the firmware is taken off the user's own phone by the device's `pull-firmware.sh`

## 1. Run the script

```sh
adb shell su -c 'cat /sys/firmware/fdt' > fdt.dtb              # then mask serials
adb shell su -c 'dd if=/dev/block/sdX bs=4096 count=34' > gpt.bin   # the LUN that holds the GPT; mask the GUIDs
python3 bringup.py <name> --android-device DIR --android-kernel DIR --fdt fdt.dtb --gpt gpt.bin --ours <our trees> --out out
```

Read `out/BRIEF.md` first: facts with where each was read, gaps, and the read-only commands that settle what the sources cannot
(registers as LK leaves them, rails, the run-time memory pools, the real part numbers). Run those on the phone, keep the answers
in your own notes (outside the repositories). `out/report.md` says which of our drivers already name each thing and which
nothing names: those are the drivers to write. The skeleton `device.conf` has every status `missing`; a status changes only
when a run shows it.

Check the brief against the phone before you trust it. spaced's trees were wrong about three things: the big cores (the dts says
A75, the part is an A76), the PMIC (the defconfig and dts say MT6358, the part is an MT6366 with the MT6358's register layout)
and the torch (the tree names an MT6370, the phone has an MT6371).

## 2. The order of work

One boot-arg per hardware step; nothing runs without its argument. A run is expensive: put several steps in one run, each behind
its own argument, so one failure is still pinned to one. Isolate a risky step (a new voltage, clock, modem or SCP power, a storage
write) in a boot of its own with a record line. On a failure, halve the new arguments in the next run.

| step | do | done when | spaced |
| --- | --- | --- | --- |
| 1 | the loader chained in front of Android's kernel in `boot`; its log goes to ramoops (address from the live tree) | the loader's lines are in `console-ramoops-0` after the reboot | ramoops at `0x4d010000`, 896 KiB |
| 2 | the kernel's first lines: the ADT from `adt-input.conf`, RAM from the largest free range, the cores | each core checks in | 6x A55 and 2x A76 up; GIC, EINT, keypad registered |
| 3 | storage read only: the GPT and the root's first block through the controller as LK leaves it | the partition reads back | UFS LUN 2, 4096-byte blocks; LK leaves the controller ready, no re-init |
| 4 | carve `xnusys` in files, write it, put the root in it, mount | launchd runs | the iOS side needed `unentitled_ios_sim_launch=1 ignition_level=1 launchd_unsecure_cache=1`, then a read-write root (`mp_ufsrw=1 mp_rootrw=1`) |
| 5 | the display: LK's framebuffer, drawn by the CPU | the home screen is on the panel | 1080x2412 at `0x771b0000`, pitch 4320, video-mode DSI |
| 6 | keys, then touch | taps and swipes arrive | volume down on the keypad, volume up and power on the PMIC; the touch chip (ILI7807S) has no flash: the loader sends its firmware |
| 7 | all of DRAM except the holes | the kernel sees the whole of it | 2 GB at first, then 4 GB (see below) |
| 8 | PMIC, thermal, clocks: read only, then one step at a time | readings agree with Android's | LVTS read with the efuse's calibration, hardware reset armed at 117 C and tested, CPU DVFS through SSPM in four steps |
| 9 | the GPU at LK's rails and clock | the UI draws through it | Mali-G57 at 300 MHz, LK's rails (`mp_gpu_lkrails=1`) |
| 10 | Wi-Fi/Bluetooth (CONNSYS), audio, torch, haptic, camera, sensors, cellular | each part's own scan, join, sound, light | Wi-Fi and the flash LED work, Bluetooth and audio in part; the rest per `status` lines |
| 11 | write it down | `device.conf` says what a run showed | |

### Memory beyond the first range (step 7)

- The kernel is given the free memory as the script lists it: every fixed reservation of LK's tree (no-map or not), the payload
  and LK's tree itself are holes, in 16 KiB pages, merged. spaced: 13 holes, 3573 MB left for iOS.
- Everything in a hole is untouchable, not only unmapped. After a failed boot with RAM from DRAM's base, Android's dmesg showed a
  latched EMI MPU violation inside ATF's reservation (cause not pinned: the early mapping covers the holes); read dmesg after
  the first boot with holes.
- The kernel's page numbering is by index from the first page, so a hole is walked through and its pages kept off the free list,
  not skipped. Anything the loader mapped for itself (spaced: the touch firmware's table) must stay out of the free list too.
- The pools Android places at run time (Wi-Fi, CONNSYS, SCP, ssmr) are not in LK's tree: take them from `/proc/iomem`
  (the brief lists the probe). spaced's Wi-Fi window is at `0xbf000000`.
- A DMA window inside a hole is mapped cacheable, zeroed with aligned stores and flushed through its own mapping; the by-physical
  flush route misses holes and faults.

## 3. What stopped spaced

| seen | cause |
| --- | --- |
| panic at launchd's exec, wrong platform | an iOS binary on the kernel's macOS platform: the three launch arguments above |
| `mount` of `/` refused, read-only file system | root mounted read only: the read-write arguments |
| panic, a page "found" mapped when freed | the loader's own table was in the free list |
| panic in `vm_page_init` with holes | page numbers are by index; holes must be walked, not skipped |
| unaligned data abort in a reserved window | device-type mapping of a hole; use a cacheable mapping and aligned stores |
| data abort in a flush | flush by physical address skips holes |
| DVFS refused | a check misread the PMIC: its SRAM rail tracks the processor rail in hardware (+100 mV on spaced) |
| link lost seconds into a thermal trip test | the hardware reset, lowered to 55 C for the test, fired (cold reset, thermal bit set) |
| fastboot entered from a reboot | the volume key held through `adb reboot`: press it after the logo |

## 4. Writing it down

`device.conf` states: `works` only for what you saw work, `partial` with a note of what is not, `missing` otherwise. The
README says how each part is switched on. Publish the device as a repository of its own and ask for it to be added to the
manifest.
