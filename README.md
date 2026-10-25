# android-reader-pen-hack-for-the-s65pro
this is a upload of work i have already done to get the andoird reader pen s65pro into a bare asop keep in mind its android 5.1 and has nothing but android
[README.md](https://github.com/user-attachments/files/33923315/README.md)
# pen-flash-toolkit

Modular scripts for dumping, exploring, editing, and reflashing raw partitions
on MediaTek-based devices (built against a MediaTek MT6582 "scanning dictionary
pen", via [mtkclient](https://github.com/bkerler/mtkclient)). Should work on
any MTK device whose preloader has no SBC/SLA/DAA protection enabled.

**Use this only on hardware you own.** Flashing the wrong offset/length can
brick a device. Always dump and back up a partition before overwriting it.

## Setup

Pick the installer for your distro:

```bash
./install-deps-debian.sh   # Debian / Ubuntu
./install-deps-arch.sh     # Arch / CachyOS / Manjaro
```

Both clone [mtkclient](https://github.com/bkerler/mtkclient) to `~/mtkclient`
and install its Python requirements, plus `adb`, `binwalk`, `e2fsprogs`,
`erofs-utils`, and udev rules for passwordless USB access.

Copy `device.conf.example` to `device.conf` and fill in the offsets for your
own device (see comments inside - every device's layout differs).

## Scripts

| Script | Purpose |
|---|---|
| `setup-wizard.sh` | **Start here.** Walks through everything below end-to-end: install deps, dump, locate `system`, strip to a bare-AOSP keep-list, flash, verify boot. Stops and asks before every destructive step. |
| `dump.sh` | Read a raw offset/length off the device into a local file. Use this to back up a partition, or dump the whole flash. |
| `flash.sh` | Write a local image file to a raw offset/length on the device. Validates the file size against the target length before writing. |
| `explore.sh` | Mount an `.img` file read-only or read-write to browse/edit it, or scan a full flash dump for ext4 filesystem signatures. |
| `mkimage.sh` | Create a blank ext4 image of a given size, or carve a sized chunk out of a larger flash dump. |
| `drag-drop-flash-debian.sh` / `drag-drop-flash-arch.sh` | Drop an `.img` onto one of these in your file manager; it prompts for the offset/length and calls `flash.sh`. |

All offset/length arguments are hex, e.g. `0x4480000`.

## Quickest path: the wizard

```bash
./setup-wizard.sh
```

This runs the full sequence interactively - install deps, dump the flash,
find and carve out `system`, strip it to a bare-AOSP keep-list (editable at
the top of the script), flash it back, and check for ADB once it boots.
Everything it does is also available as the individual scripts below if you'd
rather run it by hand or it stops partway through.

## Typical manual workflow

1. **Dump the whole flash** to back it up before touching anything:
   ```bash
   ./dump.sh 0x0 0xe7000000 flash_full.bin
   ```
   (Use your device's actual flash size - check `mtk.py` output for `m_emmc_ua_size`.)

2. **Find partition boundaries** inside the dump:
   ```bash
   ./explore.sh find-fs flash_full.bin
   ```
   This lists every ext4 signature and its offset/size, which is how you
   figure out where `system`, `boot`, etc. actually start - MediaTek devices
   frequently have no readable GPT/MBR table, so this is the reliable method.

3. **Carve out the partition** you want to edit:
   ```bash
   ./mkimage.sh carve flash_full.bin 0x4480000 0x8FC00000 system.img
   ```

4. **Mount and edit it**:
   ```bash
   ./explore.sh mount system.img ./system_mount --rw
   # make your changes under ./system_mount
   ./explore.sh umount ./system_mount
   ```

5. **Flash it back**:
   ```bash
   ./flash.sh 0x4480000 0x8FC00000 system.img
   ```
   This waits for the device to show up in preloader mode, then writes it.

## Notes

- `wait_for_preloader` (in `lib/common.sh`) polls `lsusb` for the preloader
  VID:PID (`0e8d:2000` by default - override with `MTK_PRELOADER_ID` if yours
  differs) and proceeds the instant it appears, so you don't need to catch a
  narrow timing window by hand.
- `flash.sh` refuses to silently write a file whose size doesn't match the
  target partition length; a mismatch is the easiest way to corrupt adjacent
  partitions.
- None of these scripts touch the bootloader or do anything mtkclient's
  `mtk.py wo`/`mtk.py ro` (raw offset write/read) commands don't already do -
  they're just safety rails and conveniences around them.
