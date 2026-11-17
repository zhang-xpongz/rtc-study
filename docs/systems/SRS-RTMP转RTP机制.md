# SRS 的 RTMP → RTP 机制：按需桥接，不是预转换

> 基于 2026-11-12 的实验：ffmpeg 推 RTMP → SRS 5.0.225 → Chrome 用 WHEP 拉流。
> 目的：讲清楚「什么时候转、谁触发、转的是什么、代码在哪」。

---

## 1. 结论先行

**SRS 不会在收到 RTMP 推流时默认把流转换成 RTP，也不是靠推流 URL 里的标记。**

真正的机制是：

> **按需桥接（on-demand bridge）**：只有当某个 WebRTC 订阅者（WHEP）请求这条流时，SRS 才为这个会话临时创建一座 RTMP → RTP 的桥；会话结束桥就销毁。没有 WebRTC 观众，就不会产生任何 RTP 包。

---

## 2. 两条路径是分开的

### 2.1 推流路径（ingest）

```
ffmpeg ──RTMP──► SRS RTMP 接入 ──► LiveSource（内存中的流）
                                      │
                                      ├──► RTMP 观众
                                      ├──► HTTP-FLV 观众
                                      ├──► HLS 切片
                                      └──► WebRTC 观众（只有有人订阅时才出现）
```

- RTMP 接入只负责：鉴权（可选）、解 FLV、缓存 GOP、把帧放进 `LiveSource`。
- 它**不关心**后面有没有 WebRTC 观众，也不做任何 RTP 相关工作。

### 2.2 拉流路径（delivery）

```
Chrome ──WHEP 请求──► SRS HTTP ──► 找到 LiveSource ──► 创建 RTC 桥 ──► 打 RTP ──► SRTP ──► UDP
```

- 每个 WebRTC 订阅者对应**一个桥、一个 RTC 会话**。
- 桥是消费者：订阅 `LiveSource`，逐帧把 RTMP 帧转成 RTP。
- 订阅者断开，桥销毁。

---

## 3. 触发链路：一次 WHEP 请求发生了什么

请求：

```
POST /rtc/v1/whep/?app=live&stream=livestream
Content-Type: application/sdp

<offer SDP>
```

SRS 内部顺序：

1. HTTP 层解析 query，得到 `app=live`、`stream=livestream`
2. 用 `vhost + app + stream` 找到对应的 `LiveSource`
3. 判断请求方是 RTC 会话 → 创建 `SrsRtcFromRtmpBridge`
4. 桥作为消费者订阅 `LiveSource`（从最近的 IDR 开始）
5. 生成 answer SDP（含 H.264 参数、payload type、rtcp-fb）
6. 每来一帧：解封装 → 打包 RTP → SRTP 加密 → UDP 发送

**没有第 1 步的 WHEP 请求，第 3 步不会发生。**

---

## 4. 转换的到底是什么

不是转码，是**转封装 + 协议适配**。码流内容不变。

### 4.1 H.264：AVCC → Annex-B

| 形态 | 结构 |
|---|---|
| FLV / AVCC | `[4 字节长度][NALU]` |
| RTP / Annex-B | `00 00 00 01 [NALU]` |

SRS 把长度前缀替换成起始码，然后按 RFC 6184 打包：

| 情况 | 打包方式 |
|---|---|
| NALU < MTU | Single NAL Unit |
| 多个小 NALU | STAP-A |
| NALU > MTU（关键帧常见） | FU-A 分片 |

### 4.2 时间戳映射

| 媒体 | RTMP | RTP |
|---|---|---|
| 视频 | 毫秒 | 90 kHz（×90） |
| 音频 | 毫秒 | 采样率（如 48 kHz） |

每路流一个 SSRC，序列号连续递增，一帧最后一个包 `marker=1`。

### 4.3 SDP 参数

SPS/PPS 来自 FLV 首帧的 `AVCDecoderConfigurationRecord`，写进 SDP：

```
a=fmtp:102 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=rtcp-fb:102 nack
a=rtcp-fb:102 nack pli
```

### 4.4 加密

DTLS 握手后导出 SRTP 密钥，RTP 包加密再发。所以抓包看不到 payload。

### 4.5 音频

日志 `RTMP2RTC: Init audio codec to 10(AAC)` 说明 SRS 直接把 AAC 发给浏览器，**没有转 Opus**。Chrome 支持 AAC，所以协商成功。若对方只支持 Opus，才需要转码（另一条重路径）。

---

## 5. 同一路流可以同时有多种观众

| 观众请求 | 走哪条路 | 是否产生 RTP |
|---|---|---|
| `rtmp://host/live/livestream` | RTMP 分发 | 否 |
| `http://host/live/livestream.flv` | HTTP-FLV | 否 |
| `http://host/live/livestream.m3u8` | HLS | 否 |
| `POST /rtc/v1/whep/?app=live&stream=livestream` | 临时建桥 | **是** |

四种观众可以同时在线，各拿各的格式，互不影响。

---

## 6. 反向：RTC → RTMP 也是按需

如果是 WHIP 推流（WebRTC 发布），SRS 收到的是 RTC 源。这时有 RTMP 观众来拉，同样按需建反方向的桥（RTC → RTMP），而不是推流时就转好。

---

## 7. 配置项

| 配置 | 作用 |
|---|---|
| `rtc_server { enabled on; listen 8000; }` | 打开 WebRTC 能力（UDP 端口） |
| `rtc_server { candidate <IP>; }` | 告诉浏览器连哪个地址（Docker 里必须设宿主机 IP） |
| vhost 下 `rtc { enabled on; }` | 该 vhost 允许 WebRTC 播放 |
| `rtc.keep_bframe` | 是否保留 B 帧（默认关，WebRTC 兼容性差） |
| `gop_cache` | 新观众从最近 IDR 开始，保证秒开 |

**没有**“这条流要不要转 RTP”的 per-stream 开关。能力是全局/vhost 级的，转换是会话级的。

---

## 8. 常见误区

| 误区 | 事实 |
|---|---|
| “SRS 收到 RTMP 就自动转 RTP” | 没有观众就不转 |
| “URL 里要带个 rtc 标记” | 不需要，URL 只是 `vhost/app/stream` 寻址 |
| “转换是转码” | 转封装，不重新编码 |
| “一条流只有一份 RTP” | 每个 WebRTC 订阅者一个桥 |
| “PLI 能让 RTMP 推流端立刻出关键帧” | RTMP 端不响应，SRS 只能等自然 IDR 或用 GOP 缓存 |

---

## 9. 代码结构（SRS 5，文件名可能随版本微调）

| 层 | 职责 |
|---|---|
| `SrsRtmpConn` | RTMP 连接、握手、推拉流 |
| `SrsLiveSource` | 流对象、GOP 缓存、消费者管理 |
| `SrsRtcServer` / `SrsRtcConnection` | RTC 会话、DTLS、SRTP、UDP 传输 |
| `SrsRtcFromRtmpBridge` | **核心桥接**：订阅 LiveSource，逐帧转 RTP |
| `SrsRtpPacketizer` | H.264 FU-A/STAP-A、AAC 打包 |
| `SrsRtcSdp` | answer SDP 生成（fmtp、rtcp-fb） |

大致在 `src/app/srs_app_rtc_*` 和 `src/protocol/srs_protocol_rtp*` 下。

---

## 10. 面试答法

> “SRS 的 RTMP→WebRTC 不是预转换，是按订阅者桥接。推流只把帧放进 LiveSource；当 WHEP 请求进来，SRS 才为该会话创建 RTC 桥，订阅源流，把 FLV 里的 H.264 从 AVCC 转 Annex-B，按 RFC 6184 打成 RTP，时间戳映射到 90kHz，SPS/PPS 写进 SDP，最后 SRTP 加密走 UDP。同一路流可以同时有 RTMP、FLV、HLS、WebRTC 观众，各走各的分发路径。”

---

## 11. 实测记录

```
推流：ffmpeg → rtmp://localhost:1935/live/livestream
拉流：POST http://localhost:1985/rtc/v1/whep/?app=live&stream=livestream
日志：RTC: Subscriber url=/live/livestream established
      RTC: start play url=/live/livestream, realtime=1
      RTC: Server conns=1, spkts=(349,rtp:343), rtcp=(pli:2)
结果：Chrome 解码 8 帧，1280×720，31fps，jitter 3ms，0 丢包
```

`rtp:343` 是桥发出的包，`pli:2` 是 Chrome 请求关键帧——都发生在 WHEP 会话建立之后，印证了“按需桥接”。
