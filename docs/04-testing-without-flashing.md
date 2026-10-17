# 4. Testing on the phone without flashing

Everything was tested on a Nexus 5X running a LineageOS 15.1 userdebug build from the same tree,
**without writing to any partition**. A normal reboot puts the phone back exactly as it was.

## Build only the changed modules

```bash
source build/envsetup.sh
lunch lineage_bullhead-userdebug
m -j6 SystemUI Settings camera.msm8992 bootimage
```

`bootimage` is needed because on bullhead (Android 8.1, not full Treble) the SELinux policy and
`*_property_contexts` are in the boot image ramdisk.

## 1. RAM-boot the new boot image

```bash
adb reboot bootloader
fastboot boot out/target/product/bullhead/boot.img
adb wait-for-device && adb root
adb shell grep torch /nonplat_property_contexts     # persist.vendor.torch. u:object_r:torch_prop:s0
```

## 2. Bind-mount the new SystemUI, Settings and camera HAL over /system

`ro.boot.veritymode` is `enforcing`, so `adb remount` would first need `adb disable-verity`, which
changes the system partition permanently. Bind mounts avoid that:

```bash
OUT=out/target/product/bullhead/system
adb shell mkdir -p /data/local/tmp/flash
adb push $OUT/priv-app/SystemUI $OUT/priv-app/Settings $OUT/lib/hw/camera.msm8992.so /data/local/tmp/flash/
adb shell '
  cd /data/local/tmp/flash
  chmod -R a+rX . ; find . -type f -exec chmod 644 {} \;
  chcon -R u:object_r:system_file:s0 SystemUI Settings camera.msm8992.so
  mount -o bind /data/local/tmp/flash/SystemUI /system/priv-app/SystemUI
  mount -o bind /data/local/tmp/flash/Settings /system/priv-app/Settings
  mount -o bind /data/local/tmp/flash/camera.msm8992.so /system/lib/hw/camera.msm8992.so
  stop; stop cameraserver; start cameraserver; start'
```

Things that bit along the way:

- The APKs are dexpreopted, so the code is in `oat/arm64/*.odex|vdex`, not in the APK. Mount the
  whole app directory, not just the `.apk`.
- The camera HAL is **32-bit only** (`/system/lib/hw`), loaded by `cameraserver`.
- `adb push` replaces a file instead of overwriting it in place, so a bind mount of a *single file*
  keeps pointing at the old inode. To update the HAL: `stop cameraserver`, `umount`, copy, mount
  again, `start cameraserver`. Directory bind mounts don't have this problem.
- The files need the `system_file` label, or `zygote` and `cameraserver` can't load them.
- The `adb` from the build tree is older than the host's. Mixing them keeps restarting the adb
  server, so use one of them, for example `/usr/bin/adb`.

## 3. Check it

```bash
# SystemUI loaded the new controller with the bullhead overlay
adb shell dumpsys activity service com.android.systemui/.SystemUIService | grep -A5 "FlashlightController state"
#   strengthLevel=10/15

# open the dialog the way Settings does (root has every permission)
adb shell am broadcast -a com.android.systemui.action.SHOW_FLASHLIGHT_STRENGTH -p com.android.systemui

# what the driver and the PMIC got
adb shell 'getprop persist.vendor.torch.current
           cat /sys/class/leds/led:torch_0/brightness /sys/class/leds/led:switch/brightness
           cd /sys/kernel/debug/spmi/spmi-0; echo 0x3d342 > address; echo 6 > count; cat data'
```

## Results

| Test | Property | Driver (torch_0 / torch_1) | PMIC current regs | Torch |
|---|---|---|---|---|
| Tap top of the cone | 187 | 187 / 187 | `0E 0E` | on, 15/15 |
| Tap middle | 94 | 94 / 94 | `07 07` | on, 8/15 |
| Tap bottom | 13 | 13 / 13 | `00 00` | on, 1/15 |
| Turn off | 13 (kept) | 0 / 0 | strobe `00` | off |
| 0.4 s drag bottom → top | 187 | | `0E 0E` | on, 15/15 |
| 0.3 s drag top → middle | 94 | | `07 07` | on, 8/15 |
| 6 rapid taps, last = middle | 94 | | `07 07` | on, 8/15 |

- No SELinux denials, no SystemUI or cameraserver crashes.
- `CameraService` logged one `AVAILABLE_ON` per session, so the internal off/on for live changes is
  invisible to the framework.
- Long press on the QS Flashlight tile opens the dialog over the shade (see the demo GIF).

What the registers can't show is the light itself: no sensor on the phone faces the rear flash. The
live-change bugs in [docs/01](01-hardware-and-driver.md) (findings 2 and 3) were found by watching
the LED while the register values already looked right.

## Undo

```bash
adb reboot          # back to the flashed boot image and the original /system files
adb shell rm -rf /data/local/tmp/flash
```
