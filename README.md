# EFIGUARD 无日志无感启动

把 [EfiGuard](https://github.com/Mattiwatti/EfiGuard) 开机时那一屏绿色日志和随之而来的清屏全部去掉，
让 UEFI 引导看起来和原生 Windows 11 一样：

```
主板 Logo  →  Windows Logo + 转圈
```

全程没有文字、没有闪屏、没有"绿字黑底"。补丁功能（PatchGuard / DSE bypass）与原版完全一致。

> 本仓库只做"显示层"的裁剪，不含任何新增功能。原作者：**[Mattiwatti](https://github.com/Mattiwatti)**，
> 项目地址 [EfiGuard](https://github.com/Mattiwatti/EfiGuard)，请先给原作者点 star。

---

## 为什么需要它

EfiGuard 默认会：

1. 在驱动加载时把控制台切成绿字、`ClearScreen()` 掉固件画好的开机 Logo，并打印一张 ASCII 横幅；
2. 在 `ExitBootServices`（Windows 内核接管前的最后一刻）再次 `ClearScreen()`，然后滚动打印内核补丁信息；
3. 在 loader 里调用 `EfiBootManagerConnectAll()`，把所有设备（以及 ConOut/GOP）重新连一遍。

结果就是：**主板 Logo 消失 → 黑屏几秒 → Windows Logo 闪一下又没 → 只剩黑底转圈**。

把上面三处的"显示副作用"去掉之后，固件画的 Logo 会一直留到 Windows 接手，转圈直接叠在上面，
和原生启动一致。

---

## 改了什么

只动了两条指令（详见 [PATCH.md](PATCH.md)）：

| 文件 | 位置 | 改动 |
|---|---|---|
| `EFI/Boot/EfiGuardDxe.efi` | `0x1B72` | `ConOut->ClearScreen()` 调用 → `NOP` ×6 |
| `EFI/Boot/bootx64.efi`（loader） | `0x1CD1` | `EfiBootManagerConnectAll()` 调用 → `NOP` ×5 |

源码级等价改法：

```c
// EfiGuardDxe/EfiGuardDxe.c · ExitBootServicesEvent()
- SetConsoleTextColour(EFI_GREEN, TRUE);     // TRUE = 顺手清屏
+ SetConsoleTextColour(EFI_GREEN, FALSE);

// Application/Loader/Loader.c · UefiMain()
- EfiBootManagerConnectAll();
```

日志文字本身在这个构建里已经全部移除（横幅、`[HookedLoadImage]`、`Booting "..."`、
`Press <HOME> to configure`、内核补丁信息等）。

---

## 文件

```
EFI/Boot/bootx64.efi       49152   SHA256 065ECB2563A0D95B81D580EC22FC043CB5D7403B1C52545444C2F3247C7A38A5
EFI/Boot/EfiGuardDxe.efi  348160   SHA256 D09C059A9FD2C9049AFCF24A543F4755FE03C8989BF2F5FF140A5F530EF45468
```

`bootx64.efi` 就是改过的 loader，这是 EfiGuard 官方文档推荐的做法（把 `Loader.efi` 改名为 `bootx64.efi`）。
不需要 shim / HashTool / PreLoader，Secure Boot 关闭时直接可用。

---

## 安装

### 方式一：U 盘 / 独立分区（推荐）

1. 准备一个 **FAT32** 分区（U 盘或单独的小分区，≥ 64 MB）；
2. 把本仓库的 `EFI` 文件夹整个拷到该分区根目录，最终路径应为
   `X:\EFI\BOOT\BOOTX64.EFI` 和 `X:\EFI\BOOT\EfiGuardDxe.efi`；
3. 重启，按 F8/F11/F12 选该设备启动；
4. 看到主板 Logo 直接接 Windows 转圈即为成功。

### 方式二：装到现有 ESP，加独立启动项

```cmd
mountvol X: /S
xcopy /E /I EFI X:\EFI
bcdedit /copy {bootmgr} /d "EfiGuard"
bcdedit /set {新GUID} path \EFI\BOOT\BOOTX64.EFI
```

或者在 UEFI Shell 里：

```
bcfg boot add 0 FS0:\EFI\BOOT\BOOTX64.EFI "EfiGuard"
```

> loader 会自动顺着 `BootOrder` 找 Windows Boot Manager 并启动它，所以它**必须**放在能被固件读到的 FAT32 分区上。

### Secure Boot

本包**未签名**。Secure Boot 开启时无法加载，需要：

- 关闭 Secure Boot，或
- 自签名后把证书/哈希加进 `db`（可参考 [EfiGuard wiki](https://github.com/Mattiwatti/EfiGuard/wiki)）。

---

## 已知限制

- **仍有约 3 秒黑屏**，这段不是 EfiGuard 造成的，而是 Windows bootmgr 自己切换显示模式
  （固件 Logo 的分辨率 ≠ bootmgr 选的分辨率）+ EfiGuard 打补丁的固有耗时。原生启动同样有这一下，只是短（约 1 秒）。
  想要 0 黑屏只能由程序自己去 `SetMode` 并重绘 BGRT 里的 OEM Logo，风险远大于收益，本仓库不做。
- 只支持 **UEFI + 64 位 Windows**（Vista SP1 ~ Windows 11）。
- 机器上有多套 Windows 时，loader 会启动 `BootOrder` 里第一个 Windows 项，不一定是你想进的那个。
- **杀软 / 反作弊会报毒**。这本来就是一个 bootkit，会关闭 PatchGuard 和驱动签名强制。
- 不保证在所有主板上视觉效果一致（固件对 Logo 的处理方式不同）。

## 免责声明

仅供学习 UEFI 引导流程与驱动签名机制使用。**不要用于绕过反作弊、破解或任何违反软件许可的用途**，
因此造成的一切后果由使用者自负。作者不对任何数据丢失、无法启动、封号负责。

## 许可证

本仓库是 [EfiGuard](https://github.com/Mattiwatti/EfiGuard) 的修改版二进制，遵循 **GPLv3**，见 [LICENSE](LICENSE)。
源码获取：上游仓库 + [PATCH.md](PATCH.md) 中记录的改动即可完整复现本构建；
需要完整可编译源码树可通过 issue 索取。

## 致谢

- **[Mattiwatti](https://github.com/Mattiwatti)** —— EfiGuard 作者，本项目 99% 的工作都是他的
- [zyantific](https://github.com/zyantific) —— [Zydis](https://github.com/zyantific/zydis) 反汇编引擎
- [tianocore](https://github.com/tianocore/edk2) —— EDK2 与 UefiBootManagerLib
- [hfiref0x](https://github.com/hfiref0x) / [Fyyre](https://github.com/Fyyre) —— UPGDSED
- dude719 —— UEFI-Bootkit
