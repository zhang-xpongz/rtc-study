# WebRTC 调试指南：加日志、构建、验证

> 基于本机环境（macOS + 2026-11-12 的 clone）
> 目标：搞清楚「改源码 → 让改动生效 → 看到日志」的完整闭环

---

## 0. 先说结论：三条路

| 方式 | 能改源码 | 需要构建 | 现在可用 | 适用场景 |
|---|---|---|---|---|
| **A. Chrome 日志 + SRS** | ❌ | ❌ | ✅ **已实测** | 观察真实 WebRTC 行为、验证协议假设 |
| **B. 完整构建 + 加日志** | ✅ | ✅ 20–30GB | ❌ 缺 deps | 加自定义日志、断点、单步 |
| **C. 单元测试 target** | ✅ | ✅ 同上，但只编小目标 | ❌ | 快速验证某个模块 |

**关键认知**：Chrome 用的是它自带的 libwebrtc 构建产物，**改我们这份 clone 不会影响 Chrome**。要让自己加的日志生效，必须编译出自己的二进制。

---

## 1. 为什么现在改日志不生效

我核实过，当前 clone 缺三样东西：

```
缺 build/         ← gn 的构建脚本
缺 buildtools/    ← clang、gn 二进制
缺 third_party/   ← absl、libyuv、boringssl、ffmpeg…（DEPS 管理）
```

所以 `gn gen` 会直接失败。这也是为什么昨天只能做「读代码」，不能做「跑代码」。

---

## 2. WebRTC 的日志系统

### 2.1 宏

| 宏 | 编译行为 | 用途 |
|---|---|---|
| `RTC_LOG(LS_INFO)` | 始终编译（除非 `RTC_DISABLE_LOGGING`） | **最常用**，调试日志 |
| `RTC_LOG_F(LS_INFO)` | 同上，自动加函数名 | 定位调用点 |
| `RTC_LOG_T_F(LS_INFO)` | 同上，加 `this` 指针 | 多实例区分 |
| `RTC_LOG_IF(LS_INFO, cond)` | 条件日志 | 只打异常分支 |
| `RTC_DLOG(LS_INFO)` | **仅 Debug**（`NDEBUG` 时裁掉） | 高频日志，release 自动消失 |
| `RTC_LOG_V(sev)` | 级别是变量 | 动态过滤 |

### 2.2 级别与输出

```cpp
enum LoggingSeverity { LS_VERBOSE, LS_INFO, LS_WARNING, LS_ERROR, LS_NONE };
```

- 默认级别 **LS_INFO**
- 默认输出 **stderr**（`rtc_base/logging.cc` 里 `log_to_stderr_ = true`）
- 日志格式：`[pid:tid:时间:级别:文件:行] 消息`
- `RTC_LOG` 的宏会先判断 `IsNoop`，**被过滤时参数不会被求值**，所以开销很低

### 2.3 在应用里调整级别

```cpp
#include "rtc_base/logging.h"

webrtc::LogMessage::LogToDebug(webrtc::LS_VERBOSE);  // 打开 verbose
webrtc::LogMessage::SetLogToStderr(true);
```

---

## 3. 加日志：写法与示例

### 3.1 示例：在帧缓冲入口打日志

文件：`video/video_stream_buffer_controller.cc`

```cpp
#include "rtc_base/logging.h"   // 加在 include 区

std::optional<int64_t> VideoStreamBufferController::InsertFrame(
    std::unique_ptr<EncodedFrame> frame) {
  RTC_DCHECK_RUN_ON(&worker_sequence_checker_);

  // 新增：
  RTC_LOG(LS_INFO) << "InsertFrame pid=" << frame->Id()
                   << " key=" << frame->is_keyframe();

  FrameMetadata metadata(*frame);
  ...
}
```

### 3.2 示例：在收包路径打日志

文件：`video/rtp_video_stream_receiver2.cc`，`OnRtpPacket` 里：

```cpp
RTC_LOG(LS_INFO) << "rtp seq=" << packet.SequenceNumber()
                 << " ts=" << packet.Timestamp()
                 << " size=" << packet.size();
```

> 收包路径频率很高，每包都打会拖慢，建议加计数器：
> ```cpp
> static int n = 0;
> if (++n % 100 == 0) RTC_LOG(LS_INFO) << "seq=" << packet.SequenceNumber();
> ```

### 3.3 注意事项

1. **热路径别无条件打**：每帧/每包会显著影响性能，用取模或条件。
2. **release 想保留就用 `RTC_LOG`**，`RTC_DLOG` 在 `is_debug=false` 时会被裁掉。
3. **别在锁里拼长字符串**：虽然过滤时不求值，但启用时会有开销。
4. **多线程日志会交错**：带上 `RTC_LOG_T_F` 或关键字段（pid/ssrc）才能对上。

---

## 4. 构建：让修改生效

### 4.1 前置条件

- Xcode Command Line Tools：`xcode-select --install`
- Python 3、git
- **能访问 `chromium.googlesource.com` 的网络**（当前直连被墙，需要代理）
- 磁盘 ≥ 30GB（`/Volumes/SSD` 还有 427GB）

### 4.2 装 depot_tools

```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git ~/depot_tools
export PATH="$HOME/depot_tools:$PATH"
```

### 4.3 拉完整源码（带依赖）

```bash
# 代理（按你的实际端口改）
export http_proxy=http://127.0.0.1:7890
export https_proxy=$http_proxy
export all_proxy=$http_proxy

mkdir -p /Volumes/SSD/Code/webrtc-build && cd /Volumes/SSD/Code/webrtc-build
gclient config --name src https://webrtc.googlesource.com/src.git
gclient sync --with_branch_heads --with_tags
```

> 也可以直接 `fetch webrtc`，效果一样。
> 这一步会拉 20–30GB，首次 30–90 分钟，取决于代理速度。

### 4.4 生成构建目录

```bash
cd src
gn gen out/Debug --args='
  is_debug=true
  rtc_include_tests=true
  use_rtti=true
  symbol_level=2
'
```

- `is_debug=true`：`RTC_DLOG` 才会生效
- `rtc_include_tests=true`：`peerconnection_client` 是 `testonly`，不加编不出来
- `symbol_level=2`：断点调试需要

### 4.5 编译目标

```bash
autoninja -C out/Debug peerconnection_client
```

- 首次 30–60 分钟（8 核）
- 之后**改一个 .cc，增量编译 + 链接约 1–2 分钟**

常用 target：

| target | 用途 |
|---|---|
| `peerconnection_client` | 完整的 P2P 客户端（GUI） |
| `video_coding_unittests` | 视频编码/帧缓冲单测 |
| `modules_unittests` | 模块层单测 |
| `rtc_unittests` | 基础库单测 |

找 target：`ninja -C out/Debug -t targets | grep 关键字`

### 4.6 运行

```bash
./out/Debug/peerconnection_client
```

日志默认打到 stderr，直接能看到你加的 `RTC_LOG`。

### 4.7 用 lldb 断点

```bash
lldb ./out/Debug/peerconnection_client
(lldb) br set -f video_stream_buffer_controller.cc -l 153
(lldb) run
```

---

## 5. 增量迭代：改日志的正确节奏

```
改 .cc（1 分钟）
  → autoninja -C out/Debug peerconnection_client（1–2 分钟）
  → 运行，观察 stderr
  → 不对就再改
```

技巧：

1. **只编受影响的 target**，不要 `ninja` 全量。
2. **用单测代替完整客户端**：`--gtest_filter=*FrameBuffer*` 几秒跑完。
3. 开启 `use_sccache=true` 可加速重复编译（可选）。
4. 日志加完先在单测里验证，再进完整链路。

---

## 6. 现在不构建也能调试：Chrome 日志（已实测）

这是今天就能用的方式，**能看到真实的 RTC_LOG 输出**，只是不能加自己的日志。

### 6.1 命令

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --user-data-dir=/tmp/chrome-debug --no-first-run \
  --enable-logging=stderr --v=1 \
  "http://localhost:8080/players/rtc_player.html?stream=livestream" \
  2>/tmp/chrome-rtc.log
```

### 6.2 过滤

```bash
# 所有 WebRTC 日志
grep "third_party/webrtc" /tmp/chrome-rtc.log

# 按模块
grep "p2p/base"        /tmp/chrome-rtc.log
grep "video/"          /tmp/chrome-rtc.log
grep "modules/pacing"  /tmp/chrome-rtc.log
grep -i "nack\|pli"    /tmp/chrome-rtc.log
```

### 6.3 实测结果（2026-11-12）

一次 640×360 的 WHEP 播放，抓到 476 条 WebRTC 日志，出现最多的文件：

| 文件 | 条数 | 对应源码 |
|---|---|---|
| `p2p/client/basic_port_allocator.cc` | 70 | candidate 收集 |
| `modules/video_coding/rtp_seq_num_only_ref_finder.cc` | 60 | 帧参考关系 |
| `p2p/base/connection.cc` | 43 | ICE 连接检查 |
| `p2p/base/p2p_transport_channel.cc` | 25 | ICE 传输通道 |
| `media/engine/webrtc_voice_engine.cc` | 22 | 音频引擎 |
| `media/engine/webrtc_video_engine.cc` | 18 | 视频引擎 |
| `pc/channel.cc` | 16 | 媒体通道 |
| `video/video_receive_stream2.cc` | 13 | 视频接收流 |
| `p2p/base/basic_ice_controller.cc` | 13 | ICE 选路 |
| `call/rtp_transport_controller_send.cc` | 10 | 发送控制器 |

样本：

```
[INFO:third_party/webrtc/p2p/base/p2p_transport_channel.cc:559]
  Set backup connection ping interval to 25 s milliseconds.
[INFO:third_party/webrtc/modules/pacing/bitrate_prober.cc:54]
  Bandwidth probing enabled, set to inactive
[INFO:third_party/webrtc/call/rtp_transport_controller_send.cc:...]
```

> 注意：`LS_VERBOSE` 级别的日志在 Release 版 Chrome 里大多被过滤，想看得更细还是得构建 Debug 版。

### 6.4 配合其他观测手段

- `chrome://webrtc-internals`：码率、RTT、jitter、丢包、帧率曲线
- SRS 日志：`RTC: Subscriber established`、`rtp`/`pli` 计数
- `tshark`：抓非加密的 RTSP/RTP（WebRTC 是 SRTP，payload 看不到）

---

## 7. 一个完整闭环示例：验证「丢包时 NACK 何时触发」

1. 起 SRS + ffmpeg 推流（见打卡表附录）。
2. Chrome 带 `--enable-logging=stderr --v=1` 播放。
3. 用 `tc netem` 或 Network Link Conditioner 注入 10% 丢包。
4. 过滤日志：
   ```bash
   grep -iE "nack|retransmit|packet.*lost" /tmp/chrome-rtc.log | tail -50
   ```
5. 对照源码：`modules/video_coding/nack_requester.cc`
6. 记录结论：丢包率多少开始请求、请求间隔、重传是否成功。

这个闭环不需要构建，但能让你在面试里讲出「我实际观察过 NACK 的触发行为」。

---

## 8. 常见坑

| 现象 | 原因 | 解决 |
|---|---|---|
| 改了源码没变化 | 没重新编译，或跑的是旧二进制 | `autoninja` 后确认时间戳；Chrome 不受影响 |
| `RTC_DLOG` 不输出 | release 构建 | 改 `RTC_LOG` 或 `is_debug=true` |
| `gn gen` 报缺 `build/` | 没 `gclient sync` | 先同步依赖 |
| `gclient sync` 卡住 | 代理没配到 git | 设 `http_proxy/https_proxy`，或 `git config --global http.proxy` |
| 日志太多刷屏 | 热路径无条件打印 | 加计数器/条件 |
| 多进程日志混乱 | Chrome 有 renderer/gpu 多进程 | 用 pid 区分，或只看主进程 |
| 断点不进 | 符号被裁 | `symbol_level=2`，`is_debug=true` |

---

## 9. 建议路线

| 阶段 | 做什么 | 时间 |
|---|---|---|
| 今天 | 走通 Chrome 日志（第 6 节），学会过滤 | 1–2 小时 |
| 有代理后 | 完整构建（第 4 节），先确保 `peerconnection_client` 能跑 | 半天到一天 |
| 构建完成后 | 加第一条日志，走完整闭环（第 5 节） | 1 小时 |
| 之后 | 按面试专题加日志：GCC、帧缓冲、NACK、3A | 穿插 |

> 如果暂时没有稳定代理，**不要硬啃构建**。先用 Chrome 日志把「行为」看明白，等网络条件具备再补「源码级」调试。
