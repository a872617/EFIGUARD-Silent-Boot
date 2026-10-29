# 改动说明 / PATCH

本构建 = [EfiGuard v1.2.1](https://github.com/Mattiwatti/EfiGuard) 上游源码，
EDK2 `RELEASE` / `X64` 构建（`build -a X64 -t VS2019 -p EfiGuardPkg/EfiGuardPkg.dsc -b RELEASE`），
外加下列"去界面化"改动。功能路径（bootmgr / winload / ntoskrnl 补丁、DSE bypass、SetVariable hook）**一行没动**。

---

## 1. 去掉所有日志输出

上游 loader 与驱动里有大量 `Print()`，构建时被逐条删除（不是靠 `DEBUG` 宏，`Print` 是无条件输出）：

| 位置 | 删除的内容 |
|---|---|
| `EfiGuardDxe.c · EfiGuardInitialize` | 整块 `SetConsoleTextColour(EFI_GREEN, TRUE)` + ASCII 横幅 + `RtlSleep(1500)` |
| `EfiGuardDxe.c · HookedLoadImage` | `[HookedLoadImage] %S %S`、`PrintLoadedImageInfo()` 的 `[+]` 输出、`RtlSleep(500)` |
| `EfiGuardDxe.c · ExitBootServicesEvent` | `PrintKernelPatchInfo()` 的文字输出、`Successfully patched ntoskrnl.exe` |
| `Loader.c · UefiMain` | `Press <HOME> to configure EfiGuard...`，并把 `WaitForKeyWithTimeout(1500)` 改成 `(0)`（不再等 1.5 秒） |
| `Loader.c · SetHighestAvailableTextMode` | 函数体只剩 `QueryMode` 查询循环，`SetMode` / `ClearScreen` / `EnableCursor` 三行删除 |
| `Loader.c · TryBootOptionsInOrder` | `Booting "%S"...` 的输出 |

**只删文字是不够的** —— `SetConsoleTextColour(EFI_GREEN, TRUE)` 的第二个参数是"清屏"，
删掉 `Print` 之后它仍然会把固件画好的开机 Logo 清掉。这是下面两处补丁存在的原因。

---

## 2. 驱动：`ExitBootServices` 时不再清屏

### 现象

Windows 内核接管前的最后一步会调用 `ExitBootServices`，EfiGuard 在这里注册了回调，
原本会先 `ClearScreen()` 再打印内核补丁信息。删掉文字后，**清屏还在**：

```
主板 Logo(固件) → bootmgr 画 Logo+转圈 → ExitBootServices → 被清成黑 → 只剩黑底转圈
```

这就是"Logo 不能和转圈同时存在"的直接原因 —— 清屏发生在 bootmgr 画完之后，
而 winload/ntoskrnl 之后是直接往帧缓冲里画帧的，不会再把 Logo 画回来。

### 字节改动

```
文件  EFI/Boot/EfiGuardDxe.efi
偏移  0x1B72
改前  48 8B 41 30 FF D0      mov rax,[rcx+0x30] ; call rax   (ConOut->ClearScreen)
改后  90 90 90 90 90 90
```

调用点上下文（`ConOut` 是 `EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL`，`+0x28` = SetAttribute，`+0x30` = ClearScreen）：

```
0x1B5A  and edx,0x50 / or rdx,0x20 / shr rdx,4     ; SetConsoleTextColour(EFI_GREEN, TRUE) 的属性计算
0x1B65  call rax                                   ; SetAttribute
0x1B6E  mov rcx,[rax+0x40]                         ; gST->ConOut
0x1B72  mov rax,[rcx+0x30] / call rax              ; ClearScreen   ← 改成 NOP
0x1B78  cmp byte ptr [...],r12b                    ; WaitForKeyPress？
```

### 源码级改法

```c
- SetConsoleTextColour(EFI_GREEN, TRUE);
+ SetConsoleTextColour(EFI_GREEN, FALSE);   // 或整块删掉
```

---

## 3. Loader：不再 `EfiBootManagerConnectAll()`

### 现象

`UefiMain()` 第一句就是 `EfiBootManagerConnectAll()`，它会：

- 把 `ConOut/ConIn/ErrOut` 的默认控制台设备重新连接一遍（可能触发一次 `GOP->SetMode()` → **提前黑屏**，
  并且让显示器重新同步一次，实测一段黑屏 ~2 秒）；
- 对所有 handle 递归 `ConnectController()` 并 `Dispatch()` 到派发不出新驱动为止（机器上盘多、设备多时很慢）。

### 字节改动

```
文件  EFI/Boot/bootx64.efi
偏移  0x1CD1
改前  E8 86 29 00 00        call 0x465C (EfiBootManagerConnectAll)
改后  90 90 90 90 90
```

### 为什么安全

loader 后面在真正 `LoadImage(bootmgfw)` 之前会调用
`EfiBootManagerConnectDevicePath(BootOptions[Index].FilePath, NULL)`，
按目标设备路径精确连接，不依赖前面这一次"全机重连"。实测（ASUS TUF B460M-PLUS + WD SN570）
启动正常、Windows 引导项枚举正常，黑屏由 ~5 秒降到 ~3 秒。

### 源码级改法

```c
- EfiBootManagerConnectAll();
```

---

## 4. 复现 / 校验

```powershell
Get-FileHash .\EFI\Boot\bootx64.efi      -Algorithm SHA256   # 065ECB25...C7A38A5
Get-FileHash .\EFI\Boot\EfiGuardDxe.efi  -Algorithm SHA256   # D09C059A...EF45468
```

从源码重建后，把上面两处 5/6 字节改成 `90`（NOP）即可得到行为一致的二进制
（偏移量取决于编译器/EDK2 版本，请用 disassembler 定位 `call [rcx+0x30]` 与
`call EfiBootManagerConnectAll` 的实际地址，不要照抄偏移）。

## 5. 残余黑屏说明

剩下的约 3 秒黑屏不是本补丁能消除的：

1. bootmgr 自己 `SetMode()` 到它要用的分辨率（原生启动约 1 秒）；
2. EfiGuard 读取启动项镜像并用 Zydis 扫描 + 打补丁 bootmgfw（约 1~2 秒）。

要彻底消掉只能由程序自己先 `SetMode` 再把 BGRT 里的 OEM Logo `Blt` 回帧缓冲，
让 bootmgr 认为"模式没变"而跳过切换。这会引入画面错位/花屏风险，本仓库不做。
