# RustDesk 自定义编译设置

本文档说明如何为 `rustdesk` 仓库配置 Android 签名、GitHub Actions secrets 以及触发编译流程。

## 1. 概述

本仓库使用 GitHub Actions 进行自动构建。对于 Android 正式签名包，需要在仓库中配置额外的 secrets；对于自建 ID/API 服务器，也需要配置对应的环境变量。

## 2. Android 签名配置

Android 签名相关的 4 个 secrets 说明：

| Secret | 含义 |
| --- | --- |
| `ANDROID_SIGNING_KEY` | keystore 文件 (`.jks`) 的 Base64 编码 |
| `ANDROID_ALIAS` | keystore 中的证书别名，对应 `-alias` |
| `ANDROID_KEY_STORE_PASSWORD` | keystore 文件密码 |
| `ANDROID_KEY_PASSWORD` | 别名对应密钥密码。对于现代 PKCS12 格式，通常与 keystore 密码相同 |

### 2.1 生成 keystore

建议在本机用户目录下生成 keystore，不要在仓库目录里生成，以避免误提交。

在 Windows PowerShell 中运行：

```powershell
cd ~
keytool -genkeypair -v -keystore rustdesk-release.jks -keyalg RSA -keysize 2048 -validity 10000 -alias rustdesk
```

在 Linux/macOS 终端中运行：

```bash
cd ~
keytool -genkeypair -v -keystore rustdesk-release.jks -keyalg RSA -keysize 2048 -validity 10000 -alias rustdesk
```

执行后会交互式提示输入：

- `输入密钥库口令`：设置一个密码并记住它。这就是 `ANDROID_KEY_STORE_PASSWORD`，对于 PKCS12 也将作为 `ANDROID_KEY_PASSWORD`
- `您的名字与姓氏是什么` 等信息：可随意填写，例如公司名
- `-alias rustdesk`：这就是 `ANDROID_ALIAS`，可根据需要修改
- `-validity 10000`：有效期约 27 年

### 2.2 转换 keystore 为 Base64

在 Windows PowerShell 中运行：

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$HOME\rustdesk-release.jks")) | Set-Clipboard
```

在 Linux/macOS 终端中运行：

```bash
base64 "$HOME/rustdesk-release.jks" | xclip -selection clipboard
```

如果没有 `xclip`，也可以直接输出到终端并复制：

```bash
base64 "$HOME/rustdesk-release.jks"
```

该命令会将 Base64 结果复制到剪贴板或输出到终端，避免将敏感内容写入文件。

### 2.3 添加 GitHub Secrets

打开仓库的 GitHub Secrets 页面：

https://github.com/tashifool/rustdesk/settings/secrets/actions

逐个添加以下 secrets：

| Name | Value |
| --- | --- |
| `ANDROID_SIGNING_KEY` | 粘贴剪贴板中的 Base64 字符串 |
| `ANDROID_ALIAS` | `rustdesk` |
| `ANDROID_KEY_STORE_PASSWORD` | 第 1 步设置的 keystore 密码 |
| `ANDROID_KEY_PASSWORD` | 与上面相同 |

### 2.4 重新触发构建

新增 `ANDROID_SIGNING_KEY` 后，工作流会自动走签名分支，构建完成后会使用你的 keystore 对 APK 进行正式签名。

触发构建的示例命令：

```bash
git tag android-v1.1
git push origin android-v1.1
```

### 2.5 重要提醒

- keystore 文件和密码必须备份好。Android 应用升级要求旧签名与新签名一致，密钥丢失后只能通过更换包名重新发布，旧用户无法直接覆盖升级。
- 不要把 `.jks` 文件提交到仓库。建议保存在用户目录或安全存储中。
- 之前 `android-v1.0` 构建的是 debug 签名 APK，和正式签名 APK 签名不同，不能直接覆盖安装，需先卸载旧包再安装新包。

## 3. GitHub Actions 额外 Secrets

如果你使用自建 `hbbs`/`hbbr` 或 RustDesk Server Pro，需要配置以下额外 secrets：

| Secret | 填写内容 | 示例 |
| --- | --- | --- |
| `RENDEZVOUS_SERVER` | 自建 hbbs(ID 服务器) 的域名或 IP。默认端口 `21116` 可省略。 | `rustdesk.example.com` / `1.2.3.4` |
| `RS_PUB_KEY` | hbbs 生成的公钥，即服务器上 `id_ed25519.pub` 文件的内容（Base64 文本） | `OeVuKk5nlHi...qmBw=` |
| `API_SERVER` | API 服务器地址。只有使用 RustDesk Server Pro 或自建 API 时才需配置。普通开源版 hbbs/hbbr 不需要。 | `http://rustdesk.example.com:21114` |

`RS_PUB_KEY` 可在部署 hbbs 的机器上找到，通常与 Docker 挂载目录 `/root` 或 `data` 中的 `id_ed25519.pub` 文件内容一致。它与 hbbs 启动日志中打印的 `Key:` 内容相同。

## 4. 触发编译

使用 Git tag 触发对应平台的构建：

```bash
# Android
git tag android-v1.2
git push origin android-v1.2

# Windows
git tag win64-v1.1
git push origin win64-v1.1
```

## 5. 其他注意事项

- 这个 README 主要用于说明定制构建所需的 GitHub Actions secrets。
- 如果你需要完整的项目开发或运行说明，请参考仓库中的其他文档或 `README` 的上游版本。

## 6. RustDesk 项目概览

RustDesk 是一个远程桌面解决方案，本仓库以 Rust 为核心，Flutter 作为当前 UI 层。

### 6.1 核心目录

- `src/`：Rust 应用主代码
  - `src/server/`：音频、剪贴板、输入、视频、网络相关服务
  - `src/platform/`：平台特定实现
  - `src/ui/`：旧版 Sciter UI（已退役/不推荐继续开发）
- `flutter/`：当前跨平台界面实现
  - `flutter/lib/desktop/`：桌面 UI
  - `flutter/lib/mobile/`：移动端 UI
  - `flutter/lib/common/`、`flutter/lib/models/`：共享代码块
- `libs/`：共享库与平台插件
  - `libs/hbb_common/`：配置、协议、公共工具
  - `libs/scrap/`：屏幕采集
  - `libs/enigo/`：输入控制
  - `libs/clipboard/`：剪贴板支持
  - `libs/virtual_display/`：虚拟显示
  - `libs/remote_printer/`：远程打印
- `res/`：资源文件、图标、桌面条目、服务文件
- `build.py`：多平台打包与构建辅助脚本
- `Cargo.toml`：Rust 包配置与依赖声明

### 6.2 关键组件

- `src/rendezvous_mediator.rs`：RustDesk 与服务器通信的远端协议入口
- `libs/scrap/`：不同平台的屏幕采集实现
- `libs/enigo/`：跨平台输入仿真
- `src/server/`：音视频、输入、网络传输、服务管理
- `libs/hbb_common/`：配置与协议共享逻辑

## 7. 开发与构建流程

### 7.1 先决条件

- Rust 工具链：建议使用 `rustup` 安装并保持最新稳定版（`rustc 1.75+`）
- Cargo：Rust 自带
- Python：`build.py` 脚本依赖 Python
- Flutter：如果需要构建 UI 或打包 Flutter 版本
- 平台特定依赖：Windows、macOS、Linux、Android 等平台各自的构建工具链

### 7.2 快速编译

在仓库根目录执行：

```bash
cargo build --release
```

如果你希望运行已编译程序：

```bash
cargo run --release
```

### 7.3 Flutter UI 构建

如果你要构建 Flutter 前端，请先进入 `flutter/` 目录：

```bash
cd flutter
flutter pub get
flutter build windows --release
# 或
flutter build linux --release
# 或
flutter build macos --release
```

### 7.4 使用 `build.py` 辅助构建

`build.py` 用于统一构建 Rust 端和 Flutter UI，并支持多平台打包：

```bash
python build.py --flutter
```

常见选项：

- `--flutter`：构建 Flutter 包
- `--hwcodec`：启用硬件编码相关功能（Linux 可能需要 `libva-dev`）
- `--vram`：启用 VRAM 相关支持（当前仅 Windows）
- `--portable`：构建 Windows 便携版
- `--unix-file-copy-paste`：启用 Unix 剪贴板文件复制粘贴功能
- `--skip-cargo`：仅打 Flutter 版本，不执行 Rust 编译（目前仅 Linux 支持）

### 7.5 典型平台构建示例

- 构建 Windows Flutter 版本：

```bash
python build.py --flutter
```

- 构建 macOS Flutter 版本：

```bash
python build.py --flutter --screencapturekit
```

- 构建 Linux Flutter 版本：

```bash
python build.py --flutter
```

### 7.6 代码检查与测试

建议在改动后进行基本检查：

```bash
cargo test
cargo clippy --all-targets --all-features
```

## 8. 贡献与开发建议

- 遵循 Rust 项目惯例，避免无谓的 `.clone()` 和 `unwrap()`，优先使用 `Result`/`?`
- Async 代码中不要在 `.await` 期间持有锁，也不要创建嵌套 Runtime
- 仅改动必要代码，避免无关重构
- 现有开发工作流程以 `flutter/` 为当前 UI，`src/ui/` 为 legacy 旧版界面
