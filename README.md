# PCL_patch

> 一个单文件、离线运行的本地补丁工具：解除 **Plain Craft Launcher 2（PCL2）** 客户端中「Minecraft 下载」功能的限制。

<p align="center">
  <img alt="platform" src="https://img.shields.io/badge/platform-Windows%2010%20%2F%2011-0078D4">
  <img alt="runtime" src="https://img.shields.io/badge/runtime-.NET%20Framework%204.8-512BD4">
  <img alt="size" src="https://img.shields.io/badge/size-1.2%20MB-lightgrey">
  <img alt="network" src="https://img.shields.io/badge/network-none-success">
</p>

---

## ⚠️ 重要声明

> **本仓库不提供、也不分发任何 PCL2 的二进制文件（无论原版还是补丁版）**， 请自行从官方渠道获取 PCL2 后用本工具打补丁

- 本仓库**只有** `pcl_patch.exe`、本次逆向分析的会话记录（session log）以及这份 README。
- 工具本身不含、不下载、不内置任何 PCL2 代码或可执行文件；它只就地修改**你自己电脑上**已有的 `Plain Craft Launcher 2.exe`。
- 请遵守你所在地区的法律法规以及 PCL 的许可与使用条款；因使用本工具产生的一切后果由使用者自行承担。

---

## 这是什么

PCL2 正式版 2.99.0.0 在源码层面设置了：

```vb
' Plain Craft Launcher 2/Modules/ModSecret.vb
Public Const RestrictMcDownload As Boolean = True
```

这会导致启动器在「下载」页无法加载原版版本列表、无法安装整合包，并提示*“为遵守相关法律法规，PCL 暂时不再提供 MC 下载功能”*。

`pcl_patch.exe` 会在本地把这一限制对应的机器码改回正常逻辑，使下载功能恢复可用。整个过程**完全离线**，不联网、不上传任何数据。

---

## 使用方法

### 1. 准备

- Windows 10 / 11（需要 .NET Framework 4.8，PCL2 本身就依赖它，通常已具备）
- 从**官方渠道**获取的 `Plain Craft Launcher 2.exe`

### 2. 打补丁

把 `pcl_patch.exe` 放到与 `Plain Craft Launcher 2.exe` **同一个文件夹**，然后：

| 方式 | 操作 |
|---|---|
| **推荐** | 双击 `pcl_patch.exe` |
| 指定路径 | `pcl_patch.exe "D:\Games\PCL\Plain Craft Launcher 2.exe"` |
| 拖拽 | 把 `Plain Craft Launcher 2.exe` 拖到 `pcl_patch.exe` 图标上 |
| 只检测 | `pcl_patch.exe --check` |

运行后会看到类似输出：

```text
PCL2 下载限制解除补丁  v1.0

目标文件：D:\Games\PCL\Plain Craft Launcher 2.exe
程序集  ：Plain Craft Launcher 2 2.99.0.0
限制字段：True
下载加载器：仍是抛异常的占位加载器

已应用以下修改：
  - ModSecret.RestrictMcDownload: True -> False
  - PageDownloadInstall.LoaderInit: 占位加载器 -> ModDownload.DlClientListLoader
  - ...

完成！原件已备份为：Plain Craft Launcher 2.exe.bak
现在可以启动 PCL，下载页应恢复正常。
```

### 3. 退出码（便于脚本调用）

| 退出码 | 含义 |
|---|---|
| `0` | 打补丁成功，或文件已经是补丁版 |
| `1` | `--check`：文件仍是受限版 |
| `2` | 找不到目标文件，或目标不是 PCL2 |
| `3` | 版本不匹配，缺少必要字段/方法 |
| `4` | 写入失败（通常是 PCL 正在运行，文件被占用） |

---

## 工具做了什么

| # | 位置 | 修改内容 |
|---|---|---|
| 1 | `ModSecret.RestrictMcDownload` | 常量值 `True → False` |
| 2 | `PageDownloadInstall.LoaderInit` | 抛异常的占位加载器 → 真实的 `ModDownload.DlClientListLoader` |
| 3 | `PageDownloadModify.LoadInstanceFinished` | `CardMinecraft` 由 `Collapsed` 恢复为 `Visible` |
| 4 | `PageDownloadResourceDetail.InitializeStackResourceVersions` | 整合包版本列表恢复「安装」入口 |
| 5 | `ModModpack.ModpackInstall(File, …)` | 恢复 7 种整合包格式（CurseForge / HMCL / MMC / MCBBS / Modrinth / 启动器包 / 普通压缩包）的安装分派 |
| 6 | `ModModpack.ModpackInstall()` | 恢复「选择整合包文件」对话框并在线程中安装 |

> 说明：这些补丁只是把被编译器删掉的代码按原语义补回，**没有**新增功能、没有修改网络地址、没有改变启动器的其他行为。

---

## 安全性与可验证性

| 机制 | 说明 |
|---|---|
| **只改本机文件** | 仅读写你指定的那个 exe，不写入注册表、不创建服务、不联网 |
| **只认 PCL2** | 校验程序集名必须为 `Plain Craft Launcher 2`，其他文件直接拒绝 |
| **自动备份** | 原文件改名为 `Plain Craft Launcher 2.exe.bak` |
| **原子替换 + 自检** | 先写临时文件 → 重新载入校验（常量 + 加载器）→ 通过后才替换；任一步失败自动回滚 |
| **幂等** | 已经是补丁版时直接提示并退出，不会重复改写 |
| **占用保护** | 检测到文件被占用（PCL 正在运行）时中止，原文件不受影响 |
| **版本保护** | 版本不是 2.99.0.0 时给出警告；指令特征不匹配则中止，绝不“盲改” |

关于杀毒软件 / SmartScreen：本工具未做代码签名，可能出现「未知发布者」或启发式误报。可以先运行 `pcl_patch.exe --check` 观察行为，或用下方哈希核对文件。

当前版本 `pcl_patch.exe` 的 SHA-256：

```text
F67AF3461FA15F03AB147BAEC60044C4AFBFDA0A475D5A4B8920B17754819ACB
```

> 哈希仅对应本仓库当前发布的这一版；自行重新编译会得到不同结果。

---

## 适用版本

| 项目 | 说明 |
|---|---|
| 目标版本 | PCL2 **2.99.0.0**（`AssemblyVersion 2.99.0.0`，MVID `926d447e-daff-4169-8c64-adc3d37d5f78`） |
| 架构 | AnyCPU（x64 / x86 系统均可运行） |
| 运行环境 | .NET Framework 4.8 |
| 其他版本 | 会先警告再按指令特征尝试匹配；特征不符即中止，不会破坏文件 |

PCL 更新后本补丁会失效（新版本是官方受限版），重新运行一次 `pcl_patch.exe` 即可。

---

## 仓库内容

```text
PCL_patch/
├── pcl_patch.exe     # 补丁工具（单文件，已内嵌依赖）
├── session-log.md    # 本次逆向分析的完整会话记录（文件名以你导出的日志为准）
└── README.md         # 本文件
```

> 仓库**不包含** PCL2 本体、补丁后的 PCL2，也不包含补丁器源码。

---

## 常见问题

**Q：打补丁后 PCL 还能正常更新吗？**
A：能，但官方更新会把 exe 换回受限版，需要重新运行 `pcl_patch.exe`。

**Q：会不会影响存档、账号、Mod？**
A：不会。补丁只改启动器自身的下载相关代码，不触碰 `.minecraft` 目录里的任何内容。

**Q：为什么提示“该文件已经是补丁版”？**
A：说明这个 exe 已经打过补丁，无需重复处理。

**Q：支持其他 PCL 版本 / 其他启动器吗？**
A：不支持其他启动器。其他 PCL 版本会尝试按特征匹配，失败即安全中止。

**Q：需要联网吗？**
A：不需要。工具全程离线，不会请求任何网络资源。

**Q：为什么仓库里只有一个 exe，怎么让人放心？**
A：可以先用 `--check` 观察它只读不写；仓库中的 session log 记录了完整的反编译、定位与验证过程（含 IL 级证据），可供复核。

---

## 技术细节（简述）

- 工具用 `dnlib` 读写 .NET 元数据与 CIL，`dnlib.dll` 以嵌入资源打包，运行时通过 `AppDomain.AssemblyResolve` 加载，因此是**单文件**。
- 改写范围严格限制在 6 个已知位置；写出后会重新载入文件，校验关键特征通过才替换原件。开发阶段另对整个程序集的 5999 个方法体做过 IL 栈校验（0 错误），确保未引入非法指令。
- 补丁后的程序集保持原版本号、MVID、类型/方法数量与资源不变，仅目标方法的指令序列被替换。

---

## 免责声明

- 本项目与 PCL（Plain Craft Launcher）作者 **Meloong-Git** 及贡献者**没有任何关联**，也未获得其授权或认可。
- 本工具仅用于学习、研究与个人使用；请勿用于商业用途或再分发修改后的 PCL 二进制文件。
- 使用本工具可能违反你所在地区的法律法规或相关服务条款，请在**下载后自行判断并承担风险**。
- 作者不对因使用本工具导致的任何数据丢失、账号异常、软件损坏或其他后果承担责任。

---

## 致谢

- [Meloong-Git/PCL](https://github.com/Meloong-Git/PCL) —— PCL2 及其开源代码
- [dnlib](https://github.com/0xd4d/dnlib) —— .NET 元数据读写库
- [ILSpy / ICSharpCode.Decompiler](https://github.com/icsharpcode/ILSpy) —— 反编译引擎
