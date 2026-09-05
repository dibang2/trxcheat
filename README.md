# TRON 剪贴板守卫

一个本地运行的防御向工具，用于防止剪贴板劫持木马偷偷替换你的 TRON 收款地址。

## 项目背景

剪贴板劫持木马会在你复制钱包地址后，将地址偷偷替换成攻击者的地址，导致转账时资金损失。本程序反其道而行：持续监听剪贴板，一旦发现任何非白名单的 TRON 地址，立刻还原成你自己的地址。

## 核心特性

- **地址识别**：基于 base58check 校验（双 SHA-256），识别 `T` 开头、34 字符、载荷首字节 `0x41` 的合法 TRON 地址
- **三种模式**（托盘右键切换）：
  - `pin` 锁定模式：自动还原为你的地址（默认）
  - `alert` 仅提醒：不修改剪贴板
  - `off` 关闭
- **白名单**：可配置多个允许通过的地址
- **严格校验**：可选 base58check 校验，避免普通文本被误判为地址
- **剪贴板对抗**：`WM_CLIPBOARDUPDATE` 事件监听 + 400ms 周期复查 + 60/150/400ms 三级延迟复查，压住会反复回写的劫持程序
- **进程守护**：`guard_dog.exe` 每 1.2 秒轮询，主进程被杀后 1~2 秒内静默重启
- **完全静默**：无控制台窗口、无气泡通知、不生成日志文件

## 技术栈

- 语言：C（C89 风格）
- 平台：Windows / Win32 SDK
- 编译器：MSVC（Visual Studio 2026）或 MinGW-w64
- 无第三方依赖，自带 SHA-256 实现

## 文件结构

| 文件 | 说明 |
|------|------|
| `clipboard_guard.c` | 主守卫程序源码 |
| `guard_dog.c` | 守护进程源码，负责被杀后自动拉起主程序 |
| `clipboard_guard.cfg` | 运行时配置文件 |
| `build.bat` | 编译脚本 |
| `start.bat` | 启动脚本（启动守护 + 主守卫） |
| `shutdown.bat` | 停止脚本（强杀两个进程） |

## 快速开始

### 编译

双击 `build.bat`（需要已安装 Visual Studio 2026）。

或使用 MinGW-w64：

```
gcc -O2 -municode -mwindows -o clipboard_guard.exe clipboard_guard.c -luser32 -lshell32 -lgdi32 -lole32
gcc -O2 -o guard_dog.exe guard_dog.c -luser32
```

### 启动

双击 `start.bat`，或直接运行 `guard_dog.exe`。

守护进程会自动拉起主守卫，两者均以无窗口方式运行，仅在系统托盘显示一个盾牌图标。

### 停止

双击 `shutdown.bat`，或手动执行：

```
taskkill /f /im guard_dog.exe & taskkill /f /im clipboard_guard.exe
```

杀掉 `guard_dog.exe` 后不会再有进程拉起主守卫。

## 配置说明

编辑 `clipboard_guard.cfg`：

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `pin` | 你自己的收款地址。非白名单地址会被还原成这个 | `TStqZhsa8yr7mkaWLax4zE3c5KPD1g6666` |
| `mode` | `pin` 锁定还原 / `alert` 仅提醒 / `off` 关闭 | `pin` |
| `strict` | `1` 启用 base58check 校验 / `0` 宽松匹配 | `1` |
| `watchdog` | `1` 每 400ms 复查剪贴板 / `0` 关闭 | `1` |
| `allow` | 白名单地址，可多行 | 空 |

修改后在托盘图标右键 → 重新加载配置。

## 自检

```
clipboard_guard.exe --selftest
```

会弹出控制台窗口，显示地址校验测试结果。

## 安全说明

- 全程本地运行，不联网、不上传、不注入其他进程
- 不写自启动项，重启后需手动启动
- 不做双进程互保，用户可随时通过 `taskkill` 彻底关闭
- 仅针对 TRON 地址进行检测和替换，不处理其他内容

## 工作原理

1. 主程序注册剪贴板格式监听器，同时启动 400ms 定时器
2. 剪贴板内容变化时，扫描其中是否包含合法 TRON 地址
3. 对非白名单地址，等长替换为 `pin` 地址后写回剪贴板
4. 为对抗劫持程序的反复回写，在每次变更后追加三级延迟复查
5. 守护进程定时检测主程序是否存活，不存活则静默重启
