# 1. Hardware and kernel: what the flash driver can do

No kernel patches. Everything here is about the stock `kernel/lge/bullhead` (3.10) and what it
already supports.

## How the flash is wired

The rear flash on the Nexus 5X is two LEDs driven by the **flash block of the PMI8994 PMIC**
(SPMI peripheral `0xd300`). The driver is `drivers/leds/leds-qpnp-flash.c`, and the device tree node
is in `arch/arm/boot/dts/qcom/msm-pmi8994.dtsi` (`qcom,leds@d300`):

| Node | LED class device | `qcom,current` | `qcom,max-current` |
|---|---|---|---|
| `pmi8994_flash0` / `flash1` | `led:flash_0`, `led:flash_1` | 1000 mA | 1000 mA |
| `pmi8994_torch0` / `torch1` | `led:torch_0`, `led:torch_1` | 120 mA | **200 mA** |
| `pmi8994_switch` | `led:switch` | 625 mA | 1000 mA |

The flash and torch nodes don't drive the LEDs directly. Writing a brightness to `torch_0` or
`torch_1` only stores a current. Writing `1` to `led:switch` is what actually switches the LEDs on
with those currents.

```
/sys/class/leds/led:torch_0/brightness   current for LED 0, in mA (0..200)
/sys/class/leds/led:torch_1/brightness   current for LED 1, in mA (0..200)
/sys/class/leds/led:switch/brightness    0 = off, non-zero = strobe the stored currents
```

So **the driver already supports brightness**: in `qpnp_flash_led_brightness_set()` the brightness
*is* the current in mA, clamped to `max-current` and raised to at least `FLASH_LED_MIN_CURRENT_MA`
(13 mA).

### 16 hardware steps

The torch current register only has 4 bits. `qpnp_flash_led_work()` converts the mA value like
this:

```c
val = (u8)(prgm_current * FLASH_TORCH_MAX_LEVEL / max_current);   /* FLASH_TORCH_MAX_LEVEL = 0x0F */
```

So with `max_current = 200` there are 16 register values (`0x0`..`0xF`), roughly 12.5 mA apart.
Anything between two steps is rounded down.

## How the camera stack reaches it

Android doesn't write to sysfs for the torch. The camera HAL opens the MSM flash V4L2 subdevice and
sends `VIDIOC_MSM_FLASH_CFG` ioctls to `drivers/media/platform/msm/camera_v2/sensor/flash/msm_flash.c`.
`msm_flash` is wired to the PMIC nodes via `qcom,camera-flash` in
`arch/arm64/boot/dts/lge/msm8992-bullhead-camera.dtsi` and drives them through **LED triggers**
(`torch0_trigger`, `torch1_trigger`, `switch_trigger`):

```c
/* msm_flash_low(), the torch path */
for each torch source i:
    max_current = torch_max_current[i];            /* 200, from the DT */
    if (flash_current[i] >= 0 && flash_current[i] < max_current)
        curr = flash_current[i];
    else
        curr = torch_op_current[i];                /* LED_HALF = 127 */
    led_trigger_event(torch_trigger[i], curr);
led_trigger_event(switch_trigger, 1);
```

The current comes from the ioctl, so the camera HAL can choose any torch brightness.

## Finding 1: the torch was never at full strength

The bullhead camera HAL (`device/lge/bullhead/camera/QCamera2/util/QCameraFlash.cpp`) always sends:

```c
#define QCAMERA_TORCH_CURRENT_VALUE 200
cfg.flash_current[i] = QCAMERA_TORCH_CURRENT_VALUE;
```

`msm_flash_low()` only accepts values **below** `max_current` (`<`, not `<=`). 200 is rejected and
replaced by `LED_HALF`, so **the stock torch has always run at 127 mA per LED**, not 200 mA.

The highest current you can reach through `msm_flash` without a kernel change is therefore 199 mA,
which lands on register value `0xE` (187 mA). Step `0xF` is unreachable from userspace.

## Finding 2: the PMIC only latches the current when the strobe turns on

The first version changed the current while the torch was already on by sending another
`CFG_FLASH_LOW` with a new value. The driver did write the new value to the registers, but the
light didn't change until the torch was switched off and on again.

This is visible in the PMIC registers. They can be read through SPMI debugfs. The PMI8994 is
slave ID 3, so the flash block starts at `0x3d300`:

```bash
adb root
adb shell 'cd /sys/kernel/debug/spmi/spmi-0
           echo 0x3d342 > address; echo 6 > count; cat data'
# 3D340 -- -- 0E 0E 0F 00 80 C0
#              |  |  |        `-- 0x47 STROBE_CTRL  (C0 = both LEDs strobing)
#              |  |  `---------- 0x44 MAX_CURRENT
#              `--`------------- 0x42/0x43 LED0/LED1 current
```

Changing the slider while the torch was on:

| Step | LED0/LED1 current regs | Strobe | Light |
|---|---|---|---|
| on at level 15 | `0E 0E` | `C0` | bright |
| live change to level 1 | `00 00` | `C0` | **still bright** |
| live change to level 8 | `07 07` | `C0` | **still bright** |
| turn off | `07 07` | `00` | off |

The current registers follow the slider, but the strobe register never changes, and the PMIC keeps
driving the current it latched at the last strobe-on edge. The fix (in the HAL, see
[02](02-camera-hal-and-sepolicy.md)) is to strobe off and on again when the strength changes.

## Finding 3: back-to-back off/on gets merged by the workqueue

Strobing off and on worked only some of the time. The reason is in
`qpnp_flash_led_brightness_set()`: it doesn't touch the hardware, it stores the value in
`cdev.brightness` and queues `flash_node->work` on an ordered workqueue. `qpnp_flash_led_work()`
later reads whatever `cdev.brightness` is **at the time it runs**.

```
CFG_FLASH_OFF  -> switch brightness = 0, queue_work(switch.work)     work pending
CFG_FLASH_LOW  -> switch brightness = 1, queue_work(switch.work)     already pending: no-op
work runs      -> reads brightness = 1 -> "on" path only, no off, no strobe edge
```

If the second ioctl arrives before the work item has run, the off is lost. The turn-off path itself
takes a little over `FLASH_RAMP_DN_DELAY_US` (2.16 ms), so the HAL now waits 10 ms between the off
and the on. That gives a short dark gap on each change, which is hard to notice.

A cleaner fix would be a kernel patch that toggles only the strobe bit when the current changes
while the torch is on. That was left out on purpose to keep the kernel untouched.

## Summary

| Question | Answer |
|---|---|
| Does the hardware support torch brightness? | Yes, 16 steps of ~12.5 mA per LED |
| Does the driver expose it? | Yes, the LED brightness is the current in mA |
| Can the camera HAL set it? | Yes, `flash_current[]` in `VIDIOC_MSM_FLASH_CFG` / `CFG_FLASH_LOW` |
| Usable range from userspace | 13..199 mA → register `0x0`..`0xE`, 15 steps |
| Stock torch current | 127 mA (`LED_HALF`), because 200 mA is rejected |
| Live changes | Need a strobe off→on edge, with time for the off to run |
