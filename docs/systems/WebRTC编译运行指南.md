# WebRTC 编译与运行指南（macOS）

> 环境已验证：2026-11-12
> 源码：`/Volumes/SSD/Code/webrtc-build/src`（26GB，含 third_party）
> 构建：`out/Debug`
> 已经在跑的目标：`pc:peerconnection_unittests`（146MB，全量编译 4m25s）

---

## 0. 三条命令（TL;DR）

```bash
export PATH="$HOME/depot_tools:$PATH"
cd /Volumes/SSD/Code/webrtc-build/src
autoninja -C out/Debug pc:peerconnection_unittests

./out/Debug/peerconnection_unittests --log_level=info \
  --gtest_filter=PeerConnectionInterfaceTestPlanB.AddedStreamsPresentInOffer
```

或者用我放在工作区的脚本：

```bash
./webrtc-build.sh pc:peerconnection_unittests
./webrtc-test.sh peerconnection_unittests --gtest_filter=*Offer*
```

---

## 1. 目录与前置

| 路径 | 用途 |
|---|---|
| `/Volumes/SSD/Code/webrtc-build/src` | 可编译源码（含 `build/`、`buildtools/`、`third_party/`） |
| `/Volumes/SSD/Code/webrtc-build/src/out/Debug` | 构建输出 |
| `~/depot_tools` | gclient / gn / autoninja |
| `/Users/zxp/Documents/code/webrtc-src` | 只读走读用的浅克隆，不参与编译 |

前置条件（已满足）：

- Xcode Command Line Tools（`xcode-select -p` 有输出）
- Python 3
- depot_tools 已 bootstrap（`~/depot_tools/python3_bin_reldir.txt` 存在）

每次新开终端都要：

```bash
export PATH="$HOME/depot_tools:$PATH"
```

---

## 2. 首次准备（只需要一次）

### 2.1 代理（只在下载时需要）

```bash
export http_proxy=http://127.0.0.1:2080
export https_proxy=$http_proxy
export all_proxy=$http_proxy
```

### 2.2 bootstrap depot_tools

```bash
cd ~/depot_tools && ./ensure_bootstrap
```

如果报 `python3_bin_reldir.txt not found`，就是漏了这一步。

### 2.3 同步源码（已有则跳过）

```bash
mkdir -p /Volumes/SSD/Code/webrtc-build && cd /Volumes/SSD/Code/webrtc-build
gclient config --name src https://webrtc.googlesource.com/src.git
gclient sync --with_branch_heads --with_tags -j8
```

约 26GB，视代理速度 1–3 小时。**编译本身不需要网络**，代理只用于 sync。

---

## 3. 生成构建目录

```bash
cd /Volumes/SSD/Code/webrtc-build/src
gn gen out/Debug --args='
  is_debug=true
  rtc_include_tests=true
  use_rtti=true
  symbol_level=2
'
```

参数含义：

| 参数 | 作用 |
|---|---|
| `is_debug=true` | Debug 构建，`RTC_DLOG` 生效，带符号 |
| `rtc_include_tests=true` | 生成测试 target（不加就没有 unittests） |
| `use_rtti=true` | 方便 lldb 看类型（可选） |
| `symbol_level=2` | 完整调试符号（体积大，但断点准） |

生成成功会输出 `Done. Made 7792 targets from 447 files`。

重新生成不会清掉 `out/Debug` 的编译产物，但改 args 会触发大量重编。

---

## 4. 编译

```bash
autoninja -C out/Debug <target>
```

target 写法是 `路径:目标名`，例如：

| target | 说明 |
|---|---|
| `pc:peerconnection_unittests` | PeerConnection 相关单测（已验证，146MB） |
| `modules/video_coding:video_coding_unittests` | 视频编码/帧缓冲单测 |
| `:rtc_unittests` | 基础库单测 |
| `pc:rtc_pc_unittests` | 更底层的 pc 单测 |

找目标：

```bash
gn ls out/Debug | grep video_coding
gn desc out/Debug //pc:peerconnection_unittests deps | head
```

### 为什么没有 `peerconnection_client`

这个版本里它只对 Linux/ChromeOS/Windows 定义：

```gn
if (is_linux || is_chromeos || is_win) {
  rtc_executable("peerconnection_client") { ... }
}
```

macOS 上无法编译。替代方案：

1. **单元测试**（最省事，推荐）
2. `examples:AppRTCMobile`（iOS 工程，不是 macOS 桌面）
3. 自己写一个 executable target（见第 8 节）

### 编译耗时（实测）

| 场景 | 时间 |
|---|---|
| 首次全量 | 4 分 25 秒（8 核） |
| 改 1 个 .cc 后 | 约 4 分钟（siso 重算较多） |
| 只改测试文件 | 通常 1 分钟内 |

---

## 5. 运行

```bash
./out/Debug/peerconnection_unittests \
  --log_level=info \
  --gtest_filter=PeerConnectionInterfaceTestPlanB.AddedStreamsPresentInOffer
```

### 关键参数

| 参数 | 作用 |
|---|---|
| `--gtest_filter=Suite.Test` | 只跑指定测试，支持 `*` 通配 |
| `--gtest_list_tests` | 列出所有测试 |
| `--log_level=info` | **打开 INFO 日志（默认是 warning）** |
| `--log_level=verbose` | 更详细 |
| `--gtest_repeat=10` | 重复跑，排查偶现 |
| `--gtest_break_on_failure` | 失败即断（配合 lldb） |

> **最容易踩的坑**：`RTC_LOG(LS_INFO)` 默认不输出，因为测试框架默认 `LS_WARNING`。必须加 `--log_level=info`，或者把日志写成 `LS_WARNING`。

---

## 6. 加日志 → 生效的完整流程

### 6.1 改代码

```cpp
#include "rtc_base/logging.h"   // 文件顶部已有则跳过

void PeerConnection::CreateOffer(CreateSessionDescriptionObserver* observer,
                                 const RTCOfferAnswerOptions& options) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_LOG(LS_INFO) << "=== MY DEBUG LOG: CreateOffer called ===";
  sdp_handler_->CreateOffer(observer, options);
}
```

### 6.2 增量编译

```bash
autoninja -C out/Debug pc:peerconnection_unittests
```

### 6.3 运行看日志

```bash
./out/Debug/peerconnection_unittests --log_level=info \
  --gtest_filter=PeerConnectionInterfaceTestPlanB.AddedStreamsPresentInOffer
```

实测输出：

```
(peer_connection.cc:1495): === MY DEBUG LOG: PeerConnection::CreateOffer called ===
(peer_connection.cc:1495): === MY DEBUG LOG: PeerConnection::CreateOffer called ===
[  PASSED  ] 1 test.
```

日志格式是 `(文件:行): 消息`，来源文件行号都在，方便定位。

### 6.4 日志级别选择

| 宏 | 编译 | 默认可见 | 适用 |
|---|---|---|---|
| `RTC_LOG(LS_INFO)` | 始终 | 否（需 `--log_level=info`） | 调试 |
| `RTC_LOG(LS_WARNING)` | 始终 | 是 | 想默认就看到 |
| `RTC_LOG(LS_ERROR)` | 始终 | 是 | 异常路径 |
| `RTC_DLOG(LS_INFO)` | 仅 Debug | 否 | 高频日志 |

---

## 7. 用 lldb 调试

```bash
lldb ./out/Debug/peerconnection_unittests
(lldb) settings set target.run-args --log_level=info --gtest_filter=PeerConnectionInterfaceTestPlanB.AddedStreamsPresentInOffer
(lldb) br set -f peer_connection.cc -l 1495
(lldb) run
```

常用：

```
bt                # 调用栈
frame variable    # 当前帧变量
p options         # 打印变量
thread backtrace all
```

---

## 8. 自己写一个可执行 target（可选）

如果一定要一个“能跑起来的程序”，可以在 `src/` 下加一个 `BUILD.gn`，例如：

```gn
rtc_executable("probe") {
  testonly = true
  sources = [ "probe.cc" ]
  deps = [
    "../pc:peerconnection",
    "../api:create_peerconnection_factory",
    "../api:peer_connection_interface",
  ]
}
```

然后 `autoninja -C out/Debug probe`。

注意：macOS 上需要自己处理视频采集/渲染（或用 fake source），比跑单测麻烦。建议先用单测验证逻辑，需要端到端时再写。

---

## 9. 常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| `unknown target "peerconnection_client"` | macOS 不定义该 target | 用 `pc:peerconnection_unittests` |
| 日志不输出 | 默认 `LS_WARNING` | 加 `--log_level=info` |
| `python3_bin_reldir.txt not found` | depot_tools 未 bootstrap | `cd ~/depot_tools && ./ensure_bootstrap` |
| `gn: command not found` | PATH 没设 | `export PATH="$HOME/depot_tools:$PATH"` |
| `gclient sync` 卡住/超时 | 没走代理 | 设 `http_proxy/https_proxy` |
| 编译很慢 | 首次全量 + siso | 正常；后续改小文件会快 |
| `out/Debug` 占空间 | Debug 符号大 | 可 `symbol_level=1`，或定期清 `out/Release` |

---

## 10. 脚本

工作区里有两个脚本，直接可用：

```bash
# 编译（默认 pc:peerconnection_unittests）
./webrtc-build.sh

# 指定 target
./webrtc-build.sh modules/video_coding:video_coding_unittests

# 跑测试（自动带 --log_level=info）
./webrtc-test.sh peerconnection_unittests --gtest_filter=*CreateOffer*
```

脚本内容在 `webrtc-build.sh`、`webrtc-test.sh`，路径都写死为 `/Volumes/SSD/Code/webrtc-build/src`，换目录时改一下即可。
