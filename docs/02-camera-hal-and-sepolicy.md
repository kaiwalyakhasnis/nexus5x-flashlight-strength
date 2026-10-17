# 2. Camera HAL and SELinux

Patches: [`patches/device_lge_bullhead/`](../patches/device_lge_bullhead/) (0001, 0002, 0003)

## Is there an API for this already?

Not on Android 8.1.

| | Torch brightness API |
|---|---|
| Android 8.1 SDK (API 27) | No. `CameraManager.setTorchMode(id, boolean)` is on/off only, and so is the camera HAL's `setTorchMode`. |
| Lineage SDK 15.1 | No. Its only torch settings are `TORCH_LONG_PRESS_POWER_GESTURE` and `TORCH_LONG_PRESS_POWER_TIMEOUT`. |
| Android 13+ (API 33) | Yes: `CameraManager.turnOnTorchWithStrengthLevel()`, `getTorchStrengthLevel()`, `CameraCharacteristics.FLASH_INFO_STRENGTH_MAXIMUM_LEVEL` / `_DEFAULT_LEVEL`, backed by a matching camera HAL call. The Quick Settings slider in newer Android uses this. |

Backporting the Android 13 API would touch the public SDK, `CameraService`, the camera provider
HIDL interface and the HAL. This project uses a much smaller route: a **system property** that the
camera HAL reads, set from SystemUI.

## The torch call path on bullhead

```
SystemUI FlashlightControllerImpl
  CameraManager.setTorchMode("0", true)                       Binder
    CameraService::setTorchMode                               cameraserver
      CameraFlashlight / ProviderFlashControl
        CameraProviderManager -> legacy/0 provider            passthrough, same process
          QCamera2Factory::setTorchMode                       camera.msm8992.so
            QCameraFlash::initFlash / setFlashMode
              ioctl(VIDIOC_MSM_FLASH_CFG, CFG_FLASH_LOW, flash_current[])
                msm_flash_low  ->  LED triggers  ->  leds-qpnp-flash  ->  PMI8994
```

The device has no `android.hardware.camera.provider@2.4-service`, so the provider is loaded
**passthrough inside `cameraserver`**. That matters for SELinux below.

## 0001: read the torch current from a property

`QCameraFlash::setFlashMode()` now calls `getTorchCurrent()`:

```c
#define QCAMERA_TORCH_CURRENT_PROP    "persist.vendor.torch.current"
#define QCAMERA_TORCH_CURRENT_MIN     13                               /* driver minimum */
#define QCAMERA_TORCH_CURRENT_MAX     (QCAMERA_TORCH_CURRENT_VALUE - 1) /* 199, msm_flash uses < */
#define QCAMERA_TORCH_CURRENT_DEFAULT 127                              /* what the stock torch got */
```

The default keeps the old brightness until someone picks a level. The property is `persist.`, so
the choice survives reboots.

### Re-applying while the torch is on

Before, `setFlashMode(id, true)` on a torch that was already on returned `-EALREADY` without doing
anything. Now it re-applies the current:

```c
if (mode && m_flashOn[camera_id]) {
    cfg.cfg_type = CFG_FLASH_OFF;                 /* strobe edge, see docs/01 finding 2 */
    ioctl(fd, VIDIOC_MSM_FLASH_CFG, &cfg);
    usleep(QCAMERA_TORCH_RESTROBE_DELAY_US);      /* 10 ms, see docs/01 finding 3 */
    cfg.cfg_type = CFG_FLASH_LOW;
}
ioctl(fd, VIDIOC_MSM_FLASH_CFG, &cfg);
...
retVal = -EALREADY;                               /* when it was already on */
```

It still returns `-EALREADY` in that case. `QCamera2Factory::setTorchMode` already treats
`-EALREADY` as success **without** sending a torch status callback, so the framework never sees
the internal off/on, and the Quick Settings tile doesn't flicker.

The SystemUI side only needs to call `CameraManager.setTorchMode(id, true)` again. `CameraService`
doesn't filter repeated "on" calls; it passes them down to the HAL.

## 0002: SELinux

```
# property.te
type torch_prop, property_type;

# property_contexts
persist.vendor.torch. u:object_r:torch_prop:s0

# platform_app.te: SystemUI runs as platform_app
set_prop(platform_app, torch_prop)

# hal_camera.te
get_prop(hal_camera, torch_prop)
```

`cameraserver` gets the `hal_camera` attribute through `hal_client_domain(cameraserver, hal_camera)`
on a device that isn't full Treble, which is why the existing bullhead `hal_camera.te` rules (perfd
socket and so on) apply to the passthrough HAL. Adding the rule there covers it.

A new type is needed because `default_prop` can't be set by apps (`neverallow { domain -init }
default_prop:property_service set`), and `persist.sys.*` (`system_prop`) is settable by
`system_app` but not by SystemUI.

On bullhead (Android 8.1) these files end up in the **boot image ramdisk**
(`/nonplat_property_contexts`, `/sepolicy`). If you test the new SystemUI against an old boot image,
`SystemProperties.set()` is denied. The controller catches that, so the dialog still works, but
the brightness doesn't change.

## 0003: the per-level currents (overlay)

The SystemUI dialog works in levels, and the device overlay says what each level means:

```xml
<string name="config_flashlightStrengthProperty">persist.vendor.torch.current</string>
<integer-array name="config_flashlightStrengthValues">
    <item>13</item> <item>14</item> <item>27</item> ... <item>174</item> <item>187</item>
</integer-array>
<integer name="config_flashlightStrengthDefaultLevel">10</integer>
```

Each value is the lowest mA that lands on a new register value (`mA * 15 / 200`):

| Level | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| mA | 13 | 14 | 27 | 40 | 54 | 67 | 80 | 94 | 107 | 120 | 134 | 147 | 160 | 174 | 187 |
| Register | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E |

Level 10 is the step the old 127 mA landed on (`127 * 15 / 200 = 9`).

## Scope

- This only changes **torch mode** (the QS tile, `CameraManager.setTorchMode`, Lineage's
  long-press-power torch, third-party flashlight apps). The camera app's flash during capture goes
  through the mm-camera daemon and isn't affected.
- Both LEDs always get the same current.
