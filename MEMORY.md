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

### 遗留待办
- [x] PRD V2.0 一期/二期拆分版撰写完成（2026-05-11）
- [ ] 维修进站-出站模型的具体页面设计（工序管理下新页面）
- [x] n8n工作流中 status 校验的改造范围确认 — 结论：加status=4后现有校验不需改
- [ ] 扫码校验规则何时上线（一期还是二期）
- [ ] 日本/平贴合工单自动结单方案（二期）
- [ ] 远峰MES工具使用情况（问思雨，二期）
- [ ] ~~cron记忆同步修复~~（需持续观察）
