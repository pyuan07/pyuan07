# 短线交易 AI 辅助系统 MVP — 设计文档（全免费版）

> 核心原则：**确定性计算交给代码，情境理解交给 AI，最终下单交给人。**
> 本系统只做「报告 + 警报 + 覆盘」，**永不接券商下单**。

---

## 0. 目标与边界

| 项目 | 内容 |
|---|---|
| 标的 | 单一标的（默认 `NVDA`，可换 `QQQ`），配置文件可改 |
| 策略 | 5 分钟 ORB（开盘区间突破）+ 同时段相对成交量 RVOL ≥ 2.0 |
| 时间级别 | 5 分钟 K 线，持仓观察 30～60 分钟（不做 Tick / 剥头皮） |
| 输出 | ① 盘前报告 ② 盘中突破警报 ③ 盘后覆盘报告，全部推送 Telegram |
| 成本 | **$0**：所有数据源、LLM、推播、主机都使用免费方案 |
| 不做 | 自动下单、多标的、多策略、前端网页 |

MVP 要回答的唯一问题：

> **「规则信号」本身有没有正期望值？加上 LLM 过滤后，期望值是否更高？**

---

## 1. 全免费技术栈

| 层 | 选用（免费） | 备选 | 备注 |
|---|---|---|---|
| 盘中 K 线 | `yfinance`（5m / 1m，含盘前 `prepost=True`） | Alpaca 免费版（IEX 实时） | yfinance 为非官方接口，需重试与容错 |
| 历史回测数据 | Alpaca 免费账户历史 bars（多年 5m 数据） | yfinance（5m 只有约 60 天） | Alpaca 需注册免费 paper 账户，无需入金 |
| 新闻 | Yahoo Finance RSS + Finnhub 免费 `company-news` | Google News RSS | 免费新闻有延迟，只作「情境参考」 |
| 经济日历 | ForexFactory 公开 JSON（本周日历） | 手动维护 FOMC/CPI 日期 | 只取 USD 高影响事件 |
| 财报日 | Finnhub 免费 `calendar/earnings` | yfinance `Ticker.calendar` | |
| 交易日历 | `pandas_market_calendars`（开源） | `exchange_calendars` | 处理休市、半日市、夏令时 |
| 指标计算 | `pandas` + `numpy`（自己写，不依赖 TA 库） | | 公式见第 5 节 |
| LLM | Google Gemini API 免费层（Flash 系列） | Groq 免费层 / 本机 Ollama | 通过抽象层可随时切换；免费额度以官方当前规定为准 |
| 推播 | Telegram Bot API | Discord Webhook | 均免费 |
| 存储 | SQLite（单文件） | | |
| 排程 | `APScheduler`（常驻）或 cron | GitHub Actions cron | 见第 14 节部署 |
| 主机 | Oracle Cloud Always Free VM | 自己的电脑 / GitHub Actions | 见第 14 节 |

> 付费模型（例如 Claude API）没有免费层，但判断质量通常更好；设计上保留 `provider` 抽象，将来要换只改配置。

---

## 2. 系统架构

```
                 ┌──────────────── 排程器 (APScheduler / cron) ────────────────┐
                 │                                                            │
   08:45 ET      ▼            09:45–11:30 ET 每 5 分钟         16:30 ET         ▼
┌──────────────────┐   ┌───────────────────────────────┐   ┌──────────────────┐
│ premarket_job    │   │ intraday_job                  │   │ postmarket_job   │
│  盘前报告         │   │  1. 拉 5m K 线（只用已收完的）   │   │  覆盘报告         │
└────────┬─────────┘   │  2. 算指标                     │   └────────┬─────────┘
         │             │  3. 触发器判断 ──否──► 结束     │            │
         │             │        │是                     │            │
         │             │  4. 代码算进场/停损/目标/风报比   │            │
         │             │  5. 抓新闻 + 日历               │            │
         │             │  6. LLM 情境判断 (TAKE / SKIP)  │            │
         │             │  7. 全部写入 SQLite            │            │
         │             │  8. 推送警报卡片                │            │
         │             └──────────────┬────────────────┘            │
         ▼                            ▼                             ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ 共享模块：data / indicators / news / llm / notify / store / evaluate │
   └──────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                           Telegram ──► 人工看盘，手动下单
```

### 2.1 每日时间表

美股夏令时（EDT，约 3 月中～11 月初）与冬令时（EST）对应的 UTC+8（台北/北京）时间：

| 任务 | 美东时间 ET | UTC+8 夏令时 | UTC+8 冬令时 |
|---|---|---|---|
| 盘前报告 | 08:45 | 20:45 | 21:45 |
| 开盘 | 09:30 | 21:30 | 22:30 |
| ORB 区间完成 | 09:45 | 21:45 | 22:45 |
| 盘中监控窗口 | 09:45–11:30 | 21:45–23:30 | 22:45–00:30 |
| 盘后覆盘报告 | 16:30 | 04:30（次日） | 05:30（次日） |

程序内部**一律用 `America/New_York` 时区**计算，夏令时切换由 `zoneinfo` 自动处理，不要手写 UTC 偏移。

---

## 3. 项目目录结构

```
day-trading-ai/
├── DESIGN.md                # 本文件
├── config.yaml              # 标的、阈值、时间窗口、LLM 设置
├── .env.example             # API Key 模板（真正的 .env 不入库）
├── requirements.txt
├── main.py                  # 入口：常驻排程 or 单次执行子命令
├── tradebot/
│   ├── config.py            # 读取 config.yaml + .env
│   ├── clock.py             # 交易日判断、ET 时间、K 线是否已收完
│   ├── data.py              # yfinance / Alpaca 取数，带重试与缓存
│   ├── indicators.py        # PMH/PML、ORB、VWAP、ATR、RSI、ToD-RVOL
│   ├── trigger.py           # 触发规则 → Signal
│   ├── risk.py              # 进场区 / 停损 / 目标 / 风报比（纯代码）
│   ├── news.py              # RSS + Finnhub 新闻、经济日历、财报日
│   ├── llm/
│   │   ├── base.py          # LLMProvider 抽象
│   │   ├── gemini.py
│   │   ├── groq.py
│   │   ├── ollama.py
│   │   └── prompts.py       # Prompt 模板 + 输出 schema
│   ├── notify.py            # Telegram / Discord
│   ├── store.py             # SQLite 读写
│   ├── evaluate.py          # 覆盘：信号结果判定
│   └── reports.py           # 三种报告的文字排版
├── jobs/
│   ├── premarket.py
│   ├── intraday.py
│   └── postmarket.py
├── backtest/
│   └── orb_backtest.py      # 纯规则历史回测（不含 LLM）
├── data/
│   └── trading.db           # SQLite（不入库）
└── tests/                   # 指标与规则的单元测试
```

---

## 4. 配置文件 `config.yaml`

```yaml
symbol: NVDA
timezone: America/New_York

session:
  premarket_start: "04:00"
  open: "09:30"
  orb_minutes: 15              # ORB = 09:30–09:45（3 根 5m K 线）
  watch_start: "09:45"
  watch_end: "11:30"           # 之后不再发新信号
  bar_close_buffer_sec: 20     # K 线结束后等 20 秒再取数，避免拿到未收完的 K 线

trigger:
  rvol_min: 2.0                # 同时段相对成交量门槛
  rvol_lookback_days: 20
  require_vwap_side: true      # 做多须在 VWAP 之上，做空须在 VWAP 之下
  max_signals_per_side: 1      # 每天每个方向最多触发 1 次
  rsi_block_long_above: 85     # 极端超买不追多（可选过滤）
  rsi_block_short_below: 15

risk:
  atr_period: 14
  stop_atr_mult: 1.5           # 停损 = 进场 − 1.5×ATR(5m)，但不越过 ORB 另一端
  target_r_multiple: 2.0       # 目标 = 2R
  entry_zone_atr_mult: 0.15    # 进场区 = 突破价 ~ 突破价 + 0.15×ATR
  signal_valid_minutes: 10     # 超过 10 分钟未进场 → 信号作废
  slippage_pct: 0.02           # 覆盘时假设滑价 0.02%

evaluate:
  horizons_min: [30, 60]       # 观察 30 与 60 分钟
  same_bar_rule: stop_first    # 同一根 1m K 线同时碰到停损和目标 → 算停损

news:
  lookback_minutes: 60
  max_headlines: 10

llm:
  provider: gemini             # gemini | groq | ollama
  model: ""                    # 留空用 provider 默认的免费模型
  temperature: 0
  timeout_sec: 20
  max_retries: 1

notify:
  channel: telegram            # telegram | discord
  heartbeat: true              # 每天开盘后推一条「系统在线」
```

`.env.example`：

```
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
FINNHUB_API_KEY=
GEMINI_API_KEY=
GROQ_API_KEY=
ALPACA_API_KEY=        # 只在回测时需要
ALPACA_API_SECRET=
DISCORD_WEBHOOK_URL=   # 选用
```

---

## 5. 数据层与指标定义（全部由代码计算）

### 5.1 取数规则

- `yfinance.download(symbol, interval="5m", period="5d", prepost=True)`，转换为 ET 时区。
- 历史同时段成交量（算 RVOL 用）：`period="60d"` 的 5m 数据，**每天开盘前抓一次并缓存**到 SQLite，盘中不重复抓。
- **只使用已收完的 K 线**：K 线开始时间 `t` 满足 `t + 5min + buffer ≤ now` 才算收完。例如 09:50:20 才读取 09:45 那根。
- 失败处理：重试 3 次（2s / 4s / 8s）；仍失败就跳过本轮并记日志；连续 3 轮失败时推送「数据源异常」。

### 5.2 指标公式

| 指标 | 定义 |
|---|---|
| 盘前高/低 PMH / PML | 当日 04:00–09:29 ET 所有 K 线的最高价 / 最低价 |
| 昨收 / 跳空 | `gap% = (今日 09:30 开盘价 − 昨收) / 昨收` |
| ORB 高/低 ORH / ORL | 09:30–09:44 三根 5m K 线的最高价 / 最低价 |
| VWAP | 从 09:30 起累计：`Σ(典型价 × 量) / Σ量`，典型价 = `(H+L+C)/3` |
| ATR(5m, 14) | 常规盘 5m K 线的 Wilder ATR，周期 14（可延用前一日资料补足前几根） |
| 日线 ATR(14) | 日 K 的 Wilder ATR，给盘前报告参考用 |
| RSI(14) | 5m 收盘价的 Wilder RSI |
| **同时段 RVOL** | `当根 5m 量 / 过去 20 个交易日同一时刻那根 5m 量的平均` |
| 累计 RVOL（参考） | `今日 09:30 至今累计量 / 过去 20 天同时段累计量的平均` |

> **为什么不用「前 20 根均量」**：美股日内量呈 U 型，开盘时段本来就放量。用前 20 根当分母，开盘后几乎每根都 > 2 倍，过滤等于没做。同时段 RVOL 才能反映「今天比平常同一时刻更活跃」。

---

## 6. 触发器（Trigger）

在 09:45–11:30 之间，每根 5m K 线收完后检查一次：

**做多信号 LONG**（全部满足）
1. 已收完 K 线 `close > ORH`
2. 同时段 RVOL ≥ `rvol_min`
3. `close > VWAP`（若 `require_vwap_side`）
4. RSI < `rsi_block_long_above`
5. 今天还没发过 LONG 信号

**做空信号 SHORT**：条件镜像（`close < ORL`、`close < VWAP`、RSI > 下限）。

**额外标注（不阻挡，只写入信号，交给 LLM 判断）**
- 是否同时突破盘前高点：`close > PMH`
- 跳空方向与突破方向是否一致
- 今天是否有高影响经济数据，或本周是否有财报

不满足条件 → 直接结束本轮，**不调用 LLM**。

---

## 7. 风控计算（`risk.py`，纯代码）

以 LONG 为例（SHORT 镜像）：

```
entry_ref   = 触发 K 线收盘价
entry_zone  = [ORH, ORH + entry_zone_atr_mult × ATR]
              （若 entry_ref 已高于上缘，进场区改为 [entry_ref, entry_ref + 0.1×ATR]）
stop_loss   = max(entry_ref − stop_atr_mult × ATR, ORL)    # 不越过 ORB 另一端
risk_per_sh = entry_ref − stop_loss                         # 1R
target      = entry_ref + target_r_multiple × risk_per_sh
rr          = (target − entry_ref) / risk_per_sh
valid_until = 触发时间 + signal_valid_minutes
invalidation= 「5 分钟 K 线收回 ORH 之下即失效」
```

合理性检查（任何一条不通过就不推送，只记日志）：
- `stop_loss < entry_ref < target`（做空则相反）
- `risk_per_sh > 0.1 × ATR`（停损太近，容易被噪音打掉）
- `risk_per_sh < 3 × ATR`（停损太远，风险不成比例）

---

## 8. 新闻与事件层（`news.py`）

触发时才抓（盘前报告也会用）：

1. **Yahoo Finance RSS**：`https://feeds.finance.yahoo.com/rss/2.0/headline?s={symbol}`
2. **Finnhub** `GET /api/v1/company-news?symbol=NVDA&from=YYYY-MM-DD&to=YYYY-MM-DD`，按 `datetime` 只保留最近 `lookback_minutes` 分钟
3. 合并后按标题相似度去重，按时间倒序，最多保留 `max_headlines` 条
4. **经济日历**：ForexFactory 本周 JSON，只保留 `country=USD` 且 `impact=High` 的事件，每天盘前抓一次并缓存
5. **财报日**：Finnhub earnings calendar，每天盘前抓一次并缓存

每条新闻只送「时间 + 来源 + 标题」给 LLM，不送全文：省 token，也降低 LLM 编造细节的机会。

---

## 9. LLM 推理层

### 9.1 LLM 的职责（刻意限缩）

| LLM 负责 | LLM **不负责** |
|---|---|
| 这次突破在当前新闻和事件背景下，比较像真突破还是假突破 | 任何价格计算（进场、停损、目标、风报比） |
| `TAKE` / `SKIP` 判断与理由 | 决定仓位大小 |
| 标出风险（例如「15 分钟后有 FOMC 纪要」「利多是旧闻」） | 预测具体价格 |

### 9.2 输入（由代码组装的 JSON）

```json
{
  "symbol": "NVDA",
  "time_et": "2026-09-28T09:50:00-04:00",
  "signal": {
    "side": "LONG",
    "close": 129.05, "orh": 128.80, "orl": 127.40,
    "pmh": 128.95, "pml": 126.90, "vwap": 128.31,
    "above_pmh": true, "gap_pct": 1.2,
    "rvol_tod": 2.6, "rvol_cum": 1.9, "rsi14": 68.4, "atr5m": 0.42
  },
  "risk_plan": {
    "entry_zone": [128.80, 128.86], "stop_loss": 127.42,
    "target": 132.31, "rr": 2.0, "valid_until_et": "10:00"
  },
  "events_today": ["10:00 ET 美国消费者信心指数 (High)"],
  "earnings_within_7d": false,
  "headlines": [
    {"time_et": "09:12", "source": "Reuters", "title": "..."}
  ]
}
```

### 9.3 输出 Schema（强制 JSON）

```json
{
  "decision": "TAKE",
  "bias": "LONG",
  "confidence": 0.65,
  "breakout_quality": "STRONG",
  "news_alignment": "SUPPORTIVE",
  "risk_flags": ["10:00 有经济数据公布，突破后可能剧烈波动"],
  "key_reason": "放量突破 ORB 与盘前高点，且盘前有正面新闻支撑。",
  "watch_for": "若 10:00 数据公布后 5m 收回 128.8 以下，视为假突破。"
}
```

字段约束（用 `pydantic` 校验）：
- `decision` ∈ {`TAKE`, `SKIP`}
- `bias` 必须等于信号方向；不一致就视为无效输出
- `confidence` ∈ [0, 1]。只用于事后分组统计，**不当作胜率**
- `breakout_quality` ∈ {`STRONG`, `MODERATE`, `WEAK`}
- `news_alignment` ∈ {`SUPPORTIVE`, `NEUTRAL`, `CONFLICTING`, `NO_NEWS`}
- `key_reason`、`watch_for` 各不超过 80 字

### 9.4 System Prompt 要点

```
你是日内交易的情境评估员。价格、停损、目标都已由程序计算好，你不得修改或重新计算任何数字。
你的唯一任务：根据提供的量价摘要、新闻标题与事件日历，判断这次突破更可能延续 (TAKE) 还是失败 (SKIP)。
- 只能使用输入中提供的信息，不得引用输入以外的新闻或数据。
- 没有相关新闻时，news_alignment 填 NO_NEWS，不要臆测。
- 只输出符合 schema 的 JSON，不要输出其他文字。
```

### 9.5 容错

- 调用超时（`timeout_sec`）或 JSON 校验失败：重试 1 次；仍失败就推送**「纯规则警报（LLM 不可用）」**，并照常写入数据库。
- Provider 抽象介面：`evaluate(payload: dict) -> LLMDecision`，依配置选 Gemini / Groq / Ollama。
- 免费层限流：每天只有 0～2 次触发，外加两份报告，远低于免费额度。

---

## 10. 推播层（Telegram）

### 10.1 建立 Bot（一次性，约 5 分钟）

1. 在 Telegram 找 `@BotFather` → `/newbot` → 取得 `TELEGRAM_BOT_TOKEN`
2. 对你的 Bot 发一句话，然后打开 `https://api.telegram.org/bot<TOKEN>/getUpdates`，找到 `chat.id` 即 `TELEGRAM_CHAT_ID`
3. 发送：`POST https://api.telegram.org/bot<TOKEN>/sendMessage`，参数 `chat_id`、`text`、`parse_mode=HTML`

### 10.2 盘中警报卡片

```
🟢 NVDA LONG 突破 ｜ AI: TAKE (0.65)
⏰ 09:50 ET ｜ 有效至 10:00 ET

进场区  128.80 – 128.86
停损    127.42   (−1R)
目标    132.31   (+2R)
风报比  2.0

📊 RVOL 2.6x ｜ RSI 68 ｜ 高于 VWAP 128.31
   突破 ORB 高 128.80 ✅ ｜ 突破盘前高 128.95 ✅

🤖 放量突破 ORB 与盘前高点，且盘前有正面新闻支撑。
⚠️ 10:00 有经济数据公布，突破后可能剧烈波动
👀 若数据公布后 5m 收回 128.8 以下，视为假突破。

#sig_20260928_01
```

- `SKIP` 的信号**照样推送**，但加上 ⚪ 标记并使用静音通知（`disable_notification=true`），方便你自行比对。
- LLM 不可用时，标题改为「🟡 纯规则警报（AI 离线）」。

---

## 11. 存储与覆盘

### 11.1 SQLite 表结构

```sql
CREATE TABLE signals (
  id            TEXT PRIMARY KEY,        -- sig_20260928_01
  symbol        TEXT NOT NULL,
  ts_et         TEXT NOT NULL,           -- 触发 K 线收盘时间
  side          TEXT NOT NULL,           -- LONG / SHORT
  close         REAL, orh REAL, orl REAL, pmh REAL, pml REAL, vwap REAL,
  rvol_tod      REAL, rvol_cum REAL, rsi14 REAL, atr5m REAL, gap_pct REAL,
  entry_low     REAL, entry_high REAL, stop_loss REAL, target REAL, rr REAL,
  valid_until   TEXT,
  config_hash   TEXT                     -- 参数版本，改参数后可分组比较
);

CREATE TABLE llm_decisions (
  signal_id     TEXT REFERENCES signals(id),
  provider      TEXT, model TEXT,
  decision      TEXT, confidence REAL,
  breakout_quality TEXT, news_alignment TEXT,
  risk_flags    TEXT,                    -- JSON 数组
  key_reason    TEXT, watch_for TEXT,
  latency_ms    INTEGER,
  raw_input     TEXT, raw_output TEXT,   -- 完整留档，方便改 prompt 后比对
  error         TEXT
);

CREATE TABLE outcomes (
  signal_id     TEXT REFERENCES signals(id),
  horizon_min   INTEGER,                 -- 30 / 60
  filled        INTEGER,                 -- 有效期内价格是否进入进场区
  fill_price    REAL,
  result        TEXT,                    -- WIN / LOSS / TIMEOUT / NO_FILL
  exit_price    REAL,
  r_multiple    REAL,                    -- 含滑价
  mfe_r         REAL, mae_r REAL,
  PRIMARY KEY (signal_id, horizon_min)
);

CREATE TABLE my_trades (                 -- 选填：你实际有没有跟单
  signal_id     TEXT REFERENCES signals(id),
  taken         INTEGER, note TEXT
);

CREATE TABLE cache (k TEXT PRIMARY KEY, v TEXT, updated_at TEXT);
```

### 11.2 结果判定规则（`evaluate.py`）

用**当日 1m K 线**（yfinance 1m 数据只保留约 7 天，所以当天收盘后就要跑）：

1. **成交**：在 `valid_until` 之前，1m K 线的价格区间碰到进场区 → 以进场区上缘成交（做空用下缘），再加上滑价。没碰到就记为 `NO_FILL`。
2. **结果**：从成交后的下一根 1m K 线开始，逐根检查：
   - 先碰到停损 → `LOSS`，记为 −1R 再扣滑价
   - 先碰到目标 → `WIN`，记为 +2R 再扣滑价
   - 同一根 K 线同时碰到停损和目标 → 按 `stop_first` 规则记为 `LOSS`
   - 到达观察时长（30 / 60 分钟）都没碰到 → `TIMEOUT`，以当时收盘价换算成 R
3. 记录这段期间的 MFE（最大有利波动）与 MAE（最大不利波动），都以 R 为单位。

### 11.3 统计口径

盘后报告与累计统计**一律分三组**：

| 组别 | 定义 | 用途 |
|---|---|---|
| A. 纯规则 | 所有触发的信号 | 规则本身的期望值 |
| B. LLM TAKE | A 中 LLM 判断 TAKE 的 | 听 AI 的结果 |
| C. LLM SKIP | A 中 LLM 判断 SKIP 的 | AI 放弃掉的信号实际表现如何 |

指标：笔数、成交率、胜率、平均 R、**期望值 = 平均 R**、最大连续亏损次数、平均 MFE / MAE。

判读方式：若 B 的期望值明显高于 A，且 C 的期望值明显较低，才表示 LLM 真的有价值。样本数少于 30 笔前，任何结论都只是参考。

---

## 12. 三份报告内容

### 12.1 盘前报告（08:45 ET）

```
📋 NVDA 盘前报告 ｜ 2026-09-28 (一)

昨收 127.60 ｜ 盘前 128.70 (+0.86%)
盘前高/低 128.95 / 126.90 ｜ 盘前量 = 20 日均值的 1.4 倍
日线 ATR 3.85（今日盘前已走 0.8 ATR）

🎯 今日关注价位（由代码计算）
  突破盘前高 128.95 ↑ ｜ 跌破盘前低 126.90 ↓
  ORB 在 09:45 ET 确定后另行推送

📅 今日事件：10:00 消费者信心指数 (High)
📅 财报：无（下次：2026-11-18）

📰 重点新闻（最多 5 条）
  ...

🤖 AI 情境摘要（3～5 句）
  ...
```

09:45 ORB 确定后，另外推一条短讯：`ORB 128.80 / 127.40，多方触发价 > 128.80，空方触发价 < 127.40`。

### 12.2 盘中警报

见第 10.2 节。

### 12.3 盘后覆盘（16:30 ET）

```
📊 NVDA 覆盘 ｜ 2026-09-28

今日信号 1 笔
  #sig_20260928_01 LONG ｜ AI: TAKE
  成交 128.86 → 30m: WIN +2.0R ｜ 60m: WIN +2.0R ｜ MFE 2.4R / MAE 0.3R

累计（最近 20 个交易日，30m 口径）
            笔数  胜率   期望值
  A 纯规则   14   43%   +0.21R
  B AI TAKE   9   56%   +0.52R
  C AI SKIP   5   20%   −0.40R
```

---

## 13. 回测（先证明规则本身有效）

`backtest/orb_backtest.py`：

- 数据：Alpaca 免费账户的历史 5m 与 1m bars（可回溯多年），`feed=iex` 或 SIP 历史数据（免费账户可查询 15 分钟以前的 SIP 历史）
- 完全复用 `indicators.py`、`trigger.py`、`risk.py`、`evaluate.py`，**确保回测与实盘使用同一套代码**
- 输出：每笔信号明细 CSV 与分年度、分月统计
- 回测**不包含 LLM**：历史新闻的时间戳容易带入未来信息，模型本身也可能「知道」历史行情的后续走势，结果不可信

**上线门槛**：纯规则回测期望值 > 0，且样本数 ≥ 100 笔，再进入实盘警报阶段。否则先调整规则。

---

## 14. 部署（全免费）

### 方案 A（推荐）：Oracle Cloud Always Free VM

- 永久免费的 ARM VM，足够跑 Python 常驻程序
- 注册时需要信用卡验证身份，但不会扣款；部分区域资源常被抢光，需要多试几次
- 部署方式：
  ```
  python -m venv .venv && pip install -r requirements.txt
  cp .env.example .env   # 填入 key
  systemd 服务：python main.py run   # APScheduler 常驻，三个任务按 ET 时间排程
  ```
- APScheduler 设定 `timezone="America/New_York"`；每个任务开始前先检查 `clock.is_trading_day()`

### 方案 B：GitHub Actions（完全不用主机）

| Workflow | cron（UTC） | 作法 |
|---|---|---|
| premarket | 夏令时、冬令时两个时间都排 | 脚本启动后检查 ET 时间，不在窗口内就直接结束 |
| intraday | ET 09:20 左右启动**一个长任务** | 任务内部自己循环到 11:35 ET 才结束（约 2 小时），每 5 分钟检查一次 |
| postmarket | ET 16:30 左右 | 覆盘后把 `trading.db` 提交到专用的 `data` 分支保存 |

注意：
- GitHub cron 常有 5～15 分钟以上的延迟，所以盘中任务要提早启动，改由程序内部控制时间。
- **公开 repo** 的 Actions 分钟数免费不限；**私有 repo** 每月只有 2000 分钟，每天约 130 分钟 × 21 天会超过额度。
- API key 放在 GitHub Secrets，repo 公开也不会外泄；但策略代码会公开。
- 公开 repo 超过 60 天没有活动，排程会被自动停用。

### 方案 C：自己的电脑

免费但要保持开机。适合开发阶段测试。

---

## 15. 容错与维运

| 情况 | 处理 |
|---|---|
| 休市 / 半日市 | `pandas_market_calendars` 判断；半日市（13:00 收盘）监控窗口与盘后报告时间自动提前 |
| 夏令时切换 | 全程使用 `zoneinfo("America/New_York")` |
| yfinance 失败 / 缺 K 线 | 重试 → 跳过本轮 → 连续 3 次失败推送告警；缺 K 线时不计算 RVOL |
| 同一信号重复推送 | `signals.id` 唯一；每天每个方向只触发 1 次 |
| 程序挂掉 | systemd `Restart=always`；开盘后推一条心跳讯息，没收到就代表系统异常 |
| 参数变更 | `config_hash` 写入每笔信号，统计时可按参数版本分组 |
| 日志 | 按天轮替的文件日志，保留 30 天 |

---

## 16. 开发里程碑与验收标准

| 阶段 | 内容 | 验收标准 |
|---|---|---|
| M1 基础 | `data`、`indicators`、`clock` 与单元测试 | 与 TradingView 图上的 VWAP / ORB / PMH 对照，误差 < 0.05% |
| M2 回测 | `trigger`、`risk`、`evaluate`、`orb_backtest` | 产出 ≥ 1 年回测报告；期望值 > 0 才继续 |
| M3 报告 | 盘前报告 + 盘后覆盘 + Telegram | 连续 5 个交易日准时收到两份报告 |
| M4 盘中 | 盘中警报（先不接 LLM） | 信号时间与价位跟回测逻辑一致，推送延迟 < 60 秒 |
| M5 AI | 接上 LLM，进入影子模式三组统计 | 所有信号都有 LLM 决策记录；失败时能自动降级为纯规则警报 |
| M6 评估 | 累积 ≥ 30 笔实盘信号 | 依 11.3 节判断 LLM 是否有价值，决定保留、调整或移除 |

---

## 17. 已知限制与风险

- **免费数据不是专业级**：yfinance 可能延迟、缺 K 线或事后修正；免费新闻比付费快讯慢，LLM 看到的「最新消息」可能已经被市场消化。
- **样本量小**：单一标的每天 0～2 笔，需要 1～3 个月才有统计意义。
- **LLM 不可回测**：只能靠实盘影子记录评估。
- **执行落差**：人工下单有 30 秒到 2 分钟的延迟，覆盘结果会比实际交易略为乐观。
- **法规**：美股保证金账户、资金低于 2.5 万美元时，要留意 PDT 规则（FINRA 近年在推动修改，以券商当前规则为准）。
- **免责**：本系统只是决策辅助工具，不构成投资建议；所有下单决定与风险由使用者自行承担。
