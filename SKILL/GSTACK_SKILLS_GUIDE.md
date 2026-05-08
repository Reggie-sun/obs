# GStack Skills 使用指南

  

> 所有 skill 通过 `/skill名` 调用，例如 `/investigate`、`/qa`、`/review`。

> 调用后 skill 会自动加载并引导流程，你只需按提示回答问题。

> 推荐模型速记：`gpt = 判断/规划/审查`，`codex = 编码/测试/执行`，`glm = 轻量工具/状态切换`。

  

---

## 零、GitHub 这波已同步的 gstack skill 更新

> 当前本地 gstack 源已在 `main`，最新看到的是 `v1.20.0.0`。

### 1. 浏览器 skill 这条线更新最大

- `/gstack` 根 skill 现在更像“**路由器 + 浏览器默认面**”：命中意图时会转到 `/office-hours`、`/qa`、`/review`、`/context-save` 等专项 skill；没命中更窄 skill 时，自己仍落到 browse/headless QA
- `/browse` 还是无头浏览器主入口，适合 QA、截图、点页面、验证状态
- `/open-gstack-browser` 现在不只是“可见浏览器”，而是 **GStack Browser**：带 sidebar、stealth、实时看 agent 操作
- `/pair-agent` 是把你的浏览器共享给另一个 AI agent，不只是“远程看页面”

### 2. 新增了 `/scrape` + `/skillify` 这一组浏览器技能流

- `/scrape`：从网页拉数据，**只读**；第一次会先 prototype 流程，返回 JSON
- `/skillify`：把最近一次成功的 `/scrape` 流程固化成永久 browser skill
- 结果就是：第一次慢一点，后面同类抓取会变成接近 `200ms` 的复用流程
- 会写页面、会提交表单的自动化不属于 `/scrape`；那是后续 `/automate` 路线

### 3. gbrain 相关现在有正式入口了

- `/setup-gbrain` 已经是正式 skill
- 它负责 gbrain CLI、brain 初始化、MCP 注册、per-remote trust policy
- 常见扩展入口包括：`--repo`、`--switch`、`--resume-provision`、`--cleanup-orphans`

### 4. 保存/恢复上下文现在有新版命名

- `/context-save`：保存当前工作上下文
- `/context-restore`：恢复之前保存的上下文
- 这两个是新版正式入口
- `/checkpoint` 这类旧叫法在某些宿主里会被原生命令抢占，所以现在更推荐记新版名

### 5. 升级 gstack 时，已经会顺手迁移部分新能力

- `/gstack-upgrade` 不只是拉最新版
- 现在还会处理部分迁移，比如 gbrain wireup 相关的一次性迁移
- 所以“功能不对劲但你之前用过旧版 gstack”时，先想一下是不是该跑 `/gstack-upgrade`

---

  

## 一、调试与排查

  

### /investigate — 系统化调试

**推荐模型：** `gpt`

  

**什么时候用：** 遇到 bug、500 错误、奇怪行为、"昨天还好好的"

  

**会改代码吗：** 会，但必须先找到根因才能改

  

**工作流程：**

1. 收集症状（读报错、堆栈、复现步骤）

2. 读代码追踪数据流

3. 检查 git log 看最近改动

4. 尝试复现

5. 形成根因假设

6. 验证假设（加临时日志/断言）

7. 假设错误 → 重新收集证据，3 次失败后停下来问你要不要升级

8. 确认根因后写最小修复 + 回归测试

9. 跑全量测试

  

**铁律：** 没有根因就不修复。3 次假设失败 → 停下来。

  

**prompt 示例：**

```

/investigate

qwen3-rerank 的 DashScope endpoint 返回 403 错误，

影响检索排序质量。排查是 API key 问题、模型名错误还是 endpoint 路径问题。

```

  

---

  

## 二、质量保证

  

### /qa — 系统化测试 + 自动修复

**推荐模型：** `codex`

  

**什么时候用：** 功能准备好了要测试、"这能用吗？"

  

**会改代码吗：** 会。发现 bug 后自动修复、提交、重新验证

  

**三个级别：**

- Quick：只测 critical/high

- Standard：+ medium

- Exhaustive：+ cosmetic

  

**输出：** 修复前后健康评分对比、修复证据、发布就绪摘要

  

**prompt 示例：**

```

/qa

测试 http://localhost:3000 的文档上传和聊天检索功能，

关注跨部门检索和 supplemental 触发逻辑

```

  

### /qa-only — 只报告不修复

**推荐模型：** `codex`

  

**什么时候用：** "帮我看看有什么 bug"、"出个报告就行"

  

**会改代码吗：** 不会。只出报告

  

**输出：** 健康评分、截图、复现步骤

  

**prompt 示例：**

```

/qa-only

检查 http://localhost:3000 的 SOP Center 和 Logs 页面

```

  

---

  

## 三、代码审查

  

### /review — PR 前代码审查

**推荐模型：** `codex`

  

**什么时候用：** "review 一下"、"检查我的 diff"、"要合并了"

  

**会改代码吗：** 可能会小修，主要是出审查意见

  

**关注点：** SQL 安全、LLM 信任边界、条件副作用、结构性问题

  

**prompt 示例：**

```

/review

审查 feature/retrieval-priority 分支的改动，

重点看 retrieval_service.py 和 retrieval_query_router.py

```

  

### /codex — 外部第二意见

**推荐模型：** `codex`

  

**什么时候用：** "第二意见"、"让 codex 看看"

  

**三个模式：**

- review：独立的 diff 审查，pass/fail 门控

- challenge：对抗模式，试图打破你的代码

- consult：问 codex 任何问题

  

**prompt 示例：**

```

/codex review

审查当前分支的 retrieval 改动

```

  

---

  

## 四、计划审查（Plan Reviews）

  

> 这些 skill 都是**只读审查**，输出修改建议，不会直接改计划文件。

  

### /plan-eng-review — 工程视角审查计划

**推荐模型：** `gpt`

  

**什么时候用：** 有个计划要开始编码前、审查架构合理性

  

**关注：** 架构、数据流、边界情况、测试覆盖、性能

  

**prompt 示例：**

```

/plan-eng-review

审查 RETRIEVAL_OPTIMIZATION_PLAN.md，关注：

1. Phase 1-4 的执行顺序是否合理

2. chunk 优化和 supplemental 校准的依赖关系

3. 测试覆盖和回归策略是否充分

```

  

### /plan-ceo-review — 老板视角审查计划

**推荐模型：** `gpt`

  

**什么时候用：** "格局够不够大"、"范围要不要扩展"

  

**四个模式：**

- SCOPE EXPANSION：放开了想

- SELECTIVE EXPANSION：保持范围，挑选扩展点

- HOLD SCOPE：最大严谨度审查

- SCOPE REDUCTION：砍到最小

  

**prompt 示例：**

```

/plan-ceo-review

用 HOLD SCOPE 模式审查 RETRIEVAL_OPTIMIZATION_PLAN.md

```

  

### /plan-design-review — 设计视角审查计划

**推荐模型：** `gpt`

  

**什么时候用：** 计划有 UI/UX 组件，编码前审查

  

**关注：** 每个设计维度打 0-10 分，解释怎么到 10 分

  

**prompt 示例：**

```

/plan-design-review

审查前端重构计划中的 UI 组件设计

```

  

### /plan-devex-review — 开发体验审查计划

**推荐模型：** `gpt`

  

**什么时候用：** 计划涉及 API、CLI、SDK、开发者工具

  

**三个模式：**

- DX EXPANSION：竞争优势

- DX POLISH：打磨每个触点

- DX TRIAGE：只看关键缺口

  

### /autoplan — 自动跑全部审查

**推荐模型：** `gpt`

  

**什么时候用：** "把所有审查自动跑一遍"

  

**做了什么：** 自动执行 CEO + 设计 + 工程 + DX 审查，自动决策，只把需要你判断的提交给你

  

**prompt 示例：**

```

/autoplan

自动审查 RETRIEVAL_OPTIMIZATION_PLAN.md

```

  

---

  

## 五、浏览器操作

  

### /browse — 无头浏览器

**推荐模型：** `codex`

  

**什么时候用：** "打开浏览器看看"、"截个图"、"测试页面"

  

**会改代码吗：** 不会。只操作浏览器

  

**能力：** 导航、点击、截图、读页面内容、填表单、对比前后状态

**补充：** 这是当前 gstack 浏览器族的主入口；如果你想看 agent 实时操作过程，用 `/open-gstack-browser`

  

**prompt 示例：**

```

/browse

打开 http://localhost:3000，截图首页，然后导航到 SOP Center 页面再截图

```

  

### /open-gstack-browser — 可见浏览器

**推荐模型：** `glm`

  

**什么时候用：** 想看浏览器操作过程、想用带 sidebar 的 GStack Browser

  

**区别：** `/browse` 是无头的；这个会打开可见的 GStack Browser，带 sidebar、stealth 和实时操作可视化

### /scrape — 从页面拉数据

**推荐模型：** `codex`

**什么时候用：** "帮我抓这个页面的数据"、"提取网页内容"、"把页面信息变成 JSON"

**会改代码吗：** 不会。只读抓取

**特点：**

- 首次同类 intent 会先 prototype
- 成功后会建议你用 `/skillify`
- 后续同 intent 可以走已固化的 browser skill，速度会快很多

### /skillify — 把抓取流程固化成 skill

**推荐模型：** `codex`

**什么时候用：** `/scrape` 成功后，想把这次流程永久保存下来复用

**会改代码吗：** 会生成 skill 文件，但会先测试并过确认门

**做了什么：**

- 回看最近一次成功的 `/scrape`
- 合成脚本、测试和 fixture
- 先写到临时目录验证
- 测试通过并确认后，才正式落盘成永久 browser skill

  

### /setup-browser-cookies — 导入 cookies

**推荐模型：** `glm`

  

**什么时候用：** 测试需要登录的页面

  

**做了什么：** 从你的真实浏览器导入 cookies 到无头浏览器

  

---

  

## 六、发布流程

  

### /ship — 发布工作流

**推荐模型：** `codex`

  

**什么时候用：** "发布"、"deploy"、"创建 PR"、"push to main"

  

**流程：** 检测基线分支 → 合并 → 跑测试 → 审查 diff → 更新版本号 → 更新 CHANGELOG → 提交 → 推送 → 创建 PR

  

**prompt 示例：**

```

/ship

把 feature/retrieval-priority 发布到 main

```

  

### /land-and-deploy — 合并并部署

**推荐模型：** `codex`

  

**什么时候用：** PR 已创建，要合并到生产

  

**流程：** 合并 PR → 等 CI → 部署 → canary 检查验证生产健康

  

### /setup-deploy — 配置部署

**推荐模型：** `codex`

  

**什么时候用：** 第一次使用 /land-and-deploy 前

  

**做了什么：** 检测你的部署平台（Fly.io、Render、Vercel 等），配置写入 CLAUDE.md

  

---

  

## 七、代码健康

  

### /health — 代码健康仪表盘

**推荐模型：** `codex`

  

**什么时候用：** "代码质量怎么样"、"跑一下检查"

  

**会改代码吗：** 可能会小修

  

**做了什么：** 跑类型检查、linter、测试、死代码检测，输出 0-10 综合评分，追踪趋势

  

**prompt 示例：**

```

/health

检查 backend 代码健康度

```

  

### /benchmark — 性能基准

**推荐模型：** `codex`

  

**什么时候用：** "性能怎么样"、"页面加载速度"、"Core Web Vitals"

  

**会改代码吗：** 不会。只出报告

  

**做了什么：** 页面加载时间、Core Web Vitals、资源大小，PR 前后对比

  

---

  

## 八、设计与 UI

  

### /design-review — UI 视觉审查

**推荐模型：** `gpt`

  

**什么时候用：** "看起来怎么样"、"视觉 QA"、"设计打磨"

  

**会改代码吗：** 会。发现视觉问题后自动修复

  

**关注：** 视觉不一致、间距问题、层级问题、AI 套路化设计、慢交互

  

### /design-shotgun — 生成多个设计变体

**推荐模型：** `gpt`

  

**什么时候用：** "给我看看几个方案"、"不知道要什么样子"

  

**会改代码吗：** 不会。生成对比展示板

  

### /design-consultation — 设计系统

**推荐模型：** `gpt`

  

**什么时候用：** 新项目没设计系统时、"定一下品牌规范"

  

**输出：** 创建 DESIGN.md（字体、颜色、布局、间距、动效）

  

### /design-html — 生成生产级 HTML

**推荐模型：** `codex`

  

**什么时候用：** 设计定稿后要转成代码、"把这个设计实现出来"

  

---

  

## 九、安全

  

### /cso — 安全审计

**推荐模型：** `gpt`

  

**什么时候用：** "安全审计"、"威胁建模"、"OWASP"、"漏洞扫描"

  

**会改代码吗：** 可能会

  

**两个模式：**

- daily：快速扫描，只报告高置信度问题

- comprehensive：月度深度扫描

  

**关注：** 密钥泄露、依赖供应链、CI/CD 安全、LLM/AI 安全、OWASP Top 10、STRIDE 威胁建模

  

---

  

## 十、安全模式

  

### /careful — 危险操作警告

**推荐模型：** `glm`

  

**什么时候用：** 操作生产环境时、怕误操作

  

**做了什么：** 执行 rm -rf、DROP TABLE、force-push 等危险命令前先警告你

  

### /freeze — 限制编辑范围

**推荐模型：** `glm`

  

**什么时候用：** 调试时防止改到不相关的代码

  

**做了什么：** 只允许编辑指定目录内的文件

  

```

/freeze src/auth/

```

  

### /unfreeze — 解除编辑限制

**推荐模型：** `glm`

  

**什么时候用：** 要扩大编辑范围时

  

### /guard — 最高安全模式

**推荐模型：** `glm`

  

**什么时候用：** 动生产环境、调试线上系统

  

**做了什么：** /careful + /freeze 的组合

  

---

  

## 十一、项目管理

  

### /checkpoint — 保存工作进度

**推荐模型：** `glm`

  

**什么时候用：** 要中断工作、"保存进度"、"下次从哪继续"

  

**做了什么：** 保存 git 状态、已做决策、剩余工作

**备注：** 这是旧叫法；现在更建议优先记 `/context-save` 和 `/context-restore`

### /context-save — 新版保存上下文

**推荐模型：** `glm`

**什么时候用：** "save progress"、"save state"、"context save"

**做了什么：** 保存当前工作状态，方便之后跨会话继续

### /context-restore — 新版恢复上下文

**推荐模型：** `glm`

**什么时候用：** "resume"、"restore context"、"where was I"

**做了什么：** 恢复最近保存的工作状态，继续上次的上下文

  

### /learn — 管理经验库

**推荐模型：** `gpt`

  

**什么时候用：** "之前遇到过类似问题吗"、"看看积累了什么经验"

  

**做了什么：** 查看、搜索、清理 gstack 跨会话积累的经验

  

### /retro — 周回顾

**推荐模型：** `gpt`

  

**什么时候用：** 周末、sprint 结束、"这周做了什么"

  

**做了什么：** 分析 commit 历史、工作模式、代码质量，按人分解贡献

  

### /office-hours — YC 式头脑风暴

**推荐模型：** `gpt`

  

**什么时候用：** "我有个想法"、"这值得做吗"、"帮我想想"

  

**两个模式：**

- Startup：6 个强迫性问题（需求真实性、现状、最窄切入点）

- Builder：设计思维头脑风暴

  

### /document-release — 更新文档

**推荐模型：** `codex`

  

**什么时候用：** 发布后更新文档、"同步 README"

  

**做了什么：** 读取 diff，更新 README/ARCHITECTURE/CONTRIBUTING/CHANGELOG

### /setup-gbrain — 配 gbrain

**推荐模型：** `codex`

**什么时候用：** "setup gbrain"、"connect gbrain"、"install gbrain"、"让当前 agent 能用 gbrain"

**做了什么：**

- 安装/检查 gbrain CLI
- 初始化本地 PGLite 或 Supabase brain
- 注册 MCP
- 记录 per-remote trust policy

**常见扩展模式：**

- `--repo`：只重配当前 repo 的信任策略
- `--switch`：PGLite 和 Supabase 之间切换
- `--resume-provision <ref>`：继续上次中断的 provision
- `--cleanup-orphans`：清理孤儿 Supabase 项目

  

---

  

## 十二、其他工具

  

### /canary — 部署后监控

**推荐模型：** `codex`

  

**什么时候用：** 部署后观察、"监控一下"

  

**做了什么：** 定期截图、对比部署前基线、异常告警

  

### /pair-agent — 共享浏览器给其他 AI

**推荐模型：** `glm`

  

**什么时候用：** 让另一个 AI agent 用你的浏览器

**补充：** 这是当前 gstack 的远程浏览器协作入口，适合把你本地浏览器会话通过受控方式分享出去

  

### /connect-chrome — 启动 GStack 浏览器

**推荐模型：** `glm`

  

**什么时候用：** 和 /open-gstack-browser 相同

  

### /gstack-upgrade — 升级 gstack

**推荐模型：** `glm`

  

**什么时候用：** "更新 gstack"

**补充：** 现在除了升级本体，也可能顺手跑掉部分迁移逻辑

  

---

  

## 快速参考：什么时候用什么

  

| 你想说的话 | 用这个 skill |

|-----------|-------------|

| "这坏了，帮我查" | `/investigate` |

| "测试一下这个功能" | `/qa` |

| "只看有什么 bug，别改" | `/qa-only` |

| "帮我 review 这个 PR" | `/review` |

| "让我找个第二意见" | `/codex` |

| "看看我的计划合不合理" | `/plan-eng-review` |

| "格局够不够大" | `/plan-ceo-review` |

| "UI 看起来怎么样" | `/design-review` |

| "代码质量怎么样" | `/health` |

| "帮我发布" | `/ship` |

| "安全检查" | `/cso` |

| "打开浏览器看看" | `/browse` |

| "保存进度" | `/checkpoint` |

| "这个想法值不值得做" | `/office-hours` |

| "更新文档" | `/document-release` |

| "限制只改这个目录" | `/freeze` |

| "操作生产环境要小心" | `/guard` |

  

---

  

## 正常开发的 Skill 使用顺序

  

下面按一个完整开发周期的阶段排列，说明每个阶段该用什么 skill、为什么。

  

### 阶段 0：项目初始化（一次性）

  

```

/setup-deploy → 配置部署平台，写入 CLAUDE.md

/design-consultation → 建立设计系统，生成 DESIGN.md

/health → 跑一次基线健康评分，作为后续对比参照

```

  

这一步只在项目开始时做一次。后续开发以这个基线为参照。

  

---

  

### 阶段 1：构思与计划

  

```

/office-hours → "我有这个想法，值不值得做？"

→ 输出：经过 6 个强迫性问题验证的想法文档

  

/plan-ceo-review → "格局够不够？范围要不要调？"

→ 用 HOLD SCOPE 或 SELECTIVE EXPANSION 模式

  

/plan-eng-review → "技术方案合不合理？"

→ 输出：架构建议、数据流、边界情况、测试策略

  

/plan-design-review → "UI/UX 设计对不对？"（前端相关）

→ 输出：每个设计维度 0-10 分 + 改进建议

  

/plan-devex-review → "开发者体验行不行？"（API/SDK 相关）

→ 输出：竞品对标、摩擦点、魔法时刻设计

  

/autoplan → 懒人版：自动跑上面全部审查，只把需要你决策的提交给你

```

  

**为什么这个顺序：** 先确认值不值得做（CEO），再确认怎么做（Eng），最后打磨细节（Design/DX）。不要反过来。

  

---

  

### 阶段 2：编码实现

  

```

/freeze src/目标模块/ → 锁定编辑范围，防止改到不相关代码

→ 特别是调试时防止"顺手改了别的地方"

  

正常编码（不用 skill）→ 这是 Claude Code 本体的工作，不需要 skill

  

/careful → 涉及数据库迁移、删文件等危险操作时开启

```

  

编码阶段一般不怎么用 skill。skill 的价值在编码前（计划）和编码后（审查）。

  

---

  

### 阶段 3：遇到问题

  

```

/investigate → bug、错误、异常行为

→ 铁律：先找根因，再修复。3 次假设失败就停下来

  

/codex consult → "这个实现方案对不对？让 codex 给个意见"

→ 外部第二意见，不受当前上下文影响

  

/learn → "之前遇到过类似问题吗？"

→ 查 gstack 跨会话积累的经验库

```

  

**为什么这个顺序：** 先自己查（investigate），查不到问别人（codex），再看看历史经验（learn）。不要一上来就问 codex，浪费第二意见的价值。

  

---

  

### 阶段 4：编码完成，准备发布

  

```

/health → 先跑一次健康评分，看整体状态

→ 输出：0-10 分，如果低于 7 分先解决基础问题

  

/qa-only → 先只看问题，不急着修

→ 输出：bug 列表、截图、复现步骤

  

/qa → 如果 qa-only 发现问题较多，用 qa 自动修

→ 输出：修复前后对比、发布就绪摘要

  

/design-review → 前端改动需要视觉审查

→ 发现视觉问题自动修复

  

/review → 最终代码审查，合并前的最后一道关

→ 关注：SQL 安全、LLM 信任边界、结构性问题

  

/codex review → 再找外部意见做 pass/fail 门控

→ 独立于 /review 的第二双眼睛

```

  

**为什么这个顺序：** 先看整体健康度（health），再找功能 bug（qa），再看 UI（design-review），最后做代码审查（review + codex）。不要跳过 health 直接跑 qa，基础问题会干扰 qa 结果。

  

---

  

### 阶段 5：发布

  

```

/ship → 创建 PR，自动跑测试、更新版本号和 CHANGELOG

  

/land-and-deploy → 合并 PR → 等 CI → 部署到生产

  

/canary → 部署后监控，截图对比，发现异常告警

  

/document-release → 更新 README/ARCHITECTURE/CHANGELOG

→ 让文档和代码保持同步

```

  

**为什么这个顺序：** ship 创建 PR → land-and-deploy 合并部署 → canary 监控验证 → document-release 收尾。不要跳过 canary 直接 document-release，万一部署有问题文档就白写了。

  

---

  

### 阶段 6：收尾与回顾

  

```

/checkpoint → 保存当前工作状态，方便下次恢复

→ 特别是要切换分支或下班时

  

/retro → 周末或 sprint 结束时做回顾

→ 分析 commit 历史、代码质量趋势

```

  

---

  

### 针对本项目（Enterprise RAG）的典型一天

  

```

上午：

1. /checkpoint → 恢复昨天的工作状态

2. /learn → 看看有没有相关的历史经验

3. 正常编码 → 修 retrieval_service.py / reranker_client.py 等

  

中午遇到问题：

4. /investigate → qwen3-rerank 403？检索质量下降？系统化排查

5. /freeze backend/ → 调试时锁定编辑范围

  

下午编码完成：

6. /health → 跑一次健康评分

7. /qa-only → 先只看问题

8. /review → 代码审查

9. /browse → 浏览器截图验证前端

  

准备下班：

10. /checkpoint → 保存进度

```

  

周五：

```

11. /retro → 周回顾

12. /document-release → 同步文档

```

  

---

  

### 总结：最小可用的 skill 子集

  

如果你不想记 30 个 skill，记住这 6 个就够覆盖 80% 的场景：

  

| 排名 | Skill | 一句话说明 |

|-----|-------|-----------|

| 1 | `/investigate` | 遇到 bug 必用 |

| 2 | `/review` | 合并前必用 |

| 3 | `/health` | 编码完成后先跑 |

| 4 | `/qa-only` | 想快速看有没有问题 |

| 5 | `/ship` | 要发布时用 |

| 6 | `/checkpoint` | 要中断工作时用 |
