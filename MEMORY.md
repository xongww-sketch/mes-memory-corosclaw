# MEMORY.md - 长期记忆

_此文件存储重要的上下文和记忆。_

---

## 🏭 MES 组装段项目（2026-04-20 启动）

**项目负责人**：熊旺
**助手**：虾王（本 agent）

**记忆文件索引**：
- `memory/2026-04-week-MES-assembly.md` — 4/20-4/24 完整周记忆（含8个核心收口、工厂沟通结论、一期/二期PRD V1.0、任务拆解表、业务模型）
- `memory/2026-04-27.md` — 4/27 方案重新校准（一期真实约束、12个Q&A、收敛后的落地方案）
- `memory/2026-05-11.md` — 5/11 F3/F4/F5 PRD V1.0→V2.0 撰写（熊旺纠偏、一期/二期拆分、工时6.5+3.5天）
- `memory/2026-05-21.md` — 5/21 scrap_record验证、标准字段规范、线别架构讨论、PRD知识库整理
- `memory/2026-06-03.md` — 6/3 MES→coros后台同步架构重设计（三通道+状态机+对账、6工作流盘点、V1设计稿交付）
- `memory/2026-06-11.md` — 6/11 n8n-mcp-gateway 搭建与迭代（v0.1→v0.4 + Skill 封装分发 + token 精简）、cron 定时任务修复验证
- `memory/2026-06-12.md` — 6/12 n8n 全链路数据完整性论证（两轮分析+熊旺决策：先稳后推+测试先行）

---

### ⚠️ 一期重大校准（2026-04-27）

4/27 熊旺重新梳理了真实现状，推翻了部分上周设计。以下是**最新确认版**，以此为准：

#### 05-11 熊旺关键纠偏
- **维修没有"修不好"的分支**：维修就是修好回产线，不存在"维修后作废"。作废在F5做，跟维修工站无关
- **F5异常SN筛选**：不用固定48h阈值，改为自由填写时间（如≥24h、≥48h、≥72h）
- **F5作废可以做解绑**：有动态SN作废时自动查绑定关系+一键解绑，但要做权限和逻辑强校验
- **补料记录表一期就做**：追溯链（new_sns ↔ void_sns）一期直接实现
- **scrapQty字段名保持不变**：不改为 voidQty

#### 一期真实约束
- **没有SAP对接、没有WMS**
- **5月9号上线，2个开发**（约12个工作日）
- 已有：工单、SN生成规则（Lowcode）、维修工序页面（Lowcode）
- 一期明确不做：SN跨工单、样机审批、返工拆分、SAP状态、配置化筛选、历史挂账修复、报废审批流、缺陷代码库

#### 工单状态（确认版）
```
CREATED → PRODUCING → CLOSED
```
- 任何状态可修改计划数，不支持删除工单
- 关单后不能重新打开（一期）

#### 自动关单（确认版）
- **仅 HS 开头料号**（不是上周的4种前缀）
- 条件：completionQty >= planQty 且 repairingQty=0 且 pendingReplenishQty=0
- completionQty = saleableGoodQty + sampleQty

#### 组装看板（一期大屏版）
- 顶部5统计卡 + 活跃工单列表（12列含四工序进度）+ 底部4汇总 + 右侧异常面板
- 四工序表头加短名：老化/点胶/静置/外观
- 活跃工单池：status=PRODUCING
- 异常条件：repairingQty>0 或 pendingReplenishQty>0
- 筛选栏：工单类型(多选) + 工单状态(多选) + 物料号(搜索)
- 大屏适配：数字≥48px，表格内容≥24px

#### 维修/报废（方案重大演进 — 2026-05-08 最终确认）
**旧方案（F3）**：入维修/出维修/直接报废三个操作
**新方案（进站-出站模型）**：
- 进站：扫SN确认送维修 → status=4（维修中）→ repairingQty+1
- 出站二选一（⚠️ 05-08 变更：报废合并到作废）：
  - **修复**：填维修记录 → status恢复 → repairingQty-1 → 回产线
  - **作废**：填报废/作废原因 → 解绑 → status=3 → repairingQty-1, voidQty+1, pendingReplenishQty+1
- 新增维修中SN列表Tab（显示滞留时长预警）
- 任意工序都能转入维修
- 工序扫码校验：status IN (1,2) 允许；status IN (0,3,4) 拒绝并给不同提示
- 一期不做报废审批

**⚠️ 05-08 概念重构：作废 vs 报废**
- 熊旺提出：**报废 ≠ SN报废，报废 = 物料报废**。SN层面只有一个动作——**作废**
- 不管什么原因（中框坏了/丢码/误生成），SN不能继续用了，统一叫**作废**
- 直接用现有 `status=3（已失效）`，不需要新增 status=5（报废）
- 物料报废是人工登记，不在MES系统里做（一期不改）
- 工单统计表 scrapQty 字段名**保持不变**，不改为 voidQty（05-11 熊旺确认）

#### 补料（F4 — 2026-05-08 确认方案）
- 放在**工单管理**的工单列表页，不在Lowcoder里
- 工单列表加"待补料"列 + "补料"按钮（仅待补料>0时显示）
- 点击补料 → 弹窗输入数量 → 生成新SN → pendingReplenishQty -= N
- 熊旺决策：作废在MES后台，补料也应在MES后台，不跨系统

#### SN状态查询（F5 — 2026-05-08 确认方案）
- 放在**数据查询**菜单下，新增页面"SN状态查询"
- 3个Tab：工单SN列表 / 无动态SN / 异常SN（48h阈值）
- 作废操作在SN状态查询页内做（不单独做页面）
- 替代旧的"删除工单无动态SN"页面（Lowcoder）
- 一期不做SN详情页，所有操作在列表页完成

#### 样机处理
- 方案："关单时算总账"——手动关单时必须填样机数量
- 有样机 → 自动关单条件不满足 → 变手动关单 → 填样机数 → 闭环
- **Q13 待熊旺确认**

#### SN 状态扩展（2026-05-08 最终确认）
- 原状态：0=反投机, 1=已生成, 2=已绑定苹果token, 3=已失效
- **新增状态：4=维修中**（唯一新增）
- ~~5=报废~~ → 取消，复用 status=3
- ~~6=丢失~~ → 取消，丢失直接走作废（status=3）
- 作废统一用 status=3（已失效），所有30+个n8n工作流已兼容

#### 中框报废后SN处理
- **已确认**：闭环旧SN + 生成新SN（不复用旧SN）
- SN复用导致追溯断裂、售后混乱、数据不一致
- SN是物理个体身份证，不可转移（ISO 13485 / IATF 16949）

#### 丢码处理方案（05-08 简化）
- 丢失直接走作废流程（status=3）→ 补料
- ~~不再设独立的"丢失"状态~~（熊旺认为缓冲期实际价值有限）
- 三层防线简化为：预防（工序校验）→ 发现（异常SN 48h预警）→ 处置（作废+补料）

---

### n8n 工作流全量分析（2026-05-08 完成）
- 322个工作流全部解压，索引见 `references/n8n/all-workflows/`
- 核心发现：60+处 `status in ('1','2')` 校验，加status=4不需要改这些
- 分类：SN/retroid(160) / 维修/报废(55) / 绑定工序(139) / 工序校验(3) / 工单管理(4)

### PRD文档链接（最新 V2.0 一期/二期拆分版）
- 📦 组装看板：https://www.feishu.cn/docx/IBTCdp3uGoPqVLxZOnycW7WGnHg
- 📋 工单结单：https://www.feishu.cn/docx/BcggdmZhvo1D8txqkj5cq2Xdnge
- 🔧 维修报废（旧）：https://www.feishu.cn/docx/WGnJdeblgopw8WxZcmicCSL4nxd
- 🔧 **F3 维修工站 PRD V2.0**（2026-05-11）：https://www.feishu.cn/docx/CzlZd4s5ho3sg1xRBKYcDlNondg
- 🔍 **F5 SN状态查询与作废 PRD V2.0**（2026-05-11）：https://www.feishu.cn/docx/TLvadwR6GoeqJfxmyMocbgzjnvg
- 📦 **F4 补料SN生成 PRD V2.0**（2026-05-11）：https://www.feishu.cn/docx/Dd9qd6pepoGSiuxwgzqc2nlHnbc

### 二期规划（2026-05-07 确认）
- 工单类型治理、SAP映射对接、历史挂账修复、试产结案、版本管理、审批流、自动结单完善、远峰对接、工时统计

### 开发量估算（一期/二期拆分版 — 2026-05-11 确认）
| 模块 | 一期 | 二期 |
|------|------|------|
| F3 维修工站 | 1.5天 | 0.5天 |
| F5 SN查询与作废 | 3天 | 2.5天 |
| F4 补料SN生成 | 2天 | 0.5-1天 |
| **合计** | **6.5天** | **3.5-4天** |

原粗略估算 10-13 天，拆分后一期 6.5 天可覆盖全流程。

### 一期新建数据库表
- `repair_flow_record` — 维修进出站记录
- `void_record` — SN作废记录
- `replenish_record` — 补料记录（含追溯链 new_sns ↔ void_sns）

### 一期新建 n8n 工作流
- `repair_checkin_api` — F3进站API
- `sn_query_api` — F5查询API
- `sn_void_api` — F5作废API
- `sn_void_precheck_api` — F5作废前查询
- `sn_replenish_api` — F4补料API
- `replenish_precheck_api` — F4补料前查询

### 一期改造 n8n 工作流
- `repair_message`（50节点）— 末尾追加3节点（出站逻辑）
- `repair_message_line`（68节点）— 末尾追加3节点（出站逻辑）

---

### 🤖 Agent 知识库适配方案（2026-05-14 启动）

**发起人**：熊旺
**目标**：将 MES 知识库转化为跨 agent 通用的知识底座，让任何同事用任何 agent 都能即插即用地查询业务/技术问题

**核心约束**：
- 文件结构神圣不可侵犯（人类规划目录/章节，agent 只在内容槽位内操作）
- 写入路径：Agent → 代码仓库 PR → 守护人（熊旺）审核 → 合并
- 飞书 → Git 同步：每日一次（凌晨）
- Source of Truth：公司 GitLab `http://git.coros.team/`，GitHub 作外部镜像

**基础设施**：
- 服务器 `192.168.64.54`（Docker / OpenClaw / Hermes / 飞书机器人通道）
- 待确认：各角色分配（同步任务 / Hub 服务 / 飞书问答机器人 / Agent 沙箱）

**待办**：
- [ ] 具体方案设计（Agent 架构与 Harness Engineering）
- [ ] 服务器角色分配

---

### 📐 数据库表设计规范（2026-05-21 确认）

**熊旺明确要求**：所有新建表必须包含以下 5 个标准字段：
```typescript
createBy?: string;   // 创建者
createTime?: Date;   // 创建时间
updateBy?: string;   // 更新者
updateTime?: Date;   // 更新时间
remark?: string;     // 备注
```
**适用范围**：所有未来新建的数据库表，包括一期表和二期表。
**已验证通过的表**：`scrap_record`（2026-05-21 SQL验证通过）

---

### 📚 MES系统架构文档体系（2026-05-13 启动）

**发起人**：熊旺
**目标**：新人快速上手 + 未来Agent数据源
**原则**：一手资料为先（322个n8n JSON + MES Server源码 + 数据库结构），不直接用Agent仓库整理成果

**5份文档（全部完成）**：
1. MES系统全景图 → https://www.feishu.cn/wiki/WA9lwEC16iMlkqkoO31cmsRenqf
2. 业务流程总览 → https://www.feishu.cn/wiki/DRGEw2bcFiVSQkkYoyAcYnXe
3. 数据库表结构与ER图 → https://www.feishu.cn/wiki/Rp2wwKTRoiBHsik4uzQcP25FnMb
4. n8n工作流全景图 → https://www.feishu.cn/wiki/EtQBwJE73ijIETkxWaRcOePCnoh
5. 技术架构与接口标准 → https://www.feishu.cn/wiki/Zb5pwhPYRiBq9IkvKGicwxI6nrs

**知识库节点**：数据系统地图 (TIiawnp4vijBefkCdsicsOfknad), space_id: 7231809821765517316
**GitHub**：https://github.com/xongww-sketch/Coros-Mes-System-architecture-documentation

**关键教训**：
- 写文档要宏观思考，不能只从最近项目出发
- 新增内容要整体融入（如mes_daily_data不能只追加一章）
- 画板优于文字，能用Mermaid就不写文字
- 飞书Mermaid不支持\n换行

**待办**：
- [ ] Step 7: 全部推送GitHub（Markdown + n8n工作流JSON + 数据库结构文件）
- [ ] 知识库目录调整建议
- [ ] 熊旺审阅后可能的修改

---

### 🚧 新增记忆（2026-05-26）

#### MES 与售后系统的职责边界
- **决策**：MES 不主动承接售后维修重新绑定的子数据
- **理由**：MES 管产线制造段（SN生成 → 工序绑定 → 成品出站），售后是出厂后生命周期，由售后系统管理
- **例外**：售后换中框需重新过 MFI 绑定的场景，可能需走 MES 工序（待熊旺确认具体流程）
- **已验证**：322 个 n8n 工作流中不存在售后设备 ID 修改推送链路

---

### 组装看板大屏前端设计（2026-05-27 定稿，05-28 确认）

**设计方向**：浅色主题（#F5F7FA），车间电视大屏，3-5米观看距离

**顶部5个统计卡片（已确认）**：
1. 活跃工单 — 不变
2. 今日产出 ← 替换「今日计划」（前端累加4工序当日完成数）
3. 总计划 — 不变
4. 总完成 + 完成率进度条 — 不变
5. 直通率 ← 替换「总在制」（前端计算：良品/(良品+作废+维修)×100%，≥95%绿/90-95%黄/<90%红）

**底部3个汇总（从4个砍到1个）**：
1. 总维修中（红）
2. 总待补料（橙）
3. 可结单（绿）
- ✅ 删除「总作废」（累计值无行动价值，作废分析应在报表页）

**表格重组（已确认）**：
- 工序列8列压缩为4列（双行单元格：上行总到达/总数量 + 下行今日完成/计划）
- 字段分组：身份区|进度核心|工序流转|异常区|明细
- 分组表头背景色 + 右侧粗边框区分
- 新增完成率列（进度条+百分比）
- 斑马条纹 + 异常行红色左边框
- 报废→作废（全局改名）

**Vue文件已交付确认（多轮迭代）**：
- Element Plus el-table-column 不支持百分比宽度，必须用 min-width + px
- 今日产出Bug修复：updateStatCards 在 fetchTodayPlans 之前调用导致永远为0，通过 fetchTodayPlans 的 finally 块补刷解决

---

### 📋 MES 菜单结构优化（2026-05-28 启动）

**发起人**：Jackson
**目标**：优化 MES 系统菜单排列，提升使用便利性

**核心约束**：
- menuId 和维度 ID 不能改（已有大量业务关联）
- 物料与辅料查询仍在"数据查询"下，不移动
- 系统监控 / 工具 / 管理菜单不动
- 其他菜单名、层级、排序可调整

**交付方式**：创建 `sys_menu_v2` 备份表 + 纯 INSERT SQL，不直接修改生产数据

**状态**：方案已确认，SQL 待交付

---

### 🎨 可用工具 / Skill

**guizang-ppt-skill**（2026-05-28 安装）：
- 位置：`~/.claude/skills/guizang-ppt-skill/`
- 作者：歸藏 [https://github.com/op7418/guizang-ppt-skill](https://github.com/op7418/guizang-ppt-skill)
- 用途：生成单文件 HTML 横向翻页 PPT，两套视觉系统：
  - **Style A 电子杂志风**：序事、观点、个人风格（Monocle 周刊感）
  - **Style B 瑞士国际主义**：事实、产品、分析、方法论（网格、锁定色、高对比）
- 触发词："做一份瑀藏风 PPT"、"瑞士风 PPT"、"杂志风演讲仁"等
- **Jackson 偏好**：以后 MES 业务划图优先用这个 skill（粉丝他）

---

### 📊 渠道（channel）数据回补（2026-06-01 启动，06-02 持续）

**问题**：8000 个 SN 有激活数据但 `daily_info.channel` 为空，影响新 MES 和旧 MES 供给 coros 后台。

**初始交付**：`channel_backfill.json`（28 节点 n8n 工作流），并行双分支架构。

**06-02 遗留问题**：606 个 SN 回补后仍无渠道。特征：China 517 + 海外 89，激活 4/1~5/31，retroid 全部 10 位（非过滤问题）。大概率源头本身无渠道数据。

**06-02 迭代交付**：
- `channel_diagnose.json`（9 节点）— 诊断 606 个 SN 卡在哪个环节（4 分类：源头无/生产缺失/流程漏网/已有渠道）
- `channel_backfill_by_list.json`（17 节点）— 简化版回补，无生产信息也插 daily_info（production_id 允许 NULL）
- `channel_check_readonly.json`（6 节点）— 纯只读查询，先看清哪些有渠道再决定补

**06-02 管易调研**：
- 管易 API 按**单据编号 code** 查询，不能直接按 SN 查
- SN → channel 路径：SN → 单据编号 → 管易 detail → shop_name/customer_name → 映射表转 channelId
- 管易同步源头工作流 `TofAt0klEbQoXdN5`（52 节点），日期硬编码在签名节点（6 小时窗口）
- 管易查询限 24 小时间隔 → 按天拆分交付 3 个工作流：guanyi_sync_0405 / 0406 / 0407
- 先跑管易同步补源头 → 再跑回补流程 → daily_info 补上

**管易映射表**（channelId 重要参考）：
- 线上：抖音10851 / 天猫10587 / 京东10760 / 小红书10920 / 有赞10764 / 微信小店11049 / 得物11048 / 订货商城11064 / COROS高驰数码旗舰店11096
- 线下：飞亚达11066 / 上海蔚景11055 / 北京京东11066 / 云南九机11054 / 迪脉10866 / 顺电11090 / 京东闪送11075 / 无锡汇跑11020
- 特殊：shop=COROS线下渠道→默认10470；shop=订货商城→customer_name本身就是channelId

**性能优化建议**：`CREATE INDEX idx_daily_info_retroid ON mes_daily_data.daily_info(retroid)`

**文件**：`/home/node/.openclaw/workspace/channel_backfill.json` 及 06-02 各迭代版本

---

### 🔄 MES→coros后台 同步架构重设计（2026-06-03 启动，2026-06-04 选型定稿）

**发起人**：熊旺
**目标**：解决长期存在的"漏同步/渠道空/wifimac空/无法重试/无法及时发现/实时性不足"

**根因**：用"定时批量快照"承担了本应"事件驱动 + 状态机 + 对账补偿"的职责。

**⚠️ 架构选型定稿（2026-06-04）**：全 n8n 路线，不动 Server
- 熊旺提出：MES Server 有 MQ 能力，能否用 Server 统一做多数据源操作？
- 分析：Server 用 Prisma 只连主 PG 库，同步涉及多异构数据源（PG mes/mes_jp、SQL Server 旧MES、MongoDB OMS、管易 API），Server 天然不适合。
- 熊旺纠正现状：**n8n 已把绑定/解绑工作流跑起来了，Server 基本是空壳**。
- **决策**：先全用 n8n。绑定/解绑实时推送直接在现有 n8n 工作流末尾挂 HTTP 节点；Server 不碰。

**目标架构**：三通道 + sync_task状态机 + 对账补偿
- 🔴 实时通道：绑定(工序39)/解绑(工序62)末尾挂推送 → coros实时PUSH接口
- 🟡 批量通道：生产/包装/渠道保留定时但加状态登记 + 幂等upsert
- sync_task表：retroid+sync_type唯一，记录PENDING/SUCCESS/FAILED+retry_count+last_error
- 对账工作流：定时扫缺口（缺渠道/缺wifimac）→ 重入sync_task → 仍失败告警
- daily_info结论：保留小改不重做

**全 n8n 实现步骤（4步 + 1卡点）**：
1. 建 `sync_task` 表 → 2. 改6个工作流追加 HTTP 节点 + 状态登记 → 3. SHA1 签名节点 → 4. 对账工作流
- ⚠️ 唯一卡点：coros 实时 PUSH 接口文档（地址+入参+SHA1签名规则+secret），没有则第3步做不了，但1/2/4可并行开工

**交付物**：
- 设计稿 V1：`workspace/sync_architecture_design_v1.md`
- 飞书文档：https://www.feishu.cn/docx/FubzdyBq0ogPLIx5RVjcZnxNn4e
- 工作流备份：`workspace/references/n8n/current-sync/`

**coros实时PUSH接口**：
- URL: `POST https://simpleadmincntest.coros.com/coros/admin/product/mes/push`
- 签名: SHA1 40位hex（⚠️ 签名拼接规则+secret密钥待提供，落地阻塞点）
- Body: `{ "msg": [{完整daily_info记录}] }`，msg数组可批量，同一SN可多次push增量补全

**5个待确认**：
1. coros实时推送接口文档
2. 绑定/解绑的n8n工作流位置
3. 包装完成事件判断方式
4. 旧MES回写范围（含日本数据）
5. 重试/对账频率

---

### 🔌 n8n-mcp-gateway 团队查询网关（2026-06-11 上线）

**发起人**：熊旺
**目标**：团队内任意 agent 通过标准 MCP 协议或 REST API 只读查询 n8n 工作流信息，无需暴露 n8n 真实 API key。

**部署**：192.168.64.54:9130（内网，物理隔绝外部）
**代码**：workspace/n8n-mcp-gateway/（Node+TS+@modelcontextprotocol/sdk，Streamable HTTP）
**容器**：/opt/n8n-mcp-gateway/，docker compose

**最终形态（v0.4）**：
- `/mcp` — MCP Streamable HTTP 端点
- `/api/query/{tool}` — REST JSON 查询端点（Bearer token 鉴权）
- `/skill/n8n` — 裸 SKILL.md（text/markdown），agent web_fetch 即装
- 无管理 UI，token 管理走配置文件 config/tokens.json

**4 个只读工具**：find_workflow / find_email_recipients / read_workflow_logic / find_failed_executions

**架构**：token → 角色 → 能力白名单 → n8n key 池（四层解耦）
**Token**：tok_8iOYaRZKVBf2vH3Piw58EWVe7WQp8qcj（全体团队-只读）
**设计文档**：https://www.feishu.cn/docx/Eh8Xd4kv0oHF1AxzzlJcBhV7nHf

**Skill 分发**：
- 统一安装 URL（aihot 风格）：http://192.168.64.54:9130/skill/n8n
- 一个 SKILL.md 含两种接入方式：A）装 MCP 客户端 / B）直接 curl REST
- 支持平台：Claude Code / Cursor / Codex / OpenClaw / Gemini CLI / GitHub Copilot / OpenCode / Cline / Windsurf 等
- Skills Registry：http://skills.apps.coros.team（slug=n8n，n8n-mcp，n8n-curl）

**安全**：n8n 真 key 藏后台；token 只读可控；仅内网可访问

---

### 🔄 n8n 全链路数据完整性论证（2026-06-12 启动）

**发起人**：熊旺
**目标**：确保一条 retroid 的所有信息（包装、生产、渠道、wifimac、电池、屏幕、组件、uuid）进入 daily_info 时一个都不少。

**两轮论证已完成**：
- 第一轮：5 个生产同步工作流数据缺失风险分析
- 第二轮：13 个关键工作流全链路分析（fullchain 目录）

**关键风险点**：
1. 时序缺口：定时任务跑在数据到位前 → 空值写入后不再回补
2. 渠道依赖链过长：管易 API → 本地表 → get_channel → daily_info，任一环节断裂 → channel 为空
3. daily_info 非唯一键：一个 retroid 有多条记录，现有逻辑取最新 create_time

**熊旺决策**：
- 先稳后推：数据稳定性优先于推送 coros 后台
- 测试先行：复制全套测试工作流，验证 0 bug 后再替换生产
- 可以更改/重写有问题的逻辑和工作流，结果必须理清楚
- 未来方向：从"coros 后台拉取"改为"我们主动推送"

**下周一待交付**：SQL 建表 + 测试工作流复制 + 实施计划

---

### 🔧 activation_records 表治本改造（2026-06-15）

**背景**：品质团队需追溯手表原始激活日期（第一次激活 → 登记不良的间隔），但换主板重新激活时旧记录被删除，原始激活时间永久丢失。

**方案**：停止删除旧记录（改为 deactivestatus='0' 标记失效），新建 `sn_first_activation` 表存储每个 retroid 的首次激活记录（不可变），4 个 Excel 历史文件（25年/26年 国内/海外）共 25,448 条导入完成。

**品质查询走新表**：Defective 系列 API 查询原始激活时间从 `sn_first_activation` 获取，不影响现有常规查询。

**⚠️ 教训**：工作流复制必须通过 n8n API key 拉取最新生产定义，不能用本地缓存的 n8n.zip。每个工作流必须独立 JSON 文件。

**状态**：方案已确认执行，数据已导入验证。完整工作流 JSON 最终交付待完成。

---

### 遗留待办
- [x] PRD V2.0 一期/二期拆分版撰写完成（2026-05-11）
- [x] F3 维修工站 PRD 增补：进站设计增加纯前端统计列表（工单号+数量，带清零按钮）（2026-05-25 新增，2026-05-26 已写入文档）
- [ ] 维修进站-出站模型的具体页面设计（工序管理下新页面）
- [x] n8n工作流中 status 校验的改造范围确认 — 结论：加status=4后现有校验不需改
- [ ] 扫码校验规则何时上线（一期还是二期）
- [ ] 日本/平贴合工单自动结单方案（二期）
- [ ] 远峰MES工具使用情况（问思雨，二期）
- [x] cron记忆同步修复：2026-06-11 彻底定位根因（jobs.json文件存在但调度器内存未加载），重建为完整 agentTurn 版本。新任务ID `7432242d-2bd0-45e6-ba13-6f0243779c3f`「MES记忆每日同步」，cron `0 21 * * *` Asia/Shanghai，isolated agentTurn，delivery announce→飞书。force run 验证成功（90秒真干活）。
- [ ] 检查一期新建表（repair_flow_record、void_record、replenish_record）是否已包含标准字段（2026-05-21 新增）
- [ ] PRD 知识库按业务分类整理（2026-05-21 启动）
- [ ] 线别架构全方位方案设计（行业参考 + 扩展性 + 可维护性）（2026-05-21 启动）
