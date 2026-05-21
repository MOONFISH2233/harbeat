# Jetson 后端 → App 端 交接文档

> 状态：2024 cypher 改造完成后第一版。所有接口已在 Jetson 实现，可在 `http://<jetson>:8000` 或经云网关 `https://<gw>/...` 直接调用。
> 维护人：Jetson 后端组
> 关联设计文档：`docs/cypher-feature-flows.md`、`docs/team-mobile-app.md`

---

## 0. 基础约定

| 项 | 值 |
|---|---|
| Base URL（直连 Tailscale） | `http://100.87.142.21:8000` |
| Base URL（云网关，公网） | `https://<aliyun-gw>/`（与 Jetson 直连同 schema，由网关透传） |
| 鉴权 | JWT Bearer Token，header：`Authorization: Bearer <token>` |
| 响应包络 | `{"code":0,"message":"ok","data":<T>}`；非 0 即业务错 |
| 时间 | 全部 UTC ISO-8601（`2024-05-01T08:00:00`），后端 naive datetime |
| song_id | 字符串（uuid hex），来自 LibrarySong；切勿与 catalog `songs.id` 整数混淆 |

错误约定：

| HTTP | 含义 |
|---|---|
| 401 | token 无效或过期，前端跳登录 |
| 403 | 不是你的资源 |
| 404 | 资源不存在 |
| 409 | 资源未就绪（例如歌曲未分析完）|
| 5xx | 服务端异常，重试 + 上报 |

---

## 1. 用户与登录（既有）

- `POST /api/users/register` — 注册
- `POST /api/users/login` — 登录，返回 `access_token`
- `GET  /api/users/me` — 当前用户

> 已稳定，无变动。

---

## 2. 歌曲库 Library

### 2.1 上传

`POST /api/library/songs/upload` （multipart/form-data，`file` 字段）

后端会：
1. 计算文件 sha256
2. 推入分析队列（Phase1 BPM/beat → Phase2 Demucs stems → Phase3 CLAP embedding）
3. 返回 `LibrarySongData`，初始 `analysis_status="pending"`

### 2.2 列表

`GET /api/library/songs?only_ready=true|false`

- `only_ready=true` 时只返回 `analysis_status="completed"` 的歌（**App 进入 cypher 模式应传这个**）
- 默认 `false`，返回全部（含未分析完的）

### 2.3 单曲

`GET /api/library/songs/{song_id}`

返回完整字段（含 `analysis_status / analysis_stage / analysis_error / analyzed_at`）。

### 2.4 ★ 状态轮询（新增，cypher 协议 P1）

`GET /api/library/songs/{song_id}/status`

返回：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "song_id": "ab12...uuid",
    "title": "...", "artist": "...",
    "duration_sec": 213.4,
    "bpm": 128.0, "key": "8B",
    "analysis_status": "ready",
    "analysis_stage": "embed_done",
    "analysis_error": null,
    "analyzed_at": "2024-05-01T08:00:00",
    "has_stems": true
  }
}
```

`analysis_status` 取值（**已映射为 cypher 协议词汇**）：

| 值 | 含义 |
|---|---|
| `pending` | 入队中，尚未分析 |
| `analyzing` | 正在分析 |
| `ready` | 分析完成，可投入 cypher / 离线 mix |
| `failed` | 分析失败，看 `analysis_error` |

`analysis_stage`（更细的进度，可用作进度条）：
`none → beats_done → stems_done → embed_done`

**App 端推荐轮询策略**：

- 上传后 → 间隔 2s 拉一次，直到 `analysis_status ∈ {ready, failed}`
- 也可改用 `only_ready=false` 列表 + 客户端缓存
- 每首歌典型耗时 30~90s（不含排队）

### 2.5 删除

`DELETE /api/library/songs/{song_id}` — 现状不变。

---

## 3. 歌单 Playlist

| 接口 | 说明 |
|---|---|
| `POST /api/playlists/create-empty` | 建空歌单，返回 `playlist_id`（int）|
| `GET  /api/playlists` | 我的歌单列表 |
| `GET  /api/playlists/{playlist_id}` | 详情（含每首歌的分析状态、bpm、key）|
| `POST /api/playlists/{playlist_id}/add-library-songs` | body: `{"library_song_ids":[...]}` |
| `POST /api/playlists/{playlist_id}/reorder` | 重排序 |
| `POST /api/playlists/{playlist_id}/update-song-tags` | 改标签 |
| `DELETE /api/playlists/{playlist_id}` | 删除 |
| `POST /api/playlists/import` | 导入 |

> 这一块改造前已稳定，App 端继续按原逻辑用即可。

---

## 4. ★ DJ MixPlan（新增 SSE，cypher 协议 P2）

老接口仍可用：

- `POST /api/playlists/generate-dj-mix-plan` — 一次性返回；典型耗时 60~280s（同步阻塞）

**推荐切换到 SSE 版**：

`POST /api/playlists/{playlist_id}/dj-mix-stream`

请求体：和 `generate-dj-mix-plan` 完全一致（`DjMixPlanRequest`），但 `playlist_id` 自动覆盖。

响应：`Content-Type: text/event-stream`

事件流：

```
event: plan_started
data: {"playlist_id":12,"cache_hit":true}

event: plan_final
data: {"playlist_id":12,"result":{...DjMixPlanResult...},"elapsed_sec":0.04}
```

- `cache_hit=true` 时 ~50ms 内拿到结果（Redis 缓存，TTL 7 天）
- `cache_hit=false` 时仍需等待计算完成（数十秒～数分钟）
- 失败：`event: error data: {"message":"..."}`

最近一次缓存可单独拉：

`GET /api/playlists/{playlist_id}/mix-plan/latest` — 返回最近成功的 MixPlan，无则 404。

> **App 端接入建议**：用 EventSource 或 `fetch + ReadableStream`；展示 `plan_started` 后立刻显示骨架屏，`plan_final` 后展示完整 mix 时间轴。

### 4.1 离线渲染（既有）

`POST /api/playlists/generate-dj-offline-mix` — 渲染出 WAV/MP3 文件，不变。

---

## 5. ★ AssetManifest（新增，cypher 协议 P3）

供 App 端**展示下载进度**和**校验完整性**用。RK3588 也会调同一接口拉清单。

`GET /api/playlists/{playlist_id}/manifest?plan_id=<可选>`

成功（200）：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "plan_id": null,
    "playlist_id": 12,
    "tracks": [
      {
        "song_id": 88,
        "library_song_id": "ab12...uuid",
        "title": "...", "artist": "...",
        "duration_sec": 213.4, "bpm": 128.0, "key": "8B",
        "files": {
          "original": {
            "url": "/api/stream/ab12...uuid",
            "size": 5234123,
            "sha256": "f2c1...",
            "format": "mp3"
          },
          "stems": {
            "vocals": { "url": "/api/stream/ab12...uuid/stem/vocals", "size": 2..., "sha256": "..." },
            "drums":  { "url": "/api/stream/ab12...uuid/stem/drums",  "size": ..., "sha256": "..." },
            "bass":   { "url": "/api/stream/ab12...uuid/stem/bass",   "size": ..., "sha256": "..." },
            "other":  { "url": "/api/stream/ab12...uuid/stem/other",  "size": ..., "sha256": "..." }
          }
        }
      }
    ]
  }
}
```

未就绪（409）：

```json
{
  "code": 1,
  "message": "songs not ready",
  "detail": {
    "error": "songs not ready",
    "tracks": [{"song_id":88,"title":"x","status":"analyzing"}, ...]
  }
}
```

> sha256 首次计算时较慢（按文件大小，几百毫秒～几秒），结果会写回 DB，后续秒级返回。

---

## 6. 音频流 Stream

| 接口 | 说明 |
|---|---|
| `GET /api/stream/{song_id}` | 原始音频，支持 Range |
| `GET /api/stream/{song_id}/stem/{vocals\|drums\|bass\|other}` | 单轨 stem，支持 Range |

- 直接给 `<audio>` 或 ExoPlayer/AVPlayer 用
- 也可用 manifest 中的 `url` 字段拼出来（已自带 `/api/` 前缀，需补 baseUrl）
- 优先返回 MP3（后端会把 Demucs WAV stems 转 MP3 留作播放用）

---

## 7. Session（练舞 / cypher 现场）

App 端如果走自己跑 cypher（非 RK3588 现场盒）：

- `POST /api/sessions/start` — 开 session
- `POST /api/sessions/event` — 上报事件
- `POST /api/sessions/end` — 关 session
- `GET  /api/sessions/practice-list` — 推荐练舞清单
- `POST /api/sessions/interaction-log` — 用户交互日志

如果是 RK3588 现场盒：App **不直接调** session 接口，由 RK 自己上报（见 RK3588 文档第 5 节）。App 想看本场事件流用：

- `GET /api/sessions/rk/{session_id}/events?type=&limit=500`（需 token + 可选 `X-RK-Token` header）

---

## 8. 设备/网关相关

- `GET /health` — Jetson 健康检查
- `GET /jetson/health`（云网关）— 网关回连 Jetson 探测
- `GET /edge/registry`（云网关）— 列出当前已注册的 RK 设备 id

App 一般只需访问 `https://<gw>/`（公网），云网关会自动转发到 Jetson；走 Tailscale 时直接打 Jetson。

---

## 9. 关键改动一览（vs 旧版）

| 改动 | 影响 App |
|---|---|
| LibrarySong 新增字段 `analysis_stage / analysis_error / analyzed_at / original_sha256 / stems_sha256` | 自动迁移，无需 App 关心；可消费 `analysis_stage` 做进度条 |
| `/api/library/songs?only_ready=true` 新增 | 选择性使用 |
| `/api/library/songs/{id}/status` 新增 | 用于轮询 |
| `/api/playlists/{id}/manifest` 新增 | cypher 模式必用 |
| `/api/playlists/{id}/dj-mix-stream` 新增 | 推荐替代 `/generate-dj-mix-plan` |
| `/api/playlists/{id}/mix-plan/latest` 新增 | 可省去重算 |
| `/api/sessions/rk/{sid}/events` 新增 | App 可只读，不需写 |

---

## 10. cURL 速查

```bash
# 状态轮询
curl -H "Authorization: Bearer $TOKEN" \
  http://100.87.142.21:8000/api/library/songs/ab12abcd/status

# 仅 ready 的库
curl -H "Authorization: Bearer $TOKEN" \
  "http://100.87.142.21:8000/api/library/songs?only_ready=true"

# Manifest
curl -H "Authorization: Bearer $TOKEN" \
  http://100.87.142.21:8000/api/playlists/12/manifest

# SSE MixPlan
curl -N -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"style":"hiphop","duration_minutes":30}' \
  http://100.87.142.21:8000/api/playlists/12/dj-mix-stream
```

---

## 11. FAQ

**Q: `analysis_status` 老的 `none/pending/analyzing/completed/error` 还能用吗？**
A: 在所有「非 `/status` 端点」返回的 LibrarySong 对象里仍是老值（向后兼容）。只有 `/status` 端点做了 cypher 词汇映射。App 端二选一即可，建议统一用 `/status`。

**Q: manifest 报 409 怎么处理？**
A: 把 `detail.tracks` 列出哪些歌还在跑，进入轮询模式（`/status`）直到全部 `ready`，再重拉 manifest。

**Q: stream 的 sha256 不一致？**
A: 多半是文件被替换/重传，后端 `analyzed_at` 之后才写 sha256，manifest 内的 sha256 是当时计算的快照；重传后重新分析即可。

**Q: SSE 断线？**
A: 重连后再发同样的 POST，命中 Redis 缓存会即时返回 `plan_final`，无副作用。
