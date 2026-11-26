# OpenBeken for BL602 modules with 1 MB flash

Ready-to-flash [OpenBeken](https://github.com/openshwprojects/OpenBK7231T_App) image for BL602 modules that only have
**1 MB of flash**, with a partition table that lets OpenBeken save its configuration.

Tested on a **DOIT DT-BL02** module (BL602 + Macronix 1 MB flash, JEDEC `c22014`) taken from a CozyLife smart switch
(product `k2jxgw`).

## Why this exists

The stock OpenBeken BL602 images target 2 MB flash. The OpenBL602 SDK already builds with
`CONFIG_BL602_USE_1M_FLASH := 1`, so the firmware boots on 1 MB parts, but the SDK's `partition_cfg_1M.toml` only gives
the PSM (config) partition **8 KB / 2 sectors**. OpenBeken's config blob is about 3.6 KB and EasyFlash keeps one sector
free for garbage collection, so saving fails:

```
[EF] Found Valid PSM partition, XIP Addr 230e8000, flash addr 000f9000, size 8192
ENV AREA SIZE 8192, SECTOR NUM 2
Error: Alloc an ENV (size 3619) failed
```

Symptom: the device keeps its `OpenBL602_xxxx` access point after you enter Wi-Fi credentials, and pin settings are
lost on reboot.

This repo uses a custom 1 MB table that takes space from the unused end of the FW partition and gives PSM **32 KB**
(same as the 2 MB layout).

## What about the regular 2 MB release?

The regular `OpenBL602_*.bin` is the same firmware (the 1 MB flash define is already on), so it boots on 1 MB parts,
and people report it working. The catch is the 2 MB partition table: a 1 MB SPI flash ignores the address bit it does
not have, so every address above 1 MB wraps around:

| 2 MB partition | Declared | Lands on a 1 MB flash at | Result |
|---|---|---|---|
| FW | `0x10000`–`0xE7FFF` | same | OK |
| PSM | `0x1E9000`, 32 KB | `0xE9000`–`0xF0FFF` (free space) | OK, which is why the config saves |
| factory | `0x1F8000` | `0xF8000` | OK |
| media (LittleFS) | `0x1A2000`, 284 KB | `0xA2000`–`0xE8FFF`, **on top of the firmware** | creating `autoexec.bat` or any script overwrites firmware code |
| FW OTA slot | `0xE8000`, 544 KB | wraps to `0x00000`–`0x6FFFF`, **over boot2, the partition table and the firmware** | an OTA update bricks the device (UART reflash needed) |

So the 2 MB release is fine if you only use pins and MQTT, but scripts or OTA will break it. The table in this repo
keeps media and PSM outside the firmware and has no OTA slot.

## Partition layout

| Partition | Address | Size | Notes |
|---|---|---|---|
| boot2 | `0x00000` | — | Bouffalo boot2 (`boot2_isp_bl602_v6.4_rc6`) |
| partition table | `0x0E000` / `0x0F000` | 4 KB each | `partition/partition_cfg_1M_obk.toml` |
| FW | `0x10000` | 880 KB (`0xDC000`) | firmware is ~770 KB |
| media | `0xEC000` | 28 KB | LittleFS (`autoexec.bat`, small scripts) |
| PSM | `0xF3000` | 32 KB | OpenBeken config (EasyFlash) |
| factory | `0xFB000` | 16 KB | RF / XTAL params (generic IoTKitA 40 MHz DTS) |

## Files (see Releases)

| File | Use |
|---|---|
| `openbeken-bl602-1mb-14ac3f5-whole-flash.bin` | full image, write at `0x0` (also blanks media + PSM) |
| `openbeken-bl602-1mb-14ac3f5-app.bin` | OpenBeken app only, for tools that build the image themselves |
| `partition_cfg_1M_obk.toml` / `partition-1mb-psm32k.bin` | the custom partition table |
| `boot2_isp_release.bin` | boot2 used for the image |
| `factory-params-40M.dtb` | factory params (40 MHz crystal) |
| `SHA256SUMS` | checksums |

Built from OpenBeken `14ac3f5` (2026-11-25) with OpenBL602 SDK `e0efddc`. The firmware reports version `1.0.0-1MB`.

## Before you flash

1. **Back up the original firmware.** Read the whole 1 MB first (see below). The backup contains your Wi-Fi password
   and the vendor cloud keys, so keep it private.
2. **Check the flash size.** This image is only for 1 MB parts. The flash tool prints the JEDEC ID; the capacity
   byte must be `0x14` (1 MB).
3. **No OTA.** There is no second FW slot in 1 MB, so updates are UART only.

## Wiring (UART download mode)

| Adapter | BL602 module |
|---|---|
| TX | RX0 (GPIO7) |
| RX | TX0 (GPIO16) |
| GND | **GND (required: without it the chip boot-loops and the log is garbage)** |
| 3V3 | 3V3 |
| — | **IO8 (BOOT) tied directly to 3V3** |

Then pulse **EN** to GND. On the DT-BL02 a 10 kΩ pull-up on IO8 was **not** enough (the relay driver on that pin
pulls it down), so tie IO8 straight to 3V3. If the module is still on its switch board, the relay will click while IO8
is high: never flash with mains connected.

The app log is at **2,000,000 baud**. In download mode the module is silent and answers `OK` to a `0x55` handshake.

## Flashing

Tested with the `bflb_iot_tool` from the OpenBL602 SDK (`tools/flash_tool`), which detects the flash and adjusts the
flash config in the boot header:

```sh
./bflb_iot_tool-macos --chipname=bl602 --port=/dev/cu.usbserial-XXXX --baudrate=460800 \
  --pt=partition_cfg_1M_obk.toml \
  --dts=chips/bl602/device_tree/bl_factory_params_IoTKitA_40M.dts \
  --boot2=chips/bl602/builtin_imgs/boot2_isp_bl602_v6.4_rc6/boot2_isp_release.bin \
  --firmware=openbeken-bl602-1mb-14ac3f5-app.bin
```

If you flash over an older layout, also erase `0xEC000`–`0xFAFFF` so the new media/PSM areas start blank (the
whole-flash image already contains `0xFF` there).

Backup / read and erase with Bouffalo's Python tool (`pip install bflb-mcu-tool`; on Python 3.13+ also
`pip install standard-telnetlib`), using its eflash loader (`libs/bflb_eflash_loader.py`):

```sh
# read the whole flash
--chipname bl602 --read --flash --start 0x0 --end 0xFFFFF --file backup-1MB.bin --port <port> --baudrate 460800 --xtal 40m
# erase media + PSM
--chipname bl602 --erase --flash --start 0xEC000 --end 0xFAFFF --port <port> --baudrate 460800 --xtal 40m
```

## First boot

The boot log should show the 32 KB PSM, and the sector-format warnings only on the first boot:

```
[EF] Found Valid PSM partition, XIP Addr 230e2000, flash addr 000f3000, size 32768
ENV AREA SIZE 32768, SECTOR NUM 8
```

Connect to `OpenBL602_xxxxxxxx`, open `http://192.168.4.1`, set Wi-Fi (2.4 GHz only) and continue as with any
OpenBeken device.

## CozyLife DT-BL02 switch pin map

Taken from the original CozyLife firmware (`doit_switch`, config table `08 01 15 00 14 00 01 03`):

| Pin | Function | Active | OpenBeken role |
|---|---|---|---|
| GPIO8 | relay | high | `Rel` |
| GPIO20 | on-board button | low | `Btn` |
| GPIO14 | external wall switch (toggle on every change) | auto | `TglChanOnTgl` |
| GPIO21 | status LED | low | `LED_n` |

```
SetPinRole 8 Rel;  SetPinChannel 8 1
SetPinRole 20 Btn; SetPinChannel 20 1
SetPinRole 14 TglChanOnTgl; SetPinChannel 14 1
SetPinRole 21 LED_n; SetPinChannel 21 1
SetStartValue 1 -1
```

## Building it yourself

Standard OpenBeken build (`make APP_VERSION=<ver> OpenBL602`). On macOS this needed GNU make 4.x (Apple's make 3.81
fails on the component rules), a wrapper for GNU-style `sed -i`, and Python with `fdt toml configobj pycryptodomex`. The
SDK ships a Darwin RISC-V toolchain (x86_64, runs under Rosetta). Then build the flash image with `bflb_iot_tool
--build` and the custom partition table.

## Credits

All firmware credit goes to the [OpenBeken](https://github.com/openshwprojects/OpenBK7231T_App) project and the
[OpenBL602 SDK](https://github.com/openshwprojects/OpenBL602) (Apache-2.0), plus the 1 MB BL602 discussion on
[Elektroda](https://www.elektroda.com/rtvforum/topic4086736.html). This repo only adds the partition table, the
prebuilt image and the notes above. Flash at your own risk.
