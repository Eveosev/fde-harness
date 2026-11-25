# FDE Harness 架构设计方案

**版本**：0.3（决策落实稿）  
**日期**：2026-11-25  
**原则**：宿主复用、端侧优先、薄接口、渐进披露、证据驱动、权限收敛、验证后复用  
**配套文档**：[PRD v0.3](./FDE_Harness_PRD_v0.3.md)、[变更说明](./FDE_Harness_v0.3_Changelog.md)  
**决策状态**：D01–D08 按“建议采纳，待试点确认默认参数”实施；下述工具、接口和代码仅为推荐实现路径，未做 POC。

## 1. 架构边界与组件

### 1.1 不重造运行时

本设计落实 PRD 的降熵交付路径：前期探查与澄清，中期工程搭建与交付，后期运维与复用。宿主负责模型推理、编码、工具调用、用户交互、身份和已有安全控制；Harness 提供跨会话工作记录、动作前提检查、压缩上下文、证据与业务能力图、计划检查、验证契约、端侧资产和复用规则。业务价值、模式含义、产品范围与指标以 PRD 为准，本篇描述可实现的数据、接口和算法。

Core 是可被宿主调用的轻量库 / 工具集合，不是持续自主执行的 Agent；Harness 不设置常驻守护进程。D01 / D02 包装为单一 Skill + 可选本地 stdio MCP Server + 等价 CLI：CLI 是基础，stdio MCP 是首选封装，共用 Python 核心且工具 / 子命令一一对应；不提供本地 HTTP。stdio 子进程仅随宿主会话启动 / 结束，不构成后台监控守护服务。无 MCP 时经 Shell 调 CLI；无可信执行边界则降级人工交接而非绕过门禁。工作流由宿主或客户执行环境运行，代码由宿主构建，运维事件由已有运行环境提供，不绑定特定宿主产品。

```text
用户（业务 / 工程视图、决定、批复）
  ↕ 宿主交互与现有权限机制
宿主 Coding Agent（推理 / 编码 / 工具 / 工程交付）
  ↕ 单一 Skill + 按当前缺口选用 reference + HostPort
CLI / 会话级 stdio MCP → 同一 Python Core（无常驻守护进程）
  ├─ ContextPack / WorkRecord / Checkpoint（修订摘要） / 动作前提检查
  ├─ ObserveImport / Redaction / Normalizer / Evidence
  ├─ CapabilityGraph / ViewProjection / QuestionCard
  ├─ WorkflowSpec / PolicyCheck / Validation
  └─ AssetRegistry / Recall / Memory / EventLog
  ↕ 已有工具适配 / AssetStore
客户受控环境（DB / 页面 / API / 文件 / 日志 / 执行器）
```

### 1.2 职责与信任边界

Skill 管流程判断、reference 路由和模板，宿主模型提出解释和候选工程方案；MCP / CLI 共用的 Core 确定性完成 JSON Schema 校验、动作前提与预算检查、资产存取、批准校验、遮罩和指纹。模型不能直接修改批准记录或宣布资产验证通过；Core 依据工具结果和显式确认更新状态。HostPort 将宿主已有工具映射为观察、决定和可选执行，环境连接器声明真实权限与副作用。AssetStore 只负责受范围约束的存取与检索，不能批准执行。

本地工具服务如暴露给宿主，应仅暴露有限动词和结构化参数，不默认接受任意 shell、SQL 或自由形式网络请求。确需编码工具时，仍由宿主现有执行与权限机制控制。Core 的预检在工具调用边界落实；没有受控执行边界的 Level 0 只能给出候选与人工交接，不能保证宿主在协议之外的调用被 Harness 阻断。

环境文字、DOM、日志、数据中的提示均是不可信输入。采集器输出数据而非新指令；宿主不得从观测材料提取授权或提高优先级。用户侧存储减少外传面，但不意味着设备安全、备份和权限问题已自动解决。

## 2. Skill 包装、渐进披露与压缩上下文

### 2.1 最小包装与 reference 路由

D01 固定一个 Skill 入口；正文只包含定位、阶段识别、安全底线、必选产物索引和加载规则。完整 schema、接口、连接器和模板独立存放，业务 / 工程模式仅改变策略配置与投影，不复制流程。模式存于 `.fde/preferences.json`，优先级为“任务明确要求 > 项目设置 > 用户默认 > 自适应”，解析与安全限制见第 11 节。首宿主验收四项：Skill / 等价指令包、stdio MCP、本地 Shell 与文件读写、执行前用户确认；具体产品仍待试点选择。

```text
skill/
  SKILL.md                       # 精简入口，不承载全量知识
  references/discovery.md        # G0、采集策略、DiscoveryBrief
  references/alignment.md        # 语义图、双视图、问题卡、G1
  references/build.md            # WorkflowSpec、接口、G2
  references/validation.md       # 测试、批准边界、G3 / G4
  references/operations.md       # 复验、升级、退役、G5
  schemas/                       # 可机器检查的产物契约
  importers/                     # MVP 四类文件导入、遮罩及适配说明
  templates/                     # 通用产物；不含客户规则
```

每次阶段入口先读对应 reference 的索引与当前 ContextPack；遇字段含义或测试规则才展开相关 schema。连接器 reference 仅在能力协商确定实际路径后加载。业务模式优先加载业务投影，工程模式优先加载工程投影；同一安全 reference 不能按模式隐藏。

### 2.2 上下文预算算法

预算输入包括宿主可用上下文、输出预留、当前已用量和本阶段预算。取本阶段上限与剩余可用量中的较小值；无法获得 token 信息时以配置的字符量 / 条目量作为估算，标记 `budget_unit=chars_estimated`。预算参数是配置，不是未经验证的性能目标。

装包顺序固定为：目标与约束、阻塞项与当前批准、关键事实与冲突、下一步候选、相关能力与证据索引、可选背景。Core 去重稳定事实，按资源指纹合并重复观测，保留变化差异；已完成阶段只携带结论和引用。若必须保留内容仍超预算，则拆成多个包并请求宿主按需读取，不静默截掉批准边界或失败证据。

原始材料默认不入主上下文。模型可按 `evidence_id` 请求限定片段；每次展开记录原因和范围。压缩不通过“把所有历史再交给模型总结”实现，而优先利用采集器结构化字段、已确认断言和增量差异；必要的模型摘要必须保留来源、冲突和不确定性。

```yaml
schema_version: '0.3'
task_id: t_expense
stage: ALIGNING
scope: {namespace: customer_local, project: demo_expense}
budget: {unit: chars_estimated, used: 采集时填入}
goal: 减少退回后的重复填报；不自动审批
facts:
  - claim: 提交后界面显示待审核
    evidence_refs: [ev_submit_ui, ev_submit_response]
    coverage: 单个沙箱样例
hypotheses:
  - claim: 退回后附件可复用
    blocking: true
    evidence_refs: [ev_return_detail]
conflicts: []
capabilities: [expense.read, expense.submit_sandbox]
open_questions: [q_attachment_reuse]
active_decisions: []
next: 生成含附件复用对照例子的业务问题卡
expand_refs: [graph:g1, evidence:ev_return_detail]
```

### 2.3 探查摘要与检查点

`DiscoveryBrief` 是环境压缩摘要，固定包含范围与指纹、资源目录、候选对象关系、动作 / 状态表、权限与风险、关键未知、证据引用、采集成本及下一步。DDL 摘要只呈现相关表、主外键、关键字段与候选关系来源；HAR 摘要呈现去重请求模板、响应字段、操作前后状态和异常。推断 Join、未见失败路径和缺失权限信息均显式保留。

需要交接或跨会话继续的委托，用稳定的 `work_id` 保存工作记录。可选的 `Checkpoint` 从指定记录修订生成，列出目标、固定产物、未决问题和相关的动作号。接手时重新核对引用、环境和权限。外部动作结果未知时，按原动作号查执行器并回读业务状态。

## 3. 工作记录与动作前提

### 3.1 工作记录的实现

宿主管理会话，Harness 定义工作记录。一个会话可处理多项委托，一项委托可跨多个会话。需要持续保存时，`load(work_id)` 返回目标、验收条件、证据引用与观测、决定及来源、未决问题和当前修订。`save(work_id, expected_revision, changes)` 在可信项目范围内创建或更新记录。首次写入须原子确认编号尚不存在。并发更新时仅相符修订可写，冲突方重读并核实仍适用的变更。一次性问答可在会话内完成。

工作记录在逻辑上独立，物理上可复用宿主 Store。复用前核实跨会话寻址、授权隔离、保留期及原子条件写入。每项工作只选一个权威后端。DataAgent 试点优先复用现有版本仓储。资产正文由 AssetStore 维护，动作回执由原执行器维护，记录保存引用。接口、积木版本、恢复例和试点验收见 DataAgent 文档集[《FDE Harness 多源输入与工作记录设计》](https://modelbest.feishu.cn/wiki/JMf6w4x6xi7u7gk9LpIcaarhnLR)。

### 3.2 如何选择下一步

宿主依据目标、证据与未决问题选择下一步，界面可据此显示探查、待确认、搭建或交付进度。以下 G0–G5 分别描述某次操作或交付的前提，按实际动作触发。补采或换源只重新核对依赖该材料的结论。执行、批准和验收各自保留可核查的记录。

| 门禁 | 必查信息 | 失败处理 |
|---|---|---|
| G0：观察前 | 目标未知、可信身份 / 环境、导入材料范围、存储位置 / 保留期、读写声明和遮罩预算 | 范围不明先阻塞；MVP 不录制；第二阶段录制另需告知和显式启停 |
| G1：搭建前 | 意图、交付形态、非目标、验收、关键语义与剩余未知 | 形成问题卡；只允许明确排除未知分支后收窄范围 |
| G2：执行准备 | 计划 hash、对象 / 参数、前后条件、错误边、人工 gate、允许动作、L0 / L1 及核验 / 恢复 | 禁止清单 / 结构不完整不执行；逐次批准，调用前指纹与权限再检查 |
| G3：验证完成 | 结构 / 模拟 / 沙箱证据、环境与覆盖、缺陷关闭或限制 | 失败修订到相关子图；未覆盖分支保持候选 |
| G4：交付 / 部署 | 业务 / 工程分别签署、固定版本、运维责任人、告警渠道、停止指引；部署另取批准 | 两类签署或运维责任人 / 告警渠道缺一即阻塞；测试批准不继承 |
| G5：复用 / 运行 | 身份、环境指纹、资产状态、所需能力、复验和本次批准；每次执行前检查 | 指纹检查不通过立即停止该运行并转人工；漂移转 STALE，事故转 SUSPENDED |

门禁产物为 `GateResult{gate, outcome, blocking_refs, accepted_limits, evidence_refs, decided_by, at}`，其中 `outcome=PASS|BLOCKED|PASS_WITH_LIMITS`。带限制通过必须绑定明确范围和用户决定，不能豁免安全检查。G2 是计划结构门禁，不是覆盖之后所有动作的永久批准；新范围、新参数影响或新环境均触发重新批准。

`IntentContract` 至少包含目标、范围、非目标、约束、交付形态、验收项、未决问题和模式。验收项指向可观察结果、证据采集方法与负责确认的人，不只写“流程正确”。决定、批准、执行和验证事件均使用稳定 `task_id / plan_id / action_id / correlation_id` 关联。

## 4. 有界采集与证据存储

### 4.1 采集器契约与复用

统一采集输入为 `scope, environment_id, resources, purpose, budget, redaction_policy, stop_conditions, since_fingerprint`。输出为 `EvidenceBatch, ResourceDigest, costs, incomplete_reasons`。预算可限制记录量、字节、时间和工具调用，触达上限后返回部分完成与原因，不能把部分采集伪装成完整环境。

MVP 仅支持导入 HAR、Playwright trace、DDL 导出文件与 API 文档；不做实时数据库目录扫描、网络录制、浏览器扩展或系统级抓包。DDL 导入只提相关结构，不扫描业务行；trace 中的 DOM / 表单提取语义标签、输入约束、角色和稳定定位候选，忽略无关样式。HAR 限目标系统与已授权动作窗口，去轮询噪声，不导入无关域；缺少快照或异步状态必须记为证据缺口。执行后的独立只读核验仍经 HostPort.observe，不属于实时录制。

导入脚本负责确定性解析、遮罩、去重和差异，宿主负责解释。适配器按片段报告格式、粒度、来源、位置及覆盖范围。宿主结合任务给出有证据引用的解释，工具核验技术事实，业务负责人确认口径。一份 HAR 可以同时包含请求观测和订单数据，缺少样本时可指导用户导出相关操作的脱敏 HAR。普通查询记录只能提供结构线索，数据库约束须用目录或 DDL 核对。每个脚本随附输入样例、输出 schema、权限声明、限制和自测；宿主只补已有工具到采集器的参数映射。增量读取优先复用未变化摘要，但权限、关键字段或业务状态变更不能被缓存吞掉。

### 4.2 录制与隐私

D05 的数据流固定为“输入流 / 受控临时区 → 规则引擎 → 类型占位或项目内一致伪名 → 客户字段白名单裁剪 → 可选模型二次识别 → 脱敏证据库 / 模型上下文”。规则第一关必选，移除 Cookie、认证头、令牌、认证参数、已知 PII 与字段黑名单；业务值默认占位或同项目一致伪名，映射不得跨项目共用。白名单仅保留确需的样例字段，不能恢复已移除的凭证；伪名映射如含敏感值只能存加密任务 / 项目区，不进入资产正文或共享包。模型识别仅为第二道补充，只能看已过规则遮罩的数据，失败不能放行原文。

导入采用流式解析或内存处理；若格式需要解包临时文件，先进入任务级应用层加密区，不复制原始敏感文件到普通目录或日志。用户已有文件的原始保管责任在 G0 说明，Harness 不承诺可删除客户独立持有的源文件；Harness 管理的正文、临时副本、缓存均适用第 11 节删除规则。遮罩失败 / 存储密钥不可用即阻塞，不静默降级。

第二阶段才增加受控浏览器：独立 profile、域名白名单、用户显式开始 / 结束、持续可见采集提示；停止后关闭采集并按 D03 清理任务临时区。系统级 MITM 非默认，仅工程师模式且客户安全批准的特殊适配器；录屏仅辅助理解，不能作为唯一证据，敏感原视频不长期保留。

录制前明确告知域名、时长 / 停止条件、数据类型、附件范围、存储位置、保留期与可能的真实写入；用户演示操作批准独立于观察授权。无法实时遮罩时用已脱敏文件或隔离沙箱演示，禁止全量明文落盘后再处理。原始敏感观测默认不落盘，必要时任务级加密临时存放且任务结束清理；脱敏结构证据随资产生命周期，均为建议默认值，试点确认。

### 4.3 Evidence 最小格式

```json
{
  "schema_version": "0.3",
  "id": "ev_submit_response",
  "scope": {"namespace": "customer_local", "project": "demo_expense"},
  "kind": "api_response",
  "source_ref": "environment://sandbox/expense/submit",
  "captured_at": "采集时的实际时间",
  "environment_id": "expense_sandbox",
  "resource_fingerprint": "采集器实际计算的摘要",
  "content_ref": "local://evidence/redacted_response.json",
  "summary": "本次样例提交响应出现状态字段，随后界面显示待审核",
  "redaction": {"status": "redacted", "policy_ref": "local://policies/redaction"},
  "integrity_hash": "脱敏内容实际计算的摘要",
  "limitations": ["仅覆盖一个测试单"]
}
```

以上占位文本必须在真实输出中替换，不是可直接签名的正式记录。完整性 hash 只证明所引用脱敏内容未变，不证明来源语义正确。用户确认也作为证据，但需区分 `user_statement` 与 `test_result`；两者不能相互替代。

## 5. 观测归一化与实体 / 状态 / 动作推断

### 5.1 从碎片到一次业务动作

先把请求 URL 参数化，区分业务对象 ID 与真正的契约路径；保留方法、输入字段、响应结构、错误分类和时间顺序。不能仅按同一 URL 合并不同动作。将 DOM 快照、用户示范步骤、网络请求和后续状态读回按关联标识、对象、时间窗口与因果线索组织为 `InteractionTrace`；时序相邻不自动证明因果。

```yaml
trace_id: trace_submit
actor_scope: 当前获授权会话
entity_ref: expense:E-001
before: {business_state: 草稿, evidence_ref: ev_before}
intent_hint: 用户点击提交审核
requests:
  - resource_ref: api_submit
    input_schema_ref: schema_submit
    response_schema_ref: schema_result
    evidence_ref: ev_submit_response
after: {business_state: 待审核, evidence_ref: ev_submit_ui}
uncertainties: [响应成功是否已完成持久化]
```

请求归一化保留脱敏证据、原系统来源引用和遮罩说明，不保存可直接复用的认证头。附件仅记录来源引用、类型 / 限制、关联关系和是否验证成功，文件正文不默认复制。异步轮询与提交分成触发动作和状态观察，避免将一次接口接受当作最终业务成功。

### 5.2 实体与状态推断

实体候选来自表主键、接口对象、页面业务标识和表单分组；跨来源相同值只是匹配线索，不自动合并跨系统实体。关系优先使用显式外键或已证实接口关系，再用只读样例与用户确认补充。无外键时记录推断 Join 及证据，不伪造数据库约束。

状态候选来自页面标签、枚举字段、响应及操作前后差异。建立“观察值 → 候选业务状态”的独立断言，保留多个解释。例如某样例值 `4` 与待审核同时出现，尚不能证明所有版本中 `4` 均表示待审核；需结合页面、读回、测试覆盖与用户确认。冲突记录到问题卡，不取平均置信度消除。

### 5.3 动作抽取与路径选择

动作边界以业务效果为中心：保存草稿、上传附件、提交、修订、读取审批结果。每个动作包含输入来源、允许前态、目标后态、实现候选、核验来源、风险与异常。一次页面点击可能触发多个动作；多个接口也可能共同实现一个业务动作，不能按点击数量或请求数量直接生成节点。

选择路径依次检查合法授权、业务语义等价、可观察成功 / 失败、幂等与恢复、维护成本。授权且契约明确的 API 可优先；若页面执行了额外字段校验或权限约束，API 路径必须补齐并验证同等约束，否则保留 UI。UI 使用语义角色、标签和稳定属性优先，定位失败后重新观察，不盲猜坐标。

降级条件必须写入实现选择记录。尚未发起动作且接口不可用，可在同范围批准与已验证等价路径下选择 UI；已发起但结果未知，先读回状态或人工接管，不能靠切换路径重试。UI 结构漂移也不能自动调用未批准 API。每个实际执行记录选中的实现及其版本。

## 6. 最小业务能力图、双视图与问题卡

### 6.1 精简 schema

`schemas/CapabilityGraph.schema.json` 等 JSON Schema 是唯一权威契约；下面 TypeScript 仅展示由 Schema 生成的薄类型形态，不另立规范。Python 模型、投影器与 TS 客户端均以同一 Schema 版本校验，生成结果纳入契约测试。

首期节点仅五类：`entity / attribute / state / action / resource`。Workflow 单独作为计划资产，不必增加复杂图数据库。边限定 `has / relates_to / transitions / implemented_by`，连接的断言均可带证据与确认状态。字段级技术映射用资源 selector 表达，避免把每个 DDL token 都建成节点。

```ts
type ClaimState = "candidate" | "confirmed" | "rejected";
type TechState = "unverified" | "verified" | "failed" | "stale";
interface CapabilityGraph {
  schema_version: "0.3";
  scope: { namespace: string; project: string };
  nodes: Array<{
    id: string; kind: "entity" | "attribute" | "state" | "action" | "resource";
    label: string; description?: string;
    resource?: { environment_id: string; kind: string; locator: string };
  }>;
  claims: Array<{
    id: string;
    subject: string;
    predicate: "has" | "relates_to" | "transitions" | "implemented_by";
    object: string;
    selector?: Record<string, unknown>;
    evidence_refs: string[];
    business: { state: ClaimState; decision_ref?: string };
    technical: { state: TechState; validation_ref?: string };
    validity: { environment_id: string; fingerprint_refs: string[]; limits: string[] };
  }>;
}
```

`transitions` 连接前后状态，selector 引用触发动作；`implemented_by` 可将实体、属性、状态或动作绑定到数据库、接口或页面资源。图节点不重复承载完整动作执行参数，引用 WorkflowSpec 的动作定义。单纯资源从属如不涉及业务解释，业务状态仍可保留候选并注明“不要求业务确认”；关键业务断言必须在 G1 明确确认或排除。

例子为 `state_pending → implemented_by → resource_expense_status`，selector 为 `{field: "status", value: 4}`，绑定页面、响应与读回证据；业务确认和技术验证独立更新。`entity_expense → relates_to → entity_line` 的 selector 表明显式 FK 或推断 Join。稳定 ID 在投影、工作流与问题卡间共享。

### 6.2 双视图生成规则

业务投影以实体及关系为骨架，使用已确认业务名称，展示动作、状态、例子和限制；未确认名称加“候选”。表名、字段类型默认折叠，但来源和风险入口始终可见。关系样例可使用虚构对象，明确不是客户真实数据；不凭字段命名生成确定的业务描述。

工程投影呈现资源 locator、关键 schema、主外键 / 推断 Join、字段 selector、动作实现、指纹差异和测试引用，同时显示业务确认状态。没有用户确认的映射在工程视图也不能呈现为确定规则。视图是同一图版本的确定性投影，修改经过 claim / decision 更新，不在两份 Markdown 视图中分别改事实。

输出必须显示图版本与证据时间。按实体或动作切片加载，业务专家能看到 ER 所表达的业务关联，工程师能展开技术依据。冲突、无来源断言与失败验证优先呈现，不能被模式过滤器隐藏。

### 6.3 问题卡生成

优先选择阻塞验收、状态解释、权限或不可逆动作的问题；同一因果主题合并，非阻塞项移到项目待办。卡片包含 `question_id, claim_refs, question, reason, options, example, evidence_refs, blocking, decision_scope`。example 至少展示一条输入与对应结果，让用户知道选项如何改变流程。

```yaml
question_id: q_attachment_reuse
question: 报销退回后，原有效附件是否允许复用？
reason: 决定“修改后重新提交”是否需要上传节点
claim_refs: [claim_attachment_validity]
evidence_refs: [ev_return_detail]
options: [仍有效则复用, 每次必须重传, 都不对或暂不确定]
example:
  label: 虚构示例，未验证
  input: E-001 因用途说明缺失退回；附件未被删除
  compare: 复用方案只补说明；重传方案还要求上传附件
blocking: true
decision_scope: 本项目所选报销类型
```

用户选择后生成业务 `DecisionRecord`，更新相关断言并保留旧候选。只保存项目内规则，不自动跨行业推广。工程问题卡可展开文件格式限制和接口返回，但仍围绕同一业务决定。

## 7. Workflow 候选、构建与验证

### 7.1 WorkflowSpec

工作流是有向图，节点为 `action / condition / observe / human_gate / end`，错误恢复可引用补偿动作。首期不设计通用编排语言；表达业务状态和控制边后，宿主编译为现有代码、Skill 或客户执行器配置。

```yaml
schema_version: '0.3'
id: wf_expense_revise
version: '0.3.0'
scope: {namespace: customer_local, project: demo_expense}
input_schema_ref: schema_expense_input
entry: read_state
nodes:
  - id: read_state
    kind: observe
    capability: expense.read
    success: state_known
  - id: submit
    kind: action
    business_action_ref: action_submit
    preconditions: [draft_or_confirmed_returned, fields_valid, attachments_valid]
    implementation_refs: [impl_submit_api_sandbox]
    risk: R2
    approval: {minimum: L0, customer_sandbox_minimum: L1, per_action: true}
    timeout_policy: unknown_outcome_then_read_state
    retry_policy: no_blind_retry
    verify: read_state_is_pending_review
  - id: manual
    kind: human_gate
    reason: 未知状态、未知提交结果或未覆盖退回原因
edges:
  - {from: read_state, on: eligible, to: submit}
  - {from: read_state, on: unknown, to: manual}
  - {from: submit, on: unknown_outcome, to: read_state}
limits: [此处为片段，正式图还需成功终点和完整失败边]
```

该片段仅展示契约，不能通过 G2；正式产物必须包含所有引用节点、成功终点、必需输入与错误边。候选生成从已确认动作及示范轨迹出发，补齐状态检查、输入校验、人工点和核验，所有新补分支仍标为推断并验证。

### 7.2 静态门禁与有限循环

检查节点引用、可达性、每个有影响动作的风险 / 批准、状态前后条件、失败边、核验及人工退出。循环必须有次数 / 时间边界、退出条件与耗尽后的人工接管；审核等待不能无限轮询，退回修订不能无限自动重复提交。业务规则不明时生成问题卡，不由模板填补。

输入与批准绑定稳定计划 hash 和对象范围，计划变化重新计算。实现是具体工具 / 参数 schema 的引用，不允许模型任意替换操作。生成工程文件后用契约测试核验是否保持门禁；若代码无法提供可观察后态，不标为已验证路径。

### 7.3 验证分层与证据覆盖

验证顺序为结构检查、静态策略检查、已导入脱敏响应 / 假数据回放、双轨授权沙箱、业务验收、工程验收。开源 ERP 报销基准与一个客户沙箱执行相同允许清单；基准可额外模拟退回。回放只验证解析与分支逻辑，不能证明实时权限、副作用、异步持久化或真实幂等。沙箱记录前后状态和外部证据，不只检查 HTTP 成功。

`ValidationRecord` 包含资产内容版本、环境指纹、验证类型、覆盖节点与规则、输入数据类别、结果、证据、验证者、时间和未覆盖项。业务专家签署结果与例外的业务验收，FDE 工程师独立签署能力、日志、部署和恢复的工程验收；两者均写入 DeliveryRecord，不能互相替代。只有符合限定范围的关键分支通过，才能建立该范围的 `VALIDATED` 资产；缺失分支从执行范围中排除并显式记录。

## 8. 最小接口与能力分级

### 8.1 Level 0 / 1 / 2

| 级别 | 必需能力 | 可交付边界 |
|---|---|---|
| Level 0 | HostPort 的 `describe / observe / decide`；AssetStore 的 `put / get / search` | 探查、问题卡、候选设计、人工交接；不能宣称实时写入已验证 |
| Level 1 | 在 Level 0 上增加 `execute`，通过 observe 核验 | 仅在实际授权范围执行与验证；MVP 只开放沙箱 |
| Level 2 | 按场景增加 `watch`、向量排序或外部执行器连接等扩展 | 自动漂移事件、连续运行接入、增强召回；扩展互不依赖 |

`describe` 返回实际能力与环境范围；`observe` 仅调用声明为只读的能力；`decide` 将业务确认或批准请求提交给宿主。工作记录是 Core 的内部持久化接口，后端按第 3.1 节选择。展示普通摘要使用宿主会话，不增加 `present`；Core 写本地事件，不要求 `emit`；调用方提供任务身份范围，不重复做 `getSession`；业务状态核验复用 `observe`，不增加 HostPort.verify。CLI / MCP 的 `verify` 是调用 Core 验证流程并复用 observe 的产品工具，与这六个 SDK 接入接口不是同一层。删除、导出、迁移是本地管理扩展，不扩大六个接入接口。

### 8.2 Python Protocol 参考签名

D02 以 JSON Schema 为唯一权威契约，覆盖 Evidence、CapabilityGraph、Action / Workflow、Asset、Validation、DecisionRecord 与 DeliveryRecord，并定义请求 / 响应、ApprovalToken 和 RunSummary。Python 核心以 uv / pipx 隔离安装，Protocol / 数据模型由 Schema 生成或以其校验；以下 Protocol 仅展示 Python 胶水形态，不替代 Schema。薄 TS 包仅含生成类型与客户端，禁止复制核心门禁逻辑。分发路径为推荐方案，不宣称包已发布。调用方 scope 必须来自宿主可信上下文，不能采用模型随意提交的项目名称。

```python
from typing import Any, Literal, Protocol, TypedDict

class Scope(TypedDict):
    namespace: str
    project: str

class AssetRef(TypedDict):
    scope: Scope
    id: str
    revision: int

class Asset(TypedDict):
    schema_version: str
    scope: Scope
    id: str
    kind: str
    version: str
    state: str
    summary: str
    content_ref: str
    content_hash: str
    applicability: dict[str, Any]
    validation_refs: list[str]

class AssetQuery(TypedDict):
    scope: Scope
    kind: str
    environment_id: str | None
    states: list[str]
    text: str
    limit: int

class ObservationRequest(TypedDict):
    scope: Scope
    environment_id: str
    capability: str
    operation: str
    args: dict[str, Any]
    purpose: str
    budget: dict[str, int]
    redaction_policy_ref: str

class DecisionRequest(TypedDict):
    kind: Literal["business", "engineering", "approval", "memory"]
    scope: Scope
    task_id: str
    subject_ref: str
    plan_hash: str | None
    view: dict[str, Any]
    required_approval_level: Literal["L0", "L1", "L2"] | None

class ExecutionRequest(TypedDict):
    observation_context: ObservationRequest
    plan_hash: str
    approval_ref: str
    approval_level: Literal["L0", "L1", "L2"]
    environment_fingerprint: str
    action_ref: str
    object_scope: dict[str, Any]
    idempotency_key: str | None
    deadline: str

class HostPort(Protocol):
    async def describe(self) -> dict[str, Any]: ...
    async def observe(self, request: ObservationRequest) -> dict[str, Any]: ...
    async def decide(self, request: DecisionRequest) -> dict[str, Any]: ...

class ExecutingHostPort(HostPort, Protocol):
    async def execute(self, request: ExecutionRequest) -> dict[str, Any]: ...

class AssetStore(Protocol):
    async def put(self, asset: Asset, expected_revision: int | None) -> AssetRef: ...
    async def get(self, ref: AssetRef) -> Asset: ...
    async def search(self, query: AssetQuery) -> list[Asset]: ...
```

`ExecutionRequest.observation_context` 复用范围、环境、能力、操作与参数字段，并不表示该操作只读；调用 observe 时适配器必须再次校验只读声明，调用 execute 时校验实际副作用和批准。为避免扩大协议，公共响应先以字典表示，必填字段如下，真实实现应以 schema 校验而非任意返回。

| 返回值 | 必填信息与错误语义 |
|---|---|
| describe | `level, environments, capabilities`；每项能力含 ID、操作集合、资源范围、读写副作用、风险、可核验方式 |
| observe | `status, evidence, resource_digests, costs, limitations`；状态为 OK、PARTIAL、DENIED、UNAVAILABLE；不以空数组掩盖失败 |
| decide | `status, decision_ref`；批准时还需可信 `approval_ref, approval_level`，L1 关联签名令牌；状态为 APPROVED、REJECTED、NEEDS_USER，无法校验一律 NEEDS_USER |
| execute | `status, action_id, evidence_refs, observed_effects`；状态为 SUCCEEDED、FAILED、UNKNOWN_OUTCOME、DENIED；SUCCEEDED 仍需业务核验 |

AssetStore 使用递增 revision 做条件写入，content version 表示不可变内容版本。新 ID 的 `expected_revision=None` 只允许创建；更新必须提供上次 revision，不匹配抛 `ConflictError`。get 读取明确 revision；search 仅返回当前允许状态的最新记录，不让较早的 VALIDATED 记录掩盖当前 SUSPENDED 状态。环境过滤由 `AssetQuery.environment_id` 表达：客户执行召回必须传明确环境 ID，通用模板查询可传 None，随后必须实例化并验证；不能借 None 绕过客户环境边界。响应中的候选还需 Core 做适用性检查。

### 8.3 薄 TS 客户端与 CLI / MCP 映射

薄 TypeScript 包从 JSON Schema 生成对象类型、枚举与响应校验器，客户端调用 CLI JSON 输入输出或 stdio MCP；不另写生命周期与批准判断。CLI 和 MCP 均路由到同一 Python handler，参数、结果、错误及版本语义相同；不提供本地 HTTP。观察导入与执行是不同边界，不靠提示词区分读写。下表是推荐命名，一项子命令对应一项工具，并非已发布 SDK 承诺。

| CLI | stdio MCP 工具 | 确定性职责 |
|---|---|---|
| `fde init` | `fde_init` | 初始化受控目录、配置和契约版本 |
| `fde observe import` | `fde_observe_import` | 仅导入四类文件、规则遮罩、摘要与证据绑定 |
| `fde graph` | `fde_graph` | 校验图断言、版本与双视图投影 |
| `fde plan` | `fde_plan` | 校验 workflow / 计划并计算规范化 hash |
| `fde approve` | `fde_approve` | 发起可信宿主确认、记录 L0 或签发 L1；不能自批 |
| `fde verify` | `fde_verify` | 验证门禁与证据覆盖，复用 HostPort.observe |
| `fde asset put` | `fde_asset_put` | Schema / ACL / CAS 校验，文件与元数据提交 |
| `fde asset get` | `fde_asset_get` | 受范围约束读取固定 revision 与内容引用 |
| `fde asset search` | `fde_asset_search` | 结构化过滤后 FTS5，返回候选而非执行权限 |
| `fde check` | `fde_check` | 幂等指纹 / 资产状态检查、RunSummary 汇总 |
| `fde export` | `fde_export` | 清单、哈希、验证、审计摘要与遮罩说明导出 |
| `fde import` | `fde_import` | 校验迁移包并恢复为待复验，不恢复批准 |

`fde observe import` 是证据材料导入，`fde import` 是资产恢复；参数和授权不能混用。所有入口均按 Schema 校验并产生同构结构化结果；Shell 降级不得绕过遮罩或门禁。执行由 Level 1 的 HostPort.execute 在实际边界校验，不提供自由 shell / SQL / 网络工具来替代。

能力协商缓存到当前会话与环境；能力缺失返回降级边界，不自动安装连接器或申请权限。不存在可信审批通道时 decide / approve 返回 NEEDS_USER；无可信 L0 回执时真实环境只允许 R0 / R1，R2 及以上不做真实执行，沙箱也必须具备可校验批准与禁止清单。没有执行能力时生成交接产物，不能绕过 Level 1。

## 9. 宿主胶水示例：SQLite / FTS5 + 版本化文件

D02 采用混合存储：SQLite（WAL）存元数据、状态 revision、索引、审计；资产正文以内容寻址的版本化文件保存，路径如 `assets/<id>/<version>/<sha256>/body.json`。数据库只保存 manifest 元数据、正文引用和哈希，不保存 workflow / 代码正文；文件中保留 manifest 与状态快照，使索引可从文件重建。审计只追加元数据的完整实现见第 11、13、14 节。

下列单进程示例实现第 8 节 put / get / search 与索引重建，保留 CAS、最新状态和先结构过滤再 FTS5；调用前 Core 完成 JSON Schema、身份 / ACL、不可变内容版本、状态门禁与允许路径检查。调用方先写正文及 hash，再传入引用。它不是生产安全存储，也未实现敏感层加密、批准或审计；这些在正式 Core 中必选，不可用样例替代。

```python
import hashlib
import json
import re
import sqlite3
from pathlib import Path

class ConflictError(RuntimeError):
    pass

class SQLiteAssetStore:
    def __init__(self, root: str):
        self.root = Path(root).resolve()
        self.root.mkdir(parents=True, exist_ok=True)
        self.db = sqlite3.connect(self.root / 'registry.sqlite', isolation_level=None)
        self.db.execute('PRAGMA journal_mode=WAL')
        self.db.execute('PRAGMA busy_timeout=3000')
        self.db.execute('''CREATE TABLE IF NOT EXISTS assets (
            namespace TEXT, project TEXT, id TEXT, revision INTEGER,
            kind TEXT, state TEXT, environment_id TEXT, manifest TEXT,
            UNIQUE(namespace, project, id, revision))''')
        self.db.execute('CREATE VIRTUAL TABLE IF NOT EXISTS asset_fts USING fts5(summary)')

    def _segment(self, value):
        if not isinstance(value, str) or not re.fullmatch(r'[\w-]+', value):
            raise ValueError('非法目录标识')
        return value

    def _content(self, asset):
        p = (self.root / asset['content_ref']).resolve()
        if not p.is_relative_to(self.root) or asset['content_hash'] not in p.parts:
            raise ValueError('正文须在受控根目录的内容寻址路径内')
        if hashlib.sha256(p.read_bytes()).hexdigest() != asset['content_hash']:
            raise ValueError('正文哈希不一致')

    def _insert(self, asset, rev):
        s = asset['scope']
        # manifest 是元数据与引用，不是 workflow / 代码等资产正文。
        cur = self.db.execute('INSERT INTO assets VALUES(?,?,?,?,?,?,?,?)',
            (s['namespace'], s['project'], asset['id'], rev, asset['kind'],
             asset['state'], asset['applicability'].get('environment_id'),
             json.dumps(asset, ensure_ascii=False)))
        self.db.execute('INSERT INTO asset_fts(rowid, summary) VALUES(?,?)',
                        (cur.lastrowid, asset['summary']))

    async def put(self, asset, expected_revision=None):
        self._content(asset)  # 正式调用前 Core 还须按 JSON Schema、ACL 与生命周期校验。
        s = asset['scope']
        key = (s['namespace'], s['project'], asset['id'])
        base = self.root / 'records'
        for part in key:
            base /= self._segment(part)
        snapshot = None
        self.db.execute('BEGIN IMMEDIATE')
        try:
            old = self.db.execute('''SELECT COALESCE(MAX(revision),0) FROM assets
                WHERE namespace=? AND project=? AND id=?''', key).fetchone()[0]
            if old != (0 if expected_revision is None else expected_revision):
                raise ConflictError('资产已更新，请重新读取')
            rev = old + 1
            self._insert(asset, rev)
            base.mkdir(parents=True, exist_ok=True)
            snapshot = base / f'{rev}.json'
            # 崩溃遗留快照须先进入恢复检查，不覆盖它。
            with snapshot.open('x', encoding='utf-8') as f:
                json.dump({'revision': rev, 'asset': asset}, f, ensure_ascii=False)
            self.db.execute('COMMIT')
            return {'scope': s, 'id': asset['id'], 'revision': rev}
        except Exception:
            self.db.execute('ROLLBACK')
            # 已落盘快照保留用于诊断；恢复必须复验，不据其直接执行。
            raise

    async def get(self, ref):
        s = ref['scope']
        row = self.db.execute('''SELECT manifest FROM assets WHERE
            namespace=? AND project=? AND id=? AND revision=?''',
            (s['namespace'], s['project'], ref['id'], ref['revision'])).fetchone()
        if row is None:
            raise KeyError(ref['id'])
        asset = json.loads(row[0])
        self._content(asset)
        return asset

    async def search(self, query):
        states = query['states']
        if not states:
            return []
        s = query['scope']
        marks = ','.join('?' for _ in states)
        # 先物化结构化过滤集合；历史 VALIDATED 不能越过最新 SUSPENDED。
        sql = f'''WITH eligible AS MATERIALIZED (
            SELECT a.rowid AS rid, a.manifest, a.id FROM assets a WHERE
            a.namespace=? AND a.project=? AND a.kind=? AND a.state IN ({marks})
            AND (? IS NULL OR a.environment_id=?)
            AND a.revision=(SELECT MAX(b.revision) FROM assets b WHERE
                b.namespace=a.namespace AND b.project=a.project AND b.id=a.id))
            SELECT e.manifest FROM eligible e'''
        args = [s['namespace'], s['project'], query['kind'], *states,
                query['environment_id'], query['environment_id']]
        if query['text'].strip():
            # 用户文本按字面量传入，不能作为任意 FTS 表达式。
            terms = query['text'].strip().replace('"', '""')
            sql += ' JOIN asset_fts f ON f.rowid=e.rid WHERE asset_fts MATCH ?'
            args.append('"' + terms + '"')
        sql += ' ORDER BY e.id LIMIT ?'
        args.append(max(1, min(query['limit'], 100)))
        return [json.loads(r[0]) for r in self.db.execute(sql, args)]

    def rebuild_indexes(self):
        # 演示从版本化元数据快照重建；正式恢复还需审计 / 墓碑约束与待复验状态。
        snapshots = sorted((self.root / 'records').glob('*/*/*/*.json'))
        self.db.execute('BEGIN IMMEDIATE')
        try:
            self.db.execute('DELETE FROM asset_fts')
            self.db.execute('DELETE FROM assets')
            for p in snapshots:
                record = json.loads(p.read_text(encoding='utf-8'))
                self._content(record['asset'])
                self._insert(record['asset'], record['revision'])
            self.db.execute('COMMIT')
        except Exception:
            self.db.execute('ROLLBACK')
            raise
```

示例等待时间与查询上限仅是保护参数，不是产品 KPI。异步签名方便接入 Protocol，但 SQLite 操作同步，只适于参考；正式实现应隔离并发调用并测试崩溃 / 迁移。FTS5 是全文检索，不宣称语义理解；中文索引 / 查询须采用一致的分词归一化并做试点测试。向量仅 Level 2 可选增强。

正文先完成遮罩、加密（适用敏感层）和完整性检查，再提交版本化 manifest / 状态快照与数据库 CAS。文件 / SQLite 不构成跨存储原子事务；生产应使用提交标识、临时文件原子替换与恢复检查。样例异常或崩溃可能留下快照 / 孤儿文件，必须隔离并诊断，不能据此宣称 VALIDATED；可信日常索引重建与备份恢复不同，后者全部置待复验。删除时必须清正文及目标所有历史快照 / 缓存后重建，不能重建出已删除资产。

内容 hash 同一版本不可变；新内容派生新版本 / 资产并记录 supersedes_ref，原 ID 可更新状态 revision。交付固定版本 / revision，运行前另查当前状态。管理删除 / 导出由授权扩展完成，不暴露任意 SQL。

## 10. 批准、权限与执行边界

### 10.1 风险收敛

沿用 PRD 的 R0 只读、R1 隔离低风险、R2 业务写入 / 可恢复、R3 高影响 / 不可逆。分类依据实际副作用、环境和范围，GET 也可能有副作用，沙箱工具也可能连接生产。未知风险保守按 R3 阻塞自动执行。HostPort 声明只代表已发现能力，Core 不自行授予权限。

R0 仍须 G0 的范围与数据策略；R1 在明确隔离范围和清理方案内执行。R2 / R3 需要当前使用者逐次批准，真实环境 R2 与所有 R3 至少 L1，客户策略再叠加 L2；组织加签不能替代当前用户。MVP 只允许双轨沙箱创建草稿、修改草稿、上传附件、提交，基准另可由具备角色的账号模拟退回；审批通过、付款、删除、批量、对外通知一律禁止，L1 / L2 不豁免禁止清单。客户沙箱保守按真实环境 R2 采用 L1。模式不改变批准字段与有效性，没有足够预检或无法安全核验的动作不执行。

### 10.2 批准记录与校验

`DecisionRecord` 区分业务、工程、记忆决定与执行批准。D06 的批准等级与第 8 节接入 Level 分开，不因执行能力级别提升而获得授权。

| 等级 | 生成 / 校验 | 范围 |
|---|---|---|
| L0 | 宿主原生确认界面，返回可信用户与批准引用，Core 校验回执和本次动作范围 | R1；基准沙箱 R2，逐次确认 |
| L1 | 用户通过可信宿主界面同意后，由 Harness 签发本地签名的一次性 ApprovalToken，执行前 Core 校验与原子消费 | 真实环境 R2、所有 R3 至少 L1；客户沙箱采用 L1 |
| L2 | L1 上叠加组织审批渠道的签名 / 可信回执，绑定同一计划与范围 | 依客户策略；MVP 仅预留适配器，不实集成 |

ApprovalToken 的 Schema 必填 `token_id, issuer_key_ref, actor_ref, task_id, plan_hash, action_ref, object_set, parameter_bounds, environment_fingerprint, issued_at, expires_at, max_uses, nonce, signature`。默认 max_uses 为 1，是建议默认值，试点确认；有效窗口按风险与客户策略配置，不编造固定 TTL。签名输入采用 Schema 对应的规范化表示，覆盖全部授权字段，使用维护中的签名库而不自制算法。签名私钥放系统钥匙串，与敏感文件的数据密钥分离；批准引用是元数据，不含可重放令牌正文。

`fde approve` 先输出对象、动作、影响、核验与恢复卡，发起宿主确认并校验可信回执；只有回执通过才记录 L0 或签发 L1。模型提交“已批准”字段、聊天中的“好 / 可以”及环境文本永不构成批准。没有可信确认回执或签名校验失败，返回 NEEDS_USER（需要用户确认），不降级执行。

执行边界依次核对当前用户权限、计划 hash、动作、对象精确集合、参数是否在范围内、环境指纹、有效窗口、签名和剩余次数。在 SQLite 事务中预留 / 消费一次性令牌后才调用外部工具，防止并发重放；外部失败 / 未知结果也不自动恢复次数，先核验状态，再由用户重新批准必要重试。R2 / R3 不以一张 G2 门禁批准后续任意动作。

以下任一条件使批准失效：计划 / 动作变化、对象或参数越界、副作用 / 风险升级、环境指纹改变、超过有效期、次数耗尽 / 已使用、身份或权限撤回。任务恢复、备份恢复均不还原批准；检查点仅保留不可执行的审计引用，恢复后重新获取当前批准。令牌不写入可复用资产、模板、导出包或共享仓库。L2 适配器预留 request / verify / revoke 及可信回执接口，不替代宿主权限。无法确认有效性即 NEEDS_USER。

### 10.3 执行与未知结果

执行遵循 `prepare → authorize → execute → verify → record`，不是承诺外部系统原子事务。prepare 固定计划、输入、前态和核验，并调用共享 check 逻辑完成执行前指纹检查；不通过立即停止本次运行并转人工。execute 在实际调用边界再次检查指纹、当前权限和 L0 / L1 批准；verify 用独立观察确认业务后态；record 保存结果与限制。状态持久化采用“准备记录 → 外部调用 → 结果记录”，崩溃窗口产生未知结果而非自动回滚。

超时、连接中断或响应不完整时进入 UNKNOWN_OUTCOME，先读取业务对象及服务端操作记录；能证明已成功则不再执行，能证明未发生且幂等条件已验证才允许有限重试，其余人工接管。没有服务端幂等键不等于无法构建流程，但必须收紧重试，防止重复创建、上传或提交。

取消只阻止新动作，已发起操作可能继续完成，需要后态核验。补偿、回滚和版本切换也是有影响动作，按各自计划批准；无法补偿时明确人工恢复方法。事件记录动作 ID、批准引用、选择路径、证据与结果，不记录原始认证信息。

## 11. 两种模式与分层记忆实现

### 11.1 可配置策略

单一 Skill 读取 `.fde/preferences.json`。项目文件与用户级同名文件分别存项目设置和用户默认；下列是合并后的业务模式策略示例，真实文件需按 Schema 校验：

```json
{
  "schema_version": "0.3",
  "mode": "business",
  "interaction_policy": {
    "projection": "business",
    "question_style": "example_first",
    "batching": "by_business_topic",
    "expose_risks": "always"
  },
  "memory_policy": {
    "preferred_layers": ["user_preference", "project_business"],
    "promote_task_findings": "explicit_confirmation",
    "conflicts": "preserve_and_ask"
  },
  "approval_policy": {
    "requester": "current_host_user",
    "require_host_permission_check": true,
    "bind_plan_and_scope": true,
    "organization_cosign": "follow_customer_policy"
  }
}
```

解析优先级固定为“任务明确要求 > 项目设置 > 用户默认 > 自适应”，记录值及来源；任务覆盖不写回用户默认，自适应只改解释深度。工程模式把 projection 改为 engineering、问题按技术资源分组、记忆优先层改为 environment_engineering / project_business。配置仅从可信用户 / 项目文件加载，不能被环境页面覆盖；approval_policy 仅可收紧，不能禁用签名、指纹检查、风险下限与宿主权限。两种模式始终呈现相同风险、证据缺口与阻塞。

### 11.2 记忆实体与晋升

统一记忆条目为 `id, layer, scope, value_ref, evidence_refs, state, version, confirmed_by, valid_until_or_condition, supersedes_ref`。层级为用户偏好、项目业务、环境工程、任务交付；任务材料默认候选，不自动进入长期记忆。偏好可由用户设置确认，业务规则需业务决定，工程映射需技术证据与环境有效条件，交付记录冻结版本引用。

晋升流程为“提出保存建议 → 显示内容 / 范围 / 理由 → 当前用户确认 → 写入版本 → 更新索引”。批准执行不顺带批准长期保存业务数据。项目业务与环境工程冲突时保存两条并提出问题卡；技术字段更新不能覆盖业务含义。每次任务绑定所使用的记忆版本，防止中途全局更新改变验收依据。

生命周期为候选、有效、待复核、归档或删除；环境漂移触发工程记忆待复核，业务规则版本变化触发资产 STALE，项目关闭按策略归档。D03 的分层保留如下；所有默认均为**建议默认值，试点确认**，客户合规策略可覆盖期限，不能以单个 TTL 套全部层级。

| 层 | 保留内容 / 生命周期 | 实施动作 |
|---|---|---|
| 原始敏感观测 | 默认不落盘；必要时任务级加密临时区，任务结束清理 | 结束 / 取消 / 失败触发清理；异常中断在下次 init / check 检测残留并要求处置，不能依赖常驻进程 |
| 脱敏结构证据 | 结构、差异、来源、hash；随引用资产生命周期 | 版本化证据文件，资产退役时按引用关系归档 / 删除 |
| 用户偏好 | 模式 / 展示；用户删除前有效 | 可查看、修改、导出、删除，不含权限 |
| 项目业务记忆 | 术语 / 规则 / 例外与确认；项目存续期，关闭归档 | 版本化、业务决定后晋升，变化时复核 |
| 环境工程记忆 | 映射 / 依赖 / 限制 / 指纹；漂移转待复核，长期不用归档 | 有效条件与配置期限共同判断，不含凭证明文 |
| 任务交付记录 | 固定验收 / 实现引用；随交付物生命周期 | 冻结版本，退役时处置，批准仅不可重放引用 |
| 审计事件 | 只追加元数据；按客户合规策略，未配置时至项目归档 | SQLite 追加日志，不存业务值；需抹除的加密元数据载荷销毁密钥而不改写历史事件 |

TTL / 清理策略放可信项目配置，由用户或客户合规责任人确认。任务结束清理是生命周期回调；逾期残留的定期处置由客户已有调度调用管理逻辑，不新增 Harness 守护进程。

### 11.3 删除、导出与边界

删除偏好不删除已交付系统；删除业务记忆 / 证据若使关联资产缺依据，同步暂停其复用并提示影响。确定性删除事务按“检查引用并暂停使用 → 删除正文与所有受管历史副本 / 缓存 → 销毁对应任务 / 项目数据密钥（加密粉碎）→ 追加墓碑审计 → 重建文本 / 可选向量索引”执行；失败显式阻塞，未完成清理不得宣称删除成功。墓碑只留谁、何时、因何删除什么引用及结果，不保留业务值或可恢复正文。共享项目密钥覆盖其他存续内容时，先迁移那些内容到新密钥，再粉碎目标密钥；备份副本纳入客户删除流程与密钥处置清单。

审计由追加接口写 SQLite 元数据日志，禁止更新 / 删除历史业务事件；删除以新墓碑表示，必要加密载荷用密钥销毁，不改写其密文。资产 / 状态文件快照在合法删除后同步清理，重建时应用墓碑，不能重新索引被删版本。

`fde export` / `fde import` 的清单、完整性校验、恢复与密钥边界见第 13 节；不导出原始敏感观测、秘密、签名密钥或可重放批准。宿主原生记忆只映射获准层级，不形成两份权威记录；缺失时使用端侧文件与 AssetStore。

## 12. 资产、召回与环境指纹

### 12.1 AssetManifest 与有效状态

资产内容包含语义图、workflow、Skill 或代码等；Manifest 至少有 schema / content version、scope、类型、内容引用 / hash、摘要、适用条件、所需能力、资源指纹、验证引用、敏感级别、限制和来源。验证记录独立保存，交付引用固定内容，不复制一份会漂移的正文。

| 状态 | 进入依据 | 允许行为 |
|---|---|---|
| DRAFT | 初始或尚未形成完整候选 | 编辑、问题澄清，不推荐执行 |
| CANDIDATE | 结构完整、等待验证 | 回放 / 受控验证，不作为可靠路径 |
| VALIDATED | 限定环境与范围的关键验证通过 | 可召回；适用性、复验和新批准后才执行 |
| STALE | 指纹、业务规则或证据有效性发生变化 | 影响分析与复验，不静默继续受影响写入 |
| SUSPENDED | 事故、权限撤回、策略或安全阻断 | 诊断与人工恢复，不执行 |
| RETIRED | 明确退出或被替代并停用 | 审计查看；不默认召回或直接复活 |

STALE 或 SUSPENDED 恢复需要新验证证据与必要的人工决定；内容变化派生新版本，不只改状态。退役只可派生新资产。状态是 registry 元数据 revision，内容 hash 同一 version 不可变；部署是否使用某版本在 DeliveryRecord / 执行器引用中独立表达。

### 12.2 指纹计算与影响传播

环境指纹分层记录系统身份、关键资源结构、契约版本、权限能力与依赖；每个工作流只依赖实际使用的资源，避免整个数据库无关变化导致全量失效。结构 hash 前先规范排序和字段表示；不含密码、token、样例业务值或会话随机量。页面指纹关注语义字段、约束和稳定定位结构，忽略无关广告或样式变化。

API 指纹包含已观察请求 / 响应结构、状态语义映射的证据版本；DDL 指纹包含相关字段类型、键、约束与关系；能力指纹包含当前权限范围，变化后重新协商。未明确的环境身份不能仅用主机名推定为同一客户环境。

漂移分类为相关契约变化、行为 / 语义变化、权限变化、无关展示变化与未知。通过图的 implemented_by 和 workflow 引用传播到受影响资产；无关变化可在新观察证据与影响记录支持下继续，未知变化保守暂停相关动作。业务语义不一定体现在 hash 中，必须结合规则版本、验收与运行异常复核。

### 12.3 四步召回与复验

第一步使用宿主可信身份做 ACL 和结构化过滤，限定 namespace、project、环境 ID、资产类型与状态；默认 customer_local 当前项目和明确环境。所需能力在第三步检查。跨项目私有资产只有显式允许才进入候选。第二步仅对已过滤集合做 FTS5 全文检索；完整首期链路为“结构化过滤 → FTS5 → 适用性检查 → 复验”。向量排序只作为 Level 2 可选增强，失效退回 FTS5 与结构过滤，不能为召回把私有数据发往未知服务。

第三步 Core 检查适用条件、规则版本、资源指纹、缺失能力、验证覆盖与证据期限，输出 `APPLICABLE / NEEDS_REVALIDATION / INCOMPATIBLE` 及原因。第四步做风险相称的复验：无相关变化可进行结构与关键只读核验，有相关写入契约变化则沙箱复测；不可安全验证时仅返回模板建议与人工接管。

复验通过后形成新记录并更新当前有效状态，本次执行仍需要新的批准。排序分数、历史通过率或“上次成功”均不能替代适用性检查。记录命中、排除、展开、复验和使用成本，支持 PRD 的复用指标。

### 12.4 公共模板与客户私有资产

`template_public` 只放通用结构、合成示例和参数槽，`customer_local` 放项目映射、私有代码、指纹及验证。公共模板不继承任何客户的 VALIDATED、生产权限或业务规则；实例化后重新形成候选。模板使用和客户资产命名空间在索引之前就隔离，不能仅靠检索提示词区分。

提炼模板需人工审查来源、去标识、业务敏感性与共享许可。客户私有表名、接口、记录和附件不自动泛化到模板。团队共享第二阶段通过客户自托管 Git 仓库，只共享去标识 workflow 模板、能力图结构、Skill 与代码；入库须遮罩检查、来源 / 许可审查和 PR 评审，敏感证据永不入库。仓库身份 / ACL 使用客户已有机制，拉取后重新实例化与复验，不继承旧批准，不改变端侧隔离原则。

## 13. 端侧目录、版本与一致性

### 13.1 权威数据与目录

```text
.fde/
  manifest.json                         # Schema / 存储配置，非秘密
  preferences.json                      # 项目模式策略；用户级文件存用户默认
  templates/                            # 已许可通用模板
  records/<namespace>/<project>/<id>/    # manifest / 状态 revision 快照，可重建索引
  projects/<project-id>/
    memory/business/                    # 确认规则与版本
    memory/environments/<env-id>/       # 映射与有效条件
    tasks/<task-id>/                    # 可重建的工作摘要及任务临时区
      temporary.encrypted/             # 必要原始观测，任务密钥保护、结束清理
    evidence/                          # 脱敏结构；敏感层应用加密
    graphs/<graph-id>/<version>/
    assets/<asset-id>/<version>/<hash>/ # 内容寻址不可变正文及 manifest
    validations/                       # 验证记录
    deliveries/                        # 固定验收、运维责任人与告警渠道
    audit/archives/                    # 授权导出的元数据审计摘要，非第二权威日志
    indexes/                           # 可重建派生缓存
  registry.sqlite                      # WAL：元数据 / 状态 / FTS5 / 追加审计
```

此目录为本地资产后端的参考布局。工作记录按第 3.1 节选择唯一后端，使用宿主 Store 时本地只保留可重建的摘要。SQLite 是本地资产的在线元数据 / 状态事务库，文件是资产正文权威副本；数据库不存正文，只存引用与 hash。manifest、版本化状态快照和脱敏证据文件使资产索引可重建，审计不依赖文本索引作唯一记录。第 9 节示例将正文目录直接置于传入项目根，正式实现由受控项目根映射上述布局。

scope 映射为校验后的目录 / 数据库键，禁止模型路径直接拼接；scope 不是认证，HostPort 可信身份和客户 ACL 决定访问。内容引用只能在受控根或明确许可原系统内。写入先完成脱敏 / 加密文件与 hash，再以 CAS 提交 manifest / 状态和追加事件，最后更新派生索引；索引失败可重建，文件与数据库间崩溃用提交标识及恢复检查处理，不宣称跨存储原子性。迁移先验证 Schema 版本、hash、引用与契约测试，再切换 registry；未知主要版本拒绝自动加载。

### 13.2 加密与密钥

D08 基础层复用系统磁盘加密，交付清单检查其是否开启；原始敏感观测、含敏感字段的证据及必要敏感载荷必用应用层加密。密钥由系统钥匙串保存，任务 / 项目按范围独立分配，正文只保存非秘密 key_ref，不与密文同包备份；销毁目标数据密钥实现加密粉碎。签名密钥与数据密钥分离，不自建 KMS；优先复用客户安全基础设施。钥匙串不可用、密钥遗失或加密配置缺失时，阻塞敏感层存取并告知恢复限制，不以明文替代。

### 13.3 导出、恢复与共享

`fde export` 生成 manifest（scope、Schema / 内容版本、固定引用、依赖）、资产文件及哈希、验证记录、必要元数据审计摘要和遮罩策略说明；不含原始敏感观测、凭证明文、任何秘密密钥、伪名还原映射或批准令牌。敏感证据不进入共享包；客户备份所需敏感层只能通过受控加密备份路径，密钥用客户既有安全恢复渠道管理，不混入迁移包。日常备份由客户已有体系承担，导出不是自动备份服务。

`fde import` 先验证 Schema 版本、包路径 / scope、全部 hash、依赖和遮罩说明，失败不切换存储。恢复后所有资产统一标为待复验（STALE），同时保存恢复前状态作为非执行元数据；原退役 / 安全暂停限制不得因恢复消失。环境身份、权限、指纹和必要验证重新通过后才生效；批准令牌、使用次数和会话批准都不恢复，审计引用只供追溯。可信原文件的日常索引重建不等于 import，重建不得绕过墓碑或恢复隔离。

设备丢失后依靠磁盘加密和钥匙串减少暴露面，从最近有效导出包恢复并复验；由客户撤销丢失设备相关权限 / 受信签名身份，旧令牌仍须过期 / 指纹 / 使用次数校验，不能假设设备丢失即自动失效。第二阶段团队共享沿第 12.4 节的客户自托管 Git、去标识、遮罩检查与 PR 评审，不共享敏感证据。

## 14. 可观察性、漂移与后期运维

### 14.1 事件与运行契约

标准事件包括 `stage.entered, gate.checked, evidence.captured, claim.proposed / confirmed, decision.requested / resolved, action.started / completed / failed / unknown, validation.completed, asset.state_changed, delivery.accepted, drift.detected, recovery.completed`。每条含时间、可信 actor 引用、task / action / asset / environment、关联 ID、结果、证据及脱敏限制；不包含原始请求头和业务文件。

运行环境提供按 JSON Schema 校验的 RunSummary，不含输入正文与敏感值：

| 标准字段 | 语义 |
|---|---|
| `schema_version, run_id, workflow_version, environment_id` | 契约、运行唯一标识、固定 workflow 版本与环境 |
| `input_scope, started_at, ended_at, duration_ms` | 脱敏输入范围、实际时间与耗时；未知耗时显式注明 |
| `success_count, failure_count, handoff_count` | 成功、失败、人工接管次数；按同一动作结果口径计数 |
| `error_categories, unknown_outcomes, duplicate_suspicions` | 错误分类、结果未知与重复操作疑点，不含原始响应 |
| `evidence_refs, limitations` | 可追溯证据及未观测 / 未覆盖限制 |

Core 与当前验证、规则版本、指纹比较，窗口汇总可派生业务结果与时延分布。普通性能异常生成建议；重复提交疑点、未知状态或权限撤回阻断受影响写入并通知当前用户 / 运维责任人及交付记录中的告警渠道。

D07 提供幂等 `fde check` / `fde_check`：输入可信 scope、环境、固定资产引用与 RunSummary 窗口，输出 `check_id, status, fingerprint_diff, asset_findings, run_summary_refs, recommended_actions, evidence_refs`。它不执行业务写入、不签发批准、不自动补偿；同一观测产生同一判定，状态仅在变化时更新，重复调用不重复创建状态转换，允许追加最小调用审计。检查接口可执行确定性只读核验，缺证据即返回 BLOCKED / NEEDS_USER，不伪造健康。

MVP 手动 check + 每次交付物执行前轻量指纹检查；执行入口强制调用共享检查逻辑，指纹不通过即停止本次运行并转人工，不允许“带病运行”。周期检查由客户调度器、CI 或宿主定时执行，频率依风险 / 业务节奏在 DeliveryRecord 约定（建议默认值，试点确认）；客户升级 / 发布前主动检查。外部周期 / 事件接入是 Level 2，可选 watch 仅传递变化，不新增 Harness 常驻进程。

G4 的 DeliveryRecord 必填运维责任人 + 告警渠道，并写运行位置、检查约定、人工接管与停止入口；缺一不进入 DELIVERED。会话结束后无外部日志 / 调度来源，显示“未启用连续监控”，不作持续监控保证。

### 14.2 故障处理与影响范围

故障流程为“暂停风险动作 → 保留最小证据 → 重读实际状态 → 分类与人工接管 → 修订 → 复验 → 明确恢复”。权限失败返回宿主重新授权流程，不自行申请扩大范围；字段或页面变化回到受影响资源采集；业务规则变化回到问题卡；外部结果未知使用第 10 节未知结果路径。

资产转 SUSPENDED 或 STALE 同时通知正在使用该版本的外部执行器停止新动作。若执行器无法响应停止，交付验收必须披露该限制，不能宣称完整受控。已经发起的外部动作继续核验；补偿需要独立批准。故障报告包含最后安全状态、可能副作用、证据、影响资产、恢复选项与责任人。

### 14.3 升级与退役

升级先派生新内容版本，复用未变证据但对变更部分重新验证；在沙箱 / 客户允许的隔离范围比较输入输出、状态和失败路径。业务验收后，经对应部署批准切换固定引用，保留旧版本。回退仅在旧版仍适用、权限有效且获得对应批准时执行，不能盲目恢复不兼容旧代码。

退役记录原因、替代版本、停用时间和责任人，停止召回与外部执行引用，清理缓存索引。保留必要验收与审计，具体删除按客户策略；不能为了清理本地资产删除 ERP 中的业务单据。无人维护或依赖消失的资产应退役或暂停，不永远保持 VALIDATED。

## 15. 报销审核技术走查

本节补充 PRD 第 13 节的实现，使用虚构 E-001；路径 / 状态值仅说明契约。开源 ERP 报销基准（如 Odoo / ERPNext，未定型、未 POC）预置固定合成种子 / 异常数据，一个客户授权沙箱检验陌生语义，两轨采用相同写入禁止清单。

**导入与归一化**：G0 绑定账号、环境、范围、存储与保留策略，`fde observe import` 仅接 HAR / Playwright trace / DDL / API 文档，先规则遮罩、占位 / 一致伪名与白名单，再可选模型复检。从导入 trace 提取 ev_before、点击、提交响应和后态；无足够证据时保留缺口，不临时启动实时采集。示例路径归一为 `POST /expenses/{id}/submit`，样例值 4 与“待审核”组成候选 claim；头表 / 明细关系注明明确键或推断 Join。

**语义与对齐**：`fde graph` 校验同一图与双投影；工程 selector 与业务状态共享证据。问题卡确认附件复用，排除未确认退回原因。G1 固定辅助填报工作流 + HTML 工具，明确只创建 / 修改草稿、附件、提交；禁止审批通过、付款、删除、批量、通知，基准有测试角色可模拟退回。

**候选与执行**：`fde plan` 固定完整状态图和 hash；G2 检查前后态、API / UI 等价授权、失败边和核验。每个 R2 写入通过 `fde approve` 获取逐次批准：基准 L0 可信回执，客户沙箱 L1 绑定 E-001 / 动作 / 参数 / 环境指纹 / 有效期 / 一次使用并本地签名。调用 execute 前必查指纹与权限，再消费令牌；observe 独立读回状态，超时只进 UNKNOWN_OUTCOME，不重复提交，聊天确认不能代替令牌。L2 只保留接口。

**验证与交付**：`fde verify` 回放缺字段、附件失效、未知状态与超时；两轨沙箱验证所选创建 / 修订 / 上传 / 提交路径，基准模拟退回。每轮通过环境管理员已授权的重置 / 清理流程恢复基线，记录 CleanupRecord 的环境、轮次、方式、执行者、结果和残留；不得调用禁用删除来“清理”。ValidationRecord 分离模拟和真实沙箱证据。业务专家 / FDE 工程师分别签署，G4 检查运维责任人与告警渠道、日志 / 恢复 / 停止说明；AssetManifest 固定 0.3.0 内容 hash 与限定 applicability，生产不自动开放。

**运维与复用**：每次输出标准 RunSummary，MVP 手动 check 且执行前强制指纹检查。响应漂移则 BLOCKED，资产 STALE，当前运行停止转人工；重新导入、问题卡、沙箱复验后派生 0.3.1 并取切换批准，无关只读资产可独立检查。相似任务经结构化过滤 → FTS5 → 适用性 → 复验 → 新批准；export / import 校验清单与 hash，恢复所有资产待复验且无旧批准；客户私有映射 / 敏感证据不进入公共模板或共享 Git。

## 16. 自测、MVP 实施与质量权衡

### 16.1 可执行验收矩阵

| 测试主题 | 必须证明的行为 |
|---|---|
| 预算与渐进披露 | 超预算拆包、关键风险不丢；不默认读入全量 DDL / HAR；证据可按需展开 |
| 导入与归一化 | 仅四类来源；规则遮罩先于模型 / 正式落盘；一致伪名不跨项目；部分导入有标记，同 URL 不同动作不误合并，无 FK 不伪造约束 |
| 业务能力图 | 两视图同 ID / 版本 / 来源；技术验证不替代业务确认；冲突生成问题卡 |
| 工作流静态检查 | 未处理失败边、无限循环、缺核验 / gate 的候选无法过 G2 |
| 批准与权限 | 聊天非批准；L0 / L1 风险下限、签名 / 重放 / 越界 / 过期 / 换环境 / 计划变化 / 权限撤回均拒绝；恢复不还原，模式不扩权，禁止清单不可豁免 |
| 工作记录与交接 | 仅凭工作编号和有效权限在新会话恢复，越权读取被拒绝。换源后保留仍适用的决定。并发创建与更新均有冲突保护 |
| 未知结果与恢复 | 注入超时或崩溃后先读状态；不重复非幂等提交；取消不声称回滚 |
| AssetStore | WAL / FTS5；正文不进数据库；文件索引可重建；CAS 冲突不覆盖；revision 可追溯；最新 SUSPENDED 不召回旧 VALIDATED；跨 scope 无泄漏 |
| 召回与漂移 | 先过滤再排序；变化传播到相关资产；新证据前不自动恢复有效；旧批准不复用 |
| 运维与生命周期 | check 幂等且无业务写入；执行前指纹失败停止转人工；无持续来源显示未启用；任务临时区结束清理，墓碑 / 加密粉碎与重建不复活内容 |
| 交付 | 双轨每轮清理留记录；模拟 / 沙箱 / 业务 / 工程证据分离；两类签署、运维责任人与告警必填，版本 / 限制可追溯 |
| 加密与迁移 | 钥匙串不可用不写明文；包路径 / Schema / hash 异常拒绝；export 不含原始敏感观测 / 密钥 / 令牌；import 全部待复验；敏感证据不入共享仓库 |

核心单元测试使用合成导入材料、临时 SQLite 和假宿主；MVP 端到端验收必须在可重置开源 ERP 基准及一个授权客户沙箱双轨完成，不以纯模拟替代。负向故障仅在获授权隔离范围注入，每轮重置 / 清理留记录，不对生产制造故障。统计接口、胶水、调用、上下文、交互与复验成本，按 PRD 第 15 节收集基线，不把代码样例行数当性能承诺。

### 16.2 对齐产品里程碑

M0 固定 JSON Schema、Python / 薄 TS 与 CLI / MCP 对应，核实首宿主四项能力，测试 L0 / L1 / 禁止清单，落实双轨授权及重置；M1 接入 Level 0 和四类文件导入，验证遮罩 / 分层保留 / 模式配置，测量摘要及问题卡；M2 加限定双轨沙箱的 Level 1，执行前检查与逐次批准，覆盖成功 / 失败 / 未知结果、每轮清理及双签门禁；M3 第二次召回 / FTS5 / 复验、幂等 check / RunSummary、墓碑删除 / 密钥粉碎 / 重建、导出恢复待复验与退役演练。MVP 不包含常驻监控、实时录制或本地 HTTP。

优先证明同一流程在两模式具有一致事实和安全约束，再扩展第二宿主或连接器。第二阶段再评审受控浏览器、向量索引、客户周期 / 事件接入、L2 首审批渠道、客户自托管 Git 共享和生产限定执行，不提前引入重依赖；MVP 包含 export / import，不能把必要恢复工具推迟。

### 16.3 架构取舍

薄协议减少宿主胶水，但核心仍须完成 schema、门禁和状态检查；不能把这些责任省略到提示词。结构与文本召回易部署，但候选少时收益需测量；向量可增强排序，不增强授权。证据与固定版本增加元数据，但可减少重复探查和故障定位成本。端侧优先保护控制权，但备份、多设备和高可用仍是客户决策。

不构建运行时意味着无法仅靠 Harness 保证持续运行、外部原子性或组织权限；交付记录必须明确宿主 / 执行器的责任。强模型可以更快提出候选，确定性采集和门禁则把能力转成可检查的工程结果，两者互补。

## 17. 已采纳决策摘要与兼容约定

决策依据：[技术选型决策记录 D01–D08](./FDE_Harness_Decision_Record_D01-D08.md)。原记录保持只读；本版将其视为已采纳建议，状态为**建议采纳，待试点确认默认参数**，不是已发布 SDK、完成 POC 或生产认证。

| 编号 | 已采纳实施路径 | 本篇落点 |
|---|---|---|
| D01 | 单 Skill / preferences 优先级、可选 stdio MCP / 等价 CLI、四项宿主条件 | 第 1、2、8、11 节 |
| D02 | JSON Schema 源头、Python / 薄 TS、共享 handler、WAL / FTS5 与内容文件 | 第 1、6、8、9、12、13 节 |
| D03 | 分层保留 / 生命周期清理、元数据追加审计、墓碑删除与加密粉碎 | 第 4、11、13、14、16 节 |
| D04 | 双轨 / 固定种子、允许与禁止清单、逐轮重置、分别签署验收 | 第 3、7、10、15、16 节 |
| D05 | MVP 四类导入、规则优先遮罩、第二阶段受控浏览器 / 非默认 MITM | 第 4、8、15、16 节 |
| D06 | L0 / L1 签名与消费、L2 接口、失效 / 恢复 / 文本边界 | 第 7、8、10、15、16 节 |
| D07 | 无常驻、幂等 check、执行前必查、RunSummary、责任人 / 告警门禁 | 第 1、3、8、10、14、16 节 |
| D08 | 磁盘 / 应用加密、钥匙串、export / import 待复验、后续 Git 共享 | 第 8、11、12、13、16 节 |

### 17.1 兼容约定

本版 `schema_version=0.3` 是参考契约标识，不表示已发布兼容 SDK。所有产物及 CLI / MCP 请求响应从同一 JSON Schema 生成 / 校验；新增可选字段保持读取兼容，必填字段、状态或安全语义变化提升契约版本并迁移。未知主要版本拒绝自动恢复，未知权限 / 环境 / 批准无法校验时阻塞，不以配置默认值固化生产授权。默认保留 / 检查参数只有经试点和客户策略确认才生效。

### 17.2 剩余开放问题

与 PRD 第 17.1 节共享待办，不重开已采纳 D01–D08：首个宿主产品及可信确认回执实测；分层 TTL / L1 有效窗口 / 归档阈值 / 检查频率的建议默认值试点确认；开源 ERP、种子与客户沙箱重置方案；首个 L2 审批渠道与回执；桌面客户端采集、受控浏览器的 SSO / 安全可行性；Node 系宿主占比及 Python 分发痛点是否触发 TS 核心迁移；宿主外执行器 RunSummary 通道 / 停止能力和共享仓库 ACL 的客户验证。

