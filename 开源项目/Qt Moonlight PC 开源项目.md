## Qt Moonlight PC 开源项目

学习Qt和音视频的必不可少的开源项目之一。

### 1、介绍

- “Moonlight PC” 是该客户端对应 PC 的版本 —— 用于将主机 (host) 上的游戏或桌面，通过网络串流 (streaming) 到其他设备 (client) 来运行/显示。
- 除 PC 之外，该项目还有面向 Android、iOS 等其他平台的客户端 (client) 实现。 

简言之，moonlight-qt 是 “为 PC (Windows/macOS/Linux/Steam Link 等) 提供的 Moonlight 客户端版本”。

### 2、编译要求

推荐使用 **Qt 6.7 SDK** 或更高 (在 Windows /macOS 上)；也支持 Qt 5.12+ (在 Linux 上) 。 

Windows 构建推荐使用 Visual Studio 2022 (MSVC)；MinGW 不被官方支持。 

Linux 下一般需要 FFmpeg (4.0 以上), SDL2, OpenGL / EGL /VA‑VA/Vulkan (视渲染方式) 等视频 / 解码 /图形库。

构建步骤 (概要)：

1. `git submodule update --init --recursive`
2. 使用 Qt Creator 或 `qmake6 / qmake` + make/make‑release 构建
3. 如果需要发布包 (installer / portable / embedded 等)，使用仓库中的脚本 (如 Windows 下 `scripts/build-arch.bat` + `scripts/generate-bundle.bat` / macOS 下 `scripts/generate-dmg.sh`) 。

### 3、功能 / 特性 (Feature Highlights)

moonlight‑qt / Moonlight PC 支持很多强大的流媒体 / 游戏 /远程桌面功能 — 包括：

- 硬件加速的视频解码 (hardware‑accelerated video decoding)，支持 Windows、macOS、Linux。 
- 支持多种视频编解码格式 (codecs)：H.264, HEVC, 以及 AV1（如果 host 支持 AV1 + 使用对应 server，如 Sunshine）
- 支持高画质流 (包括 HDR streaming) 与高帧率 (最高可达 120 FPS，在合适条件下) 流式传输。 
- 支持 7.1 声道音频 (surround sound)、多通道音频输出。
- 支持 Gamepad（手柄）输入，包括力反馈 (force feedback) 和 motion／motion control，最多支持 16 个玩家手柄 (取决 host 和客户端支持情况) 。
- 支持指针 (鼠标) 捕获 (pointer capture) — 对于游戏输入，以及直接的鼠标控制 (适合远程桌面 / 非游戏使用)。
- 支持将系统级快捷键 (system‑wide shortcuts)，例如 Alt+Tab 之类 — 转发给 host（远端主机）。
- 支持非常广泛的平台环境：Windows, macOS, Linux, Steam Link, Raspberry Pi (甚至包括 ARM 架构 / Debian /其他嵌入式设备) —— 官方提供多种构建 /打包 /安装方式 (例如 Snap / Flatpak / AppImage / Debian 包 / 通用 ARM 包 / Steam Link 等) 。 

此外，Moonlight 的目标是让你 “不必在每个设备上都安装游戏” — 只要主机 (host) 有游戏 (或桌面环境)，使用 Moonlight 就能把画面 + 输入流 (video + input) 串流到另一个设备上，从而“远程玩游戏 /远程使用桌面 /远程工作 /远程娱乐”。 

### 4、使用注意 / 兼容 & 要求

- 要使用 full‑feature (例如 AV1 解码 / 高帧率 / HDR /多控 /手柄) — host (主机) 和 client (你的 PC /设备) 都需要有支持的 GPU / 解码器 /网络状况。
- 如果 host 使用传统 NVIDIA GeForce Experience + GameStream，则只对 NVIDIA GPU 合法；如果想用 AMD / Intel GPU 或更通用场景，通常需要使用第三方 host 实现 (例如 Sunshine) — 这样可以支持更多硬件。 
- 网络延迟 /带宽 /稳定性对串流质量影响很大 — 特别是高分辨率、HDR、120 FPS、高码率 /多个玩家输入的时候。

### 5、使用场景 / 优点

Moonlight PC (moonlight‑qt) 的最大优势在于：

- **把“高性能 PC + 好 GPU + 游戏库 +驱动” 的能力，通过网络“串流(Streaming)” 给性能一般 /硬件弱 /便携设备** —— 这样你不需要在每台设备上都配置游戏，只用一台主机器即可。
- **跨平台 + 广泛兼容** —— Windows / macOS / Linux / 嵌入式 / ARM / Raspberry Pi / Steam Link / 等多种平台均支持。
- **低延迟 + 高画质 + 手柄 + 多输入支持** —— 如果网络 /硬件条件合适，你几乎可以像在主机一样远程玩游戏、远程桌面、远程工作。
- **开源 & 社区驱动 / 无广告 / 无强制付费** —— 对研究、二次开发、定制、贡献很友好。

因此，这种架构很适合 “家庭内 / 局域网游戏串流 /远程桌面 / 家庭云 + 游戏 + 多设备” 方案。