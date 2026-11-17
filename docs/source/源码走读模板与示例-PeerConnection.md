# 源码走读模板 + 示例：PeerConnection

> 用途：每次走读一个模块，按「问题清单 → 卡片 → 记录」产出，避免读完就忘。
> 示例基于 2026-11 的 WebRTC 源码：`/Volumes/SSD/Code/webrtc-build/src`

---

## 0. 每次走读必须搞清的 12 个问题

### 0.1 定位（5 分钟）

1. **它在哪一层？** 属于 api / pc / call / modules / p2p 哪一层？
2. **接口和实现分别在哪？** 接口文件（`api/`）与实现文件（`pc/`、`video/`…）对应关系。
3. **它解决什么问题？不做什么？** 一句话职责 + 边界。

### 0.2 结构（15 分钟）

4. **公开接口有哪些？** 按功能分组（协商/轨道/数据/状态/统计）。
5. **构造、析构、Close 怎么设计？** 谁创建、谁销毁、是否引用计数、protected 还是 public。
6. **关键成员对象有哪些？** 每个成员负责什么、归哪条线程。

### 0.3 行为（20 分钟）

7. **至少两条典型调用链**：从公开 API 入口，跟到真正干活的地方。
8. **数据形态怎么变？** 输入是什么、输出是什么、中间经过哪些结构。
9. **跑在哪个线程？** `RTC_RUN_ON` / `RTC_GUARDED_BY` 标注了什么。
10. **错误怎么返回？** `RTCError`、`RTCErrorOr`、observer 回调、还是静默失败。

### 0.4 输出（10 分钟）

11. **观测点在哪？** 关键日志、可打断点的函数、`getStats` 对应字段。
12. **面试怎么讲？** 3 个可讲点 + 1 个取舍。

> 走读不是「读完文件」，而是能回答这 12 个问题。答不上来的，就是待查项。

---

## 1. 示例卡片：PeerConnection

### 1.1 基本信息

| 项 | 内容 |
|---|---|
| 接口 | `webrtc::PeerConnectionInterface`，`api/peer_connection_interface.h` |
| 实现 | `webrtc::PeerConnection`，`pc/peer_connection.h:120` / `pc/peer_connection.cc` |
| 继承 | `PeerConnectionInternal`、`JsepTransportController::Observer` |
| 创建 | `PeerConnectionFactory::CreatePeerConnectionOrError` → `PeerConnection::Create`（`.cc:438`） |
| 构造 | protected（`.h:488`），参数含 config/env/context/options/call/dependencies/stun/turn/dtls |
| 析构 | `.cc:576`，protected，经 `RefCountInterface::Release` 触发 |
| 显式关闭 | `Close()`，`.cc:1913` |

### 1.2 一句话职责

**PeerConnection 是会话门面（facade）**：把应用层的 SDP / ICE / Track / DataChannel 操作，翻译成 `Call`、传输层、媒体引擎的动作，并聚合状态与统计。**它本身不转发媒体、不做编解码。**

### 1.3 关键成员

| 成员 | 类型 | 职责 | 线程 |
|---|---|---|---|
| `sdp_handler_` | `SdpOfferAnswerHandler` | Offer/Answer 状态机、SDP 生成/解析 | signaling |
| `transport_controller_` | `JsepTransportController` | 按 m-line 建 ICE/DTLS 传输 | network |
| `call_` | `Call` | 媒体流、码率分配、RTP 传输控制器 | worker |
| `rtp_manager_` | `RtpTransmissionManager` | sender/receiver/transceiver 管理 | signaling + network |
| `data_channel_controller_` | `DataChannelController` | SCTP / DataChannel | signaling + network |
| `stats_collector_` | `RTCStatsCollector` | `getStats()` | signaling |
| `message_handler_` | `PeerConnectionMessageHandler` | 信令消息队列 | signaling |
| `tracer_` | `PeerConnectionTracerInterface` | 诊断/事件追踪 | signaling |

### 1.4 公开接口分组

| 分组 | 方法 |
|---|---|
| 协商 | `CreateOffer`、`CreateAnswer`、`SetLocalDescription`、`SetRemoteDescription`、`AddIceCandidate`、`RestartIce` |
| 轨道 | `AddTrack`、`RemoveTrackOrError`、`AddTransceiver`、`GetSenders`、`GetReceivers`、`GetTransceivers` |
| 数据 | `CreateDataChannelOrError` |
| 状态 | `signaling_state`、`ice_connection_state`、`peer_connection_state`、`ice_gathering_state` |
| 统计 | `GetStats`、`ClearStatsCache` |
| 配置 | `GetConfiguration`、`SetConfiguration` |
| 生命周期 | `Close` |

### 1.5 四条调用链

**① CreateOffer**

```
PeerConnection::CreateOffer (.cc:1492)
  → sdp_handler_->CreateOffer(observer, options)
    → SdpOfferAnswerHandler 生成 SDP
      → CreateSessionDescriptionObserver::OnSuccess(desc)
```

**② SetLocalDescription**

```
PeerConnection::SetLocalDescription (.cc:1505)
  → RunOnSignalingThread(...)
    → sdp_handler_->SetLocalDescription(observer, desc)
      → JsepTransportController::ApplyDescription_n
        → 创建/更新 DtlsTransport + P2PTransportChannel
```

**③ AddTrack**

```
PeerConnection::AddTrack (.cc:858)
  → 校验（是否关闭、track 是否为空、是否已有 sender）
  → rtp_manager()->AddTrackUnifiedPlan(...)
    → 创建 RtpSender / RtpTransceiver
  → sdp_handler_->UpdateNegotiationNeeded()
```

**④ ICE 状态变化**

```
P2PTransportChannel 状态变化
  → JsepTransportController 观察
    → PeerConnection::OnTransportControllerConnectionState (.cc:2591)
      → SetIceConnectionState()
        → 日志 "Changing to ICE connected state..." (.cc:2613)
        → observer->OnIceConnectionChange()
```

### 1.6 输入 / 输出

| 动作 | 输入 | 输出 |
|---|---|---|
| CreateOffer | `RTCOfferAnswerOptions` | SDP（通过 observer 回调） |
| SetLocalDescription | `SessionDescriptionInterface` | 传输建立、状态变化（异步） |
| AddTrack | `MediaStreamTrackInterface` + stream ids | `RtpSenderInterface` 或 `RTCError` |
| GetStats | 可选 selector | `RTCStatsReport`（异步回调） |

### 1.7 线程模型

| 线程 | 跑什么 | 代码标记 |
|---|---|---|
| signaling | 公开 API、SDP 协商、rtp_manager、stats | `RTC_DCHECK_RUN_ON(signaling_thread())` |
| network | ICE/DTLS、transport_controller | `RTC_GUARDED_BY(network_thread())` |
| worker | `call_`、媒体流 | `RTC_GUARDED_BY(worker_thread())` |

跨线程方式：

- `RunOnSignalingThread(...)` 投递任务
- `signaling_thread()->BlockingCall(...)` 同步取 SDP
- `ScopedTaskSafety` / `PendingTaskSafetyFlag` 防止对象销毁后回调

### 1.8 生命周期

```
PeerConnectionFactory::CreatePeerConnectionOrError
  → PeerConnection::Create (static)
    → protected 构造函数
      → scoped_refptr 持有
        → 应用调用 Close()
          → 置 closed 状态、sdp_handler_->Close()、释放传输/媒体
        → scoped_refptr 释放
          → Release() → delete → ~PeerConnection()
```

要点：

- 构造/析构都是 protected，只能通过 `scoped_refptr` 管理。
- `Close()` 是幂等的（`IsClosed()` 判断）。
- `Close()` 之后仍可调用部分状态接口，但不能再协商。

### 1.9 错误处理

| 场景 | 方式 |
|---|---|
| 同步失败（AddTrack、CreateDataChannel） | 返回 `RTCErrorOr<T>` |
| 异步失败（CreateOffer、SetLocal） | observer 的 `OnFailure(RTCError)` |
| 配置错误（SetConfiguration） | 返回 `RTCError` |

### 1.10 观测点

- 日志：`(peer_connection.cc:1495) CreateOffer`、`(peer_connection.cc:2613) Changing to ICE connected`
- 断点：`CreateOffer`、`SetLocalDescription`、`OnTransportControllerConnectionState`
- 统计：`getStats()` 里的 `outbound-rtp` / `inbound-rtp` / `transport`

### 1.11 面试可讲点（3 个）

1. **门面而非引擎**：PeerConnection 只做编排，媒体转发在 `Call` / `VideoSendStream`，编解码在 `modules/`。
2. **三线程模型**：signaling / network / worker 各自负责什么，成员用 `RTC_GUARDED_BY` 标注归属。
3. **职责拆分**：SDP 在 `SdpOfferAnswerHandler`，传输在 `JsepTransportController`，媒体在 `Call`——三层可以独立演进。

### 1.12 一个取舍

> 为什么 `CreateOffer` 是异步的？
> 因为 SDP 生成需要查询编解码能力、传输参数、ICE 候选等，可能跨线程；用 observer 回调避免阻塞调用线程。

---

## 2. 走读记录表（可直接复制到笔记）

| 问题 | 结论 | 证据（文件:行） | 待确认 |
|---|---|---|---|
| 职责边界 | 会话门面，不转发媒体 | `pc/peer_connection.h:118` 注释 | |
| 谁创建 | `PeerConnectionFactory::CreatePeerConnectionOrError` | `pc/peer_connection_factory.cc` | 具体行号 |
| SDP 在哪 | `SdpOfferAnswerHandler` | `.h:756`、`.cc:1492` | |
| 传输在哪 | `JsepTransportController` | `.h:745` | |
| 媒体在哪 | `Call` | `.h:699` | |
| 线程归属 | signaling/network/worker | `RTC_GUARDED_BY` 标注 | |
| 关闭流程 | `Close()` → sdp_handler_->Close() | `.cc:1913` | 后续释放细节 |

---

## 3. 走读产出标准

读完 PeerConnection，应该能：

- [ ] 画出「应用 → PeerConnection → SDP/传输/媒体」三层图
- [ ] 说出 `CreateOffer` 的完整调用链
- [ ] 说出 `SetLocalDescription` 之后传输是怎么建起来的
- [ ] 指出 `sdp_handler_`、`transport_controller_`、`call_` 各自的线程
- [ ] 解释为什么构造/析构是 protected
- [ ] 用 2 分钟讲清「PeerConnection 不做什么」

---

## 4. 常见误区

| 误区 | 事实 |
|---|---|
| PeerConnection 负责收发 RTP | 不，RTP 在 `RtpTransportControllerSend` / `RtpVideoSender` |
| PeerConnection 做编解码 | 不，编解码在 `modules/video_coding`、`modules/audio_coding` |
| `CreateOffer` 是同步的 | 异步，通过 observer 回调 |
| `Close()` 后对象立刻销毁 | 不，`Close()` 只关闭；销毁由最后一个 `Release()` 触发 |
| 所有方法都在 signaling 线程 | 不，部分状态接口可从任意线程读（有 clone 机制） |
