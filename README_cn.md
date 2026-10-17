# 桌面显示U盘图标（Windhawk 模组）

[English](README.md) | 简体中文

本项目为 Windhawk 的 Mod。

像 macOS 那样：插入外接磁盘 → 桌面出现一个图标；拔出 → 图标消失。图标用的是磁盘本身的图标，
即系统在「此电脑」里为该卷提供的图标（卷标图标、`autorun.inf` 图标或设备图标），模组不会替换成
自己的图片。

- 主文件：`desktop-disk-icon.wh.cpp`
- 支持设备：U 盘、外接硬盘与固态硬盘、光盘（CD/DVD/蓝光及已挂载的 ISO 映像）、软盘、存储卡，
  以及以存储设备方式连接的手机（MTP/PTP）。

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

设置项使用 Windhawk 的多语言机制（`$name:zh-CN` / `$description:zh-CN` / `$options:zh-CN`）：
界面语言为中文时显示中文，其他语言显示英文，两种语言各自独立、不混排。

| 设置项 | 对应键 | 默认 | 说明 |
| --- | --- | --- | --- |
| 可移动存储设备 | `includeRemovable` | 开 | 系统识别为可移动存储的驱动器：U 盘、软盘、读卡器 |
| 外接固定磁盘 | `includeExternalFixed` | 开 | 通过外部总线（USB、IEEE 1394、SD/MMC、虚拟）连接的固定类型磁盘 |
| 光盘驱动器 | `includeOptical` | 开 | 光盘驱动器及已挂载的 ISO 映像 |
| 便携设备 | `includePortable` | 开 | MTP/PTP 设备；此类设备以桌面快捷方式呈现 |
| 显示无介质驱动器 | `showEmptyDrives` | 关 | 是否显示未插入介质的读卡器插槽与空光驱 |
| 图标写入公用桌面 | `publicDesktop` | 关 | 图标写入公用桌面，本机所有用户可见 |
| 新图标位置 | `iconPlacement` | 默认位置 | 可改为「最右一列的首个空位」 |
| 图标类型 | `iconStyle` | 系统命名空间项 | `namespace`：无快捷方式箭头，由 `IShellFolder` 转发到磁盘目录；`shortcut`：普通快捷方式文件 |
| 轮询间隔（秒） | `pollSeconds` | 0 | 周期同步间隔；设为 0 时仅依赖系统事件，若插拔磁盘后桌面未更新，可改为 10 左右 |

## 工作原理

模组在 `explorer.exe` 中启动一个后台线程：

1. 枚举可用磁盘（`GetLogicalDrives` + `GetDriveTypeW`）与「此电脑」中的便携设备。外接硬盘通过
   `IOCTL_STORAGE_QUERY_PROPERTY` 读取总线类型来判断；内置 SATA/NVMe 磁盘始终不会显示。
2. 默认的 `namespace` 模式下，为每个盘符注册一个 shell 命名空间项——与「此电脑」「回收站」
   属于同一类桌面对象（详见下文）。`shortcut` 模式则写入普通 `.lnk` 文件，并在快捷方式参数中
   带一个隐藏标记 `wh-dex|...`，因此只会操作模组自己创建的图标。
3. 三条互相独立的触发通道：卷设备广播（`WM_DEVICECHANGE`）、shell 的驱动器与介质变更通知，
   以及可选的周期性轮询。三者都只是置位一个事件，由同一个同步循环处理，因此不会被 shell 的
   回调阻塞。
4. 每次同步都把桌面与当前设备对齐：图标缺失则重建，设备拔出则删除对应图标，不是本模组创建的
   快捷方式一律不改动。
5. 禁用或卸载模组时（`Wh_ModUninit`）清理自己创建的全部内容。

## 已验证的行为

在本机（Windows 11 版本 26300，Windhawk 1.7.3）上，用与模组同一份源码编译的测试程序验证：

- U 盘 `F:`、外接固态硬盘 `G:`（USB 总线的 Fixed 类型，exFAT）、作为光驱挂载的 ISO `H:`
  都能正确识别并显示；内置 NVMe 卷 `C:/D:/E:` 不显示。
- 图标继承磁盘本身的图标——光盘显示的是该光盘 `autorun.inf` 中指定的 Adobe 安装盘图标。
- 新图标无需手动刷新即可出现（桌面视图条目数立即增加）。

### 不跟用户抢（均已实测）

| 情况 | 行为 |
| --- | --- |
| 用户手动删除磁盘图标 | 尊重用户，**不会**在下一轮重新创建；设备重新插入后才会再出现 |
| 用户给图标改名 | 保留用户选择的名称 |
| 用户有自己的同名快捷方式 | 自动改用 `名称 (2)`，绝不覆盖用户文件 |
| 桌面上没有本模组标记的图标 | 一律不碰 |
| 图标位置改为「公用桌面」 | 先清除旧位置中自己创建的图标，再在新位置创建 |
| 设备拔出 | 删除对应图标，包括被用户改过名的 |
| 禁用或卸载模组 | 清除在两个桌面目录中创建的全部图标 |
| 在同一进程内禁用后重新启用 | 正常重建（不会因残留状态而失效） |

### 残留控制

- 快捷方式模式只在桌面目录中增删带 `wh-dex` 标记的 `.lnk` 文件，不触碰其他桌面文件。
- 命名空间模式每个盘符只写两个键，且都在 `HKEY_CURRENT_USER` 下（`…\Classes\CLSID\{CLSID}`
  与 `…\Explorer\Desktop\NameSpace\{CLSID}`）；拔出设备、禁用模组时都会删除，被强制终止的
  情况则在下次启动时按自己的前缀扫描清理。
- 在设置中切换过桌面位置，不会在两个位置都留下图标。

### 图标位置

把「新图标位置」设为「最右一列的首个空位」后，新图标会被移动到桌面最右一列，自上而下填充：

1. 首选 `IFolderView::SelectAndPositionItems`——位置会被资源管理器记住；
2. 若取不到桌面视图（例如 COM 激活失败），退回在进程内对桌面 `SysListView32` 发送
   `LVM_FINDITEM` + `LVM_SETITEMPOSITION`；
3. 桌面启用「自动排列图标」时布局由系统接管，两种方法都会被覆盖——该功能需要关闭此选项。

## 已知限制

- **总线类型识别**：通过雷雳（Thunderbolt）或 eSATA 连接的硬盘可能被报告为内置总线，因而不会
  显示；内置磁盘始终不会显示。
- **空介质**：未放入光盘的光驱、空的读卡器插槽默认隐藏（可通过「显示无介质驱动器」打开）。
- **手机**：MTP 设备的图标名称来自系统对该设备的描述，个别机型可能略有差异；若某台手机或相机
  的驱动枚举缓慢，可关闭「便携设备」。
- **没有弹出命令**：图标用于打开该卷，不提供安全删除；拔盘前请继续使用系统托盘的安全删除功能。
- **图标名称不含冒号**，因为 Windows 文件名不允许，所以是 `U 盘 (F)` 而不是 `U 盘 (F:)`
  （括号前的部分是系统为该驱动器提供的本地化名称）。
- **更新后需重启资源管理器**：命名空间扩展 DLL 被 shell 加载后不会卸载，因此新版本要重启
  资源管理器才会生效。

## 背景：为什么需要自己的 COM 服务器

曾在本机评估「仅通过注册表把磁盘注册为命名空间项」的做法（`Desktop\NameSpace\{CLSID}` 加
shell 自带的「文件夹快捷方式」实例类 `{0AFACED1-E828-11D1-9187-B532F1E9575D}`）。该项确实会
出现、可自定义名称、能通过 `DefaultIcon` 显示磁盘自己的图标（取自 `SHGFI_ICONLOCATION`，
光盘可得 `autorun.inf` 中声明的图标），也没有快捷方式箭头，但**打不开磁盘**：`InvokeVerb('open')`
没有任何反应，因为没有对象把调用委派给真实文件夹。测试过的若干注册表变体（同一 GUID，逐个创建、
用 `InvokeVerb('open')` 验证、逐个清理）如下：

| 变体 | Instance CLSID | ShellFolder Attributes | 结果 |
| --- | --- | --- | --- |
| V1 | `{0AFACED1-…}` Folder Shortcut | `0xF080004D` | 出现，但打不开 |
| V2 | `{0E5AAE11-…}` DelegateFolder | `0xF080004D` | 完全不显示 |
| V3 | Folder Shortcut | `0xF080004D \| BROWSABLE` | 完全不显示 |
| V4 | DelegateFolder | `0xF080004D \| BROWSABLE` | 完全不显示 |
| V5 | DelegateFolder | `… + FolderValueFlags=0x28` | 完全不显示 |
| V6 | DelegateFolder | `SFGAO_FOLDER` | 完全不显示 |

原因：`{0E5AAE11-…}` 只接受 `TargetKnownFolder`（本机的「下载」「公用文件夹」等现成实例都是
这种写法），不接受 `TargetFolderPath`；而 `{0AFACED1-…}` 这个「文件夹快捷方式」在本机只会
渲染，不会委派。因此可用的命名空间项必须自带一个实现 `IShellFolder` 的 COM 服务器——这正是
「此电脑」「回收站」的实现方式。

## 命名空间项（默认模式）

默认模式不再写入 `.lnk` 文件，而是为每个盘符注册一个 **shell 命名空间项**：

- 该项通过 `HKCU\Software\Classes\CLSID\{CLSID}` 加 `Desktop\NameSpace\{CLSID}` 注册，
  CLSID 由盘符推导；
- `InProcServer32` 指向**模组自身的 DLL**。模组本来就运行在资源管理器进程内，DLL 导出
  `DllGetClassObject`，因此 shell 可以直接创建该对象，不需要额外的安装程序或辅助 DLL；
- 该 COM 类实现了一个**转发型 `IShellFolder`**：`Initialize` 之后从 `TargetFolderPath` 读出
  卷路径，并把 `EnumObjects`、`BindToObject`、`CreateViewObject`、`GetUIObjectOf`、
  `GetDisplayNameOf` 全部转发给该卷真实的文件夹对象。因此枚举、打开、右键菜单与图标都与真实
  目录一致，而桌面上是纯命名空间项：**没有 `.lnk` 文件、没有快捷方式箭头**；
- 图标来自 `DefaultIcon`，取磁盘自己的图标位置。

本机实测：该项即时出现，可枚举出 `G:\` 的 8 个条目，打开后的窗口就是这 8 个文件夹。关键细节：
`ShellFolder\Attributes` 必须包含 `SFGAO_HASSUBFOLDER`（使用 `0x28000000`），否则资源管理器
不会打开该项。

清理策略：拔出盘 → 删除该 CLSID；禁用或卸载模组 → 删除全部（含 `HKCU\Software\Classes\CLSID`
下的键）；被强制终止 → 下次启动时清除孤儿键。便携设备（手机）仍使用桌面快捷方式，因为 MTP
设备没有可以写入 `TargetFolderPath` 的文件系统路径。

注意：命名空间扩展 DLL 被资源管理器加载后不会卸载，更新模组需要重启资源管理器。

## 许可

本模组以 GNU 通用公共许可证第 3 版（GPLv3）发布，全文见 [LICENSE](LICENSE)。

本模组按「现状」提供，不附带任何担保；详细条款见许可证正文。

