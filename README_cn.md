# 桌面显示U盘图标（Windhawk 模组）

[English](README.md) | 简体中文

本项目为 Windhawk 的 Mod。

像 macOS 那样：插入外接磁盘 → 桌面出现一个图标；拔出 → 图标消失。图标用的是磁盘本身的图标，
即系统在「此电脑」里为该卷提供的图标（卷标图标、`autorun.inf` 图标或设备图标），模组不会替换成
自己的图片。

- 主文件：`desktop-disk-icon.wh.cpp`
- 支持设备（卷）：U 盘、外接硬盘与固态硬盘、光盘（CD/DVD/蓝光及已挂载的 ISO 映像）、软盘、
  存储卡。
- 只显示「卷」，与 macOS 一致：手机（MTP/PTP、iPhone）不会被挂载成卷，因此不会出现在桌面。

## 截图

三类设备在本机上的实际显示效果（图标由系统按磁盘本身提供）：

![桌面图标](https://i.imgur.com/oTo4VIr.png)

## 安装

1. 打开 Windhawk →「我的模组」→ **创建新模组**（Create a new mod）。
2. 打开 `desktop-disk-icon.wh.cpp`，用其**全部内容**替换编辑器里的模板。
3. 点 **编译模组**（Compile mod），确认无错误后点 **安装**（Install）。
4. 确认模组处于启用状态。已连接的磁盘会在数秒内出现图标，之后的插拔即时反映。

> 也可以把 `.wh.cpp` 复制到 `C:\ProgramData\Windhawk\ModsSource\`（需管理员权限），
> 再用 Windhawk 的编辑器打开并编译安装。

模组通过 `// @include explorer.exe` 只在资源管理器进程中运行，不需要管理员权限。

## 设置项

设置项使用 Windhawk 的多语言机制（`$name:zh-CN` / `$description:zh-CN` / `$options:zh-CN`），
两种语言各自独立、不混排。

| 设置项 | 对应键 | 默认 | 说明 |
| --- | --- | --- | --- |
| 可移动存储设备 | `includeRemovable` | 开 | 系统识别为可移动存储的驱动器：U 盘、软盘、读卡器 |
| 外接固定磁盘 | `includeExternalFixed` | 开 | 通过外部总线（USB、IEEE 1394、SD/MMC、虚拟）连接的固定类型磁盘 |
| 光盘驱动器 | `includeOptical` | 开 | 光盘驱动器及已挂载的 ISO 映像 |
| 显示无介质驱动器 | `showEmptyDrives` | 关 | 是否显示未插入介质的读卡器插槽与空光驱 |
| 轮询间隔（秒） | `pollSeconds` | 0 | 周期同步间隔；设为 0 时仅依赖系统事件，若插拔磁盘后桌面未更新，可改为 10 左右 |

## 工作原理

模组在 `explorer.exe` 中启动一个后台线程：

1. 枚举可用磁盘（`GetLogicalDrives` + `GetDriveTypeW`）。外接硬盘通过
   `IOCTL_STORAGE_QUERY_PROPERTY` 读取总线类型判断；内置 SATA/NVMe 磁盘始终不会显示。
2. 为每个盘符注册一个 shell 命名空间项——与「此电脑」「回收站」属于同一类桌面对象，因此不产生
   `.lnk` 文件、也不显示快捷方式箭头。该项由系统自带的委托文件夹实现：`InProcServer32` 为
   `shell32.dll`，`Instance\InitPropertyBag\TargetFolderPath` 指向该卷根目录。shell 与其他
   进程都不会加载模组代码，行为与系统内置文件夹完全一致。
3. 三条互相独立的触发通道：卷设备广播（`WM_DEVICECHANGE`）、shell 的驱动器与介质变更通知，
   以及可选的周期性轮询。三者都只是置位一个事件，由同一个同步循环处理，因此不会被 shell 的
   回调阻塞。
4. 每次同步都把桌面与当前设备对齐：图标缺失则重建，设备拔出则删除对应项。

### 副作用与清理

- **不写系统注册表。** 项的定义放在模组存储目录下的私有 hive 中（`RegLoadAppKey`），并用 4 个
  注册表读取 hook（`RegOpenKeyExW` / `RegEnumKeyExW` / `RegQueryInfoKeyW` / `RegGetValueW` / `RegCloseKey`）
  把资源管理器对 `Desktop\NameSpace` 与 `Software\Classes\CLSID` 的读取重定向到该 hive。
  我们的条目是**追加**到真实的 `NameSpace` 枚举里，因此「此电脑」「回收站」等系统项照常显示。
- 由于 hook 只在资源管理器进程内生效，这些项不会出现在其他程序的文件对话框里，模组卸载后
  立即消失。资源管理器写进 `Shell Extensions\Cached` 的书签值也会在卸载时清除。
- 不会在桌面创建任何文件，因此不会被 OneDrive 等桌面备份同步到其他电脑。
- 模组只在拥有桌面窗口的资源管理器进程中工作；其他 `explorer.exe` 实例（“在单独进程中打开
  文件夹窗口”、`explorer.exe /select,` 等短命进程）会被跳过，不会改动共享的注册表项。

## 已验证的行为

在本机（Windows 11 版本 26300，x64，Windhawk 1.7.3）验证：

- U 盘 `F:`、外接固态硬盘 `G:`（USB 总线的 Fixed 类型，exFAT）、作为光驱挂载的 ISO `H:`
  都能正确识别并显示；内置 NVMe 卷不显示。
- 图标继承磁盘本身的图标——光盘显示的是该光盘 `autorun.inf` 中指定的安装盘图标。
- 注册出的项功能完整：能解析到该卷、枚举其内容、并打开该文件夹（`Self.Path` 为 `G:\`）。
- 插入与拔出都即时反映到桌面（设备广播 → 同步 → shell 通知）。
- 禁用模组并重启电脑后，`Desktop\NameSpace`、`Classes\CLSID` 与
  `Shell Extensions\Cached` 三处都没有模组 CLSID 的残留。

## 已知限制

- **总线类型识别**：通过雷雳（Thunderbolt）或 eSATA 连接的硬盘可能被报告为内置总线，因而不会
  显示；内置磁盘始终不会显示。
- **空介质**：未放入光盘的光驱、空的读卡器插槽默认隐藏。
- **不显示手机**，因为它们不是卷（与 macOS 一致）。
- **没有弹出命令**：图标用于打开该卷，拔盘前请继续使用系统托盘的安全删除功能。
- **图标名称不含冒号**，因为 Windows 文件名不允许，所以是 `U 盘 (F)` 而不是 `U 盘 (F:)`
  （括号前的部分是系统为该驱动器提供的本地化名称）。
- **图标位置**由资源管理器决定，模组不会移动图标。

## 许可

本模组以 GNU 通用公共许可证第 3 版（GPLv3）发布，全文见 [LICENSE](LICENSE)。

本模组按「现状」提供，不附带任何担保；详细条款见许可证正文。