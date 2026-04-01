

name: whale-control
version: "1.0.0"
user-invocable: true
description: |
  庄家控盘检测器。输入代币符号/合约地址 + 链，分析 Top Holder 地址聚类，
  输出最大实体控盘比例、关联地址集群、疑似实体归因、风险等级。
  需要 Antseer MCP 服务器（见 README.md 安装说明）。
  Use when asked to "查控盘", "庄家分析", "holder分析", "whale control",
  "who controls this token", "持仓集中度", "是不是庄家盘"
argument-hint: "SIREN bsc / 0x997A...fc18e1 bsc / DEXE ethereum"
allowed-tools:





mcp__ANTSEER_MCP_ID__ant_meme



mcp__ANTSEER_MCP_ID__ant_token_analytics



mcp__ANTSEER_MCP_ID__ant_address_profile



mcp__ANTSEER_MCP_ID__ant_futures_market_structure



mcp__ANTSEER_MCP_ID__ant_perp_dex
metadata:
  requires:
mcpServers:





name: antseer
description: "Antseer on-chain data MCP — provides ant_meme, ant_token_analytics, ant_address_profile, ant_futures_market_structure, ant_perp_dex"

author: mike
license: MIT

庄家控盘检测器

用链上 Holder 数据 + 地址聚类，判断某代币是否被庄家高度控盘。

方法论来源: 余烬 (@EmberCN) 对 $SIREN 的链上调查 — 通过 Top Holder 地址聚类发现 88.5% 供应量由同一实体控制。

输入

$ARGUMENTS = 代币符号或合约地址 + 链名称（空格分隔）

解析规则:





第一个参数以 0x 开头或长度 > 30 → 合约地址，否则视为 symbol



第二个参数为链: ethereum, bsc, solana, arbitrum, base, polygon



未提供链 → 默认 ethereum，若无结果则尝试 bsc

链名称映射（重要）

不同 API 对同一条链使用不同的 chain 参数名。必须按以下映射传参:







用户输入



ant_meme (chain_id)



ant_token_analytics / ant_address_profile (chain)





bsc



bsc



bnb





ethereum



ethereum



ethereum





solana



solana



solana





arbitrum



arbitrum



arbitrum





base



base



base





polygon



polygon



polygon





avalanche



avalanche



avalanche

ant_futures_market_structure 和 ant_perp_dex 不需要 chain 参数，只需 symbol。

步骤

Phase 1: 解析输入 & 获取代币信息

1a. 如果输入是 symbol:

调用 ant_meme → search_pairs:





query: 用户输入的 symbol

从结果中匹配用户指定链，提取 baseToken.address 作为合约地址。同一 symbol 多链部署时选流动性最高的。

1b. 如果输入是合约地址: 直接使用。

1c. 获取基本面:

调用 ant_meme → token_info:





chain_id: 链名称



token_addresses: 合约地址

记录: 名称、价格、FDV、24h Vol、流动性。token_info 为空则跳过，继续后续分析。

Phase 2: 获取 Holder 分布

并行调用:

查询 A — Top 50 Holders:





ant_token_analytics → holders



token_address: 合约地址



chain: 链名称



pagination: {"page": 1, "per_page": 50}

查询 B — 资金流向情报:





ant_token_analytics → flow_intelligence



token_address: 合约地址



chain: 链名称

数据整理: 将 Top 50 分类:





销毁/零地址: 0x0000...dead 等 → 从有效供应中扣除



交易所地址: 带 exchange 标签 → 单独统计



合约地址: LP 池、锁仓、质押合约 → 单独统计



普通地址 (EOA): 聚类分析目标

Phase 3: 地址聚类分析（核心）

Step 3a — 选取目标:

从 EOA 地址按持仓量取 Top 10。若 Top 10 合计 < 15%，跳到 Phase 5 给"健康"结论。

Step 3b — 查询关联钱包:

对 Top 10 中前 5 个地址，并行调用:





ant_address_profile → related_wallets



address: 目标地址



chain: 链名称

限 5 个以控制 API 量。若前 5 个已发现 > 50% 控盘，无需查剩余。

Step 3c — 交叉比对 & 聚类（双策略）:

策略 A: 链上关联聚类（related_wallets 有结果时）:





将每个地址的 related_wallets 与 Top 50 Holders 做交集



Holder A 的关联钱包包含 Holder B → A、B 归为同一 Cluster



同源资金聚类: 如果多个 Holder 的 First Funder 为同一地址 → 归为同一 Cluster



递归合并: A→B、B→C → A、B、C 同一 Cluster



每个 Cluster 计算: 地址数、合计持仓量、占有效供应量百分比

策略 B: 行为模式聚类（related_wallets 大部分为空时的备选）:

庄家常刻意使用全新隔离地址规避链上关联。当 related_wallets 覆盖不足时（5 个查询中 ≥ 3 个返回空），启用行为模式聚类:

满足以下 ≥ 3 条 的地址群视为疑似同一实体:





零流出: total_outflow = 0（只进不出，典型归集地址）



同期建仓: balance_change_30d ≈ token_amount（过去 30 天内一次性注入）



持仓均匀: 地址间持仓量差异 < 30%（拆分归集的指纹）



同源资金: First Funder 为同一交易所热钱包或同一地址



无交互历史: total_outflow = 0 且 total_inflow 仅 1 笔

行为模式聚类置信度标注为"中"（基于行为推断，非链上直接关联）。

在报告中明确说明使用了哪种聚类策略及原因。

Step 3d — 实体归因（可选）:

对最大 Cluster 的代表地址（持仓最大的），调用:





ant_address_profile → labels



address: 代表地址



chain: 链名称



注意: 消耗 100 credits/次，最多调用 2 次

置信度:





高: labels 直接返回已知做市商/机构



中: related_wallets 出现已知机构地址



低: 仅基于地址聚类推断

Phase 4: 合约联动检测（增强）

尝试并行调用，无合约交易对则跳过:

查询 A — CEX 合约 OI:





ant_futures_market_structure → futures_oi_aggregated



symbol: 代币符号（大写）



interval: 1d



limit: 30

查询 B — 链上永续大户持仓:





ant_perp_dex → perp_dex_position_by_coin



symbol: 代币符号（大写）

两个都返回空 → 跳过，报告注明无合约市场。

Phase 5: 综合评估 & 输出

控盘比例:

有效供应量 = 总供应量 - 销毁量 - 合约锁仓量
最大实体控盘比 = 最大 Cluster 持仓 / 有效供应量
Top 3 实体控盘比 = 前三大 Cluster 持仓 / 有效供应量

风险等级:







等级



最大实体控盘比



含义





🟢 健康



< 20%



持仓分散





🟡 关注



20-40%



有大户但未必操纵





🟠 高风险



40-65%



高度集中





🔴 庄家盘



> 65%



极度控盘

升级因子（+1 级）: 合约 OI 快速增长 + 现货高控盘；流动性 < FDV 1%；近期大额单向流入
**降级因子（-1 级）:** 最大 Cluster 为合规做市商；集中持仓来自未解锁份额；交易所持仓 > 30%

输出格式

用中文输出:

## $TOKEN 庄家控盘分析报告

**链**: {chain} | **价格**: ${price} | **FDV**: ${fdv} | **24h Vol**: ${vol} | **流动性**: ${liq}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### 控盘结论

{emoji} **{等级}** — 最大实体控盘 {X}%

| 指标 | 数值 |
|------|------|
| 最大实体控盘比 | {X}%（{N} 个地址） |
| Top 3 实体控盘比 | {X}% |
| 有效流通占比 | {X}% |
| 疑似实体 | {名称}（置信度: {高/中/低}） |

### 地址聚类详情

**实体 #1（{X}%，{N} 个地址）：**
- 代表地址: {addr}（持仓 {X}%）
- 关联地址数: {N}
- 实体标签: {label}

**实体 #2（{X}%，{N} 个地址）：**
- ...

（仅展示持仓 > 2% 的实体）

### Holder 分布图

{bar chart using █ characters}

### 操盘模式分析

{有合约数据: 现货控盘度 + OI 趋势 + 推测获利模式}
{无合约: 注明无合约交易对}

### 资金流向信号

{基于 flow_intelligence: 近期大额流入/流出、异常归集}

### 风险提示

- {3-5 条具体风险}
- 建议: {操作建议}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️ 免责: 地址聚类基于链上关联推断，不等于确认同一实体。
实体归因为推测，置信度仅供参考。不构成投资建议。
数据来源: Arkham, Antseer MCP

如果背离不成立（健康），仍输出完整数据概览和 Holder 分布，说明当前持仓分散。

降级处理





holders 返回空 → 告知数据不足，输出仅有的基本面



related_wallets 大部分为空 → 自动切换到行为模式聚类（策略 B），不放弃分析



labels 无法归因 → 标注"未知实体"，仍输出控盘比例和行为特征



代币无合约交易对 → 跳过 Phase 4



API 超时 → 重试 1 次，仍失败则跳过并说明



chain 参数报错（如 422）→ 检查链名称映射表，用正确的参数重试

注意事项





控盘比例高不一定恶意（可能是未解锁代币、DAO 金库、合规做市商库存）



避免"骗局""诈骗"等定性词汇，使用"疑似""可能""推测"



每份报告必须包含免责声明



地址聚类可能不完整，在报告中说明覆盖范围

