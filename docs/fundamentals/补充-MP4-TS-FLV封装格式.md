# 补充：MP4 / MPEG-TS / FLV 封装格式详解

> 配套 D1「封装与时间戳」第 3 节。
> 一句话：**封装格式解决「怎么把音视频帧、时间戳、索引、元数据放进一个文件或流里」。编码决定画质，封装决定能不能拖进度条、能不能直播、丢包后能不能恢复。**

---

## 0. 术语表（先读这里）

### 0.1 通用概念

| 术语 | 英文 | 含义 |
|---|---|---|
| 容器 | container | 存放音视频轨、时间戳、元数据的格式，如 MP4/TS/FLV |
| 封装 / 解封装 | mux / demux | 把编码帧写进容器 / 从容器里读出来 |
| 轨道 | track | 一条独立的媒体流（视频轨、音频轨、字幕轨） |
| 样本 | sample | 容器里的一帧/一个访问单元（视频一帧、音频一帧） |
| 时间基 | timescale | 时间戳的单位，如 1/90000、1/1000 |
| 显示时间戳 | PTS | 这一帧什么时候显示 |
| 解码时间戳 | DTS | 这一帧什么时候解码（有 B 帧时与 PTS 不同） |
| 同步样本 | sync sample | 可独立解码的关键帧（IDR） |
| 快速启动 | fast start | MP4 的 moov 放在文件头，边下边播 |

### 0.2 MP4 相关

| 术语 | 含义 |
|---|---|
| box | MP4 的基本结构单元，`size + type + data` |
| ftyp | 文件类型与兼容品牌 |
| moov | 元数据容器（时长、轨道、索引） |
| mdat | 媒体数据 |
| stbl | sample table，样本索引（stts/stsc/stsz/stco/stss/ctts） |
| stss | 同步样本表（哪些帧是关键帧） |
| ctts | 合成时间偏移（PTS − DTS） |
| fMP4 | 分片 MP4，moof + mdat，用于 DASH/HLS/CMAF |
| moof | 分片 MP4 的元数据片段 |
| avcC | H.264 的 SPS/PPS 配置 |

### 0.3 TS 相关

| 术语 | 含义 |
|---|---|
| TS | MPEG-2 Transport Stream，188 字节定长包 |
| PID | 包标识，区分音视频/PSI |
| PAT | Program Association Table，PID 0，指向 PMT |
| PMT | Program Map Table，列出各流的 PID 与类型 |
| PCR | Program Clock Reference，27 MHz 系统时钟参考 |
| PES | Packetized Elementary Stream，带 PTS/DTS 的媒体包 |
| 连续计数器 | continuity counter，检测丢包/重复 |

### 0.4 FLV 相关

| 术语 | 含义 |
|---|---|
| tag | FLV 的基本单元（audio/video/script） |
| AVCC | H.264 的长度前缀格式（与 MP4 相同） |
| Annex-B | 用 00 00 00 01 起始码的格式（TS 用） |
| CTS | Composition Time Stamp，FLV 里的 PTS−DTS 偏移 |
| onMetaData | 脚本 tag，存时长、分辨率、关键帧索引 |

---

## 1. 封装格式到底解决什么

一个视频文件/流里有四类信息：

```
① 编码后的帧数据（H.264/AAC…）
② 时间信息（PTS/DTS、时长、帧率）
③ 索引（哪些是关键帧、每帧在文件的哪个位置）
④ 元数据（分辨率、编码参数、DRM、字幕）
```

不同封装就是把这四类信息**用不同结构组织**，于是有了不同的取舍：

| 目标 | 倾向的封装 |
|---|---|
| 点播、精确 seek | MP4 |
| 直播、抗丢包、广播 | TS |
| 低延迟、简单、RTMP | FLV |
| 自适应码率、分片 | fMP4 / CMAF |

---

## 2. MP4（ISO Base Media File Format）

### 2.1 结构

MP4 是「box 套 box」的树形结构：

```
ftyp                    文件类型
moov                    元数据
 ├─ mvhd                整体时长、时间基
 ├─ trak                轨道
 │   ├─ tkhd            轨道头（宽高、时长）
 │   └─ mdia
 │       ├─ mdhd        轨道时间基
 │       ├─ hdlr        轨道类型（vide/soun）
 │       └─ minf
 │           └─ stbl    样本表
 │               ├─ stsd   编码配置（avcC/esds）
 │               ├─ stts   每帧时长
 │               ├─ ctts   PTS−DTS 偏移
 │               ├─ stsc   样本到 chunk 的映射
 │               ├─ stsz   每帧大小
 │               ├─ stco   每帧偏移
 │               └─ stss   关键帧列表
 └─ mvex                fMP4 用，分片默认参数
mdat                    媒体数据
```

### 2.2 关键机制

**样本表（stbl）** 是 MP4 的核心：

- `stts`：每个 sample 的时长（用来算 DTS）
- `ctts`：PTS − DTS（有 B 帧时需要）
- `stss`：哪些 sample 是关键帧（seek 靠它）
- `stco/stsz`：每帧在文件里的偏移和大小

有了这张表，播放器可以**不做全文件扫描**就跳到任意关键帧。

**moov 位置**：

- moov 在文件尾：编码器默认行为，边下边播必须先下完整个文件。
- moov 在前（fast start）：需要二次写文件（`ffmpeg -movflags +faststart`），适合网页点播。

**fMP4（分片 MP4）**：

```
moov（初始化段，含 mvex）
moof + mdat（媒体分片 1）
moof + mdat（媒体分片 2）
...
sidx（分片索引，可选）
```

- 每个分片独立可解码，适合 DASH/HLS/CMAF。
- 直播用 fMP4 可以边生成边发，不需要回写 moov。
- CMAF 让同一份 fMP4 同时服务 HLS 和 DASH。

### 2.3 优缺点

| 优点 | 缺点 |
|---|---|
| 索引精确，seek 快 | moov 不前置时不能边下边播 |
| 元数据丰富（DRM、字幕、章节） | 文件损坏后难恢复 |
| 点播生态最成熟 | 直播需要 fMP4/CMAF |
| 支持多轨、封面、编辑列表 | 结构复杂，解析实现成本高 |

---

## 3. MPEG-TS（Transport Stream）

### 3.1 结构

TS 是**定长 188 字节**的包流：

```
┌──────┬──────┬──────────────┬─────────────────┐
│ 4 字节头 │ ... │ 184 字节负载（或适配字段）│
└──────┴──────┴──────────────┴─────────────────┘
```

包头：

| 字段 | 作用 |
|---|---|
| sync byte (0x47) | 同步字，丢包后可重新对齐 |
| PID (13 bit) | 区分流：0=PAT，其他=PMT/音视频 |
| PUSI | 负载单元起始标志 |
| adaptation field | 填充、PCR、随机访问标志 |
| continuity counter | 检测丢包/重复 |

### 3.2 PSI：PAT / PMT

- **PAT**（PID 0）：告诉播放器「节目 1 的 PMT 在哪个 PID」。
- **PMT**：列出该节目的视频 PID、音频 PID、PCR PID、编码类型。
- 每个 HLS 分片都带 PAT/PMT，保证独立可播。

### 3.3 PES 与时间戳

媒体数据打成 PES 包：

```
PES start code (00 00 01) + stream id + 包长度 + PTS/DTS + 负载
```

- PTS/DTS 是 **33 位、90 kHz**，约 26.5 小时回绕。
- **PCR** 是 27 MHz 系统时钟（33 位 base + 9 位 extension），用于同步解码器时钟。

### 3.4 为什么直播/广播用 TS

| 特性 | 好处 |
|---|---|
| 定长 188 字节 | 丢包后可快速重新同步 |
| 独立分片可播 | 每个 HLS 分片自带 PAT/PMT |
| 抗错设计 | 适配字段、连续计数器、FEC（DVB） |
| 多节目复用 | 一个流里多套节目 |
| 硬件支持 | 机顶盒/电视/广播设备原生支持 |

代价：

- 头部开销：4/188 ≈ **2.1%**，加上 PSI/PES 头，实际 5–10%。
- 没有全局索引，seek 要自己找关键帧。
- 元数据能力弱。

### 3.5 HLS 与 TS

HLS 的经典做法是 **TS 分片 + m3u8 索引**：

```
#EXTM3U
#EXT-X-TARGETDURATION:4
#EXTINF:4.0,
seg0.ts
#EXTINF:4.0,
seg1.ts
```

每个 `seg.ts` 以关键帧开头，包含 PAT/PMT。现在的 HLS 也支持 fMP4（CMAF）。

---

## 4. FLV（Flash Video）

### 4.1 结构

FLV 是「头 + 一串 tag」：

```
FLV header (9 字节)
  Signature "FLV" (3) + Version (1) + Flags (1) + DataOffset (4)
Tag 1
Tag 2
...
```

每个 tag：

```
TagType (1)         8=audio, 9=video, 18=script
DataSize (3)
Timestamp (3)       毫秒低 24 位
TimestampExt (1)    毫秒高 8 位
StreamID (3)        通常 0
Data (DataSize 字节)
PreviousTagSize (4)
```

### 4.2 视频 tag（H.264）

```
FrameType (4 bit)      1=关键帧, 2=非关键帧
CodecID (4 bit)        7=AVC
AVCPacketType (1)      0=sequence header(avcC), 1=NALU, 2=end
CompositionTime (3)    PTS−DTS，有符号 24 位
Data                   AVCC 格式的 NALU
```

### 4.3 音频 tag（AAC）

```
SoundFormat (4 bit)    10=AAC
SoundRate (2 bit)
SoundSize (1 bit)
SoundType (1 bit)
AACPacketType (1)      0=AudioSpecificConfig, 1=raw
Data
```

### 4.4 脚本 tag：onMetaData

存时长、宽高、帧率、码率，以及**关键帧索引**（每帧的时间戳和文件偏移）。有了它，FLV 也能做 seek。

### 4.5 优缺点

| 优点 | 缺点 |
|---|---|
| 结构简单，解析快 | 时间戳 32 位毫秒，约 49.7 天回绕 |
| 低延迟，适合 RTMP/HTTP-FLV | 无标准全局索引（靠 onMetaData） |
| tag 边界清晰，易重新同步 | 编码支持有限（H.264/AAC 为主） |
| 直播生态成熟 | Flash 已死，浏览器不支持原生 FLV |

---

## 5. 三者对比

| 维度 | MP4 | MPEG-TS | FLV |
|---|---|---|---|
| 结构 | box 树 | 188 字节定长包 | tag 流 |
| 典型用途 | 点播、下载 | 直播、广播、HLS | RTMP、HTTP-FLV |
| 随机访问 | stss 索引，精确 | 无全局索引，找关键帧 | onMetaData 关键帧索引 |
| 直播友好 | 差（需 fMP4） | 好 | 好 |
| 抗丢包 | 差 | 好（sync byte/连续计数） | 中（tag 边界可重同步） |
| 头部开销 | 小（样本表几 KB） | 约 2.1% + PSI/PES | 每 tag 15 字节 |
| 时间戳 | timescale，灵活 | 33 位 90kHz + PCR 27MHz | 32 位毫秒 |
| B 帧支持 | ctts/elst | PTS/DTS | CTS 偏移 |
| 元数据 | 丰富（DRM/字幕/章节） | 弱（描述符） | 中（onMetaData） |
| 浏览器支持 | 原生 `<video>` | 需 HLS.js | 需 flv.js |
| 延迟 | VOD 无意义；fMP4 可低 | 切片决定（HLS 2–6s） | 低（HTTP-FLV 1–3s） |

### 5.1 头部开销的粗略计算

以 1 Mbps 视频、30fps、每帧一个 video tag 为例：

- **FLV**：每帧 tag 头 11 字节 + PreviousTagSize 4 字节 = 15 字节 × 30 = 450 B/s，约 0.36% 开销。
- **TS**：4/188 ≈ 2.1%，加 PES 头约 3–5%。
- **MP4**：样本表几 KB，一次性开销，可忽略。

低码率场景（如 100 kbps 音频）下，FLV 的固定开销占比会明显上升。

---

## 6. 工程上的选择

| 场景 | 推荐 | 原因 |
|---|---|---|
| 网页点播 | MP4（faststart） | 浏览器原生支持、seek 精确 |
| 短视频 | MP4 / fMP4 | 兼容性最好 |
| 直播（HLS） | TS 或 fMP4 | 分片、CDN 友好 |
| 低延迟直播 | HTTP-FLV 或 LL-HLS/LL-DASH | FLV 延迟低；LL 方案用 fMP4 |
| 录制 | MP4（fMP4 更抗崩溃）或 TS | MP4 好剪辑；TS 抗损坏 |
| 监控 | TS / 私有封装 | 抗丢包、长时间录制 |
| WebRTC | RTP（不是文件封装） | 实时传输 |

### 6.1 录制场景的坑

- **MP4 录到一半崩溃**：moov 没写，文件打不开。解决：用 fMP4 分片，或定期 flush。
- **TS 录制**：丢包可跳过，但文件会越来越大、无索引，回放要扫描。
- **FLV 录制**：简单，但时间戳回绕、seek 依赖 onMetaData。

### 6.2 H.264 的三种「打包形态」

同一个 H.264 码流，在不同封装里长这样：

| 封装 | 形态 | 说明 |
|---|---|---|
| MP4 | AVCC（长度前缀） | `[4 字节长度][NALU]`，SPS/PPS 在 avcC |
| TS | Annex-B（起始码） | `00 00 00 01 [NALU]`，SPS/PPS 可重复插入 |
| FLV | AVCC | 与 MP4 相同，序列头单独存 avcC |
| RTP | RFC 6184 | FU-A/STAP-A 分片 |

这解释了为什么「同一个 H.264 文件换封装要转码/转封装」——转封装就是换这层边界格式。

---

## 7. 面试问答（8 题）

**Q1：MP4 的 moov 是什么？为什么需要 faststart？**
A：moov 是元数据容器，含轨道信息和样本表（索引）。默认编码器把 moov 写在文件尾，浏览器必须下完整个文件才能开始播；`-movflags +faststart` 会把 moov 移到文件头，实现边下边播。

**Q2：TS 为什么用 188 字节定长包？**
A：为了抗错和复用。定长包 + 同步字 0x47，丢包后能快速重新对齐；固定长度便于硬件解复用和广播传输。

**Q3：TS 的 PAT 和 PMT 分别做什么？**
A：PAT（PID 0）告诉播放器每个节目的 PMT 在哪个 PID；PMT 列出该节目音视频流的 PID、类型和 PCR PID。

**Q4：FLV 的时间戳有什么坑？**
A：32 位毫秒，约 49.7 天回绕；长时间推流要做回绕处理，否则播放器会跳变。

**Q5：为什么 HLS 用 TS 而不是 MP4？**
A：TS 分片独立可播、抗丢包、CDN 缓存友好、硬件支持好；MP4 需要 fMP4 才能做分片直播。

**Q6：MP4 里 B 帧的 PTS 和 DTS 怎么存？**
A：stts 存 DTS 步长，ctts 存 PTS−DTS 偏移；编辑列表 elst 处理起始偏移。

**Q7：FLV 的 CTS 是什么？**
A：Composition Time，等于 PTS−DTS，有符号 24 位，用于有 B 帧时的显示顺序调整。

**Q8：转封装和转码的区别？**
A：转封装只换容器（MP4→TS），不重新编码，速度快、无损；转码要解码再编码，改变码率/分辨率/编码格式，慢且有损。

---

## 8. 自测（5 题）

1. MP4 的 stss 表存什么？没有它 seek 会怎样？
2. TS 的 PCR 和 PTS 分别是什么时钟？为什么需要两个？
3. FLV 一个 video tag 的头部有哪些字段？
4. 为什么说「MP4 不适合直播」？fMP4 解决了什么？
5. 同一个 H.264 码流，MP4 和 TS 里的 NAL 边界格式有什么不同？

> 答案在第 2、3、4、6 节。
