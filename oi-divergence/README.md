# oi-divergence

OI 背离崩盘预警 — 用合约数据检测主力高位平多的做空信号

## 功能

调用 `/oi-divergence BTC` 后，Claude 会：

1. 拉取最近 24 根 1H K 线的价格、OI、资金费率数据
2. 分析价格与 OI 是否出现背离（价格涨 + OI 降）
3. 评估背离强度（强/中/弱）
4. 如果信号成立，推荐做空入场区间、止损位、目标位

**理论来源**: @wuk_Bitcoin 的典型崩盘模型
- 价格被托着上涨，但合约持仓量连续下降
- 主力在高位平掉多单，托住价格找对手盘
- 散户/电子盘进来做多后，主力叠加空单砸盘

**背离强度分级**:
- **强**: 连续 6+ 根 K 线背离，OI 降幅 > 5%
- **中**: 连续 3-5 根 K 线背离，OI 降幅 2-5%
- **弱**: 连续 2-3 根 K 线背离，OI 降幅 < 2%

## 前置要求

**必须安装 Antseer MCP 服务器**，它提供以下数据工具：
- `ant_futures_market_structure` — 合约 OI、价格历史、资金费率数据

## 安装步骤

### 第 1 步：安装 Antseer MCP

按照 Antseer 官方文档：https://antseer.ai/ 安装并配置 MCP 服务器，确保在 Claude Code 中能看到 `ant_futures_market_structure` 工具。

### 第 2 步：克隆 skill

将此目录复制到你的 Claude Code skills 目录：

```bash
cp -r oi-divergence ~/.claude/skills/
```

### 第 3 步：运行配置脚本

```bash
cd ~/.claude/skills/oi-divergence
chmod +x setup.sh
./setup.sh
```

脚本会自动检测你的 Antseer MCP UUID 并写入 `SKILL.md`。
如果自动检测失败，按提示手动粘贴 UUID 即可。

> **如何手动找 UUID**：在 Claude Code 中输入 `/tools`，找到工具名
> `mcp__<UUID>__ant_futures_market_structure`，复制 `<UUID>` 部分。

### 第 4 步：使用

```
/oi-divergence BTC
/oi-divergence ETH
/oi-divergence SIREN
```

## 输出示例

```
## BTC OI 背离分析报告

**时间范围**: 最近 24 根 1H K 线（UTC 时间 08:00 ~ 08:00）

### 数据概览

| 指标 | 起始值 | 当前值 | 变化 | 方向 |
|------|--------|--------|------|------|
| 价格 | $84,500 | $85,200 | +0.8% | 上涨 |
| OI | $49.3B | $46.8B | -5.1% | 下降 |
| 资金费率 | 0.015% | 0.012% | — | 偏高 |

### 背离判定

**信号**: YES
**背离强度**: 强
**连续背离K线数**: 8 根
**背离区间**: 最近 8 小时

### 做空建议

- 入场区间: $85,000 - $85,500
- 止损位: $86,200（最近高点上方 1.2%）
- 目标位 1: $83,800（24H 低点）
- 目标位 2: $82,500（背离起始价格）
- 盈亏比: 2.3:1
```

## 免责声明

OI 背离不等于确认将崩盘，可能源于自然平仓、对冲调整等合理因素。本 skill 输出仅供学习参考，不构成投资建议。
