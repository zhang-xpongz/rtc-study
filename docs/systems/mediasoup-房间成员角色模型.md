# mediasoup 的房间 / 成员 / 角色模型

> 实验环境：`/Volumes/SSD/Code/mediasoup-lab`
> 启动：`node server.js` → http://127.0.0.1:3030
> 目的：对照 SRS 的「流模型」，看清楚「房间模型」是怎么组织的。

---

## 1. 结论先行

| 维度 | SRS | mediasoup |
|---|---|---|
| 组织单位 | 流（vhost/app/stream） | 房间（Router） |
| 成员 | publisher / subscriber | Peer（每个连接） |
| 角色 | 无业务角色 | 应用层自定义（host/speaker/audience） |
| 转发 | 按流分发 | 按 Producer/Consumer 订阅 |
| 房间 | 无 | 显式创建、销毁 |
| 权限 | 基本无 | 信令层强制（能否 produce） |

mediasoup 本身**只提供媒体能力**（Router、Transport、Producer、Consumer），房间、成员、角色都是**信令层**自己实现的。我们的实验里，这些都在 `server.js` 里。

---

## 2. mediasoup 的对象模型

```
Worker（进程级，一个 Node 进程一个）
 └── Router（房间级，一个房间一个）
      └── WebRtcTransport（每个成员的两条：send / recv）
           ├── Producer（我发布的音视频轨）
           └── Consumer（我订阅的别人的轨）
```

关键点：

- **Worker**：底层 C++ 媒体引擎，负责 RTP 转发、拥塞控制、Simulcast。
- **Router**：一个房间。它不关心“谁是谁”，只负责把 Producer 的流路由给 Consumer。
- **Transport**：一个成员的传输通道。**发送和接收是两条独立 Transport**。
- **Producer / Consumer**：一根轨（音频或视频）的发布与订阅关系。

mediasoup **没有** Room、Member、Role 这些类——这些是我们在信令层加的概念。

---

## 3. 我们的实现映射

`server.js` 里的数据结构：

```js
rooms: Map<roomId, {
  router,                    // mediasoup Router = 房间
  peers: Map<peerId, {
    id, name, role,          // 成员 + 角色
    ws,                      // 信令连接
    transports: Map,         // send/recv 两条
    producers: Map,          // 我发布的
    consumers: Map,          // 我订阅的
  }>,
  producers: Map<producerId, {producer, peerId}>,
}>
```

角色定义：

```js
const ROLE = { host: 'host', speaker: 'speaker', audience: 'audience' };
const canPublish = (role) => role === 'host' || role === 'speaker';
```

| 角色 | 能发布 | 能订阅 | 能改别人角色 |
|---|---|---|---|
| host | ✅ | ✅ | ✅ |
| speaker | ✅ | ✅ | ❌ |
| audience | ❌ | ✅ | ❌ |

---

## 4. 加入房间的信令流程

```
Client                          Server
  │  join {room,name,role}        │
  │ ────────────────────────────► │ 创建/复用 Router，登记 Peer
  │ ◄──────────────────────────── │ joined {peerId, rtpCapabilities, members}
  │                               │
  │  createTransport              │
  │ ────────────────────────────► │ router.createWebRtcTransport()
  │ ◄──────────────────────────── │ {id, iceParameters, iceCandidates, dtlsParameters}
  │                               │
  │  connectTransport {dtls}      │
  │ ────────────────────────────► │ transport.connect()
  │ ◄──────────────────────────── │ ack
  │                               │
  │  produce {kind, rtpParams}    │
  │ ────────────────────────────► │ 检查角色 → transport.produce()
  │ ◄──────────────────────────── │ produced {id}
  │                               │ 广播 newProducer 给其他人
  │                               │
  │  consume {producerId, caps}   │
  │ ────────────────────────────► │ router.canConsume() → transport.consume()
  │ ◄──────────────────────────── │ consumed {id, rtpParameters}
```

每个成员有两条 Transport：

- **send**：发布自己的轨（host/speaker 才有）
- **recv**：订阅别人的轨（所有人都有）

---

## 5. 角色是怎么强制的

不是 UI 限制，是**服务端在 produce 时校验**：

```js
if (action === 'produce') {
  if (!canPublish(peer.role)) throw new Error(`role ${peer.role} cannot publish`);
  ...
}
```

实验验证：让一个 audience 强行创建 send transport 并发布，客户端收到：

```
ERROR: role audience cannot publish
服务端拒绝: timeout produce
```

但它**仍然能正常订阅**别人的流——这就是“观众”的语义。

host 还可以通过 `setRole` 把 audience 提升为 speaker：

```js
if (action === 'setRole') {
  if (peer.role !== ROLE.host) throw new Error('only host can change roles');
  target.role = msg.role;
  broadcast(room, { action: 'members', members: members(room) });
}
```

### 升级与降级的完整行为

**升级（audience → speaker）**：服务端广播 `members`，客户端发现自己的 role 变了，自动创建 send transport 并发布：

```js
function onMembers(list) {
  const me = list.find(m => m.id === window.__peerId);
  if (me && me.role !== window.__myRole) {
    window.__myRole = me.role;
    if (me.role === 'host' || me.role === 'speaker') enablePublish();
    else disablePublish();
  }
}
```

**降级（speaker → audience）**：服务端**强制**关闭该成员的所有 producer，并广播 `producerClosed`；客户端停本地轨道、关闭 send transport；其他成员的客户端收到 `producerClosed` 后关闭 consumer、移除视频元素。

```js
// 服务端 setRole 里
if (target.role === ROLE.audience) {
  for (const [id, p] of target.producers) {
    p.close(); target.producers.delete(id); room.producers.delete(id);
    broadcast(room, { action: 'producerClosed', producerId: id, peerId: target.id });
  }
}
```

为什么要在服务端强制？因为客户端不可信——即使客户端不配合，服务端也要保证“audience 不能发布”。

**退出房间**：客户端关闭 producers/consumers/transports、停止本地轨道、清空 UI，并发送 `leave` 关闭 WebSocket；服务端在 `close` 事件里清理该成员的 transport 和 producer，广播 `producerClosed` 和 `members`，房间空了就关闭 Router。

---

## 6. 实测日志（2026-11-12）

```
[room r2] created (router 0132a6d5)
[room r2] join "Alice" role=host peers=1
[room r2] PRODUCE audio by "Alice" (host) id=7ef99caa
[room r2] PRODUCE video by "Alice" (host) id=9f2472f7
[room r2] join "Bob" role=audience peers=2
[room r2] CONSUME audio by "Bob" (audience) <- producer 7ef99caa
[room r2] CONSUME video by "Bob" (audience) <- producer 9f2472f7
[room r2] join "Carol" role=audience peers=3
[room r2] CONSUME audio by "Carol" (audience) <- producer 7ef99caa
[room r2] CONSUME video by "Carol" (audience) <- producer 9f2472f7
[room r2] leave "Alice" peers=2
[room r2] leave "Bob" peers=1
[room r2] leave "Carol" peers=0
[room r2] closed
```

房间状态接口：`GET /state`

```json
[{"id":"r2","peers":[
  {"name":"Alice","role":"host","producers":[{"kind":"audio"},{"kind":"video"}]},
  {"name":"Bob","role":"audience","producers":[]}
],"producers":2}]
```

---

## 7. 和 SRS 的模型对比

| 场景 | SRS | mediasoup |
|---|---|---|
| 直播 | 一路推流，多路拉流 | 一个 Producer，多个 Consumer |
| 会议 | 不适合（流模型） | 天然适合（每人 Producer） |
| 角色 | 无 | 信令层自定义 |
| 房间 | 无 | Router 即房间 |
| 权限 | callback 鉴权 | 信令层直接校验 |
| 扩展 | 级联、边缘 | 多 Worker、PipeTransport |

一句话：**SRS 是“一条流给很多人看”，mediasoup 是“很多人互相看”。**

---

## 8. 怎么自己试

```bash
cd /Volumes/SSD/Code/mediasoup-lab
node server.js
```

浏览器打开 `http://127.0.0.1:3030`：

1. 第一个标签：room=demo，name=Alice，role=host → 加入，自动发布摄像头/麦克风
2. 第二个标签：room=demo，name=Bob，role=audience → 加入，能看到 Alice 的画面
3. 第三个标签：role=audience → 也能看
4. host 可以在成员列表里把 audience 提升为 speaker

调试接口：

```bash
curl -s http://127.0.0.1:3030/state | python3 -m json.tool
```

---

## 9. 这个实验没做的（生产要考虑）

- 鉴权（token、房间密码）
- **心跳超时**：当前依赖 WebSocket 关闭事件，进程被强杀时成员可能残留（实测遇到过“僵尸成员”，重启服务才清掉）
- 角色持久化与断线重连
- Simulcast / SVC（demo 只用了单层 VP8）
- 录制、转推、混流
- 多 Worker 与 PipeTransport 级联
- TURN（当前只监听 127.0.0.1，跨网需要 TURN）

这些是 mediasoup 生产化的常见工作，面试里可以主动提，说明你知道 demo 和生产的差距。
