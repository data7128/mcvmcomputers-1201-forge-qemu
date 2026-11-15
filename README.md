# VM Computers — 1.20.1 Forge + QEMU (Android ARM64)

在 Minecraft **1.20.1 + Forge 47.2.0** 上运行真实虚拟机，后端由 VirtualBox 换成了 **QEMU**，
因此可以在 **Android ARM64**（FCL / PojavLauncher）上直接跑起来，不需要 VirtualBox。

## 来源与授权

本项目是三层上游的继续改造，遵循 **GPL-3.0-or-later**：

1. 原始模组 [VM Computers](https://github.com/Delta2Force/MCVmComputers) by Delta2Force（Fabric / 1.16.5）
2. 1.20.1 粉丝移植 [*NEW* VM computers](https://github.com/Gamma-mods-studio/new-vm-computers) by FB527（分支 `(Deprecated)-1.20.1-forge`）
3. 本项目：把 VirtualBox 后端替换为 QEMU 后端（`newvmcomputers/vm/`），并捆绑 Android ARM64 静态 QEMU 二进制

原始 1.16.5 版 QEMU 移植见 [mcvmcomputers-android](https://github.com/data7128/mcvmcomputers-android)。

## 与原版的关键差异

| | 上游 1.20.1 Forge 版 | 本项目 |
|---|---|---|
| 虚拟化后端 | VirtualBox 6.1（需 PC 装 VB + vboxwebsrv） | **QEMU**（二进制随 jar 分发） |
| Android 可运行 | 否（Android 无 VirtualBox） | **是**（ARM64 静态 QEMU） |
| 画面获取 | VBox API `takeScreenShotToArray` | 内置 VNC 客户端 + 纯 Java PNG 编码 |
| 硬盘格式 | .vdi / .vmdk | **.qcow2**（`qemu-img create`） |
| 依赖 `java.awt` | VNC 路径用 `BufferedImage` + `ImageIO` | 无 AWT 依赖（`PngEncoder` 手写 Deflater/CRC32） |

## 依赖

- Minecraft **1.20.1**
- **Forge 47.2.0**
- 无需 VirtualBox

## 构建

CI 自动完成（`.github/workflows/forge-qemu-build.yml`）：

1. `build-qemu`：在原生 ARM64 runner 的 Alpine 容器内静态编译 QEMU 8.2.4
   → `qemu-system-x86_64` + `qemu-img`（aarch64 静态 ELF）
2. `build-mod`：把二进制塞进 `src/main/resources/assets/qemu/`，JDK 17 + ForgeGradle 6 打包

本地构建（已有 QEMU 二进制时）：

```bash
mkdir -p src/main/resources/assets/qemu
cp <qemu-system-x86_64> <qemu-img> src/main/resources/assets/qemu/
./gradlew build
```

产物在 `build/libs/`。

## 在手机上使用（FCL）

1. FCL 里装 **1.20.1 Forge**（Forge ≥ 47.2.0）
2. jar 放进 `.minecraft/mods/`
3. 启动，进设置向导 → 填最大内存 / 显存 → 完成（不再要求 VirtualBox 目录）

首次运行会把 QEMU 二进制解压到 `<gameDir>/vm_computers/bin/` 并 `chmod +x`。

**兜底**：若应用私有目录不允许执行二进制，装 Termux 后执行

```bash
pkg install qemu-system-x86-64-headless qemu-utils
```

模组会自动探测 `/data/data/com.termux/files/usr/bin/` 下的 `qemu-system-x86_64` 与 `qemu-img`。

## 玩法（沿用上游）

- 用**订购平板（Ordering Tablet）**买 PC 机箱、主板、内存、显卡、显示器、硬盘
- 手持硬盘右键 → 创建新硬盘（本版为 `.qcow2`）或选择已有
- ISO 放 `.minecraft/vm_computers/isos`，在机箱菜单里选择
- 机箱菜单点 **Power On** 开机；右键显示器进入交互，按屏幕右上角提示的按键退出

## 已知限制

- 纯软件模拟（TCG），**无 KVM/HVF 加速**，客户机性能有限；跑 Windows 很慢，跑 Tiny Core / Alpine 这类轻量 Linux 体验尚可
- 画面走 VNC Raw 编码，带宽/CPU 开销偏高，默认 50ms 一帧
- 网络：QEMU 启动参数带 `-net none`（避开 Android 上缺失网卡 ROM 的问题），客户机暂无网络
- 声音未实现

## 日志排查

FCL 日志 `.minecraft/logs/latest.log`，关键前缀：

- `[vmcomputers]` — QEMU 启动参数、VNC 连接、首帧分辨率
- `[VMComputers]` — 模组侧生命周期与错误

## 新增代码

```
src/main/java/newvmcomputers/vm/
├── QemuBackend.java      # 门面：启动/停止/输入/抓帧
├── QemuProcess.java      # qemu-system-x86_64 子进程封装
├── VncClient.java        # RFB 3.3 客户端（Raw 编码）
├── PngEncoder.java       # 纯 Java PNG 编码（Android 无 ImageIO）
├── AndroidBinLoader.java # 从 jar 解压二进制并 chmod
└── X11Keysym.java        # 扫描码 -> X11 keysym
```
