# 终末地排轴功能调研与可行性评估

> 调研日期：2026-09-10。本文档整理"用户给四个角色，bot 输出排轴"功能的全部已验证调研成果，作为后续开发的资料底座。
> 关联文档：[ake_data_intro.md](ake_data_intro.md)（AKEData《终末地数据导论》全文收录）。

## 1. 需求定义

**排轴** = 一整套战斗流程编排：使用战技消耗技力、触发连携、保持普攻/重击回技力、为终结技充能、释放终结技。

核心优化目标：

- 最大化所有角色的 buff 利用率与覆盖率
- 最大化技力与终结技的使用收益

必须支持的关键机制：

- **停表折叠时间**：终结技动画播放期间 buff 计时暂停，但连携技 CD 继续计时——可以利用这一点拖出某些连携技
- **充能是阈值型而非线性**：充能获取来源固定且离散，60% 充能效率可能差一点充满，61% 刚好充满——资源花费不连续
- **冷/热启动**：危境副本 100% 充能开局（热启动）；影拓丰碑等 0% 充能开局（冷启动）

## 2. 数据源调研结论（全部实测验证可获取）

### 2.1 AKEData（机制与帧数原始数据）

- 数据 CDN：`https://data.akedata.wiki`，入口 `manifest.json` → `versions[].tableCfgPath`。当前最新 `1.5.3@9885010-4`（2026-09-03）
- 仓库已有接入：`plugins/endfield/akedata_client.py`（目前仅奖章，链路可直接扩展）
- **TableCfg 表**（`/<tableCfgPath>/<Name>.json`）：
  - `CharacterTable`（33 名角色，含 `attributes` 94 项逐级面板、`mainAttrType`/`subAttrType`）
  - `SkillPatchTable`（技能等级数值缩放）、`CharGrowthTable`（成长/技能组：普攻/战技/连携/终结技四类 `skillIdList`）
  - `WeaponBasicTable`、`WeaponTalentTemplateTable`、`EquipTable`、`EquipSuitTable`（套装效果 skillID）
  - `EnemyTable`、`EnemyAttributeTemplateTable`（敌人属性，防御恒 100、抗性 0.92 之类）、`I18nTextTable_CN`（~18MB 文本）
- **技能帧数**：`/public/Json/SkillData/<skillId>.json`，共 2621 个文件（`chr_*` 727 个）。实测字段：
  - `durationFrame` / `exclusiveFrame`（独占帧）/ `startCdFrame` / `offsetRecordFrame`
  - `costData: {costType: "UltimateSp", costValue, atbValueThreshold}`（技力消耗/充能阈值）
  - `actionGroupData.timelineActions` + `passiveEventActions`（逐 hit 时间轴动作）
  - `buffs` / `toggleBuffs`（挂 buff）、`blackboard: [{key: "atk_scale", valueDouble: 0.42}]`（倍率）
- **Buff 数据**：`/public/Json/BuffData/*.json`，共 2872 个。关键字段 `useTimeDilationDt` / `onlyUseSelfTimeDilation` —— **停表机制的原始编码**
- **文件清单**：`https://data.akedata.wiki/asset-sync-index.json`（10.7MB，`datasets.json.files` 按路径索引，含 size/hash/version）
- **隐藏模块**：`https://www.akedata.wiki/plugin/manifest.json`（42 个模块，`v3_skill`/`v3_buff` 等 `hidden: true`）；渲染/解析逻辑在 `plugin/js/v3-skill-data.js`、`v3-skill.js`，可读，可反推数据语义
- **研究文章**：`https://www.akedata.wiki/public/CH/research/manifest.json`（6 篇，其中 2 篇需令牌 `cc-adoa192knkla09` 解锁）：
  - 《终末地数据导论》→ 已收录为 `docs/ake_data_intro.md`（8 种属性修改器 + 属性计算顺序 + 各 table 结构；3.5/3.6/3.7/3.8.3 未完稿）
  - 《藕粉选修三 新乘区导论》→ 伤害计算全流程：100 项属性枚举表、6 复合属性、8 属性聚合乘区、6 个伤害乘区（ProdCalcZone 乘算/其余加算）、10 步伤害管线、防御系数 `1/(Def*0.01+1)`、抗性 `1-Resist/100`、暴击 RNG（老版 .NET Knuth 减法生成器）、源石技艺强度三公式（伤害 `1+0.01x`、失衡 `1+0.005x`、减益 `2x/(300+x)`）
- 合规：AKEData 官方在 NGA（tid=47480426）明确支持第三方用 AKE 作数据源开发

### 2.2 zmdlogs.com（幻想科技战斗日志，真实轴）

- 站点：`https://zmdlogs.com`（NGA 发布帖 tid=46694978），Next.js SSR，**无公开 REST API**（`/api/*` 404），数据内嵌在 SSR HTML 的 flight payload（`self.__next_f.push`），可直接解析
- **战斗页** `/battle/btl_upload_<id>`（单页 0.4~1.1MB），内嵌结构化数据实测字段：
  - `timelineEvents`：逐技能施放时间线，`laneType: skill|buff`，毫秒级 `tsMsFromStart`，`sourceCharacterKey`/`targetCharacterKey`，`eventKey`（如 `chr_0016_laevat_ultimate_skill`）——**这就是玩家打出的真实轴**。样本战斗：920 条 skill 事件 + 92 条 buff 事件 + 1012 条伤害事件
  - buff 事件：`startTsMsFromStart`、`durationMs`、`effects: [{zone, element, rate, baseRate}]` → **buff 覆盖率可直接重算**
  - `roster`：每角色武器（等级/精炼/技能）+ 逐槽位装备散件 itemId → 真实轴自带真实配装
  - 每角色 `totalDamage / dps / rdps / maxHit / critRate`；`roleSkillStats`（每技能 castCount/totalDamage）；伤害事件含 rDPS 贡献归因
  - `timeSource: game_timer`、官方计时器起止标记
- **榜单**：首页挂战斗链接（实测 135 场）；`/boss/<bossId>` 竞速榜；`/boss/<bossId>/statistics?metric=dps|rdps&range=7d|14d|30d|all&potential=all|0|1-5` 角色 DPS 分位统计（P10/P25/中位数/P75/P90/最高 + 样本数 + 1.5×IQR 离群剔除）——现成的社区基准
- 注意：抓取需缓存 + 限速 + 署名回链；上传功能需 QQ 群工具，浏览公开

### 2.3 Endaxis（开源排轴器，格式与本体论参照）

- 仓库：`https://github.com/Lieyuan621/Endaxis`（Vue3 + Vite），线上 `https://www.end-axis.com/`，1 帧精度时间轴编辑、连携 SVG 可视化、分享链接
- ⚠️ **无 LICENSE**（2026-08-20 调查、2026-09-10 复查均未声明）——只可做**格式互操作**（读写其分享码），不可复制代码/数据文件
- `src/data/types.ts`（1290 行）= 完整终末地战斗本体论，与排轴概念一一对应：
  - 条件：`OperatorComboCdCondition`（连携 CD）、`UltimateCooldownReadyCondition`（充能就绪）、`SkillCooldownReadyCondition`、`EnemyHpCondition`、`ActionLinkConsumedCondition`
  - 效果：`SpGainEffect`、`UltimateEnergyGainEffect`（充能获取枚举）、`CooldownReductionEffect`、`ConsumeEffect`（消耗型状态）、`DamageHitEffect`/`DamageOverTimeEffect`、`ConditionalFreeze`（条件停表）
  - `Hit/HitGroup`：逐 hit `offset`（帧偏移）、`spRecovery`/`spReturn`、`stagger`、按等级数组的 `multiplier`
  - 每干员一个数据文件（6~40KB，33 人全覆盖）、武器/装备散件 sheet、`enemies/*.json`、`system.json`（系统常量）
- 分享码：gzip + URL-safe Base64 的项目 JSON（TODO #6 已立项兼容，含 `scenarioList`/轨道/动作/帧率/敌人配置校验）
- 社区生态：基于 Endaxis 前端的**全自动伤害模拟器**（NGA tid=46555808，1.2 版 tid=46587039）——拖技能上时间轴自动算输出/增益占比，证明"时间轴 → 总伤模拟"管线社区已趟通

### 2.4 其他参考

- NGA《终末地数据机制导论》（666bj，tid=46094556）：社区通用简化伤害公式（《数据导论》是对它的精确化）
- 官方 Skland game-data（`game.skland.com/endfield/game-data`）：配队参考
- Endaxis 移动端视频：BV1GQ8F6YE6n；介绍视频 BV1gSSvB6E69

## 3. 可行性结论（分层）

| 层级 | 做什么 | 可信度 | 工程量 |
|---|---|---|---|
| L1 检索式 | 索引 zmdlogs 真实日志，按 4 角色 + boss + 环境匹配，输出真实轴摘要 + 配装 + DPS + 原战链接 | ★★★★★ 本身是真实数据 | ~2-3 周 |
| L2 校验式 | 帧级资源模拟器：技力池、充能逐来源累加、连携 CD、buff 时间线（含停表分支）；轴可行性校验 + buff 覆盖率 + **充能断点计算器** | ★★★★ 资源机制数据齐全且精确 | ~1-2 月 |
| L3 生成式 | 模拟器当打分器，beam search/贪心生成理论轴（热/冷启动双版本），导出 Endaxis 分享码 | ★★★ 须标注"理论参考轴" | ~2-3 月 |
| L4 伤害计算 | 轴 → 期望总伤/DPS，武器装备对比 | ★★★ 公式已齐（《乘区导论》），需 zmdlogs 回测校准 | 3 月+（或走 Endaxis 互操作省一个数量级） |

关键判断：

- **充能阈值问题是模拟器优势**：充能获取是离散事件枚举（`UltimateEnergyGainEffect`）+ 充能效率属性（属性 44 `UltimateSpGainScalar`），可精确计算"60% 差 3 点、61% 充满"的断点——独立实用功能
- **敌人行为/真人手速不可建模**：生成轴天花板是"资源层面自洽的理论轴"，检索到的真实轴与生成轴必须区分标注
- **冷/热启动是场景参数**：初始能量状态不同，同一引擎出两套轴；zmdlogs 按 boss 分页，真实轴天然分环境

## 4. 核心机制 → 数据字段映射

| 机制 | 数据来源 | 字段/类型 |
|---|---|---|
| 技力消耗与回复 | SkillData + Endaxis Hit | `costData`、`spRecovery`/`spReturn`、`SpGainEffect` |
| 终结技充能 | SkillData + BuffData + 属性 44 | `UltimateEnergyGainEffect`、`UltimateSpGainScalar`、`atbValueThreshold` |
| 连携 CD | 属性 23/47/93/100 + Endaxis | `ComboSkillCooldown*`、`OperatorComboCdCondition` |
| 停表 | BuffData + Endaxis | `useTimeDilationDt`、`ConditionalFreeze` |
| 帧数 | SkillData | `durationFrame`/`exclusiveFrame`/`startCdFrame`/hit `offset` |
| 伤害倍率 | SkillData blackboard + SkillPatchTable | `atk_scale` + 等级 patch |
| 属性聚合 | 《数据导论》2.2/2.3 | 8 修改器 + 三段 Clamp 计算顺序 |
| 伤害管线 | 《乘区导论》 | 6 乘区 + 10 步流程 + 防御/抗性公式 |
| 敌人防御/抗性 | EnemyAttributeTemplateTable | `*DmgResistScalar`、`levelDependentAttributes` |

待确认细节：帧率精确语义（30/60fps，Endaxis 为 1 帧精度，`system.json` 常量待核）；停表的具体触发边界需实录交叉校准。

## 5. 风险与约束

1. **Endaxis 无许可证**：仅实现分享码格式读写（格式互操作），不复制源码/素材/数据文件；接入前建议再联系作者确认（TODO #6 已记录）
2. **zmdlogs 无 API**：SSR 抓取需节制（缓存、限速、署名、回链）
3. **数据体量**：SkillData 全量 ~200MB，只需 chr_* 子集（727 个）+ BuffData 按需，本地缓存 + 版本化
4. **生成轴可信度**：必须与真实轴区分标注，避免误导

## 6. 建议路线

- **Phase 1（~2-3 周）**：zmdlogs 抓取索引 + 队伍匹配检索（`排轴 <角色×4> [boss]`），输出真实轴 + 原战链接；充能断点计算器雏形
- **Phase 2（~1-2 月）**：拉 33 名干员 SkillData/BuffData 建本地知识库；帧级资源模拟器；轴校验（导入 Endaxis 分享码 → 可行性报告 + buff 覆盖率）
- **Phase 3（~2-3 月）**：搜索生成理论轴（热/冷启动）+ Endaxis 分享码导出闭环
- **Phase 4（可选）**：自研伤害估计，用 zmdlogs 同队伍同配装记录回测校准（预测 vs 真实逐版本校准系数）
