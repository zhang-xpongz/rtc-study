# GCC / TWCC / 弱网：详细学习资料

> 对应 D5。目标：能画控制环、能跟源码、能回答面试追问。
> 配套：[GCC 控制环模板](../source/GCC控制环.md)

---

## 0. 术语表（先读这里）

| 术语 | 英文 | 含义 |
|---|---|---|
| 带宽估计 | BWE, Bandwidth Estimation | 估计当前链路能承载多少码率 |
| GCC | Google Congestion Control | WebRTC 使用的拥塞控制算法 |
| TWCC | Transport-Wide Congestion Control | 基于传输层序列号的反馈机制 |
| REMB | Receiver Estimated Maximum Bitrate | 旧的接收端码率估计（RTCP 消息） |
| 反馈 | feedback | 接收端告诉发送端哪些包到了、什么时候到 |
| 单向延迟 | one-way delay | 包从发送到到达的延迟 |
| 排队延迟 | queue delay | 包在瓶颈队列里等待的时间 |
| RTT | Round-Trip Time | 往返时延 |
| 丢包率 | loss rate | 丢失包 / 总包 |
| 抖动 | jitter | 到达间隔的变化 |
| AIMD | Additive Increase Multiplicative Decrease | 加性增、乘性减 |
| 探测 | probe | 主动用更高码率发一小段，试探链路容量 |
| 整形 | pacing | 把包按时间均匀发出，避免突发 |
| 目标码率 | target bitrate | BWE 输出，给编码器/分配器的码率 |
| 分配 | allocation | 把总码率分给音频、视频、各轨道 |
| 降级偏好 | degradation preference | 带宽不足时优先保分辨率还是帧率 |
| ALR | Application Limited Region | 应用发送量低于目标码率，链路未被跑满 |
| overuse | 过度使用 | 排队延迟持续上升，说明发多了 |

---

## 1. 为什么需要拥塞控制

发送码率超过链路容量时：

```
发送 > 链路容量
  → 瓶颈队列增长
    → 排队延迟上升
      → 延迟变大、jitter 变大
        → 队列满后开始丢包
          → 卡顿、花屏、延迟累积
```

拥塞控制要同时做到：

1. **尽量跑满带宽**：不要保守到浪费容量。
2. **不把队列撑爆**：延迟不能持续增长。
3. **稳定**：码率不要剧烈震荡。
4. **公平**：多条流共享瓶颈时不过度抢占。

---

## 2. GCC 总体架构

GCC 是**发送端控制器**：

```
接收端：记录每个包的到达时间
  → TWCC 反馈
    → 发送端：还原发送/到达时间
      → 延迟控制器 + 丢包控制器
        → 目标码率
          → 编码器 / 分配器 / Pacer
```

两条控制腿：

| 控制器 | 观察量 | 反应速度 | 作用 |
|---|---|---|---|
| 延迟控制器 | 排队延迟趋势 | 快（在丢包前） | 提前发现拥塞 |
| 丢包控制器 | 丢包率 | 慢（丢包已发生） | 兜底、恢复 |

为什么不能只用丢包：

> 丢包是拥塞的**滞后信号**。等看到丢包，队列已经满了、延迟已经上去了。延迟趋势能在丢包前就发现「队列在变长」。

---

## 3. TWCC：反馈机制

### 3.1 它解决什么问题

RTP 的序列号是**每个 SSRC 独立**的。一个 PeerConnection 可能有多条流（音频、视频、重传），接收端很难用一个统一的序号告诉发送端「哪些包到了」。

TWCC 给每个发送的包加一个**传输层序列号**（transport-wide sequence number），跨 SSRC 统一编号。

### 3.2 接收端做什么

- 收到包，记录 `transport sequence number` 和到达时间。
- 周期性发 RTCP TWCC 反馈：
  - base sequence number
  - packet status（收到/丢失）
  - 每个包的到达时间增量

### 3.3 发送端做什么

`TransportFeedbackAdapter::ProcessTransportFeedback`：

1. 用传输层序号找回发送历史（`send_time`、`size`、` pacing_info`）。
2. 把「到达时间增量」还原成绝对接收时间。
3. 生成 `TransportPacketsFeedback`，里面每个 `PacketResult` 有：
   - `sent_packet.send_time`
   - `sent_packet.size`
   - `receive_time`（丢失则为无穷）
   - `data_in_flight`

### 3.4 为什么把计算放在发送端

| 原因 | 说明 |
|---|---|
| 接收端不知道发送时间 | 只有发送端有 `send_time`、pacing、probe 信息 |
| 可以跨流统一 | transport-wide 序号覆盖所有 SSRC |
| 发送端有全局上下文 | 编码器目标、pacer 队列、probe 结果都在发送端 |
| 反馈更紧凑 | 只传序号和到达时间增量，不传完整统计 |

> 这也是「send-side BWE」和旧「receive-side REMB」的核心区别。

---

## 4. 延迟控制器（Delay-Based BWE）

### 4.1 到达时间模型

对包 i：

```
send_time  = T_i
receive_time = t_i
one_way_delay = t_i - T_i
```

但收发两端时钟不同步，绝对单向延迟不可用。用**相对变化**：

```
delay_variation(i) = (t_i - t_0) - (T_i - T_0)
```

如果链路空闲，`delay_variation` 接近常数；如果队列在增长，`delay_variation` 会持续上升。

### 4.2 inter-arrival delta

`InterArrivalDelta` 把包按发送时间分组，计算：

- `send_delta`：两组包的发送时间差
- `recv_delta`：两组包的到达时间差
- `size_delta`：两组包的字节数差

如果 `recv_delta > send_delta`，说明包在链路里待得更久了，队列在增长。

### 4.3 trendline

`TrendlineEstimator` 对「累积延迟 vs 时间」做线性回归，得到斜率 `trend`。

- 斜率为正且大：延迟在持续上升 → overuse
- 斜率为负且大：延迟在下降 → underuse
- 接近 0：normal

为什么不看单次延迟：

> 单次延迟会被抖动、调度、重传干扰；trendline 看的是**趋势**，更稳。

### 4.4 overuse 判定

`TrendlineEstimator::Detect`：

```
modified_trend = min(num_deltas, kMinNumDeltas) * trend * threshold_gain
```

- `modified_trend > threshold` 且持续超过 `10ms`，且 `trend` 没有下降 → `kBwOverusing`
- `modified_trend < -threshold` → `kBwUnderusing`
- 否则 → `kBwNormal`

阈值不是固定的：

- 初始 `12.5ms`
- 自适应范围 `6–600ms`
- 大尖刺不参与自适应，避免被突发带偏

### 4.5 AIMD

`AimdRateControl` 三个状态：

| 状态 | 行为 |
|---|---|
| `kRcIncrease` | 增加码率 |
| `kRcHold` | 保持 |
| `kRcDecrease` | 下降 |

增加：

- 有链路容量估计：加性增加
- 没有容量估计：乘性增加，`1.08^seconds`（上限 1 秒），即约 8%/s

下降：

```
new = estimated_throughput * 0.85
if new > 5kbps: new -= 5kbps
```

然后进入 hold，等队列排空。

---

## 5. 丢包控制器（Loss-Based BWE）

`SendSideBandwidthEstimation` 维护：

- `fraction_loss`：丢包率
- `round_trip_time`：RTT
- `delay_based_limit_`：延迟控制器给的上限
- `receiver_limit_`：接收端估计（REMB）
- `min/max_bitrate_configured_`

### 5.1 三个区间

| 丢包率 | 行为 |
|---|---|
| `< 2%` | 增加：`min_bitrate_history × 1.08 + 1kbps` |
| `2% – 10%` | 保持 |
| `> 10%` | 下降：`target × (1 - 0.5 × loss)` |

### 5.2 RTT backoff

RTT 持续高于阈值时，按周期下降码率，避免「RTT 很大但还没丢包」时继续加压。

### 5.3 合并规则

```
target = min(
  loss_based_target,
  delay_based_limit,
  receiver_limit,      // 若启用
  max_configured
)
```

代码里：

- `UpdateDelayBasedEstimate()` 把延迟估计写成 `delay_based_limit_`
- `GetUpperLimit()` 取 `delay_based_limit_`、`receiver_limit_`、`max_bitrate_configured_` 的最小值
- `UpdateTargetBitrate()` 把 loss-based 结果 clamp 到上限

所以：

> **延迟控制器更像「上限」，丢包控制器负责主要增减。**

---

## 6. Probe、Pacing、分配

### 6.1 Probe

什么时候探测：

- 从 overuse 恢复
- ALR 结束（应用重新开始发满）
- 周期性 ALR 探测
- 初始启动
- 网络路由变化

做法：

- 发 probe cluster，用高于当前目标的速率短时间发送
- `ProbeBitrateEstimator` 测这组包的吞吐
- 如果成功，提升目标码率

### 6.2 Pacing

Pacer 把编码器输出的突发包按时间均匀发出：

- 避免瞬间排队
- 让 BWE 的延迟测量更准
- probe 时按 probe 速率发

### 6.3 分配

`BitrateAllocator` 把总目标码率分给音频和视频：

- 音频优先保障（通常几十 kbps）
- 视频拿剩余
- 视频内部再分给各轨道/层

带宽不足时，视频编码器会：

- 先降目标码率
- 再按 `degradation preference` 降分辨率或帧率
- simulcast/SVC 可能直接丢层

---

## 7. 弱网下的表现

| 现象 | 发送端反应 | 接收端/媒体反应 |
|---|---|---|
| 丢包 | 丢包控制器降码率；NACK/FEC 补偿 | 花屏、卡顿；PLC/隐藏 |
| 延迟上升 | 延迟控制器降码率 | jitter buffer 增大，延迟变高 |
| 带宽不足 | 目标码率下降；probe 试探 | 分辨率/帧率下降，切层 |
| 突发 | pacer 平滑 | 队列不炸，延迟更稳 |

音频优先：

- 音频码率低，通常优先保
- Opus 有 FEC/DTX/PLC
- 视频可以大幅降码率甚至暂停

---

## 8. 源码地图

| 文件 | 关键函数 | 看什么 |
|---|---|---|
| `modules/congestion_controller/rtp/transport_feedback_adapter.cc` | `ProcessTransportFeedback`（`:221`） | TWCC → `TransportPacketsFeedback` |
| `modules/congestion_controller/goog_cc/goog_cc_network_control.cc` | `OnTransportPacketsFeedback`（`:409`）、`MaybeTriggerOnNetworkChanged`（`:587`） | 总流程、合并、输出 |
| `modules/congestion_controller/goog_cc/delay_based_bwe.cc` | `IncomingPacketFeedbackVector`（`:97`）、`MaybeUpdateEstimate`（`:210`） | 延迟控制器 |
| `modules/congestion_controller/goog_cc/trendline_estimator.cc` | `Update`（`:247`）、`Detect`（`:267`） | 趋势与 overuse |
| `modules/remote_bitrate_estimator/aimd_rate_control.cc` | `Update`（`:149`）、`ChangeBitrate`（`:210`） | AIMD |
| `modules/congestion_controller/goog_cc/send_side_bandwidth_estimation.cc` | `UpdateDelayBasedEstimate`（`:300`）、`UpdateLossBasedEstimator`（`:321`）、`GetUpperLimit`（`:557`） | 丢包与合并 |
| `modules/congestion_controller/goog_cc/probe_controller.cc` | `RequestProbe`、`Process` | 探测 |
| `modules/pacing/pacing_controller.cc` | `ProcessPackets` | 发送节奏 |
| `call/bitrate_allocator.cc` | `AllocateBitrate` | 音视频分配 |

### 一条完整调用链

```
TWCC feedback (RTCP)
  → TransportFeedbackAdapter::ProcessTransportFeedback
    → GoogCcNetworkController::OnTransportPacketsFeedback
      → DelayBasedBwe::IncomingPacketFeedbackVector
        → TrendlineEstimator::Detect
        → AimdRateControl::Update
      → SendSideBandwidthEstimation::UpdateLossBasedEstimator
      → MaybeTriggerOnNetworkChanged
        → TargetTransferRate
          → BitrateAllocator
          → Pacer
```

---

## 9. E5 实验步骤

1. Chrome 打开 WebRTC demo，两个标签页互连。
2. 打开 `chrome://webrtc-internals`。
3. 找到：
   - `candidate-pair` 的 `availableOutgoingBitrate`
   - `outbound-rtp` 的 `targetBitrate`、`framesPerSecond`、`frameWidth`、`frameHeight`、`qualityLimitationReason`
   - `remote-inbound-rtp` 的 `fractionLost`、`roundTripTime`
4. 用 Network Link Conditioner 或手机热点限速。
5. 记录三组数据：正常、弱网、恢复。

### 观测表

| 阶段 | availableOutgoingBitrate | RTT | 丢包 | 分辨率 | FPS | 关键帧请求 |
|---|---|---|---|---|---|---|
| 正常 |  |  |  |  |  |  |
| 弱网 |  |  |  |  |  |  |
| 恢复 |  |  |  |  |  |  |

### 要回答

- 先降的是码率、分辨率还是帧率？
- `qualityLimitationReason` 是 `bandwidth` 还是 `cpu`？
- 恢复后码率回升是加性还是乘性？

---

## 10. 自检题

1. 为什么 TWCC 把计算放在发送端？
2. 延迟控制器为什么用 trendline，而不是单次延迟？
3. overuse 为什么要持续 10ms？
4. 2% 和 10% 两个丢包阈值分别做什么？
5. AIMD 的 0.85 是什么含义？
6. 为什么延迟估计是「上限」而不是直接目标？
7. probe 在什么时机触发？
8. ALR 是什么，为什么 ALR 期间不增加延迟估计？
9. Pacer 解决什么问题？
10. GCC 和 BBR 的区别是什么？

---

## 11. 面试 2 分钟表达

> WebRTC 用 GCC 做拥塞控制，核心是发送端根据 TWCC 反馈同时跑两个控制器。延迟控制器用 trendline 拟合排队延迟的趋势，发现 overuse 就用 AIMD 降码率；丢包控制器按 2%/10% 两个阈值做增减。两者合成目标码率，延迟估计通常作为上限。目标码率再交给 BitrateAllocator 分给音视频，Pacer 负责平滑发送。弱网时先降视频码率，再按降级偏好降分辨率或帧率，音频优先保障。

---

## 12. 一句话总结

> **TWCC 提供「每个包什么时候到」，trendline 提供「队列有没有变长」，AIMD 提供「怎么增减」，丢包控制器提供「兜底」，Pacer 和 BitrateAllocator 把目标码率变成实际发送。**
