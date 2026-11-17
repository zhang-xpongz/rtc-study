# 补充：Opus 与 AAC 的延迟、码率自适应、抗弱网对比

> 配套 D1「音频基础」1.4 节。
> 一句话：**Opus 是为「实时交互」设计的，AAC 是为「存储/广播」设计的。延迟、自适应、抗弱网这三件事，一半由算法决定，一半由设计目标和传输栈决定。**

---

## 0. 术语表（先读这里）

### 0.1 延迟相关

| 术语 | 英文 | 含义 | 量级 |
|---|---|---|---|
| 算法延迟 | algorithmic delay | 编解码器本身引入的延迟，不含网络和缓冲 | ms |
| 帧长 | frame size / frame duration | 一帧覆盖的音频时长 | Opus 2.5–60ms；AAC 约 21.3ms |
| 前视 | lookahead | 编码器需要「未来样本」才能编码当前帧 | Opus 2.5–6.5ms；AAC ≥1024 样本 |
| 预热 | priming / encoder delay | 编码器在真实音频前补的静音样本 | AAC ≥1024 样本 |
| MDCT | modified DCT | 改进离散余弦变换，50% 重叠 | AAC 2048 点；CELT 低重叠 |
| 重叠 | overlap | 相邻变换窗共享的样本比例 | AAC 50%；CELT 约 2.5ms 固定 |
| 抖动缓冲 | jitter buffer | 接收端吸收网络抖动的缓冲 | 通常 1–3 帧 |

### 0.2 Opus 结构

| 术语 | 含义 |
|---|---|
| SILK | 语音编码层，LPC + 基音预测，帧长 10/20/40/60ms |
| CELT | 音乐/低延迟层，低重叠 MDCT，帧长 2.5/5/10/20ms |
| Hybrid | SILK + CELT 混合，帧长 10/20ms |
| TOC | Opus 包头的 Table of Contents 字节，自描述模式/帧长/带宽/帧数 |
| LBRR | Low Bit-Rate Redundancy，SILK 层的带内 FEC |
| DTX | Discontinuous Transmission，静音时不发包 |
| PLC | Packet Loss Concealment，丢包隐藏 |

### 0.3 AAC 结构

| 术语 | 含义 |
|---|---|
| AAC-LC | Low Complexity，最常用档次，1024 样本帧 |
| MDCT 长块/短块 | 2048 点（长块）/ 256 点（短块，8 个短块） |
| TNS | 时域噪声整形 |
| PNS | 感知噪声替换 |
| SBR | 频带复制，HE-AAC 用，增加高频但**增加延迟** |
| PS | 参数立体声，HE-AAC v2 |
| ADTS | 带同步字的 AAC 传输格式，头里带 profile/采样率/声道 |
| ASC | AudioSpecificConfig，MP4 里的编解码配置 |

### 0.4 传输与鲁棒性

| 术语 | 含义 |
|---|---|
| RTP 载荷 | Opus 用 RFC 7587（一包一 Opus 包）；AAC 用 RFC 3640（带 AU headers） |
| 带内 FEC | 编码器在码流里塞冗余，丢包时可恢复 |
| 传输层 FEC | RTP 层的 ULPFEC/FlexFEC，与编码器无关 |
| 队头阻塞 | TCP 丢包后后续数据被阻塞 | 
| CBR/VBR | 恒定/可变码率 |

---

## 1. 结论先行

| 维度 | Opus | AAC-LC |
|---|---|---|
| 设计目标 | 实时交互（RTP/WebRTC） | 存储/广播（MP4/HLS/RTMP） |
| 默认算法延迟 | **26.5ms**（20ms 帧） | **约 42.7ms**（2048 样本 @48k） |
| 最低延迟 | **5ms**（2.5ms 帧，CELT） | 标准 LC 做不到；需 AAC-LD/ELD |
| 帧长 | 2.5–60ms，可逐包变 | 固定 1024 样本（约 21.3ms） |
| 码率范围 | 6–510 kbps，**逐帧可改** | 通常会话内固定 |
| 带内 FEC | ✅ SILK LBRR | ❌ 标准里没有 |
| DTX | ✅ | ❌ |
| PLC | ✅ 解码器内置 | 依赖实现，非标准 |
| RTP 封装 | 简单（无 AU 头） | 需要 AU headers，开销更大 |
| 典型场景 | WebRTC、Zoom、Discord | HLS、MP4、RTMP、音乐 |

> 结论：**不是「Opus 算法更先进」，而是两者的优化目标不同。** Opus 用「短帧 + 低重叠 + 自描述包头 + 带内冗余」换低延迟和鲁棒性；AAC 用「长帧 + 高重叠 + 固定配置」换高码率下的编码效率。

---

## 2. 延迟：从帧长和 lookahead 算起

### 2.1 延迟的构成

```
端到端延迟 = 采集缓冲 + 编码算法延迟 + 打包 + 网络 + 抖动缓冲 + 解码 + 播放缓冲
```

编码器能控制的只有「算法延迟」：

```
算法延迟 ≈ 帧长 + lookahead
```

- 帧长：必须攒够一帧才能编码。
- lookahead：编码器需要未来样本（窗函数、噪声整形、线性预测）。

### 2.2 Opus 的延迟

| 帧长 | 模式 | lookahead | 算法延迟 |
|---|---|---|---|
| 2.5 ms | CELT | 2.5 ms | **5 ms** |
| 5 ms | CELT | 2.5 ms | 7.5 ms |
| 10 ms | CELT/SILK | 2.5/6.5 ms | 12.5–16.5 ms |
| 20 ms | SILK/Hybrid | 6.5 ms | **26.5 ms**（默认） |
| 40 ms | SILK | 6.5 ms | 46.5 ms |
| 60 ms | SILK | 6.5 ms | 66.5 ms |

关键设计：

- **CELT 用低重叠 MDCT**，lookahead 只有 2.5ms，所以能做到 5ms 延迟。
- **SILK 有 5ms 噪声整形 lookahead + 1.5ms 重采样延迟 = 6.5ms**。
- 帧长短于 10ms 时强制走 CELT（SILK 不支持）。

### 2.3 AAC 的延迟

AAC-LC 的变换结构（Apple 文档）：

- 对 **2048 个连续样本**做 MDCT，每 **1024 个样本**滑动一次（50% 重叠）。
- 为了正确解码任意 1024 样本，需要前后两个变换窗。
- 编码器至少补 **1024 样本静音**（priming），解码器还需要 **1024 样本**预卷。

所以：

```
编码器延迟 ≥ 1024 样本
解码器延迟 ≈ 1024 样本
合计 ≈ 2048 样本 = 42.7 ms @ 48 kHz
```

再加上：

- **AAC 帧长固定 1024**（21.3ms），无法像 Opus 那样切到 2.5ms。
- **HE-AAC 的 SBR/PS 额外增加延迟**，端到端常见 60–100ms。
- 块切换（短块 256）是为了抗预回声，不是为了降延迟。

### 2.4 对比

| | Opus 默认 | Opus 最低 | AAC-LC | HE-AAC |
|---|---|---|---|---|
| 算法延迟 | 26.5 ms | 5 ms | ≈42.7 ms | 60–100 ms+ |

> 注意：这里的数字是**算法延迟**。实际端到端还要加抖动缓冲，Opus 也常配 40–60ms 的 jitter buffer。

---

## 3. 码率自适应：为什么 Opus 能逐帧改

### 3.1 Opus 的包头是自描述的

Opus 包的第一个字节是 **TOC**，里面编码了：

- 模式（SILK / CELT / Hybrid）
- 帧长（2.5–60ms）
- 带宽（NB/WB/SWB/FB）
- 声道数（mono/stereo）
- 帧数（一包可含 1–48 帧）

这意味着：**每个包自带解码所需信息，接收端不需要重新协商**。发送端可以在任意一帧切换码率、模式、带宽、帧长。

### 3.2 编码器 API 允许随时改

```c
opus_encoder_ctl(enc, OPUS_SET_BITRATE(bitrate));   // 任意时刻
opus_encoder_ctl(enc, OPUS_SET_FORCE_MODE(...));
```

配合 GCC 的带宽估计，可以做到每 20ms 调一次目标码率，不需要信令。

### 3.3 码率范围宽

- 6 kbps 窄带语音（极端弱网）
- 16–32 kbps 宽带语音（通话）
- 64–128 kbps 音乐
- 510 kbps 全带宽立体声

### 3.4 AAC 为什么不行

- **ADTS/ASC 是会话级配置**：profile、采样率、声道在流开始时确定，包头里没有「本帧码率」。
- **帧预算固定**：每帧时长固定 21.3ms，码率 × 帧时长 = 每帧比特数；改码率意味着改每帧比特预算，编码器的心理声学模型和比特分配需要重新收敛。
- **没有逐帧模式切换**：AAC 不能在这一帧用语音模式、下一帧用音乐模式。
- **传输栈通常固定码率**：RTMP/HLS 的生产链路一般按固定码率配置，运行时改码率需要重新协商或重启编码器。

> 所以 AAC 也能做 VBR，但它是「平均码率可变」，不是「每帧可随网络瞬时调整」。

---

## 4. 抗弱网：codec 级工具 vs 传输层

### 4.1 Opus 的三件套

| 工具 | 作用 | 代价 |
|---|---|---|
| **带内 FEC（LBRR）** | 在 SILK 帧里嵌入前一帧的低码率冗余，丢包时可恢复 | 额外码率；只能修复**前一帧**；不是所有模式都有 |
| **DTX** | 静音段不发包，只发舒适噪声参数 | 需要 VAD，音乐场景不适用 |
| **PLC** | 解码器用前一帧外推，掩盖丢包 | 音质下降，长丢包会失真 |

配合小帧（20ms），抖动缓冲可以做得更细，丢包影响范围更小。

### 4.2 AAC 为什么弱

- **标准 AAC-LC 没有带内 FEC**：丢包只能靠解码器 PLC（实现相关，质量参差）。
- **没有 DTX**：静音也照发。
- **RTP 封装更重**：RFC 3640 需要 AU-headers（每个 AU 的大小/长度），增加开销和解析复杂度。
- **常见传输是 RTMP over TCP**：丢包触发 TCP 重传，**队头阻塞**会把延迟推高，这比编码器本身更致命。

### 4.3 但要公平

- AAC **可以**跑在 RTP 上，配合 ULPFEC/FlexFEC、NACK，抗弱网并不必然差。
- Opus 的带内 FEC 也有局限：只在 SILK 层、只能修复前一帧、修复质量低。
- 「AAC 抗弱网差」更多是**部署方式**（RTMP/TCP、无 FEC）造成的，不完全是算法。

---

## 5. 算法结构差异

### 5.1 Opus 的双层结构

```
SILK：语音，LPC + 基音预测，10–60ms 帧，6.5ms lookahead
CELT：音乐，低重叠 MDCT，2.5–20ms 帧，2.5ms lookahead
Hybrid：两者结合，10/20ms 帧
```

编码器按码率、带宽、内容自动选层：

- 低码率语音 → SILK
- 高码率音乐 → CELT
- 中码率混合 → Hybrid

### 5.2 AAC 的单层结构

```
MDCT（长块 2048 / 短块 256） + 心理声学模型 + 量化 + 熵编码
+ TNS / PNS / M-S / 强度立体声
```

- 帧长固定，块切换只为抗瞬态。
- 为存储优化：高码率下编码效率高，但不考虑实时交互。

### 5.3 为什么算法决定了下限

- **MDCT 的重叠长度决定 lookahead**：AAC 50% 重叠 → 至少 1024 样本；CELT 低重叠 → 2.5ms。
- **帧长决定最小延迟**：AAC 固定 1024；Opus 可到 2.5ms。
- **包头设计决定能否逐帧自适应**：Opus TOC 自描述；AAC ADTS/ASC 是会话级配置。

这三条是「算法层面」的决定性因素，不是实现优化能绕过的。

---

## 6. 公平比较

| 场景 | 更合适 | 原因 |
|---|---|---|
| WebRTC 通话 | Opus | 低延迟、FEC、DTX、逐帧自适应 |
| 直播（HLS/RTMP） | AAC | 生态成熟、硬件支持好、高码率音质好 |
| 音乐存储 | AAC/FLAC | 高码率效率高；无损用 FLAC |
| 会议（低延迟） | Opus / AAC-LD/ELD | AAC-LD 专为低延迟设计，但生态小 |
| 极致弱网语音 | Opus（SILK 模式） | 6kbps 仍可懂，带内 FEC |

补充：

- **AAC-LD/ELD** 是 AAC 的低延迟变体，算法延迟可到 15–25ms，但主要用于专业会议/广播，WebRTC 不常用。
- **Opus 在高码率音乐上不一定赢 AAC**：128kbps 以上，AAC-LC 的编码效率仍然很强。
- **HE-AAC 的 SBR 不是为延迟设计**，它是为了低码率下的高频重建，代价是延迟更高。

---

## 7. 面试答法

### 30 秒版

> Opus 的低延迟来自短帧和低重叠 MDCT，默认 20ms 帧 26.5ms 算法延迟，最低 5ms；码率自适应来自自描述 TOC 和可逐帧改的编码器 API；抗弱网来自 SILK 带内 FEC、DTX 和 PLC。AAC-LC 的 2048 点 MDCT 和 1024 样本预热决定了它约 42.7ms 的延迟下限，标准里没有带内 FEC/DTX，加上常跑在 RTMP/TCP 上，所以实时性差。

### 2 分钟版

1. **延迟**：算法延迟 ≈ 帧长 + lookahead。Opus 帧长可到 2.5ms，CELT lookahead 2.5ms；AAC 固定 1024 样本帧、50% 重叠，编解码各 1024 预热，合计约 2048 样本。
2. **自适应**：Opus 每包 TOC 自描述模式/帧长/码率，解码器无需重协商；AAC 的 ADTS/ASC 是会话级配置，帧预算固定。
3. **抗弱网**：Opus 有 LBRR 带内 FEC（仅 SILK、修复前一帧）、DTX、PLC；AAC 标准没有，靠传输层。
4. **归因**：一半是算法（MDCT 重叠、帧长、包头设计），一半是设计目标（实时 vs 存储）和传输栈（UDP/RTP vs TCP/RTMP）。

---

## 8. 自测（5 题）

1. Opus 默认 20ms 帧的 26.5ms 是怎么算出来的？
2. AAC-LC 的 42.7ms 由哪几部分组成？
3. 为什么 Opus 能逐帧改码率，AAC 不能？
4. Opus 的带内 FEC 有什么局限？
5. 为什么说「AAC 抗弱网差」不完全公平？

> 答案在第 2、3、4 节。

---

## 参考

- RFC 6716（Opus）与勘误：https://www.rfc-editor.org/errata_search.php?rfc=6716&eid=4392
- Apple：AAC 编码背景（2048/1024、priming、decoder delay）：https://developer.apple.com/documentation/quicktime-file-format/background_aac_encoding
- Wikipedia：Opus 帧长与模式：https://en.wikipedia.org/wiki/Opus_(audio_format)
- Opus 带内 FEC 与 ULPFEC 的组合：https://datatracker.ietf.org/doc/draft-oancea-avtcore-opus-ulpfec/
