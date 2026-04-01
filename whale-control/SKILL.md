# whale-control

庄家控盘检测器 — 用链上 Holder 地址聚类分析代币是否被庄家高度控盘

## 功能

调用 `/whale-control SIREN bsc` 后，Claude 会：

1. 获取代币 Top 50 Holders 分布
2. 排除销毁地址、交易所、合约地址，筛选出 EOA 大户
3. 对 Top 5 大户查询关联钱包，做地址聚类
4. 交叉比对关联钱包与 Holder 列表，合并同一实体
5. 尝试归因最大实体身份（DWF Labs、Wintermute 等）
6. 检查合约市场联动信号（OI 趋势、链上永续大户仓位）
7. 输出控盘比例、风险等级、操盘模式分析

**方法论来源**: 余烬 (@EmberCN) 对 $SIREN 的链上调查
- 通过 Top Holder 地址聚类发现 88.5% 供应量由同一实体控制
- 庄家操纵模式: 控盘现货消除卖压 → 通过合约获利

**风险等级分级**:
- 🟢 **健康** (< 20%): 持仓分散，正常市场
- 🟡 **关注** (20-40%): 有大户但未必操纵
- 🟠 **高风险** (40-65%): 高度集中，谨慎参与
- 🔴 **庄家盘** (> 65%): 极度控盘，独角戏

## 前置要求

**必须安装 Antseer MCP 服务器**，它提供以下数据工具：
- `ant_meme` — 代币搜索与基本面数据
- `ant_token_analytics` — Holder 分布、资金流向情报
- `ant_address_profile` — 关联钱包、地址标签
- `ant_futures_market_structure` — 合约 OI 数据
- `ant_perp_dex` — 链上永续合约大户持仓

## 安装步骤

### 第 1 步：安装 Antseer MCP

按照 Antseer 官方文档安装并配置 MCP 服务器，确保在 Claude Code 中能看到 `ant_token_analytics` 工具。

### 第 2 步：克隆 skill

将此目录复制到你的 Claude Code skills 目录：

```bash
cp -r whale-control ~/.claude/skills/
```

### 第 3 步：运行配置脚本

```bash
cd ~/.claude/skills/whale-control
chmod +x setup.sh
./setup.sh
```

脚本会自动检测你的 Antseer MCP UUID 并写入 `SKILL.md`。
如果自动检测失败，按提示手动粘贴 UUID 即可。

> **如何手动找 UUID**：在 Claude Code 中输入 `/tools`，找到工具名
> `mcp__<UUID>__ant_token_analytics`，复制 `<UUID>` 部分。

### 第 4 步：使用

```
/whale-control SIREN bsc
/whale-control DEXE ethereum
/whale-control 0x997A58129890bBdA032231A52eD1ddC845fc18e1 bsc
```

## 输出示例

```
## SIREN 庄家控盘分析报告

**链**: BSC | **价格**: $0.265 | **FDV**: $193M | **24h Vol**: $16.5M | **流动性**: $5.1M

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### 控盘结论

🔴 **庄家盘** — 最大实体控盘 88.5%

| 指标 | 数值 |
|------|------|
| 最大实体控盘比 | 88.5%（52 个地址） |
| Top 3 实体控盘比 | 92.1% |
| 有效流通占比 | 7.9% |
| 疑似实体 | DWF Labs（置信度: 中） |

### Holder 分布图

████████████████████████████████████████████ 88.5% 实体#1
███ 3.6% 实体#2
██ 2.1% 交易所
█ 1.2% 销毁/锁仓
░░░ 4.6% 其他散户
```

## 免责声明

地址聚类基于链上关联推断，不等于确认同一实体。控盘比例高不一定意味着恶意操纵（可能是未解锁代币、DAO 金库、合规做市商库存）。本 skill 输出仅供学习参考，不构成投资建议。
