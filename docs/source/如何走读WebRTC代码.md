# 如何走读 WebRTC 代码（C++ 生疏版）

> 适用：很久没写 C++，但要读 WebRTC 这种大型 C++ 工程。
> 核心方法：**先补最小语法集 → 配好跳转工具 → 按调用链读 → 用测试/日志验证 → 写卡片。**

---

## 0. 一句话方法

**不要从头到尾读文件，要顺着一条调用链读。**
每读一个模块，只回答 12 个问题（见《源码走读模板与示例-PeerConnection.md》），答不上来的记为待查。

---

## 1. 先补最小 C++ 语法集（2–3 小时）

WebRTC 里反复出现的就这几类，不用系统复习 C++。

### 1.1 类、继承、虚函数

```cpp
class PeerConnection : public PeerConnectionInternal,   // 继承
                       public JsepTransportController::Observer {
 public:
  void CreateOffer(...) override;          // 重写基类虚函数
  virtual void SetOptions(...) = 0;        // 纯虚函数 = 接口
 protected:
  ~PeerConnection() override;              // protected 析构
};
```

读法：

- 看到 `override` → 去基类找同名函数。
- 看到 `= 0` → 这是接口，实现在子类。
- 看到 `protected` 构造/析构 → 不能直接 new/delete，通常配合引用计数。

### 1.2 智能指针与引用计数

| 写法 | 含义 |
|---|---|
| `std::unique_ptr<T> x_` | 独占所有权，离开作用域自动 delete |
| `scoped_refptr<T> x_` | WebRTC 的引用计数指针，内部调 `AddRef/Release` |
| `T* x_` | 通常是非拥有指针（借用），生命周期由别人管 |
| `std::shared_ptr<T>` | 标准库引用计数，WebRTC 里较少用 |

WebRTC 的 `scoped_refptr` 依赖对象实现 `AddRef()` / `Release()`（`api/ref_count.h`）。

### 1.3 模板与容器

```cpp
std::vector<scoped_refptr<RtpSenderInterface>> senders_;
std::map<std::string, TransportStats> stats_;
std::optional<std::string> mid_;
absl::string_view label;
```

读法：把 `std::vector<T>` 当「T 的数组」，`std::optional<T>` 当「可能为空的 T」，不用深究模板实现。

### 1.4 回调与 lambda

```cpp
pc->GetStats([this](scoped_refptr<RTCStatsReport> report) {
  // 异步回调
});
```

常见形式：

- `std::function<void(RTCError)> callback`
- `absl::AnyInvocable<void()>`
- `observer->OnSuccess(desc)`（观察者模式）

读法：先找「谁注册回调」，再找「谁触发回调」，中间就是异步路径。

### 1.5 宏（WebRTC 高频）

| 宏 | 作用 | 读法 |
|---|---|---|
| `RTC_LOG(LS_INFO) << ...` | 日志 | 当 `std::cout` 读 |
| `RTC_DLOG(...)` | 仅 Debug 的日志 | 同上 |
| `RTC_DCHECK(cond)` | Debug 断言 | 当 `assert` 读 |
| `RTC_DCHECK_RUN_ON(&checker)` | 断言当前线程 | **告诉你这段跑在哪个线程** |
| `RTC_GUARDED_BY(mutex_)` | 成员受哪个锁保护 | 告诉你并发约束 |
| `RTC_EXPORT` | 符号导出 | 忽略 |
| `[[deprecated]]` | 已废弃 | 忽略 |

### 1.6 线程注解

```cpp
void SetLocalDescription(...) {
  RTC_DCHECK_RUN_ON(signaling_thread());   // 必须在 signaling 线程
}
```

看到 `RTC_GUARDED_BY(network_thread())`，就知道这个成员归 network 线程。

### 1.7 移动语义（只需认识）

```cpp
std::move(desc)          // 把所有权转走
T(std::forward<P0>(p0))  // 完美转发，构造时常见
```

读法：看到 `std::move(x)`，表示 x 之后不再使用。

---

## 2. 配置跳转工具（30 分钟）

### 2.1 已有条件

- `out/Debug/compile_commands.json`（34MB，gn 生成）
- `clangd`（Xcode 自带，`xcrun --find clangd`）
- 源码根目录已建好软链：`src/compile_commands.json → out/Debug/compile_commands.json`

### 2.2 Cursor 配置

1. 用 Cursor 打开 `/Volumes/SSD/Code/webrtc-build/src`
2. 安装扩展：**clangd**（llvm-vs-code-extensions.vscode-clangd）
3. 设置里加：

```json
"clangd.arguments": [
  "--background-index",
  "--header-insertion=never",
  "--compile-commands-dir=/Volumes/SSD/Code/webrtc-build/src"
]
```

4. `Cmd+Shift+P` → `clangd: Restart language server`

### 2.3 常用跳转

| 操作 | Mac 快捷键 |
|---|---|
| 跳到定义 | `F12` 或 `Cmd+Click` |
| 找引用 | `Shift+F12` |
| 文件内搜索 | `Cmd+F` |
| 全局搜索 | `Cmd+Shift+F` |
| 按文件名打开 | `Cmd+P` |
| 按符号跳转 | `Cmd+T` |
| 返回 | `Ctrl+-` |

### 2.4 没有 IDE 时的命令

```bash
cd /Volumes/SSD/Code/webrtc-build/src

# 找类/函数
rg -n "class FrameBuffer" api video
rg -n "InsertFrame" video api

# 找调用点
rg -n "InsertFrame\(" video modules

# 看某行历史
git log -S "InsertFrame" --oneline -- video/video_stream_buffer_controller.cc

# 看某个 target 的依赖
gn desc out/Debug //video:video deps | head
```

### 2.5 断点

```bash
cd /Volumes/SSD/Code/webrtc-build/src
lldb ./out/Debug/peerconnection_unittests
(lldb) br set -f video_stream_buffer_controller.cc -l 153
(lldb) run --log_level=info --gtest_filter=...
```

---

## 3. 单个模块的走读流程（45–60 分钟）

### Step 0：选一条链，不选一个文件

错误做法：「今天读 peer_connection.cc」。
正确做法：「今天读 CreateOffer 这条链」。

### Step 1：读头文件（10 分钟）

只看：

- `class X : public Y`（继承关系）
- `public:` 下的方法（能力清单）
- 成员变量（依赖哪些对象）

**跳过** private 方法实现、模板细节。

### Step 2：找接口 vs 实现（5 分钟）

- 接口在 `api/`
- 实现在 `pc/`、`video/`、`modules/`

例：`PeerConnectionInterface`（api）↔ `PeerConnection`（pc）。

### Step 3：跟一条调用链（20 分钟）

从公开 API 入口开始，用 `F12` 一跳一跳跟：

```
入口函数
  → 调了谁
    → 谁真正干活
      → 边界（到另一个模块就停）
```

**规则**：跟到「不是本模块职责」的地方就停，记下边界，不要陷进去。

### Step 4：标线程（5 分钟）

在调用链每一跳旁边写：

- signaling / network / worker
- 依据：`RTC_DCHECK_RUN_ON`、`RTC_GUARDED_BY`

### Step 5：读测试（10 分钟）

测试是最好的使用说明：

```bash
rg -n "CreateOffer" pc/*unittest*.cc | head
```

看它怎么构造对象、怎么断言、怎么等回调。

### Step 6：验证（10 分钟）

- 加一条 `RTC_LOG`，重编跑测试（1–2 分钟）
- 或直接 `lldb` 断点看调用栈

### Step 7：写卡片（5 分钟）

用 `学习记录/模板/源码卡片.md`：

- 职责一句话
- 关键类/函数 3–5 个
- 输入/输出
- 调用链
- 线程
- 面试可讲点 3 条
- 没搞懂 1–2 条

---

## 4. WebRTC 阅读顺序（由小到大）

不要一上来读 `peer_connection.cc`（近万行）。建议顺序：

| 顺序 | 模块 | 文件 | 为什么先读 |
|---|---|---|---|
| 1 | 帧缓冲接口 | `api/video/frame_buffer.h` | 纯数据结构，短 |
| 2 | 帧缓冲使用 | `video/video_stream_buffer_controller.cc` | 看接口怎么被用 |
| 3 | Call 门面 | `call/call.h` | 流的管理入口 |
| 4 | PeerConnection 头 | `pc/peer_connection.h` | 会话层全貌 |
| 5 | CreateOffer 链 | `pc/peer_connection.cc` + `pc/sdp_offer_answer.cc` | 协商 |
| 6 | 视频发送 | `video/video_send_stream_impl.cc` | 发送链路 |
| 7 | 视频接收 | `video/rtp_video_stream_receiver2.cc` | 接收链路 |
| 8 | 拥塞控制 | `modules/congestion_controller/goog_cc/` | 控制环 |

---

## 5. 第一个 30 分钟练习

打开 `api/video/frame_buffer.h`，回答 5 个问题：

1. `FrameBuffer` 存的是什么？（`EncodedFrame` 的队列）
2. `InsertFrame` 返回什么？为什么是 `std::optional<int64_t>`？
3. 有没有线程约束？
4. 谁调用它？（去 `video/video_stream_buffer_controller.cc` 找）
5. 它和 `RtpFrameReferenceFinder` 的分工是什么？

然后跳一次调用链：

```
VideoReceiveStream2::OnCompleteFrame
  → VideoStreamBufferController::InsertFrame
    → FrameBuffer::InsertFrame
```

能画出这条链，今天的走读就算成功。

---

## 6. 十个技巧

1. **先 .h 后 .cc**：头文件是目录，源文件是正文。
2. **先 public 后 private**：公开接口决定能力，私有实现决定细节。
3. **看到 `override` 就去基类**：虚函数的分派在基类定义。
4. **看到 `= 0` 就当接口**，不要找实现，去找子类。
5. **`unique_ptr` = 拥有，`T*` = 借用**，`scoped_refptr` = 共享。
6. **宏当函数读**，不用展开。
7. **模板当类型读**，不追实现。
8. **回调找注册点和触发点**，中间是异步路径。
9. **大文件用 `rg "Class::Method"` 直接跳**，不要滚动。
10. **测试当文档**：`*unittest.cc` 里有最标准的用法。

---

## 7. 七天恢复计划（每天 2–3 小时）

| 天 | 内容 | 产出 |
|---|---|---|
| D1 | C++ 语法速览 + Cursor/clangd 配置 + 读 `frame_buffer.h` | 能跳转、能说 5 个问题 |
| D2 | 跟 `InsertFrame` 调用链 + 加日志跑测试 | 一张调用链图 |
| D3 | `call/call.h` + `CreateVideoSendStream` | Call 的职责卡片 |
| D4 | `peer_connection.h` + `CreateOffer` 链 | PeerConnection 卡片 |
| D5 | `video_send_stream_impl.cc` 发送链路 | 发送链路图 |
| D6 | `rtp_video_stream_receiver2.cc` 接收链路 | 接收链路图 |
| D7 | `goog_cc_network_control` + 复盘 | GCC 卡片 + 周总结 |

---

## 8. 常见坑

| 坑 | 正确做法 |
|---|---|
| 从第一个文件按顺序读 | 按调用链读 |
| 一上来读 `peer_connection.cc` | 先读头文件和小模块 |
| 追模板/宏展开 | 当类型/函数读 |
| 忽略 `RTC_RUN_ON` | 它是线程地图 |
| 只读不写 | 每模块写 5 行卡片 |
| 读完就忘 | 加日志跑一遍，眼见为实 |
| 卡在一个函数 1 小时 | 设 40 分钟上限，记待查 |
| 想读懂每一行 | 先懂主路径，细节按需 |

---

## 9. 一句话总结

> **C++ 生疏不影响走读**：你需要的不是重学 C++，而是认识 20 个高频写法、配好 clangd 跳转、按一条调用链走、用测试验证、写 5 行卡片。读 3 个模块后，语感自然回来。
