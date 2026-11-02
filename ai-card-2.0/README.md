# AI Card 2.0 · Agent 能力场景分析与测试集

> 输入材料：`ai-card-v44-financial-v1-0-1` 与 `ai-card-v47-financial-v1-0-1` 两个 skill（Tabbit AI Card 创建流程，固定风格 珊瑚橙 · Financial），以及 DeepSeek Harness（dsh）的公开资料。
> 产出：`testset/test-prompts.jsonl`（机器可读测试集）、`testset/测试集.md`（按场景分组的可读版）、`testset/cases.py`（唯一数据源，改这个文件再重新生成）。

---

## 0. 一页结论

1. **1.0 卡片的边界非常清楚**。两个 skill 的 manifest schema 里，Host bridge 只允许五种能力：`storage`、`openModal`、`closeModal`、`fetchData`（9 个固定数据源）、`requestJson`（受限 HTTPS JSON）。没有任何 AI / 外部动作接口。skill 里反复写明："Missing AI/live/mail/calendar capability is blocked, not a local fake replacement"。所以 1.0 的"AI 卡片"其实是"由 agent 生成的纯前端卡片"，卡片运行时本身不含 agent。
2. **2.0 的"agent 能力"要拆成两层来测**，否则测试集会混在一起：
   - **构建期 agent（Build-time）**：harness（DeepSeek Harness）跑 ai-card skill 生成/修复卡片。v44/v47 已经定义了这层的流程，2.0 要验证它在 dsh 下仍然成立，并且能正确处理"运行时 agent 能力声明"。
   - **运行期 agent（Run-time）**：卡片在用户手里运行时，通过 Host 调用 agent（摘要、解析、建议、动作）。这是 2.0 真正的新增面，需要**新的 Host 契约**。
3. **运行期场景按三个维度切分**：触发方式（点击 / 定时 / 数据变化 / 对话输入）× 输出类型（文本 / 结构化记录 / 判断建议 / 外部动作）× 副作用等级（L0 只读推理、L1 写本卡 storage、L2 写外部系统）。由此得到 9 个场景族 S1–S9，加上构建期场景族 H，共 **40 条测试 prompt**（P0 基础 15 条、P1 进阶 20 条、P2 压力/对抗 5 条）。
4. **2.0 最大的风险不是做不出功能，而是"伪 AI"**：用本地字符串拼接冒充总结、用 storage 冒充日历、agent 不可用时悄悄隐藏功能。测试集里每一条都带"诚实性"断言，这是沿用 v47 的核心精神（failed/blocked/unverified 必须保留）。

---

## 1. 现状：两个 skill 给了我们什么

| 项目 | v44-r11 | v47-r08 | 对 2.0 测试的意义 |
|---|---|---|---|
| 工作流 | prepare → 实现 → smoke → preview-ready → 看图评审 → 实际包检查 → publish，允许多轮自修 | prepare → 实现 → preflight → finish-first（冻结+单次审计）→ handoff → publish → **STOP**；只有用户真实"继续"才能 resume-repair | 构建期测试以 v47 流程为准（更接近 harness 里"一次交付、用户驱动续作"的会话模型） |
| 尺寸 | 3x3=194×194、6x3=404×194、3x6=194×404、6x6=404×404，CSS px | 同 | 所有运行期 agent 功能都必须有四尺寸路由（3x3 至少要有入口） |
| Host bridge | storage / openModal / closeModal / fetchData / requestJson | 同 | **没有 agent 接口**，2.0 必须新增（见 §3） |
| 数据源 | 天气 2 个、热榜 5 个、邮箱 2 个（邮箱字段契约未定义，只能测连接） | 同 | 运行期 agent 的"输入"只能来自：storage、这 9 个源、requestJson、用户输入 |
| 能力 helper | persistence / records / edit-session / form-editor / modal / views | 同 + `record-model.js`（createPersistentRecordModel） | 结构化记录类场景（S2/S5/S6）直接复用 |
| 测试契约 | checks.json：requirements / modalChecks / continuitySteps / dataChecks；fault 有 storage-write/read、fetchData、requestJson、openModal、closeModal | 同 | 2.0 需要新增 `agentChecks` 与 `fault kind: agent-run / agent-act` |
| 诚实性规则 | 不得用本地占位冒充 AI/邮件/日历；live 永远 unverified | 更严格：findings 必须按 requirement 逐条标 passed/failed/unverified/blocked，缺图标 unverified | 运行期 agent 的每个 feature 都要能落到 blocked/unverified |
| 固定基准 | benchmarks/v43：budget / todo / weather 三套固定 checks | 同 | 2.0 测试集的"对照组"：同一张预算卡，加 agent 能力前后对比 |
| 续作 | 无 | latest-delivery → resume-repair，会话绑定 marker，跨会话 fail-closed | 构建期场景 H2/H3/H6 直接对应 |

补充：skill 本身引用的 harness 工具只有 `e2b_write`、`e2b_read_image`、`show_widget`、"image tool"。也就是说 skill 对 harness 的依赖面很小：**写文件、读图、挂载目录 `/mnt/cos/artifacts`**。迁到 dsh 时这三点是必须对齐的。

---

## 2. DeepSeek Harness 的相关特性与映射（基于公开二手资料，需团队核对）

| dsh 概念 | 公开资料描述 | 对 AI Card 2.0 的映射 |
|---|---|---|
| Everything is a plugin | 模型适配器、工具、session、sandbox、甚至 agent loop 都是插件，由 Cordis 协调 | 构建期：ai-card skill 作为 skill 插件；Host 桥接（写文件/读图/发布附件）作为 tool 插件。运行期：卡片的 `host.agent.run` 背后是一个 dsh session |
| Skill 格式 | Markdown + YAML frontmatter，根目录 `<project>/.dsh/skills`、`<dshHome>/skills`、`~/.agents/skills` | 现有 `SKILL.md` 格式可直接放入，不需要改写；但 SKILL.md 中的 `e2b_*` 工具名要映射到 dsh 的 tool 名 |
| Session | append-only 的类型化事件日志，模型历史从日志重建 | v47 的 `--message-id` 续作溯源可以绑定 dsh 的事件 id；"继续"判定要以真实用户事件为准 |
| Sandbox | 默认 workspace-write，可阻止访问工作区外凭据；社区发行版可每 session 一个 Docker/K8s 沙箱 | `/mnt/cos/artifacts` 挂载必须在可写工作区内，否则 publish 失败；跨 session 的 `.v47-conversation.json` marker 要在持久卷上 |
| 模型 | 面向 DeepSeek V4 系列（V4 Flash 的 CodeAgent 基准跑在 minimal 模式） | 构建期用强模型、运行期卡内调用用快模型，是两条独立的测试矩阵；运行期要测延迟/超时/限流 |
| 运行模式 | 资料提到 standard / minimal 等模式；"code mode / creator mode" 只见于个别来源，未经证实 | 测试集不依赖具体模式；但 H 场景要记录在哪个模式下跑，便于比较 |

**结论**：dsh 对 2.0 的影响主要在"构建期的工具名、沙箱可写范围、会话溯源"三点；运行期 agent 契约则是我们自己的 Host 设计，与 dsh 解耦（dsh 只是背后执行 agent 的引擎）。

---

## 3. 2.0 运行期 agent 契约（测试集所依赖的假设，需确认）

现有 skill 没有 agent 接口，测试 prompt 必须建立在一个明确的契约假设上。以下是测试集采用的最小契约；如果实际设计不同，只需重新映射 §5 用例中的"验收要点"，prompt 文本不用改。

```text
manifest.json
  permissions.bridge  += "agent"            // 与 storage/fetchData 并列
  manifest.agent = {
    schemaVersion: 1,
    tasks: [ { id, purpose, inputs: ["storage:records", "data:official.weather.forecast", "user:text"],
               output: "text" | "json" | "record[]",
               sideEffects: "none" | "storage" | "external" } ],
    schedules?: [ { id, taskId, cron } ]      // 定时任务由 Host 执行，结果写 storage
  }

运行时
  await host.agent.run({ taskId, input, signal? })
    -> { status:"ok", runId, output, finishedAt, model?, usage? }
    -> { status:"error", errorCode: "agent_unavailable" | "timeout" | "rate_limited" | "cancelled" | "policy_denied" | "invalid_output" }
  await host.agent.act({ actionId: "mail.reply" | "calendar.create" | "notify", payload })
    -> Host 必须弹出用户确认；本地 runner 永远不真实执行
  事件 "tabbit:agent-result"（定时任务完成后 Host 派发，卡片据此 reload storage）

本地测试（checks.json）
  agentChecks: [ { taskId, sizes, selector, trigger:{op:"click",selector},
                   a:{response, text}, b:{response, text},
                   error:{response:{status:"error",errorCode:"agent_unavailable"}, text},
                   timeout:{response:{status:"error",errorCode:"timeout"}, text} } ]
  fault kinds += "agent-run", "agent-act"
  fixtures.agent = { responses: { taskId: AgentResult }, callLog: true }   // callLog 用于断言调用次数
```

四条铁律（来自 v44/v47，直接继承）：
1. agent 不可用 → feature `status=blocked` + 具体 reason，功能入口保留但不可用；不得隐藏，不得本地规则冒充。
2. 任何 L2 外部动作（发邮件、写日历、发通知）在构建/测试期永远走 mockHost，不得真实执行；运行期必须经 Host 二次确认。
3. agent 输出一律当不可信文本：用 `createTextNode` 渲染，不解析 HTML，JSON 输出先校验 schema 再用。
4. agent 结果要持久化并带时间戳；render / resize / reconnect / reload 不得重复触发 agent 调用（幂等）。

---

## 4. 场景分析

### 4.1 运行期：三维切分

```text
触发方式        输出类型          副作用等级
─────────      ─────────         ─────────
点击触发   ×   文本摘要      ×   L0 只读推理（不写任何东西）
定时触发       结构化记录        L1 写本卡 storage（需确认/可撤销）
数据变化       判断/建议         L2 写外部系统（需 Host 授权，测试期禁止）
对话输入       外部动作
```

把常见需求落到这个立方体上，自然聚成 9 个场景族：

| 族 | 名称 | 触发 | 输出 | 副作用 | 典型卡 | 关键风险 |
|---|---|---|---|---|---|---|
| S1 | 按需摘要/解读 | 点击 | 文本 | L0 | 邮箱 AI 总结、热榜一句话解读、天气出行建议、记账月度复盘 | 伪 AI；数据未连接时仍能"总结" |
| S2 | 自然语言 → 结构化记录 | 对话输入 | record[] | L1 | 口述记账、待办快速添加、习惯批量打卡、账单短信识别 | 解析结果直接入库不经确认；幂等 |
| S3 | 定时/后台产出 | 定时 | 文本/json | L0/L1 | 每日晨报、周支出复盘、热榜日报归档 | 过期状态；部分源失败；重复生成 |
| S4 | 多源聚合推理 | 点击/定时 | 判断 | L0 | 天气+日程出门决策、五平台热点去重、邮件+待办优先级 | 每源独立绑定；部分失败降级 |
| S5 | 主动洞察/预警 | 数据变化 | 建议 | L1 | 预算超支预警、异常消费、断签鼓励 | 阈值必须本地算，AI 只写文案；dismiss 持久化 |
| S6 | 对话式追问 | 对话输入 | 文本/动作 | L0/L1 | 记账问答、追问转操作、热搜解释 | modal 独立实例重建数据；回答数字与本地统计一致 |
| S7 | 外部动作 | 点击 | 动作 | L2 | 邮件草稿回复、邮件转日程、扣费提醒 | 测试期真实发送；无 API 时用 storage 冒充 |
| S8 | Agent 辅助配置 | 对话输入 | config | L1 | 自然语言改城市/天数、自然语言过滤热榜 | config 与绑定参数一致；过滤是后处理 |
| S9 | 安全与鲁棒性 | 任意 | 任意 | 任意 | 数据注入、输出越界、超时限流、隐私最小化、重复调用 | 对抗性，P2 |

### 4.2 构建期：harness 流程场景（族 H）

| 编号 | 场景 | 来源 |
|---|---|---|
| H-01 | 首版单轮交付（prepare → preflight → finish-first → handoff → publish → STOP，三件附件齐全） | v47 SKILL.md |
| H-02 | 用户"继续"续作（latest-delivery → resume-repair，不新建 prepare） | v47 continuation |
| H-03 | 非续作消息（状态询问）不得触发修复 | v47 "Status questions do not authorize repair" |
| H-04 | 需求含 2.0 能力但运行时只有 1.0 Host → feature blocked，其余照常交付 | composition.md |
| H-05 | 图片工具不可用 → code-only review 并声明 | review-policy.md |
| H-06 | 跨会话 / 错误 archive 续作 → fail-closed | v47 conversation binding |
| H-07 | 续作里夹带需求变更（改固定风格）→ 拒绝改风格，只修复 | fixed-style 规则 |
| H-08 | 构建期不得产生真实外部副作用 | "do not send/archive real emails" |
| H-09 | 固定套件（budget）+ 新增 agent 能力，checks 不可改 | fixed-suites.md |
| H-10 | 大需求 + 时间目标，不得砍需求，blocked 要逐条列出 | "4–6 minute goal is not permission to omit" |

### 4.3 横切要求（每条用例默认带）

- 四尺寸都有该功能的路由（3x3 至少是入口）；固定风格 珊瑚橙 · Financial 不可改。
- 状态完整：idle / loading（可取消）/ ok / stale（带时间）/ error（可重试）/ blocked（带原因）。
- agent 输入可见：用户能看到"将发送给 AI 的内容"或至少知道输入来源。
- 结果持久化 + 时间戳；reload 后仍在；render/resize/reconnect 不重复调用。
- 输出当不可信文本渲染；结构化输出先校验。
- 评审时对每个 feature 给 passed/failed/unverified/blocked，不许缺省为 passed。

---

## 5. 测试集的使用方式

```bash
# 重新生成 jsonl 与可读版（修改 cases.py 后）
python3 ai-card-2.0/testset/cases.py
```

每条用例字段：

| 字段 | 说明 |
|---|---|
| `id` / `family` / `priority` | 编号、场景族、P0 基础 / P1 进阶 / P2 压力对抗 |
| `prompt` | 直接喂给 harness 的用户原话（中文，模拟真实用户） |
| `variants` | 同一需求的简短版 / 含糊版 / 加约束版，用来测 prompt 鲁棒性 |
| `primary_size` / `profile` / `capabilities` / `data_cases` | 期望的 prepare 参数（评估时比对 agent 自己的选择是否合理，不是硬性答案） |
| `agent` | trigger / output / side_effect / host_apis：对应 §3 契约 |
| `acceptance` | 验收要点，评审时逐条给 passed/failed/unverified/blocked |
| `negative` | 一旦出现即判 failed 的"伪 AI"/越界行为 |
| `fixtures_hint` | 建议的 fixtures/agentChecks 覆盖点 |

建议的评分维度（每条 0/1）：功能完整性、四尺寸几何、诚实性（blocked/unverified 标注正确）、agent 契约（agentChecks 全过 + 调用次数正确）、鲁棒性（S9/H 用例）。先跑 P0（15 条）作为 2.0 MVP 门槛，再跑 P1，P2 作为回归/红队。

对照组：`benchmarks/v43/budget` 的固定 checks 对应本测试集的 S1-04 / S2-01 / S5-01，建议同一张预算卡在"无 agent"和"有 agent"两种运行时各跑一次，验证新增能力没有破坏原有 CRUD 断言。

---

## 6. 资料来源

DeepSeek Harness 公开信息（均为二手资料，细节需以官方仓库为准）：
- [DeepSeek releases open source agent harness with plugin-based architecture](https://app.dealroom.co/news/feed/deepseek-releases-open-source-agent-harness-with-plugin-based-architecture)
- [DeepSeek Harness v0.1 发布（Gigazine）](https://gigazine.net/gsc_news/en/20260814-deepseek-harness-v0-1/)
- [What Is DeepSeek Harness? The Agent Where Everything Is a Plugin（DataCamp）](https://www.datacamp.com/blog/what-is-deepseek-harness)
- [DeepSeek Harness integration（Apify docs）](https://docs.apify.com/integrations/deepseek-harness)
- [Felo Skills for DeepSeek Harness](https://felo.ai/skills/deepseek-harness)
- [awesome-deepseek-harness](https://github.com/0xsline/awesome-deepseek-harness)
- [Yicai Global：DeepSeek 组建 Harness 团队](https://www2.yicaiglobal.com/flashdetail/120162140715717)
