# exchange-truthfulness

CEX 交易量真实性排名 — 用链上数据检测各交易所刷量信号

## 功能

调用 `/exchange-truthfulness` 后，Claude 会：

1. 从链上数据获取各 CEX 的真实储备（BTC + ETH + USDT + USDC）
2. 获取 Hyperliquid 作为基准（链上永续合约 DEX，造假成本最高）
3. 计算各所的 **OI/储备比率**，与 Hyperliquid 基准对比
4. 输出带信号灯评级的排名表 + 逐所定性分析

**覆盖交易所**: Binance、OKX、Bybit、Bitget、Gate.io、MEXC、KuCoin、HTX

**信号分级**:
- 🟢 正常：≤ 基准 × 1.0
- 🟡 关注：基准 × 1.0 ~ 1.5
- 🟠 可疑：基准 × 1.5 ~ 2.5
- 🔴 异常：> 基准 × 2.5

## 前置要求

**必须安装 Antseer MCP 服务器**，它提供以下链上数据工具：
- `ant_fund_flow` — CEX 链上储备追踪
- `ant_futures_market_structure` — 合约 OI 和交易量数据
- `ant_perp_dex` — DEX 永续合约数据
- `ant_protocol_tvl_yields_revenue` — DeFi 协议 TVL 数据

## 安装步骤

### 第 1 步：安装 Antseer MCP

按照 Antseer 官方文档安装并配置 MCP 服务器，确保在 Claude Code 中能看到 `ant_fund_flow` 等工具。

### 第 2 步：克隆 skill

将此目录复制到你的 Claude Code skills 目录：

```bash
cp -r exchange-truthfulness ~/.claude/skills/
```

### 第 3 步：运行配置脚本

```bash
cd ~/.claude/skills/exchange-truthfulness
chmod +x setup.sh
./setup.sh
```

脚本会自动检测你的 Antseer MCP UUID 并写入 `SKILL.md`。
如果自动检测失败，按提示手动粘贴 UUID 即可。

> **如何手动找 UUID**：在 Claude Code 中输入 `/tools`，找到工具名
> `mcp__<UUID>__ant_fund_flow`，复制 `<UUID>` 部分。

### 第 4 步：使用

```
/exchange-truthfulness
```

## 输出示例

```
══════════════════════════════════════════════════════════════
  CEX 交易量真实性排名  |  2026-03-30  |  基准: Hyperliquid 0.607x
══════════════════════════════════════════════════════════════

📊 基准数据
  Hyperliquid  TVL: $592M  |  BTC OI: $359M  |  OI/TVL: 0.607x

📋 排名（按 OI/储备 从低到高）

  排名  交易所      储备(B)   OI(B)    OI/储备   信号
  ──────────────────────────────────────────────────
  1.   Binance     $26.6    $6.8     0.256x    🟢
  2.   OKX        $8.5     $3.4     0.400x    🟢
  3.   Bybit       $5.5     $3.2     0.583x    🟢
  ...
```

## 方法论说明

- **比率** = 24h 合约 OI / 核心储备（BTC + ETH + USDT + USDC）
- **储备取值**：链上追踪数据（对交易所最宽容估算）
- **基准**：Hyperliquid 链上永续合约 DEX（每笔交易需真实保证金）

## 免责声明

比率偏高不等于确认刷量，可能源于做市激励、平台自营、超额储备等合理因素。本 skill 输出仅供参考，不构成投资建议。
