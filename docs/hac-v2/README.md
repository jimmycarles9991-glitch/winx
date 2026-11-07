# HAC-001-01 (Switch V2 / Mariko) port of WinX

Goal: run the same Windows 11 ARM64 chain (Hekate → BL31 → BL33 → Windows Boot
Manager → Windows) on a modchipped **ordinary Switch V2** (HAC-001-01,
Tegra210 B01 / T214). The SoC is identical to the HDH-001 Lite this port was
built for; the differences are board-level. A modchip (payload inflater such as
hwfly/picofly-family) is irrelevant to WinX itself — it only supplies Hekate.

Status marker used below: `same`, `adapt`, `generic path exists`, `new work`.

## Build flag map (NintendoSwitch.dsc / NintendoSwitch.fdf)

| Lite flag | Purpose | V2 (HAC V2) treatment |
|---|---|---|
| `SWITCH_HEKATE` | Hekate handoff boot chain | `same` — keep |
| `SWITCH_EXCFB=1` + `SWITCH_FB_ROTATE=2` | Reuse Hekate's POST framebuffer | `same` — panel pipeline identical on B01 units |
| `SWITCH_ACPI_SD_NVTSDHC` + `SWITCH_LITE_BOOT_SD` | Bounded WINARM SD boot path | `adapt` — SoC-side code is T214-common; only the DSC gate *forces* `SWITCH_LITE_TOUCH`. Relaxed with `SWITCH_HAC_V2` (this branch) |
| `SWITCH_ACPI_SWOA=1` | Generic SWOA device set (GPIO SWOA0101, I2C SWOA0201, touch SWOA0501 = **ordinary HAC touch panel**, audio SWOA0C01 = ALC5639, sensors SWOA0901, BT SWOA0B01) | `generic path exists` — inherited from hayato41200's ordinary-Switch port. Preferred over the Lite device set on V2 |
| `SWITCH_LITE_TOUCH` | Powers the FTM4 panel (LDO6 2.9 V, PJ7 reset) and emits ACPI SWOA0502 + `LiteTouchPower.inf` | `adapt` — the FTM4 controller and its rails exist on the HAC V2 too; power sequence is expected to match, but the ACPI _HID must stay SWOA0501 so the generic `switchtouch` INF binds. Resolution track for stage 2 |
| `SWITCH_LITE_FAN` | Fan bring-up on I2C1 + `LiteFanPower.inf`, initial duty via `SWITCH_FAN_DUTY` | `same family` — V2 has the same 5 V PWM fan class; `FanDxe.c` states the path is "Mariko/HDH-001". V2 probably accepts the same sequence; verify TMP451 thermal guard reads on HAC |
| `SWITCH_LITE_POWER` | PMIC/I2C5 power profile for the Windows power drivers | `adapt` — both boards run MAX77620 on I2C5; rail map differs (V2 has dock/PD rails). Compare against `drivers/Tegra214Power` and the Lite PMIC notes before trusting anything |
| `SWITCH_LITE_SYSTEM` | Lite battery/audio/buttons set | `adapt` — battery gauge and volume buttons are board-common enough to try; audio codec (RT5639 on Lite vs ALC5639 on HAC) differs, prefer the generic `SWOA0C01` |
| `SWITCH_LITE_SIO` | Lite's internal UART-C controller gamepad | **skip** — V2 uses Joy-Cons/Joy-Con grip over Bluetooth HID (`switchbt`), not synthesized HID. Expect to use the Windows Bluetooth stack with Joy-Con HID games later; no SIO equivalent exists |
| `SWITCH_LITE_XUSB` + `SWITCH_LITE_*` bundle | Lite USB-C PD profile (BM92T on I2C1 0x18, lanes 0/5, PD ceilings 2000/1500/1200/900 mA) | `adapt` — V2 USB-C BM92T is also on I2C1 0x18 but the PD tree differs (dock-capable). Do a dedicated map from switchroot `tegra210b01-hac` power-tree before enabling |
| `SWITCH_LITE_DISPLAY` / winx-display | Display-only ACPI device for the Windows miniport | `same` idea; keep off until storage boots |
| `SWITCH_LITE_SLEEP` / `SWITCH_LITE_PROCESSOR_IDLE` | Sleep/wake work | **skip in stage 1** — unaccepted even on the Lite |
| `SWITCH_LITE_RESET`, `SWITCH_LITE_PERFORMANCE` | Reset/clock management under `t214/` | `same` (T214-common code under `t214/`) |
| (FDF) `PinMuxDxe`, `TouchPanelDxe`, `SidebandButtonDxe` | Erista-profile drivers the FDF includes whenever neither `SWITCH_WOA` nor `SWITCH_LITE_TOUCH` is set | **excluded for V2** (`!ifndef SWITCH_HAC_V2` in the FDF). `PinMuxDxe` applies a fixed GPIO/pingroup/drvgroup table with no SoC or board check, so linking it would reprogram a V2's pads from an Erista table; hekate's pinmux is relied on instead, as on the Lite. `TouchPanelDxe` is its only hard dependent. Revisit with the V2 pin tables in stage 2 |

Rules preserved:

- `SWITCH_LITE_TOUCH` and `SWITCH_ACPI_SWOA` stay mutually exclusive (DSC
  `!error`), so the V2 profile must pick one. Stage 1 defines **neither**
  (`SWITCH_HAC_V2` replaces `SWITCH_LITE_TOUCH` as the DSC/FDF gate); the touch
  path is chosen in stage 2 (see the `SWITCH_LITE_TOUCH` row above).
- Do not use `Tools/build.sh` / `build-woa.sh` defaults: they select the Erista
  profile, which errors against this fork intentionally.

## Driver kit map (WindowsDrivers submodule, pinned 3901abb)

| Lite INF | V2 replacement |
|---|---|
| `SwitchLiteTouch/switchtouch.inf` (ACPI\SWOA0502) | `SwitchTouch/switchtouch.inf` (ACPI\SWOA0501, HAC FTM4, 77 × 137 mm extents) — `generic path exists`; the two sources diff only in HID physical extents and Lite display-idle integration |
| `switchsio` (Lite UART-C gamepad) | none — Joy-Con HID over Bluetooth (`switchbt`) is the only control path on a V2 |
| `switchaudio` (Lite codec) | generic audio driver binding `SWOA0C01` (ALC5639/I2C1) |
| `switchbattery`, `switchpwrbtn`, `switchfan`, `switchrtc`, `tegra210gpio`, `tegra210i2c`, `nvtsdhc`, `switchwifi`, `switchbt` | same driver sources; expected board deltas are rail map, PD profile, thermal table |

## Card and storage

Unchanged from the Lite guide (`docs/hdh001-home/README.md`): the 80 GiB
`WINARM` GPT/hybrid-MBR layout, card-image-derived `winarm-target.h`, and the
single-header pairing between BL33 and the Windows `nvtsdhc` miniport are all
*T214-common* and can be reused as-is for the V2.

## Stage plan

Build command (BL31 is built once with `Tools/build-t214-atf.py` as in the
Lite guide; the C/ifdef inputs below select the V2 profile for BL33):

```sh
build -a AARCH64 -t GCC5 -b RELEASE -n 8 -p NintendoSwitchPkg/NintendoSwitch.dsc \
  -D SWITCH_HAC_V2=1 -D SWITCH_HEKATE=1 -D SWITCH_FB_ROTATE=2 -D SWITCH_EXCFB=1 \
  -D SWITCH_ACPI_SD_NVTSDHC=1 -D SWITCH_LITE_BOOT_SD=1 \
  -D SWITCH_LITE_FAN=1 -D SWITCH_FAN_DUTY=76 \
  -D SWITCH_LITE_POWER=1 -D SWITCH_LITE_RESET=1 \
  -D SWITCH_LITE_BLUETOOTH=1 -D SWITCH_WIFI_RUNTIME=1 \
  -D SWITCH_LITE_BACKLIGHT=1
```

Deliberately NOT selected in stage 1: `SWITCH_LITE_TOUCH` (must be re-pointed
to the SWOA0501/`SwitchTouch` HAC path first), `SWITCH_LITE_XUSB`,
`SWITCH_LITE_SIO`, `SWITCH_LITE_AUDIO`, `SWITCH_LITE_SYSTEM`,
`SWITCH_LITE_SLEEP`, `SWITCH_LITE_PROCESSOR_IDLE`. See the flag map above for
the reason each is deferred.

1. **Branch `hac-v2-port`** (this branch): relax DSC gates so the bounded SD
   boot profile can build without the HDH-001-only flags when
   `SWITCH_HAC_V2=1` is passed. Lite profile builds must be unchanged.
2. ~~Build both profiles and compare sizes.~~ **Done (compile-only), see
   "Stage 1 build validation".**
3. Boot bring-up order on the V2 (Hekate level, after BL31):
   framebuffer/console → SD/WINARM boot → UEFI shell → WinPE → Windows.
   Capture serial debug — `SWITCH_SERIAL_` debug exists in the tree.
4. Bring up devices one at a time on the SWOA0501 path: GPIO → I2C3 → touch.
   Then power/fan/battery (rail-map check on V2 first), audio, Wi-Fi/BT.
5. Controls: verify Joy-Con over Bluetooth in Windows before attempting any
   HID gamepad layer, since there is no SIO equivalent.
6. USB-C host and sleep/wake stay last, mirroring the Lite's acceptance bar.

## Stage 1 build validation (compile-only)

Built with EDK2 `edk2-stable202402` + `patches/edk2/0001`, Ubuntu 24.04
`aarch64-linux-gnu-gcc` 13.3, `GCC5` profile. The card header was a **synthetic
stub** (fake GUIDs/serial) because no card image was available; these results
say the profile compiles and links, nothing about boot behaviour, and the
produced images must not be flashed.

| Profile | Result | `FVMAIN_COMPACT` (limit 0xE0000) |
|---|---|---|
| Stage 1 V2 (command above) | builds | 730,424 B, 79% |
| Lite (command in `docs/hdh001-home/README.md`) | builds | 796,328 B, 86% |

- Lite regression check: the Lite `TEGRA210_EFI.fd` is byte-identical
  (same SHA-256) with and without the V2 FDF change, so the gates are inert
  for non-V2 builds.
- Image contents were checked in the generated `FVMAIN.inf`, not just the
  build log: V2 contains `PmicDxe`, `ClockManagementDxe`, `SdMmcDxe`,
  `SimpleFbDxe`, `LiteFanPower`, `LiteReset` and `LiteBluetooth`; it does not
  contain `PinMuxDxe`, `TouchPanelDxe` or `SidebandButtonDxe`.
- The stock EDK2 USB stack (`EhciDxe`, `UsbBus`, `UsbKb`, `UsbMassStorage`) is
  unconditional in the FDF in every profile; without `SWITCH_USB` there is no
  PciEmulation to bind it to, so it is inert here as on the Lite.
- The NTFS driver (`LiteSdTest/ntfs_aa64_ro.efi`) is only linked when
  `SWITCH_LITE_SYSTEM` is set, so stage 1 does not need it.

### BL31 reuse (unverified)

The T214 ATF patches (`patches/atf`) are selected by the T210B01 chip check, use
the MAX77620 on I2C5 and, for final power-off, bit-bang the SoC's dedicated
`PWR_I2C` pads (PY3/PY4). These are SoC-level, not Lite-specific, so reuse on a
V2 is a reasonable working assumption. Restart and power-off behaviour on a V2
is unobserved until tested; keep a serial/known-good Hekate path available.

## Non-goals / known risks

- No hardware acceptance can be recorded from build success alone. Everything
  marked `adapt` must be verified against the switchroot B01 HAC device trees
  and the actual board before it is trusted.
- Never reuse another card's generated header, and always back up the card
  before touching boot files.
- Do not delete existing Hekate entries when merging INI files.
