# 单元测试运行逻辑：以 `PeerConnectionEndToEndTest.Call` 为例

> 环境：`/Volumes/SSD/Code/webrtc-build/src`，目标 `pc:peerconnection_unittests`
> 实测命令：
> ```bash
> ./webrtc-test.sh peerconnection_unittests \
>   --gtest_filter='PeerConnectionEndToEndTest/PeerConnectionEndToEndTest.Call/0'
> ```
> 实测耗时 346–419 ms，日志约 1280 行。

---

## 0. 一句话概括

一个 gtest 二进制就是：**`main` → `TestMain` 初始化 → `RUN_ALL_TESTS()` → 逐个执行 `TEST_P`**。
每个测试是一个 fixture 对象：**构造函数搭环境 → `TestBody` 跑场景 → 析构函数清理**。
参数化的 `TEST_P` 会被展开成 `Call/0`、`Call/1` 两个独立测试。

---

## 1. 进程启动链路

```
main()                                  test/test_main.cc:58
 ├─ testing::InitGoogleMock(&argc, argv)   // 解析 --gtest_filter 等
 ├─ absl::ParseCommandLine                  // 解析 --log_level/--logs/--verbose
 ├─ TestMain::Create()                      // test_main_lib.cc:397
 ├─ main->Init()                            // test_main_lib.cc:192
 │   ├─ 设置日志级别/输出                   // :198-207
 │   ├─ metrics::Enable()
 │   └─ InitializeSSL()                     // DTLS 需要
 └─ main->Run(argc, argv)                   // test_main_lib.cc:230
     └─ RUN_ALL_TESTS()                     // test_main_lib.cc:269
```

关键点：

- `--log_level` 在这里落地（`config.set_min_severity`），**默认是 `LS_WARNING`**，所以 `RTC_LOG(LS_INFO)` 默认看不到。
- `InitializeSSL()` 在这里调用，后面 DTLS 握手才能跑。
- `RUN_ALL_TESTS()` 之后还会输出 metrics（`RESULT: ... TotalTestCount` 那几行）。

---

## 2. 测试是怎么注册的

`pc/peer_connection_end_to_end_unittest.cc`：

```cpp
// :395
TEST_P(PeerConnectionEndToEndTest, Call) {
  scoped_refptr<AudioDecoderFactory> real_decoder_factory =
      CreateOpusAudioDecoderFactory();
  CreatePcs(CreateOpusAudioEncoderFactory(),
            CreateForwardingMockDecoderFactory(real_decoder_factory.get()));
  GetAndAddUserMedia();
  Negotiate();
  WaitForCallEstablished();
}

// :784
INSTANTIATE_TEST_SUITE_P(PeerConnectionEndToEndTest,
                         PeerConnectionEndToEndTest,
                         Values(SdpSemantics::kPlanB_DEPRECATED,
                                SdpSemantics::kUnifiedPlan));
```

`Values(...)` 有两个参数，所以生成两个实例：

```
PeerConnectionEndToEndTest/PeerConnectionEndToEndTest.Call/0   ← PlanB
PeerConnectionEndToEndTest/PeerConnectionEndToEndTest.Call/1   ← UnifiedPlan
```

这就是为什么测试名里带 `/0`，也是为什么 filter 必须加引号（zsh 会展开 `*`）。

---

## 3. Fixture：搭一个"假但真实"的环境

### 3.1 构造函数

`PeerConnectionEndToEndBaseTest`（:77）：

```cpp
explicit PeerConnectionEndToEndBaseTest(SdpSemantics sdp_semantics)
    : env_(CreateTestEnvironment()),
      network_thread_(std::make_unique<Thread>(&pss_)) {
  RTC_CHECK(network_thread_->Start());
  caller_ = make_ref_counted<PeerConnectionTestWrapper>(
      "caller", env_, &pss_, network_thread_.get());
  callee_ = make_ref_counted<PeerConnectionTestWrapper>(
      "callee", env_, &pss_, network_thread_.get());
  ...
}
```

它创建了：

| 对象 | 作用 |
|---|---|
| `PhysicalSocketServer` | 套接字服务器（供测试线程使用） |
| `Thread network_thread_` | 模拟网络线程 |
| `caller_` / `callee_` | 两端 PeerConnection 的封装 |
| `RTCConfiguration` | SDP 语义（PlanB / UnifiedPlan） |

### 3.2 `CreatePcs()`：造两端 PeerConnection

:97 调用 `PeerConnectionTestWrapper::CreatePc`（wrapper:175），内部：

```cpp
auto port_allocator =
    std::make_unique<FakePortAllocator>(env, socket_server_, network_thread_);
fake_audio_capture_module_ = FakeAudioCaptureModule::Create();  // :190
...
dependencies.adm = fake_audio_capture_module_;
...
cert_generator = std::make_unique<FakeRTCCertificateGenerator>();  // :215
```

- **FakePortAllocator**：ICE 候选是伪造的，不走真实网络
- **FakeAudioCaptureModule**：假音频采集
- **FakeRTCCertificateGenerator**：假证书
- 但 **DTLS 用的是真 OpenSSL**，SDP、RTP、编解码都是真实代码

然后 `Connect()`（wrapper:117）把两端的回调接起来：

```cpp
caller->SubscribeOnIceCandidateReady(callee, ...);  // 候选互发
callee->SubscribeOnIceCandidateReady(caller, ...);
caller->SubscribeOnSdpReady(callee, ...);           // Offer 交给对方
callee->SubscribeOnSdpReady(caller, ...);           // Answer 交回
```

### 3.3 `GetAndAddUserMedia()`：加音视频轨

:123 → wrapper:496：

```cpp
GetUserMedia(audio, audio_options, video);   // 内部创建 fake 源
peer_connection_->AddTrack(audio_track, {stream->id()});
peer_connection_->AddTrack(video_track, {stream->id()});
```

- 音频：`CreateAudioSource` + `FakeAudioCaptureModule`
- 视频：`FakePeriodicVideoTrackSource`（按固定帧率产生假画面）

### 3.4 `Negotiate()`：走一遍 Offer/Answer

:135：

```cpp
caller_->CreateOffer(RTCOfferAnswerOptions());
```

后续由 wrapper 的 SDP 回调驱动：

```
caller CreateOffer
  → SdpReady → callee ReceiveOfferSdp
  → callee CreateAnswer
  → SdpReady → caller ReceiveAnswer
  → ICE candidates 互发
```

### 3.5 `WaitForCallEstablished()`：等待三个条件

wrapper:435：

```cpp
if (!WaitForConnection()) return false;  // ICE connected
if (!WaitForAudio()) return false;       // 收到足够音频帧
if (!WaitForVideo()) return false;       // 渲染足够视频帧
```

| 等待 | 判定 | 日志 |
|---|---|---|
| 连接 | `ice_connection_state == connected/completed` | `Connected.` |
| 音频 | `frames_received >= kTestAudioFrameCount` | `Got enough audio frames.` |
| 视频 | `renderer_->num_rendered_frames() >= kTestVideoFrameCount` | `Got enough video frames.` |

---

## 4. 实测日志逐段对照

下面是这次运行的关键日志（节选），按执行顺序排列：

```
[ RUN      ] PeerConnectionEndToEndTest/PeerConnectionEndToEndTest.Call/0
(peer_connection_factory.cc:252): PeerConnection constructed with legacy SDP semantics!
(peer_connection_test_wrapper.cc:385): PeerConnectionTestWrapper caller: CreateOffer.
(peer_connection.cc:1495): === MY DEBUG LOG: PeerConnection::CreateOffer called ===
(p2p_transport_channel.cc:494): Set ICE ufrag: h/AC pwd: ... on transport audio
(p2p_transport_channel.cc:494): Set ICE ufrag: h/AC pwd: ... on transport video
(p2p_transport_channel.cc:505): Received remote ICE parameters: ufrag=...
(peer_connection_test_wrapper.cc:392): PeerConnectionTestWrapper callee: CreateAnswer.
(dtls_transport.cc:592): DtlsTransport[audio|1|__]: DTLS setup complete
(dtls_transport.cc:1076): ... Start DTLS handshake active=1 role=client
(dtls_transport.cc:1076): ... Start DTLS handshake active=1 role=server
(dtls_transport.cc:1001): DtlsTransport[audio|1|__]: DTLS handshake complete.
(peer_connection.cc:2613): Changing to ICE connected state because all transports are writable.
(peer_connection_test_wrapper.cc:451): PeerConnectionTestWrapper caller: Connected.
(peer_connection_test_wrapper.cc:468): PeerConnectionTestWrapper caller: Got enough audio frames.
(peer_connection_test_wrapper.cc:484): PeerConnectionTestWrapper caller: Got enough video frames.
(peer_connection_test_wrapper.cc:451): PeerConnectionTestWrapper callee: Connected.
(peer_connection.cc:2640): Changing to ICE completed state because all transports are complete.
(dtls_transport.cc:1032): DtlsTransport[audio|1|_W]: DTLS transport closed by remote
[       OK ] PeerConnectionEndToEndTest/PeerConnectionEndToEndTest.Call/0 (346 ms)
```

| 日志 | 代码位置 | 阶段 |
|---|---|---|
| `PeerConnection constructed` | `pc/peer_connection_factory.cc:252` | CreatePcs 建好两端 |
| `caller: CreateOffer.` | `pc/test/peer_connection_test_wrapper.cc:385` | Negotiate 开始 |
| `MY DEBUG LOG ... CreateOffer` | `pc/peer_connection.cc:1495` | **我们加的日志** |
| `Set ICE ufrag` | `p2p/base/p2p_transport_channel.cc:494` | 本地 ICE 参数 |
| `Received remote ICE parameters` | 同文件 :505 | 收到对端 ICE 参数 |
| `callee: CreateAnswer.` | wrapper:392 | 对端回 Answer |
| `DTLS setup complete` | `p2p/dtls/dtls_transport.cc:592` | DTLS 上下文就绪 |
| `Start DTLS handshake` | `p2p/dtls/dtls_transport.cc:1076` | 客户端/服务端角色确定 |
| `DTLS handshake complete` | `p2p/dtls/dtls_transport.cc:1001` | 握手完成，SRTP 密钥就绪 |
| `Changing to ICE connected` | `pc/peer_connection.cc:2613` | ICE 连通 |
| `Connected.` | wrapper:451 | WaitForConnection 通过 |
| `Got enough audio frames.` | wrapper:468 | WaitForAudio 通过 |
| `Got enough video frames.` | wrapper:484 | WaitForVideo 通过 |
| `Changing to ICE completed` | `pc/peer_connection.cc:2640` | 连接稳定 |
| `closed by remote` | `p2p/dtls/dtls_transport.cc:1032` | 测试结束，析构关闭 |
| `[ OK ] ... (346 ms)` | gtest | 整体通过 |

> 日志格式 `(文件:行): 消息`，可以直接 `vim +行号 文件` 跳过去看上下文。

---

## 5. 时序图

```
gtest                caller (PC)              callee (PC)
  │                     │                       │
  ├─ SetUp: 构造 fixture ────────────────────────┤
  │   env / network thread / 两个 wrapper        │
  ├─ CreatePcs ─────────┤                       │
  │                     │ FakePortAllocator     │
  │                     │ FakeAudioCapture      │
  ├─ GetAndAddUserMedia ┤ AddTrack(audio/video)  │
  ├─ Negotiate ─────────┤ CreateOffer           │
  │                     ├── SDP ───────────────►│ SetRemoteDescription
  │                     │                       ├─ CreateAnswer
  │                     │◄──────── SDP ─────────┤ SetLocalDescription
  │                     ├─ SetRemoteDescription  │
  │                     ├── ICE candidates ────►│
  │                     │◄── ICE candidates ────┤
  │                     ├── DTLS handshake ────►│
  │                     │◄── DTLS handshake ────┤
  ├─ WaitForCallEstablished                    │
  │   ├─ ICE connected   │                       │
  │   ├─ audio frames    │                       │
  │   └─ video frames    │                       │
  ├─ TearDown: 析构 fixture ─────────────────────┤
  │   StopFakeVideoSources / pc->Close()         │
  └─ [ OK ]                                     │
```

---

## 6. 用这套方法学源码的步骤

1. **选一个测试**：先看 `TEST_P` 的函数体，通常只有 5–10 行，场景一目了然。
2. **读 fixture**：顺着 `CreatePcs` / `GetAndAddUserMedia` / `Negotiate` / `WaitFor...` 读下去，环境是怎么搭的。
3. **跑一遍存日志**：
   ```bash
   ./webrtc-test.sh peerconnection_unittests \
     --gtest_filter='PeerConnectionEndToEndTest/PeerConnectionEndToEndTest.Call/0' \
     > /tmp/e2e.log 2>&1
   ```
4. **按阶段切日志**：
   ```bash
   grep -nE "CreateOffer|CreateAnswer|DTLS|ICE connected|Got enough" /tmp/e2e.log
   ```
5. **加日志到你想看的点**，重编重跑：
   ```bash
   autoninja -C out/Debug pc:peerconnection_unittests
   ./webrtc-test.sh peerconnection_unittests --gtest_filter='...'
   ```
6. **对照行号读源码**：日志里 `(file:line)` 就是最好的路标。

---

## 7. 关键文件

| 文件 | 作用 |
|---|---|
| `test/test_main.cc` | 进程入口 |
| `test/test_main_lib.cc` | 日志、SSL、`RUN_ALL_TESTS` |
| `pc/peer_connection_end_to_end_unittest.cc` | 测试定义与 fixture |
| `pc/test/peer_connection_test_wrapper.cc` | 两端 PC 的封装、SDP/ICE 回调 |
| `p2p/base/p2p_transport_channel.cc` | **ICE 状态机**（GetState/ComputeState） |
| `p2p/dtls/dtls_transport.cc` | DTLS 状态机与握手（日志在这里） |
| `pc/dtls_transport.cc` | DTLS 的 API 层封装（不含握手实现） |
| `pc/peer_connection.cc` | 连接状态机 |

---

## 8. 注意点

1. **网络是假的**：`FakePortAllocator` 不真正打洞，但 ICE 状态机、DTLS、RTP、编解码都是真的。
2. **音视频源是假的**：`FakeAudioCaptureModule`、`FakePeriodicVideoTrackSource`，但编码器是真实 VP8/VP9/H264。
3. **日志默认 `LS_WARNING`**：不加 `--log_level=info` 看不到 `RTC_LOG(LS_INFO)`。
4. **测试名带 `/0` `/1`**：参数化实例，filter 要写全或加引号通配。
5. **耗时不代表复杂度**：346ms 是因为 fake 网络和 fake 源，真实网络会慢得多。
