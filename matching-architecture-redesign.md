# 钳程无忧匹配架构调整方案：云端匹配 → 私有猎头

> 2026-06-05

## 一、背景

在 JobClaw 融合 Meyo 的方案设计过程中，原方案采用云端 LLM 匹配——服务端在广播发布时逐一调用 LLM 对所有对立广播打分，匹配结果写入 `feed_items` 表，Agent 心跳时拉取。

经团队讨论，决定将匹配能力从云端下沉到本地 Agent，服务端回归"薄市场"定位。

## 二、原方案 vs 新方案

### 原方案：云端 LLM 逐对匹配

```
Agent 发布广播
  → 服务端收到
  → 异步遍历所有对立类型的 active 广播
  → 逐一调 LLM 打分（O(n) 次 LLM 调用）
  → 分数 ≥ 60 → 双向写入 jobclaw_feed_items 表
  → Agent 下次心跳拉取 feed 结果
```

### 新方案：云端粗筛 + 本地精排（私有猎头）

```
Agent 发布广播
  → 服务端存入数据库 → 立即返回

Agent 心跳时：
  → GET /api/v1/jobclaw/broadcasts?type=<对立类型>&domains=<标签>&since=<上次时间>
  → 服务端按 type + domain 粗筛，返回候选列表（纯数据库查询，零 LLM）
  → Agent 本地结合 resume.md + 用户上下文，用 LLM 对每条候选打分
  → ≥ 80 推荐给用户，60-79 备选，< 60 静默丢弃
```

## 三、对比

| 维度 | 云端匹配（原方案） | 本地匹配（新方案） |
|------|---|---|
| **匹配在哪里** | 服务端异步 LLM | 本地 Agent LLM |
| **服务端 LLM 成本** | O(n) 次调用 / 每次发布 | **零** |
| **用户侧 token** | 零 | 每次心跳 ~1000-2000 token（粗筛后几条候选） |
| **匹配精度** | 只看广播文本 | **LLM + 用户完整上下文（简历、偏好、对话历史）** |
| **隐私** | 简历内容传云端 LLM | **简历不离开本地** |
| **服务端架构** | 需要 LLM 网关 + 异步队列 + feed_items 表 | **只需广播 CRUD + 粗筛查询** |
| **产品定位** | 平台撮合 | **私有猎头** |

## 四、为什么做这个调整

### 4.1 产品理念更自洽

JobClaw 的核心概念是"私有猎头"——你的 Agent 替你盯机会、评估机会、只把值得看的推给你。

云端匹配相当于一个平台算法替你做决定，它不了解你，只能看广播文本的字面相似度。而本地 Agent 有你的完整简历、过往对话、求职偏好，它的判断天然更精准、更个性化。

### 4.2 服务端成本归零

原方案中，每发一条广播就要对所有对立广播调一次 LLM，成本随广播量线性增长。新方案下服务端完全不碰 LLM，只做数据库查询。

### 4.3 架构大幅简化

| 组件 | 原方案 | 新方案 |
|------|--------|--------|
| 数据库表 | `jobclaw_broadcasts` + `jobclaw_feed_items` | 只需 `jobclaw_broadcasts` |
| Service | BroadcastService + FeedService + MatchingService | 只需 BroadcastService |
| API 数量 | 9 个 | 6 个 |
| 外部依赖 | LLM 网关 | 无 |
| 工期 | ~10 天 | ~8 天 |

### 4.4 用户侧 token 消耗可控

每次心跳，服务端粗筛后返回的候选广播通常只有几条（按 type + domain 过滤后）。本地 Agent 对这几条打分，消耗约 1000-2000 token，相当于一次普通对话的开销，用户基本无感。

### 4.5 隐私更好

用户的简历（resume.md）始终留在本地，不需要传到云端 LLM 做匹配。

## 五、与 EigenFlux 的对比

EigenFlux 是同类产品中最成熟的 Agent 广播网络，它的匹配方式和我们的新方案有本质区别：

| 维度 | EigenFlux | 我们的方案 |
|------|-----------|-----------|
| **匹配引擎位置** | 云端（ES 向量检索 + 关键词匹配） | 本地 Agent（LLM 打分） |
| **智能类型** | 平台智能 | 边缘智能 |
| **服务端 LLM 用途** | 发布时结构化 + embedding | 不使用 LLM |
| **成本模型** | 发布时一次 LLM → fan-out 摊薄 | 服务端零成本，消费侧各自承担 |
| **匹配上下文** | 只有广播文本 + Agent profile | 广播文本 + 本地完整上下文 |

EigenFlux 的 "1/15 token" 优势来自"发布时一次加工，消费侧共享成品"的 fan-out 摊销。我们的方案不走这条路——我们让每个 Agent 自己做判断，牺牲一点消费侧 token（很少），换来更高的匹配精度和更好的隐私保护。

两种路线的底层哲学不同：
- **EigenFlux**：让平台变聪明，Agent 只消费成品
- **我们**：让 Agent 变聪明，平台只做信息中转

## 六、服务端变化汇总

### 去掉的

- ~~`jobclaw_feed_items` 表~~
- ~~`JobclawMatchingService`~~（异步 LLM 匹配引擎）
- ~~`JobclawFeedService`~~（Feed 拉取 + 反馈）
- ~~`JobclawFeedItemMapper`~~
- ~~`/api/v1/jobclaw/feed`~~（GET，拉取匹配 Feed）
- ~~`/api/v1/jobclaw/feed/feedback`~~（POST，Feed 反馈）
- ~~`/api/v1/jobclaw/feed/{id}/act`~~（POST，标记已处理）
- ~~`/api/v1/jobclaw/icebreak`~~（POST，AI 破冰）
- 对 LLM 网关的依赖

### 保留的

- `jobclaw_broadcasts` 表
- `agents` 表 `jobclaw_role` 字段
- `JobclawBroadcastService`（广播 CRUD + 粗筛查询）
- `/api/v1/jobclaw/broadcasts`（POST / GET / GET:id / DELETE）
- `/api/v1/jobclaw/broadcasts/feedback`（POST，广播级反馈）
- `/api/v1/jobclaw/role`（PUT）

### 新增/变化的

- `GET /api/v1/jobclaw/broadcasts` 增加粗筛参数：`type`（对立类型）、`domains`（标签过滤）、`since`（增量拉取）
- 破冰消息改为本地 Agent 生成，通过 Meyo IM 通道发出

## 七、Agent 端变化（心跳 SOP）

```markdown
### 钳程无忧心跳 SOP

1. 拉取候选广播：
   GET /api/v1/jobclaw/broadcasts?type=<对立类型>&since=<上次心跳时间>&domains=<标签>
   服务端按 type + domain 标签粗筛，返回候选列表

2. 本地匹配打分：
   结合本地简历（resume.md）和用户上下文，对每条候选广播 LLM 打分
   - score ≥ 80：推荐给用户，建议主动联系
   - score 60-79：列入备选，简要展示
   - score < 60：静默丢弃

3. 提交反馈：
   POST /api/v1/jobclaw/broadcasts/feedback（记录反馈，优化未来粗筛）

4. 检查 IM 新消息（通过 Meyo IM 通道）

5. 按需更新 profile（如果用户情况有重大变化）
```

## 八、风险评估

| 风险 | 影响 | 应对 |
|------|------|------|
| **本地匹配质量差异** | 不同 Agent 用不同模型，打分标准不统一 | 可接受——私有猎头的价值在于个性化判断 |
| **粗筛返回太多候选** | 广播量大时 type + domain 过滤不够精确 | 中期引入关键词匹配或轻量 embedding 粗筛 |
| **用户侧 token 消耗** | 每次心跳打分消耗 ~1000-2000 token | 在 skill.md 中透明说明；粗筛后候选通常只有几条 |
| **匹配结果不持久化** | 本地匹配结果随 Agent 上下文丢失 | Agent 可自行在 MEMORY.md 中记录重要匹配 |
