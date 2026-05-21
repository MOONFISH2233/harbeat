# Jetson 后端 → RK3588 现场盒 交接文档

> 状态：cypher 改造完成第一版。
> 维护人：Jetson 后端组
> 关联文档：`docs/cypher-feature-flows.md`、`docs/team-rk3588-edge.md`

---

## 0. 网络与地址

| 项 | 值 |
|---|---|
| RK 到 Jetson 直连 | Tailscale，`http://100.87.142.21:8000` |
| RK 到云网关 | `https://<aliyun-gw>/`（仅在 Jetson 不可达时使用，**带宽贵**）|
| 反向：App/管理端 → RK | 经云网关 `https://<gw>/edge/{rk_id}/{path}` 透传到 RK 自己暴露的 HTTP（默认 `:9000`）|

注册表（云网关 `deploy/cloud_gateway/app/main.py`）：

```python
RK_REGISTRY = {
    "rk-001": os.getenv("RK001_BASE_URL", "http://100.91.30.54:9000"),
}
```

新加设备：在该 dict 加一行或设环境变量 `RKnnn_BASE_URL` 然后重启网关。

鉴权：

- **RK → Jetson**：和 App 同一套 JWT（推荐为每台 RK 单建一个 service-account 用户）。
  外加可选 header `X-RK-Token: <值>`（用于 `/api/sessions/rk/*` 路径，环境变量 `HARBEAT_RK_TOKEN` 在 Jetson 上配置；不设则不校验）。
- **管理端 → RK**：通过云网关透传，RK 自行做鉴权。

---

## 1. 启动流程（RK 上电后）

1. 设备自检 → 上报 DeviceInfo（协议 P8）：可选，暂未在 Jetson 强制要求；先记日志即可
2. 等待用户在 App 选歌单 + style 触发 cypher
3. App 调 Jetson `POST /api/playlists/{playlist_id}/dj-mix-stream` 拿到 `MixPlan`
4. App 调 Jetson `GET /api/playlists/{playlist_id}/manifest` 拿到 `AssetManifest`
5. App 把 `MixPlan` + `AssetManifest` 通过 LAN/云网关推给 RK
6. RK 按 manifest 下载所有 `original` + 4 个 `stems` 到本地（**带 sha256 校验**）
7. RK 拉起本地 GstPlayer / 自研播放器，按 MixPlan 切换

> 说明：MixPlan ↔ AssetManifest 协议层都在 Jetson 已落地。但「App 把这俩推给 RK」的具体通道（蓝牙 / Wi-Fi Direct / 局域 WS）属于 App ↔ RK 私有协议，**Jetson 不参与**。RK 也可以自己直连 Jetson 拉。

---

## 2. ★ 资产清单 AssetManifest（协议 P3）

`GET /api/playlists/{playlist_id}/manifest`

详见 App 文档第 5 节。RK 关注点：

- 每首歌的 `files.original.url` 与 `files.stems.{vocals,drums,bass,other}.url`
- 每个文件的 `size`（用于校验长度）+ `sha256`（用于校验完整性）
- 全部 URL 形如 `/api/stream/<song_id>` —— **缺 baseUrl，RK 端要拼**：

```
final_url = jetson_base + entry.url
         = "http://100.87.142.21:8000" + "/api/stream/ab12..."
```

返回 409 时 `detail.tracks` 列出未 ready 的歌。RK 端策略：

- 不要立即重试。回到 App 让用户决定（或等 App 重新触发）。

---

## 3. ★ 文件下载（HTTP Range）

`GET /api/stream/{song_id}`
`GET /api/stream/{song_id}/stem/{vocals|drums|bass|other}`

- 支持 `Range: bytes=...`，可断点续传
- Content-Type：mp3 / wav / mp4 等按源文件
- 建议下载策略：
  - **首曲 fully + 部分预取**：第一首全文件下载 → 解码 → 立刻可放；同时后台 prefetch 下一首
  - **分块**：可设 8MB chunk + 失败回退到 1MB
  - **并发**：单 song 内 stems 可并发（4 路），跨 song 串行（避免 Jetson IO 抖动）
  - **超时**：connect 5s / read 30s / total 5min
  - **重试**：最多 3 次，指数退避（1s/3s/9s）
- 校验：下载后比对 `size`（必须）和 `sha256`（必须）；不符直接删文件 + 重下载

---

## 4. ★ MixPlan（协议 P2，从哪里拿）

RK 一般不直接调 `dj-mix-stream`（那是 App 触发的）。RK 可以：

- 接受 App 推过来的 `MixPlan` JSON（推荐路径）
- 或自己拉 `GET /api/playlists/{playlist_id}/mix-plan/latest` 取缓存

返回结构 `DjMixPlanResult`：

```json
{
  "playlist": [PlaylistSongData...],
  "processed_files": {"<song_id>": "/.../processed.wav", ...},
  "meta": {"<song_id>": {...}},
  "transition_plan": [
    {"from_song_id": 88, "to_song_id": 89,
     "from_out_sec": 200.0, "to_in_sec": 0.0,
     "crossfade_sec": 8.0, "match_score": 0.91, ...}, ...
  ]
}
```

RK 只需消费：

- `playlist[*].song_id` / `library_song_id` ↔ 与 manifest 对齐
- `transition_plan[*]` → 转场参数（出点、入点、淡入淡出时长）
- 其他字段供调试 / metrics

---

## 5. ★ SessionEvent 上报（协议 P7，新增）

RK3588 现场盒每场 cypher 都生成一个 `session_id`（UUID，自管理），向 Jetson **批量**上报事件。

`POST /api/sessions/rk/{session_id}/events`

Header：
- `Authorization: Bearer <jwt>`（service account）
- `Content-Type: application/json`
- `X-RK-Token: <token>`（如 Jetson 设了 `HARBEAT_RK_TOKEN`）

Body：

```json
{
  "rk_id": "rk-001",
  "events": [
    {"ts": "2024-05-01T08:00:00", "type": "play_started", "data": {"song_id": 88}},
    {"ts": "2024-05-01T08:00:30", "type": "crossfade_start",
     "data": {"from": 88, "to": 89, "fade_sec": 8}},
    {"ts": "2024-05-01T08:00:38", "type": "crossfade_end", "data": {}},
    {"ts": "2024-05-01T08:00:45", "type": "key_press",
     "data": {"key": "C", "fx": "echo", "latency_ms": 18}},
    {"ts": "2024-05-01T08:01:00", "type": "bpm_lock", "data": {"bpm": 128.0}}
  ]
}
```

响应：

```json
{"code":0,"message":"ok","data":{"accepted":5,"session_id":"<sid>"}}
```

**RK 端 flush 策略**（建议）：

- 满 50 条事件 **或** 5 秒（取早者）→ flush 一次
- 失败时本地 sqlite 落盘 + 启动定时重试
- 网络断开期间继续本地累计；恢复后顺序上送

**事件 `type` 推荐词表**（已与 App 团队对齐）：

| type | data 字段 |
|---|---|
| `play_started` | `{song_id, position_sec}` |
| `play_paused` | `{song_id, position_sec}` |
| `crossfade_start` | `{from, to, fade_sec, mode}` |
| `crossfade_end` | `{from, to}` |
| `key_press` | `{key, fx, latency_ms}` （9-key FX 触发）|
| `bpm_lock` | `{bpm}` |
| `bpm_unlock` | `{}` |
| `stem_mute` | `{stem, muted: bool}` |
| `cue_set` | `{position_sec}` |
| `cue_jump` | `{position_sec}` |
| `error` | `{code, message}` |
| `heartbeat` | `{cpu_pct, mem_mb, temp_c}`（可选）|

查询本场事件：

`GET /api/sessions/rk/{session_id}/events?type=<可选过滤>&limit=500`

返回最近的 N 条（按 ts 倒序），用于 App 端事后回放或后台分析。

---

## 6. 设备健康 / 心跳

暂未要求 RK 主动调 Jetson 心跳。建议做法：

- RK 自己在 `:9000` 上暴露 `GET /health` → `{"status":"ok","cpu":...,"temp":...}`
- 云网关已加 `GET /edge/registry` 列出已注册 rk_id
- 后续可以通过 `/edge/{rk_id}/health` 透传查询

---

## 7. 反向通道：Jetson/App 控制 RK

**Jetson 不主动连 RK**。控制走云网关：

```
管理端 / App ─► https://<gw>/edge/rk-001/control/play  ─►  RK :9000/control/play
```

云网关 (`deploy/cloud_gateway/app/main.py`) 已实现 `/edge/{rk_id}/{path:path}` 透传：

- 支持 GET/POST/PUT/DELETE/PATCH
- 60s 超时
- 502 表示 edge 不可达（Tailscale 断 / RK 关机）
- 404 表示 `rk_id` 不在 RK_REGISTRY

WebSocket 透传暂未实现（已记入 TODO）。短期内控制指令走 HTTP；播放状态用 SessionEvent 批量上报。

---

## 8. 时间同步

Jetson 的时间是 server-side UTC。RK 上报 `ts` 时：

- 推荐 RK 自己 ntp 对时
- Jetson 收到时会**保留 RK 提供的 `ts`**，同时另存 `received_at`（Jetson 本机时钟）
- 分析时用 `ts`，重放与排序也用 `ts`

如果 RK 时钟漂移严重，后台分析时会以 `received_at` 兜底（人工纠偏）。

---

## 9. 错误与重试矩阵

| 场景 | 后端响应 | RK 行为 |
|---|---|---|
| 401 token 过期 | 401 | 重新登录 service account，取新 token 重试 |
| 403 不是你的资源 | 403 | 上报 + 不重试（配置错误）|
| 404 找不到 | 404 | 不重试 |
| 409 歌未 ready | 409 | 通知 App，等 App 重新下发 manifest |
| 5xx | 5xx | 指数退避 + 最多 3 次重试 |
| 网络断 | timeout | 本地缓存，最多保留 30 分钟，恢复后回传 |

---

## 10. 关键改动一览（Jetson 端新增/变更）

| 改动 | 对 RK 的影响 |
|---|---|
| `/api/playlists/{id}/manifest` 新增 | RK 必须用它取 url + sha256 |
| `/api/playlists/{id}/mix-plan/latest` 新增 | RK 可在 App 未推送时主动拉 |
| `/api/sessions/rk/{sid}/events` 新增 | RK 必须批量上报 |
| `X-RK-Token` header 可选校验 | 由 `HARBEAT_RK_TOKEN` 环境变量开关；不设即放行 |
| 云网关 `/edge/{rk_id}/*` 透传 | 反向控制走这里 |
| LibrarySong sha256 缓存 | 减少 manifest 接口耗时 |

---

## 11. cURL 速查

```bash
# 拉 manifest
curl -H "Authorization: Bearer $TOKEN" \
  http://100.87.142.21:8000/api/playlists/12/manifest

# 下载原始音频（Range）
curl -H "Authorization: Bearer $TOKEN" \
  -H "Range: bytes=0-1048575" \
  -o chunk0.mp3 \
  http://100.87.142.21:8000/api/stream/ab12abcd

# 上报事件
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "X-RK-Token: $RKTOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"rk_id":"rk-001","events":[{"ts":"2024-05-01T08:00:00","type":"play_started","data":{"song_id":88}}]}' \
  http://100.87.142.21:8000/api/sessions/rk/abcd-uuid/events

# 经云网关 反控 RK
curl -X POST https://<gw>/edge/rk-001/control/play \
  -H 'Content-Type: application/json' \
  -d '{"plan_id":"...","start_song":88}'
```

---

## 12. FAQ

**Q: manifest 拿到的 url 没有域名怎么用？**
A: 是 `/api/...` 形式，要自己拼 base。base 一般就是 RK 当前连 Jetson 用的那个（Tailscale ip:port）。

**Q: 下载到一半 Jetson 重启了？**
A: HTTP 会断。RK 走 Range 重连即可；sha256 在所有分片下完之后再算。

**Q: sha256 校验失败要不要联 Jetson？**
A: 重下一次。还是失败再 GET manifest 一次（sha256 也许已经被后端重算）；仍不匹配再人工介入。

**Q: SessionEvent 漏报会怎样？**
A: 后台分析会缺数据，但不影响 cypher 进行。优先保证现场体验，事后再补送。

**Q: 多台 RK 共用一台 Jetson 行不行？**
A: 行。每台 RK 用不同的 `rk_id` 上报；service account 也建议每台独立，方便追溯。

**Q: WebSocket 什么时候有？**
A: TODO。短期建议用 HTTP 长轮询或事件批量上报顶替。
