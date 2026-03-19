# 交易监控系统 技术方案与架构设计

> 状态：草稿，待评审
> 依赖文档：transaction-monitoring-design-v0.md
> 最后更新：2026-03-19

---

## 一、架构总览

```
┌──────────────────────────────────────────────────────────────┐
│                      用户端（Browser）                        │
│   结算流程页面（主站）              监控 Dashboard（独立 Next.js）│
│   └── Transaction Tracking Module  └── ECharts 图表           │
│        ├── 生成 transaction_id          ├── 工程师视图          │
│        └── 上报事件                     ├── PM 视图            │
│                                         └── 领导视图           │
└───────────┬──────────────────────────────────┬───────────────┘
            │ POST /api/events/ingest           │ GET /api/monitoring/*
            ▼                                  ▼
┌──────────────────────────────────────────────────────────────┐
│              Next.js 独立项目（API Routes）                    │
│                                                              │
│  /api/events/ingest       事件接收、校验、写入 Redis 队列        │
│  /api/monitoring/health   读 Redis → 健康状态                  │
│  /api/monitoring/funnel   查 ClickHouse MV → 漏斗数据          │
│  /api/monitoring/channels 查 ClickHouse MV → 渠道数据          │
│  /api/monitoring/errors   查 ClickHouse → 错误分布             │
│                                                              │
│  /api/cron/flush-events   [Cron] Redis 队列 → ClickHouse      │
│  /api/cron/aggregate      [Cron] ClickHouse → Redis 健康缓存   │
│  /api/cron/push-datadog   [Cron] 推送指标到 Datadog            │
│  /api/cron/check-accounts [Cron] 支付账号健康检测              │
└───────┬──────────────────────────────────────┬───────────────┘
        │                                      │
        ▼                                      ▼
┌───────────────────┐              ┌───────────────────────────┐
│   Redis           │              │   ClickHouse              │
│   事件缓冲队列     │              │   transaction_events      │
│   健康状态缓存     │              │   funnel_hourly_mv        │
│   告警状态         │              │   channel_hourly_mv       │
└───────────────────┘              └───────────────────────────┘
        │
        ▼
┌───────────────────┐
│   PostgreSQL      │
│   市场配置         │
│   渠道配置         │
│   告警规则         │
│   告警历史         │
└───────────────────┘
        │
        ▼
┌──────────────────────────────────┐
│   告警层                          │
│   Datadog Monitors + Slack        │
└──────────────────────────────────┘
```

---

## 二、职责边界

监控系统作为独立 Next.js 项目，前端工程师完全主导，包含前后端逻辑。

| 层 | 拥有方 | 说明 |
|---|---|---|
| Transaction Tracking Module | 前端（主站） | 独立 TS 模块，打包后引入主站结算页 |
| Event Schema 定义 | 监控项目（共享类型） | 主站和监控项目共用同一套类型定义 |
| Event Ingestion API Route | 监控项目 | `/api/events/ingest`，完全自主 |
| Dashboard API Routes | 监控项目 | `/api/monitoring/*`，完全自主 |
| Cron Job API Routes | 监控项目 | `/api/cron/*`，由外部调度触发 |
| Dashboard 页面 | 监控项目 | Next.js App Router + ECharts |
| 数据存储（ClickHouse/Redis/PG） | 基础设施/运维 | 监控项目消费，运维团队维护实例 |
| 告警规则与通知 | 监控项目定义 | 规则存 PostgreSQL，触发由 Cron Job 执行 |

---

## 三、技术选型

### 3.1 前端追踪模块

**选型：自建轻量 TypeScript 模块**

不使用 Segment / Amplitude 等第三方 CDP 的原因：
- 跨境多市场有数据出境合规要求，第三方 CDP 的数据流向不可控
- 核心数据（交易失败率、支付渠道数据）不应依赖外部服务的可用性
- 现有 GA 继续保留，用于通用行为分析，本模块专注交易链路

**模块职责：**
- 生成并管理 `transaction_id`（进入结算流程时创建，存入 sessionStorage）
- 监听并上报交易生命周期事件
- 捕获支付 SDK 的回调结果
- 批量上报（每 N 秒或每 X 条事件发送一次，减少请求数）
- 失败重试（本地队列，页面关闭前 flush）

```typescript
// 核心接口定义
interface TransactionEvent {
  transaction_id: string;      // 前端生成的 UUID
  event_type: TransactionEventType;
  timestamp: number;           // Unix ms
  market: string;              // 'DE' | 'JP' | 'US' ...
  channel: 'online';           // v1 只有 online
  session_id: string;          // 关联同一用户会话
  payment_method?: string;     // 支付方式（payment_selected 之后才有）
  failure_reason?: FailureReason;
  metadata?: Record<string, unknown>; // 扩展字段，不放 PII
}

type TransactionEventType =
  | 'checkout_initiated'
  | 'address_entered'
  | 'shipping_selected'
  | 'payment_selected'
  | 'payment_submitted'
  | 'payment_completed'
  | 'payment_failed'
  | 'checkout_abandoned';

type FailureReason =
  | 'api_timeout'
  | 'sdk_error'
  | 'payment_declined'
  | 'validation_error'
  | 'network_error'
  | 'unknown';
```

**为什么用 sessionStorage 存 transaction_id：**
- 跨页面刷新保留（比内存变量稳定）
- 关闭标签页自动清除（比 localStorage 安全）
- 同一会话内多次进入结算，需要判断是新交易还是重试（通过时间戳判断）

---

### 3.2 Event Ingestion API（接口契约）

**前端对后端的要求：**

```
POST /api/events/transaction
Content-Type: application/json
Authorization: Bearer {internal_token}

Body:
{
  "events": [TransactionEvent]  // 批量上报
}

Response:
{
  "accepted": number,     // 成功接收的事件数
  "rejected": number,     // 拒绝的事件数（格式错误等）
  "server_transaction_id": string  // 后端绑定的订单 ID（如果已有）
}
```

**接口要求（前端对后端提出）：**
- 响应时间 P95 < 200ms（不能影响结算主流程）
- 支持批量（单次最多 50 条事件）
- 后端异步处理，不等待写库完成再返回（fire-and-forget 模式）
- 接口失败不影响结算流程（前端捕获错误，静默处理）

---

### 3.3 数据存储

**这是后端的决策领域，前端提出查询需求，后端选择实现方式。**

**前端需要支持的查询场景（告知后端）：**

| 查询场景 | 时间范围 | 维度 | 频率 |
|---|---|---|---|
| 实时健康状态 | 最近 5 分钟 | 全局 + 市场 | 每 30 秒轮询 |
| 漏斗转化率 | 自定义（最长 90 天） | 市场、时间 | 按需 |
| 渠道成功率 | 自定义（最长 90 天） | 市场、渠道 | 按需 |
| 错误分布 | 最近 7 天 | 错误类型、市场 | 按需 |
| 账号异常检测 | 滚动 7 天基线 | 市场、账号 | 每小时 |

**推荐方向（供后端参考）：**
- 事件明细存储：ClickHouse（列式存储，聚合查询性能优秀，适合时序事件数据）
- 实时聚合缓存：Redis（健康状态、告警判断，TTL 5 分钟）
- 配置数据：PostgreSQL（市场配置、渠道配置、告警规则，读多写少）

**不推荐：**
- 直接在 PostgreSQL 上做全量事件查询（数据量大后性能不可接受）
- 实时写 Elasticsearch（运维成本高，查询场景不匹配）

---

### 3.4 Dashboard API（接口契约）

**前端定义需要的数据格式，后端实现。**

核心接口：

```
# 健康总览
GET /api/monitoring/health
Response: {
  status: 'healthy' | 'warning' | 'critical',
  markets: [{
    market: string,
    status: string,
    conversion_rate: number,
    technical_failure_rate: number,
    updated_at: number
  }],
  active_alerts: Alert[]
}

# 漏斗数据
GET /api/monitoring/funnel?market=DE&from=&to=
Response: {
  steps: [{
    step: TransactionEventType,
    entered: number,
    completed: number,
    conversion_rate: number,
    failure_breakdown: Record<FailureReason, number>
  }]
}

# 支付渠道表现
GET /api/monitoring/channels?market=DE&from=&to=
Response: {
  channels: [{
    payment_method: string,
    selection_rate: number,
    success_rate: number,
    p50_ms: number,
    p95_ms: number,
    volume: number
  }]
}

# 技术错误追踪
GET /api/monitoring/errors?market=DE&from=&to=
Response: {
  errors: [{
    failure_reason: FailureReason,
    count: number,
    affected_transactions: number,
    estimated_revenue_impact: number
  }]
}
```

**API 设计原则：**
- 所有接口返回聚合数据，不返回明细事件（Dashboard 不做 raw data 查询）
- 市场、时间范围作为 query 参数，不做路由区分
- 货币统一返回原始金额 + 货币代码，前端负责展示格式化

---

### 3.5 Dashboard 前端

**独立 Next.js 项目**

项目结构：
```
monitoring-dashboard/
├── app/
│   ├── (auth)/
│   │   └── login/page.tsx
│   ├── ops/page.tsx            工程师视图
│   ├── product/page.tsx        PM 视图
│   ├── business/page.tsx       领导视图
│   └── layout.tsx
├── app/api/
│   ├── events/ingest/route.ts  事件接收
│   ├── monitoring/
│   │   ├── health/route.ts
│   │   ├── funnel/route.ts
│   │   ├── channels/route.ts
│   │   └── errors/route.ts
│   └── cron/                   定时任务端点（见 4.1）
│       ├── flush-events/route.ts
│       ├── aggregate/route.ts
│       ├── push-datadog/route.ts
│       └── check-accounts/route.ts
├── components/charts/          ECharts 封装组件
├── lib/
│   ├── clickhouse.ts
│   ├── redis.ts
│   └── postgres.ts
└── types/
    └── transaction.ts          与主站共享的类型定义
```

**认证方案：NextAuth.js**

独立项目需要自己的认证，不依赖现有 Admin 的 session。
推荐使用 NextAuth.js 对接公司现有 SSO（Google Workspace / Okta 等），
配置角色映射（engineering / product / business）控制页面访问权限。

**图表库：ECharts（via echarts-for-react）**

选择理由：
- 多系列折线图、热力图等监控场景性能优于 Recharts
- 多市场数据叠加展示时配置更灵活
- 大数据量渲染（如 90 天漏斗趋势）基于 Canvas，不会卡顿

**实时数据：轮询（Polling），不用 WebSocket**

- 健康状态 30 秒轮询，直接读 Redis 缓存，响应 < 10ms
- 业务分析数据（漏斗、渠道）按需查询，不轮询
- 第一版先跑通，后续有需要再考虑 SSE 或 WebSocket

---

### 3.6 告警层

**复用 Datadog，不重新建设**

理由：
- Datadog 已有，基础设施成本为零
- Datadog Monitors 支持自定义指标告警，配置灵活
- 团队已有 Datadog 的使用经验

**实现方式：**
- 后端将聚合指标定期推送到 Datadog Custom Metrics
- 在 Datadog 上配置 Monitor 规则（阈值、持续时间、恢复条件）
- 告警通知通过 Datadog → Slack 集成发送到对应频道

**告警频道设计：**
```
#alert-transaction-critical   → 工程 on-call（故障级别）
#alert-transaction-warning    → 工程 + PM（警告级别）
#alert-payment-accounts       → 工程 + 各市场运营负责人
```

**为什么不用 Sentry 告警：**
- Sentry 适合代码错误告警，不适合业务指标告警
- 把业务指标（转化率下降）和代码错误混在一起，会增加噪音
- 继续用 Sentry 做代码错误监控，与本系统互补而非替代

---

## 四、Node.js 后端架构设计

### 4.1 API Routes 结构与定时任务方案

**Next.js API Routes 的限制：**
API Routes 是无状态的请求驱动模型，进程不持久，无法直接运行 `setInterval` 或 `node-cron`。
定时任务需要由**外部调度器**定期调用对应的 API Route 端点来触发。

**定时任务实现方案：**

```
外部调度器（二选一）
  │
  ├── 方案 A：Vercel Cron Jobs（如果部署在 Vercel）
  │   vercel.json:
  │   {
  │     "crons": [
  │       { "path": "/api/cron/flush-events",   "schedule": "*/1 * * * *"  },
  │       { "path": "/api/cron/aggregate",       "schedule": "*/1 * * * *"  },
  │       { "path": "/api/cron/push-datadog",    "schedule": "*/5 * * * *"  },
  │       { "path": "/api/cron/check-accounts",  "schedule": "0 * * * *"    }
  │     ]
  │   }
  │
  └── 方案 B：外部 Cron 服务（自托管 / GitHub Actions / Render）
      定时 GET 请求调用对应端点，Header 携带 CRON_SECRET 验证身份
```

**Cron API Route 鉴权（防止被外部随意调用）：**

```typescript
// app/api/cron/flush-events/route.ts
export async function GET(request: Request) {
  const secret = request.headers.get('x-cron-secret');
  if (secret !== process.env.CRON_SECRET) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  await eventIngestionService.flushToClickHouse();
  return Response.json({ ok: true });
}
```

**各 Cron 端点职责：**

| 端点 | 频率 | 职责 |
|---|---|---|
| `/api/cron/flush-events` | 每 1 分钟 | Redis 队列 → ClickHouse 批量写入 |
| `/api/cron/aggregate` | 每 1 分钟 | ClickHouse 聚合 → Redis 健康缓存 |
| `/api/cron/push-datadog` | 每 5 分钟 | Redis 指标 → Datadog Custom Metrics |
| `/api/cron/check-accounts` | 每小时 | 支付账号成功率异常检测 |

**整体 API Routes 目录：**

```
app/api/
├── events/
│   └── ingest/route.ts          POST  事件接收入队
│
├── monitoring/
│   ├── health/route.ts          GET   读 Redis（< 10ms）
│   ├── funnel/route.ts          GET   查 ClickHouse MV
│   ├── channels/route.ts        GET   查 ClickHouse MV
│   └── errors/route.ts          GET   查 ClickHouse
│
└── cron/                        GET   由外部调度器触发，需 CRON_SECRET
    ├── flush-events/route.ts
    ├── aggregate/route.ts
    ├── push-datadog/route.ts
    └── check-accounts/route.ts
```

---

### 4.2 Event Ingestion 流程（API Route 实现）

**核心设计：接收快，写入异步**

收到事件 → 立即校验 → 写入 Redis 队列 → 返回响应 → 后台批量写入 ClickHouse

```
POST /api/monitoring/events
        │
        ▼
  Zod Schema 校验
  （格式、必填字段、market 枚举值）
        │
    校验失败 ──→ 返回 400，记录拒绝数
        │
    校验通过
        │
        ▼
  写入 Redis List（事件缓冲队列）
  key: monitoring:event_queue
        │
        ▼
  返回 202 Accepted（不等待写库）
        │
        ▼（异步，后台 Job）
  每 5 秒批量从队列取出（最多 500 条）
        │
        ▼
  批量写入 ClickHouse
  INSERT INTO transaction_events (...)
```

**为什么用 Redis List 而不是直接写 ClickHouse：**
- ClickHouse 对批量插入友好，单条插入性能差
- Redis 队列天然支持批量消费，解耦接收和写入
- 即使 ClickHouse 临时不可用，事件不丢失（Redis 持久化兜底）

**实现关键点：**

```typescript
// EventIngestionService 核心逻辑
class EventIngestionService {
  async ingest(events: TransactionEvent[]): Promise<IngestResult> {
    const { valid, invalid } = this.validate(events);

    if (valid.length > 0) {
      // 批量写入 Redis 队列，pipeline 减少 RTT
      const pipeline = this.redis.pipeline();
      valid.forEach(e => pipeline.rpush('monitoring:event_queue', JSON.stringify(e)));
      await pipeline.exec();
    }

    return { accepted: valid.length, rejected: invalid.length };
  }

  // 后台批量消费，由定时 Job 调用
  async flushToClickHouse(): Promise<void> {
    const raw = await this.redis.lrange('monitoring:event_queue', 0, 499);
    if (raw.length === 0) return;

    const events = raw.map(r => JSON.parse(r));
    await this.clickhouse.insert('transaction_events', events);

    // 只有写入成功才删除队列数据
    await this.redis.ltrim('monitoring:event_queue', raw.length, -1);
  }
}
```

---

### 4.3 ClickHouse 数据模型

**主表：transaction_events**

```sql
CREATE TABLE transaction_events (
  transaction_id    String,
  event_type        LowCardinality(String),
  timestamp         DateTime64(3, 'UTC'),
  market            LowCardinality(String),
  channel           LowCardinality(String),
  session_id        String,
  payment_method    LowCardinality(Nullable(String)),
  failure_reason    LowCardinality(Nullable(String)),
  metadata          String,                    -- JSON 字符串，扩展字段
  server_order_id   Nullable(String),          -- 后端绑定的订单 ID
  ingested_at       DateTime DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY (toYYYYMM(timestamp), market)     -- 按月 + 市场分区，查询时裁剪分区
ORDER BY (market, transaction_id, timestamp);  -- 排序键，按 market 过滤最快
```

**Materialized View：漏斗小时聚合**

```sql
-- 预聚合漏斗数据，避免每次查询全表扫描
CREATE MATERIALIZED VIEW funnel_hourly_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (market, hour, event_type)
AS SELECT
  market,
  toStartOfHour(timestamp)                          AS hour,
  event_type,
  count()                                           AS total,
  countIf(failure_reason IS NOT NULL)               AS failed,
  countIf(failure_reason = 'api_timeout')           AS failed_api_timeout,
  countIf(failure_reason = 'sdk_error')             AS failed_sdk_error,
  countIf(failure_reason = 'payment_declined')      AS failed_payment_declined
FROM transaction_events
GROUP BY market, hour, event_type;
```

**Materialized View：渠道小时聚合**

```sql
CREATE MATERIALIZED VIEW channel_hourly_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (market, hour, payment_method)
AS SELECT
  market,
  toStartOfHour(timestamp)  AS hour,
  payment_method,
  countIf(event_type = 'payment_selected')    AS selected_count,
  countIf(event_type = 'payment_completed')   AS completed_count,
  countIf(event_type = 'payment_failed')      AS failed_count,
  -- 支付耗时（submitted → completed 的时间差，需要 JOIN，单独计算）
  avg(if(event_type = 'payment_completed', toUnixTimestamp64Milli(timestamp), 0)) AS avg_completion_ts
FROM transaction_events
WHERE payment_method IS NOT NULL
GROUP BY market, hour, payment_method;
```

**索引策略说明：**
- `PARTITION BY (toYYYYMM(timestamp), market)`：按月 + 市场分区，时间范围查询 + 市场过滤可裁剪 95% 的数据
- `ORDER BY (market, transaction_id, timestamp)`：market 放第一位，单市场查询最快
- Materialized View 预聚合：漏斗和渠道的时间段查询直接走 MV，不扫主表

---

### 4.4 Redis 使用策略

```
Key                               用途                TTL
─────────────────────────────────────────────────────────
monitoring:event_queue            事件缓冲队列          无（消费后删除）
monitoring:health:global          全局健康状态          35s
monitoring:health:{market}        各市场健康状态        35s
monitoring:alert:active           当前活跃告警列表      60s
monitoring:channel_baseline:{id}  渠道成功率 7 日基线   24h（每日更新）
```

**健康状态的读写分离：**
- 写：`healthAggregator` Job 每 30 秒查 ClickHouse，结果写入 Redis
- 读：Dashboard API 直接读 Redis，不查 ClickHouse
- 这样健康总览页面的响应时间 < 10ms，且不给 ClickHouse 增加高频查询压力

---

### 4.5 PostgreSQL 配置表

```sql
-- 市场配置
CREATE TABLE markets (
  code          VARCHAR(10) PRIMARY KEY,   -- 'DE', 'JP', 'US'
  name          VARCHAR(100),
  timezone      VARCHAR(50),              -- 'Europe/Berlin'
  currency      VARCHAR(10),
  alert_contact VARCHAR(255),            -- 对应市场的 Slack 告警接收人
  active        BOOLEAN DEFAULT true
);

-- 支付渠道配置
CREATE TABLE payment_channels (
  id            SERIAL PRIMARY KEY,
  market_code   VARCHAR(10) REFERENCES markets(code),
  method_id     VARCHAR(50),             -- 'stripe', 'paypal', 'klarna'
  display_name  VARCHAR(100),
  active        BOOLEAN DEFAULT true
);

-- 告警规则
CREATE TABLE alert_rules (
  id                SERIAL PRIMARY KEY,
  name              VARCHAR(100),
  metric            VARCHAR(100),        -- 'conversion_rate', 'failure_rate'
  market_code       VARCHAR(10),         -- NULL 表示全局规则
  condition         VARCHAR(20),         -- 'lt'（小于）| 'gt'（大于）
  threshold         DECIMAL,
  duration_minutes  INTEGER,             -- 持续多少分钟才触发
  severity          VARCHAR(20),         -- 'critical' | 'warning'
  slack_channel     VARCHAR(100),
  active            BOOLEAN DEFAULT true
);

-- 告警历史
CREATE TABLE alert_history (
  id            SERIAL PRIMARY KEY,
  rule_id       INTEGER REFERENCES alert_rules(id),
  triggered_at  TIMESTAMPTZ,
  resolved_at   TIMESTAMPTZ,
  metric_value  DECIMAL,
  notified      BOOLEAN DEFAULT false
);
```

---

### 4.6 定时任务设计

```typescript
// app/api/cron/aggregate/route.ts
// 由外部调度器每 1 分钟触发

export async function GET(request: Request) {
  if (request.headers.get('x-cron-secret') !== process.env.CRON_SECRET) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const markets = await postgresClient.getActiveMarkets();

  for (const market of markets) {
  const markets = await postgresClient.getActiveMarkets();

  for (const market of markets) {
    const metrics = await clickhouse.query(`
      SELECT
        countIf(event_type = 'checkout_initiated')                    AS initiated,
        countIf(event_type = 'payment_completed')                     AS completed,
        countIf(failure_reason IS NOT NULL)                           AS failed,
        completed / initiated                                         AS conversion_rate,
        failed / nullIf(initiated, 0)                                 AS failure_rate
      FROM transaction_events
      WHERE market = '${market.code}'
        AND timestamp >= now() - INTERVAL 5 MINUTE
    `);

    const status = evaluateHealth(metrics, market);
    await redis.setex(
      `monitoring:health:${market.code}`,
      90,                            // TTL 90s，1 分钟触发一次，留足余量
      JSON.stringify({ ...metrics, status, updated_at: Date.now() })
    );
  }

  return Response.json({ ok: true });
}
```

```typescript
// app/api/cron/flush-events/route.ts
// 由外部调度器每 1 分钟触发，Redis 队列 → ClickHouse 批量写入

export async function GET(request: Request) {
  if (request.headers.get('x-cron-secret') !== process.env.CRON_SECRET) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // 同时处理事件写入和告警评估
  await Promise.all([
    eventIngestionService.flushToClickHouse(),
    alertEvaluatorService.evaluate(),
  ]);

  return Response.json({ ok: true });
}
```

```typescript
// app/api/cron/push-datadog/route.ts
// 由外部调度器每 5 分钟触发

export async function GET(request: Request) {
  if (request.headers.get('x-cron-secret') !== process.env.CRON_SECRET) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const markets = await postgresClient.getActiveMarkets();

  for (const market of markets) {
    const data = JSON.parse(await redis.get(`monitoring:health:${market.code}`));
    if (!data) continue;

    await datadog.gauge('transaction.conversion_rate', data.conversion_rate, [`market:${market.code}`]);
    await datadog.gauge('transaction.failure_rate', data.failure_rate, [`market:${market.code}`]);
  }

  return Response.json({ ok: true });
}
```

---

### 4.7 告警触发流程

```
AlertService.trigger(rule, metricValue)
        │
        ▼
  检查是否已有同一 rule 的活跃告警（防重复触发）
        │
  已有活跃告警 ──→ 跳过，不重复通知
        │
  无活跃告警
        │
        ▼
  写入 alert_history（triggered_at = now()）
        │
        ▼
  发送 Slack Webhook（按 rule.slack_channel）
  消息格式：
  ┌─────────────────────────────────┐
  │ [CRITICAL] DE 市场交易失败率异常  │
  │ 当前值：8.3%  阈值：5%           │
  │ 持续时间：3 分钟                 │
  │ 查看详情：{admin_url}/monitoring │
  └─────────────────────────────────┘
        │
        ▼
  推送 Datadog Event（用于告警历史记录和 PagerDuty 集成）
```

---

## 六、支付账号健康检测方案

**数据来源策略：两阶段**

**第一阶段（v1）：从支付 SDK 回调数据推断**

不依赖各支付平台 API 权限，直接从已有的 SDK 错误和成功回调中计算各账号的成功率。

优势：不需要协调各市场账号权限，立即可做。
局限：只能检测"支付失败率上升"，无法检测"账号被限制但尚未产生失败"的情况。

**第二阶段（v2，视优先级）：轮询支付平台 API**

需要各市场分管人员提供 API Key，统一存储到密钥管理系统（如 AWS Secrets Manager）。
后端定时任务（每小时）拉取各账号状态，写入 PostgreSQL。

**异常判断逻辑（第一阶段）：**
```
基线：过去 7 天同一账号的日均成功率
告警：当前小时成功率 < 基线 × 0.85（即下降超过 15%）
且：该账号本小时交易量 > N 笔（排除小样本误报）
```

---

## 七、实现分阶段

### Phase 1（第 1-6 周）：数据基础

**目标：** Transaction ID 流转，事件数据开始采集

- [ ] 前端：Transaction Tracking Module 开发
- [ ] 前端：Event Schema 定义，与后端对齐
- [ ] 后端：Event Ingestion API 实现
- [ ] 后端：ClickHouse 建表，事件写入验证
- [ ] 验证：用 SQL 查询确认数据准确性

**这一阶段结束时，你应该能回答：** 过去 24 小时，各市场各步骤的交易事件数量是多少？

---

### Phase 2（第 7-10 周）：核心 Dashboard

**目标：** 三个核心视图可用

- [ ] 后端：Dashboard API 实现（健康状态、漏斗、渠道）
- [ ] 前端：Dashboard 框架搭建（Next.js + 路由）
- [ ] 前端：健康总览页面
- [ ] 前端：漏斗分析页面
- [ ] 前端：渠道表现页面
- [ ] 告警：Datadog Monitors 配置（先配置 Critical 级别）

**这一阶段结束时，你应该能回答：** 上周 DE 市场的转化率是多少？主要流失在哪一步？

---

### Phase 3（第 11-14 周）：完善与扩展

**目标：** 错误追踪 + 账号健康 + 告警完善

- [ ] 前端：错误追踪页面
- [ ] 后端：错误影响量化（关联交易金额）
- [ ] 后端：支付账号健康检测（第一阶段，基于 SDK 数据）
- [ ] 前端：账号健康页面
- [ ] 告警：Warning 级别告警配置，Slack 频道设计

**这一阶段结束时：** 系统基本完整，可以向 PM 和领导演示。

---

## 八、关键技术风险

| 风险 | 可能性 | 影响 | 应对 |
|---|---|---|---|
| 追踪模块影响结算页性能 | 中 | 高 | 异步上报 + 批量发送，主流程不等待 |
| transaction_id 重复计算 | 中 | 中 | 后端做幂等处理，以 transaction_id 去重 |
| 支付 SDK 回调未能稳定捕获 | 中 | 高 | 做兜底：超时未收到回调则标记为 unknown |
| 多市场时区导致数据聚合错误 | 高 | 中 | 所有事件统一存 UTC，展示时按市场时区转换 |
| ClickHouse 冷启动学习成本 | 中 | 低 | 后端决策，前端不承担 |
| 各支付平台 SDK 版本不一致 | 低 | 中 | Event Schema 设计扩展字段，兼容不同 SDK |

---

## 九、不在技术方案中的东西

- 不引入 Kafka / Kinesis：v1 数据量不需要，增加不必要的运维复杂度
- 不做实时流计算（Flink / Spark Streaming）：轮询 + 定时聚合足够
- 不做多租户权限系统：URL 路由区分用户视图，v1 内部工具不需要复杂权限
- 不接入 ML 异常检测：规则-based 阈值告警先跑，积累数据后再考虑
