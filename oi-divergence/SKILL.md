---
name: oi-divergence
version: "1.0.0"
user-invocable: true
description: |
  OI 背离崩盘预警。输入币种，分析 1H 级别价格与未平仓合约量(OI)的背离信号，
  当价格上涨但 OI 持续下降时触发做空预警，推荐入场/止损/目标点位。
  需要 Antseer MCP 服务器（见 README.md 安装说明）。
  Use when asked to "看OI背离", "做空信号", "崩盘预警", "OI divergence", "short signal"
argument-hint: "BTC / ETH / SOL / SIREN"
allowed-tools:
  - mcp__ANTSEER_MCP_ID__ant_futures_market_structure
metadata:
  requires:
    mcpServers:
      - name: antseer
        description: "Antseer on-chain data MCP — provides ant_futures_market_structure for OI, price, and funding rate data"
author: mike
license: MIT
---

# OI 背离崩盘预警

基于崩盘模型：价格上涨 + OI 持续下降 = 主力高位平多，准备砸盘。

**理论来源**: @wuk_Bitcoin 的典型崩盘模型
1. 价格被托着往上走，合约持仓量连续降低 — 崩盘前的预谋行为
2. 主力高位平多，托住价格找对手盘，让散户/电子盘进来做多
3. 有了对手盘后悄悄叠加空单，布局完成后撤掉托价订单砸盘

## 输入

`$ARGUMENTS` = 币种符号（如 BTC、ETH、SOL、SIREN），大小写均可。

将输入转为大写作为 symbol 参数。

## 步骤

### 1. 并行拉取数据

同时调用以下 3 个查询（全部使用 `ant_futures_market_structure` 工具）：

**查询 A — OI 数据：**
- query_type: `futures_oi_aggregated`
- symbol: $COIN（用户输入的币种，大写）
- interval: `1h`
- limit: `24`

**查询 B — 价格数据：**
- query_type: `futures_price_history`
- symbol: $COIN
- exchange: `binance`
- interval: `1h`
- limit: `24`

**查询 C — 资金费率：**
- query_type: `futures_funding_rate_oi_weight`
- symbol: $COIN
- interval: `1h`
- limit: `24`

如果查询 B 从 binance 拿不到数据，依次尝试 bybit、okx。

### 2. 数据对齐

将 OI 和价格数据按时间戳对齐。只分析两者都有数据的时间段。

### 3. 背离分析

重点看最近 6-12 根 1H K 线：

**价格趋势判断：**
- 计算价格 close 的变化方向
- 是否在走高（higher highs / higher closes）

**OI 趋势判断：**
- 计算 OI close 的变化方向
- 是否在持续下降

**背离判定：**
- 价格上涨 + OI 下降 = 背离成立（做空信号）
- 价格下跌 + OI 下降 = 正常平仓，不算背离
- 价格上涨 + OI 上涨 = 多头加仓，不算背离

**背离强度评级：**
- **强**: 连续 6 根以上 K 线背离，OI 降幅 > 5%
- **中**: 连续 3-5 根 K 线背离，OI 降幅 2-5%
- **弱**: 连续 2-3 根 K 线背离，OI 降幅 < 2%

**辅助确认：**
- 资金费率偏高（> 0.01%）进一步确认多头拥挤
- 资金费率持续为正但 OI 在降 = 主力在平多

### 4. 做空点位推荐

仅在背离成立时给出：

- **入场区间**: 当前价格附近，或最近高点回踩位
- **止损位**: OI 开始下降时对应的价格高点上方 1-2%
- **目标位 1**: 最近 24H 价格低点
- **目标位 2**: OI 下降起始点对应的价格位

### 5. 输出格式

用中文输出，格式如下：

```
## $COIN OI 背离分析报告

**时间范围**: 最近 24 根 1H K 线（UTC 时间 XX:XX ~ XX:XX）

### 数据概览

| 指标 | 起始值 | 当前值 | 变化 | 方向 |
|------|--------|--------|------|------|
| 价格 | $XX | $XX | +X% | 上涨 |
| OI | $XX | $XX | -X% | 下降 |
| 资金费率 | X% | X% | — | — |

### 背离判定

**信号**: YES / NO
**背离强度**: 强 / 中 / 弱
**连续背离K线数**: X 根
**背离区间**: 最近 X 小时

### 详细数据（最近 12 根 K 线）

| 时间 | 价格 Close | 价格变化 | OI Close | OI 变化 | 背离 |
|------|-----------|---------|---------|--------|------|
| ... | ... | ... | ... | ... | Y/N |

### 做空建议（仅背离成立时显示）

- 入场区间: $XX - $XX
- 止损位: $XX（最近高点上方 X%）
- 目标位 1: $XX（24H 低点）
- 目标位 2: $XX（背离起始价格）
- 盈亏比: X:1

### 风险提示

- 此分析基于 1H 级别数据，短周期信号可能被更大级别趋势覆盖
- OI 下降不一定代表主力行为，也可能是自然平仓
- 建议结合更大级别趋势和盘口数据综合判断
- 仅供学习参考，不构成投资建议
```

如果背离不成立，仍然输出数据概览和判定结果，说明当前不满足做空条件，并简要说明原因。
