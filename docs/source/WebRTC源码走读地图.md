# WebRTC 源码走读地图

> 基于本机克隆：`/Users/zxp/Documents/code/webrtc-src`
> commit `afedb52`（2026-11-11，同步官方 `24d4c4c`），7,674 个文件，119MB 源码。
> 文中所有路径、类名均已在本仓核实。

---

## 0. 先记住三句话

1. **WebRTC 是分层的**：会话层（pc）→ 引擎层（media/engine）→ 流层（call/video/audio）→ 协议层（modules/rtp_rtcp、pacing、congestion）→ 传输层（p2p、rtc_base）。
2. **数据只有两条主链路**：发送（采集→编码→打包→发送）和接收（收包→组帧→抖动缓冲→解码→渲染），所有模块都挂在这两条链上。
3. **不要按目录顺序读，按调用链读**。先找一条链的入口，顺着回调往下走，走不通再跳目录。

---

## 1. 分层总览

```
┌─────────────────────────────────────────────────────────────┐
│ 应用层  sdk/android (Java/JNI)   sdk/objc (ObjC)             │
├─────────────────────────────────────────────────────────────┤
│ 会话层  pc/  PeerConnection、SDP、JSEP、DTLS、SCTP、DataChannel │
├─────────────────────────────────────────────────────────────┤
│ 引擎层  media/engine/  VideoEngine、VoiceEngine（m-line→流）  │
├─────────────────────────────────────────────────────────────┤
│ 流层    call/     Call、VideoSendStream、AudioSendStream       │
│         video/    视频收发实现、编码器、帧缓冲                  │
│         audio/    音频收发实现                                 │
├─────────────────────────────────────────────────────────────┤
│ 协议层  modules/rtp_rtcp/    RTP/RTCP、NACK、FEC、RTX          │
│         modules/pacing/      发送节奏、探测                    │
│         modules/congestion_controller/  带宽估计（GCC）        │
│         modules/video_coding/ 编解码、SVC、组帧                │
│         modules/audio_coding/ 编解码、NetEq                    │
│         modules/audio_processing/  3A                          │
├─────────────────────────────────────────────────────────────┤
│ 传输层  p2p/  ICE、STUN、TURN、DTLS                            │
│         rtc_base/  线程、任务队列、Socket、时间、IP            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 两条端到端链路

### 2.1 视频发送

```
VideoSource
   │ OnFrame
   ▼
VideoStreamEncoder ──► VideoEncoder（VP8/VP9/H264/AV1）
   │                        │
   │                        ▼ OnEncodedImage
   │                 VideoSendStreamImpl（编码完成回调）
   │                        │
   │                        ▼
   │                 RtpVideoSender ──► 分包 / FEC / RTX
   │                        │
   │                        ▼
   │                 RtpRtcpImpl2 ──► RtpSenderEgress
   │                        │
   │                        ▼
   │                 TaskQueuePacedSender ──► PacketRouter
   │                        │
   ▼                        ▼
码率分配                UDP Socket
```

### 2.2 视频接收

```
UDP Socket
   │ OnRtpPacket
   ▼
RtpVideoStreamReceiver2
   │ H26xPacketBuffer / PacketBuffer（按 seq 缓存、组帧）
   ▼
RtpFrameReferenceFinder（处理乱序、丢包、参考帧依赖）
   │ OnCompleteFrame
   ▼
VideoReceiveStream2::OnCompleteFrame
   │
   ▼
VideoStreamBufferController::InsertFrame
   │ FrameBuffer（按 PID 排序、等参考帧）+ FrameDecodeScheduler
   ▼
VideoStreamDecoder ──► VideoDecoder（含 PLC）
   │
   ▼
VideoReceiveStream2::OnFrame ──► VideoSink（渲染）
```

---

## 3. 目录地图

| 目录 | 文件数 | 一句话职责 | 优先级 |
|---|---|---|---|
| `api/` | 727 | 公共接口契约（不依赖实现） | ★★★ 先读 |
| `pc/` | 277 | PeerConnection、SDP、JSEP、DTLS、SCTP | ★★★ |
| `media/engine/` | 80 | 媒体引擎，把 m-line 变成流 | ★★ |
| `call/` | 119 | Call、流管理、码率分配、RTP 传输控制器 | ★★★ |
| `video/` | 308 | 视频收发实现、编码器、帧缓冲 | ★★★ |
| `audio/` | 67 | 音频收发实现 | ★★ |
| `modules/rtp_rtcp/` | 323 | RTP/RTCP、NACK、FEC、RTX | ★★★ |
| `modules/pacing/` | 24 | 发送节奏、探测 | ★★★ |
| `modules/congestion_controller/` | 113 | GCC 带宽估计 | ★★★ |
| `modules/video_coding/` | 265 | 编解码、SVC、组帧、参考帧 | ★★★ |
| `modules/audio_coding/` | 349 | 音频编解码、NetEq | ★★ |
| `modules/audio_processing/` | 546 | 3A（AEC3/NS/AGC2） | ★★ |
| `p2p/` | 137 | ICE、STUN、TURN、DTLS | ★★★ |
| `rtc_base/` | 525 | 线程、任务队列、Socket、时间 | ★★ |
| `sdk/` | 787 | Android/iOS 平台 API | ★ |
| `test/`、`tools_webrtc/` | 1,061 | 测试与工具 | 当入口用 |

---

## 4. 模块详解

### 4.1 `api/` —— 公共契约

**职责**：定义所有对外接口，实现方在别的目录。读它是为了先建立"能力清单"。

**关键文件**

| 文件 | 内容 |
|---|---|
| `api/peer_connection_interface.h` | PeerConnection 的所有能力 |
| `api/video/video_frame.h` / `video_frame_buffer.h` | 帧与缓冲区 |
| `api/video_codecs/video_encoder.h` / `video_decoder.h` | 编解码接口 |
| `api/transport/network_control.h` | 带宽估计的输入输出接口 |
| `api/video/frame_buffer.h` | `FrameBuffer`（视频抖动缓冲核心） |
| `api/audio/audio_processing.h` | 3A 接口 |
| `api/neteq/neteq.h` | 音频抖动缓冲接口 |
| `api/rtp_sender_interface.h` | RTP 发送接口 |

**读法**：只读接口签名和注释，10 分钟一个文件，不求细节。

---

### 4.2 `pc/` —— 会话层

**职责**：把 SDP 协商成实际的传输和媒体流。

**关键类**

| 类 | 文件 | 作用 |
|---|---|---|
| `PeerConnection` | `pc/peer_connection.cc` | 总入口，约 1 万行，**不要通读** |
| `SdpOfferAnswerHandler` | `pc/sdp_offer_answer.cc` | Offer/Answer 状态机 |
| `JsepTransportController` | `pc/jsep_transport_controller.cc` | 按 m-line 建传输 |
| `DtlsTransport` | `pc/dtls_transport.cc` | DTLS 握手、SRTP 密钥 |
| `SctpTransport` | `pc/sctp_transport.cc` | DataChannel 底层 |
| `RtpTransceiver` | `pc/rtp_transceiver.cc` | 收发器 |
| `RTCStatsCollector` | `pc/rtc_stats_collector.cc` | 统计 |

**实现逻辑**
1. `CreateOffer` → `SdpOfferAnswerHandler` 生成 SDP（编解码、ICE ufrag、DTLS 指纹）。
2. `SetLocalDescription` 触发 `JsepTransportController::ApplyDescription_n`。
3. `MaybeCreateJsepTransport` 为每个 m-line 建 `DtlsTransport` + `P2PTransportChannel`。
4. ICE 连通后跑 DTLS 握手，导出 SRTP 密钥。

**读法**：只看 `CreateOffer`、`SetLocalDescription`、`SetRemoteDescription` 三个入口，再跟 `ApplyDescription_n`。

---

### 4.3 `media/engine/` —— 媒体引擎

**职责**：介于 `pc` 和 `call` 之间，把 m-line 的编解码配置翻译成流。

**关键类**
- `WebRtcVideoEngine` / `WebRtcVideoChannel` / `WebRtcVideoSendChannel` / `WebRtcVideoReceiveChannel`（`media/engine/webrtc_video_engine.h`）
- `WebRtcVoiceEngine`（`media/engine/webrtc_voice_engine.h`）
- `WebRtcMediaEngine`（组合）
- `SimulcastEncoderAdapter`（`media/engine/simulcast_encoder_adapter.h`）
- `internal_encoder_factory`（内置编解码工厂）

**实现逻辑**：`AddSendStream` 时创建 `VideoSendStream`；编码配置、RTP 参数、Simulcast 层数都在这里落到 `VideoEncoderConfig`。

**读法**：先看 `WebRtcVideoSendChannel::AddSendStream`，看它怎么调 `Call::CreateVideoSendStream`。

---

### 4.4 `call/` —— 流管理与 RTP 传输控制器

**职责**：一次通话（Call）里所有流的生命周期、码率分配、发送调度。

**关键类**

| 类 | 文件 | 作用 |
|---|---|---|
| `Call` | `call/call.cc` | 创建/销毁流 |
| `BitrateAllocator` | `call/bitrate_allocator.cc` | 按优先级分码率 |
| `RtpTransportControllerSend` | `call/rtp_transport_controller_send.cc` | pacer + 带宽估计 + 反馈 |
| `RtpVideoSender` | `call/rtp_video_sender.cc` | 视频 RTP 发送封装 |
| `VideoSendStreamImpl` | `video/video_send_stream_impl.cc` | 视频发送流（实现在 video/） |
| `AudioSendStream` | `call/audio_send_stream.cc` | 音频发送流 |

**实现逻辑**
- `RtpTransportControllerSend` 是发送侧的中枢：持有 `TaskQueuePacedSender`、`TransportFeedbackAdapter`、`CongestionControlHandler`、`NetworkControllerInterface`。
- 带宽估计结果经 `CongestionControlHandler` 转成 `TargetTransferRate`，交给 `BitrateAllocator`，再回调到各流的 `OnBitrateUpdated`。
- `RtpVideoSender` 持有 `RtpRtcpImpl2`、FEC、RTX，负责把编码帧变成 RTP 包。

**读法**：`Call::CreateVideoSendStream` → `VideoSendStreamImpl::Start` → `RtpVideoSender` 的构造，一条线下来。

---

### 4.5 `video/` —— 视频收发实现（重点）

**发送侧**

| 类 | 文件 | 作用 |
|---|---|---|
| `VideoSendStreamImpl` | `video/video_send_stream_impl.cc` | 流对象，实现 `VideoSendStream` + `EncoderSink` |
| `VideoStreamEncoder` | `video/video_stream_encoder.cc` | 编码调度：降级、Simulcast、码率 |
| `VideoStreamEncoderInterface` | `video/video_stream_encoder_interface.h` | 接口 |

流程：`SetSource` → 采集帧进 `OnFrame` → 编码器 → `OnEncodedImage` → `RtpVideoSender::OnEncodedImage`。

**接收侧**

| 类 | 文件 | 作用 |
|---|---|---|
| `VideoReceiveStream2` | `video/video_receive_stream2.cc` | 接收流对象 |
| `RtpVideoStreamReceiver2` | `video/rtp_video_stream_receiver2.cc` | 收包、组帧、NACK |
| `H26xPacketBuffer` / `PacketBuffer` | `modules/video_coding/` | 按 seq 缓存包 |
| `RtpFrameReferenceFinder` | `modules/video_coding/rtp_frame_reference_finder.h` | 处理参考帧依赖 |
| `VideoStreamBufferController` | `video/video_stream_buffer_controller.cc` | 帧缓冲 + 解码调度 |
| `FrameBuffer` | `api/video/frame_buffer.cc` | 按 PID 排序、等参考帧 |
| `VideoStreamDecoder` | `video/video_stream_decoder2.cc` | 调解码器、回调渲染 |

流程：`OnRtpPacket` → `PacketBuffer::InsertPacket` → `OnAssembledFrame` → `RtpFrameReferenceFinder::ManageFrame` → `OnCompleteFrames` → `VideoReceiveStream2::OnCompleteFrame` → `buffer_->InsertFrame` → `StartNextDecode` → `VideoStreamDecoder` → `OnFrame` → sink。

**辅助**
- `video/timing/`：`jitter_estimator`、`frame_delay_variation_kalman_filter`、`default_video_jitter_timing`
- `video/adaptation/`：降分辨率/降帧率策略
- `video/frame_decode_scheduler.h`：解码时机

**读法**：从两头夹——发送看 `VideoSendStreamImpl::OnEncodedImage`，接收看 `VideoReceiveStream2::OnCompleteFrame`。

---

### 4.6 `audio/` —— 音频收发

**关键类**
- `AudioSendStream`（`audio/audio_send_stream.cc`）：`SendAudioData`
- `ChannelSend`（`audio/channel_send.cc`）：`ProcessAndEncodeAudio`，编码后交 `RtpRtcp`
- `AudioReceiveStream`、`ChannelReceive`（`audio/channel_receive.cc`）：收包 → `NetEq` → 播放
- `AudioTransportImpl`：混音与播放

**实现逻辑**：发送侧 PCM → 编码器（Opus 等）→ RTP；接收侧 RTP → NetEq（抖动缓冲、PLC、变速）→ PCM → 混音播放。

---

### 4.7 `modules/rtp_rtcp/` —— RTP/RTCP 协议栈

**关键类**

| 类 | 文件 | 作用 |
|---|---|---|
| `RtpRtcpImpl2` | `source/rtp_rtcp_impl2.h` | 门面 |
| `RTPSender` / `RTPSenderVideo` / `RTPSenderAudio` | `source/rtp_sender*.h` | 打包与发送 |
| `RtpSenderEgress` | `source/rtp_sender_egress.cc` | 实际发送、重传、优先级 |
| `RtpReceiver` | `source/rtp_receiver*.h` | 收包 |
| `RTCPReceiver` / `RTCPSender` | `source/rtcp_*.h` | RTCP 反馈 |
| `TransportFeedback` | `source/rtcp_packet/transport_feedback.cc` | TWCC 报文 |
| `UlpfecGenerator` / `FlexfecSender` | `source/` | FEC |
| `RtpPacketizerH264/Vp8/Vp9/Av1` | `source/` | 分片 |

**实现逻辑**：发送时按 MTU 分片、加 RTP 头、按优先级入 pacer；接收时解析头、去重、丢包检测触发 NACK；RTCP 周期发 SR/RR，收到 TWCC 反馈交给带宽估计。

---

### 4.8 `modules/pacing/` —— 发送节奏

**关键类**
- `TaskQueuePacedSender`（`task_queue_paced_sender.h`）：对外入口
- `PacingController`（`pacing_controller.h`）：漏桶算法，按 5ms 预算发
- `BitrateProber`：探测可用带宽
- `PrioritizedPacketQueue`：音频 > 重传 > 视频
- `PacketRouter`：把包路由到对应 RTP 模块

**实现逻辑**：目标码率 + 预算 → 每轮取出队列中优先级最高的包 → 交给 `PacketRouter` → socket。探测时短暂超发一簇包，用结果更新带宽估计。

---

### 4.9 `modules/congestion_controller/` —— 带宽估计

**关键类**

| 类 | 文件 | 作用 |
|---|---|---|
| `GoogCcNetworkController` | `goog_cc/goog_cc_network_control.cc` | 总控制器 |
| `DelayBasedBwe` | `goog_cc/delay_based_bwe.cc` | 延迟趋势估带宽 |
| `TrendlineEstimator` | `goog_cc/trendline_estimator.cc` | 趋势线拟合 |
| `AcknowledgedBitrateEstimator` | `goog_cc/acknowledged_bitrate_estimator.cc` | 确认码率 |
| `ProbeController` | `goog_cc/probe_controller.cc` | 探测决策 |
| `AlrDetector` | `goog_cc/alr_detector.cc` | 应用受限检测 |
| `TransportFeedbackAdapter` | `rtp/transport_feedback_adapter.cc` | TWCC 反馈适配 |
| `CongestionControlHandler` | `rtp/control_handler.cc` | 输出整形 |

**实现逻辑**：输入 TWCC 反馈、RTT、丢包率 → 延迟控制器判断 overuse → 结合探测结果 → 输出 `TargetTransferRate`（目标码率 + 网络状态）。

---

### 4.10 `modules/video_coding/` —— 视频编解码与组帧

- `codecs/`：`vp8/`、`vp9/`、`av1/`、`h264/`（`libvpx`/`aom`/`openh264` 封装）
- `svc/`：`ScalabilityStructure`（K-SVC、Full SVC）、`scalability_mode_util`
- `packet_buffer.cc` / `h26x_packet_buffer.cc`：按 seq 组帧
- `rtp_frame_reference_finder.cc`：帧依赖与可解码性
- `frame_dependencies_calculator.cc`：参考帧关系
- `timing/`：`timing.cc`（VCMTiming）
- `nack_requester.cc`：NACK 生成

**实现逻辑**：包进来先按 seq 缓存，凑齐一帧后交给 reference finder 判断是否可解码（依赖的参考帧是否齐），再输出 `EncodedFrame`。

---

### 4.11 `modules/audio_coding/` —— 音频编解码与 NetEq

- `acm2/audio_coding_module.cc`：`AudioCodingModuleImpl`，编码/解码调度
- `neteq/`：`NetEqImpl`，抖动缓冲、丢包隐藏（PLC）、加速/减速
- `audio_network_adaptor/`：按网络状况调编码参数
- `codecs/`：Opus、G.711、AAC 等

**实现逻辑**：NetEq 维护抖动缓冲，根据延迟目标决定"加速播放/减速播放/丢包隐藏"，输出连续 PCM。

---

### 4.12 `modules/audio_processing/` —— 3A

- `audio_processing_impl.cc`：`AudioProcessingImpl`，capture/render 两条流
- `aec3/`：回声消除（`echo_canceller3.cc`）
- `ns/`：降噪
- `agc2/`、`agc/`：自动增益
- `capture_levels_adjuster/`、`capture_mixer/`

**实现逻辑**：render 流提供参考信号 → capture 流依次做 AEC → NS → AGC，10ms 一帧。

---

### 4.13 `p2p/` —— ICE/STUN/TURN/DTLS

- `base/p2p_transport_channel.h`：ICE agent
- `base/basic_ice_controller.h`：候选对选择
- `base/port.cc`、`base/connection.cc`：端口与连接
- `base/stun_port.cc`、`turn_port.cc`：STUN/TURN
- `dtls/dtls_transport.h`：DTLS 握手
- `client/`：P2P 客户端

**实现逻辑**：收集 host/srflx/relay candidate → 配对 → 连通性检查（STUN binding）→ 选出可用连接 → 在其上跑 DTLS。

---

### 4.14 `rtc_base/` —— 基础设施

- `thread.h`、`task_queue`（`api/task_queue/`）：线程与任务
- `async_udp_socket`：异步 UDP
- `ip_address`、`mdns_responder`：地址与 mDNS
- `ssl_*`、`crypto/`：DTLS/加密
- `time_utils`、`bit_buffer`、`checks`

**线程模型**（读代码时最容易绕晕的地方）：
- **signaling thread**：PeerConnection API 调用
- **network thread**：收发包、ICE、RTCP
- **worker thread**：编解码、媒体处理
- 代码里的 `RTC_DCHECK_RUN_ON(&network_sequence_checker_)` 就是线程约束，看到它就知道这段跑在哪个线程。

---

### 4.15 `sdk/` 与外围

- `sdk/android/`：Java API（`api/org/webrtc/*.java`）+ JNI（`src/jni/`）
- `sdk/objc/`：ObjC API
- `logging/rtc_event_log/`：事件日志，配合 `rtc_tools/rtc_event_log_visualizer` 可可视化
- `pc/rtc_stats_collector.cc`：`getStats`
- `test/`：测试工具（`test/fake_*` 很好用）

---

## 5. 五条函数级调用链

### 5.1 视频发送

```
VideoSendStreamImpl::Start
  └─ VideoStreamEncoder::SetSource / Start
       └─ OnFrame（采集线程）
            └─ VideoEncoder::Encode
                 └─ VideoSendStreamImpl::OnEncodedImage
                      └─ RtpVideoSender::OnEncodedImage
                           └─ RTPSenderVideo::SendVideo
                                └─ RtpRtcpImpl2::SendOutgoingData
                                     └─ RtpSenderEgress::SendPacket
                                          └─ TaskQueuePacedSender → PacketRouter → socket
```

### 5.2 视频接收

```
RtpVideoStreamReceiver2::OnRtpPacket
  └─ PacketBuffer / H26xPacketBuffer::InsertPacket
       └─ OnAssembledFrame
            └─ RtpFrameReferenceFinder::ManageFrame
                 └─ OnCompleteFrames
                      └─ VideoReceiveStream2::OnCompleteFrame
                           └─ VideoStreamBufferController::InsertFrame
                                └─ StartNextDecode
                                     └─ VideoStreamDecoder::Decode
                                          └─ VideoReceiveStream2::OnFrame → sink
```

### 5.3 连接建立

```
PeerConnection::CreateOffer
PeerConnection::SetLocalDescription
PeerConnection::SetRemoteDescription
  └─ SdpOfferAnswerHandler
       └─ JsepTransportController::ApplyDescription_n
            └─ MaybeCreateJsepTransport
                 ├─ DtlsTransport（握手）
                 └─ P2PTransportChannel（ICE）
                      └─ Port / Connection（连通性检查）
```

### 5.4 带宽估计闭环

```
收到的 TWCC 包
  └─ TransportFeedbackAdapter::ProcessTransportFeedback
       └─ GoogCcNetworkController::OnTransportPacketsFeedback
            ├─ DelayBasedBwe（趋势）
            ├─ AcknowledgedBitrateEstimator（确认码率）
            └─ ProbeController（探测）
                 └─ TargetTransferRate
                      └─ CongestionControlHandler
                           └─ BitrateAllocator
                                └─ VideoStreamEncoder::OnBitrateUpdated → 编码器 + pacer
```

### 5.5 音频收发

```
发送：AudioSendStream::SendAudioData
   └─ ChannelSend::ProcessAndEncodeAudio
        └─ AudioEncoder::Encode
             └─ RtpRtcpImpl2 → pacer → socket

接收：ChannelReceive::OnReceivedPayloadData
   └─ NetEqImpl::InsertPacket
        └─ NetEqImpl::GetAudio → AudioTransport → 播放
```

---

## 6. 走读计划（4 周，每天 1–1.5 小时）

> 和面试复习并行：上午专题复习，晚上源码走读。周末做串讲图。

### 第 1 周：骨架与连接

| 天 | 目标 | 文件 | 产出 |
|---|---|---|---|
| D1 | 建立全局图 | `api/` 接口扫一遍 | 分层图 |
| D2 | 会话层 | `pc/peer_connection.cc` 三个入口、`pc/sdp_offer_answer.cc` | Offer/Answer 状态机 |
| D3 | 传输建立 | `pc/jsep_transport_controller.cc`、`pc/dtls_transport.cc` | JSEP 时序图 |
| D4 | ICE | `p2p/base/p2p_transport_channel.h`、`basic_ice_controller.cc` | NAT 穿透决策树 |
| D5 | 串讲 | — | 连接建立 15 分钟录音 |

### 第 2 周：视频发送

| 天 | 目标 | 文件 | 产出 |
|---|---|---|---|
| D6 | 流对象 | `call/call.cc`、`video/video_send_stream_impl.cc` | 流创建链路 |
| D7 | 编码调度 | `video/video_stream_encoder.cc` | 码率/降级流程图 |
| D8 | 打包 | `call/rtp_video_sender.cc`、`modules/rtp_rtcp/source/rtp_sender_video.cc` | 分包/FEC 图 |
| D9 | 发送节奏 | `modules/pacing/task_queue_paced_sender.cc`、`pacing_controller.cc` | 漏桶 + 优先级 |
| D10 | 串讲 | — | 发送链路图 |

### 第 3 周：视频接收 + 拥塞控制

| 天 | 目标 | 文件 | 产出 |
|---|---|---|---|
| D11 | 收包组帧 | `video/rtp_video_stream_receiver2.cc` | 组帧状态机 |
| D12 | 帧缓冲 | `video/video_stream_buffer_controller.cc`、`api/video/frame_buffer.cc` | 抖动缓冲图 |
| D13 | 解码渲染 | `video/video_stream_decoder2.cc`、`video_receive_stream2.cc` | 解码链路 |
| D14 | 带宽估计 | `goog_cc/goog_cc_network_control.cc`、`delay_based_bwe.cc` | GCC 控制环 |
| D15 | 串讲 | — | 接收 + 带宽闭环图 |

### 第 4 周：音频 + 编解码 + 端到端

| 天 | 目标 | 文件 | 产出 |
|---|---|---|---|
| D16 | 音频发送 | `audio/audio_send_stream.cc`、`channel_send.cc` | 音频发送链路 |
| D17 | 音频接收 | `audio/channel_receive.cc`、`modules/audio_coding/neteq/neteq_impl.h` | NetEq 状态机 |
| D18 | 3A | `modules/audio_processing/audio_processing_impl.cc`、`aec3/echo_canceller3.cc` | 3A 链路图 |
| D19 | 编解码/SVC | `modules/video_coding/codecs/`、`svc/` | SVC 结构对比 |
| D20 | 端到端串讲 | — | 30 分钟白板讲解 |

---

## 7. 走读方法

1. **先读 unittest**：`*_unittest.cc` 是最好的使用说明，能快速看懂类怎么被调用。
2. **用 BUILD.gn 看依赖**：一个模块依赖谁、被谁依赖，`BUILD.gn` 里很清楚。
3. **用 `git log --follow <file>` 看演进**：理解为什么这么设计。
4. **用 Chromium Codesearch 跳转**：在线看交叉引用比本地 grep 快。
5. **大文件不要通读**：`pc/peer_connection.cc`、`video_stream_encoder.cc` 都上千行，只读入口函数和关键分支。
6. **每读一个模块，画一张图**：画不出来就是没懂。
7. **碰到 `third_party` 的 include 先跳过**：理解主流程不需要看 absl/libyuv 实现。

---

## 8. 每阶段验收问题

**第 1 周**
1. Offer/Answer 里 `a=setup:actpass` 决定什么？
2. ICE 的 host/srflx/relay 三种 candidate 分别在什么场景产生？
3. DTLS 握手完成后，SRTP 密钥怎么来？

**第 2 周**
4. 一帧视频从 `OnFrame` 到发出，经过哪些线程？
5. Simulcast 的三路流在哪一层编码、哪一层选择？
6. pacer 的优先级顺序是什么？为什么音频优先？

**第 3 周**
7. 收到乱序包，`PacketBuffer` 和 `RtpFrameReferenceFinder` 各做什么？
8. `FrameBuffer` 为什么按 PID 而不是按 RTP 时间戳排序？
9. GCC 的延迟控制器如何判断 overuse？
10. TWCC 为什么把计算从接收端搬到发送端？

**第 4 周**
11. NetEq 的加速/减速/PLC 分别在什么条件下触发？
12. AEC3 的双讲检测怎么工作？
13. K-SVC 和 Full SVC 的结构差异？
14. 从 `PeerConnection` 到 `socket`，一共经过几层？

---

## 9. 与旧版路径的差异（避免读错）

| 旧路径（M79/2020） | 现路径（2026-11） |
|---|---|
| `modules/video_coding/jitter_buffer.cc` | `api/video/frame_buffer.cc` + `video/video_stream_buffer_controller.cc` |
| `modules/congestion_controller/rtp/transport_feedback`（目录） | `source/rtcp_packet/transport_feedback.cc` + `rtp/transport_feedback_adapter.cc` |
| `video/rtp_video_sender.*` | `call/rtp_video_sender.*` |
| `modules/congestion_controller/send_side_congestion_controller.*` | 已并入 `call/rtp_transport_controller_send.*` |
| 顶层 `base/` | 已移除，改用 absl |

> 你 SSD 上那份 2019 M79 工程可以对照这些差异看演进，但别按它的路径读新代码。
