---
name: token-resilience-weekly
version: "1.0.0"
user-invocable: true
description: |
  代币韧性周报：输入一组代币，对比 ETH/SOL 基准，计算过去一周每日涨跌幅，
  输出韧性排名（T1 抗跌超额 / T2 温和抗跌 / 弱势）。
  用 CoinGecko 每日收盘价量化"大盘跌时跌得少、涨时涨得多"的韧性指标。
  需要 Antseer MCP 服务器（见 README.md 安装说明）。
  Use when asked to "韧性周报", "抗跌排名", "代币韧性", "哪些币抗跌",
  "resilience report", "token resilience", "weekly resilience",
  "跑赢大盘的币", "本周最强代币", "哪些币逆势上涨"
argument-hint: "TAO,KAITO,HYPE,IO,SPEC,STORY,DRIFT"
allowed-tools:
  - mcp__ANTSEER_MCP_ID__ant_spot_market_structure
metadata:
  requires:
    mcpServers:
      - name: antseer
        description: "Antseer on-chain data MCP — provides ant_spot_market_structure for historical price data"
author: mike
license: MIT
---

# 代币韧性周报

对比一组代币与大盘基准（ETH、SOL）过去一周的每日涨跌幅，
量化哪些代币在大盘承压时展现超额抗跌能力，输出韧性排名。

**方法论来源**: Eric @Guolier8 的代币韧性周报
- 韧性定义：大盘反弹时涨得更多，回调时跌得更少
- 数据源：CoinGecko 每日收盘价
- 核心指标：周均日涨跌幅 vs 基准均值

## 输入

`$ARGUMENTS` = 逗号分隔的代币列表（CoinGecko coin_id 或常见 ticker）。

例如：`TAO,KAITO,HYPE,IO,SPEC,STORY,DRIFT`

如果用户没有提供代币列表，询问他们想分析哪些代币。

### 参数解析

1. 将输入按逗号分割为代币列表
2. 基准币种固定为 ethereum 和 solana（用户可通过对话修改）
3. 默认回看 7 天，截止日期为今天

### Ticker 到 coin_id 映射

用户通常输入 ticker（如 TAO、HYPE），但 MCP 需要 CoinGecko 的 coin_id。
对于不确定的 ticker，用 `ant_spot_market_structure` 的 `search` query_type
搜索匹配的 coin_id。

常见映射参考：
- BTC → bitcoin, ETH → ethereum, SOL → solana
- TAO → bittensor, HYPE → hyperliquid, KAITO → kaito
- IO → io-net, SPEC → spectral, DRIFT → drift-protocol
- STORY → story-protocol, AAVE → aave, UNI → uniswap
- LINK → chainlink, ARB → arbitrum, OP → optimism

对于不在上面列表中的 ticker，先调用 search 确认 coin_id 再继续。

## 步骤

### Step 1: 批量拉取历史价格

对代币列表 + 基准（ethereum, solana）中的每个 coin，调用：

- **工具**: `ant_spot_market_structure`
- **query_type**: `coin_market_chart`
- **参数**: coin_id={coin}, days="8", vs_currency="usd"

拉 8 天是因为计算 7 天涨跌幅需要 8 个收盘价数据点（day0 作为 day1 的前日基准）。

各代币的调用相互独立，尽量并行。

**数据提取**: 从返回的 `prices` 数组中提取每日收盘价。
`prices` 是 `[[timestamp_ms, price], ...]` 格式的数组。
将 timestamp 转为日期（UTC），每天取最后一个数据点作为收盘价。

### Step 2: 计算每日涨跌幅矩阵

对每个代币，用相邻两天的收盘价计算涨跌幅：

```
daily_change_pct = (close_today - close_yesterday) / close_yesterday * 100
```

构建矩阵：行 = 日期，列 = 代币，值 = 涨跌幅%（保留两位小数）。

### Step 3: 计算周均涨跌幅

对每个代币取日涨跌幅的算术平均值：

```
avg_daily_change = sum(daily_changes) / count(days)
```

同时计算基准均值：
```
benchmark_avg = (avg_ETH + avg_SOL) / 2
```

### Step 4: 韧性评分与分级

**韧性得分**：
```
resilience_score = avg_daily_change_token - benchmark_avg
```
正值 = 跑赢大盘，负值 = 跑输大盘。

**分级规则**（依次判断）：
- **T1** (🟢)：周均涨跌幅 > 0%（大盘跌的周期还能正收益）
- **T2** (🟡)：周均涨跌幅 < 0%，但 > benchmark_avg × 0.5（跌幅明显小于大盘）
- **弱势** (🔴)：周均涨跌幅 < benchmark_avg × 2（跌幅远超大盘）
- **中性** (⚪)：其余

**辅助维度**（丰富报告，帮助用户理解）：
- 单日最大涨幅及日期
- 单日最大跌幅及日期
- 正收益天数 / 总天数

按 resilience_score 从高到低排序。

### Step 5: 输出周报

用以下格式输出完整报告（中文）：

```
📊 代币韧性周报 | {起始日期}–{结束日期}

━━━ 基准表现 ━━━
- ETH 周均日变动: {avg}%
- SOL 周均日变动: {avg}%
- 基准均值: {benchmark_avg}%

━━━ 日涨跌幅矩阵 ━━━
| 日期 | ETH | SOL | {代币1} | {代币2} | ... |
|------|-----|-----|---------|---------|-----|
| {MM/DD} | {%} | {%} | {%} | {%} | ... |
| ...  |     |     |         |         |     |
| **周均** | {%} | {%} | {%} | {%} | ... |

━━━ 韧性排名 ━━━
| 排名 | 代币 | 周均涨跌幅 | 韧性得分 | 最大涨 | 最大跌 | 正收益天数 | 分级 |
|------|------|-----------|---------|--------|--------|-----------|------|
| 1 | {coin} | {avg}% | {score} | {max_up}% | {max_down}% | {n}/{total} | T1 🟢 |
| ... |

━━━ 分析摘要 ━━━

**韧性 T1** (本周跑赢大盘):
- {coin}: {一句话点评，基于数据特征}

**韧性 T2** (温和抗跌):
- {coin}: {一句话点评}

**弱势警示**:
- {coin}: {一句话风险提示}

━━━ 下周观察 ━━━
{基于本周韧性排名，提出 2-3 个下周值得关注的方向}

---
方法论来源: Eric @Guolier8 | 数据: CoinGecko 每日收盘价
本报告不构成投资建议。韧性是回顾性指标，不代表未来走势。
```

**点评写作原则**：
- 基于数据说话：引用具体数字（如"3/23 单日反弹 +6.78%"）
- 避免主观预测：描述已发生的事实，不预测未来
- 简洁有力：每个代币一句话，不超过 30 字

## 边界

- 这个 skill 只做回顾性的韧性排名，不预测未来
- 不提供买卖建议
- 数据粒度为日级，无法捕捉盘中波动
- 部分新币/小币在 CoinGecko 可能无数据，遇到时跳过并在报告中注明
- 分级阈值是固定规则，市场极端情况下（如全面牛市/全面暴跌）分级意义会减弱
