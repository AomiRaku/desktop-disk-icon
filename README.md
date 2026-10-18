# Desktop Disk Icon (Windhawk mod)

English | [简体中文](README_cn.md)

This project is a mod for Windhawk.

Behaves like macOS: plug in an external disk and an icon appears on the desktop; unplug it and the
icon disappears. The icon is the disk's own icon — the same one Windows shows for the volume in
"This PC" (volume label icon, `autorun.inf` icon or device icon). The mod never substitutes its own
artwork.

- Main file: `desktop-disk-icon.wh.cpp`
- Supported volumes: USB flash drives, external hard disks and SSDs, optical media (CD/DVD/Blu-ray
  and mounted ISO images), floppy disks and memory cards.
- Only volumes are shown, matching macOS: phones (MTP/PTP, iPhone) are not mounted as volumes and
  therefore never appear on the desktop.

## Screenshot

Three device types on the test machine. Each icon is provided by the system for that disk:

![Desktop icons](https://i.imgur.com/oTo4VIr.png)

## Installation

1. Open Windhawk → "My mods" → **Create a new mod**.
2. Open `desktop-disk-icon.wh.cpp` and replace the template in the editor with its **entire**
   contents.
3. Click **Compile mod**, then **Install** once it compiles without errors.
4. Make sure the mod is enabled. Disks that are already attached receive an icon within a few
   seconds; subsequent plug and unplug events are reflected immediately.

> Alternatively, copy the `.wh.cpp` into `C:\ProgramData\Windhawk\ModsSource\` (administrator
> rights required) and open it with the Windhawk editor before compiling and installing.

The mod targets `explorer.exe` only (`// @include explorer.exe`) and requires no administrator
rights.

## Settings

Settings use Windhawk's localization mechanism (`$name:zh-CN`, `$description:zh-CN`,
`$options:zh-CN`); the two languages are stored independently and never mixed.

| Setting | Key | Default | Description |
| --- | --- | --- | --- |
| Removable drives | `includeRemovable` | on | Drives Windows reports as removable: USB flash drives, floppy disks, card readers |
| External fixed disks | `includeExternalFixed` | on | Fixed-type disks attached through an external bus (USB, IEEE 1394, SD/MMC, virtual) |
| Optical drives | `includeOptical` | on | Optical drives and mounted ISO images |
| Show drives without media | `showEmptyDrives` | off | Whether empty card reader slots and empty optical drives are shown |
| Polling interval (seconds) | `pollSeconds` | 0 | Periodic synchronization interval; 0 relies on system events only. Set it to about 10 if plugging or unplugging a disk does not update the desktop |

## How it works

The mod starts a background thread inside `explorer.exe`:

1. Enumerates the available drives (`GetLogicalDrives` + `GetDriveTypeW`). External hard disks are
   recognised by bus type through `IOCTL_STORAGE_QUERY_PROPERTY`; internal SATA/NVMe disks are never
   shown.
2. Registers one shell namespace item per drive letter — the same category of desktop object as
   "This PC" and "Recycle Bin", so no `.lnk` file is created and no shortcut arrow is drawn. The
   item is implemented by the shell's own delegated folder: `InProcServer32` is `shell32.dll` and
   `Instance\InitPropertyBag\TargetFolderPath` points at the volume. No mod code is ever loaded by
   the shell or by any other process, and the item behaves exactly like a built-in folder.
3. Reacts to three independent triggers: volume device broadcasts (`WM_DEVICECHANGE`), shell drive
   and media change notifications, and an optional periodic poll. All three just set an event, so a
   single synchronization loop does the work and cannot be blocked by a shell callback.
4. Every pass reconciles the desktop with the attached devices: a missing item is re-created, the
   item of a removed device is deleted.

### Side effects and cleanup

- **Nothing is written to the system registry.** The item definitions live in a private hive
  (`RegLoadAppKey`) inside the mod's storage, and five registry read hooks
  (`RegOpenKeyExW` / `RegEnumKeyExW` / `RegQueryInfoKeyW` / `RegGetValueW` / `RegCloseKey`) redirect
  Explorer's reads
  of `Desktop\NameSpace` and `Software\Classes\CLSID` to that hive. The mod's entries are appended
  to the real `NameSpace` enumeration, so built-in items such as This PC and Recycle Bin keep
  working.
- Because the hooks only run in Explorer, the items never show up in other applications'
  Open/Save dialogs, and they disappear as soon as the mod is unloaded. The bookkeeping values the
  shell writes under `Shell Extensions\Cached` are removed on unload as well.
- No files are created on the desktop, so nothing is synced away by OneDrive or other desktop
  backups.
- The mod only acts in the Explorer process that owns the desktop window; other
  `explorer.exe` instances (folder windows in a separate process, `explorer.exe /select,` and
  similar short-lived processes) are skipped, so they never touch the shared registry keys.

## Verified behaviour

Verified on Windows 11 build 26300 (x64) with Windhawk 1.7.3:

- USB flash drive `F:`, external SSD `G:` (fixed-type over USB, exFAT) and a mounted ISO as optical
  drive `H:` are recognised and displayed; internal NVMe volumes are not.
- Icons inherit the disk's own icon — the optical drive shows the disc's own installer icon taken
  from its `autorun.inf`.
- The registered item is fully functional: it resolves to the volume, enumerates its entries and
  opens that folder (`Self.Path` is `G:\`).
- Plug and unplug are reflected in the desktop view immediately (device broadcast → sync → shell
  notification).
- After disabling the mod and rebooting, neither `Desktop\NameSpace`, `Classes\CLSID` nor
  `Shell Extensions\Cached` contains an entry for the mod's CLSIDs.

## Known limitations

- **Bus type detection**: disks attached through Thunderbolt or eSATA may report an internal bus
  and are therefore not shown. Internal disks are never shown.
- **Empty media**: optical drives without a disc and empty card reader slots are hidden by default.
- **Phones are not shown**, because they are not volumes (same as macOS).
- **No eject command**: the icon opens the volume; use the system tray safe-removal function before
  unplugging.
- **Icon names contain no colon**, because Windows filenames do not allow it — hence `U 盘 (F)`
  rather than `U 盘 (F:)` (the label part is the localized shell name for the drive).
- **Icon position** is whatever Explorer decides; the mod does not move icons.

## License

This mod is released under the GNU General Public License v3.0. See [LICENSE](LICENSE).

It is distributed in the hope that it will be useful, but **without any warranty**; see the license
text for details.