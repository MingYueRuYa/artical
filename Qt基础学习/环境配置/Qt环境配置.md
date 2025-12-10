## Qt 开发环境配置为 VS 和 Qt Creator（最全配置指南）

在实际项目中，**Qt Creator** 和 **Visual Studio（VS）** 是开发 Qt 程序最常用的两种 IDE。
 前者适合 Qt 专项开发、QML/UI 调试、跨平台；后者适合与大型 C++ 项目集成、团队协作、调试体验更强。

很多开发者希望同时具备两套环境，以便根据项目需求快速切换。本文将系统介绍如何正确安装和配置：

- **VS + Qt MSVC 套件 开发环境**
- **Qt Creator + Qt Kits 环境**
- 常见坑点与排错指南

适用于 Windows（macOS/Linux 可类似参考）。

### 1、为什么要同时使用 VS 和 Qt Creator？

| 需求                                              | 推荐 IDE      | 原因                      |
| ------------------------------------------------- | ------------- | ------------------------- |
| UI 设计（Qt Designer）、QML 调试                  | Qt Creator    | 官方统一支持、体验好      |
| 大型纯 C++ 工程，需利用 VS 插件、分析器、Profiler | Visual Studio | Windows 最强 C++ IDE      |
| 需要在一个项目用到 Qt + 第三方 C++ 库             | VS            | 项目组织更灵活            |
| 跨平台编译                                        | Qt Creator    | 支持 CMake + 多平台工具链 |

### 2. 正确安装 Qt（关键步骤）

#### 2.1 下载 Qt（强烈推荐使用官方在线安装器）

从 Qt 官网下载 Online Installer：

- 选择 **MSVC 对应的 Qt 版本**（例如：Qt 6.7 + MSVC 2019）
- 如果你想在 VS 中开发，**必须选择 MSVC 套件**，不要选 MinGW。
- 如果你只用 Qt Creator，可以两者都装。

#### 建议的安装项

- Qt → 6.x → MSVC 2019/2022 64-bit
- Developer Tools → CMake
- Developer Tools → Ninja
- Additional Libraries（可选）

### 3. 配置 Visual Studio + Qt

VS 本身并不直接支持 Qt，你需要安装 **Qt VS Tools 插件**。

#### 3.1 安装 Qt VS Tools 插件

途径：

#### 方法 A：VS 内置扩展市场

VS → 扩展 → 管理扩展 → 搜索 “Qt VS Tools” → 安装 → 重启 VS

#### 方法 B：从 Qt Marketplace 下载离线插件

(适用于离线环境)

#### 3.2 在 VS 中设置 Qt 版本路径

安装插件后：

> 工具 → Qt VS Tools → Qt Versions → Add

选择 Qt 安装目录：

```bash
C:\Qt\6.7.0\msvc2019_64
```

#### 3.3 新建 Qt 项目

VS → 新建项目 → Qt GUI Application → 选择 Qt 版本即可。

插件会自动生成 `.pro` 或 CMake 项目。

#### 3.4 常见 VS 集成问题（解决方案）

##### ① VS 里没有 Qt 的模板

原因：Qt VS Tools 未安装好
 解决：重新安装插件 或 修复 VS

##### ② moc、uic 生成失败

原因：Qt Kits 与 MSVC 版本不一致
 建议：

- VS 使用 MSVC 2019 → Qt 也要安装 **msvc2019_64**
- VS 使用 MSVC 2022 → Qt 必须安装 **msvc2022_64**

##### ③ 中文路径导致构建失败

解决：路径全部用英文字母（Qt 官方强烈建议）

### 4. 配置 Qt Creator（适用于 UI/QML）

Qt Creator 会自动识别 Qt SDK，不需要额外安装插件。

#### 4.1 检查 Kits 设置是否正确

打开 Qt Creator：

> Tools → Options → Kits

在 **Qt versions** 中应该能看到类似：

```bat
Qt 6.7.0 (MSVC 2019 64-bit)
Qt 6.7.0 (MinGW 64-bit)
```

在 **Compilers** 中能看到相关 MSVC 编译器：

```bat
Microsoft Visual C++ Compiler 16.0 (amd64)
```

如果没有，请点击 *Add* 手动添加：

- Compiler → C → 查找：`cl.exe`
- Compiler → C++ → 查找：`cl.exe`

VS 的 `cl.exe` 一般在：

```bath
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\*\bin\Hostx64\x64\
```

### 6. 常见坑点与排错

#### ① VS 和 Qt Creator Kits 不一致

会导致 moc 错误 / 链接失败
 解决：确保两个环境都使用同版本 Qt（如 6.7 + MSVC2019）

#### ② MinGW 和 MSVC 冲突

两者编译器 ABI 不兼容，不可混用！
 你在 VS 必须使用 MSVC 套件。

#### ③ Python、CMake、Ninja 缺失

Qt Creator 报错找不到 CMake
解决：安装 Qt SDK 自带的 Developer Tools

#### ④ 中文路径导致构建失败

路径全部英文，不要有空格。

### 7. 最终推荐的开发方式（最佳实践）

#### 搭配使用 VS + Qt Creator 的最佳方案

| 操作                                  | 使用工具      |
| ------------------------------------- | ------------- |
| 写 C++ 业务代码                       | Visual Studio |
| 调试 C++                              | Visual Studio |
| 写 UI（Qt Designer）                  | Qt Creator    |
| 调试 QML、查看 QML Profiler           | Qt Creator    |
| 构建系统                              | CMake         |
| 多平台构建（Windows + Linux + macOS） | Qt Creator    |
