# UEFI for munch

The POCO F4's platform for HakiOS's UEFI (`uefi/` of the common repository: Mu-Silicium with HakiOS's changes and banner),
on Silicium's `KonaPkg` (SM8250, `SOC_TYPE = 2`: SM8250-AC, the Snapdragon 870).

It is the platform of the branch `bck` of [hackios-xnu-kernel-munch](https://github.com/olzhas0986/hackios-xnu-kernel-munch/tree/bck)
(`devices/poco-f4/uefi/files/`, commit `f0a204a`), unchanged; only the UEFI is taken from there. That platform was adapted
from `uefi/umi/files` of [hackios-xnu-kernel-umi](https://github.com/JopangJovan/hackios-xnu-kernel-umi) (commit
`6b6fb2b0c5a1720496ec71e88c95d2a45a0046a4`); the Project Mu files keep their BSD-2-Clause-Patent notices.

What is here, in `files/`:

- `Platforms/Xiaomi/munchPkg`: the memory map, the driver list and the boot manager;
- `Resources/Configs/munch.toml`: the FD at `0x9FC00000`, 3 MiB, gzip and the device tree behind it, boot image v1;
- `Resources/DTBs/munch.dtb`: the device tree of the phone's running Android (`/sys/firmware/fdt`, SHA-256 `8b2beab3…d366`);
- `Binaries/munch`: the vendor's UEFI drivers, **not here**: `extract-xbl.py` makes them from your own phone's `xbl`.

## How it differs from redwood

| | redwood | munch |
|---|---|---|
| SoC package | `KodiakPkg` | `KonaPkg` |
| the FD | `0x9FD00000`, 2 MiB | `0x9FC00000`, 3 MiB (the F4's XBL) |
| the framebuffer | `0xE1000000` | `0x9C000000`, 1080x2400, BGRA (seen on the phone) |
| the boot manager | HakiOS's `XnuLoader.c` in SiliciumPkg (`hakios.patch`) | its own, `MunchXnu.c`: the `XNUP` container at `0xF0000000`, `loader.efi` started from RAM, nothing read from storage |
| `FeatureEnablerDxe` | patched by Silicium | left out: with it the DEBUG build stopped with `Device Error` (`TEST-RESULTS.md`) |

The memory map keeps the 64 MiB at `0xF0000000` for the payload, as redwood's does, and the DDR hole
`[0xBCC00000, 0xC0000000)` the phone's tree shows.

## The vendor's drivers

From your own phone's `xbl` (a dump of the partition, or the `xbl.img` of its firmware package), with
[uefi_firmware](https://pypi.org/project/uefi-firmware/) 1.16 and the templates of `alioth` (also SM8250):

```sh
python3 devices/munch/uefi/extract-xbl.py xbl.img --templates Mu-Silicium/Binaries/alioth --output Mu-Silicium/Binaries/munch
```

It keeps the vendor's bytes as they are (no patch) and writes `manifest.json` with the XBL's hash and each section's. It refuses
an output directory that exists. Never commit what it writes.

The XBL this port was built with: `BOOT.XF.3.2-00354-SM8250-1`, SHA-256 `696eaefe…3c3b`, 91 modules. The same 91 sections,
byte for byte, as those of the Windows port of the F4 made from the same file. `bck` was tested with another dump
(`1535264e…6f2e`); its build script checks that hash, this one does not.

## Build

```sh
uefi/apply.sh munch                    # or by hand on a Mu-Silicium at BASE: hakios.patch, uefi/files/, the banner, files/ here
cd Mu-Silicium
python3 Platforms/Xiaomi/munchPkg/DeviceBuild.py TARGET=RELEASE ENABLE_SECUREBOOT=0 \
    FD_BASE=0x9fc00000 FD_SIZE=0x300000 FD_BLOCKS=0x300 DEVICE_MODEL=0 MUNCH_STORAGE=1
                                       # -> Build/munchPkg/RELEASE_CLANGPDB/FV/SILICIUM_UEFI.fd, 3145728 bytes
```

`MUNCH_STORAGE=1` adds the vendor's UFSDxe, DiskIoDxe and PartitionDxe. `TARGET=DEBUG` writes the firmware's log on the
screen. `BASE` here is the Mu-Silicium commit `bck` was made on (`42fe5637`); the common `uefi/BASE` (`e51cd877`) is six commits
older, and they differ in Samsung's platforms and two submodules only. `hakios.patch` applies to both.

## What has been checked

16 November 2026, on the host: `hakios.patch` applied to Mu-Silicium at `42fe5637` with no conflict; the drivers extracted
from the XBL above; the RELEASE build with `MUNCH_STORAGE=1` succeeded (clang 21), 48 images verified, the FD 3145728 bytes,
`HakiOS` as its vendor and `MunchXnu`'s message in it, no `FeatureEnablerDxe`. **This build has not been started on the phone.**

On the phone, with `bck`'s builds (`TEST-RESULTS.md`, kept as it was): the boot image accepted by `fastboot boot`, the shim,
SEC and DXE, the GOP at 1080x2400, an EFI application started from the `XNUP` container, USB fastboot round trips, and XNU's
early start up to `arm_init`.

Not yet: packing this FD with a payload. `mkxnuimg.py` and `xl` take redwood's FD base and size and its framebuffer address;
munch needs its own.
