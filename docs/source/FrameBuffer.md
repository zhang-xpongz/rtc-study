# 源码卡片 · FrameBuffer

> 文件：`api/video/frame_buffer.h` ｜ 日期：2026-11-17

## 这个模块解决什么问题

FrameBuffer 是一个**线程不安全**的帧缓冲数据结构：按 frame ID 排序、按时间戳组成 temporal unit，判定哪些 unit 可解码/连续，并记录解码历史；它不负责调度和计时。

## 关键类 / 函数

| 类/函数 | 文件:行 | 作用 | 调用方 |
|---|---|---|---|
| `FrameBuffer` | `api/video/frame_buffer.h:33` | 存储/排序 `EncodedFrame`，判定连续性/可解码性 | `VideoStreamBufferController` |
| 构造函数 | `api/video/frame_buffer.cc:71` | `max_size` 帧上限、`max_decode_history` 历史窗口 | controller |
| `InsertFrame` | `api/video/frame_buffer.cc:76` | 插入已编码帧 | `VideoStreamBufferController::InsertFrame` |
| `ExtractNextDecodableTemporalUnit` | `:128` | 取出下一个可解码 unit（可能多帧） | `FrameReadyForDecode` |
| `DropNextDecodableTemporalUnit` | `:145` | 丢弃该 unit | `Extract...`（取完自动丢）、`MaybeScheduleFrameForRelease`（无法调度时丢） |
| `LastContinuousFrameId` | `:159` | 连续**帧**的最大 ID | `VideoReceiveStream2::OnCompleteFrame` |
| `LastContinuousTemporalUnitFrameId` | `:163` | 连续 **unit** 的最后一帧 ID | 内部字段（`FindNext...` 扫描边界） |
| `DecodableTemporalUnitsInfo` | `:167` | 可解码范围（next/last RTP timestamp） | `MaybeScheduleFrameForRelease`、`FrameReadyForDecode` |
| `NewContinuousTemporalUnits` | `:172` | 本次插入新增的连续 unit 时间戳 | controller 的 `InsertFrame` |
| `IsContinuous` | `:187` | 单帧参考是否都可用 | `PropagateContinuity` |
| `PropagateContinuity` | `:205` | 沿 ID 向后传播连续标记 | `InsertFrame` |
| `FindNextAndLastDecodableTemporalUnit` | `:225` | 找下一个可解码 unit、更新可解码范围 | `InsertFrame`、`Extract...`、`Drop...` |
| `Clear` | `:285` | 清空帧、历史、连续性状态 | `InsertFrame`（关键帧重置） |
| `DecodedFramesHistory` | `modules/video_coding/utility/decoded_frames_history.h` | 记录已解码帧 ID/时间戳 | `IsContinuous`、`FindNext...` |

## 输入 / 输出

- **输入**：`InsertFrame(std::unique_ptr<EncodedFrame>)`
- **输出帧**：`ExtractNextDecodableTemporalUnit()` 返回 `absl::InlinedVector<std::unique_ptr<EncodedFrame>, 4>`
- **状态**：`LastContinuousFrameId()`、`LastContinuousTemporalUnitFrameId()`、`DecodableTemporalUnitsInfo()`、`NewContinuousTemporalUnits()`
- **计数**：`CurrentSize()`、`GetTotalNumberOfContinuousTemporalUnits()`、`GetTotalNumberOfDroppedFrames()`

关键数据结构：

- `FrameInfo { std::unique_ptr<EncodedFrame> encoded_frame; bool continuous = false; }`
- `FrameMap = std::map<int64_t, FrameInfo>`：按 frame ID 排序
- `TemporalUnit { FrameIterator first_frame; FrameIterator last_frame; }`
- `DecodabilityInfo { uint32_t next_rtp_timestamp; uint32_t last_rtp_timestamp; }`
- `DecodedFramesHistory`：已解码帧的历史窗口

## 容量与参数

```cpp
// api/video/frame_buffer.h:43
FrameBuffer(int max_size,
            int max_decode_history,
            const FieldTrialsView& field_trials);
```

controller 创建时传的默认值（`video/video_stream_buffer_controller.cc:46-48`）：

| 参数 | 值 | 含义 |
|---|---|---|
| `kMaxFramesBuffered` | **800** | 最多缓存的帧数 |
| `kMaxFramesHistory` | **8192**（`1 << 13`） | 解码历史窗口 |

缓冲满时：

- 非关键帧 → 丢弃
- 关键帧 → `Clear()` 后插入

## Frame ID 从哪来

Frame ID 由接收端参考帧查找器赋值，`references[]` 里存的也是 Frame ID，不是 RTP 时间戳：

| codec | Frame ID 来源 | 文件 |
|---|---|---|
| H.264 / 默认 | 最后一个 RTP 包的 seq，展开成 64 位 | `rtp_seq_num_only_ref_finder.cc:122` |
| VP8 | `pictureId` 展开 | `rtp_vp8_ref_finder.cc:42` |
| VP9 | `picture_id` 展开 | `rtp_vp9_ref_finder.cc:44` |
| AV1 / DD | DependencyDescriptor 的 `frame_id` | `rtp_generic_ref_finder.cc:46` |

codec 切换时会加 `picture_id_offset_` 防止新旧 ID 重叠（`rtp_frame_reference_finder.cc:211`）。

## 调用链

```
插入：
VideoReceiveStream2::OnCompleteFrame (network)
  → VideoStreamBufferController::InsertFrame
    → FrameBuffer::InsertFrame
      → 返回 LastContinuousFrameId

取帧：
FrameDecodeScheduler 到点
  → VideoStreamBufferController::FrameReadyForDecode   (.cc:287)
    → buffer_->DecodableTemporalUnitsInfo()            // 先校验
    → buffer_->ExtractNextDecodableTemporalUnit()      (:128)
      → VideoStreamBufferController::OnFrameReady      (.cc:208)
        → receiver_->OnEncodedFrame → VideoStreamDecoder

丢弃：
VideoStreamBufferController::MaybeScheduleFrameForRelease (.cc:380)
  → 当前 unit 无法调度
    → buffer_->DropNextDecodableTemporalUnit()          (:145)
```

### InsertFrame 内部步骤（`api/video/frame_buffer.cc:76`）

| 步骤 | 行 | 行为 |
|---|---|---|
| 1. 清空本次新增 | `:77` | `new_continuous_temporal_units_.clear()`，每次插入重置 |
| 2. 校验参考 | `:78` | `ValidReferences`：参考必须更早、不能重复；失败丢帧 |
| 3. 旧帧判断 | `:84` | ID ≤ 最后解码 ID：普通旧帧丢弃；**关键帧且时间戳更新则 `Clear()`** |
| 4. 缓冲满 | `:98` | 关键帧 `Clear()` 后插入；非关键帧丢弃 |
| 5. 按 ID 插入 | `:111` | `map::emplace`，重复 ID 失败 |
| 6. 传播连续性 | `:122` | `PropagateContinuity`：标记连续帧、更新 `last_continuous_*` |
| 7. 重算可解码单元 | `:123` | `FindNextAndLastDecodableTemporalUnit`：按时间戳分组，找第一个可解码 unit |
| 8. 返回 | `:124` | `true`=已插入；`false`=丢弃 |

分支汇总：

| 场景 | 结果 |
|---|---|
| 参考非法（前向/重复） | 丢弃，`false` |
| 重复 frame ID | 丢弃，`false` |
| 旧帧（ID ≤ 最后解码） | 丢弃，`false` |
| 关键帧 ID 旧但时间戳新 | `Clear()` + 插入，`true` |
| 缓冲满 + 非关键帧 | 丢弃，`false` |
| 缓冲满 + 关键帧 | `Clear()` + 插入，`true` |
| 正常帧 | 插入 + 更新连续性，`true` |

两个容易混的概念：

- **连续（continuous）**：单帧的所有参考要么已解码，要么在缓冲里且连续。
- **可解码 unit（decodable）**：整个 temporal unit 的参考都能满足（已解码或 unit 内）。

两个 `last_continuous_*`：

| 字段 | 含义 |
|---|---|
| `last_continuous_frame_id_` | 连续**帧**的最大 ID |
| `last_continuous_temporal_unit_frame_id_` | 连续 **unit** 的最后一帧 ID，`FindNext...` 用它做扫描边界 |

> 关键帧能清空满缓冲，是因为它不参考任何帧，是新的依赖根；旧帧对新流没有价值，清掉才能避免死锁。

## 关键日志

FrameBuffer 没有常规 `RTC_LOG`，只有 4 条 `RTC_DLOG(LS_WARNING)`，**仅在 Debug 构建且触发异常路径时输出**：

| 日志 | 位置 | 触发条件 |
|---|---|---|
| `Frame X has invalid references, dropping frame.` | `frame_buffer.cc:79` | 参考关系非法（重复引用/前向引用），丢帧 |
| `Keyframe X has newer timestamp but older picture id, clearing buffer.` | `:88` | 关键帧时间戳更新但 picture id 更旧，清空缓冲 |
| `Keyframe X inserted into full buffer, clearing buffer.` | `:100` | 缓冲满时收到关键帧，清空重来 |
| `Frame X inserted, buffer is now full.` | `:118` | 缓冲达到上限 |

正常路径没有日志，排查主要靠断点和统计：

- `FrameBuffer::InsertFrame`（`:76`）
- `ExtractNextDecodableTemporalUnit`（`:128`）
- `LastContinuousFrameId`（`:159`）
- `getStats` 里的 jitter buffer 指标（`jitterBufferDelay`、`jitterBufferEmittedCount`）

## 线程与生命周期

- **跑在哪个线程**：network 线程。`Call::CreateVideoReceiveStream` 在 network 线程（`call/call.cc:925`），创建 `VideoReceiveStream2` 时把 `call->network_thread()` 传给 controller（`video/video_receive_stream2.cc:199-200`）；`OnCompleteFrame` 也在 network 线程（`:909`）。成员名 `worker_sequence_checker_` 是历史命名，不代表 worker 线程。
- **谁创建**：`Call::CreateVideoReceiveStream` → `new VideoReceiveStream2` → 构造函数 → `CreateBuffer()` 辅助函数创建 `VideoStreamBufferController` → controller 构造函数中 `std::make_unique<FrameBuffer>`（`video_stream_buffer_controller.cc:109`）。
- **谁销毁**：`Call::DestroyVideoReceiveStream`（`call/call.cc:954`）→ `UnregisterFromTransport()` → `video_receive_streams_.erase()` → `delete`；FrameBuffer 随 controller 一起销毁。
- **重建**：`Clear()` 会 `make_unique<FrameBuffer>` 重新开始（`video_stream_buffer_controller.cc:148`）。

## 复杂度与测试

- 插入：`map::emplace`，`O(log n)`
- 连续性传播 / 找可解码 unit：从插入点向后扫描，`O(n)`
- 单次最多处理 `n ≤ 800` 帧，实际开销可控
- 已解码查询：`DecodedFramesHistory::WasDecoded`，`O(1)`

### 单测解析（测试即规格）

文件：`api/video/frame_buffer_unittest.cc`，suite 名 `FrameBuffer3Test`。

```bash
autoninja -C out/Debug frame_buffer_unittest
./out/Debug/frame_buffer_unittest --gtest_filter='FrameBuffer3Test.*'
```

**测试脚手架**

```cpp
test::FakeFrameBuilder().Time(10).Id(1).Refs({...}).AsLast().Build()
```

| 方法 | 含义 |
|---|---|
| `Time(ts)` | RTP 时间戳，**相同 Time = 同一个 temporal unit** |
| `Id(id)` | Frame ID |
| `Refs({...})` | 参考帧 ID；不写 = 关键帧 |
| `AsLast()` | `is_last_spatial_layer = true`，**关闭当前 unit** |
| `Build()` | 生成 `FakeEncodedFrame` |

> 不加 `AsLast()`，unit 就不完整，即使帧已经连续——这模拟 SVC 的基层/增强层。

**核心用例**

| 用例 | 场景 | 保护的不变量 |
|---|---|---|
| `RejectInvalidRefs` | Id0 refs{0}→false；Id1→true；Id2 refs{1,1}→false | 参考必须更早、不能重复；非法帧不改状态 |
| `LastContinuousUpdatesOnInsertedFrames` | Id1 Time10 非 last；Id2 Time10 refs{1} AsLast | 帧连续 ≠ unit 连续；`AsLast` 才关闭 unit |
| `LastContinuousFrameReordering` | Id3 refs{2} 先到不连续；Id2 到达后 1→3 | 乱序补齐后向前传播 |
| `LastContinuousTemporalUnitReordering` | 缺 Id2 卡住 unit20；补 Id2 后完成 unit10/20 | 跨 unit 级联解锁 |
| `NextDecodable` | 初始 nullopt；Id1 AsLast → next=10 | 关键帧单独可解码 |
| `AdvanceNextDecodableOnExtraction` | 1/2/3(ref2)，逐次 Extract | 提取标记 decoded，解锁依赖帧 |
| `AdvanceLastDecodableOnExtraction` | 2、3 都 ref{1}；提取 1 后 last 10→30 | `last_rtp_timestamp` 是最后可解码 unit，可跳变 |
| `KeyframeClearsFullBuffer` | max=5 满；Id6 非关键帧 false；Id7 关键帧 Clear | 满缓冲靠关键帧重置 |
| `OldFramesAreIgnored` | 提取 1/2 后重插 false；Id3 ref{1} true | 旧帧拒绝，新帧可引用已解码帧 |
| `ReturnFullTemporalUnitKSVC` | Time10 三帧，Extract 返回 [1,2,3] | 整组提取、按 ID 顺序 |

**其余用例**

| 用例 | 保护点 |
|---|---|
| `LastContinuousTemporalUnit` | 两帧同 Time，第二帧 AsLast 才关闭 unit |
| `FrameUpdatesNextDecodable` | 先插 Id2、后插 Id1，next 回退到 10 |
| `DropNextDecodableTemporalUnit` | 提取 1 后 Drop unit2，3 仍可解 |
| `InterleavedStream` | 交错依赖按序提取；旧帧 6 被拒 |
| `LegacyFrameIdJumpBehavior` | Id3 先到、Id2 旧帧、Id1 关键帧：旧非关键帧丢弃，关键帧时间戳新则清空 |
| `TotalNumberOfContinuousTemporalUnits` | 补齐 Id3 后计数 1→3，按 unit 累加 |
| `TotalNumberOfDroppedFrames` | Drop unit20 两帧，只统计 Drop 的帧 |
| `NewContinuousTemporalUnits` | 只报告本次新完成 unit 的时间戳 |
| `...WithReordering` | 补 Id2 后一次报 20/30/40，级联按时间顺序 |
| `...ReportsOlderFrames` | 后完成的 unit 先报，旧 unit 补报 |
| `...WithWraparound` | 0xFFFFFFF0 → 2000，回绕比较正确 |

**测试 → 代码分支**

| 测试 | 覆盖代码 |
|---|---|
| `RejectInvalidRefs` | `ValidReferences`（`:31`）、`InsertFrame:78` |
| `LastContinuous*` | `PropagateContinuity`（`:205`）、`IsContinuous`（`:187`） |
| `NextDecodable*` | `FindNextAndLastDecodableTemporalUnit`（`:225`）、`Extract`（`:128`） |
| `KeyframeClearsFullBuffer` | `InsertFrame:98-102` |
| `OldFramesAreIgnored` | `InsertFrame:84-95` |
| `ReturnFullTemporalUnitKSVC` | `Extract` 的 unit 循环（`:128-143`） |
| `TotalNumberOf*` | `Drop`（`:145`）、计数器（`:176-181`） |
| `NewContinuous*` | `PropagateContinuity:213-218` |

**为什么可信**

1. 纯数据结构：无时钟、网络、调度器，输入只有 ID/时间戳/参考，输出确定。
2. 直接断言内部状态：`LastContinuousFrameId`、`DecodableTemporalUnitsInfo`、计数都是契约。
3. 覆盖边界：乱序、缺帧、满缓冲、旧帧、ID 回退、时间戳回绕。
4. 测试即规格：读不懂代码时先读测试，能反推字段语义。

## 面试可讲点

1. **为什么按 frame ID 排序，而不是 RTP 时间戳**：frame ID 单调递增、反映解码顺序，且每帧只能引用更早的帧；时间戳可能重复（SVC 多层）或乱序。H.264 没有 picture ID，Frame ID 直接用 RTP 序号展开。**排序用 ID，分组用时间戳**。
2. **连续 vs 可解码**：`continuous` 只要求参考链上的帧可用（已解码或在缓冲且连续）；`decodable unit` 要求整个 unit 的参考都已解码或在 unit 内。两者用 `last_continuous_frame_id_` / `last_continuous_temporal_unit_frame_id_` 区分。
3. **数据与控制分离**：FrameBuffer 是纯数据结构（`api/`，无时钟/调度，线程不安全，易单测）；VideoStreamBufferController 负责收帧、计时、调度、超时。收益是职责单一、可测试；代价是两层要约定线程（都在 network 线程）和 API 契约。

## 没搞懂的地方

1. `DecodedFramesHistory` 的具体淘汰策略和容量边界？
2. SVC 多层时 `is_last_spatial_layer` 由哪一层设置？H.264 下是否恒为 marker？
3. codec 切换时 `picture_id_offset_` 如何影响 FrameBuffer 的连续性判定？
