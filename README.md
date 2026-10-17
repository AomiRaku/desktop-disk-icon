# Desktop Disk Icon (Windhawk mod)

English | [简体中文](README_cn.md)

This project is a mod for Windhawk.

Behaves like macOS: plug in an external disk and an icon appears on the desktop; unplug it and
the icon disappears. The icon is the disk's own icon — the same one Windows shows for the volume
in "This PC" (volume label icon, `autorun.inf` icon or device icon). The mod never substitutes
its own artwork.

- Main file: `desktop-disk-icon.wh.cpp`
- Supported devices: USB flash drives, external hard disks and SSDs, optical media (CD/DVD/
  Blu-ray and mounted ISO images), floppy disks, memory cards, and phones connected as storage
  devices (MTP/PTP).

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
`$options:zh-CN`): a Chinese UI shows Chinese, any other language shows English. The two
languages are stored independently and are never mixed.

| Setting | Key | Default | Description |
| --- | --- | --- | --- |
| Removable drives | `includeRemovable` | on | Drives Windows reports as removable: USB flash drives, floppy disks, card readers |
| External fixed disks | `includeExternalFixed` | on | Fixed-type disks attached through an external bus (USB, IEEE 1394, SD/MMC, virtual) |
| Optical drives | `includeOptical` | on | Optical drives and mounted ISO images |
| Portable devices | `includePortable` | on | MTP/PTP devices; these are presented as desktop shortcuts |
| Show drives without media | `showEmptyDrives` | off | Whether empty card reader slots and empty optical drives are shown |
| Place icons on the public desktop | `publicDesktop` | off | Write the icons to the public desktop so all users see them |
| New icon placement | `iconPlacement` | default | Can be changed to "first free slot of the rightmost column" |
| Icon type | `iconStyle` | namespace | `namespace`: no shortcut arrow, an `IShellFolder` forwards to the disk folder; `shortcut`: ordinary shortcut files |
| Polling interval (seconds) | `pollSeconds` | 0 | Periodic synchronization interval; 0 relies on system events only. Set it to about 10 if plugging or unplugging a disk does not update the desktop |

## How it works

The mod starts a background thread inside `explorer.exe`:

1. Enumerates the available drives (`GetLogicalDrives` + `GetDriveTypeW`) and the portable
   devices listed under "This PC". External hard disks are recognised by bus type through
   `IOCTL_STORAGE_QUERY_PROPERTY`; internal SATA/NVMe disks are never shown.
2. In the default `namespace` mode, registers one shell namespace item per drive letter, the
   same category of desktop object as "This PC" and "Recycle Bin" (see below). In `shortcut`
   mode it writes ordinary `.lnk` files instead, tagged with a hidden `wh-dex|...` argument so
   that only its own icons are ever touched.
3. Reacts to three independent triggers: volume device broadcasts (`WM_DEVICECHANGE`), shell
   drive and media change notifications, and an optional periodic poll. All three just set an
   event, so a single synchronization loop does the work and cannot be blocked by a shell
   callback.
4. Every pass reconciles the desktop with the attached devices: a missing icon is re-created,
   the icon of a removed device is deleted, and shortcuts that were not created by this mod are
   never modified.
5. Disabling or uninstalling the mod (`Wh_ModUninit`) removes everything it created.

## Verified behaviour

Verified on the test machine (Windows 11 build 26300, Windhawk 1.7.3) with test programs built
from the same source as the mod:

- A USB flash drive `F:`, an external SSD `G:` (fixed-type over USB, exFAT) and a mounted ISO as
  optical drive `H:` are all recognised and displayed; the internal NVMe volumes `C:/D:/E:` are
  not.
- Icons inherit the disk's own icon — the optical drive shows the disc's own Adobe installer
  icon taken from its `autorun.inf`.
- New items appear on the desktop without a manual refresh (the desktop view item count
  increases immediately).

### It does not fight the user

All of the following were verified:

| Situation | Behaviour |
| --- | --- |
| The user deletes a disk icon | Respected — it is **not** re-created in the next pass; it comes back when the device is plugged in again |
| The user renames an icon | The chosen name is kept |
| The user has a shortcut with the same name | The mod switches to `name (2)`; user files are never overwritten |
| Icons on the desktop without the mod's marker | Never touched |
| Icon location changed to the public desktop | Icons in the previous location are removed before the new ones are created |
| A device is unplugged | Its icon is deleted, including icons the user renamed |
| The mod is disabled or uninstalled | Every icon it created in both desktop folders is removed |
| Disabled and re-enabled inside the same process | Rebuilt correctly (no state left behind that would silence it) |

### Residue control

- The shortcut mode only creates and deletes `.lnk` files carrying the `wh-dex` marker; it never
  touches other desktop files.
- The namespace mode writes only two keys per drive, both under `HKEY_CURRENT_USER`
  (`…\Classes\CLSID\{CLSID}` and `…\Explorer\Desktop\NameSpace\{CLSID}`), and removes them when
  the drive goes away, when the mod is disabled, and — after a forced termination — on the next
  start, by scanning for its own prefix.
- Switching the desktop location in the settings does not leave icons in both places.

### Icon placement

With *New icon placement* set to "first free slot of the rightmost column", each new icon is
moved to the rightmost column of the desktop, filling it from the top:

1. `IFolderView::SelectAndPositionItems` is used first — the shell remembers the position.
2. If the desktop view cannot be obtained (for example when COM activation fails), the mod falls
   back to `LVM_FINDITEM` + `LVM_SETITEMPOSITION` on the desktop `SysListView32` from inside
   Explorer.
3. With "Auto arrange icons" enabled the shell owns the layout and both methods are overridden;
   turn that option off for this feature.

## Known limitations

- **Bus type detection**: disks attached through Thunderbolt or eSATA may report an internal bus
  and are therefore not shown. Internal disks are never shown.
- **Empty media**: optical drives without a disc and empty card reader slots are hidden by
  default (the *Show drives without media* setting changes that).
- **Phones**: the icon name of an MTP device comes from the system description of that device
  and may differ slightly between models. If a phone or camera driver enumerates slowly, disable
  *Portable devices*.
- **No eject command**: the icon opens the volume; it does not provide a safe-removal command.
  Keep using the system tray function before unplugging.
- **Icon names contain no colon**, because Windows filenames do not allow it — hence
  `U 盘 (F)` rather than `U 盘 (F:)` (the label part is the localized shell name for the drive).
- **Explorer restart after an update**: the namespace extension DLL stays loaded once the shell
  has loaded it, so a new build only takes effect after Explorer is restarted.

## Background: why a COM forwarder is required

Registering a drive as a plain shell namespace item through the registry
(`Desktop\NameSpace\{CLSID}` plus the shell's own "folder shortcut" instance class
`{0AFACED1-E828-11D1-9187-B532F1E9575D}`) was evaluated on the test machine. The item appears,
takes a custom name, shows the disk's own icon through `DefaultIcon`
(`SHGFI_ICONLOCATION` — for optical media this yields the icon declared in `autorun.inf`) and
carries no shortcut arrow, but it does not open the drive: `InvokeVerb('open')` does nothing,
because nothing delegates to a real folder. Several registry variants were tried, each with the
same GUID, verified through `InvokeVerb('open')` and cleaned up afterwards:

| Variant | Instance CLSID | ShellFolder Attributes | Result |
| --- | --- | --- | --- |
| V1 | `{0AFACED1-…}` Folder Shortcut | `0xF080004D` | Appears, but does not open |
| V2 | `{0E5AAE11-…}` DelegateFolder | `0xF080004D` | Does not appear at all |
| V3 | Folder Shortcut | `0xF080004D \| BROWSABLE` | Does not appear at all |
| V4 | DelegateFolder | `0xF080004D \| BROWSABLE` | Does not appear at all |
| V5 | DelegateFolder | `… + FolderValueFlags=0x28` | Does not appear at all |
| V6 | DelegateFolder | `SFGAO_FOLDER` | Does not appear at all |

`{0E5AAE11-…}` only accepts `TargetKnownFolder` — every built-in instance on the machine
(Downloads, Public Folder) uses that value — and rejects `TargetFolderPath`, while the
`{0AFACED1-…}` folder shortcut only renders but never delegates. A working namespace item
therefore needs its own COM server implementing `IShellFolder`, which is exactly how "This PC"
and "Recycle Bin" work.

## Namespace items (default mode)

In the default mode the mod registers one **shell namespace item** per drive letter instead of
writing `.lnk` files:

- The item is registered through `HKCU\Software\Classes\CLSID\{CLSID}` plus
  `Desktop\NameSpace\{CLSID}`; the CLSID is derived from the drive letter.
- `InProcServer32` points at **the mod's own DLL**. The mod already lives inside Explorer, and
  the DLL exports `DllGetClassObject`, so the shell can create the object directly and no
  separate installer or helper DLL is needed.
- That COM class implements a **forwarding `IShellFolder`**: after `Initialize` it reads the
  volume path from `TargetFolderPath` and forwards `EnumObjects`, `BindToObject`,
  `CreateViewObject`, `GetUIObjectOf` and `GetDisplayNameOf` to the real folder object of the
  volume. Enumeration, opening, context menus and icons therefore behave exactly like the real
  directory, while the desktop shows a pure namespace item: **no `.lnk` file and no shortcut
  arrow**.
- The icon comes from `DefaultIcon`, filled with the disk's own icon location.

Measured on the test machine: the item appears immediately, enumerates the 8 entries of `G:\`,
and opening it shows those 8 folders. One detail is essential — `ShellFolder\Attributes` must
include `SFGAO_HASSUBFOLDER` (`0x28000000`), otherwise Explorer refuses to open the item.

Cleanup: unplugging a drive deletes its CLSID; disabling or uninstalling the mod deletes all of
them (including the keys under `HKCU\Software\Classes\CLSID`); after a forced termination the
orphans are removed on the next start. Portable devices (phones) still use desktop shortcuts,
because an MTP device has no file system path that could be placed in `TargetFolderPath`.

Note: once Explorer has loaded the namespace extension DLL it is not unloaded, so updating the
mod requires restarting Explorer.

## License

This mod is released under the GNU General Public License v3.0. See [LICENSE](LICENSE).

It is distributed in the hope that it will be useful, but **without any warranty**; see the license
text for details.

