
#我该在实践中怎么优化

10:45 AM

最有效的优化方式，不是再先想一套更大的 workflow，而是把它当产品做“带数据的小步迭代”。

我建议你在实践里按这 5 步优化：

1. 先只盯“真实摩擦”，不要凭感觉大改  
    每做完一轮任务，记 3 件事：
    - 哪一步最卡
    - 哪一步最容易出错
    - 哪一步其实没提供价值

你现在最值得持续观察的就是：

- .agent/runs/*.yaml 还哪里手写太多
- preflight/verify 有没有误拦
- 哪些任务其实不该进重流程

2. 把优化目标限定成 3 类指标  
    别泛泛说“更顺”。每次只看：
    - 时间：从接需求到开始改代码花多久
    - 噪音：为了通过 workflow 额外改了多少非业务文件
    - 返工：有没有因为边界不清、漏验证、误路由而返工

如果一个改动不能明显改善这三类之一，就先别做。

3. 优先优化“默认行为”，不要再加太多规则  
    高 ROI 的方向通常是：
    - scaffold 自动补更多正确默认值
    - planning artifact 默认少污染 allowed_paths
    - runtime 自动给更准的 lane / check 建议
    - review/preflight 报错更具体，少让你猜

也就是说，尽量让系统“默认做对”，而不是要求你“记住更多”。

4. 每次只改一层  
    按顺序最稳：
    - 先入口层
    - 再执行层
    - 最后工件层

你现在入口层已经差不多了，所以后面更该只动：

- 执行层：scaffold / preflight / verify
- 工件层：plan_ref / handoff / run yaml 去重

不要同时改入口、分类、任务契约、review 口径，不然很难判断到底哪一刀有效。

5. 给每次 workflow 改动设回归样例  
    以后每改 workflow，至少拿这 4 类任务回归一次：
    - 一个 Lite 小改
    - 一个 runtime_host/control_plane 中改
    - 一个 retrieval/ingestion 高风险改
    - 一个 dirty worktree 场景

如果新规则让这 4 类里有一类明显更笨重，就说明改过头了。

如果你想要最落地的实践版本，我建议你现在固定这个节奏：

- 每完成 5 个真实任务，再做一次 workflow 小复盘
- 每次复盘只允许选 1 个摩擦点优化
- 每次优化都必须补一条 harness/host 回归测试
- 连续两轮没有明显收益，就停止继续动 workflow，回到业务开发

一句话说：  
在实践里优化 workflow，最好的方法不是“设计更完整”，而是“用真实任务找最贵的摩擦，然后只削那一刀”。

如果你愿意，我下一步可以直接帮你整理一份“workflow 复盘模板”，让你以后每 5 个任务就能快速判断该优化哪一处。












runtime-workflow 统一主入口

9:19 AM

Worked for 55s

我更倾向于中间态，不是“现在这样”，也不是“所有东西都强制先过 runtime-workflow”。

更准确地说，最好的形态是：

- runtime-workflow 作为 repo code workflow 的统一默认主入口
- 显式 repo-native 命令继续直达，比如 /plan、/review、/spec
- 全局 superpowers:* 不再抢前台入口
- prompt 级的 specialized bypass 大幅收缩，只保留极少数真正独立于 repo coding workflow 的能力，比如纯浏览器截图、纯 checkpoint 之类

我不太赞成“现在这样”，因为现在的实际体验是“专项入口优先，主入口只接剩余流量”，这会让控制面分裂，用户也很难形成稳定心智模型。

我也不太赞成“绝对统一总闸门”，因为那会让已经很明确的 /plan、/review、固定 lane 也被迫绕一层，反而增加摩擦。

所以我的结论是：

- 比现在更好的方案：runtime-workflow 统一默认入口
- 但不是唯一入口
- repo-native 显式命令保留直达
- superpowers:* 和大部分 prompt 级 specialized route 应该退到二线

如果你要我直接给一个一句话版本：  
这个仓库最合适的是“runtime-workflow 做主入口，/plan /review 做保留直达”，不是“现在这种多前门”，也不是“所有请求都必须过唯一总闸门”。

这个skill能干什么

11:48 AM

runtime-workflow 这个 skill 的作用很聚焦：它不是拿来直接写业务逻辑的，而是把一句自然语言的仓库请求，接到仓库自己的 runtime adapter 上，让系统先判断“下一步该怎么走”。

它主要能做两件事：

1. resolve  
    用来判断下一步 workflow。  
    比如你问“我现在该怎么办”“这次改动该走哪个 lane”，它会结合当前改动路径，返回像下面这类结果：
    
    - 应该走 flow-ui / flow-api / flow-rag
    - 需要先 /plan
    - 需要 /review
    - 需要补 task contract
    - 不适合 auto-flow，要人工判断
2. drive  
    用来在安全时自动往前推进。  
    比如你说“直接做”“继续执行”，它会先 resolve；如果 runtime 判断这一步可以自动继续，就会进一步调用 advance，把推荐的下一步直接跑掉。
    

它本质上做的是“路由和推进”，不是“替代所有 skill”。

更直白一点：

- 它擅长回答“这次仓库改动现在该走哪条工作流”
- 它擅长把自然语言请求转成 runtime 的结构化判断
- 它不负责重新发明 lane 规则
- 它不该自己维护第二套 prompt 分类表
- 它不是“全技能宇宙总入口”

你可以把它理解成这个仓库里的“工作流调度入口”：

- 你不知道下一步该 /plan 还是 flow-ui 时，用它
- 你想让系统根据当前改动自动判断时，用它
- 你已经明确知道要跑 flow-ui 或 flow-api 时，就不一定非要先过它

结合你现在这个仓库，它最常见的价值是：

- 帮你根据 changed paths 判断当前改动属于 lite、api_schema、control_plane、retrieval 等哪一类
- 决定是否能 auto-flow
- 告诉你是不是必须先 /plan
- 告诉你是不是必须 /review

所以这个skill不会被被动触发是吗

1:41 PM

不是“完全不会”，但也不是那种后台自动监听型触发。

更准确地说，runtime-workflow 有两种触发方式：

1. 显式触发  
    你直接点名 runtime-workflow，就会用它。
    
2. 隐式触发  
    当你给的是“仓库内代码任务的自然语言请求”，而且没有明确指定别的 skill / lane / 命令时，按这个仓库的规则，它是默认入口，我会主动用它做 resolve 或 drive。
    

但它不会因为这些情况自己跑：

- 你只是打开了文件
- 工作区有改动
- bridge 已经启动
- 你没有发出需要 workflow 路由的请求

所以结论是：

- 不会“环境级被动触发”
- 会“意图级隐式触发”

一句话版：  
它不是后台守护进程，但它是这个仓库里自然语言代码请求的默认 workflow 入口。

例如：

- “我现在该怎么办” -> 会隐式走 runtime-workflow
- “直接继续做这个 portal 改动” -> 也可能隐式走它
- “跑 flow-ui” -> 不需要它，直接跑指定 lane
- “解释一下这个文件” -> 通常也不需要它

如果你要，我可以顺手把“哪些请求会走 runtime-workflow，哪些不会”给你列成一个很短的判断表。

1:42 PM

可以

1:44 PM

可以，按这个仓库当前规则，简化成这张表就够了。

会走 runtime-workflow：

- 你给的是仓库内代码目标，但没指定 lane、skill、命令  
    例：我现在该怎么办、继续推进这个改动
- 你希望系统先判断下一步 workflow  
    例：这次该走哪个 flow
- 你希望 agent 直接根据当前改动继续推进  
    例：直接做、继续处理这个问题

通常不会走 runtime-workflow：

- 你已经明确指定了别的 skill  
    例：用 ui-ux-pro-max、用 review
- 你已经明确指定了命令或 lane  
    例：跑 flow-ui、跑 flow-api
- 你是在做专项入口任务  
    例：写 spec、做 plan、做 review、做 QA
- 你只是问代码解释、文档解释、概念问题  
    例：这个函数是干什么的

容易混淆的一类：

- 明确、局部、低歧义的编辑请求  
    例：把 Makefile 里这行改掉

这类通常可以直接实现，不一定先过 runtime-workflow。因为仓库规则里明确说了，局部直接编辑请求不强制先走 runtime。

判断原则可以压缩成一句：

- “先判断 workflow”时，用 runtime-workflow
- “我已经知道要做什么”时，通常不需要它

如果你愿意，我可以再把它收成一个更实用的版本：什么时候我应该主动点名 runtime-workflow。

1:44 PM

可以

1:48 PM

你应该主动点名 runtime-workflow 的场景，基本就这几类。

1. 你不知道下一步该走什么  
    例子：
    - 用 runtime-workflow 看看我现在该怎么办
    - 这次改动该走哪个 flow

适用原因：

- 你要的是路由判断，不是直接改代码

2. 你怀疑当前改动范围已经跨了边界  
    例子：
    - 同时碰了 frontend/、backend/、docs/、AGENTS.md
    - 不确定这次到底算 lite、api_schema 还是 control_plane

适用原因：

- 它会根据 changed paths 给出 workflow 结论，比手猜稳定

3. 你想让系统决定能不能 auto-continue  
    例子：
    - 用 runtime-workflow 继续推进
    - 直接看看这次能不能 auto-flow

适用原因：

- 它先 resolve，再决定是否适合 drive

4. 你在用 Node host / IDE 插件链路，想验证整条 runtime 通路  
    例子：

- node scripts/codex_runtime_nl_entry.mjs --prompt "我现在该怎么办"

适用原因：

- 这不只是 workflow 判断，也是在验证 bridge + host adapter + runtime service 整体是通的

不必主动点名它的场景：

- 你已经明确知道要跑 flow-ui / flow-api
- 你已经明确要我直接改某个文件
- 你只是要解释代码或文档
- 你已经在 spec / plan / review / QA 的专项流程里

最实用的一句判断：

- “我不知道该怎么走”时，点名 runtime-workflow
- “我知道要你直接干什么”时，不需要点名

你这个仓库里，最值得主动点名它的其实只有两种时刻：

- 刚接手一个混合改动工作区
- 改到一半，怀疑已经从 lite 变成了要 /plan 或 /review 的任务



现在可以把它当成一套“控制面 + 执行面”的 workflow 来用：

控制面还是 AGENTS.md。也就是说，决策顺序没变，仍然是 spec → plan → implement → review。  
新加的这套东西只是执行面适配层，入口在 scripts/agent_runtime.py，共享逻辑在 backend/app/services/agent_runtime_service.py，返回结构在 backend/app/schemas/agent_runtime.py。

**最实用的日常用法**

1. 先看当前改动该走哪条 lane

`python scripts/agent_runtime.py inspect`

2. 如果是高风险改动，先补 task contract，再做 preflight

`python scripts/agent_runtime.py preflight --task .agent/runs/xxx.yaml`

3. 需要跑固定执行流时，直接执行 flow

`python scripts/agent_runtime.py execute-flow flow-ui python scripts/agent_runtime.py execute-flow flow-api python scripts/agent_runtime.py execute-flow flow-rag`

4. 实现后跑 verify

`python scripts/agent_runtime.py verify --task .agent/runs/xxx.yaml`

5. 想看当前 session / workflow / eval 工件，用 run/artifact 能力

`python scripts/agent_runtime.py list-runs python scripts/agent_runtime.py list-runs --run-kind workflow_session --limit 5 python scripts/agent_runtime.py read-run workflow-session:WFS-portal-chat-search python scripts/agent_runtime.py fetch-artifact workflow_session:.workflow/active/WFS-portal-chat-search/workflow-session.json`

**一条完整心智模型**

- inspect：先判断你这次改动属于 lite / control_plane / api_schema / retrieval / ingestion
- preflight：高风险改动先检查 task contract、必读项、allowed paths
- execute-flow：只负责执行固定 lane，不替代 plan/review
- verify：跑该 task 规定的检查
- list-runs / read-run / fetch-artifact：回看状态和证据

**如果你现在只是想开始用**

最推荐你先记住这 4 条：

`python scripts/agent_runtime.py inspect python scripts/agent_runtime.py preflight --task .agent/runs/your-task.yaml python scripts/agent_runtime.py execute-flow flow-api python scripts/agent_runtime.py verify --task .agent/runs/your-task.yaml`

**如果你是在代码里接插件/runtime**

直接 import service，而不是自己拼脚本：

`from backend.app.services.agent_runtime_service import get_agent_runtime_service service = get_agent_runtime_service() result = service.inspect_paths_capability({"scripts/agent_runtime.py"}) print(result.to_payload() if hasattr(result, "to_payload") else result.model_dump(mode="json"))`

**一句话版**

先 inspect，高风险就 preflight，需要固定 lane 就 execute-flow，收尾用 verify，证据用 read-run / fetch-artifact。

如果你愿意，我下一条可以直接按你现在手头某个真实任务，给你写一版“这次该输入哪几个命令”的最短操作清单。


那你之前指的插件和codex的插件有什么区别


我前面说的“插件”，更接近“你准备接入这套 runtime 的宿主/扩展层”，不一定特指你截图里这个官方 Codex IDE 插件本身。

区别可以这样看：

**1. Codex 插件**  
就是你现在在 IDE 里聊天的这个现成产品。  
它负责：

- 聊天
- 读 IDE 上下文
- 展示执行过程
- 帮你改代码、跑命令

但它不会因为你仓库里多了 agent_runtime_bridge 和 SDK，就自动知道：

- 什么时候该调 resolveWorkflow
- 怎么把 changed paths/task contract 映射进去
- 什么时候自动 advanceWorkflow

也就是说，官方 Codex 插件是“通用 agent 宿主”，不是你仓库专属的 workflow runtime consumer。

**2. 我前面说的“插件”**  
更像是“接你这套 runtime 的定制宿主层”或者“扩展层”。  
它可以是：

- 一个你自己写的 IDE extension
- 一个包在 Codex/VS Code 外面的 sidecar
- 一个在现有插件宿主里加的 adapter 层
- 一个内部工具面板

它的职责是把：

- 自然语言请求
- IDE 当前文件/changed paths
- task contract
- 你仓库的 workflow runtime

真正连起来。

所以我前面说“插件里 agent 可以自己识别并走 workflow”，说的不是“Codex 官方插件天然已经支持这个”，而是“如果有一层接入你现在这套 SDK/bridge 的宿主逻辑，就能做到”。

**一句话区别**

- Codex 插件：通用聊天/执行产品
- 我说的插件：你这套 runtime 的定制接入层

所以你刚才那个疑惑是对的。更准确的说法应该是：

不是“Codex 插件已经会用这套 workflow”，  
而是“你现在已经把这套 workflow 做成了一个 Codex 插件或其他宿主都能接的 runtime”。

如果你愿意，我下一条可以直接把这两个概念画成一张很短的“分层图”，你一眼就能看出：  
Codex IDE 插件 / 你的接入层 / runtime SDK / bridge / repo workflow 各自在哪一层。

8:38 AM

可以

8:39 AM

可以，把它看成 5 层最清楚：

**1. Codex IDE 插件**  
你现在在用的聊天界面。  
负责：

- 接收自然语言
- 读取 IDE context
- 展示执行过程
- 让 agent 改代码、跑命令

它是通用宿主，不懂你仓库私有 workflow 语义。

**2. 你的接入层**  
这是我前面说的“插件”更准确的意思。  
负责把：

- 用户自然语言
- 当前文件 / changed paths
- task contract
- runtime SDK

串起来。

这一层可以是：

- Codex 插件未来支持的一段定制逻辑
- 你自己写的 IDE extension
- 一个 sidecar/service adapter

**3. Runtime SDK**  
就是现在这层：

- sdk/agent-runtime-host/index.mjs (line 1)
- sdk/agent-runtime-host/index.d.mts (line 1)

负责提供稳定调用面：

- resolveWorkflow
- advanceWorkflow
- driveAgentWorkflow

**4. Local Bridge**  
就是：

- scripts/agent_runtime_bridge.py (line 1)

负责把外部调用转成仓库内结构化 capability。

**5. Repo Workflow 真源**  
就是你仓库里真正决定规则的地方：

- backend/app/services/agent_runtime_service.py (line 1)
- scripts/agent_harness.py (line 1)
- scripts/agent_flow.py (line 1)
- AGENTS.md (line 1)

它们才真的定义：

- 怎么 inspect
- 什么时候 preflight
- 什么时候 flow
- 什么时候该停下来 review

**一句话图**

Codex IDE 插件  
→ 你的接入层  
→ Runtime SDK  
→ Local Bridge  
→ Repo Workflow 真源

所以区别就是：

- Codex 插件 是通用聊天宿主
- 你的接入层 是把这个宿主接到你仓库 workflow 的那一层
- 现在我们已经把下面三层基本搭好了
- 还差的是把 Codex 插件上的自然语言入口，正式接到这套 runtime 上

如果你愿意，我下一步可以直接给你写一版“从用户发一句自然语言，到宿主如何调用 driveAgentWorkflow”的时序图。

8:39 AM

可以。时序图写完然后把 Codex 插件上的自然语言入口，正式接到这套 runtime 上

8:41 AM

Worked for 15s

已经按你说的“轻正式化”接好了，现在它是一个仓库内的正式 workspace 包，但还没有进入外部发布流程。

改动在 package.json (line 1)，根 workspace 已挂上 sdk/*；SDK 自身还是 private，包名固定为 sdk/agent-runtime-host/package.json (line 1) 里的 @enterprise-rag/agent-runtime-host。仓库根其他 Node 宿主现在可以稳定直接写：

`import { driveAgentWorkflow } from "@enterprise-rag/agent-runtime-host";`

我还补了根级消费验证 scripts/agent_runtime_workspace_import.test.mjs (line 1)，并把 README/接入文档同步成 workspace 用法：sdk/agent-runtime-host/README.md (line 45)、docs/agent-runtime-plugin-reference.md (line 35)。

验证是新鲜跑过的：

- npm install
- node --test scripts/agent_runtime_workspace_import.test.mjs -> 1 passed
- node --test sdk/agent-runtime-host/index.test.mjs -> 7 passed
- frontend/node_modules/.bin/tsc -p sdk/agent-runtime-host/tsconfig.json -> 通过
- git diff --check -- ... -> 通过


我需要在插件里agent能自己识别不需要我敲命令

8:06 AM

Worked for 1m 21s

现在已经补上了“插件里 agent 自己识别下一步”的入口。

你不用再让插件自己拼 inspect / preflight / flow 了，直接调：

- workflow.resolve

对应实现：

- 共享逻辑在 backend/app/services/agent_runtime_service.py
- CLI 入口在 scripts/agent_runtime.py

它现在会自动做这几件事：

- 先看当前改动属于什么类别
- 如果是高风险且没 task contract，推荐下一步 harness.preflight
- 如果已有 task contract 且 preflight 通过，并且能路由到固定 lane，直接推荐 flow.execute
- 如果是 control-plane 这类不能 auto-flow 的情况，就返回手动 review / plan 的建议，不会乱跑

你在插件里最该接的其实就是这一条：

`from backend.app.services.agent_runtime_service import get_agent_runtime_service service = get_agent_runtime_service() result = service.resolve_workflow_capability( changed_paths={"backend/app/services/retrieval_service.py"}, task_path=None, ) payload = result.to_payload()`

或者直接走 CLI：

`python scripts/agent_runtime.py resolve-workflow --paths backend/app/services/retrieval_service.py`

返回里最关键的是：

- payload.recommended_capability
- payload.recommended_args

比如现在它会返回这种判断：

- retrieval 高风险但没 task：推荐 harness.preflight
- API 改动且 preflight 通过：推荐 flow.execute + flow-api

这就已经够插件做“自动识别下一步”了。插件只要：

1. 收集当前 changed paths
2. 可选带上 task_path
3. 调一次 workflow.resolve
4. 按 recommended_capability + recommended_args 自动继续

我已经把这部分回归也跑过了：

- 44 passed

如果你愿意，我下一步可以继续把它往“插件真正可直接串起来”推进一层：  
把 workflow.resolve -> auto execute recommended capability 也做成一个受控能力，这样插件连第二次调用都不用自己写。

8:07 AM

4 files changed+216-0

Undo

Review

backend/app/services/agent_runtime_service.py

backend/tests/test_agent_runtime.py

backend/tests/test_agent_runtime_service.py

scripts/agent_runtime.py

可以

8:08 AM

Worked for 1m 39s

现在插件这边已经不只是“能问下一步是什么”了，还能“一次调用直接往前走一步”。

我加了一个新的受控能力：

- workflow.advance

落点在：

- backend/app/services/agent_runtime_service.py
- scripts/agent_runtime.py

它的行为是：

- 先内部调用 workflow.resolve
- 再只在白名单里自动执行推荐的下一步
- 目前支持自动执行：
    - harness.preflight
    - flow.execute
    - 回退时 harness.inspect

所以插件现在可以有两种集成方式：

1. 只想拿建议，不自动跑  
    调 workflow.resolve
    
2. 想让 agent 自己继续往前走一步  
    调 workflow.advance
    

最关键的是，这不是开放式链式执行，它只会执行 runtime 自己推荐的那一步，所以边界还在 control-plane 下面，不会变成任意脚本代理。

你现在在插件里最实用的调用方式会变成这样：

`from backend.app.services.agent_runtime_service import get_agent_runtime_service service = get_agent_runtime_service() result = service.advance_workflow_capability( changed_paths={"backend/app/services/retrieval_service.py"}, task_path=None, ) payload = result.to_payload()`

返回里你会同时拿到两层信息：

- payload["resolution"]：为什么它判断下一步是这个
- payload["executed"]：它实际执行了什么，以及结果如何

比如现在：

- retrieval 高风险但没 task contract：
    - resolution.recommended_capability == "harness.preflight"
    - executed.capability == "harness.preflight"
    - 结果会是 blocked
- API 改动且 preflight 通过：
    - resolution.recommended_capability == "flow.execute"
    - executed.capability == "flow.execute"

CLI 也已经通了：

`python scripts/agent_runtime.py resolve-workflow --paths backend/app/services/retrieval_service.py python scripts/agent_runtime.py advance-workflow --paths backend/app/services/retrieval_service.py`

验证结果：

- pytest backend/tests/test_agent_runtime_service.py backend/tests/test_agent_runtime.py backend/tests/test_agent_harness.py backend/tests/test_agent_flow.py backend/tests/test_run_ledger.py -q
- 47 passed

这一步做完以后，插件侧其实已经可以不再自己写“先 inspect 再 preflight 再 decide”这套判断了，直接把 changed paths 和可选 task contract 丢给 runtime 就行。

还没碰的边界也很明确：

- 我没有去动你当前前端那批脏改动
- 也没有把 workflow.advance 做成无限递归执行器，它只前进一步

如果继续往下做，我建议下一步就是：  
把 workflow.advance 的结果再补一个更插件友好的字段，比如 should_continue / terminal_state / ui_intent，这样插件连状态映射都不用自己猜。

8:10 AM

4 files changed+151-0

Undo

Review

backend/app/services/agent_runtime_service.py

backend/tests/test_agent_runtime.py

backend/tests/test_agent_runtime_service.py

scripts/agent_runtime.py

可以

8:10 AM

Worked for 1m 16s

我把这一步也做完了：现在插件不光能拿到“推荐下一步”，还能直接用顶层字段判断要不要继续、当前卡在哪、该展示什么 UI。

这次主要改了两处：

- backend/app/schemas/agent_runtime.py
- backend/app/services/agent_runtime_service.py

CapabilityResult 现在新增了 3 个插件友好的顶层字段：

- should_continue
- terminal_state
- ui_intent

我把这些字段先稳定地用在最关键的几条能力上了，尤其是：

- workflow.resolve
- workflow.advance

现在插件可以直接按这几个字段做判断，而不用自己去深挖 payload 语义：

- 缺 task contract 的高风险改动：
    
    - should_continue = false
    - terminal_state = "needs_input" 或 "blocked"
    - ui_intent = "collect_task_contract" / "fix_preflight"
- 已经可以自动执行固定 flow 的场景：
    
    - workflow.resolve
    - should_continue = true
    - terminal_state = "ready"
    - ui_intent = "auto_execute_recommended"
- workflow.advance 自动执行完一步之后：
    
    - 成功执行 flow：terminal_state = "completed"，ui_intent = "review_execution"
    - 被 preflight 卡住：terminal_state = "blocked"，ui_intent = "fix_preflight"

这样插件接法就更简单了。现在你甚至可以优先看顶层，不必先读 payload：

`result = service.advance_workflow_capability( changed_paths={"backend/app/services/retrieval_service.py"}, task_path=None, ) if result.terminal_state == "blocked": # show blocker UI ... elif result.should_continue: # continue agent loop ...`

验证结果：

- pytest backend/tests/test_agent_runtime_service.py backend/tests/test_agent_runtime.py backend/tests/test_agent_harness.py backend/tests/test_agent_flow.py backend/tests/test_run_ledger.py -q
- 47 passed

我还是没动你当前那批前端脏改动，也没碰 .agent/context/session-handoff.md，因为它明显是别的 lane 的活跃状态。

如果继续往下，我建议下一步就不是再加 capability 了，而是把这 3 个字段的取值收成一份小枚举表/文档，让插件端完全不用猜字符串含义。

8:12 AM

4 files changed+48-0

Undo

Review

backend/app/schemas/agent_runtime.py

backend/app/services/agent_runtime_service.py

backend/tests/test_agent_runtime.py

backend/tests/test_agent_runtime_service.py

可以

8:14 AM

Worked for 1m 2s

我把这条线真正收口成“插件可直接依赖”的状态了。

现在不只是有 should_continue / terminal_state / ui_intent 这几个字段，而且它们已经被收成明确的 schema 枚举，插件不用再猜字符串：

- backend/app/schemas/agent_runtime.py

我还补了一份很短的消费说明，插件侧如果有人接这层，直接看这个就够：

- docs/agent-runtime-contract.md

这次具体做了两件事：

1. 把 runtime 顶层状态收成显式枚举类型  
    现在 schema 里导出了：
    - CapabilityTerminalState
    - CapabilityUiIntent

CapabilityResult 也不再是随便塞字符串，而是显式约束到这些枚举值。这样插件侧不管是静态绑类型还是运行时读 schema，都能拿到确定集合。

2. 给插件一个最小消费规则  
    文档里明确了：
    - should_continue 什么时候该是 true/false
    - terminal_state 的几个合法值和含义
    - ui_intent 每个值该对应什么 UI/agent 动作
    - 推荐的接法依旧是：
        1. workflow.resolve
        2. 如果 should_continue == true 且 ui_intent == "auto_execute_recommended"，再调 workflow.advance

验证都过了：

- pytest backend/tests/test_agent_runtime_schema.py backend/tests/test_agent_runtime_service.py backend/tests/test_agent_runtime.py backend/tests/test_agent_harness.py backend/tests/test_agent_flow.py backend/tests/test_run_ledger.py -q
- 50 passed
- git diff --check 也通过了

我还是没碰你当前前端那批脏改动，也没去改 .agent/context/session-handoff.md，因为它明显是别的 lane 的活跃状态。

现在这套 runtime 对插件来说，已经基本够做：

- 自动判定下一步
- 自动推进一步
- 用稳定枚举做 UI 和 agent loop 状态机

如果继续往下做，我建议下一步就不是再加 capability 了，而是补一个“插件接入示例”小文件，直接给前端/插件开发一个最短 reference implementation。

8:15 AM

3 files changed+128-3

Undo

Review

backend/app/schemas/agent_runtime.py

backend/tests/test_agent_runtime_schema.py

docs/agent-runtime-contract.md

可以

8:15 AM

Worked for 40s

我把“插件接入示例”也补好了，现在这条线基本已经能直接交给插件开发接。

新增的是：

- docs/agent-runtime-plugin-reference.md
- 以及从 docs/agent-runtime-contract.md 过去的链接

这份 reference 里已经给了：

- 最短接法
- TypeScript 参考实现
- Python 参考实现
- ui_intent 到插件 UI 行为的建议映射
- 典型返回样例

所以插件侧现在可以很直接地按这套接：

1. 先调 workflow.resolve
2. 如果返回：
    - should_continue == true
    - ui_intent == "auto_execute_recommended"  
        再调 workflow.advance
3. 其他情况直接按：
    - terminal_state
    - ui_intent  
        做 UI 和 agent loop 控制

这意味着插件开发基本不需要再自己推断：

- 是不是 high-risk
- 应不应该 preflight
- 能不能 auto-flow
- 什么时候该停下来等用户

这些都交给 runtime 了。

我也顺手确认了这套文档和 schema/service 仍然一致：

- pytest backend/tests/test_agent_runtime_schema.py backend/tests/test_agent_runtime_service.py backend/tests/test_agent_runtime.py -q
- 17 passed
- git diff --check 通过

我还是没碰你当前那批前端脏改动，也没去覆盖 .agent/context/session-handoff.md。

现在如果你要把这套真的接进插件，已经不缺 contract 了。下一步最自然的就是两条路二选一：

1. 我继续帮你做一个“本地插件桥接器”小脚本/HTTP shim  
    这样插件可以直接调，不必自己起 Python 进程。
    
2. 或者我直接帮你写一份“插件端最小接入 PR 草稿”  
    按你现在插件宿主的技术栈去写。
    

8:16 AM

2 files changed+202-0

Undo

Review

docs/agent-runtime-contract.md

docs/agent-runtime-plugin-reference.md

可以

8:17 AM

Worked for 4m 28s

本地插件桥接层已经补上了。现在插件不用再起 python scripts/agent_runtime.py ... 子进程去拼命令，而是可以直接打本地 HTTP shim。

核心落点在 backend/app/schemas/agent_runtime.py (line 29)、backend/app/services/agent_runtime_service.py (line 423)、scripts/agent_runtime_bridge.py (line 33)。我新增了结构化 AgentRuntimeBridgeRequest、service 里的白名单 dispatch，以及本地 FastAPI bridge，暴露：

- GET /health
- GET /plugin-runtime/schema
- POST /plugin-runtime/agent-runtime

现在最短用法就是：

`python scripts/agent_runtime_bridge.py --host 127.0.0.1 --port 8011`

然后插件直接发：

`{ "command": "resolve-workflow", "task_path": ".agent/runs/demo.yaml", "changed_paths": ["backend/app/api/v1/endpoints/chat.py"] }`

接入说明我也改到了 docs/agent-runtime-contract.md (line 9) 和 docs/agent-runtime-plugin-reference.md (line 13)，之前那个 {command,args} 示例已经换成结构化 JSON 版本了。

验证这轮是新鲜跑过的：

- pytest backend/tests/test_agent_runtime_schema.py backend/tests/test_agent_runtime_service.py backend/tests/test_agent_runtime.py backend/tests/test_agent_runtime_bridge.py backend/tests/test_agent_harness.py backend/tests/test_agent_flow.py backend/tests/test_run_ledger.py -q -> 57 passed
- bridge smoke：POST /plugin-runtime/agent-runtime 返回 200，capability=workflow.resolve，ui_intent=collect_task_contract
- git diff --check -- ... 通过

我没有碰你当前那批前端脏改动，也没有改 .agent/context/session-handoff.md。如果你继续点头，我下一步就直接补“插件端最小 host adapter”示例文件，让宿主侧几乎只剩复制接线。

8:21 AM

8 files changed+387-20

Undo

Review

backend/app/schemas/agent_runtime.py

backend/app/services/agent_runtime_service.py

backend/tests/test_agent_runtime_bridge.py

backend/tests/test_agent_runtime_schema.py

backend/tests/test_agent_runtime_service.py

docs/agent-runtime-contract.md

docs/agent-runtime-plugin-reference.md

scripts/agent_runtime_bridge.py

可以

8:22 AM

Worked for 1m 20s

我把宿主侧最小接线也补上了，现在插件开发可以直接抄 scripts/agent_runtime_host_adapter_example.mjs (line 1)。它已经封装了 callAgentRuntime、resolveWorkflow、advanceWorkflow、driveAgentWorkflow、readBridgeSchema，默认连本地 bridge http://127.0.0.1:8011，宿主侧基本只需要传 changedPaths 和可选 taskPath。

文档也一起收口了：docs/agent-runtime-plugin-reference.md (line 13) 现在直接指向这个脚本并给了最短调用方式，docs/agent-runtime-contract.md (line 5) 也补了 host adapter 入口。验证我新鲜跑过了：node --check scripts/agent_runtime_host_adapter_example.mjs 通过，模块导出检查通过；我还把 scripts/agent_runtime_bridge.py (line 33) 起在 127.0.0.1:8017，再用这个 host adapter 真打了一次请求，拿到了 workflow.resolve / collect_task_contract 的结构化返回；GET /plugin-runtime/schema 也返回 ok，git diff --check 干净。

我没有碰你当前前端那批脏改动，也没去改 .agent/context/session-handoff.md。如果继续往下推，下一步最值的是把这个 host adapter 再包成一个真正的插件 SDK 小目录，而不是再加更多 capability。

8:24 AM

3 files changed+136-80

Undo

Review

docs/agent-runtime-contract.md

docs/agent-runtime-plugin-reference.md

scripts/agent_runtime_host_adapter_example.mjs

可以

8:24 AM

Worked for 2m 29s

我把宿主侧这半步真正收成了一个小 SDK 目录：新增 sdk/agent-runtime-host/index.mjs (line 1)、sdk/agent-runtime-host/package.json (line 1)、sdk/agent-runtime-host/README.md (line 1)。现在可复用真源在 SDK 里，原来的 scripts/agent_runtime_host_adapter_example.mjs (line 1) 只保留成 CLI/example 壳，不再自己维护第二套实现。文档也同步改了，docs/agent-runtime-plugin-reference.md (line 13) 和 docs/agent-runtime-contract.md (line 5) 现在都直接指向 SDK。

我还补了独立 Node 验证 sdk/agent-runtime-host/index.test.mjs (line 1)，覆盖 resolveWorkflow、advanceWorkflow、driveAgentWorkflow、readBridgeSchema 和错误返回。新鲜验证结果是：

- node --test sdk/agent-runtime-host/index.test.mjs -> 6 passed
- node --check sdk/agent-runtime-host/index.mjs
- node --check scripts/agent_runtime_host_adapter_example.mjs
- 真实 smoke：起 python scripts/agent_runtime_bridge.py --host 127.0.0.1 --port 8018 后，用 SDK 直接打 bridge，拿到了 workflow.resolve / collect_task_contract 的结构化结果，readBridgeSchema() 也返回 ok
- git diff --check -- ... 通过

focused review 这轮我也一起做了，没发现新的 control-plane 漂移：SDK 只是薄宿主层，没有引入第二套 lane 判定，也没有暴露 shell。我还是没碰你当前前端那批脏改动和 .agent/context/session-handoff.md。如果继续，我下一步就会把这个 SDK 再补一份 TypeScript d.ts 或 JSDoc 类型，让插件宿主拿到更稳的自动补全。

8:26 AM

7 files changed+395-86

Undo

Review

docs/agent-runtime-contract.md

docs/agent-runtime-plugin-reference.md

scripts/agent_runtime_host_adapter_example.mjs

sdk/agent-runtime-host/README.md

sdk/agent-runtime-host/index.mjs

sdk/agent-runtime-host/index.test.mjs

sdk/agent-runtime-host/package.json

可以

8:27 AM

Worked for 1m 22s

SDK 的类型出口已经补稳了。新增了 sdk/agent-runtime-host/index.d.mts (line 1)，把 CapabilityResult、AgentRuntimeBridgeRequest、CapabilityTerminalState、CapabilityUiIntent 这些宿主最需要的类型都显式导出来；sdk/agent-runtime-host/package.json (line 1) 也加了 types 和 exports.types。另外我补了 sdk/agent-runtime-host/tsconfig.json (line 1) 和 sdk/agent-runtime-host/typecheck.ts (line 1) 做静态类型验证，docs/agent-runtime-plugin-reference.md (line 38) 现在也明确写了声明文件位置。

验证是新鲜跑过的：

- node --test sdk/agent-runtime-host/index.test.mjs -> 6 passed
- frontend/node_modules/.bin/tsc -p sdk/agent-runtime-host/tsconfig.json -> 通过
- node --check sdk/agent-runtime-host/index.mjs
- 真实 smoke：起 python scripts/agent_runtime_bridge.py --host 127.0.0.1 --port 8019 后，用 SDK 调 readBridgeSchema() 和 driveAgentWorkflow()，拿到 ok / workflow.resolve / collect_task_contract
- git diff --check -- ... -> 通过

我没碰你当前前端那批脏改动，也没去改 .agent/context/session-handoff.md。现在这套 SDK 对插件宿主来说已经不只是“能跑”，而是“有静态类型、可直接接线”。如果继续做，下一步最自然的是补一个发布/消费层面的入口，比如把这个 SDK 变成更正式的 workspace package 名称和导入路径。

8:28 AM

6 files changed+175-1

Undo

Review

docs/agent-runtime-plugin-reference.md

sdk/agent-runtime-host/README.md

sdk/agent-runtime-host/index.d.mts

sdk/agent-runtime-host/package.json

sdk/agent-runtime-host/tsconfig.json

sdk/agent-runtime-host/typecheck.ts

我还有必要补一个发布/消费层面的入口，比如把这个 SDK 变成更正式的 workspace package 名称和导入路径吗

8:29 AM

现在不一定“必须”，但有一条很清晰的分界线：

如果这套东西短期只在这个仓库里被一个插件宿主消费，那我觉得没必要立刻做正式发布入口。现在的 sdk/agent-runtime-host/index.mjs (line 1) + package.json (line 1) + index.d.mts (line 1) 已经够当“稳定私有 SDK”用了，继续往 npm/workspace 发布层推进，收益不大，反而会把精力花在包管理、版本、导出兼容这些次要问题上。

但如果接下来会出现下面任一情况，就值得做：

- 不止一个宿主会消费这套 SDK
- 插件代码不在这个仓库里
- 你希望导入路径稳定成 @enterprise-rag/agent-runtime-host
- 你需要版本化、变更说明、升级兼容策略

所以我的判断是：

- 只服务当前仓库内插件：暂时没必要
- 准备跨仓库/多人消费：有必要，而且应该尽早做

如果你想走一个中间路线，我更推荐“轻正式化”，不要一步到位发包：

- 保持 private，先固定包名和 exports
- 在根 workspace 里让它可被稳定 import
- 先不做外部发布，不引入版本发布流程

这会比直接上“发布/消费层”更值。

8:29 AM

可以，你的意思是我可以把这个包发出去给别人用是吗

8:30 AM

可以“走到能发出去”，但不是“现在这个状态就适合直接发给别人用”。

因为当前 sdk/agent-runtime-host/package.json (line 1) 还是 private: true，而且它本质上还是仓库内私有 SDK：

- 默认强绑定本地 bridge 地址 127.0.0.1:8011
- 还没有正式的版本策略
- 还没有对外 README、兼容承诺、发布流程

所以我的意思更准确是：

- 技术上：这套结构已经接近“可发布包”
- 产品上：现在更适合先当仓库内私有包用
- 如果你想给别人用：可以再补一层正式化后发布

最小还需要补的通常是：

- 去掉 private: true
- 固定对外包名与导入方式
- 补外部使用文档和启动 bridge 的说明
- 明确 Node 版本、运行前提、错误语义
- 定一个最基本的版本策略

所以答案是：能走到可发布，但我不建议把当前这个版本直接当公共包发。

如果你愿意，我下一步可以直接把它整理成“可发布版本”的最小 checklist，或者直接帮你把 package metadata 补到接近可发的状态。

8:30 AM

发布是什么意思，发布到哪

8:31 AM

这里的“发布”，指的是把这个 SDK 做成一个别人能安装和引用的包，而不只是你仓库里的一段本地代码。

最常见有 3 种去处：

- 发到 npm 这类包仓库  
    别人可以 npm install @your-scope/agent-runtime-host
- 发到公司私有 npm registry  
    只有你们团队内部项目能装
- 不发到 registry，只放在 monorepo/workspace 里当内部包  
    仓库里的其他应用可以稳定地 import 它，但外部人拿不到

所以“发布到哪”，本质上是问这包要给谁用：

- 只给当前仓库用：不用发到外部，放 workspace/internal package 就够
- 给你自己别的仓库用：可以发到私有 registry
- 给外部开发者也能用：才考虑发到 npm 公网

结合你现在这条线，我更推荐你先理解成：  
不是“马上发到公网”，而是“先决定它是不是一个正式的可消费包”。

当前更现实的落点通常是：

- 第一步：仓库内正式 internal package
- 第二步：如果跨仓库复用，再发私有 registry
- 第三步：真要开放生态，再考虑公网 npm

所以，发布不一定是“公开到网上”，也可以只是：  
“让它成为一个有稳定名字、稳定入口、别人能安装/引用的包”。

如果你愿意，我下一条可以直接按你这个项目语境，给你画一个：  
“本地脚本 / workspace 包 / 私有 registry / 公网 npm” 四种形态的差别表。

8:31 AM

想走一个中间路线，我更推荐“轻正式化”，不要一步到位发包： 保持 private，先固定包名和 exports 在根 workspace 里让它可被稳定 import 先不做外部发布，不引入版本发布流程

8:33 AM

Worked for 1m 25s

这条“轻正式化”我已经直接接好了。根 package.json (line 1) 现在加了 private: true 和 workspaces: ["sdk/*"]，跑过一次 npm install 之后，仓库根 Node 宿主已经可以稳定地 import "@enterprise-rag/agent-runtime-host"。我还补了一个仓库根视角的验证 scripts/agent_runtime_workspace_import.test.mjs (line 1)，避免之前只测到包内自引用。

文档和 SDK 说明也一起收口了：sdk/agent-runtime-host/README.md (line 45) 现在明确写了 workspace 消费方式，docs/agent-runtime-plugin-reference.md (line 38) 也补了“可直接裸 import”的口径。验证结果是：

- node --test scripts/agent_runtime_workspace_import.test.mjs -> 1 passed
- node --test sdk/agent-runtime-host/index.test.mjs -> 7 passed
- frontend/node_modules/.bin/tsc -p sdk/agent-runtime-host/tsconfig.json -> 通过
- git diff --check -- ... -> 通过
- npm install 已把 node_modules/@enterprise-rag/agent-runtime-host -> ../../sdk/agent-runtime-host 链好

有个顺手发现但我没展开处理的点：npm install 输出了 4 high severity vulnerabilities，这是依赖审计信息，不是这次 workspace 接线造成的功能阻塞。我这轮没有碰你当前前端脏改动，也没去改 .agent/context/session-handoff.md。现在这条中间路线已经落成了：内部 workspace 包、稳定包名、稳定 import 路径，但还没有引入外部发布流程。

8:34 AM

5 files changed+42-1

Undo

Review

docs/agent-runtime-plugin-reference.md

package.json

scripts/agent_runtime_workspace_import.test.mjs

sdk/agent-runtime-host/README.md

sdk/agent-runtime-host/index.test.mjs

![User attachment](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAABAgAAATICAYAAACRcfeuAAAABHNCSVQICAgIfAhkiAAAABl0RVh0U29mdHdhcmUAZ25vbWUtc2NyZWVuc2hvdO8Dvz4AAAA9aVRYdENyZWF0aW9uIFRpbWUAAAAAADIwMjblubQwNOaciDIy5pelIOaYn+acn+S4iSAwOOaXtjMz5YiGNDbnp5Lao5y+AAAgAElEQVR4nOzdd1RUV9cG8GfovcNI70UFbIhRxAqK2LtGY6oxsSfRlDcxJu+bXkwsMdHEJJrEEmPvJbE3ULGB0ntTivQ6zPfHwJVhZmgCGr/ntxZrMbeeGWaGe/c5ex+RtY2tFI3w9PTEqJGhAIAtW7YiIzOzsc0fiqmpKfLz89vt+ERERERERESkyNTUFGpNbRQdHY3MrCwAwKDBg9q7TURERERERET0CDQZIACAE/+cAABYd+oEV1fXdm0QEREREREREXW8ZgUIMrOyEB8fDwAI7B/Qrg0iIiIiIiIioo7XrAABAJw8dRpSqRTm5uZwcXZuzzYRERERERERUQdrdoDg/v37uHfvHgDA0sqy3RpERERERERERB1Pw9HRsVkb6mhrw9zcHABw/35Be7aJiIiIiIiIiDqYxqSJE1q0Q15eHmJjY9upOURERERERET0KGg0d8OKigokJSfjzJmzqKmpac82EREREREREVEHE1nb2EofdSPqmJqaIj8//1E3g4iIiIiIiOj/FVNT0+YXKSQiIiIiIiKiJxcDBERERERERETEAAERERERERERMUBARERERERERGCAgIiIiIiIiIjAAAERERERERERgQECIiIiIiIiIgIDBEREREREREQEBgiIiIiIiIiICAwQEBEREREREREYICAiIiIiIiIiMEBARERERERERGCAgIiIiIiIiIjAAAERERERERERgQECIiIiIiIiIgIDBEREREREREQEBgiIiIiIiIiICAwQEBEREREREREYICAiIiIiIiIiMEBARERERERERGCAgIiIiIiIiIjAAAERERERERERgQECIiIiIiIiIgIDBEREREREREQEBgiIiIiIiIiICIDGo24AERGRMmpqalATiQCRCKJ6PwDkficiIiJqilQqhVQqlftdKpUCUilqpFLU1NQ84hY+HhggICKix4K6urosKFD7Q0RERNRWmtO5UFNTI/xIJJIO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9RCDhoraAcCZM2dRVloKX19fpKena41j12g0SExMQkhIsM7UgyKRCP379xMel5WVIyYmRqf3//q1awgNC0OPiB44fPgINBoNMjOzUFWlhp+vL3r2jAAAVFWpkZaWhqQk7QBDY6KjY+Dt5QUXFxd4eXlptT0//7gwPaJGo0FkZCR8vL3h7uamc05u5d1q0vO2NX3nraCgAOfOnYevjw/Cw8O0tk9LSxOmjXRzdYWTs5PW+oqKCkRHR3N4ARERERERdXoiJ2eXDiihp5+1tTVu3fpn3ZAay9nZGd26BeDYseMQi8WQy+UoLy9HaWlpg/vIZDKYmZkhX8/UinK5HCYmJiguLm7xzauZmRm6dOmCysrKBmdRAGqCG2ZmZjA1NYVGo0FxcTEqK0e6HhMAACAASURBVCtb9NxtwdB5k0gksLS0gFgsQUFBgc7wBhMTEyGroLpajYKCwjatkUBERERERHQnsLa27twZBG2lrKzMqJR1lUol9OQ39xjGKC0tNRioqKXRaFBSUvKPn/bP0HlTq9XIzy/Quw4AKisrmzQDBRERERERUWfRqWsQEBEREREREVENBghaSVFREW7euNnotIJERERERERE/0QcYtBKioqKDI7vJyIiIiIiIvonYwYBERERERERETFAQEREREREREQMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIgDS+4YNRdeutgCAyqpK3LxxEzGxsaioqOjgphERERERERFRe5GGh4drLfD28sLgwYMQH5+ACxcvIi0trYOaRkRERERERETtRVpaWorLl+NRXlEOV1dXuLu5QSKRICgoEEFBgbh16xbOn49E3KVLUKlUHd1eIiIiIiIiImoDIhdXN011dbWwQKlUIDQkFEFBgTA3NxeWV1ZW4vLleERGRiI7J6dNGmNtbY1bt261ybGJiIiIiIiISD9ra2uInJxdNA1t4O/vh5DgEHh6emgtz8jIRNSFC4iNjW31BjFAQERERERERNS+Gg0Q1LKyskJISDCCu3eHhYWFsPzmzZvY+f0PrdogBgiIiIiIiIiI2pe1tbVx0xwWFhbi1KnTOHHyJCorK4Xl7u7uCPD3b7MGEhEREREREVH7kDa2gY2NDXqEhyEoKAgymUzveiIiIiIiIiK6szUYIOjWrRvCQkPh6uqitfzateu4GB2NXj0j4OLiAojavI1ERERERERE1Ma0AgSWlpYIDwtD9+5BWjMYlJSUIDY2DhcuXkRhYSEAIKJHj/ZtKRERERERERG1GSkAeHt5ISw8DN5eXlorbyYn40LUBSQkJgIAFApF+7eQiIiIiIiIiNqc9MknHoeVlZWwoEKlQkxMDC5cuKg1o0B4WBgGDR6EDRs+7oh2EhEREREREVEbktYGB9IzMnAh6gIux8dDrVYDAEJCghHRowe2bvsScjM5TPUUKSQiIiIiIiKiO5/0wsWLiIq6gJycHACAh4cHfLy9cfDQIVhYWMDW1raDm0hEREREREREbU28f/8BqNVq+Pv5AQCcnZ3Qo0d4BzeLiIiIiIiIiNqTGAC6BQRgwoTxHd0WIiIiIiIiIuogYgCAqINbQUREREREREQdStzRDSAiIiIiIiKijift6AYQEREREd0pxGIxpFKp8J9YJAJETMclagv5+fkd3YROhwECIiIiIqJGiMViyGQymJiYQCKRdHRziIjahNFDDK5fu44zZ860ZVs6nJm5Oe4Zei8UCgVMTU0xffoMBAYGAgCUSoXBfZVKJaZPnwFXV5f2aGqTRfTsiQkT/9XRzTDI2toa06fPgL2Dg7AsODgEzz73HGxsbJp93JGjR+H9tR9ALpe3RjOpA4wcNRL33Tdc7zpLKyssf/MN9Ovf3+jjBQeHYObsWa3VPC0enp6Yv2AhugV2E5YplQoEBHQzsJe22bMfQr/+A9qieXcdsVhs8LstlUq1fsuba8zYsejdp3eLjlFLqVRg5OhRsLO3b9J+MpkMy998Q+uzdSd6YOpUhIWFdXQzqAXc3N3h6+snPJbJZFi0eHGLP5uenp5YtHgxnJydm7zvpMmTEdAtwOA2CxYuxPARI5rVNolYjC5duqBLly4MDhDRXc1gBkFebh7SMzIAAOkZGUjPyIBSqURYWCgcHGoubEpLStu+lW3EzMwMgUFBCOwWiIBuAXD3cIdIJEZUxHls+mQTxk0YD1WlCjdu3MDKVe8iMSEBH65fD41Go3MshVKJcRPG4+q1K0hJSQUA9OnTF15eXo224/jxY0hOTtZadv+kSTCVmTbp9eg7Tq3QsDBERPTErp9/adIx67J3cBDe9+aKiY7Re/4AwMbaGuMmjEfcpVhkZWYCAEJCQ9Cnbx9s2by52c+ZkZ4Oe3s73Dd8OH7dtavZx7kbDRg0EG4ubi06xu7fd6OosFB43LdfPzz73HNNOkZ8/GW88/bKBtcPGXIPqjUaHDiwX2edVCKBr68fzp4+a/Tz2dnbYvToMTh65E+d74yllRU8PT20lslkpsjMzEBKcorOsZa/+Sbyb+Xjw/XrAADu7h7o1bsXTp44IWwzdfoM9O7dG2tWrcKVq1cNts0/IAAjR4+C+vffcfLEcaNfU1sIDAxEaGjLb+R++H4nqtRq4fGq1WuaHExdvWoV4uLidJY//MijcPdwx79Xr0ZpWZnOehMTE4ybMB7l5eW4dOlS0xuPmvdkxoMPIvnmTZw53fJAeWVlFaZOnYbg7t2x9v/WGr2f5PZn3dLSUlg279lnGw2OLXj+edy6dUs4xqIXFiMqKgr79+0Vthk1ejRmzZ7daBvy8/Px5htvIC83FwDg6uaKec8+jx3fbMeFCxeMeh3jxo/H3j17jN6eOp5YLEb34O7w9vJBWI9w+Pr6IicnB0sWL4ZarcbIUaPQo0cPnD1zFiYmJsJ+yTeTm5QenJaWhm7dumHSpPux8eONTWrj6DFjUFlZifjL8Q1u4+XtjaysrCYdF6j53nTp0kXrtRER3a10AgR5uXkoKioCAMQnJCA+IQEAEBAQgNDQELi7/X0zUVxcjMvxDf8Q/1NNnToN9w69F5ZWVsKynOxsfLntSyTfTEZCQjwk4prkCrFEjNLSUmza+AkWL3kRTzz5BD7b8lmDN7l1RUREYMCggY1ud/3GdZ2blNGjR8PM3LxJr0vfcQwZO3YcZsx80OA2zz/7LAoKCgAAQ+8dinEtnA7zicceg0qlAgD06z8ACoUV9vzxR4Pbe3p54caNm8I+DZk+fUajbZs2fTqmTZ9uVNta06dbtrQ4e+G7HTvaJLjRu1dv9OzVq0XH+PPon1oBglobP96IkpKiRvd/cKbhmxJTU1O4uLri0MEDzW4jAAR17w6JpOZ7XVBQCLVajXHjJ+DYsaPCNlevXIWfnx8WLlqks396ejqWLlmitUypVMDb2xu/7fpVWOboWJMBk5r6dzDhi88/h5OTExa9+CLeXL4cOTk5DbZz5qxZKC4uxq6ff27eC21F/v4BLf7OA8BPP/0I1AkQAED0xWj88cfuRvf18PA0+L09cGA/Xn3tNby2/A2semcliouLW9zeuuRyOebOnYvy8nK4ubshomdPnD93rtH9ZDKZwd7MK1euIDQsHBE9e6KyUv/vzqXYOK3ASkPycnPxn/98prM8MDAI4ydM0FomAlBaWow5D8+Bs7Mjtm3dBgA4c+YM0tJqgtv2dg7Iys4U9jEzM8fE+yfBxcUZR4/8KQQHACAjLR3ZWZl4fsECrHjrLdy8caPR9t6JekRE4PEnnsDrr74qBFs6k5dffUXIglKr1di2dSvOnT0LtVoNWzs7TLz/fgDAk089qbVfSkoqXlm2VGuZqakp/AP8G3yu1NQU9OrdB2Fhx1Ctqda7TfzleKP+Xn+44WOcOH4M32zf3ui2DRGJRDA1NWVwgIg6DSkAVKoqob59EVI3KKBUKhAaGoruQUEwMzMTdsrJyUVkVCTi4i6hqqqqA5rdMpGRkbh1Kw9p6enIyszEvOeeg6pShf379gnbVKnVUKvVwh+EmJhobNmyGXPnzkVFhQrbtm416rny8vKwcP58AMA9Q+/F448/gTm3e2l8vL3xxooVDe679489+OqrLxt9Dj8/P7z+xhtGtaeupCtJWjc2+pSXlwv/3rHjW+zY8a3e7RQKBT76+GN8/dVXBm/46+rVuxecnV0a3F4mk8HP31+rl8uQqqoqfLD2/4zatq577r0Xffr0bfJ+xvrj998hM5HpXefh5YHg4BAcPHgQZQaycRITE9qkbevXrTO4vm/fvnj2+edx8sQJbPz44yYdO/7yJaMupCdNNpyF1CMiAhKJBOXlFQgJDRGWK5XW+OvoUQN7anth8WLIZNrvw4CBAzBg4N+p/P9+b43w79qAkUwmw6bNm3H+rO5N4dBh90EsFuPo0T+FZR4enlCpVEhLSxOWVVZWYu377+PdNWswavRofP3VV3rbOHzESHh7ewMANm7aZNTr2r37N3y7/Rujtm2qn3/+H37++X9610klEmzasgVJSUlYvWpVk4+dX5CP6IvRLW0ibt64gU0bN2L+woVY8tJSvLH89RYfs5apqSleWrYMFpaWWLliBQYNGYJ5zz6LNatXIzHB8Heyq60tlry01OA2APQGo2o9M3cuqkpKGj1GhUql91zWDYLXqlKr8cnGTyCWSDF8xEjcupWPXb/8grzcXOTl5qJPnz6YNmM6sjIzcfrMaUglUowaMwaVKhXWvv++Tq9/lVqNDR99hOVvvIGXX3kFbyxfLmSA3U2sFFaw0nM+O4vP//M5FAorBPh3w4SJ/xKul0xMTLDohRdw88YNrHz7baHzxM3NDStWrsT2r3V/6+zt7Y36biyuF5Cta+lLLyG9zm9sc4X3iEB6agoyDWQVmJqa6vztMGTAgIHo278vtv33C+Tm5bW4jUTtYcGiRUhKTMRvvxq+L6DOQQoAZ8+dw/Ub14WFAf7+CA0L1coWAGqCB+fPR2pd+N6JkpISkZSUWGeJ/myAsrIyWJhbCI+P/3UM7u7uyM7MFpY5ODrC3t4O9nY1vYZubu4oLy/Hjesd34vi5OQEWztbADXj+01MpMINVkZGJhLi45HwD84A6du3L6QSCUaPHoPRo8fo3abuRUJ1dbVwkWxhYYFHHnsU1kprrFm9WuhpmD37IXS17Yod336LjNvDZ4KDQ/Qeu7X89OOPDa4bNXo0goNDsPu335p8UV2bWrxl8xYc/fNIS5upw9raGo8+9hgqKirw3XffNXn/9R99ZPS28fGXG1w3ZMgQAMD4CRN0ekOvXrmC0lLjhjk98dhjGDpsKBwdnbR6kxwdHTF9xgz8+MMPSE5ORkTPnlr79e3bF1KpFAfqZTBYWFhg6LD7EBUVJXyWAMDbxxupKak6WUbFxcVY/8E6XL16RW/7HB0dMXXaVCQnJ+NilJGp2hPGt0nWizF8/f0gk8lw+uSpZu0/ePBgDB48uFXaEhkZiXVr12ql3reUTCbDkpdegru7O95bsxrJycn45uuvobC0wtJly7Dhow2Iijzf6HE++nC9zrCEgG4B6NuvP37dtUurN14fExMT/Oe//9Va9vz8BQCAY3/91cRXVUOj0eDjjz5C4cMPIzAoEL/9+iuqq2t6ak+fPo2oqCgMHDQQQ+65Fz4+PigsLMRnmzc3OCSgsrISa1avxrKXX4ajowNGjhiJkaNHGWzDmLFjMWbsWADAOyvfNpgWTh0vPS0N6Wlp8PbyEZYpFAosWrwYVlZW+Pmn/yE4JBhATY/77Ifm4EpSEkSimr+xMTG6Aax1H3xgVDZOXbUdLS3l5+eH+ydNRkhoCE6fPo0NH36od7vaooRNYWNjAy8vb0hld0/GgbOLC6bPmI6DBw7ggpF/nxoTEhqKJ596Cp9s/BiX4i7h3dWrcSnuErZta7gDTiqVwtXVFaqKCmRkZgq/W+2hLc7BP4mXlxcK9WSDUuckDDGoqqrCkCGDdbIFiktKcCHqAqJjYlBiRE/G3aSktBTm9dL86/fUDRs2TLjIAWpqBwA1F4UdbcTIkTrFeGqj9j/s/L7BnsHGbPr00waHP8yaPVvvONZtW7cZnQlQq/+AAaioqMD+vfu0lg8dfh/ycnJxISpKSCc+dPgQomOiIRKJENGzJx5+5FGYmsrw+Wefad1AnTx5Eg9Mm4rV772HUydP4fvvd+LgwYOIiYlGZWVlk9rXkeRyuTDuePjw4a0eIBCJRJg77xmYmZtj7569yMnObnyneuqOezZk+ZsNZ7/Y2dsjMCgI+/ft08raeerpp9G3b1+kpaVBqVQa3aaioiI8+tjj2Ld3r5Dm37tvHwSHhGDTJ5/o3advv/6IuxSncw4mTZ4MpVKB8PBwbKuXEWBhYaGzrL4XFy8WgkJyuRwvvrQUIpEIH6xda9T5lkokGDdhPCpVbfu5tbO3F4ZN1FWbdVNZqdLK7KivobojR48exZZPP230+UNCQ4zqbWzN8ewuLi6Y99xzsLOzw7oP1mrdvH766SaUVZRh0QsL8f333+O3Xb82+SJ19OgxCA4JwQ87dza6raa6Wsj06mrbFf3698fZM2eRmZGBa9euoVfvXnBycmr086ZzXI0G2774QnhsaMiZlZWVVm/uF5//FwfrBcyKi4vx2quvAgDEIrHB39Mx48bi6tWriL9UExjMzWUv651IIpWiuroaVlZWeG7+8zrrHR0dseSlpVCr1Xj04YfbpA3jJ0zQGoJUO5QwKSkRK958S2tbmUwGE6kU9wwdijFjxyIzIwP5+QWoNjCMx8TEBGKx0fW8AQBJSUkQ7xOjpPjuuWaWy7vAy8sbCqumBXRai7OLC2bMmAFXNzdIpTW3LkVFRfj8s89w5Yr+oHtr6+hz0NYO7N+PtNQ7uwOYWo+0traAm6srRLfncNVoNEhOTkZkVBSuXr3WrhG69tDQDa6+CywHe3u9y2vHhH+zfTu+2b4dHp6eeHvlSq3eop4RLRvbPXL0qEZ7YQzZtnUrtm3dCnNzc2zYuBEqlQoL589HWVkZLK2sDF7U16VvrN/p06dw5PBh4bHZ/7N3ngFNXm0YvpJAIOw9RWUqCCg4Ebc46t6r7lVXHbVq7bCOanf9rNXauq3burVO3FqhKuBgCyhOBERkQ+D7EYiEJCy12jbXL/KunLy845z7PM/9SPSY/P77BAaeVJgREIt1mDZ9epXbbmllRT1P2WzE0aNH5D4IYrGYLt26EBwcxL69e+XbJz1+jIeHB+PHj8fM3JyoqEj27tmLv78/CfHx8vDB2NgYvlqyhDp16zBg4CC++fZbLl28wJYtWyvlK/G6MDUxrVIEQXZ2NidPHKdR4yb88cfhV96eEaNG4uHuAYBUWoC9vT1CobBKHhevgq7duiISiThxQlEkcnR05MF9xVl6WztbbG1tefjwodrjhV4LITc3l1atW7Nn924AWvi34OLFi+Tm5iptb2JiTD3Peqxc8ZPSuqt/XUGkJSI784UxnlsdN1zd3Lh04WKF4khmsbhlZm7OzA8/xMrKkgP792Nra4OtrY3Kfe4k3JEr/NrFs1ql04BeB839mtO3fz+168e99165+48eMaJSefRVpawJbHr6M44cOfJSxxQIBHTp0pV+A/rzLC2NRQsWKF3zRUVFbFy/gdSUVPr260er1m3YvnUrV6+oN8ksO4gp4edyBJKS2dUCqVSe2tW9Rw+a+fkRfusmJ0/KDDsbNW5UJQ+C8iibpjV4yFBSUpI5fkyWBiYSafPBzA8qPE5oaCihoaEYGhmp9Cfp3OUdYqKj1aasvQ7EYjE1a9YiOflJuaZ5AoEAC0tLLMzNKSgoIDY2tsrvBi0tLWxtbTExlYmXt2OVI51s7eywsDBHKi3k4YMH5T4vnJ2d0dPXIzsrm7i4uAr7ZBaWltjYyJ4h0VGqc/WNjIywsrImIT6uwvuzpH9TmpJ+0YJ588o1Xg3o0JF3y6kY08yvObVq1iz3+0t48uQJ0iLFtsbERMsFtM5d3iEqOor42DielBJZdXUl9O7Th46dOqGvr8+jR49Yv2YNQUFB5aZ5AtWqVhAZGUFkZPUMUTUo4+Pry7tDh/LwwQO2/LaZqKhIjIyNade+PZOmTOHLxYvL9fXRUDkOHqi+ibmGfx9a3bq+mP3Oy8vj1q1wQkJD/9UmPEePHlWoEODn35xCqZSgMqGyDXx9MDMz49RJZXO015UTXprbcXFE3lJ27S6LsakJLVq0ULu+ub8/IpEIiUTChEmTWPr992qN2FShKtcvJTlFIefV2FhWBvLRw0cKy6trztezZ0+khYXk5uTQuHFjeWfY3d0dgUBIbEyswvYCgYBGjRrz8OFDNm7cSMi1a8z44AM8vbzYpSI8PioyikULFtC0WTPatm9HdiXD1F8Xn3z2KdeuXmXH9u3lDnBLs2njJrnB2KvknS5daNeuPTHR0bi6uYFAwISJE6nh4MC+PXsrHXnysikGZubmtGrVmvCIcIXrTyKRYGdnR+BJRdGgdZs2NPPzY9XPP6sdrBVIpYRcuybv8HvUq4etnR0/LlMd8dOmbTuePn3Klb+UjxceEU54hOL9+fnChRRIpaxbt7bSof/+/v44ODiwacMGfBs1pEfPnmq3LS1A6urqApCToyxsvA5KD/T/9+OPhIaGsmHdOkAWMv/Jp5/x7Tdfy+//nj17lSssvGyKQVkT2IcPH5YrEPTt30+hPWVnGOvXr8+w4cOxsramqKiIsLDrNG9evsns1StXcXJ2Ytr06cTExLBz+w6V1/Lly5e5U5zG16NHLxxq1eTnn35Sa8AGkKAiTa1uXVmpxl59+nDu3Dn5NVYVD4LyKJ2mBTJ/kLSnT+XLyoZaCwQCeVh5CRHF3kQNGzVi8vvvs3jhwmrP8PXq1Zs+/foyZdIkhdBXU1NTli1fzupff+X8uXMK+/ywdCkRkZHy6BSHmjUZO3YctWrXks8Ep6WlceL4caUO8bTp0/Hy9lb4nYGBJ9m4foPK9tna2fH5558TEREhr3DUqnVrRowcqWBo9+D+fT6aI4uCMTEx5suvv1GKTszIyODyn5fYunmLwoC9Y+dODB06TP45Pz+fW7du8euqVQqGnEKhkKHDhtHc318hCjQuLo758+bJP/v4+DD43XflAoJUKuXq1Sv88vMqtVEfz9LS5IPwJk2bYGFpwR9//AGFkPr0KbUdHTE0NFC5r52dstiZmJgo92KaMnUqTZo0UblvWUru2bOnz8iXRUVGySN82gW05+b1G3JDX1dXV3R1dWjbri15eXmcO3eOpk2aEHLtGkFBsv6eACiUqr8PqyIQzCgjnqnyIHBxdaF7jx7cTbhLo8aNyM3N5bffNtG+fQCubm7cvXuHjes3yK/3kaNGUiAt5MH9e7Rs2QoTU1OSU1I4cewowUHBCscWCAR07doN34a+mJiZkZWRQULCHbZv26rSvLVd+/bUb1CfX1f9QoeOnXD3cMfGxoasrCw2rt9AZGQEo0aPxsTUBF0d2fumbft2NGz8Ig3vl1W/kPUao4v19PUZPGQIN2/cYPNvv8l9zzIyMti8aRPW1tZ07tKFzZte9IX8W7SgVatWmJqZUVgoE+C2bt3Kk1I+E1paWrw/bSo3rt/AxMQYT08vjIyNSXr8mJ07dhBXSvSq6jkwNjam/4CBONSqiaGhIc/S0rj61xUOHVI2mtbV1WXi5EmEhYaRmHiXgA4dcXR0RFdXl+th11mzWlbBqyrXAYCbmxtdu3fj8MFDWFpZ0aRpExwcZELcH4cPyz1E/Fu0oEnTF/dfdFS0Wg8CPT09Or/zDu7u7phbWKClpUVqaioH9x/g6lXFPpKWlhY9evbE08sLU1NTMjMyiI2NZfu2ba99UkPDq0ELZKpsaFjYP9Z0sKqUnnkGqONeh7z8fKXZDC1tLQI6dGDnzh0UFRWhra3NoMGD2b1792t9IJZwOzqmUjMsrq6u5QoELVq0pLCwkJzsbBwdHeneowcHDxyQv6BBNuPww9KlKkNHVWFuYa4QgaAnkXVIbGxtFJaLq1CqsbDYC8LZ2YWWrVpx/tw5ioqKaODjKxcI3OrUQSqVKnXCi4qK+O7bb+SfGzVuhI+vL6Eh13Co6YBDTdWl/LKyMjl88CCeXp7ySImADh0qVZ6yIg4cOMDjUrnp5XH61Gn8W/jTwMeHPy9dYsf27VUqDfWqeKdLFwYPGUJiYiIrfvqJ//34IxQV8eXixYwcPYq+/fvh6ubGip+Wk62ipBxA0OXLBF2+/NJtmThpIkKhkKt/XaFnz14cOLCfoqIivL29EQgEXL9+U2H7U6cCqefpxftTp7J2zVqFtIvBQ4YopAKBLEWghC+/lpkT3rx5g5MnXpRSdHV1IyszE6FQKDdyVYetnR3OTk4kP3lC7959VG6TkZnBqcBAhXN38MABoqOiiYqKxLdRQ5UlH23t7Pj6m28UlhkYyDrkOTn/vFKzZV3Nq8OqVT+zapUsLWT4iBHyiCN1REZEcjv2hbBYeobRwsKCiZMno6WlxeHDhzA2NqFtu7YVtuFJUhJzZs2ie88edO78DrUda8ufTSKhLCKvoEBK8pMnJD95gomJMc4uzhw5/Ic8JcLDw4PklJQKI4j0JBI86smierS0tendtw87tsneDzpiscqIsBr2NZSWzZg5Ex8fH/nnA/v383txqoNYLFaKmHNxcaV1mzYq26Stra2U/lGSWhQaEsLD+/d5f+pUPv7442q9M2/dukmffn2pX78+50sZkjbwaQDI3n2lBQIrK0ssLC2JKI4OkrVvNgUFBaxds5bQ0BAc7GvQrkMA/QcMICU1hUsXLsr3r1WrFuG3brL0h6UUFRUhFAoxMlLta+Ho5MjMD2dx9MhR9u2T9SnMzMwYNXo0F86f5+DBgyQ9foxAIMDF1VW+n5a2GH19fX7ftYsD+/cjEolw9/DAz68ZAR06UigtUjAnDvrzMn9e+pPn6enoSSS4uLkyecr7DB4ymNW/rpZv16t3bwI6dODM6dOcOH6ce/fuycpiurrIt3GrU0duRLblt9/kIvB7Eycy5f33WfqD6pKbaWlp7NixHZFIRLuA9ggEQvJy8+T+Ou9NmiiPOFNFec9OVbn/C7/4gtzcnHJL31bEzFmzqF9fVqL11s2b/LR8OZmZmfj6+ipsp62lJe97qEJUhfSC27EyIczJyRknZ9UeBAYGBjg6OlFYWMShQ4fo3qMHkyZPISwklGNHj8iihJr5cfz4MQDsHWpiZGiIt7c3J44dJzU1lXbt2zF02HDy8vIIDQmVH7vfgAG0bNmSAi7rrQAAIABJREFUkJAQDh48iI2NDR06duTD2bP5YuFCpf69paUVjo5ODB4yBEcnJ2Kio7l18yZGxsbY2NoQGRnBnTt3SElJwcTYBFs7O5KSkrh/70WFnoKXSM3MysoiPj5ObtB85+5dkp4omkV27tyZwsJCtm7eTEFBAVpaWjTw8cHCwoJ7iYlER0ZR172ufPuevXvRvn0At8JvERh4En09A1q2asms2bP5cslinqbKJj+FQkFxuoAxUqmUI0ePYGhgQMdOnRk/YQLzPvmEvOLfVpVzYGRszNziVKvz587x4MF96tRxp0OnDkj0dNm1UzGtTCQS4ejoRG5OHl26duVe4j2C/rxMvrQAJ6cXfh9VuQ4ADA0NcXR0olXr1njUq0d0dDTnzp1FJBRhbf0iZTA1NUV+3bYPCCAt7Zna/9eQd4fi5e1FyLWrHDt2DImeBGcnZ+xr2CsJBNNmzKBGjRoEBwURFRmJja0tbdq2ZdqMGXz95Zdqv0PD24PW9h07uX///ptux1vJ48dJiEQiLC0tSEp6Qp++fQno0IE/L/1ZxuTwBW3btqNbtx5ERcs6iWZmZkodrqrmir4MNWvVwtHJkZs3blDDoSZrV69mxsyZRIRHqP0NlaFJk6Yqnf/btw+gffuAah1TiKxDnV+QT3p6Ort27sTezp4PP5qDiYkxaWnP8PFtSGxsTLn5rba2trw3YSIgcyhu4OOrdtvSlERK1KtX76VL/wGcO3uu0gLB4cOH2Ld3D0PefRf/Fi3w8fVl/bp1r2SgXVmGjxhOQIeOxERH8/1338lTjgCysrNZuWIl0dGxDB02lPkLF7L8x2XcS3zxknwV5lElVT8MDAzQ09Pnj8OHEeuIZYMEnwZ8/+23+LfwJycnh/BbigJB0qMk9u5ewGefz2fsuLEIhQJ5GkyJz0RFPE9/jpm5ufzz2jWrWbBoEaPHjq0wX75jp46ATGwrrzTg3bt3lGZ7yzNpVIeevkyUy87656jxPyxdioWl5Usdo7IiZllu3bypNvolOTmZ/y39gUcPH5GWlsbwESMoLCxk5PDhao83bfp0rKytycvLY/eu3zl5/IQ8FQpApCUbHJR+Vg0a/C55+fkcLjWTNG3GDE6dDKxQDG7ctCnJT55gbWPD8aNH6d6jh3xwa2ZuXimfBoC/goJ4cE/2zi+bwpaXl8fY0aPln+fN/5z79+6zdo0sfUEsFrOmOGqkZPsSobllq1aMGz9evk4qlbJs2TK+WLyYUaNHs6IKEUUlxMbG8jw9Hfd69RQEAi+v+kilUlxcFUvledf3QSqVci0kRP77DA0N+WjOHPmzODxdFvnz0ccf07tPXwWBAOB5RqY8wqiwsFBlh9m7vjdTp00nMDBQLg4A1KxZE5FIxMWLF+SCT1FRkcqKFyVRmlKplJs3bnDzxg1ycnJp3yGAP/44TGrxzHPpayorO5vrYdcJvhyEd4P68uUikYhOnTtz9coV1q1dK19eUFBAZMSLZ8ugwYN58OAhXy5ZIh+0h4WFcfjQIQYOGoSFhUW5odp+fn5IJBKKigrp0bMnISEhJMTH89Vi9VVMKkoxcHVzw9bWlksXLrzSVKTwW7c4e+Y0w0eM4u7du2o9tAQCYfmD3FLvwYo4UFyatlu37jg5O5W77anAQG5cv45bnTr4+Piwe/fvZGZm4u/fAls7O4VtJRIJv23axF/BspniW7dusvCLL2jTtr18YGhkZETz5s25HRvL+lLXQFZmFn3796Nps2ZqTU0NjQyZP2+ewrNKXCxunAqUPWudXZxp1KQxt27c5Nwr8jy6HRvL0u9fiFKrVq5U2sbb25sL586Tl5+Prq4u02bMwM7OjsePH9Opc2eysrLIL46ksrCwoHWbtty4foPVv754X//1VxDzFy6ia7fuCpEGIBvQL160iJRis9jcvDz69x+Ap5cX165dq/I56NatGxKJLt99+y2Jd2XpaSHXQsjJyaF1m9acCjwl/67S1HWvy6+rVnHz5ot+jbiMwFSZ66Asrm5uLJw/X2HCqfRxI8IjiAiXpcO0a99e5TFAFhFQz7MeoaEhbCgVUXX+7DmlbRs3aUKtWrXYt2evwrv67p07jJ8wgWbN/Lh8+U+136Xh7UBLIw7IMNQ3oEuXrji5OFGzZi1+27SRR8Wh3jVr1UJP34BOnTtz7sxZhYG1l7cXPj6+8gGlk4sLYSGhhFy9RuvWbcjOzlaZogBgZGL8yly81dGzZy8iIyJJevKEGg41CQsLK863zkEikTBpyhR279pFRqmXp4WFBcOGD2f9urVq1cQjf/yh4ASvrsyhRCLhl9WrVR1CLXfv3GHOhx+SlZ1Neno66WlpvNOlK38FB1Ojhj2bf1MfVu/i4sr0D2bwPD2dWQvmk5b2jI8++Rg9iR7zPv20Ut9fUem/18XTp09Z8dNPBAaeZMy48UyeMoW2bduyZs2aapkEVhYzMzPGjB2Hl7cXl/+8xC+rfkEqlSqFwAKcPHGcB/fvMWPmTBZ9sZgNG9bLwz3DQkIVygSWpSQfeu2aNaSmqnZuLwmZzsjIYOH8+UgLCiiQSrmfeI+Jkyczf8ECzC0tCf7zssoQ/mfPnvHVksV8/MknjB4zmkKplPPnz/P40aNKizWlBYLU1FT27t7DiFEjOXrkCIl376rcp4ZDDdq0kc04X7xwgV/KlCgUCAQs/vIrxNpar6S0H4CxkSyt51m6esX/beOXVb8g1lHv7D1mzFiePEnmwAH1aSylRalXSemBVGUwNDRUSE0qPZADEBeHmOcWh1MGBATQ3L85V/66gnOpWV2RSKQUkZWTnUNMjKKAGxAQwJWrV+jatRuJiXe5dfMW9jVkkVEPHz5kzqxZmJiYsGz5j8z//HPi4+JftLVUqsGFUoOEykRJvAxJjx/z+66dDB02nP379lb5f1dUVERoaCj16tWTLxMIBHjU8yDoz8s0b+GPsbGx/Nx7eXkSFR0lj1Zo0rgpcXFxKu/9i+fPM+6993CoWVPtfa0KV1dXxowbR3BwMNtLvQMBEhISyMnJoXefvqxd/StJSVV7bh87eoQOHTvi59ecw4fVlxp7kiyLRinBu359JBIJZ06fVruPnp4eLi4u7Nq1U2lGPzQkhIGDBuHo5FiuQNAuoD3Pnj1DIpFw4/p1ZnzwAQvmz8fe3k7tPqpSDErj36IFrVu35tzZV2u0e+SPPwAYPmJUudtp64jJe81Gr6rIeC4L+c/KzKKoqFAuYGTl5CCR6Cpsm5+fT8i1FxVTcnJyiIgIx8fHF21tbfLz83Gv54FIJOKvvxQrpgQFXaZv/354eXmpFQgCT55SmnR5E+ekLFpaWpiZmxOfIAv379ipE1ZWViz97jsS7tzB0spKIVXWx8cXLZFIoewwQEZGJnFxcTg7KYs2MTExCgP2xOLULhNT02q12bdhQ+Lj78jFgRKuh4XSrn073Nzc+PNP5cHx/Xv3FcQBUP4fVOY6KMtfwUFK0ajV/d8KhQKsrKwxMNAnoxwDzoaNGiGVSpVElJs3b1IgleLi5qoRCP4BaFW8yb8LiUSCn58fdvY1sLe3w9bODjMzMwBat9HibmIiZ8+cIfFuItnZ2UilUnx8G1LPw4O0tDS2bd2icDy/5v40bNiQ27GxmJqasnb1r/Ic4RKBQN3MkK2dHba2NipNnODlTQqdnZxo1LgRPy5bhpe3t3x5yUxo02bNqF+/PhvXr1dQyVNSUnBwcKBTp3f+NhMpQyMjfEqVl8sqDsEuKiri1KlTdOnWDTNzc7Kzszl7RnVHonWbNowcNYqnqaksWrhQQdyQ6OqqNWWUFkgJD6/Y6+HvIjIikrmzZzN0+HDqeXqS/uz1DAAFAgGdO3emV58+6OrqsGPHdnmeaXmEh4fzxaJFzJo9mzFjxqIl1CIw8CRpaWnyF5GFpSVffPEFq1evlvsBlAxSoqOjK1W/urRpYGhoKPM++4xPPv0MLZGI0+V0hJOTk/li0SKmTp+uIHy1a9eekaPVdxbv3buvMvT9zJnTDBg8iGbNmqkdSIwdN57CwkI2rF/PmLFjOXXqlMKsYcdOnahRw57169aq3L80derUrVSUkZ29PUCFZfLeJkpHSox77z3MLcwVZh9z8/J4lv7slYkorxNzCwvibqs3ZysxqHtW/HwXimShyo0aN6JRY8UIpWZ+fvKqJKB8Lfo2bEgNBwdWrlhB166y6JSSNB9f3xfpAmlpacREx+Dv35L4uHiEQiFjx4+nbt26zJ0zR6URZ2mqmmJQGU4cP8HjR4+rLeyEhFyjZatWODs7c/v2berUrYOenh579+zGp6Evvg19OX3qdPEMlye7f/9dvq+FhTk3btxUedxHxTP8tja2VRIIJk6ezPPnz1m/dq2SeWFaWho/fP8dEyZO4tvvfyA6OoqjR46Wa2BZmqSkJxRIpViXqhpSu3ZtBgwaiJ2tHcYmJohEIp6npyMQvAh9t7KyAuBxOWkq5sXCZ//+A+jff4DKbQwN1HtWeHh44OLiyskTx2nVug2/79rF5Cnv4+joWKERcXkpBjXs7YlT4VGh7jlYtqKNyra6e1C3njt7ft9d7nZaIhGGhoZkZDwvd7vXQX6BbJBWkJ9PQcGL81OQn6fgXwEy8bFsekBJ5Q8Lc3MePnqEhbmspHVSkuI1kJOTw7NnzxSE77I8uP96RNeXxao4FL5EtHJxcZFFrdyRDeKfJCVx+dKfNGzUsHh72X3QvXt33umiWBrb0tIKPT09BAKBwn1btn+VnSsTdEvSZquCsbExYrEYKytLJT8K7eKIMgsL1RF0jx9X7D1VmeugLA8fVm5ipCIKCgq4fOkyfv7NWfzlVzx5kkxUVCQ3wq4rGXJaWVpSVFTE+9OmKh1HwItnUQmGhoYMHDRIYdlffwX/K0tJ/pP4zwkEhYWFDB85EqFQSFZ2NrHRMQgEAh4+eqgyTC7hzh1atmwpn9HMKpN3vXP7dtatXo29gwOeXpWrClDCwwcPlMrwlOZlTQonTJnCo0ePuHb1qoJAUIJ/ixbEx8WTnJysEPZbVFTEuXPnaNe+vdx/QU9fH+fikDmBUFhlDwI7O9nyrIxMueOxjo4OxsZGfPLpZ7jVcVXo8JTmVGAgPXv1omnTphw7elRlJ7d9+wBGjBpJ4t27fPvNN0qKqXVxuSVVZGdn8964cSrXvSkKCgrYsG4devr6r63OvUgoxM/fn5SUVNatWVOllJM7CQl8sXAh4ydMIOy68kPcw8MDPX19lXnVbm5uWFgod1aePn1a7iBCrK2NoaEBUVGRFYbkp6amKhhzlUZVlEOvXr2R6Kku3VlYWEhsdAwOapy2e/fpg5OTE7t27uTsmTO4ubny3oSJfLFwAWlpaTg4ONCnb18SExM5U8pcSx2JiYls37ZV5bo7pYzrvLy8eJ6erjZ01tPTi569exJ+K5x9e/e+0Sodqqhbp47KqhjGRsZqxbzIiMg3Vo609DNQR0cHc3NzLlw4r3Z7Y2OZQPC0OFT8wrnznDl9Rul+/mX16nJTDLS0tBj87rv8FRys8FxT5wESHBxE3379uXrlL4YOH46FhQVrVv9aoTgAr66KQWlkho/V7+jdunmLgoICfHx9uX37Ng18fIiPk1WmCQ0JpZlfc06fOo13/fqIxWJCQ1WH2r4qNqxfz+B332X+woV8uXixkvlbZEQkM6ZNo2u3brRo0YJp06dz+OChSovthVIpguJ0u9q1azNvwQIiI8JZt24tsdExWNnY0KdPb4XUuZLtCwsrDtE/eeI4IcUpGGVR9/wVCAQMHjKEkGvXSH4iEyTv37/Pgvmfk5ubq+Bn5OrqSoMGPuzapWwOXBYtLS1qOzqS9vSp0sAtJSWFy5eUZxmjVKRrgEyAFolEdOvRAz2JpEKRcfyECbRo0YLs7GxC1ZyPN0aZrAapiqoVRcXLdIuNoEuugaJC5ed8YVERwnJSJdQ9S940kmIj3uxs2aBdIpEoeZlkZJW6/4p/Y2xsrJIoVZJnLxKJFAbZr7JKW0la5tPUVPn3lSYyMpK4eNWGrdnZFT+fK3MdlCXnFf5vt23bSkhoCG5ubnh6ecmNhrdv3calS6VStQQCcnJyVJ6D27G3SXuq2D/X0tLCwsJCYZm+vmrTUw1/H/85gSA3N5d1a9fy4P59bt++TVFRkdo67BYWFhgUuwH/8vPPPCpW52o41OD+PVmJtZc1khOJRNja2ap8Mb+sSWFsdDRXr15ROTAwMzfHy8uL9aXySUtz8eJF+vTti7e3N2FhYdSuVUthgF1VD4KADh0J6NBR7kJsZWWJu7s7YrEYEzNTjhw5QkJ8ApMmT1baNyMjg/i4ONzq1CE4KEhpPcjURhtbG3bt3KlyQJ2QkFDpFIO3iddphlkglfLNV1+RlZVVrZfko0ePWDh/vsp1DRrIZjXzVAxKxoxV7VNw6cJFuelcWcRiMZOmTEEoFLJzR8Udz/JQ1XFs1669WoEAoLCwAC0t5dD4lq1a07tPH4IuX5Y7om/auImFX3zBZ/PmsXbtWiZPmUJ+fj5Lv/++UoP0rKxMtZ3b2o6O/FQqT7M8j4oOnTpSp05d6tSpS2xszFs1K1+jhj2WVlbcvh2rtK5O3TrMqqtazPtgxowK022863uTmZH5ymtjC4APZ82SC5nPnz/n/NmzmJqaqqz6Y2trS1ZmplzQKCsuVxYbWxssLSz49mv16TulOXP6DD179mLuJ58QdPky333zTaWrElW1ikFl8W3YEFcXmVGfoZEhIpGI7KzKnY/s7GzCQkNp1qwZv+/aRcNGjTkTeAqQhU9Pmz4NAwMDmjVrSmJiokJ0UnJyCtZqSobWqCGLwHn4qHJVY0q4HhZGdHQ0H82dy+yPPuKLhQuV3jlFRUUcOniQQwcP0qdvX3r17k1IaAjRUVHlHtvCwgKxWCyfMfVr3hwB8MN338sHNQnx8URHxygIBMkpsnvCoWYttWkNJWHUAqGwys+CDh07UKt2bdauWYOHx4t0j9zcXDzq1ePRo0fUq1eP/PwCCqVSuvXoTnZONocOHsTWzo5OnTuxYd16peM2atxYNuNqbU3jxk0IDn7xfk9OflKp/o+ZuTkjRoygfoMGCIVCwkNDObh/v1KKTln27tnD2TOnuR0T+1rKsL5KDA2UB0tGxRF5JddKiWdFSUWpEoRCIUaGhkRVImqvXN6AvlwSEWNpYUH6s2ckJycreTuUPFcA+bsh6M8/eVDJalBVooJzkJaWRoFUSurTp3I/ildJZa6D101kRASREREc2L8faxtrZnwwk+49eygIBMlPkqnhUKPS5+Dp06d8pTEufOv4zwkEQKVy3Tw9vZj8/hR5WSQnZ2f5LEjTZs1wrO2k4JpfXUxNTVm8ZAnL/reMa1evvvTxSrN+3Tq1s89du3YlPT2dS2py0p4kJZGQkEBTPz/CwsIIDw+XzxKs+uUXzp49q+BBoI4SD4JNGzdx8sRx+fJnz9KJjIjk5MmT3LgehlQqVZkfBtCqdWvc6tQhJyeHwe++y5JFi5Re6Onp6WwpJyy7vBQDgLjbcWpnYt9WPpj5AQ18fJXObVVQVfroZbG2spKH/A0ZOlTJGVtV2czyEAgEfPDhh9jb27Nv716Vhl+vAxNTU6ytrTA2MsajnifHjhxVWN+pc2cGDR5MWFgYv/z8QtjIzc3lh++/Z8GiRXw0dy5ZWVksXrTolbzA7yQkyNNA0tOfERgYSG1HR+p71+fwoYMK98WtWzflbvWWasIa3xTduvUAZM/Sw4f/4E5CgnxdcHCwSmfzihAKBAwYOJCu3bqxZfNvr1wgyMzMZMQwZdPCKVOnYmVlpSRAOjo5kXBXFvGxYdMm+btEFV27d1NpbDnt/fe5l3iPH5ctI+nx40qVjc3Ly+OPw38wYNBALl269EpKFhsaGVG7di20tWUCQdla9DY2NrQLkAnDzf39lcpkCQQChd8XGxvDufPK5lbqOHUqkFmz59ChY0csLSy4eFH23goNCSE7K5u27dpRv4GPUknb4L+CGDhwEN71vbkedl2+XFtbmy7duvM4KalK6QUlPHzwgEULF7Jw0SLenzaNH777Tq34dyowkF69e1OrVq0KBYK27doBcOWKLE1RICyJDChfwL0edp3c3Fw6durEtatXVbYlKyuL2NgYmjf3Z9f2HVUSrGJiYggODiIhIUFBIBAKhbw3YQKnA0/h4upKTm4OP/34I3Z77BgwcCB/BQWRm5tLq9ZtuB0Tq2A0CdCqVSuys7NJfvKEocOHcevWzSq/h5s0boKPry/Xw67z+66dJJR6lqji55UrSEt7xpOkJHnZu7HjxhEcHKRwjZRQVFSoNrrx70JPT0+eYlOCu7s7j5OSeP5clh4RHn6LoqJC6nl5ys31AOp5eiISiV76vVmSKmWgppzl6yAjI4Pc3Fysra25ffs2p0+dYsrUqUyYNIm/goKoV8+Tuu7u8jSB0NAQunTtgp+/v0Kq0auiMufgemgoHvXqKXijvCoqcx38nTx+9JiYmBjqeXoqRABdvx5GXfe6eHp6KvkqaPjn8J8UCMpDIpEweMgQ2rRtS+Ldu/z4v/8xetw42gcEcOzIEbKyszExNiEnt2Ln8AJpAUJh+TV0a9aqhUAgJDNTeaDm7ObKwIGDVOyliHFxrmtZ1IkD1jY2tGvfnq1btpSrnIeGhNCxY0el0L9XQW5ubqUEltZt2jBq9GhOnjxJaMg1Ppw1m6kzZrBs6dIKy86VprwUA0Chfvs/AQMDA/kMUuvWraotELwORo0ZQ15eHls2b2b0mDEMHzmSXTt2VOtYEomEqdOm4eHhQUhIiLys1t+BkaEhs2bPkbmiX73GseIwaz19fSZMnEiDBg24euUKK5YvV7iP9PT06NSpEzrFs61isZi27dqxZ/fuCgWZokLKfWYUFRUpzaq5ubnSt38/rl27qhCyf/zoMc6cOs3qtWtITvl7ZhcqQ526dfDzb87JE8dxdHJmxowZ/PzzSnkt8+pgbGyCtY0N73TpwvZt2zh+7OXuh0JpIQKBoFLPvtq1HeUzuCVoa2tTs2ZNThS34/ChQwjVDDI6d3mH+Pg4oiKUf39JveiqiMdisZig4CD8W7ZkwsSJfLlkCQnx8QrbiEQiOnd5Bx1dXTy9vCp8fkilUj6cNQsQcC/xHnGlQkdtbGz4+NPPEAggNOQaAwcNws7Olp07dso7yFevXFEIQ68qN67fIDk5mcHvvsvNGzflkXuFhYVcunSJPn37AiiZsB374wjNmjZj4qTJbN2yVV7msEu3bliYm1c6KkMVyU+e8NOPPzL7o4/oP2AAO3fswNTUFIeaDsREx5CdnY1YLOadrl0BlGa0+/btS8qTZMIjwtESiWjXIYBu3btz+tRpeWWpyIhIOnd+h0lTJnPqRCDR0VEUFhVhWsZALS8vj+1btzFi1EjemzCRTZs2kpWZiYmJCbUdnQgNkQ0Yt27eyseffcon8+axfdtWbt+OIzsrCxdXV/Lz8tQOruPj4vl1lXIVF7/mzdHT0+P4ieMKpRz379tHQnw8OhIJTRrU5+L58/Ts3YdLly7J39vu7u54enlx5vRpzp8/z8effMKXX3/Nip+qVu3iwoXz3L4dqzZioKioUC60ACqfM40aN+Ze4j01AkHlChnUcHDAsHjgWJLv7+zkLPe4un//QbX9hKRSKYMGD+a33zaRmpJKt+49MDI25tTeF+/DtLQ0IsIjaNiwEXfi73Dlyl/Y2dvRv39/8vPz1RoUVpbUlBSePn1Kq9atAZlYXVhUSEx0zGstjx4WFkbjZk25dOkS0dHRbFy/nne6dGH4yBE8ePCQ/Xv3yU1fHz96TMi1EFq3aSN7Nly8yOPHj7G2scangQ8Ghgb8vqv6wkFlzsGRI3/g4+vLjJkz+X3nLqKjoygqKsKjngeNGzfl4IH95XqFlEdlroOqoKuri6PTi5LeAoEsIsHdwx2QmRmWlAa2sLCgZ+/eBF++TEREBHp6evj4+OLp6UlMTIzCe/Lyn3/SoWNHRo0Zw769ewi5FkJmZibOLs74+DbkfmIily5dqlabNfx9/OcFAqFQiJ5Ej9ycp4jFYhYvWYKFpSUnjh9n65YtSKVSft+1i8/mzWPy+1P5eeUKnJ1dFELh4EXuUWGhYg6diYkxAwcPIjoqmoICxdxZia6E3n37yrwQYpRDbZ2dnNTOqr8MT1NTZeVHAl9UV3jx/nzR/uth1xGJRGhra1c7D14gnzWrmsAgEokYNHgwnTp3JjTkGr9t3CgbHG3bzsDBg5jz0VyW/W9ppWcb/qkpBurIyMjg5InjNGrchEOHKjYW/LsYOHAQHvXqsWvXTs6cPo2JiQl9+vbFz8+PG9eVO1/l4eHuwahxY7G2siI2NpYVy5e/EqFKnQHgvXuKFV3u3r2rclAjFAiwLH5GbP7tN3mbLCwtadWqFR07dUJPT48LFy5wcP9+uvfsSUCHDrRo2ZLAwJNcCQqW+3CU5fnz5zg7O2Fra8vDCkIkhUIhrm6utG7TlpycHJVRGd179iArK1texujvRD77Wep/ZmFpyeQp7/P8+XN2bN+BRCJh3vz5fPLpZ1y9cgUdHZ0qf4+llRUNfBqQlPSEVStX4lCzhoKwKtKWCS71PD3R1dVV2v/YsaNKqWJJyUkIBAJmzZ6t4PtQFmsbG6ysLDl9WrFSTatWrRGLxfJrvuzMdmnaBbQnKiKq2oawOrq66IjFdO/ZA09PL9zc3Ni+bRvffv0VCxYtYu7HH7Nt61bOnjlDUVERJiYmfDhrFpZWVmzZvAUjY0O+/vY7Tp0KZMH8+QqiSGmPnBHDhmNiYoK1tQ0mpiY8f/6cvNxcZs+RCa+fffIxaWnPaN8+gGEjhtPcvwXBQcFERoYrRM+IhCIsLC2xtLKisECKjq4OqampHDp4kPI4e/oMffv34+xZRYPSU4EnCejQgXNnzyrlUhdIpSxauJBhw4YzavQotLRkXjMJ8fFFplx/AAAgAElEQVR8teTLapUXLU14eDjHjh6la7duXLt6DVtbG8a9957CNnl5eWzauFFJpMnIyODDj+agJZJdn1mZmezft09BBL129SoHDxygS9euCil9hYWFSs+HwMCTZGRm0KdPH1aVKsmal5fHhHHjKJBKiY2NYfGCBfQfPEhJMD9z+rRCicSylO0DiEQievfuzcmTJ8jKzERaKEWsLetSllSfaNuuLZ07deaTjz9m6bKWNPVrxqULF9HX12fkqFFIpVIO7N9PcnIy69etY/SYMXzy6Wfk5eUVD+xsSHr8uNznfkZGRrnpBBkZGdSsVUtttYoS08vYWOU+GMh8OSqTWtOte3c8PDwUlg0aMkT+95Ytmwn6s3plix89ekTaszQ+nDVbviw4OJhzZUrMbd2yhVFjxtBvQH/6DegPyKIrV/28Ui42VpfCwkLWrllNt249aNmyJZ3fkZkAfjJ37mudvT588CCfzZ9Pc39/Ll28yNWrV7laRjA9e/aM/O/t27aRm5tL23bt5BE5IKv0UlL2uLpU5hw8fvSYn1euZNDgwYyf8OJZkJubS+zt2JeKVK3sdVBZbKytmThJMa3X2dlZviw5OVmeRlpQUICbmxv1678or1pSwrWseXtBQQHLly3j3WHD6D9gIP0HDARkAsedO3dfebS0htfDf1Yg+PiTT6nrXlf+OSQkhLy8PHbu3ElmZobCbHJMdDR7du+mT9++rFy1isLCQrkrsZaWFit+/hmJREJBQQEPSnXSTxw/jpe3N127dqN4EkEBqVTKgwcPWLd6jcrZ8ONHj7F5828V/hZXV1c++1y1j4Iq8vLy5PXAS/IjS3hUyvE0NjamSsZ1JbjVqcOnn32msOxBFcppisU6fPfDD5iZmXHkyBF2bd8u7yAcPnyIlNRURo4ayZdff8PcObP/cakBr4pNGzexaaP6ko9/N3369aVr925cD7vOH4cOA7Bv715u3rjJgIEDadykCQBff6M+cqSgoICpxTn7w0eOwNrKiqCgIH5dteqVmdNV1aSwLBkZGSxasICs4hJ3hkZGzPzwQ5yKxbzbcXFs27JFHk7866pVnD19hoGDB9GlS1e6du1GWloau3//XanDcvbMaby8PPn622+r9Jt27dypEMUgFovp3bsPXbt347dNm165sV98XBxJj5RnQXR1dVm3YQNaWrJXS3Jysjw82sTEmHmfz0dXV4flPy4jNzeX3NxcPpo9mx49e9KiRQvMzMxo0qRJuVUcjh87rlDq1L+5PxHhEaxcuYKszEwGDh5InTp1lfar615X4ZlfwuWgy0oCwenAU7i51aFx48blms9mZ2dz8sQJeaRACc39m5P0+DHhEa+3OopEIsHb2xuRSETHjp2Iiopi3dp1BF3+k7y8PL7+8ktGjx3L6DFjMDQ05OCBA3Tq1BlLKyu++nKJvBTi1StX6dq1C3M//hgDFXmuqpj1wQfk5uZSIJWyccOv8qoxgYEnuXXrJu0DAnBycmLQ4CEKqREFBQWkpqSQ/vxF9Z6oyIoH6vv375O/t0pz7979cqMT8vLyWLt2Db/9tomaNWuRnPxErXfQB6VKppXl7Okz8pKupdm2das83S42NobLly/j6uKKSFuEVFpIfFycShO4Y8eOcfXKFZxdnMnIyOBOwh2VqQS7du7k6NGj1K5dS/574m7Hqbyngy5fJujyZczMzbGxseFZWhplS1nfjovjq8VLEIvF1HZ0RCwWczs2tspGdYWFhWzfvo3wYvHxxvUbDB02VOnevXDhAmlpafy8YgU3bsj6VbPnfIStnR1bNm+Wi0fnzp7lwf379BswAA8PDxwcHPj2u+/Ufn90VBRfLFpUYTsvXbjIwMGDWLxE2YS6hOth19X2dSorEKwq5Q1THqEhoUydMkX+effu39m9+8Ws9tLvf1C1G7+s/BlHRye0tEU8evhI5aA8PT2dZUuXYmxsjIWlBRnPM3jy5InaFJUd27exY/u2SrUb4O6du6xc8VOlt38VPH36lPNnzzJw0CAsrSw5sG9/uYJRTk4O27ZuZffvu7C0ssbIyJCkx0kKpQxBNjte+v9QwuNHj1UuL6Ey5yAyIoIFn3+OmakppuayCJKE+ASVkRaZmZnlfl9ZKnMdACqFlLIk3LlT6e9OS0tjzqxZCIVCbKyt0dXT497du+Sp6VskJyezbOlSDAwMsLS0REdXl4T4+JcWqjT8fQhs7ezfGmtrdWZPrwNPTy95beWU1FROnwqsMGS9rntd3NzqcD0sTCEUz9nJCYm+PnG3b8sHDS9Lr969SYiPr5Qjs5m5OR0COnDp0kWVruAgM4myr2HPwf0HFJZbWVvTtk1b8qX5REZEEn7rVqXaFh8XV64zdZ9+fdEWyUzdbty4XqkygrZ2dowbP44tm37D1a0Od+4kEBGheubT2NgYe3v7Sh132PDh6OrqsPrX1RVu+3fTpElTOnfpzPJlP/5t135lkUgkzJozm0sXL3HyxIkKt7e0sqJNG1n5K1UvcC2RCEcnJyytrTAxMsHA0EDuvFxCSmqqPNzZysoSY2OTCs2mDI2MmPHBDI4fO85lFfWFS5CV/WyicqbWoWZNdHV0iImJwa1OHQYNHsSSLxZXOnSy/4ABZGZkEhwcVK7XgKGREQ0a+FC/QX02bdhAuooSpyKRCKlUStNmzahZs6ba0HSQlcoKDg5SMjl1dXXlw9mz2bFtG6dOnarUb1CHu7s73t715RVNKiKgQ0fMzczIyMggKDhIwVSwe88e3Ai7rjaU2cTEBEdHR0xNzTA2MUasrdwxP3f+nFK0hI6OTqVc+v8urKytcXF14dKFixVuO/ujOVwJvsKpU4EVbisWi/no47ns3LGDyAjZoLp9+wAS7yUSEx2t9v/TsFEjeW66lpYWtWrWVBvFYmpqiq2dHSKR8nWX9jSNBw8eKLwrJRLJW+uC/jZiYWnJD0uXsvrXXzl/rnozf2+SZn5+tGnblq9UDLglEgmOTk4K1074rXClvlWdunVwdXVTGzVib2+Pd/36GBmqL7t469Ytbt6sXFqgQ82amJgYq1z36NFjuReBKkQiEXp6eohE5aeLvi4+mTePgvx8vv6Pm7i1bt2Gnr17kZ+fz52EOzx4KBO+4m7Hcf0lqqT8U/ivXwcvawivoWqYmpr+dwUCDRo0aPg3Y2hkxHMVAoQGDRreHP90geC/iI6OTqUMQl8H//WBYWksLCxwr+eBra0ttrZ2CIUCwkLCKiWu/tP5r18HGoHg78XU1PS/m2KgQYMGDf9mNOKABg1vH1mZmRw+eIi71aieoOHNkJeXh0gkqnaZz5fhfuJdCqRVL0P8byQ5OZnz1cy3/6ejuQ40/N1oIgg0aNCgQYMGDRo0aFCDlkiErkQi91fRoEHD34cmguDvxdTUlDdb3FWDBg0aNGjQoEGDhreYAqmUnOzsald00qBBg4Z/EhopVIMGDRo0aNCgQYOGciiQSpFmZyOVShGLxW/MuFCDBg0aXjcagUCDBg0aNGjQoEGDhgooKiqSlfcsKEAkEqGlpYWWlhYCAQjKqTijQYMGDf8kNAKBBg0aNGjQoEGDBg2VRCqVIpVKNSkHGjRo+FeikTs1aNCgQYMGDRo0aNCgQYMGDf+9CAITE5M33QQNGjS8DoqKkBYWymd28vPzKSzUlAXSoEGDBg3/XsRisTzVQSQU8taUJtOgFgEgLSykoKCAgoICTSSKhreO/5xAoEGDhn8pAgEikUhuHKWjo0NeXh65ubkUFWm6TBo0aNCg4d+Djo4Ouro6Ct4HmjfdP4MiQCgUIhaLEYvFSCS65OTkkpub+6abpkEDoEkx0KBBw78UoVCIrq4uEonkjbpNW1lbE9ChAwKBAIeaNRk4cBBm5uYAmJgYV+uYtWrX5t2hQ9GTSKq8r5mZGULhi0e/s5MT8+Z/Tu3atat0HIFAQMNGjdSud6hZk3nzP6d+/fpVbuPrwsfXl4WLFmFlbf2mm6Lhb8bC0pLpM2ZQ172u0jp7e3venzoNayurSh+vW/fudOzc6ZW0zdPTi959+rz0cQyNjKq1n4mJCXXqKJ8XfX39Su3fvIU/nTp3RiQS4enlVa02vEkMjYwYOHAQVlaWKteLxWLatw/AwcGh0sccP2ECnTu/86qaqIBAIEBfXx+JRKIxRvyXIBAIkUgk6OvrIxAIqn0ce3t72rRt+wpbBh07dWTe/M+rvF/LVq1o2apVudv07tOHMWPGVqtd1lZWjBo9htqOjtXaX0P5aCIINGjQ8K9GLBYjEAjIyc5G+jekHJiZm+Ph7o67hzt167pjWTzo0NbSIiUlla7duxEaGoq5mRmfzvuMrVu2cuzo0Uof38DAgBkzZmBqZkbgyZNkZWdXqX1Dhw2ntmNtPpg+HQA9A31cXFzR09Or0nG8vL2YNn06u3bu5OCBA0rr9fQkuLi4YmRcPRHkVWNlbc2EiRNlkSWaWZr/HC4urvg2bMiOHTuU1jXw9aFho4asWf1rpY5lYGBA7z59uHz58itpW133unTo2JG9e/ZU+xhNmzVj/HvvsWH9Bs6fO1ulfb28venduzcfzJghX+bg4MCn8+axZfNmzp0t/3i+vr7Y2dUgIyOD8e+N5+uvvib81q1q/Y6K0NfXx8nZ6aWOkZBwh+fp6fLPBgYGdO3ejRs3rpOU9ERp+/z8fDq905nGTRrz1ZdfVuo7atWqRU6pZ3O7du0ZOXpUufus/vVXzp87V+42AoEAAwMDTYnFfyna2toYGBiQkZFRbuSjRCLBytoac3NzrK2tsbe3x8PDAwtLS5IeP+bOnQTi4+JfSZusLK1xcXGt8n5NmzYDKPeatrW1xbKagr20sJBWrVthYGDA8h+XVesYGtSjEQg0aNDwr0dbW5uioiJycnJemy/BRx9/jKOjI5JSs/rh4eHs37+f27Gx3L9/nwYNGgAgEgqJiopk3959vDt0KFlZWRV2DAH0JBJmzZ6Diakp+fn5DBw0iBXLl1MglVaqjQKBAA8Pd6Kjo6r3I0txPew6e/fsoV///tyOiSU8Ivylj/m6MDQyYubMmYjFYgoLC5n54YcsWbyY7CqKKyYmxsz7fD57du/mwoULr6m1rw49PT0GDxnCsWNHuZd47003543i5ubKw4cPefjggdI6b+/6JCQkVPp66NW7DwKBgF3FYoOWSMS6jRsrte/woUMr3MbU1JRly5dXuF1hYSEjhw8HICEujvDwcMaNH4enpyeBgSf59LPPVO63aeMmAk+eYMCAgfxx5A+V29y/f5+w0FDGjhuHSCTk9KnTattRkFeAUAAXL1ygnqcnEydN4qPZs8nMzKzwN1QVJ2cnZs2e81LH+N/SpVy7erXcbVxcXJDovXiWR0dF0ap1a5r5NScz87l8eXZWDrGxMZX+7hXLl5OVnaWwTE+ix+T336/U/np6ehpx4F+OSCRCT0+v3Ptn9NixNG3alPz8fFJTUrC2seHkiROcOHFC/oybNn262ii/8+fPs/qXX15L+8tjytSpmFtYsGDevErvU6t2bYyMDFWuS0hIwLehLw0bNSIvT7XwfzsmtsoTKRo0AoEGDa+V5s39aerXlE3rN5CSmvqmm6OEs4sLPXr24MgfR4iMiHjTzXmtlAwOc3JyXsvxTwcGclFHl8ePH3H/3j1+/uUXoiIjFWbfSvILtcXaAOzdswczMzPGjR9PXm4uQUFBao9vZGTEnLlzsbOzY+VPP/H8+XNmzprF7I/msnz5jwozYurwbdgQPX19RCJtBg4cBIC5hSzdoXWbtnh5eSvtc//+PbWD4X179+Lk7Ez682cVfvebQk9fn4/mzsXWzo61a9fwPP0570+dyqw5s/nu62+q1HHQ0hZjYWmJQPjPCOtt1syP1m3aIBbr8PPKFW+6OS+NUCjEvkYN7iUmVspXREskwr2eBwD1PD1JSkrCy1sWAv/8eQYJ8fGYmBhTp04dwkJD5etKk3g3kbS0NPlnaysr2ge05+iRI/Ll0sJCDh88pLCff8sWaGlpcfb0mSr/zpycHKXjAbzTtQt3797l1o2bABQWvRA7Hycl8f2339KwUSNGjR7N5T8vKQgIJSwungEXCYV07d6Nc+dVC5OFhYX8vHIlUMSIkaOIj08gIV71jGSBtABtbdkzbeP69Xz7/ffU92nApQsXq/zbK+LG9Rvliixff/stD+7fZ9n//vdS3zN85EiVaVeTJk9S+JyYmMgnc+dW+riRkZE8e6b4vDSuZJSVjo6O/Dy/zVhaWTF02FCCg4K5+A8QUt9GtLW10dHRUetJsG3rVnZs20ZycjLOTk58vnAhISHXlATQtLQ0Lp5X/B+0C2j/2tr9stSsVQuJRJeoyBeTGP37D8C7vnLfpDTTiiMiVbFowQJiYiov4mmQoREI3iLE2tp8vnAhyclPWPr9D2+6OS+Fn58fzZr7vbUD4+piZ2/PwEEDORUYSFhoWIXbm5mZ4ejohJb47Xyp6+np4ejohIGhwZtuCv3698Pc3JxfVpWvavft24/WbdswdcoU7O3tmTN3Ltu3bePSxYo7o9ra2uTl5b2WKILyBvclZBTPCBgYvlDD165Zg4WFBalPn6rdz9nJiUlTpmBmbs6Kn5Zz5a8rAHzz9dd8MHMmi5csYeWKn4iMiCz3+9u3l3UMvLy9lAZDfs39VO4TEhKiIBDM/mgOnp6K+5b4DMTHxfP5PNWzlm8CCwsLps+YgYODA4cPHpIP1n5esZKJkycxb/58vvvuO5KfKIcV/xu4fPlPzMzNCHpFofBvmi5duzJg4EBGjxhRqagZHV1dhdlmW1tb+bUaHhHOV4uX0KZtO4RCIT6+vvj4+iod49dVqxSu/779+5Oeni5PB+jRsyfnz51jx47tCvvVreeBnkSitLw077zzDqGhoQrL+vTti6GRIRvXb1DavnOXd4iLiyv3mFevXCH81i0Fn40uXboiEAlUig7lUVRUxK+rfuHhw0ck3r0rFxXLUrNWLQwMDeXrr127irXV2+/zYWllhY2NNWZmMpHU0dEJkZaI2JhY5n36KSYmJvTt24+NGzdQUFAg369p06bYO9Rg3569Cu+ShYsWKeRDOzg4ENChA8lPnnCoiudeFbq6Oi99jL8DHR0dHB2duB17+0035R+Nrq56gSA1JaVSx0hJTVV6Xvj5N1f4bGVlSdu26kUDZzdZeoG6+x8gJSWZkydPVqpNqrCwtKRXr174t2jB8+cZTJ0yWWH97bg4pagDY2NjsrOz1VaAaNmqFePGj692m/7raASCtwj/Fi0xNDRk+7Ztb7opL42JqelbPTCuLhKJLo6OThgblR+eWEJsbCzCE0IyM159qOW/DVs7O6ytbV7rd4hEIrS0tN5YSaGM57LQVH0DRfOvr7/6SuX2QqGQXn160717D7KysliyeDEx0dHy9dFRUcz79BOmz5jJ3I8/4eqVq2zdukXlgNfd3R1PLy8OHTzIzlJ52F7eXsyaPYevliwhPLziNIGd23dwxEgxLNnU1Iyx48Zx8+aNCvf/u2jZqhVDhw1DIpGw5/fd7Nu3FxcXF6SFUoKDg8jOyeL9qdOYv2ABW7dueS2znW+arKwsft+16003441TNqR80uTJGJkYIxaL6dS5M9fDrvPdt98o7GNsbMzyFSsUnhWenl408/Nj2f/+R15eHh4eHvTr3x9ra2tW/1o5/4LSdO3enbgyecI+Pj7cuPFy91HZVAkrG2tE1Yx6KZBK2btnD1oiEV27dyt329Lrs7OzX8pToSJmzJyJj4+PynW2trZs2rxZaXlwcDA//fij/HNzv+b07d9P/nngYNkA6LNPP+VOQgIZGRm4e7gzdPgwNqxbD4CWlhYDBw0iPj5BSWg+d+48t27eAgF07daN+Lh4wm/dIiMzQ77N8hXVi+SReen8MyKXNLwaBAJZlYPX3V+xsLCs8N4Gyt3mdlwcJ0+elEczlKbkXlQV+aOnp08Ne3u++/57srKyePjgIXr6lfNDGj5iJA18GvDhBx/wtJwJFg3VQyMQvCVoaWnRqk1rHty/z82X7BxoeHuIjIwgMvLfHbr/T+N1CAS1atdm0RdfKC3v3aePSnfyoUOHMXToMKXlC+bN43ZcHAANGzWif//+2Nnbk5eXx86dO9DV1VEZBr1z50569u5Fo8aN8PH14drVaxw/flQepicQCBg6fDh5eXn8cfjwS/3WhIQEpWUffPghz9PTOXTwoNK6WrVqE2kVyZOkpJf63spia2vLu0OH4V3fm/z8fIUZ4OEjR5Kbm8PiRV9w4/oNlixezPQZM5gwYSLt2rZj44YNJCYmVup7RCIR9jVqIBIJuZNwR21UirOzM3r6emRnZRMXF1du9IqJiQmWlpZoi8XExsQoXadisZhatWpx7969KvsnlEVbWxsrKysMjQxJSnqiMCNlYGCAvX0NHjx8oDZ1RSKRULt2bbS0tcjKzCIhIQFpBbP6JZU88vPzefTwYblpAhWdi1dB79590NfXZ+/u35XWaYvFAOTkylKSHBwcmPL+FB4+eICeREK37t1p2aoVz549Y9vWrVX+brFYjKGhISmpL867tZUVNWvVYtmyqhlu6UkkDBoyhH1795L6CiL2Ajp0xNzMTP750qWLJCYmqg3t792nDz169mTUiBEv/d1V4eGDB2ze/JvCsjFjxpKSksK+fXsVlo8YOVpp//3797F//z5s7ez4+ptvlETSgoICNmzYwJAh78oHav0HDACBgLVr1ygd7+SJ44Dsnu/arRtJT5Lks7ft2slmaKvrQaClpemu/xdR1V/R1tZm7fr1StuWjpYKDbmGVFq5SMnw8HCV9/a48eNp4ONDUFAQAQEBHD50iHbt2zN96lS175/Up0/lkUqNmjYB4EpQsMI2QgS0at2a7t27Y21jQ3Z2Nju2bePkyZP069+fps2aVdhmGxsbGjZqSHRUtEYceE38p584w4fLXmYHDuynV+9eODq7YGpiQtLjx/y4bBnp6ek4Ojr+n737jm7iyh44/pUlF7k33G3cKAaDG71DQm8BQg3ppEIC6WXzyyZAspuQvptKSNuUTQ+9ht6bwTTb2Ni4V3Dvsn5/yBaWJRsZMGyS+zmHc9BoNBqNpZk39913H5NvmUJgUCAWFhZUVVVRVFTE4UOH2Lxpk1EDp0fPnkycNAl3d3dKSko4eOAANjY2eHh68tEHH7S4LzG9e+Hm5sY333xttM2p06YRHRODo6Mj9fX1FJeUkJ2VxcpffiE7J8dgXRsbG2bPmUOnzp2xtrYmIyOd37ds5aabR7B923Zijx7Vr6tSqZg0eTLhPXrg4uJCeVkZSUlJ/Pe77wzGaQ8bNoyomGjWr13P+IkT8PT0RFNXx9HYWH79+Wd96p29vT33PaBL53F2dgHgzjvvprZOd3K7eOECX5hImzTXiJtuIiIygk8++piRo0YT1i0MLy8vKioq+PLzL4iPP4NKpeKRhY9yIu4EWzZv1r/WzdWVO+6+iwP7DrB3r66nsHH8/c7tO+jVuzcBHTtiZWVFcnIy337zDSVNxgnefc89OLs4Y2NtA8Dwm0YQ0ztG//zHH31MRZOCMo898bjBvpsaajFixE107tqF8vIyevToSV5eLt99+x233347rm5uHIuN5dtvvjF4TUhoKBMnTcbTowM2ajW5OTmsW7eeuOOGaaptZae244EHH2z1GIDupmDmrFkEhYTg4OBA8cWLxMXFsfK334xufLy9vBg9dixBwcE4OzujUChIT0vj22++ITMzE4Du3bvrpwrz8vLG0tLS4Ng1/zteC+3R0CouKjJK3x0/cQIJCfEkJSYZLB9x800UFl7geGys0XYahxncdc/djBhxE1qtli1btuDi7HzZqYDijsex7PXXmDlzFr379KagsEAfIFDb2pKVlcXmTZsoKytrdTttNXjIUCIjI/noow+prKzE1taWiopLDeBhw4cxeMhgln/ysX5oRHu5d948Bg8ZgoWFBelpafzrvffIaXaObCo1JYXnn3uOhx5+mIiICJa++ip79+zhk48/bvXmdfTo0dx9772oGoqF1dXVcfrUaT768AOD4ztqzGiDQFBtbS2nTp3ik48+MljvllumMHLUSINp6jLSM3j+uWcB3Y3jfQ88QGinTlhYWKDVaklOTubDDz5oNfDi3qEDb739tkF19KioKKbPnIWfn6/Burl5eaxZvYoRw28iKPhSmnRhYSEfvP++QebKzJmzGDdhnEGPpkajIe18Gv/56iujom22trbcd//99OjZE6uGG+/KykqOHDnC5ytWUFtba/axAAwKApoaZ28OpYUF/Qf0Z+eOHfqgXFONY72rqnQpvuHh4dja2WFrZ8d9DzygX+/tt966ot9UYFAQdXV1FBYUoFAoqKurY9jwERyLjW3zkJcOnp6EhYUxYOBA1q1dy4Z1pgsPmmvI0CEG4+9Tz6e2GjirqqxEqVRel97OpiqqqjgRZ9ihUl1TQ0lJifHyavNrz5gK7H762WcGjz9qUuDt/X//22AYT7fw7oBuKMLG9etJSrp0DbjSGgTmXLcmTpqEn78/H7aSpTBq9Gi6devOv//1HnV1dW26ptvY2PDQ/Ic5fuw46elp3DxyFEFBQdjY2BB3PO6ys4BMmjyJ4JBOrF6lK9zbqHt4d0aMuAkfHx/s7O3RaDQkJiay/OOP9e1LWzs7HnjwAY4fO463jw/dunXD2tqanJxsvv/uO9KbFWFtS7sZwM3NjVunT8fX1xcHR0eUSiUZ6el8tmIFBQUF+vWcnJyYPmMm/h0DdMerqIgjhw6zZo1xcNycNtDlmPq7a5vVPInuFYO3tzf79+2jsEAXcMzMzGh1GuLL8fDoQP+BA1m/dq3+nL1xwwbGjB3L2HHj+OXnn02+7uLFi/qgmH9AAID+sXuHDnh7eeEfEMC9QfM4FhtLaVkpCgsl69ev121AoaC+WaC5eXYX6ArFAnz7jXGmUKNdO3eaVUMRcaYAACAASURBVPxZmPaXDhD4+vuhUipZ8MgjaLVaTp88RWVlBR0DO+Lg4EBJSQnePj4oVUo2bdxIdnY2tra2dOvWnQkTJ+Lk5MiPP1xK3+zWvRvz7ruP/PwCfvz+e1SWKsaOG6+b07SVxqZCoWD4sOEUFhZyxEQDunPnzpw+dYrMjEwuFhURGBhAnz59efypp3h16VKD6NnD8+cTGBTErp07SU5KIrxHT2bPmY2dnR3HjhreRC587DH8/Pw4eOAACfHxeHl7M2z4cBY+9hivNZnKx83NnaCgYGbNnsXpM6fZvnUrvfv2ZfDgwZQUF+unaKurq9OPOQsKCcbFxYX09DR9JdayslKuRocOHgQFBTN7zhyCgoM5m5jIqZMncXRywsvbi/j4M1hYKAgKCiYn2/ACoLKyJCgomLMJlxqujePvnRydyMvP55effiI4JJjBQ4Yyd+7tfPD+v/Xrnj9/nsLCQpydnPH28SEvL4/MjEsXpLomDVxAfxyCg0MIDjE91MLN3Y1u3bqxb89etm3bytix41i4aBGbN27Cy8eLfv37s3fPHn2PbXh4D+574H7y8wvYvHkTJSWl9Onbl3n3zePzz1YQe9T4htNcw0cMv+wxUCgULHrsMTy9vNj6+xbS0tLoHt6D4SNG4OTkxBdNItpWlpY8NH8+1jY2HDp4gKSkJDq4e9C1W1fc3N31F8fikhL9sXJydMLC3t5g3KKpKacKCwtISdE16KuqqkhJOWcUyGiNxVXMMdySoqIio3F+4ydO4PSp00Zptt3Du1NZVaVfPyQ4mG7h4axds0bfIPvtl1+xVKlYs3oN2dnZbdqXE3EncO/QwaBHuKK8nPfNqIreVv4BAdx5150cO3aMvbv3EBUdzb3z5vHS3y/Nmfzfb7+le3g4jy5cxA/ff2+QZdCte3cGDhx41ftx6NBhjsUepSC/gIz0dNatX8e+PXvNKmRXUV7Om8uWMXDQIGbOmkVGRsZlX1dWXs5bb7zBuXPnsFWr6dylM/Puu5/7H3yQt954Q7/egX372bd3H6UlJdiq1YR27sT8BY8we85sln+yXL+el5cXdRoN9917r37Mqaubbly0nZ0dz73wAgArPl3BkSOH8fT04M677ua555/nheeea1OhRUdnJ/z8fHn3nXeIPXoUW1tb+vXrx8zZs7nnnnvZvHETX3/9HzIzM4mJiWHmrFnMmzePZ55+Wr8NFxcXiotLePaZZ6goL8fVzY3oqCjGT5zI088+wzNPPWVwXXrgoYfo3r07q1au5MC+fVjZ2NC/f3/GjR+Ppq6OFZ9e6o1t7Vg0enPZMn2BPjP+xCZp6ut57tlnUaC7blZWVvH5Zyv0z9va6ILBjcHyffv2UVlVRUF+AUqlBY8uWsTuXbsNgu5t0aVrF86fP49Wq8XSyhKNRkNWViZHj5o3fK2p86mpPPP004wZM5Yp06Zy+vQpKiuvvBjriw3ft8ZhFpdT3hAQtLOzu64BAlsbG6OMKmsrKxwdHY2XNwT3zbFn9y6DIJetrR39+vXj0OFDJjNq0s6nGTyOjtHdmFVUVnLb7bfzcpPzYdeuXU1mEFyO0sKCy33Va2tqCAsLw83NjcIWxqhHRUWDQtdea8s1HXRZU0FBwVRX1TBu/Hgy0jM4sG8/tZo6goNDWtwvhULBnNvm0Kt3H/7z5ZcGwQH/AH/m3Xc/RUVF7Nq1i+zsLPx8/eke3h2VSqUPECgtLAgKCsbT04vSsjJWrVqJrY0to0aPYtHjT7Dk5ZcNiom2pd3s4+PD408+iUajYfu2bWRmZuLu5k738O4EBgfpAwSOTk4897e/Abqbz6ysTLp0CWPk6JGobW0M7gfMbQNdjqmhQXUajb79YGlpqS866OjsxCdNgipXEyCYdusMFMCmjRuZOHESoGvnxB6NZdSoUWxYv96gI+ByJk2ezNRp07CwsODChQu89s9/kp2VxcPz5xtMc2ipUlLf5Ivu7++vL6xqymITmZvNNdabEW3zlw4QgK5IzY5t2/m5SYqhQqHA0lJ3aI7FxhoVP9uzezfPPPsckVHRBieE8eMnUF9fz/v/fo+LF3Qnn+zsbB5duIjyVnoYwnv0wMfXl19//sWgEE6jd95+26BHP+74MVJSUrnv/vvp2TOCHTu267YT3oPAoCD279vHjz/8AMDRo0dZ+NhjhIQYnrx79+lDx44d+e2XX9m69Xf98rTz57n/wQfp168/+/fvM3hNYmIiP/z3e/12X1qyhOiYGH2AoKqqilUrVwIwdtw4QoJD2Lr1d3Jzclv87FfCwdGBl1580aDXyeoqah3U1tby0QcfUF9fz5EjR7B3cCAqKho7Ozt9cGPr77pjFBIaQq8+vTl14iQ7W5lruvE4TJgwsdU5m7VaLT///CM1NbUMHDiI8vJyNm/ehIurC71798HH15fU1FQUCgW3TLmFkpISXv/Hq9Q0fPbDhw7x1NNPM3HS5KsKEJhzDCIiI/D28WHD+vX6NPVjscewtrYmKjqa9evWkZur+1sHh4bi7OLCjz/+wK4dlyK4mxtSMBtlpKeT0dAz1TGwIypLS/2xa8n27dvZvn07oOvdbHNBz3YIELRFXn6+we/xtjvuwMPDgy2bN+vT9oqKigxuINuqaQ+knZ0dH5oxndGzzz/f6vOPzJ9v0PPl6urKY48/jpWVFW5ubkRERPDAQw+RmJBAYUEBbm66FOXqmhrefecdxk+YwIyZM+jUuTMfffABlZWV+Pn7MXjIkCv8lJcUFBRwLPaoPmX4SuzZvZtDBw9efsWGdRuHglWUl1NQUIC3jw+TJ9+Cr6/vpQBYk+NVUVlJ3PE4Du4/QM/ICKNt1tTWGhSkagzwTJg4UdcwfeopchuyBVLOpfDF55+zeMkS+vTrx/ZtLU9B15JzycnU19dTVlbGli1b8PTyYsRNN/Hf/36nvw7t2rkTe3t7Zs+Zg7u7u0FPWnVNjT5r6kJhIVu2bOHIkSO8tmwZU6dN09/0BwYFERUVxYpPP2VHw+8WID0tjZqaGqZMncKvv/5qENBq6Vg0OnXypNlTewIseuwxo2Wnz5zW/96OHYvlwYceJulsIrt27QJ0BQ4Bg9/k9m3bUKvV/N/fXyIxIYEvvzBO8zVXakoK5aW642dna0dNdbX+va9EfX0969atZceO7ZSXl9PRRAX+q3XzyJHc0WwYQV5uLt80ZLo5uzhf13Rfbx8fk1Meurq5mVyemtr6vPA9evZkyJChfPzxRwbB6TFjxtIzIoJPly+/7NSNHTw8CA4KpCA/n1OnThEVHc3NI0fpe0XNnc6wOXPiYGcbghqhnUJNBghUKhVe3l7s36dr27Xlmt5U17CufPLRR5w8eVK/rKU2mEKh4K577iE8PJyPP/rIaLakqKgolEolH33wgf69Yo/Gsnr1KpPbs7a25rV//kPfxs7ITOeRRxcyZOgQVq289Bpz280Ak265BZVKxVvLlpHVJCC/bdtWXF1c9I8nTJiAWm3DG8uWkZ6Wrt/Xqqoqhg4bytbft+qPu7ltoMu53N998OBL188unbtw9z13X1XbASAqOpr+A/qzdvUaiouL0TbZi59/+pHIV1/lvvvvb9NMIWnnz/Pdt9/SvXs3HJycTU43C6CwsKCu9lKQsaSkxCg7c8iwoajVajau32D0+mEjhqMAg2lZc/JaziIULfvLBwg0Gg1r1xp++bRaLTU1uhuwxgiZSqXCw8MDRycnFAooLinG189XH+FUqVT4BwRw9myi/sQFkHQ2idLS0lZ7LW8eeTOlpaXs2W26cdB4knN0cqKDuztW1lZYNaQ/urhdOnkFh+puOg4cMKxYffToEaMAQUyvXmg0GqOb3JMNDa/Qzp2MAgT79u41OEbZWVkmpwFqb79v2WoQHAD0f68rcejQIYNUuoy0DKKionFxdW2XeZybqq2t1e97eXm5/vvWGFBSq3XzMHcMCMDD05MtmzfpgwONTsTFMW7CBFxcXQy+e21hzjHo3lC5vnlF9EMHDxIZGUl4jx76C7yFhe77HhDQ0aAX4K/G29ubqdOmERQcTHBwMAsefpjcnBx69+6FrVrNwMGDCQ0N5aMPPzAY06dQKJgxY+ZVvfe69esoLSmhtrbWuHp5QxGtpKQkEprMfNCvYSaD/XsNf/uAQWPL1dWVv73wAu7u7oBuCsYnnnqKHdu28+UXn5vsgV+7RpcN8fD8+Tz3/PP8/cUX2bRhI5s2bLyqz3ktXU3v545t25k8+RZievVqtXcovyAfZ2fzpjUD6BkRwZlTp/XBgUapKSmUlJQQ0JDGebXS0tKwtLTE09PTYP/PN2QweXl7GwQITLl48SKxR2Pp3aePPkAwYOAA6urqTM4ysn3bNqZOm0bfPn0upZi2g19+/pnk5Eu9luMnTjR4fu+evXTq3Jnb7riDuLg4iouL9YVEm14DbG1teebZ59Bq63nv3XexUCjw8PKivl5jMtupNU1T4F1dXSk2Y5pSc7TnNSs1JYW1q9fg4+dLVFQUa1evobS0lKIi3XXH2cm53d7bFFOVzVua5tBUL6RaraZrWBjDhg0DdOfEiooKevTsYXAOGzdhPMlJyS0G+/Py8sltSFsfPWo0Z06fwcHRkdraWtavX0dAgD+pKanApUDra8uWsXPnDoNzs52dnanNm+186nk0Gg2BgcEc2H8AP39/nn7mGTZt2MiaNasJCg5CqdTN0gBtu6Y3lZmRaRAcANNtMAsLJQ88+CBdunbl3bfeIvX8eeN1GoZphXbqZPK9mks5d86gnXM28SylpaV06dIVuBQgMLfdbGNjQ7du3Th54oRBcAB07dymQ0OjY2JISTmvDw40ijt+jBE3jaBz587sawi+XI82kKWlJbdMncKe3bu5eeRINmxYz/jxE4iPT2RXKx1YrXFwdOTeefPIzs7mp590HaAKLt2/ZGZmsnrlSqZMncrgwYPNDmo2ztbSqXPnVtezsrI2aN8XFxcbZGdGx8QwfuIEo2xEgNDQToyfOIHffvuVX34yPQRCmO8vHyAoLSlpdV50lUrFhEkTGTp0GMqGE1lTVlZW1NXV4eysuzA2jv9pqrCggA4dOpjcfkhoCEFBwaxft87oxq9RcEgI02fMwNfX1+i5xrFBcOninN+soVJoolHn0aEDWq2WRxY+avScAt14rOaaFz+qrq7Gxsb8tD1TunYNY9DgQQbLVq5c2eq42qzMjBafuxLFxUUGjxt7rmwbbs7bU9OLRp1Goz8xNl5srax1f19Pb111/4iISEJCQw224WCvmzLPo4OHwYVz+PARhIReCgzV1dW2WAPCnGPg7u6ORqMx6plobBh16OChX5aYkEh2VhZ9+/YlplcvsrOzOX3yJLFHjhhdhP8MPDw6EBPTC28fH3z9/PD18QF0mTrp6emkpqTw85EjKC0sOJuYiEJhwfiJExk7fjyxsbHs3bPXYHuNc5RfjZ27dlJaUkJNTY3R8IcOHh6MnzCBA/v36zOAAEI7675brU2jFhERwf0PPIC1jQ3r161j7LhxfPnF58y9/Q6CQ0OwtLKiroWU96NHjvDhBx9gYaEwK/3/j6SgoICamhp90AQgMDCQGbNm4uPtg5OzM0qlktKSkjZVI3d2csLf399kVXbQ1X65FkpLG4OShunOpQ3BSjtb8ypL5+Rk09+2Pw6OjpSWlNDBvQNFRUVGQV3Q9crX1NTg4dW+U+KlpaUZ3JAPHjwEx2ZBmm+//oaoyCjuuvtu3n3nHezt7dFqtVQ2BG29vLy4/8EHCQoOIj8vj2VvvoljQ52E9evWXVGhwkaenp5kmJlyfDk3jxxJ/BnD2UiUFhZo6s3PuGhJUlISSUlJDB4yhKioKP15ojHg1Thd4B/F3DvuYPDgwfphAys+/ZSdO3bwzPPP0S2sm8G6zs7OdA3ranI7jX9/Jycnho0YzqfLlzNxki4te/PGTbr6EsOGG7zm8MFDDOg/QB8gGDHiJubMvY1Xli4h5VzrmQ4tqaur4/z5NAICOwIQ3r07FRUVdO3WVRcgCNIFOBITdbVp2nJNN3g+17xr+NBhQ/XtZh8/P5MBgn179jJgwEBmzprF5FtuIT0tjZOnTnLowEGTtT1MTZtdWFCAW5PzLpjfbvZpuFZnZ7X+mZycdLOeeHh0MKoxZanSBR7c3S+1869HG2ji5ElotVr27dvHzSNHcvrUKezt7LnzrjvJzW17r7lKpeLJJ5/C1s6Od956q8Wisyt/+41evXpxz7x55OXn6WsdtcTTw4PR48bxzX/+0+p6oBtq2lKQ097enrvuvpuMjEyTxZan3TqNyspKNqy9uhosQucvHyCobCU4ADB9xgx69e7Nxg0bOH36FBnpGdTX1zN7zhz6Dxigqy8AV9zYHTlqNJWVlS1G+zw9PXnwoYfIzMjg0+Wfci45ibKyMhwcHHjlH/9AQZOGZlv2QaGgqqrK5Dy1yUnJFF0sMlreHnPH26itDRrUgD7K25Irqd5tQcsZHJp2+FzXTOOftOF7lpWd3WLwpKjZTb6Do73BsW2td9SsY6Aw/T2vb1jW9AjX1dWx7PXXiYiIoEvXrnQPD2f0mDEMGz6cd95+26B+w41ibW3N7NvmEB0dg0ql4sjhw3z37bdtGlfXyNfXj9m33QboAmnHYo8xYNBAVq1caVSD4MyZM9TX1zNx0iSSkpJM1gao02iMqgpbWVnx0uIluLm5snTJEtLTLo17dXZ2oqjI/DoMnRrmNb5cym1ztmo1jyxcSF1tLctefw21jZqx48ZxsfACS5csYcnSpTzx5JMsXbKkxW0cOdy+hQpvpKZB5MDAQF58+WXiz5zms89WkJR4Fg8vL6ZOnUJkVLT5G1UoSIhPYNUq00MnTJ2rr0jDmH6jZLfG37yZQ3PqGypn39iBPG1XW1vLypW/MXLkaNRqNU6OzlRWVurPeXa2toSGhlJTU0NOTi7J55JIO3+enJycy95ctMbZ2Qn3Dh1wc3dn4KBB7GmYceNKqJRKpk6bxo7tbuxv0ivs5ORIWpp5s3NciaKiYl1wzMO91fXGjhtHRGQk27dtZ/++va2u25LwHj1QKMDBwaHNNQicHJ30y0/EnWDr5s3s3L6DktISXnv9dfLz8tBqtfrxyv7+/ixeupSff/rJoLfyvgceIDIykkcXLDC4iZo95zbKyso4fPCgPkDQ0nV33/69TJw8ib79+tGvXz+iY2JYs2q1PtPgSqWmnmPo0GGoVCq6hIWxceMGJk2ajK2dHUFBgRQWFlLaMN1uW67pTVVWVrfwjE7jNisqKvjPl18x9/a5TLv1Vs6ePWvUfsnNzWXp4sX0jIiga9cwwrqF0alzZwYMHMSyf/zDqOOseQE7gPp6rcFNf5vazWae1xrb+hcvXDDZbo6Pj+dcyqXl7d0G8vTyYsLESfz322+ob9LR9NWXX+Li4qoP8phLpVTy6KJFBAUHsfyT5QaFNZurr6/njWXLWPLKKzz2xJO8snixURFTS0tLbO3U+Pr68fqbb1KQn8+aVopxDh4ylPvuvw+NRtNiIKFv337Y2tpib2/PwwsWsHfPHo7FxuraU5Mn0T08nOWffNKmmjyiZX/5AMHl9Ordm7jjcWxolv7o3GRcEuhuCjQajcme9+aRzUY+3t6EhYWxc/sOyspMR8wiIiOxsbHh6//8xyDK2/ymGtCP/evQoQMlTdIVTb1/QX4Bfv5+lx3vfcXMbB0eiz3Gsdirq8DfVF2d7uKhbFb5tWlF7Ktygzo983J0F9Xks2f14+8vZ9XKVQZj8q7WxcKLhASHYGtnZzBjg2vDjBXNeyHq6uo4cuQIRxrmH4+JieHOu+9m7NhxpqsdX+djO33GTLw8vXj7rTepralh1uzbmD1njkHBNHPFx8fz6fLlnDx5Uj9eesAg08X3QkJCqKurw0KpZNlrr+kbkH7+fmSkm240KBQK5i9YgK+vN2+/+ZZBcGDqtGmMGz+ed99526hyd0v69O6jCxCebbkRYEpFZSXffv01J06eJD8vj8jISP1zBfn5fPzRxwQFBbZpm38Wfn6+KJVKfc2V/gMGoADeeuNNfaZQakoKiYln2xQgKLp4ETs7O7P/tjeal7cX1dXV+puQ/IJ8IqKisFWrjRpuHh4dsLKyIs/MOjWNN2MWSiW0oQaBuXbu2MnuXbupra3F2dXFoCBd8rlzPLpgfpsCcebo128AAIcPHea++++nqqrqioNo/QcMwN7eXn+NSDqrG5MeGBTMnt26IR7nz5+nqOjqp0JsLi8vHx8fXW+tSqWivr7eqFNh8mTdjWrnLl04euTwFQ3pefKpp7BoUritLTUIPDw68FRX3fI75s7Vz17h3dCL3JSVlRUPzV/AxQsXDLKs/P39GTBgAL9v3mLUwxreI5wff/jBrPoYGekZxB2PY/6CBaScS2HxSy+RnGx849lWSYlJjBhxE126dqFjYCCfr1hBVGQUPXr2wN8/gMSES7ORtPWabq7Gm+kD+/cTH3+Gzz//nAWPPMK8efNY9vrrRun2JSUl7N61i927dqFSqZgydQqDhwxl4KDBbNu21WBdR0cHo/dzdHSgqEm9l7a0mxtnufG8TBZTUVERdRoNFy5eNLvd3OY2kJnUajWPLlxIfm4uWzZvIajJUN/a2lqTVf9b4+zszKOLFhEaGsqWLVvw8fZm5sxZ+udDGjoUmi4D+H3LFsaNH8/fX36ZL7/4gl07d2JlZcWMGTMZOnxYw0xqmaxds5q9e/a02pF65NBBXfAlOanFTsDff9/C7t27GDp8GIMGDWbRY49RUV5OXNxx+vbrz5YtW2TWgmvI/DzHvyCVSoVKpUKhMPxSOzs7E9oszVur1XL27FmCQ0IMpqwJ7RSKg4PxCQ1g1JgxaDQatm773eTzoKueC5dOuI169eljtO6pU7oxYX369DVYHh0dY7RuXNxx7O3tCQ8Pb/G9r0ZJsa5h5XitbszNVF9fT3lZGW7uhoGabt2vzedsHCdq73Bt0nrNlZ6eRlFRMb379TP6LlwviQm6NLLICMMCaz169gTg5KmTRq9p6siRI5SVlWFnZzpVubS0FFtbW4PGX3uKiIrk66//Q8q5FDIyMvni88/o1bv3FW2rsrKSnTt2GBVTa27CxIk889yzaDQaVEolAR0vjR+/8667mNBsbHSju++5h6joaL78/Av9WL5GGzds4MKFCyx67HEiIoyL3zXn5+dLVHQ0hw4eNKshq1QqWbxkCZ4eunTTrVu3tpjFciz2qFHGxF+BQqFgzty51Gk07Nmj6wFWNIxBvdrMq7jjcfj5+xmlPP8vcnR0JCo6muPHj+sbg/v27kWlVDJx8mSj9adOm059fT2HDh0ya/uNgXRTgfhrQdNkmJe7m7t+2tFGzYMDVlZWREZGEhra6YreT6FQMHLUSEpKSvj3v95j69atLHjkEbN+x6aMHT+BpKSz5ObkkJuTw9IlSwgLC8Pe3p6iomJCQkL45KOPOHzosD5LqaViYc317deP25plNQV07Mj0GTMID+9BWmoq/n5+gK6T4rm/Pa+fBrTR6dO6AnUqpRIX5yurV3DXHXdwx9y5xMbGknzuHHfMnWvWv/T0dA4ePKh/fDkuzs6kp6Xh6OTEvz/4gEWPPcaQoUN54sknKS0t5bfffjV6TeMQBXM1jvGOT4i/bHDA3Kt+cnISWm09N48cRX5uLiUlJZw5c4Z+/frj6OREUpNaHFd7TTdXclISmzdtwtvHR59Z0ZK6ujr27NKdQ9W2xsM8g0NCDab+c3Nzw71DB4Ngd1vazRXl5SQnJ9M9PNxkDYimGa1xx47RpUsXs6alNOVybSBTTP3de0ZE4Ofnx/JPll/2+uLm6srMmbMM/qmbDZ8d1FAL6ccff2D1ypWMnzjB4F9IsG5oSvPlDg4OLF2yhLKyMqZMmYK9vT2enp6MGjOaoosXef9f/+L5Z59hz+7dLQYHVq9axddffklFZSUnT56gsrKSW6ZM4aabbja5fnV1NZs2bOTFF15g8UsvgUJBv/66bO6oqCjGT5xg9PnElZEMglbU1dWRlZlFz8go+vQ9zcmTp/D19WHGjJlUVVXp50lu9PvmzXTp0oVHFi7i982bsLS0Yuy4sSZTll1cXYiIjOTw4cOtFpY7d+4cI24awa3Tp7NmzRpKiosZOGgQEQ0n8ObrJicn03/gAOo0dSQlJdGzZwTe3t5G6+7ft4+Ro0Zx97338tuvvxB7NJby8nJCQkOIio4hMz2dvXuvLAUQICnpLFptPTNmzWbnju0U5OdTXV3DuWsQIb+chIQEomNiGH7TCAry8ukaFkZEk57Oq3GhsJCLFy8yZOhQQFfAq15bz9nEs/qouJ+/Pw4NAYTG6blCgkNwddVVdc/MzGrTtHyg+y5u3LCembNmMX/BAtasXk1GRgY2Njb0jIigZ0QEH33wwTX5jC05fPgQk6fcwoRJk7hYdJGUcylEREUycPAgUlLOkdVkDG2fvn3w9w8gNjaWc8nJ+Pj6MmBAf+zt7dkZH29y+2fPniU6JoYHH3qYA/v3U1FZTtGFiybnLL4ajRdUhVZrNI97e4VePDw9uXfePMLCwjh48ABff/UV/1z2BlOmTOWfDcWzHB2d9HOuN3XLLVMYNnw4v/7yC1u3Gvam6GZcseSLzz7j0YULeXTRIha//LK+sFxzKpWKefc/QGVVFT/9+KPJdZpzcHAgMCgIL29vo0J5f1U+vj6EhnYiMzODoKAgRo4aRXh4D775+mv9VFvxZ+IZM2YsDy+Yz9bNv5OYmEC9VotLs+yzy1m1aiV9+/VlwcJH+e7b7zhz+hQFBQX4+fni6OTM6VOn2uMjXpanhwd33HknP/3wAxWVlXh6ePDookUoFAr9LDqgm3FhS0NPU3VVNfv27sHKxobBQwYzYOAAfvrxx8sWP2wUH38Grbae6TNmsvyTj6mtqaF7eDipqalGc8s31dIsBi1RKBQEdAzgWOylmWGUSiWB59V7kAAAIABJREFUHTvSpWsY4T3D8fH2wdnFBQsLC9auXWMwLZ65xo0fTwcPD1atXIlWq+Xrr76ic6dO+Pn5c/z48TZtKyo6Gj8/Xz5bcWmqRrVazV333MO6tWtxcnLinnvv4Y1lb5i9r431LSbfMgVvby+2/v47N998M4MaZh5Z+sorZGZmEh9/hviEeAYMGoirmxseHropiZsPXXvv3XeIjolh4aKF5OW3rajj9Zabl8eHH7yPlZUVgwYN4papU4mO0XW2bNqw0eQ46bZMealQKCgvL2fHtu2MHTuWvJxcfv99S4vra+rrzQqeV1ZWkpOdQ0hICNsarhdnTp9m7LhxAAZTDLblmn611q9bR1hYGMOGD+fM6TPEx+uCRVOmTqWstIzYY7FcKCwkNCSUUWNGA7rMvOasrKy4/c47+fmnn3Cwt2fW7NkABgW329JuBti0cQMPPTyfBY88ys8//0hqSio2NtZMmnwLiWcTOXxQF8Bcv34dUdHRPPbEE/z0w48kJiag1Wrp1r0bvXv3ZfWqlfpCi1fSBjLF1PDPgwcOYGVlZdbv2NnZ+bL1jNas1g1tOXlSl6XWPIA2d+7tjBozusXA2osvvICTszNlZWWUlZXx7jvvcPTIEZNBAW19vcGQ3+ZDEwCCgoPR1te3+HtQKZUMHDKYqVOnYWtry7at2ygpLWb48BHMnDmLyZNvYfv2bfzw3+//sgWyrwUJEFzG9999ywMPPcTc2+8AdGOqdu7YgYODIwObpRAnJCToi9NMn6mrQL5t61YCgwJxa1a8Z9SoMbrnt7R8QQBdddQD+/fTt18/unXvjlar5XxqKp+tWMGixx83Wv/jDz9k9pw5REVHM3jIELKzslj120pmzZlNdc2lm4+6ujr+9e673Hb77UyfMZPpDRXTNRoN58+ncfRI2+dibio3J5evvviS4TeNYNKkyVhbW5Obl8crixdf1XbNsXrVKgICApgyZSoA6RkZ/PD9D8y7b95Vb7u+vp4Vny5nwoRJDB48mDFjxwLwt+ee06fUTpg4kW7dDHv7Zs2Zo///N998zYF9hlWDzbF3zx6sra0YO248jz/5pH55SXExx45d+RSH5qqrq+PjDz/k7nvv5aGH5+uXp6Wd56tmxQ9ramoYOHgwQxuqQze+/sD+fWzcaLpq/d49e/Q9kLfdPheVSsW+vXuvqvhXS58DdMNbbps7l/9++y0ajYaZs2ZzNPbaHMfGHgZtvZaAjh15afFiamtq+OKzz/Q3+WvXrGb69BnMmDmTzZs24enpSVrapSJOarVaX0SrpKQENzc3Hnv8cRwcHXCwd8DewQFbW1ujXpIFjz7K35591ih9t0vXLtx221wCOnbknbffMTkdWW1tLW6ubigUl4oIBjYUtrrWgZo/orraGuo0GsaPn8D48ZcaXenp6bz7zjsGqeFHjxxh9apVjBs/3iCrq76+nuw2FKmqrKzkpb//ndvm3s7d99xt0HuWlZnJs88Yp1JfDxUVFYT36MHNI0fqlyUmJLB0yRKj8a9fffEFhQUFjBs3jqm3TgN0Q+I+/2yFwXRUl5Ofl8c333zDlClT+Xj5cv1+vPH6slYDBM1nMYBLRRkbdevWjaysTIqKiomKjsbZ2ZmEhEvFt0aPHsOsObP1+56amsrBQwdJTEhApbJqsYhk8+XffP01GzdswMPTkylTp1JSUsK6dbqiWvX19fzj1VdN3nyqlEoUCkWLPYaTJk/WpWo3pNg6OzvzxJNPUlZays8//YRWq8XSypKnnnmaxS+9ZDBThYWFhcnG/MRJuqyPivIy/u9vzzNy1Bhuu/12UlJS+O3XXzmwf79+Ow6Ojtx5111MmzaVkpJSMjIyjLapUCgYPWYMJ+JOXvcipbpgsOn3VFoY15JSKBT4+PgQ1q0bQ4cOxcnJke3btpFfkM/IkaOIjolm/fr1bN+2zWQBzuYaCzoPGjSYzl060yUsjDOnTvGv997Dw8uTO+66Ey9vL375+WeTKdZ1dXUG4+xbk5qairePD6caZhpITU2lvLwchUJhMFNAW67pV0ur1bLi0095/oUXuOOuO1ny8sv6zzlh0kQmTr6UWVBaWsovP/1kskPp+LFYAgODWPrKK4CuzfrD998bBDPa2m4+c/oMX37+BdOm38qjCxfpl2dlZrJr16WU9dycXD784ANmzZ7N/Q8+oF9eXV1NUnKSwe/2StpAppi6wdVqtWan0pua6eOd994zWq8xOHAlSkpKDIY1tzZEqrSsjG7du9O5SxeSzp41Op/Z2tkRGhpqMKwHaKgr0pNu3cLoGRGJk5MTGRmZfPv11xw4cACAlb/8yoRJk5g4aRJjxowlOiqazz5d0WowWLRM4e3j+z9TStrFxaXd59B1voK0NgcHB/z8/SgvLyc9Lb3NF7b/e/FFSspKefettwGwt7dj8dJXOH3qtNnjkIKDg7GysiYtPc1grJg5+vTtw9zb7+Df771HYmKi0fP29vZ06NABaxsbUlNSWp3V4Y/CwsICbx8fysvK9D16fxYqlQo3NzecnZ2pqKwwmnKnvSkUCtzc3XFycuTChQutZsBYWVni4+NLbW0tubm5/xPR3IqKCmpqahqKFN5GdHQ0KpWK2KNH+Oabb9v8+2pkZWXFv95/3yC97Y1lrxN3PI5JkydzYN8+gx54CwsLnnz6KcIbppoqKipm4SML9OcXlVLJvz74AFtbWzIzs6iqqqCmtpYLhRcoLCyguKiECxcKKS8rp6yslJLSUlxcXPj7Sy+xY8cOvvriC0DXcH/2uefw9/envLycT5cvb/ECPmvObMaNG2+0PDcnh6eaBKUaRUZG8viTT/Lyiy/qx/M26tK1C3974f9Y/skn/zPjAhcvXUp1dRWvLFl6Vdvx9vHB3d0Njaae7KysVq9bDo6OBDZUFa+pqeFc8jmzbihMUSmVePv64uzsRHZ2DgU3qBf2wQcfIqRzJ556/HGCgoNQq20533AT0hqFQoF/QACaujqysrKu+CaxcTv29vacTUxs8Xja2dnx4ccf887bb1826P3EU08ZpPZnZ2Xx4v/9n35GF19fX7p2DSM5OYnUZhk63j4+DGkyH3lrjh49wtmzZ3nq6WcI79GdN994g7jjcSbXnXv7HYwaPcpgWdMgYyOlUsm9991HZmYGa1evwdramjfffptjsbF88fnnBufdh+fPp06j4ZOPPmLJ0qV0bBi/HHv0KG+/9ZbBdh95dCGlZSV8+fkXaLVanJ2dqKys0h+T5uYvWEDffv0AXaC+aSaJq6srd91zD+Hh4byyZMlVj7d/7IkncHRyMrr5aWTqfLxy5W/8/ONP+scqlYr3P/wQtVpNTU0NTz7+OEVFRSxctIjIqChAN41eXNxx9u7dqx9CplIqGT1mLLfOmE5cXBxvv/mm0fu/8o9/kBAfz1dffgnAiy+9pC9ymZiQwOnTp9m9axdFRUXYqtXMvfNOBg0axMkTJ3j9tddMfh5bM2cSaau2XNPbi52dHV5eXhQXF1NYWGh0bmgsMLhr505+/eUXgoODqa2tIzs7q8U2a1vbzQqFAjc3N1xdXCmvKG9xulqFQoGriwsubrqs0NSU1BbbNlfbBmpsr7QmJDiYvy9ezLLXXzOoVTN9xgwcHRxZscKwrtKEiRPJzc3h0EHzhnZdLoOgLTp36cJzzz9vcla4Rnl5+Sxd/LK+7X7PvfcybLhuFpDSkhIOHz7Crp07W8ygcHZ2ZtqttzJ4yBAqKip4+sknTc6IIVrm4uIiAYJrSaVSoVarL1WIRVct/JFHF7Jl8yZ9wbhevXszaPAQfvn5R9LOp7W0uSvi4uJCUVGR/uSqUChY8OgjBAR05MUXXriiGQCE+LPQaDSUl5e3y4wcAMNHDMejg67Y0dmks5e9KVEoFPTt1x83d1f27t5jdP7z9fVtsZHSkjFjxpKSes5g6qHxEydgYWHBtt+3tnqhVCmV9IiIIDgkGJWFrqe6rKyM3bt3meyhDQ3txJy5c1j+yXKjscz+AQHcfc/drPz1tzanS4v/bU0DBH8WHp6eeHrq6mxUVVWTdPZsu/ZyhwQH4+Pn3+p85R6engxvMj1eQmIix2LNS2N3dnY2GRxXq9XU1ugyYRoDXZUVlSarliuVyhanOjP9nk4sevwJ7O3sePXVVw1qskyefAtDhg7hk48/ISHB/BTrlgwaNAgbGxu2tJKFOXjIUHwahljm5OWwc/sOo79pSEgIaltbkpMuFUcLCwvDxsaG06dPtxgMAV3BwsrKSpNDZO5/8EHSUs+zYYOuwHVkZCRKlYq448dbDGiFBAdzsbi4xVo2Tk6ObZoi9c+kaYCgaeDpz0yrrae4uOSy63n7+HDf/ffxzVf/MQrUXwujRo+iX//+LH7p5WuyPVc3N3x9jQuDAhQXl5CelmbwO/X08mLS5Mkc3L+fuLg4s8/Lfv5+qJQqo4CuuDwJEFxjdnZ2LH31VQ4dPEhZeTlenp6EdetGeXk5ryxZcl1uzmfOmk1wSDCnTp7ERm1D585d8fDowJpVq9m0yfy0JiH+jKqqqv4UGTJC3Eh/xgCBaF+2trZUVVW1W3D2r8Da2vovW4DtrxggqKysbDVAJUR7cXFxQTV50kRyc/PIysoizUSxCGG+2toaEhIS8PPzw9PLi7zcPH7fvIUtWzZft5uSc+eS8fbxITomBltbW5KTzvLTjz8Qf+bMdXl/If5X1dTUyMVWiGvg6NGjpKVd2+w38edmqlizaJvq6mpUKpVRgey/Ak19PSkp5ygsuLKpF/9oamtrpb0ibijFG2+8oc8gqKqq4sSJExw+cvSGnMz/6BkEQoj/TbW1tVRXVZk1rZ8QQgjxv0ihUGBvb9/qGG7xx6bRaCgrK7vuhTyFaOTi4mI4i4GNjQ29e/emd+/exB47xt69+yQdVwjxh1ZTU0NNdbUEB4QQQvyhabVaysrKsLW1/UtmEvzZ1dbWUlFRIcEBccO1WO0kKjKSO++4HQ8Pj+u5P0IIcU1oNBqqqqqorKyU4IAQQog/Ba1WS3l5OZWVlWi1UtPhz0CrraeyspLy8nIJDoj/Ca2WQ7W3t2f6rdOwt7e/XvsjhBBXrL6+npqaGioqKigvL6eqqkoutkIIIf50qqurKS4u0U+FV19fj+JG75QwiwLD9kpxcYnUHBD/U1SXW8HGxoYpt0zmP19/cz32p92ZmvZHCCGEEEKIP5qamhpqampu9G4IIf5EzJpQ1cPDg2HDhrb3vgghhBBCCCGEEOIGMStAABATHS0zAAghhBBCCCGEEH9SZgcIAPr169te+yGEEEIIIYQQQogbqE0Bgu7dumFjY9Ne+3LD3DZ3LtNnzDD53PxHHmHK1KnXeY+EEEIIIYQQQojrq00BAoCQkJD22I/rytbWloiICP1jXz9ffHx8TK4bGBSEt7f39do1IYQQQgghhBDihrDYu3ef/kF1dTWrVq9m+/YdLb7At4Ub6T+S8PAePLpoEbNvuw2FQiaFEUIIIYQQQgghVPv27yc8vDuOjo4cjY3l7NkkAAIDAwkM7Gj0Ai8vz+u9j9fcwYMHyMvP44knnyQrK1O//Nbp05k0ebLR+p4eHvTr399g2StLl5AQn9Du+yqEEEIIIYQQQlwPKoCi4mIcHR0pKSnVP1FUdBEwDhD8WWYySE1J4YXnn6e4uJi+fXXFF3ft2EFCQrzBevfd/wBZ2VmsXb3aYHl6Wvp121chhBBCCCGEEKK9qQAcHRwAsLG21j+hVqtNvsDS0hJrKyuqa2quw+61r+LiYqJjYugY0JHExERy8/LIzcszWKe6poaSomJOxJ24QXsphBBCCCGEEEK0P1WXzp31WQGdOnci7sQJvLy86NKlS4svUtuq/9ABAqVSyZChQ7jpppvxDwigqqrqRu+SEEIIIYQQQghxQ6kmTBivf+Dj7c0jC+ab8bI/bmE/d3d3/v7yyzg6OnDwwEHef/99br/jdv3zXcO6YmlpqX9sbWWFo7MTPXr20C+rqa41GooghBBCCCGEEEL8kalu9A5cb4WFhRw6dIiNGzaQm5Nj9PzD8x/B2dnJYJmzszPdwrrpHxfk5/P4Y4+1+74KIYQQQgghhBDXy18uQKDVavnqiy8MlmVmZFLTMGTilaVLmHzLLaz67TdyTAQQhBBCCCGEEEKIP6MrChD82cbsf/P11/r/19fX069/f6ytrPnXe+/ewL0SQgghhBBCCCGunzYHCKqrq//wAYKvmgQEWtK7T+9W11u7eg3ff//fa7lbQgghhBBCCCHEDdPmAMHFi0XtsR/X1bLXXzNaZmVlzdRp0/Dx8WHTxo2MHTeO3377lbOJiQbrdOnahRNxceTmXpoO8eaRo7jp5ptwd3cnPj6e77/7loyMzOvyWYQQQgghhBBCiGuhzQGCoqKL7bEf19WJuBP6/9uq1Qy/+SZG3jySisoqlr3+OqdPncLewYEhg4ewacNGysrKGDxkKLdOvxU7Ozt+37SZvDxdgCA6JoZx48by1ZdfkpqayujRY3jk0YU8+8wzaLXaG/URhRBCCCGEEEKINmlzgKBpz/kfkUKhwM3NDW8fb/oPGEifPn2wsrJi69bf+eKzz/Xrffv117y8ZClPPPUURRcvEhEZyb69e/npxx+5ePFSkCSmVy82bNzAsWPHAPj++//St19f/Pz8SE9Pv+6fTwghhBBCCCGEuBIWbX3B+fPn22M/rhsbGxuefe45Fi56DLWNDe+89TaVlZWUlZYZrFdeXs7SxS9hbW1DTK9erF+7js9XrDAIDgCg1aJUWDRbpEVh0eZDK4QQQgghhBBC3DBtyiAoKysjv6CgvfbluqisrOSf//wnFy9cQKPRYGlpiaWVFUqVki5duxAa2onAoCAC/P35bMVnLF2ymHnz7mPi5En07d+fzZs2cuTwYQoajsPhw4e5/Y47yMrK4vz584wePQatVkuGZA8IIYQQQgghhPgDUY4aNeolc1c+cuQo6RkZ7bYzarX6usyQUFFRoa8P8NTTT+Pl5UXnzp0ZMmQo/v7+VFZWcPToUY7FxlJRUcGBA/vJyEgnrGtXBgwcyOgxY7hw8QLnU1PJyc5Gq61n0qTJTJ4yhdraOj79dDnFxcXt/jmEEEIIIYQQQohrQa1Wm59BUFdXx7Hjx9tzf26IkydPUlVVxYEDBzh96hQlJSUm1zt08BCHDh7Cw6MDkVFR7Nm5S//cpo2b2LRx0/XaZSGEEEIIIYQQ4pozO0BwPC6OioqK9tyXG2LtmjVtWj8vL1+CAUIIIYQQQggh/nTMrqR36NDh9twPIYQQQgghhBBC3EBmBQgOHjxEeXl5e++LEEIIIYQQQgghbpDLBgjy8vLYtXv39dgXIYQQQgghhBBC3CCt1iDIzMzkl19/u177IoQQQgghhBBCiBukxQDBuZQU1qxZS21t7fXcHyGEEEIIIYQQQtwARgGCouJi9u7dy5kz8Tdif4QQQgghhBBCCHEDqH79bSWenh7U1WnISE8nOyfnRu+TEEIIIYQQQgghrjPVuXPnOHfu3I3eDyGEEEIIIYQQQtxAZk1zKIQQQgghhBBCiD83CRAIIYQQQgghhBBCAgRCCCGEEEIIIYSQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAIEQQgghhBBCCCGQAMFViYmJISQ4WP9YpVLRtWsX7Ozsrut+eHp6MnjQIBQKRbu+j62tLZGREQwaNJARI4YzdOhQ1Go1VlZWhHXtilqt1q8bEhxM79692nV/hGjU/LcohBBCCCGEaDvVjd6BG8FWrcbHx6fF58srKsjOzr7sduxsbSkpLtY/9vT0xMfHB6WFklOnTwOgUCiws7OjrKzs6ne8BdbW1lhaWbbpNRYWFgQGdiQnJ5eKiorLrq9SKomOjqK6upq0tHTKysqwtramqqqKwMBAvH28qdNoOHv2LAA2ajVKC+UVfZ4/ssbvVlJy8o3elb+U5r9FIYQQQgghRNv9JQMEaltbAjoGUF5WTp1GY/T8lfbE5+bmYm1tRW5unn5ZQEAAISHBbNu2Ha1We8X7fC0plUp69uiBi6sLRUXFZgUIvH28sbS0ZP/+A9TV1Rk8l5WVhYVCQZYZQZU/M7VaTURkJGq1jQQIhBBCCCGEEH84f8kAQaOzSUlcuHDhmm2vrq6Oc+dSrtn22oOnpyfBwcFYWVm16XW2tnbU1tYaBQcAqqurST537lrt4h+OUqnEz8+PwI4doZ2HeQghhBBCCCFEe/lLBwjaytLSEjtbW8rKy03eKF8tW1tbLCwsLjscQaFQ4OjoSF1dnVm9/426du2Kj483aWnpFBUV0bNnD7Nfq1Iq0WjqzV7/cpRKJfb29pSXlZnM4mgLlVKJ2tYWS0tLysrKqKmpAXTHydnZGYVCQXV1NeXl5S1uQ6FQYGNjg7W1FTXVNVRUVpr9/r1iYrBRq0lKSsLa2orAwMAr+hxqtVpfx6G4uBiNieOiVqv1n7O+vuW/h0qlQq1Wo1RaUFpaZrAttVqNtbWV0fJrvQ1HR0dUKhUajYaSkpJWM2isrKywsbFBq9VSVlZ22Wyb9v4tCiGEEEII8VckAQIzdOzYEX9/P4Ne97KyMiyUhmPs1Wo1/fv348yZeKMaBsOHD9P/X6vVsm3bdv3joKAg/Hx99XUENHUa0tLTSUkxzEawsbGhW1gYTs5O+mEQGk09lWYGCTIzM8nIyKCsrAxnJyezXmP4+WwYMWK4/nFlZRX79u3D0dGRXr1iOHHiJPn5+ZfZhpqwsDCcnBz1n6G4uITTp09T2YabcgB3d3dCQ0OxtVUbLN+5cxd1dXWEh4fToYO7fnldbR3ZOTn6OgmguzHt2bMHDg4OBkNLdH+DNFJSUi+7H0nJyZSWllJTU0NQUFCbPoNCoaBTp054eXqisrz0cywpKeHw4SP6x6EhIXh7e+u/I1qtluKiYs7Exxsct8DAQHx9fbC2tjZ4nwsXLnCh8AL+Af4Gz5WUlHL8+HFqa2uv6Tb8/f3o1KmT/nF9vZaLFy9w+vQZg/XCw8NxdXVFpbr0W9JqtRQUFHLq1CmjIIi5v0UhhBBCCCFE20mA4DLc3d0JCQkmJzuH9IwMKioqUKvVuLm5ERxs/s3g8eNxJntFO3bsSFBQINnZ2WRkZFJVVYWnpwedO3emrq6O9PR0/bo9eoRjY23DyZOnKCkpob6+HkdHR7N7rEtLS83eX1Oqq6s5cyZe/7i+vm09/yqViujoKADi4xPIz89HrVbTpUtnoqIiOXDgYKs92s1ZWVlha6vm0KHD+s9mbW2t71FOTkrS32RaW1vj4dGBTp06UVpaSk5ODqDLZHB0dOTcuRRSU1NRKBSo1Wp8vL0JCgqiqKiYixcvtrofhYWFbToOTQUGdsTPz5esrCzS03XfL4VCgZOjo36dgIAAAjoGkJWVRXZ2DpWVlXh4eBAUGEhEz57sP3BAv66drS2A/pjYqtX4+vriH+CPg70D59PSKCwsRKPR4O3tTWBgR0JCgomPT7im28jNzSMnJ5fa2lqUSiVOTk6Eh4cTGhpi8B1ycHCguLiIEydO6v9OLi4uhIV1JSgokOTkS0NXrtVvUQghhBBCCGHaXzpA0Ck01Ci9XavVcvToUf3jgIAAKiuriE9I0PdmlpWVUVZWRseAALPf68KFC0YBAqVSSWCgLjjQ9KYpIyMTNzc3/P399AECV1dXHBwcSEhINOilLywsxM7ODicnR9pbfb32qmo2BHbsiJWVFfv3H9D3epeWlpKQkEjv3r3w9PQkKyurzdttHFIAuiBGo6bDBKqrq0lPz8DXxxc3Nzd9gKD567RaLRUVFSQlJ+Pl7YWPj89lAwRXysLCggD/APLzCwxurrVaLReLivTrBHbsSE52jsE6ukyQUqKjo/Hy8jL4PPX1Wn3ApKKykrNJSTi7OKPR1JOWlqZfLyUlBQcHe1xcXI327Wq30fRvotFouHDhAnl5ubi5uRm9V21Nrf63VV1dTU5ODu7u7nh7exsECK7Vb1EIIYQQQghhmsWN3oEbqay8nOKiIqN/jZRKJc7OTuTl5rY63vtKubm5olRakJVlXP2/sLAQGxsblA2p0+7u7tTX15s1/eL/Kjc3Ny5euGg0lKAxPd/e3r7d96Gqqsoodb4lZWVl2NvZtdu+uLq6olQpWw2KuLq6orJUkZOba/ScbgaKSjw8Olz2vcrLyo2GYoDuM9rYWJs1c8fVbqOqqtrs4pilpaVYWVnpv//t/VsUQgghhBBCgGrG9Okmn8jIzGDv3n3/z959hzV1/X8Af2dCGEnYW1CQKSJDcCFDcSDubd2jtuKu2v4crbuOWltb92i1tWqtWmdddU/cm6WC7B1QCCvJ74+QK+HehICg/bbn9Tw+j7krJyG5ufecz/l83nNz3q+MjAytI+L6+voAUKeEdXWhx1feqPr7+2neRk8PJSUl0NPTQ1lZ+f/0zRGfz4ehkaFaHoPqeDxegz6flZUlHByaQF9fD3w+HwqFAnKZHG+0JCusrqK8AgKBQYO2qTpB1edLWw4JahsNn8Gy0lII9Ok37TWVV1SAw6EHDFVUVILFYoHD4dSa7K8uxzA2NoZzs2YwMDSAnp6y86CivELnEqKqPAU8Hg8ymazRv4sEQRAEQRAEQQDIPV30AAAgAElEQVRcBwd7xhUODvbw8vTE1m3b33OT/nnY7MYJtFDdLMXGxqG0tJRxm+oh8/+GCnoSSSGSkpIY11V/re/K3t4Orq6uSEtLR0JCAorfvIGxUAg3V1edj1FbJv13pfr7N+6zVD2XphV1eI26HsPY2BgBAf4oyC9AbGwcCgsLYWBggKZNnWBubs54CPohlces2aHQWN9FgiAIgiAIgiAALtNNGZ/Pp0rpjR41Ej/v3PUBmvbhlZaWQqFQwPAdwsxVI/5sNpuWgE9a1SlQWVlR69x+aUkJzM3NwOfz1eZ3/y8pKy8Dj8t9pzwGurK2toFEUoi4uLfz9gsKClBaVso4Cv4hqP7+RkZGGiMEVNsYGBgwbiMwENRaFvN9s7aygkKhwMNHj6jP/+vXr1FYWKRzB0FNDfFdJAiCIAiCIAhCO/aP6zeg5r/qN7JmZmYYOmTwB2zihyOTyZCbmwdbW1sYGNQv1FwVcs00770gPx9yuQL29g61HiczK0tZEs/FpV7teO8UClrIQ35ePgyNDCEWixv96f8Xoi3y8/Mhk8nhYM8cxaPaprKikrFSha2tLfT19ZGdrb205PumiidoyAiMhvguEgRBEARBEAShnU7xura2tugR2b2x2/LeGRsbw9TUlPEftyo5WkJCAiorK9CqlQ+sra2rSusZwNTUVKe7UIlEAoVCAWdnZ3A4HLBYLJiZKSMBKmUyvHj+HGKxCN7e3hAKhWCz2eBwODA1NaXmXQPKRHDJScmwsraCh4c7xGIRlbjtn3jDJC2VQl9fD9bW1lSYeFJSEqTSUni3aAEbGxvq9RkaGsLExERtf3d3d4SHh9U7pFwiKYRIJESzZk2p5IdcDucfEz0AKG96E6uqA3h5eoLLVbZNT0+PGmmXy+WIi4+DSCSEl5cXRCIR+Hw+7O3t0Lx5cxTkF9AqMnxohYWFYLPZ8PLyglgsBovFAovF0jk5pCbv+l0kCIIgCIIgCEI7ne+W3N3dUVpair/PnW/M9rxXzs7NNK5T1YAvLS1FTMwtNG/eHB4e7rQ50TXLJNYklUqRmJiIpk5NERLSUblPRSUePHyI8vJyvEpJQVl5OZo6OSEgwF9t3ydPnqrlJnj+4gVKpFI4OjrCxsZGbVu5/H3MZNddamoarKys4enpgfLycuVIuEyGO3fuoLmLC9zc3MBmv30vS4pLcOPmTeoxi6V8TfVNyvjixQsIBPpwcnJSG32XyxXIZqgI8KGkpaWhsqICTZs2RceOwdRymUyOS5cuQaFQICsrG5WVMjR3caESWlZWypCeno7ExMQP1XSNcnJykJyUjCaOTdQqLCgUCpQUa07IWJt3/S4SBEEQBEEQBKEdy8bWjnZnOW3qFGo0s6bLV64gJuZWozTGxMSk0WrONwQ2mw1jY2NwOBzI5TK8fv2GlldAGyMjI/B4PBQWFjLe+KpGRFkAJIWFWkO0BQIBBAJl9vrS0lKUaMmE/6FwOByIRCIUFhbS3icWiwVDQ0Pw+XxIpVLa/PpWPj7g8Xm4dev2O7XBwMCAilSoqKjAmzdvGj35YH3p6+tDIBCgvLwcxRoqLQgEAvB4PLx58+YfX9GCx+PB2NgYACCXyVD0+nWDtfldv4sEQRAEQRAEQagzMTHRPYJAJbhDBxQVvUZsbGxjtOkfTS6Xo7CwsN7715ZMrry8XOcEhEw31f80MplMY0JChUKh8f1QJchMSU155zaUlJT8IztPmJSWlmqsZqHyv/B3V6moqD35Zn2963eRIAiCIAiCIAg6tr+fH22hrJZRvpBqodAE0dCMjY0hk8uQkpL6oZtCEARBEARBEATxn8Ft2tQJd+7eVVt45vQZtPRpCVaNyuempiYwNDRUS55HEA2tqKgIV69e+9DNIAiCIAiCIAiC+E/hZufQS6TFxccjLj6etrxTp3C08vF5H+0iCIIgCIIgCIIgCOI9Yl+6dPlDt4EgCIIgCIIgCIIgiA+MVmTez88XgwYO/BBtIQiCIAiCIAiCIAjiA6F1ELg4u8DBwf5DtIUgCIIgCIIgCIIgiA+E/dnMGVTEwGczZ1CdA9WXEwRBEARBEARBEATx78ZOSUlF4vNEAEBKSirKysqo/6uWEwRBEARBEARBEATx78b+ff9+3L17DwDw+/79yM7Oof6vWq6SmpqG4uJiVFZWvveGEgRBEARBEARBEATReFg2tnaKD90IFRMTExQUFHzoZhAEQRAEQRAEQRDEf4qJiQnYQqFQ4wbt27dTe9wmKAhWVlaN3S6CIAiCIAiCIAiCIN4zbkTnzjhw8KDawq5dIuDl5QUWi4Wy0jLcvnMHdnZ2aN++Hdq3b4fi4hJs2rz5AzWZIAiCIAiCIAiCIIiGxn3z5jVtoZOTE1gsFgAgOLgD5Ao5ggIDqfX6+nrvrYEEQRAEQRAEQRAEQTQ+9qnTZ2gLMzIy327AZiMsNBQGBgbUstdv3ryXxhEEQRAEQRAEQRAE8X5wVf9pExQEFpuF+PgEiESa8xIAgEgoRFhoKMzNzWBiYoItW7c1ekMJgiAIgiAIgiAIgmg8VAdB27ZtwGaz0a5tW7UNcnNz8eLFS9jZ2cLW1hYsFgssFgt+fr7UNuHhYTh37vz7azVBEARBEARBEARBEA2KC7ztHKjp3PnzuHfvPvVYJBJh1MgR4PF4att5uLuTDgKCIAiCIAiCIAiC+B/GBoCnT58hJzdXbUVBQYFa5wAAFBYW4rc9e9WW5eTm4tfdvzVyMwmCIAiCIAiCIAiCaExsQHnjv2vXL3j48BG1IjMzi3GH3NxcyOVyAEBlZSV27foFhYWF76GpBEEQBEEQBEEQBEE0FrV5BebmZtT/RWKR5p2qpiNwOBx0iYhopKYRBEEQBEEQBEEQBPG+sAHA3t4OUyZHw9bWllphbWUFoZBezSCqRyT1fxaLBW/vFoie9Ol7aCpBEARBEARBEARBEI2FDQClpWXg8/nqK9hsjBw5AnZ2dtSyPr17wc3NjXaQ4uLiRm4mQRAEQRAEQRAEQRCNiQso8wpIJIUQi0WQy+VUKUM9Ph9DBg9i3PH169dgs9kwMDDArdt33mujCYIgCIIgCIIgCIJoWNyePaNw9OgxHD5yGJWVMkgkEri5ualNJahJLpdjy9Zt77GZBEEQBEEQBEEQBEE0JnZCQgIAIDc3DxKJBADA5/PUNpLJZJBVVS4AlNMPxCLNSQwJgiAIgiAIgiAIgvjfwo6NjaMtdHR0pP4vl8uxcdNm7N27V+M2BEEQBEEQBEEQBEH8b+MyLTx//gJklTI0b+6Ce/fuo6ysDJmZWUhMTISNjQ3OnTuP+KrIA4IgCIIgCIIgCIIg/vexbGztFB+6ESomJiYoKCj40M0gCIIgCIIgCIIgiP8UExMTZZlDgiAIgiAIgiAIgiD+20gHAUEQBEEQBEEQBEEQpIOAIAiCIAiCIAiCIAjSQUAQBEEQBEEQBEEQBEgHAUEQBEEQBEEQBEEQIB0EBEEQBEEQBEEQBEGAdBAQBEEQBEEQBEEQBAHSQaBR68DWGDlq1Dsfx9zcHMtXrESrVq3qvG+btu1gamamcX23bt3xxdy579K8epk1ew58fX3rtI+9gz38/P1r3Y7L4WBS9CQ4OzvXt3k0NjY2mDZ9Opq7ujbYMQniQ3Nzc4eBgUGDHCsysgd69u4FFotFW9e5c2cEd+zYIM/Tq3dv+AcENMix/u0MDA0REhYKkUgEPT09DB48BB4eHgAAsVik0zEsLS3UHoeHd8L8BQsatJ3+AQGY8PGEOu0jEmlvv5+/P75c+BWEQmG92tSufTtMnjqlXvs2puCOHdGnb19wudz38nx8Ph/eLb1hXIf3MSgoCJOio8Fmf9jLw9aBrdHSp6XasoGDBsHHx6dex+NyuZgwcSIcmjShrevQoQMiI3vU67gA4N3SG65ubvXePyQ0FAaGhtTj0LAwfLnwK/D5/Dodh8ViYeCgQQhorfkcGxYehsFDh+h8zEnRkxAeHs64btr06fW6tv1fYmhoiMGDh7zT35fP52NS9CS069C+AVv27yEWi+Hp6fmhm0HU8H5+pf5h/AMCEBkZiW/XrEFxcTHjNpYWVugcEYEnT57gzu3b9XoeDoeDadOnw97eDoZGRnXaVygUYuInE3Hl8mVs37aNuY1WlnBxcan1WF27dcNHw4fX6fkBYPzYsSgvL6ctd3Fxxq3bMQCAwYOHoEfPKGpdclISlixeDBtbW7i5u6FVq1Zo0sQRQqEQWZmZSHr5Evn5+Rqfs4W3N9q0bYenT5/h+fPndWqvk5MTjIXGtOVcDheeXl6wsLDAvn17GfdNTEiEVCpVWzZr9hzaBYouxo4ahUqZrM77Ef9OwR07wtbGts77JT5P1HruGTFyBEpLS7F0yRIAQNNmTWGk5Twj0DdATMxN2nJDQ0P07tsHz54+BY/HQ9t27XDp4kUoFAoAQPuOHVH8+g0uX7qktb3tOrTHJ598Slt+7cpVbNq0EXw+H1E9e+LihQu1nlM5HA5mzPwM9+/fx9kzp6nlup7LJBIJFn71FfLz8mrd9p/CwMAAHp6e8HD3gJu7G5o4NgGLxcZ9v7vYtHETevSMQnlFOZKTk7F0+ddIiI/Huu+/p/5OTL5cuAgPHjzA1s2bAQAWFhZwad5c5zZ5eniCw+PQlkskhUh59QoAENG1C8pKy3Q+Zs9evdA5IgIrvv4aGenpjNuYmprCxaV5vW9Sc7JzEBgYhOCOD3H50sVatze3sICNjTXjupRXr7Dux/Va97937x7WrllT6/N0CA6GY5MmOPznn7Vuq8Ln8+HmXvvNSWVFJZ49e6a2zMzcHLPnfI7v1q7F3Tt3dHq+pORkTJocjZSUFBw9ckSnfYLatEH05Mk6basSFxeLZUuWMq4zNDTEhI8n4tq1a3j44CG1vEdUFLhcLh48eFCn5wKAjh1DEBwcjDOnT9PWtQ4Kgq2dHU6cOF7n4wqFQkyeMhXp6elYvHCh1u8jE0cnJ4wbPx5mZmY4eOAAAOXAkotLc3A49O+eNgqFAiampugcEYHEhARIJIW0bVr5+sFAoFvHsoGhIdq0bYfnz1/Q1jVt1hT+AQG4cuVKndr4rrxber/T/jnZOcjMzAQAdAwJgUwmw1Utr8HQyAg9ekbh9evXiI+Lq9dzlpeXw8TUFIMGDsL1q9fq9BlZvmIl7O3t6vR8K5Yvx9OnT6nHIpEI3bp1r9Mxakp+9Qo3rl9r8LYByoGDsPBw7Ni+HZcvXYJYLEbXrt10PuaLl89xK+YW9bhzRBeMHDWyTu0CgNycHMycMaPO+/1b/Sc7CIqKimBpZYV58xdg2dIljJ0Ep0+fQs/evRAZGVmvDgIWi4VPJk2Co5MTpFIpho8YgeTkJKSmpOq0f5euXQGgThcStdmwfgOKi19Tj0NCQxEYGIQ1q1dDrpBTyzt16qzTaD8AnL9wHk+fPQEAjB03HunpGejQIRijx47BmzdvwOVykZGege++/RaJiYm1Hq91YGuUl5cj5ib9RqY2g4YMRosWmn88mjg6YvaczxnXLZg/H8lJSbTlSS9fYv/+39WWhYV1gmcLL6z/YZ3a8sDAIISEhta53XU15/PPkZOTi592bG/053pXLBYL/QcMQHJyktoJ/L8kKKhNvTqatN1IOzk5oYmjI7Zs2kQtG/bRR3Bzc9d6zPnzMvEqOVlt2bCPhoHL5eLXX3+FWCzCsI8+gqWFJe1zr+Lg4ICevXpBXyDAurVr1TrDKisrsfbbtzdKQ4YOo/4fEhoKfX19SAoljBd5Bfn5SE1NAwCwAJSUvMHIUSNha2uNXTt3AQBu3bqF9HTlNpYWVsjOyaL2NzAwRO8+fWFnZ4vLFy+pdQ6YW1jg27Vrtb43mmSkp+Obb75BTnZ2vfavzcCBgxAaFqo2ypubk4Nfdv2ClFcpiI+PA6fqRpnNYaOkpASbNmzEZ7NnYfyE8di2dRvjBadT06YQCoVITUmpd9umTp+mNrKpcuf2bXz/3Xewt7eDh7s7WCw2dv36K7W+uLgYy5Yspv6e1Z04fhzNm7tg/oIFWLF8OVIY2sdmKyNZZHI5bZ0uEhISkJyUhO6RkTp1ELRp0waDBg9mXDdtijIS4djRo3j27Clt/fDhI3Rqk7FQCFdXV1y9cqVONwhiExONv1vVvS4qwrQpU7Bj505s374NF89f0Ljt5KlTERgYqPV4AwcNwsBBgxjXnTt3Dj/v2EFbXvMaQ5Ohw7R38n00YgT09fURHh5OG73uHhmJ7pGRtH2ev3iBRV9+yXg8FouFLl27ICE+HkkvX9bavroYO24cAGD9jz/WuXMAANq0bQMADXajvfuXX+Dl5QVPLy9cu3qNtt7ezg63dbym9fJSjureuH6dts7X1w9yuRyPHz16twbXkS7fBW1OnzyFX3/9BQDg5u6Odu3aIT8vj9a5Vh8eHh7g8phvq9LS0uHm5o5u3bsjNZX5nJyWmsY4gPbo4SOcPHmi1ud3dHRiPI8JhUK1gbz6iImJoXUQNETbAGDXzp2wtLTC+AkTIJEUoKjodZ3ae/nyZbXry7t3biMrK4O2HZvFxmezZ+PsmdO4d+8ebX15WYXOz/lf8J/sIEiIj8eyJUvw5cKF+Gz2LKz9di2cnBxp27188QLGxsaMF7MZ6RnIzc1lPD6LxcLETz5FUFAQbly/jj/++APzFyzAF/83Fyu//prxgqg6AwMDRHTpgqtXrlDPIRAIsHnrVsbtq1+YAcDPO37CuXN/07aLi32GgoIC6rHqZvrJ48dqF/m+vn5a21dddlYWsrOy4OLiAjMzMxw7egS5ubl49PgRcrKz8dXixUhPS6N1Dvj5+2O6lp66TVu21PrcKSkpmPd//0c9XrViJfX/fv37IzAwEP/3xRe0H+127dth9JixWLl8OZ6/oPeMV/em+A0ePVT/AfRp2QqyigrackdHp1rb3BCsra1RIJG8l+d6VxaWlujVuzfy8vL+NR0EVpaWeFNcrDH6qKZvVq+q83No+q6rDBw0GPn5+bh69Sq1TNNoXOvA1pg8ZQquXb1G6xzw8/dHcMcQ7Nq5C7k5OQCAw4f+xIBBA3Hh4gXqppjDYaN9hw4IDw9H02bNcO/uPZw7e5Z2AyeXy9W+F716K98jNpuN7t2VIxiDBzOHt16+dAlbq773lTIZNm7YCDaHi84RXVBQIMHRI0eQn5eH/Lw8BAYGYtCQwcjOykLMrRhwOVx07d4dFeXl+Pabb2gjjCXFxTh+9JjW97Sm4JCOEAqFyMzMRJ6Gc31DuHfvHgoK8pGekYHsrCxMmjwZ5RXlOHvmDLVNpUwGmUwGHo8HAHj8+BG2bt2CiRMnoqysHLt27qQdt0OHYADKKVaq99zNww0sFkvj3+De/Xtqo2SfTJwIQBlNxWazsGrlSrXtBwwchOSkZLXOpDFjx0FaUsLYOQAAMpkM3639DtOmT8f/zZuHGdOmoaxMPQKBy1G+zsoKzRdsmiJWaqr5+6iSmJiAxQsXAVDe/B87ehQAsHL1asTHxVGRe6rpEKlpqbRzPgCUlJbSlnWO6AIzU1O1ZbZ2tuBwODAwMNT4/ld39eplpKamITsrCyOromZatWqFmbNmYfOmTbh65QpEIhFev34NedX3kFuHEeeszEzs2kX/3OgiN5c5MqfmNYYmffuVaFzXoUMHtG3bFj+s+572m/Hzrl04dfIk9vz2W53a261bN9ja2WHF8uV12q82AwcOgq+fH9Z9/z11/qwLLoeDjsEdkRAfj+ysrNp30OC7detgWuPz9smnk/DJp5OoxyOHD4dYLIKFpSUSExO0Hq958+bQF+gjqE1b5OXloYmjclqGQq489wBAm7ZtkZebC1c3+tTN6p29DW2klgiycePHw8/PD9GTJmncprod27ahWbNmmDp9Or5csAA52dngcjjoP2AgtY3AUBlt0dKnJYyN30ao5uXnq0W3AcDkKVNqnc4zdNgwjet27dxFOyYASAoljOceXaWkpGh937gcDpYsXw4bGxt8PmcOsqoiLHTxrm1T+WHd91j29QpET5mKWTNmMLZ35OjR6Ny5MxZ9+aXWa/f8/HzGjhbV+TE9PbNB2vxv95/sIACAzMxM/PrLLri4uKBpUyfMmj1H47ZMPZZ7fvsNf52g95rx+XyMHTce7dq3Q1xcLDZv2gSZTIavly/HvPnzMf/LL/H9d9/h6ZMnGp+ve49IcLlcHDx4EADg6emJxMRE2gWup5cXmjg2wckTf6ktT05i7iF3c/dQ6903M1fmN/Bq0UItgqDmDw1AH4EbN248xo0bT31Ru3bvjqdPn1KdHzXD9WvKyMigvR4XVxe4ubnj3Lm/IS3Wvj8AFEg0T1V48vgJ+vTti5YtW9JuFrp27Y7MjIxaOwcampOTE+Z/+SXycnMxf948VGi5+P23yMnOxp7du5HVSKOv75tzs2b4avFifPvNN7h///4HaYOLS3N4t/TGju3boVAoIBQKUVRUxLitj48PJkVPRk5OHn755Re1dU5Nm2JSdDSkUils7Wwx8ZNPYGBgACMjI3C5XIwYMRzfrvkWAODh6QkWh40rVy5jzerVKKnl+11T54jOMLewwKqVKxlHnL5avJjWkadQKLD+hx9QNGoUPDw9cPzYMepGKCYmBvfv30f7Du3RMSQUzs7OKCoqwrYtWxjDj0tKSjROL6pJIBDg00mfQigU4vy589j580/U8zaGxMSEGhfszKOQUqkURoZvp5Bcu3IVTZo0QU4W/caEz+ejbbu2AJTzmWvSNDqjKYzW0ckRly6oj8T7BwTAz98fq1etpC62gtq0UebdWcbcWaUil8uxeeNGBAQG0joHVO0HgAqGKW41nTt3DtJizTecTNoHd6jT9tV1DAlBQIA/9d1g3qYjnJycGNdpmx9e3YuXz2k3WX369UNKSgquXrkCFouF+QsW4MnTp4yj+bWRlpbqfJFsbW0NT09PPH78CNnZmm+Ev//hB52fPy4ulrZszhefUwMXU6ZOY9xPUwSBypJFi5CQ8Pb7JBaL0adfP8THxdFCm99F54gI9OzdC/v27KVFetnZ2UFfX7/WaZLtOwbDWCiEsVDI2JGlraP404kTqU7qjRvWg8/nQywSQ1L4dvDA0NBY7ZrPpyp3FNN7e/nyZWoq0phx49VCx1XXwDKZDGNGjYKrmxusra3V1qkd69JFbN2ivZP7n0Amk2Hjhg1YtHgxJkVHY9FXX4HN4TCeHz2rojJUkpKSGG/mq0co6Kq2CLfg4GAEBwfX6Zh1MWDwINjZ2eHsmTN16hwAGq5tZWVlWLvmG3h5t8CbN29o6+0d7BEWFoZ79+5pvXYfPnwEunTrqvW5Ro4aqXUKgqbp1f81/9kOAgC4dvUaFYKlrXcNAJYuW4aysjIsWbxY4zbOzs74dNIkWFpZ4d69e/hx3TpYWFjA3MICjx89wqKFCzHniy8we84cHDt2FH8ePARZjbnqIpEIERFdcOrkSeTn5cHC0hKz5szB9WvXqNE1lZGjRsHO3k7nC99J0cy9qp/Nnl3rvoUSCVavUo4eTZ4yFRcuXMDjRw+RnpEBS0sLBAQEYM3qb3RqB6AM263ebjabjVXfrEHSy5f4ecdPOh9HZdz48Yzh/dpeW80fZKZeyRYtvDWOQGlarklIWCj4fD5sbG3h1cIb9+/drdP+/4sUCgX++uuv2jf8l5s2fTps7ezwuQ7ftdoMGToEz58/x4Xz58HlcDBv/nwkJibSzg/BHTti7LhxyMjIxKoVX6OkRsRDVmYm0tPSUFJSgvLyMpRJSyEpkKBAko/Hjx+jb79+VJLUx48e0yIhTE1NYWllidhn9Av96sSmJujvPxA3b9zQGI7KAqCQ02+MFQoFdv38M/U4MrIHhgwbyngMoVCo9n3XFEmljYeHByZ++imMjIzw47p1iImJqdP+jam4pASGNcL99/62h3HbqJ49YWxsTOuQGTx4CLr3iMTokbrPz7SwtIRIJEJ8/NuOA2tra0yYOJEa9VRF2Q376CMkJSWBx+NRy549ecqYk6VEKsWli8zh/3w9PuRyuU65XE4cP17n0Vd3r/onxDIzM4NTU+1JdL+cP1/tcUhoKMaNH48N6zcwhunqonv37mjatCmWLFJegygUCmzbug3/N28uHty7xxgyqws+nw9fX1+ITMQQGYsgEotgLBRCKBRCJBRCKBJRHTbPX7zAskWLNP5dpk2ZolMEwZcLv2JcXj0KsKF8Ej0JAoGgQTupu3TtguEjRuL8ufM4flx9oENPTw9Tp02DlbU15syerfGzyePx0L//AEgkEly9rD69wNXNFc1dXXHy5F+QVTC/19UHGOJi42BkZIQf1q/Hjm3bcPnyZbTw9sbHn0zEN6tWUQNSrXxaISc7GzE3357XIqMi8exZLO16pOaNbnh4J4youqnq3bsPioqKMHP6dNqN1MrVq1FW3riDH9p+BwDmazOJpBBTJ0fTlr9KTsZvu3fDx6cVDAQClEilavcDllZW+GbNGuz9bQ+Vo2JSdDSsbWwa4JXopnrnjTbeLb3rPAXD09MT3bsrO9z27mH+PXlfbUtLS0NaGj3yhMViYey48eBwODilw/VkzehiFS6Hgx07d2qM1NA1Ku2/4j/dQVCdvYM9OnXujN/37qONfqvm+2pKFmhgYIABAwcivFMnsNlsnD55Crt3/wqFQoHOnSPQuUsERo8ciZzsbCxcsAAzZs5E79590L59B/y+bx81v4vP52PiJ59AVlmJC+fPwalpU/Tt2xccDhvnz51/59dY88d76LBh6B4ZSUuqN3LUKHSOiFDbt6JaSL1CLkd6ehr1+KPhw5Hy6hUVelYfXbp0haWlBQ7sv1hrEprs7BxaL+fJv/5iTMBWF+kZ9DlLDZmD4OKFi2jh1QJZWdl4+uTxO7UVUJ40La2sIBKJkPTypcYeTwOBAM7NXaBQAKkprxgTF6kIBAJYWllBKDTGq+RXKCxU3wWW3GsAACAASURBVJbNZqOpkxPy8vO0HkcXbDYb5uZmMDE1RVHRa2RmZFCjyHw+Hw4ODigpKUEGw98FUJ7snZo2hcBAgIqKCiQnJdcauQIoM7sbGwuRlJRE66Crrrb34kPoHhkJZxcXzP3iCwBAWKdOsLG1xc5qIeZ8Ph/9Bw6kQvqzMjMgNjGBpMa0FKlUii81ZLTn8Xh4+vSJ1iR/Dg4O+Gz2bLXedj6fT7s4u3//Pl4lv0JA69a0db/v24djR4+CzWZDruP83Zp5DoYOG468vFycPnUSAMDh8DDzs5k6HUtFeQ4fgE6dI1BQUIBFX31V61SwhrJp82bGef5MF7lWlpaMy1XvI6DsuIns0QNJL182yPxgfz9/VFZW4knVTYZAIMDc+fNhIBDAQCCgXfSZ1Jgz/+nEiagsLkbrwNZoVuPG+ujRIygpoY/+GxkZ6fx5AICevXvBzdW9XtN5GptAIMDAQYOQ9PIlrXMguGNHTPj4Y8ycMUNriHqLFt4YMGgQnsU+g0MTe3i39IZYbAITExFKy8owZtx4JHzxOUrrGNkDKHMcRE95W/FBJpOhsLAQCoUCZmZm2L//d+Tn5SMnJwfZWVnvJQHvjM8+q3OlpCOHD+OP/fvVlnWO6AJPj4bNjt6jZxQGDx6Cmzdu4Oef6JEbn3z6KWxsbbF//+9aO6769esPsViMrZs34/Lly2rrBgwciOaurjh04KBOv2kAMGDQIBQXF+NmVf6m2KdPkZGejvHjx2P2rFkQCoXw9fPDwQMHcOTwYQDKZIg9ekbh3Nm/dZ4C6N3SG94tvbFv717Gaw4el4syhmk3jaFm/qwuXbuhuasr7dqsU6fOaOasOaH3mdOnGZNX1peLiwv8/XWv1vN3HTuyG4qNjQ2mzZhBVS+Sy2Ro07YtYm7ebNSoubrq07cvlZBdJpfB188PTx4/Zvz8KaAAFAq4ubth3vy31zcymQwTxo6lHk/4+GO16kyPHz3ClavvN+HmPx3pIKjGzc0Dy5Yvx7rvv0dStYR1Xbt1g1QqxbVq831VOoaEYNiwYTAwNERRURG2bt6sNcNucXExli1digEDBiKqVxQmRUejS9cu+OH7dbCxsUELb+XN8Tffvg03On3yFBITE+Dn74/mLm8zUDs7u4DD4dDmMx448AcqKyvr+zbUSUhYKDqGhAB4e0E7Z9YsrPrmbTSBc7NmVAjSxAkTaD94VtbWGDBIOeer/8ABtT7nXydO0OYgqnoeWwe21hiayKSkuJiaZ8ukIXMQJL18idmzZuncNm3cXF2xfuNGKmu9QiFHamoatm3dgpcv1KeYrFqzRq1cWE52Ng4dPKiWFMnX1xfDPvoIVtbqmbw/+fhjlJSUgM/nY/yE8fDzD6BGk7Kys/HbL7/UOnL17dq1eBYbS/Uyi8UiTJ/5GRwdHdWyNEulUhw8cAAeHh7w9fMFi6VMzFYileLggT9w+uQpals/f39ET55MzclWycrMxLHjx2gJulgsFgYNGYyQkFDqPauUyZAQH4+tW7aoXZxrey9UZtb4OzJl5m1ozs2aYcjQoZCWlmLGZ59BKBTCwMAAly9dxNMnT8DhcBAWFoY+/frB0NAQx48fg0AgQEhIKPwDAnDt6jUcO3pYLWzZxtYW5ubMpVTT0pQZ5ivLK2BlYw0fHx+1i7HANm2Qn59P/Ug/ffyYijIaM2Ys4uMTcPXqZRQUFCA1JRXbduzAyZN/UaNZY8a8/bFms1hQQLcLkpp5Dvr2K4GkoIBaVtfSYIBy1EsgEODIkcM4cey4zhflDeHkyZPQ4+tRj9u2bwe5TIabN9Q7PFv5+cLU1BTnztIvJhMS4qn/GxoaIjkpCT8x3LzUR+ugQGSkp8PD04Na9uDefSigQGhYGDXaFtWzJ8LDw6ks0DVHY1r5+tFCUf8+9zdjB4GxsTG4HA7EYpHGjshrV67i2hXlb7K1tQ1cXLSP6vN4POgLBHhdVERLZle9UoAenw+xiQm8W3pDUiDROH1HV6NGj4GRkZFOVQ40cfdwB4/Hg6eHJ5ybOSM1LQ0KuQzSklLcunETPr6tMHzESGzbUvtInibVp4oAyvKmI0ePxt9nzjL+jZi86xSD6qrniKgNU6eZm7sbPhr+EVJSUsDXU36/VKPBdTmOqvOGzWZjzNixCAkNxcMHD7FxwwbatKi+/frBPyAADx48wNHDmitAODg4oGtkd8THxdE6B+rDyckJ4eHhOLD/D+p8XCmTYd3332PV6lXo0rUrxCYmYLPZaiVGmzgqc28lv0pmPC6TjiGhSE9LY5xiCwBcHp9x2lBjqJk/KzAwiPHazNfXD83qWTlblSBWJq+9Y0yV+6BzRJc6Jdnbt2+v1ijmxphiYCwUYtbs2eDz+Xj+4gWcmzWDn78/JkVHI7JHD2z48Ueq2oM279q2mtXCNm3aSJ3XAWVls959+kAikUAsFkMoFGL8hI+RlZWFFcuW0aY7Pnn8GDk52YiLjaO9p9VztGzdsoUWdenQpAmOHz0GOalCBuA/2EFQM9mfKuwoNSUVXy2YjxEjR+LLRYtw8sQJHDxwAK7NXdG+QwccOniQcc54bm4uXhcX4+ixozh7+oxOJ0aFQoH9+3/H3Tu3MXz0aBQXl6CgoAASiQS7du5CVlYG9PX1MX7Cxyh6/Rq//74PANDSx4exHm3NE9GRI4cZOwg0/XjvYEhwpavMjAxs3rQJRUWFsLW1w7CPPkJOTg51ozB8+Ajk5uZRWU5rvj8GhoaYMWMG+Hw+XhcV1ZpcZpMOoUwAMCU6utYR3+HDR6DDO8xF/ZDYbDZ27dyJxIQElJSUwNnFGSNHjsJns2bTwv5WfL0cqSmpYLFYsLKyQu8+fTB2wgTExcdTSeiEYhGsrK3VqjmIxWLqwnDqtOnw8PTAkT8P4/KVy+DxeIiKisLU6dOxdNGiOuVz4PL4aNasGY4fO4Yjhw+joqICrXxbYeCgwfho+HDcvXMH33/3PWKfPUPz5s3Rt39/DB06DDE3blKj4MZCY/B4PCxeuBCJiYkwEAjQwrslunTtgnHjxuN10Wu1sl7du0eiR48oXLxwAZcuXkROTg4CAgLQb8AAzJo1Sy2ZZW3vBQD8sX8/Xr58+5qTkmu/wDI2NtYpORnAfJObmpaG+Pg4ZGfnIDk5CVFRPVFeXo5ff/kV/fr3R3inThAKhXj8+BF2//IrFap34I8D6BjSEVFRPdG2XVvs//13arS5a7duGmtc7/71V5w6eRLHjh5B7759MX3mTFrZrT27d1P/l0gKIZEoL8xKy8qRm5tDu1BLT0unlpWWvf2MsjlcyGVvOwhqjiBWHx1kilJwcWn+ThVEBAIBrl+7hgP7/6j3Merrz0OH1B67ebihvKKCNnWMy+Oic0QEfv99HxQKBXg8HoYMHYoDBw6oTR9JSUnROhWuLpybNUPzqrKI1aMCxo8dq0yCSE9voNHWzZupTsLayuKZVOXBcWjShPpM1ZehoSGievZEWHg4JAUFWLx4MW26jaocoIqpmRl8fHxw4/p17K7xWWOxWdCUJ6Km4OBgtGvfDqdPna5z2d7qTpw4gadPnyI+NhaVMhk6R3RBUWEhFTXXpm07lNZh1JYFVq0jhIVFyt9PoVBYawfBzRs3cPPGDZ2f/30ICw1HRUUFNq7/EVOmTQcAFL95w5isNKB1AIyNjRkjNVWflcFDhiAkNBQ3b97E5g0baO9fQOsA9O3XD1mZmdi4XntZTElhofJ888e7n29YLBZGjRmD/Px8/PXX25v2wMBA+Pj64sL5izA2NkanTp0gk8ng4fl2Hr2bmxuKiorqNO98w48/KqfCdumCNm3b4utly9SuN/T03l8HQc38WWITE3CqTW9SYcqrpStu1SBEebnur+nsmdO0EPbhw0dQ0cR1MfeLd6vawMTExASff/EFzC0ssHnTRri6ucO5WTPcvXMHa1avxoSJE7FoyRJs3rRJa3nUhmjbzZs3kPLqFQyNDGn5clq1aoWp06cjLTUVR44cQfTkySgqKsK3a77B7DmfY9bnn+PrZcvU7s1UuaGmTZ8O/4C3URxyuRzjx4yhHk+YOJHWsbFg/nydp2z/F/znOggqKiqoHwg3D3eYm1uorduxfTsSExMxeuxYtGnbFlAokJWZiaNVIVk1PX3yBLNn1i2cVUVVlkc1uqtQKHD2zGnw+XwsX7ESFRUVWL1yJXXy/XnHjnolI6peGqw6TWUOVbQl0QvuEIxePXth/rx5iItVzk0Vi01QWFgImUxG3QSUlJZqzHJqZGSE/5s3D8bGxrhz+zZcXenZcD+0hsxB0JCexcaqlR569PARftq+A1/Mm4uQ0FC1cDlVaU2FQoHMzEzs+e03tO/QAS29W+Lvv8+qHbeoWqeK6ma8pU9LtPRpiS2bNqlFHez8+Wf4BwQgoms3PN+4oc6vIT09nRqpvX3rNmQyOWbMnInTp05Ro/EPHjxAgUSCpcuWwadVK1y8cEHtGHlVIfAlUiliYm7i1q0YLFqyBEOGDqV+2DgcDnr27oWrV66oTRM6e/YsXqWkYP6CBWjfvj2tzBTTe6HyKjm5zllwjarqKddXWVkZVamgZ+9eMDIywrIlSyCVSlFWXoYXL17gyJ+HaRmqXxcV4fjRY7hw7jzGT5hAK9eXmpqm9kPP5/Oxrdp5xsLCAufPncPiS5e0tq+Ftzc6deqE77/7rs6vjc/jolL2tlPz1s2bSK+KdKiZcKi8vBzjq4UKfrnwK6SlplF/25rt/7fIysoGh8OBhYU5srNz0K9/f3SOiMD1a9epv7mu9ei1nbsyMjKoXBm9+/YBoCxztmH9elrYJtOxGuK8aGGh/F329PCqd7bpps2aon27DggJC4Wenh4kkkJcuniJsTJCRno6NdqkqYqBip6evk434618/TBuwgQAwK1bN6k2qSKYAGVlA0B5o2Zj8zZiKTcnV21qVUlxsVpi444hHZGemkZ1EKimLuhaxYDL40JWS5RhZqYyPN7KykrjSGJIWCjGjRuv03Nqkp+fj+lTp9KWu7g0f6fP0sGDf+D69WtqEVPFxcWMNwC29nawZbO13hwcPHAAZeVlOHTgIC1ywM7ODh9P/ARv3rzBN6tW1dqh8roq2rQhmJmZwdzcHL/v3Yu58+fj8J9/4t7du+gQ3AFGxkIsXrgQIaGhYHM4+GXnLoweOwbm5ubIzc2Fl5cXnjxmTpjdpVtX2rlXJpNBoVBAIpHg/v37GDhoEEaNHq02Equvr//eIrA05Zhimuuuy5RILpeL/v3Vo1hFJmIAQMuWPrC0sAIA2Ds0gZGREQYPHoKcnBxarhsTExMEtG6NWzExtGsHXX27di3MLSxq31ALpjw8Dg4OmP355zA2NsbGDRtx4/o1uFYrj/zgwQPMnzsXM2fNwvQZM3D0yBH8sX+/2me+Idt2ueq6wsbWVq2DoEu3rhg6dBgKCwuxetVKtSkicbFx2LB+PaZMnYpJ0dGM1xxMy1gsFo4fPYakly9w9szpBvsO/lv95zoIKisrqR+BocOGqXUQqFy6eBHpaemYO38euFwu/jx0qFHn3dUMYRwxciSMjI2wbPFi5GRnw9zCAgJ9/XrPiVWVBqtJU5lDJgaGhujRowd8/fxhYGgIWzs7XLtyFU2dmqJn715YuWIFrK2tkZ6erlObXFyaY/KUyTAwNMSqFSvRpk0bGAcwZ/Ktjx9q6cFXqTmSVN3+/b/j1Cn1hCiGhsYoK5Wq3czUVN/a3e/q6bOnyM3Jga+vr9b5dEVFRSgrK9O5V72Vrx+kUqlaST1A+V169vQZHJo4vFO7VVT1qe3s7dXC9VNevYJMJoOVlVWtx1AoFDh/7m+MGTsOTk5OSEpKgndLHxgaGtLaDwDxcXHIzspC68DABqtDrUn1G6/aaMteHdSmDQYMGIDNmzbh+fPnEAqFuHr5Ch7cvw+TqtBoTU4cP66W4RsA9PX4avtwOOrTNtp37Iji12+oH3IVU1NTtVJCQpEQTZs1ox736t0bvXr3xrUrV7Fp00atr5evp4+KajWIq/8twsLrMEz9L2FsaITIyB5o5tIMTZo44pddO5FZdcPYxNERBoZG6NqtGy5duKjWIZSelkYbITW3tEBQUBBu3riB3Jy35Rojo3ogMSEB8XHxatsXVY0cOzg4oJWvn9Y8HQCoaLHAwCC08vXFls2bAAAeHp6I6tmzzq9dLBbD2NgYJVIpvFp4Aft024/FZqNTp85wbu4CN1dXWFhaoqCgADeuXcft27fw6NGjBplXa2pighwtmfwB5Wd25KjRkMvlYFeFKAPKJI5u1S7GVSZ+8ona47NnzqiVrrR3sIeJiQn1WKCvD5FYpPa9zc7OQZ6Opfb4PB5t/q6mBGI1b8Kqd9A9uHef+vszUX0Gtm/bhvx85nwmTPOIq0/HUEV9Vk8S98XcucjKzMJPO7ZrfO7s7BytFRfqqqysDAf/OEBbLhaLMP2zz8DlcrF61SpkVV2zRfWMwp7dv2kdTa9tSqSm34Ht27dR0+hyc3Px+axZKJFK4eHlhclTpuDnn36Cd0sfKurq4oULyMjIQGJCAgYOHoTgjh1x48YNNHF0pPIR1HT71m2cP//25tKnZSt0iuhMPc7KzMSWTZsxeeoU3Lx5Aw8fPIRAIACLxYJUxykp76pm/ixNZQ5HjhqFgNaBGo8jEAjQvn17XL16VWMnfvXRaJUePaOQmJhAuwl3dXPDiJEjkZycXO8Ogs2bNoOvx9O4fty48cjJycWRI39q3EY1OKTSqVNnDBoyGAqFAitXfK0xwbBEIsHihQsRPXkyevbqBQ9PT6xZvZqqmtEYbatu8tQpCAwMQmpKKr75ZjVj544q0nTa9Gn4dFI0Nm5QXvPPWzCf8RxbXW0DNd+tXas1cuK/4j/XQaALHo+Hrt27gcPhIC42Dn369gWPy2v00BMOh4MxY8eiY0gICgsLMWvOHIhEIrDZbLx48QKrV67Exlp6vM6dO0dFGeg6H1/bFAMqG6hCgR5RUYiPi0dFVfjrxfMXYG5hAa8WLWBqZgZnZ2e81DHUXCgSgsPlYvnSpUhKSkKbNm1QIpXSksvUNHkKfaSBSUNMMUiulodC5cuFC6GvL8D2LZvfe5lEXWRmZard+LPZbPTt3w/+/q1hamZKZenl8XhqF67amIjFEAgE2PkLc+kepnqz9fH6tbIck0AgUFuuUCgglUphUFWPuDbpacobKWsbWyQlJcHKyhIAkKUhYVROTg4sLCzr2+xGMbFq5LEmhyZNMG78eBQUSNC+fTCGDB0GsViMbVu3wtPDE+06tNd63JrRAoCyxJKumY85HA7atG2Lrl27wsbWFvPmztWYiOva1WtUDgKVCR9/jAnVcjkAylFPY2Mjnec5N8YUgw9FIBCgbdu2sLWzh52dLWxsbanvb0goF69SUnDxwgWkvEqBVCqFTCaDr58/vDw9IZFIsOe33WrHS0lJof1OhYaFISgoCEcOH1brZO7eIxLPnj2jJXZTycrKQmJiIl5rmIPP4XLU8kE4OjqhorycelxbPXBN/Pz8AQDHjx3FgAEDtJbxrE4gEGDk6FFITkrC1atXcffuXarTsSG5urmBxWJh5OjRahU2VMRiEUaOGo3cnBwcPXoU46t9l1URQCq6JimM6tGT9t22sraGV4sW1OO/TpzA/n269aYYGxtTOUZUft+3D8nJSWrLho8YiZKSEhw88DYUXlZtKpBEIqFugMwtLLB06VJs3bqVKvun+gzEx8cjQ4fBA03ViGpSlV1k6jys7b1sSJZWVpg3fz4MDQ3x/dq1SIhXdrZZW1kjLCwcpqZm+PYbzdWdMtLTGac81FbFIDlJfUqbah721s2bYWFhjgkff4yKigqcP/92yoSqfGnMzRgEd+wIQ0MjSKVS3NFwI5Sfl6cWvWNhTv+NjIm5idu32mLUqNGYP28ejI2V0THF76mDoKF0j4xE7z59cOPGDdq89U8nRcPHtxX4fD527/qFFnXJpIlDE8hkMtr1MJvNZhwEY8q6Xz1Hx4SJE2FmboYVy5ZTy8rKy1FYVKhThJWNjQ3GjR8PVzc3pKak4ttv19T6HamsrMS677/HmLFjERoWhkWLF2Pe3LkoKytr0LYxuXr5CtgsDjZt3KC13OD9e3dx6OBB5Oa97fj+bfdutSit6po1dUb/gQPw26+7kZauuYMiKUn3nBz/ZqSDoAYHBwd8Gj0ZtrY22LF9Oy5euIA+ffqi34D+sLG1wQ/r1tU6olJfIpGISvhnaGSEpJcvcOqvk4iLfYbUtDRwuco/119//YXHjx7S9o+ucfMcFxvH2LsvFptg1OjRavOcExMT8echeuictEQZSllSUoLpU6dCIpGo5QHIzVFWFIjoHAE3d3ccPHhQp9d6984dxD59qpZghCm5TE0KHUeAGiKCIDy8Eypllbh08SJ69u6F9LR0rFi+HEOGDsX8r77CsWNHcfCPA9R7NmXqNJSUFGusdvE+sNnqIaaTJk+Gt7c3jhw+jLt37qCwqAheXp6YFF17GLIKi8VCcXExNqz/kXF9g9WLrXofWWDRV8nlAMNyJvKqZEI69n/8o4nFIhQVvYZcLodIJMKixYvB5XLBZrPB5XERc+MmMjIy8PDBA1y6eFFtpP6LeXMBQO2Hm0ltUwwAZYfe6LFjERQUBAMDAeJi47Fh/XqtWbqZchAcP36MCpW2srJGRWUFevXtAw6Hg+wc3UrVNUYVg7rqHNEFnTp3grm5OWJjY7Fvz2+0mvW6kMvlGDl6NNhsNkqkUiTGJ4DFYiEjM4Px75aUnIzg4GC8efMGixcupCVoYuLs7IyKigqdbtCqKy8vx4rly/FxjdFtFX2BPmQyGTWKbWVlBR7/bTSKvZ19nZ5PJSAwAMXFxThx7Dh69+6DzhEROHiAPnJbk1QqxZxZsxq82khhYSF10+Dp5QUTExMkvXwJB3tl5FTNhIcSSSH27tmDy5cuwbmWxIm62rRpo9p3e/HSpUhPTaNF5ug6xUAoEuFJjaSqyclJtO9r7LNYBLUJwuNHj2nXBjV5enrCwNCQ8Zzg6urKmAxVlcBUZfu2bbTfz5p5o1R0La/WWExNTTF/wZfQ0+Nj9aqV1FRLAHj8+BEO/nEA/QcOQFTPnlTOl5pSU9MYB57qU8UAqCp9uXkLVn/7LYqLixmvl04cO4rQsFB06twJf585+87Xs3v37sGixYthbWMNLlc5qvymqrO/uhYtvNG7b288ffIUfx46VOvniYlqio4qgXBdchDweFxq+YvnL6jRcAOBAF27dcOD+/fw5s0btf14PB58/XxxKyYGPB4PHUNCdOogcHNXdiKKxGK1G3GFQoETx47Tti+QaB9kcXdzY4wgFglFGiMGY5/FUtOEHRwc4OziguPHjuHAH7onMVcoFNixfTuKi4shl8sZo2HetW1M7tWhbGvNCJiXL17CwMCA8dxrbWON8vJyZGRo/i1MTEh8r0mK/8lIB0EVBwcH9OrTB4GBrZGXm4dlS5dSvcF//nkIefl5GDd+PL6Y+39Y++1arTeW9ZWfn48f1/2A+Pg4xrAkVQdB9RKD1clqfOGKiopo25lbWGD06DEokEjwIvE52rZri0MHD6Jvv37o2rU7Nqz/kTpx1qQpVOrM2TMYMnQYpFIpEuLjaWUSq1cxqB4mWPPilunEXhOrlru+WzG3aD3AX1Ul7NqyaRPtIllV551J125d8SolBZcuXkRERFc8uH8Pd27fxq6dO3H//j188ukkNHVqRpXWevrkCUaNGY3bt25prWTRWFgsFmxt7ZCUpBw143K5aN26NY4dOYrjx96OUtyKuYXyCbrf1Ofn58HQ0BDPE5/rPMr7Idna2QEAFQaclaWcc29na0ubfw8oe9eTdUgyCAAVVRdTbB0vxN9V98geaN26NT6bOROFhYX4bfdvSEtNQVxcXIOVIaptigEAODo2Qam0FIcOHsD16zc0jirXpnqSQlEHEXXzGRcXi4cP6J2eTBqjikFd+Pn7IzKyO3bt3ImkpCR07doNU6ZOwxeff17ni96ysjLs2L4d6WlpeP78ORQKhcYa8ebm5jAyUEbRbN64kZoXbu9gj7TUNMbn5nI4aOXri7i42HpNk9PW+Wdpbgkej0eLPqlrHW5LKyt4t/DG33+fhZ2dHVq08Ma5c39DJpPh+tVr6BwRgWNHj9baEamQy3XqHGCxWHBq6kSr9lIdh8tFK18/tPL1wZlTp6mEn7169UJCQgIKCgogEr2NkBAIBDA0NERurnIk69TJkzq88g9DT08PIpFIawlTN3c3FBRI8OTxY4SFh8Hd3R3Pnj3TetxWrZRJRcsZbiLGjWfOU1Bz+pGNrS1WrtKtVCVT9vTHjx9h1QrNUx4aioODAz6dFA0ej4uvly9njFQ5fPhPNHd1xYCBAxEXF0ddTza2rt27IT8/HwYGBpg0eTLWrF6ttj4rOxuZGRmwsbXFsWPMHRd1kZ2Vhc+mT0eJVIqgNm0AMF8rRnTtAjc3d7i5uSMxMaFeo8s1p+jUJQdB9eXVK3b0iOoJgUCAYww37n379YO+vj7OnzsHgIUvF34F75beWtsuEonQ3LU5WCw2ekT1wM6ffqbWKRSKOkci29vbwcLSEs+fJ9LWubm7YbY782utHkkTExODuLi4enee7tvL3OaGaFtjcGjioPV3SNPnBlB2+P4To4M/BNJBAGVt6aXLl0Mul+PM6bPYt3cPrXfr8qVLUMjlmDBxIjoEd1ArudaQVEmHqlPVu2fqla2Ldu3bYdToMcjNycHKFSvQI0o5D+fo4cN4/fo1RowcgVWrV2Prli1UJlBdnDl1Gv37D0BBVaj5mdOnce/eXcZtM9KZa9oDYKypXVfzFyzA7Vu38fx5IiZPnYq1a9Zg448/YPSYsViydCl2bN+Ga1eVCZ3c3N3w+f/NxU/bt9PmVzd3dYWNra3GiIiHDx5i2ZIlqKyogImJCezs7HHu3N/o1RQ4CQAAIABJREFUENIRo8aMwZxZs2i9tO4e7pg1ew7y8vKwZNEiWm/1u+rduw/EYhH1+lQj8godM25rcu/ePXSO6ILuPSI/SJb3umCxWAgODoZEIkFiovJH69HDBygpLkavPr3x4MEDtRup0LAwmJqZaQyzrkn1HXyXrMh1YWlpCYGBAdXmmpmR+Xw+Wnh7Iyc7u945SnSZYvDo4WNajXkulwufVq1w984dxptTHp+PoKAgeLdsiW0MI4BXrlxRyzUgFoveKfu1sVAIJydH8HjKDgKZonEivfwDAnDy1EnqHLlv314EtQmCvb19vf4Gly5erHWbFi28ET1lMjUtqJmzM9UJGdSmjVpHZXWhYWEQiUT4fe81ndri4+OD4JAQbFy/vtaRRVs7W9y7exdrv/0WQO1lDpn07tUb7TsGIzE+AX//fRZ9+/eHTCbDsaqw6xN/nUBIWCh6REXhkI7RabUxMTHBosVLsGf3bvz1lzLHjJ6eHpybOcPP3x9mZmawsbFB27ZtkRD/Nr9Dhw4d4OnlhRVff43wTp3UjmksFOKbNWvw6cSJGjvYG4upqSmaODrBwEAf169dx+pVK5HySvPn0KmpE1gsFp4n0i/qVYYNG47snGxs27IFJcXFCA4JoToI/Pz9aecbK0tL+Acop4YMGz6c+kyofD5njk4RLNWTRar80yIIgoKC8PEnn6CsrAzLly5Vex/09PRgYWEOsYkpTExMkZL6Ci19WmLsuHGYW61STmNp1aoVQkLDsOLrr+Ho6IiRo0YhPDwc586do7bpERUFG1tlcsz27Tvg+HH6FIe6Ug32WFpaQiaTMXYQPHnymKpMY8GQ+0sXqik6kZE9MGTYUFoOAk1UOQimTo5WW25oaIiu3bshLi6WmoKh4ujkhG6RkYiJiaGuJW5cv44JH0/EF3PmaBwsCQkNBYvFxrNnzxAeHo4b127UWtJTm6ioXgCU5/njx0+oTX2NiYnBj+u0T8tVaejIqoZsG6D87jS06pWotu3Ygb179lLXUJOioyEUi6hIPU9PT3wxd26Dt+F/2X+2g6B58+bwbukDI2Mj2DdpgnXff4/niYlaE4pcuXIF+Xn5ePqsceudi8ViBLRuDUdHR5ibm8OpaVO8KS7GwgULat+ZgYtLcwwaPBjuHu64c/s2Nm6gz+s5e+YMMjMyMXnaVMycNQuxz2Kxb+8enUozderUCXw+Hza2tvh00qfYsnmLWgZmXb1rmcP2HTrA1c0Nv/7yC3g8HkxMTGBgYICkpCSsXrUK4ydMgImJKcRiMbpHRuLPQ4dw+OAhjJ8wHllZWWo/EFFRUSiRSnH71i2Nz6caVfLx8cFns2dj5owZOHTgD8yaPQcRXbrQ6gS3bdte+T7Z2MDN3Z2ap1kfYpEYHh4eSE5OhoWlJQIDAxHVsyfu3rlDZbSulMmQnJyMsPBw5OXl4t7de1Qt2Zrl6rR59PAR7t29i6ionlDIFbh54wbS09MhFArh5uaGmJiYer+OdxU9eTK2bN6E7OwcGAgEGDFqFFzd3LBh/dsa1TKZDD/t2IFJkydjUnQ0Tp86pSxz2Lo1hgwdiqdPnuicoDA/Px+5OTmIiIjAw4cPkZ2VBWdnZ+V7zZCzQqW8vAJGhoZgsVg6XySKxWJ4enmpzWO0sbGBq5sbAlq3hp2dHUxNTaFQKLB969Z6dxDoMsVAU/umTZ+OJYsWUYkPeVwuBg4cBAsLc3Tv3h3FxcWIffYMLFbt00N69e4NoUiEH9cpy7FyOBx0i+wOPX19tPD2pnWO1CSTyTBr9mwALKSmpOJFYv3LymmlUIDDYtdYpKg1uqk+BAIBhg4bhtCwMKS8eoV1332HsRMmoFPnzjj1118okUohFolRWkbPqm9haYmBgwfjxYsXOtdaNzY2RuvWrVEzpWSbtm2VVX2qmIjFcHZxwZ7ffqvX6xKJlZUBQsJCERNzE1s2bUabtu0QGBiI0ydPUaNLGenpuHj+Anr17o1nz55qTKpVUVGuc1SPqmxjatrb0PYhw4aiU6fOkEgkuH7tGh4+fICHDx5SHVaeHp4YPXYs7t+/j6dPntA7CIyMIJPJGj3CysrSEq7u7hCLxTAzM8OmrVthUJWzJTExAdeuXqt1ZNa5mQsqKysZR72bNmuKNm3a4vadW+jXrz/27dmDkydPIrJHD/xRlZTUz88PlTKZWkWlMePGofz/2bvv8Ciq7oHj390kSzZ1A+mE9BBICCH03pHeBVQUFUFU7IqAXd/fawPBXkAsgCggKqAiNfTeSyCdkJDee9/fHyFDlt2EgALyej7P4/M4M3dm7m42Ye+Zc88tL+f75cuZ+tBDTHnggUbXQ7id3P/gAwwYUFOoLyIigrHjxqNzsMfOToe9vV29g5zmzZszcNCgBosH/1WtW7fm0ZkzWbP6J6KjooiOiqJL1y6Mu/NOwsPD0ev1jBo9mjsnTODQwUPk5ORw58QJJCQkcPq08WemvlUMGuLp5VXvkomb/tzI9m3hLF7ylcGc8Vtp5OhRaDQaVv1o+Fn18GjOnLlzKSkqYkWdpXy//fZb3ps3jxfmzOG9t982yoK1srJi2LBhpKam8uHChbz97rvMnjuH75ddX/HtwFaBdOvRnS2bN+Hj68czzzzD559/ZjCd5Vb5O/sWGhrK9BkzABg1chRpKalGY4/CwkJiYqIpLpb0/5vlXxkgqK0pkJOTw/Zt23j0sceIiY6mqrKSiIiIBlMZI85GYKXVYu/gcM1zOq/G3z+AJ59+Gt2lL0/l5eVkZWZy5swZjh65PJh0d29uMhXfzMIwNbhDx46MGDUKP19fiktK+Pbrrw0iyVc6ffoUL86Zw/Tp0wlu04bX3niDEydO8MVnnxk8FVGp1eirawY53j4+TJg0iXVr1xIdHc2TTz3Fgg8+YNPGjezfv/9vTyNSqdUmn4h7eXsz5f772fjnn5w/fx6/S9XUa9ON9Xq9shRPh44dGTpsGGvXrmXdurW0Dg7ioWnTeHHOHKqqqmjVuhVh7duzft065R/Eyopymjk2MznA8/Xzo7KykqzMTKUmQ7/+/Y0CBOHh2wgKak1aWjqnTjYundqU0tIyQtqGGHwGcnPz+PmnNaxbZzgfa9GXX/DEE0/y4NSHePDyynAUl5SQndP44oIffvABd06YQP8BAxgzdqyyX6+vJuqJJ6+7Uu9fpTYzY/6Chcp2Wno6n3z0sVEmzoEDBygtLeWeyZN55bWaFO7S0lLCt21j5Q8/XNM9Fy36koemP8z8S5W2q6qq+GHF9w0GCBLOn6db9268+NLLxFyxioApTbSWdO/WDa1Wy7atNb+zzs5OvDtvHtXV1WRmZhEXF8tPq1dx7Ogx3N3djf4m2FhZAxjtLywsbDC9unYwr780t7OyvAJbW1vUarXBtAYvb28A0tPT8fL25sEHp2Jubk7LVi35/bffOH3qlJI2D1BcXKIs62bqnt4+vsqTT51Ox/OzZuHk7Mz3y7/Hzt6Wd+fNZ9u2rbzx+usGv4dvvv6Gcp3775uCTqfDxcUVnYOOgoKCq3427ezsMDMza/TqI4cPH+a+KVNITk4mISGBwYOHoNfrSbrOAE1darUaK60VZaU5aDQa/vvWWzg6ObF50yZWfP89VVVV/LR6Na+8+iozn3iSzz/7FD8/f4PPu1qtpmv3btxzz2QqKir45OOPTd6rsrISnU5nsM/T24ucnByjgUDE2Qh+rzOPesjwYVRVVRkstWr0Wqj9HBn+vbSxsWHI4CEU5OezcuVKdu7YgbePDw9Ne4jMzExWrTL8sr58+TLatA3hmWef4+MPPzI5mElMSMSyvyXPzZpFUgNPzwF69OpJfn6+QbBh544d9T7pGz5yBBMmTCQ6OopPL72X1VVV2NjYKm08PFuQkZFxQ54QO7u4cN+UKfj4+ChLIldXV5OSkkJSYiLnE86TcD6BOBPBfDs7W6W/UPN71rtPb6IiI5Unr15engA88sij2NrZkZiYyEcLF9K7V29GjBrJiuXf069/f2bNns1nn36Km7s7B/Zf/rlPmnQXQcHBrF69iu3h4eh0OsaNH0+3bt2u6d85e3v7RtcOAtNTDKAm463uKgh/p7qrXgUHB5OSkkJ2djZJiUnk5ORQUFBAfl4+BQX5FBYWUlRUREFBAXNfepHx48ezb+9eCgsLCWwVSLvQsHrv0zKwZsnnsePHmSxSWGvlyh8xNzNj5KUVY7Zt3apM4YSaavNWVlZYWloyddo0unTpQuS5SD6/VE/IL8Cfp599hgXz5xusHATGqxgAXLkatqeXF1Cz7K9OZ09oaCgHDxhnwdYaOXoUxcUlnI1oeLrK301tpoYrvjfa2tkxYMBADh08pAS4VSoVAwYOZOzYsVRXVfHO228bTMUpLiri/fnzePGll3np1VdZ/OWXnL/0775KpeLBqVOxsrZm6dLvKC4uZsH77/PC7Nnc/+ADlJWVoVLVvGcXk5KuGmxxdHJi5uNPUFBQwMofV6LVann19dd56eVXOHL48A154t5Yf2ffOnfuwqMzH+P06dMs+vJLRo8azauvv86JEyc4fvQoCefPk5CQQFRkpPJvvY2NDa6urmitaoKjDg5NuZCQoPwsxN/jXxcgaNqsGaPHjuH06VN8+nHNfPudO3Ywbvw4nnnuuUY96YKawhiNTUturNzcHGJjoklMTOTUyZPExMQYfOGwtq75wj906FCGDh161eulpaZgb2vL+nXr2LxpY6PWgc3OyuLdd96hc+cu9O7Th5LiIiU48Orrr+Pv7091dTWpqan4+wcw96UXiYw8pxSdmfPCC4weM5YRo0YxcdIko+ufPHHSZCpsfbRaLZ989hkWdYIfqSnGEWo/Xz/iz8crT7RS0tIoLS3l2eefN3ndkydOKnUkvlq0CA8PD2Xwk5qSwpbNm1n76+UlWg4dPszQoUPrreS/e/du5Wf19ZIlJguwnI+PZ1Y9/bkWL86dg05nj0cLT1SqmjWzU1NTTX45TUpMYs7s2fj5+aO1sgQgPTWNtCvm4u8I364sm2RKdXU1q1auZPWqVTg5O+Ps7ExhYWGjqoTXphzXyszIMEojhZqMB1P7gXozSz7+8EPMLSxwdXUhLS29waJ5J06c4MSJEzg7O2Fra8f58+dN/iN9tffi3NlzzHr2WXQ6HS08W3D+fMJV5+Rv2PAH9g463N3dGTZiGCrV1Z82JyUmsWrlKmXwl56ewWuvvkLihUSj6Sv33n+/EhS70pXTByIjzxlVU4eap4dvvPkfACoqKjh7aQAVFxfL0GHD+HbpUqNzYmKiycvLo6CggOXLlnHwwIF6C/zs2bOL4cNHMHy46SWGqqurWfljTbBm8OAhODk7887bbynBjCOHjzB8+DDmvvhivVWKjV77s8bFClUqFe+9/z4uzoZVuet7On2lY0eP4uTkyN333EMzR0ciz0XywQcL/1JNiBdfeplWrS/PrT127Bjl5eWsWrWKoqJCg6fC0VFR/LxmDePGj+ezL76gurpayUYKbBXI4088ib29PSnJyXzwwQf1BmkTExPp3aePUhi31o7t2w22U1KSqU66XPehVetWDBgwkFUrVxqkrWbnZBMff57uPbrzyKOPXTo3xeipenFxMTExMSxbtoyC/HysrKyYNWsWFRUVLHz/faMAfVlZGfPfe4/Zc+fy3KznmfX880avacf2cFp4eeLr60vLgS2NVkKpKz4unp/X/GTwN7q+gNlTTz9Nh44d2bVzJ18vWaL8vThz5gxdu3VTKpKXlZWxevWqeu8JNcsqDx02zOSxBQsXGu2rXWqrrLQUHx8fYmNiiIqMIjYuhtiY2HqLfJmbmbHgw4+UBw1VVVXKF+fWrVvj3rw5v/56OZA8dNgwki9eZNu2rRw+fEQZEO3ctZNBgwbz3Tff8uknn/DkU0/x37feorikhH379gMw7s7xDB85gpMnTirF13795RdOnzrNxEmT6NS5Zmm5huoKVFZW8uTjjxsUg6zrnzTFYOl335Gbk9NggTVTvvv2Ox6cOhV7e3sKCwsJCGh51aXWAIYMafh73sqVP6JSqwkKDmLz5k2sWG64qklmRgYurq5KoHH37t0s/+47JTj00cKFPP3Ms7wwZw6rV60yqFV05SoGpoSEhDDprruU7ZKSEsK3bjVqp9FoGDt2HMNHjmDZ0qXX/P5djzlz5xIUHKxsX/lku6SoiN/X/8a+S8EutVrNK6+9hp+fHzEx0SyuJxM2Pi6e+e/N47HHZ/Lm//0fv/76Cz//tIaJd02iS9euHD1yRJnmmXD+PK+89BJ3TpxI127dUKnU/N9//1tvn0tKSnj6ySdp0kTDq6+9jqVlEz7+6EPKysooKytjzgsvMGr0aHr27EnTpk3p3Llzg0uDb9q4ieXLjP/d/it0Ovu/tW9jx48jNzeXjz/8kPLycpYvX8bRI0cYNW4MD0ydWu/5V7pydabGMjMzIyg4CE/PmmBX+U34bN4uVG7uzW/spKhr4ODgYLAk1o3Sq1cvDhw4YPRFxMrKiuLiYnx8ffD09MLZydlgKbjikmJSUpIpKyvjQsKFGzKnpyFarZZZs19g/dp1Jit8PjdrFmfPRBhEkBsycNAguvfozv+9+Z9Gfbl1c3fH0bGZMiDSaDSMHTeO1atWGZ2vUqnwDwjA0tIwkpiammayUFy7du3w8vJm7VrjdVMHDhxIs2aOACRcSGjwqVVdtXOSTTkbcbbRlVxr+fn5mVxqr6qqmrMRETd8fqG4rE+/vjz00DSeeuKJm/I3439V+w4dcHJyUoqqBQUFoVabERd3uSClWq2mZWBLgyBdrWut+Ovv769E/a+Uk52trARgbm6Ol6dnvcWCHBwccHN3x8zMONCSm5NLcnJyg09o2nfoQIB/gLJ9JuIMp09d35JMf4c2bUIIvvRlNis7m/BLRfoa0qp1K1q2DOTkiRMGT06mTZ9OVFQUu3ftavDvupW1Ne3DwmheZ7WB5JRk9u3Z0+C8XpVKRbce3dm7e4/J4xqNhsBWgZSVlRMbE9OoKukDB91BdHRUg1k4jk5OhIS0IXxbeL1t/m7de/agvKyMw4eMp4P5+wcoQdf6goR+vr5MnnIf33z9DU0dHGjVqnWj771z187rzlLs3qM7LTw8KSkt4dSpkwYBkGHDhrNp45/Kz7hps2YmCxZqtVpUXJ5fbmtnR6+evTh95jQXLhV1dXJ2pm/fvvy0enW9RTJ9fH1xcnFGZ6fDxtbGaJWarOzsBqcPaTQa5rw4l00bNyn/9j/19NPExcexfu26Rr0fj818jMSkpHrbT7rrLhydnJQMkduNtbV1vbUvVCoVd0++h0MHDipPyutSq9WMGz+eDX/8cc31M6ytrfH1qwlMV1VVExMdbTIDNyAggOdfeIGVP/zQYBZrY7UMDCSsXRirVq2s93tXq9atCG3bDoDi0mK2bt5y1SlAIW1DaNbMke3hV/8bo9FoGDt+HBs3bCA3Nw+dTsfwESNY+eOPJr9b2tvb0659GC7OLiZXagJIz0hT/r6NHD2KUydO1vtUXKfT4ePjg4NDU+x19mgsjIvzNvZvyLBhw+nYuWOjxwJ/Z99aeHpiY21tsgiqi7MzaenphIaG0ry5B7a2l7O29OjJzcslPS2DqqoKg5VWWnh68uDUB/nyiy/rnfJS13vz5+Pi4kzEmQjmz5t3w1aqu504ODj8OwMEQojbmwQIhBBCiNuDrZ3dda9+I4S4uRwcHPgfWClcCPFvk3A+gd/X/0ZpqXFxNiGEEEL8c0hwQIjby7+uBoEQ4vZ3Pj6+UfUPhBBCCCGEEI0nGQRCCCGEEEIIIYSQAIEQQgghhBBCCCEkQCCEEEIIIYQQQggkQCCEEEIIIYQQQggkQCCEEEIIIYQQQggkQCCEEEIIIYQQQgjqBAjc3d1p3brVreyLEEIIIYQQQgghbhE11AQH5s97j1defpnQ0La3uk+3rdDQUMzNzes9/uJLLzNy9KhGXUuj0fDq66/RSoI24i/S6XQ8+9yz2NrZNap9QMuWPPvcs7QMDPxL9w0NDaVdu3bK9qRJdxEQEGDQpm+/fkyYMPEv3edaqVQqJkycSMdOHett069/Pybdfdffdk9/f39aeHo2un1YWBgdOhr2r2u37o3+GQohhBBCCHE9zAF69+ql7GgX2o4TJ07esg7dDE89/bTRl+9r9fxzz5GelqZsW1lbM/OJJ4iKjOL9+fPQ6/VG5wS0DCAjM6NR1zczM8PfPwBbW1tl32MzZ9K1W7cGz3vqiSfIyclp5KsQt6PhI0YQEtqW99+bR0VFBQDTZ8ygV53fY4AF8+dz/Phx8vLy8PbxZdAdg/j5pzUNXtvMzIxp06fTrFkzwsO3E9I2RDmm10NUZCTl5eWN6ud9U6YQHR3N8ePHa/o9cgS5eblER0crbcLat8fCov6gmpu7O7179W7U/a5UVl7Gr7/8YrRfr9fj0LQpAwcNIiY6mtzcPKM27cLaY6W1uq77mnL/Aw+QnZ3NwgULrtpWpVLx4EMPERUVxZHDhwFwcXXl0cceZc1PP7Fu7dq/rV9CCCGEEELUZQ7g7e2t7EhJTb1Vfblpfv75Z7Zt22qwLyg4mOHDR/DhBx9QXl4GwJQp95OZmckff/xudI3cKwbhxUVFfPD+Al6YO4cxY8fyy88/35C+Z2dlsWTJV0b7W7cOYsTIkTfknjdCWPv2PDRtGq+89JIENK5RZmYmQa2DmDLlfoPPQmxcHD//tBqAWS/MxtzCXBngR0dF07NnL6KjogBIS0s3CHDVmnzvZNzc3AB45tlnjY4fOXyYDz/44Kp99PP1xdnFha+XLGmwXQsPDw7sP1DvcTc3N4aPHHHV+5lSXFRkMkAA8P2yZQQHBxMUHMzePXuNjns0b87hS4Pz+rz25pv4+fqaPHb+/HleffllZVulUlFdXd2ofnfq1AmdTsfmTRuVfX379qWiooKtW7Y06hpCCCGEEEJcD3MAnc5e2XHx4kWjRo6Ojrz00ou4ODvz3dKlbNy46eb18AZIvHCBxAuG+3Q6BwAizpyhpKQEgJLSUvJy8zh18lSjrhtxNoLNmzZxMSnpb+1vXWXl5Sb7c7ulHtvZ22F3m/X5n+LA/v04uzgzYcJE9u3bS0REBABFBYUGnw07W3seeOpBg3NnvTAbgN/X/8bKlT8aHOvcuTMDB93B5k2bWLZ0qcGxMWPHMm78eH7/zThYZsqgwUNISUlR+maKnZ0djk5ORMdE19vm6JEjTLn3XpPH5s2fT3x8PJ99+mmj+vTBRx/RtGlTg32PPPoYjzz6mLI95d570enscXJ2JqaBfgEs/+47rKxNZxmUFJcYbKvVZlRWVjaqn6PHjiUmJprIc5EAaLVa+vXvT052NiNG1B8EzMjIMAp8CiGEEEIIcS3MAYOBWvIVAQIXZ2deffUVHBxqBtB333UX+/cfIC/POC33dvL8rBdoa6LewpeLFxtse3t7071nD4N9n37yCQf27wdg9OgxjJ9wp8HxwUOGAFBYWMiLc+aQm5vbqD5ZWFiw5JtvDPY98eRTAOzZvbtR1xD/Dr+tW0+HDh3p17+/MgjXYzitJTs7iycff5x7Jt/D5599jl6vp2+/fpSXlxk9NQ8MbMWMRx8lKTGJH77/3uCYjY0NQ4YMYf++fVcdNENNwLFz1y6sWLbM6Pds8r33Mvnee6mqquLzTz8D4OlnnjE4P+JsBO/8962r3qeJpSWlpaVXbVfr888+RaPRoLPXkZt3+XfS2tqWoqICZTs0LAy4/LtX165du1j85ZcAxMbGNni/Ll274u3lDYD9pSDspEmX6xqYmgLRpUsXWrRowf/95z/KvhEjR2JlZYWVlVWD2RQxMdESIBBCCCGEEH/JpQwCHQAFBQXk1BnM6nT2vPLKy0pwAGqK513Ll/J/svPx8axevQpo3BSDpk2b8dC0aQbX2LtvL3HxhgMFlUrFjBmPkJae1ujgAIC+uprf1/8GQDPHZnTt1o3Dhw6TlppKfHw8HTt1xM3NjaXLl1/3axb/G/R6PQvmzyc/Px8AM5Uaa2trg5oBAIUFBbQLa09gq0DOnT3HwIEDOXPmjEGAICg4mCeffhoLCwt27dpJ6+Agg2sMGTIMrZWW48ePGV0/JzubpCTDoOLosWMxNzOjuLSE1atXsXHjBqAme2Hr1i0cPXIEfTV069GdxMREfvxhBQB9+vbF3d2DH1f80Kj3wMrKisLCwka1BYg8F4mNjQ0ff/opX3/1Fbt27aJNSAgPPzKD+e+9R8SZM0BNHZaM9HQOHjionDtsxDDOnj3H8WNHG32/Dh06GNQMsbW1xaOFh7JdkJ9vECCwtrZm8n33UV5eTlRkTfaAo5MTQ4YOZevWLXz3zbf89+23ycnOYf689xrdDyGEEEIIIRrL3NraWtmIjY1T/l+j0TDr+eeNUnIPHjxEWVnZTevgjVRYdDkluzFTDNzc3Y2ukZGeTkZ6usG+4cNHYGtnxwcm5mrbWFsTFBRkMvW6sqpKSfseOWoUXbt1I+LMabZcmnfcsVPHf0wNAo1Gg6enF5mZGQ0GQVQqFY5OTjg2a0ZlZSUxMTEmCzg2xNzcHDc3N3QONYGs2JhYiouLDdq4ubvj6NiMqqpqUpKTG6xr4Ofnh5W1FSXFJcTFxV11brijkxOurq5A/UX67OzscHZ24Xx8HJVVVdf0+q5XbXAAwEKjwd/fX5lCUKuyqor4+HgCAlqSl5ePR4sWfPzhh8rxnj17Mn3Gw6hUNSue3n3PPfXer24qfq26T9Sh5ufQr19/ZTvh/HmD9qkpqZw6eQozMzOeeuZpUlJSlN+vXr16kZKczPn4+Ku+ditrazQaDdnZ11a/4s6JEykqKuLAgZq6B+ciIkhJTmbatGnMev557OzsCGvfnp/XrFGKATo6OjJ85Ai2bdnKoYOHAHBydsbV1aXBe321eLEy/eGLxYvZtmULq1auBOCBqVNpFxqqtFXlvv5fAAAgAElEQVSpVDzx5JPodDqDz9d9U6ZQVlbGqh9XXtPrFEIIIYQQ4nqY1w0AJCQkKP//yCMz8PHxMWiclZXFokWLblrnbkceHs0ZM24sO7ZvJzoqCisrK4PBbHCbNrQLC+O39etZvWpVvddp1ao1AGPGjWPnzp3KoOGv1CAYM2Ys4+4cz+OPPWYwuHRwcODDjz9m8aJF7Nq50+CcBQsXcvbcOWUQ2MLTk2nTpuPl7YVaXTOozM3NZfOmTaxft87g3KeefpqQtm3RaDTKvtonoaa4ubvz2muvcfbsWT768EP0ej29+/Th/gcewMLCQmmXfPEic2bXDIR1Onvefvc96ga6oGZ6x/59e1mx/HuDAfsdQwZz7733KdsVFRWcOXOGRV98YfA0Wq1Wc+9999G9Rw+srC7PM4+Li+P1V19VtsPCwrh78mQlgFBVVcWRI4f58vMvlBUGbgY7e1u2bNnC0m+/VfbZ29sT0jaEA/v2kZOby8hRo7iYlISzqwuaJk1ITEzk1KlT7LmUTdChQwdmTJ9ucN2x48Yxdtw4Hp0xg6KiIoNjX9QJDEDNIPfhGTMoLSnB6oqfx5W69+hBkyZNlMAc1GToxMTGKNshbUMIah1s8nxrm5rrBwcH0eyKIOaV1vy0msqqKry9venfvz9rVv+k/D5VVlXx0Ycf8t6897hj8GB0Dg6o1Wrs7S/XZfH08gIg4cLlv4/du3U3mlp0pdqVTszMzLDSag2mZWk0GoPPx8RJkwgKDiY/Px9LS0ugZtWHsLAwPvv0UyVoKYQQQgghxI1k3rTp5S/ocfE1GQRjx4yha5cuBg0rKip49733KP4f+qLapk2IUbp+Y2oQ1Eer1fLU089QUlLC8mXLcHBw4K133uGrxYuV5coOHDhAWmoq4+8cj6urG198/pnRQNJKqyXoUpq3uYUFY8ePY+UPNZkFTTQaozRvAI/mHkb7rnTmzGnG3Tme0NBQdu3apexvF1azVn1AQIBBgMDZ2QlHJyfOrqlZGs/CwoJZL7xAZWUlS75awvHjx2jR3IP+gwYyYeJEsrKz2Lt7j3K+l5cXEWdOs3DBQvR6PWq1Gju7y8s21uXj68Nzz8/izw1/8uuvNWnXTZs25cGpU9m9axfr168nPS0NlUqFf0CAcp65hQZra2t+Wr2adWvXYmZmRuugILp168rAQXdQXaVn+fJlSvsD+/azb+8+CvLzsdJq8W8ZwMzHn+Due+5m8aLLP/sxY8cycNAgtoeHs3nTJpKSkmqWngzwV9q0DAzkqWeeISY6mu+XLSM6KoqAli2Z8eijPP7EE41a0u5aOTo5sWDhQmV7wx9/8MOKFTg4NOP4seMGbVu1bs3Mxx83usasF2Zz8OBBPvnoI/Ly8lj0xRcGc+PrcnJ2prikxCg4YMqwEcPx8/Nj+fJlBkEYUwYMHEhS0kU8PJqj1WopKSnBvXlztoVvU9oEtQ6+6goGjVmu9JdffkZVXc39Dz5IdnY2Gzb8oRzr3LkzoWFhbA/fga2tLQMGDKCqqorWQZcDE4GBgeTn55NWZ4WXtWt/Ze3aXwFY8MEHRJ47x5dffGHy/g6XAhh1M200GgvKL/3eu7q6MnzECCIiIkhISGDAgAEAhLQJYe/uPezft8/gem1D25qcZvTxRx8qGQ5CCCGEEEJcD/NmzZopGwkJCbQLDWWCiSdjX3/zrdFc49tZ3bnRtUaPHoOziwuLF31Zz1k1LiRcMNpnbmbG4088iYurK8XFxQS0bMk990ympLiYmGjDwm7r1q4lJiaGx594gv/83//xwcKFpNYZfHTq0oXMjAxcXF3Z9OefjBw1Shl4N23WzCiNvLFiYmIoyM+ndXCwQYAgJCSUqqoq/ANaGrRvGxpGVVUVR48dA2qevtva2jJn9mxlsBSRH0HE2QjmvPgiY8eNNwgQABQUFilTCqqrq02uOd82tC1PPvU0W7duVYIDAJ6enpiZmbFnz25lST69Xq8s1VdX7ZSCqqoqTp86xelTpygtLWPAoIH88cfvZGdnAxg8xS0uKeHkiZMc3H+Atu0up3ubmZkxeMgQjhw+bLBMX2VlJefOnlO277r7bpKTU3j7rbeoupSlcOLECX7/7Tcm3XUXjo6OZGZmGvX1r8jLzWXee+8CMP3hGUBNYMrRsRmT7rqLSXddHuivWrlSWQHg7nvuoXVQkMHSe43h4eFB8sXGrcqReOECMTHR7NqxUwkQaLVag6CKq5srIW1DKCgoYMMfK3n44Yfx8PAgKysLGxsbYqIvZxCsXPmj0UoLtR56aBqdunRm757d9Os/gAfvv7/Bvjk6OuLo6MiqH3/kxZdfZu2vv3Ls6FF69uqJja0db77+On369kVtZsay75bywNQHlZ9fcHAwZ06fMXndps2a4ejoyM+nTyuv18bW1mDaUW2WVm7O5QBBkyaWylSt1NRUdu3cwQ8rfmDkqFFKm/nvvYdFneybWnWXs6wr4XyC0T4hhBBCCCGuhbm9XU0qbVlZGVVV1cycaWqe8W527Nhxs/t2Q105N7p58+b4+fuzaePGRi9rWMvczIynnn1WebJfUVHBC7NnExMTwztvv2WQzl8r4swZ3nj1VWbPncvrb77Jqy+/RHp6BgADBw7k8JHDDB8+gsTEC5w5fYbmHi0ASElJYfasWeh0Oj78+CNef+014uMuz9luaKqBXq/n+PHjBAdffjqqUqkICg7iwL79dO/ZA3t7e2UQHRLShsioSIovPT3u3KkLcXFxBk9Sa+3ZtYvpM2bQwtOTxAvGAZT6BAQE8ND06Rw8eJAfV6wwOHb+/HlKS0sZO248SxYvUt6fxtr45wYG3XEH3bp15/fff6u3XUZmhsFSn21DQ9FqtWwPD6/3HCsrK/z9/Vm9epUSHKh1/NgxJt11Fz6+Pn97gKCiokL5fJZdSpNvHRSEXq+npKSE1atWk5aWwpgxY6/52lqttt4CmI0pjHnyxEnSUgw/G84uLgYBrQEDBjJgwEB2bN9OxJkzJCQkEBDQEmeXPPLy8pRAUENUKhWhYWGcjYgAVKhUqquek5mZyeznn6e4pITWwcE8/sQTfPvNN4S0DeWn1TWD7R3bt5OSkkJMdDQTJk2kV+/e7N+/H08vL6UewZU6duqIXl/N8eM12Rtjxo2jZWAgb9SZhuJ1aYpCYlKiss9CY0FZnWKvdbNXalVWVVFpImPryuUshRBCCCGE+LuY1y5xmJOTw+MzHzOay33x4kWWfP31rejbDfPV118bzIuva+iwYQwdNuyq11iz+ifWrv0VF1dXZs6cibePD7/+8gtjxo5l86ZNBLUJxsfbB52Dg8kAAUBaejrvvv02Pfv0Vga/7Tt0wKNFCz779FOGD69Jr/70k48pKSmhffsw5dzc3Fyio6Lp0aMX8XHxqNVqpj38MK1atWLu7Nn1FpI8duwovXr3xs/Pj9jYWAJbBWJlZcUvP68hrEN72ndoT/i2cMzNzQlu04Y1P/2knOvo2IxTp06bvG7qpYGdm6vbNQUIHp05k4KCAr5ZssSoeGFubi4L3p/PI48+xrz3FxAVFcmfG/5UpmtcTXp6BpVVVbjUKSbn7e3NxLsm4e7mjr1Oh5mZGQX5+UqRPgBnZ2cA0hoYrNZm3kyYMJEJEyaabGNrc/W6EH+HDh07EhkZibt7c+JjY4iNi6N//wFYWloaDexrt39c8YOyOgdA+PZwIs4aPiXv3KkLffr1ZdnSpQwfPpycvDyjJ9c5lzIzaqWlp6PVao36WDsfH+ClVy5nMZw6dZqwsDAKCws5efJkI19vB3Q6e3Zs307bOoX+rqZ2etTiL7/EycmR6Q8/TEVFBeF1AkG1qwccPHCQXr17Y21tQ0lJCUeOHDF5zfYdOhAXF6/Ur9geHs7QoUNxc3cnJTkZqPnMZaSnK4E2ACtLLZmZWY3uuxBCCCGEEDeDee2yW66urkqhtVrl5eUsWPiByartt7MF77+PmdnlAeHgwUMJaRvCV4sWGayPDjWpyQ9OfYgD+/ezc+flLIrU1JrBzsMzZuDl7cXy5cvYtnkLY8aOpbq6mnnvvMtLr7zCC7Nn89LcuQap7XWlpaezZnXNINzc3Jy7J0/m0MGDBvOV6ytQdvDgAcbfOYEjhw9x75QpODo68tXiRQ2uMnHm9BkqKysJa9+e2NhY2oWFER8XT1p6OsePHadrt+6EbwunbWgoGo1GeTJ6o3z7zTfcPXkyr7/5Jm//979Gy9adO3uOZ556iuEjRtCzZ0+eevppfl//W72p51eqrqpCRc0TZm9vb1594w3OnY3g66+XEBMVjbOrK+PGjaVdWHvlnNr21dVXX41gy+ZNHLs0BeNKSYmGqfkqlYrpM2agtbRk3bq1Bpkf10unc6BTl858vXgxE++62+BYeXm5Mh2hT9++eHl6sXTpdwCkJKcYtE1PSzN6el+7zOeWzZsZOGjQDXlyfejgASbfO5ny8nI++fijq7ZXqVSMGTOW9LQ0Tpw4cU0Bglp6vZ6vvlzEvAULKCoqQm9iFYs/fltP3359GTBwAFs3bzHKEgGwsbGhdatWfF8nCJOSnExKcjJ9+vZRlmv09vHhwhVBM621NSWJjQ+k1WVta2OyDklmZpYSlBg46A4GDByAo6Mj586dY+UPK/6npogJIYQQQogbwzw4KKjeg7/8+ispKYYDicDAQAL8/WnWrBn2Ons0FhZs2xauzFO/HdSudw41heZC2obw559/GszLr/XiSy9TVlbGsqVLTWYCrFm9GrXajNOnT2FuZqbsr6qq4pOPP+aeyffUGxy4kqubK06Ojsx7991Gtd8evp3Ro8cw96WXOLB/P/Pfe6/B5f2gJthw4vhxunbtyk+rV9OhYye2b60pDHfgwH6eevopbGxs6Nq1C4mJicqAA2oGIC5uriav6+HRHICU1BSTx+tz8sQJoqKimDN3Li/MmcP/vfmmUUBKr9fz2/r1/LZ+PePGj2fM2LEcO35MedpbH0dHRzQajZLm3617d1TAgvnvU1lZCcD5+HiioqINAgSZWTXZHC08veqd1pCVVfP0V6VWN3rQ7OnlRc+ePQGw19nz5utvNOq8hqjN1Bw9cpjdu3cbBQiqq6uVvrVpE4Kzs0uj+zp02DBcXF1Z9MUX17ws5bXIy8sjISEBBweHRvXtzjsn4Onl1ah+NW3alKqqKpO/f4OHDiE7OxsrKysee/xx3p83z+B4Wno6qSkpuLm789tv601ev0OnjlRX69mzew/OLi4EtmyJj68vVtbWdO7UmR9X/ICbmxstWrRgx47tBufaWluTn1dw1ddrip+vr8k6JNu2bePbr7+mfYcODBs2lKXffcf58+cZPHgITzz5FHNmz76hP0shhBBCCHH7M6/vQE5uLrt27aZTp44EBgYSGBiIn6+vybZardVtFSCopdFomDFjBiqVCldXV4YOHcrxEydISU5GpVIxbfo0WrVuxdLvvqt3mkBERES918/OyuKTjz5udH+SEpP46MMPSU9LM5mmfaXy8nL++P0PJt41ib179141OFBr27atzHphNoPuuAMnR0f27NkN1MydLykuoV///oS2CzNahvHgoQNMmnQXbUPbcvLE5XRwCwsLho0YSVp6+jVNL6iVkpzMf958kzf/8x+eeOopFsyfX+9AZtvWrYwZOxYvL6+rBgj69e8PwOHDNZXdVerazADjJ8Z1nTxxkrKyMu4YPJijR46Y7EtxcTExMdF0796D1T+ubNTqHinJyWRmZODo5ERTh4aX5mus7KwsfqhTu+G1N99U/j82Nva6rtm/f3/uvuceYmJi2L17d4Ntg1oHMXjYUD79+OPryjQKah2Et7c3arWakLYhDQYJhg4dysjRozh48OBV+wXg6eXNs889y0tz55KYeHn+f7t27ejTtx/vvP02Xl5eTLn/fvr378+2bZdXUBg+YgRu7u4A9OjR06iGhZW1NcOHj6Cqqor3Fy5UlsJMS0/n3LlzdOnSBT9fX7r16EllZSW7d13ur0ajwcramqwrpmc01skTJ5k/7716j3fo2JE/N/6pZP+sXPkjXbp2wcPDw+B9EEIIIYQQ4kr1BggcdLqrpvyWl5cTGRnJunWmn7D901lZadkWvg0fHx88Pb2UNe0TExMpyM8nKDiYLVu2sGXz5pvWp6P1zHU2RaPRcODgAXr06sUjjz7K22+9xfn4q6etnzp5iszMTO6ePJnTp04r0xmqq6vZu3cv48aPB2DPFYOwjX9soGuXrjz62ExWfL9CWeZw2IgRODZr1ujMB1MyMzL45KOPeGHOHCZMnMiqlStxcHCghWcLoqOiKSkpQaPRMHT4cACir1gZYvz48WRlZBJxNgJzMzP6DxrIiJEjCd8WzsWLNanV586eY8iQoTz2+Ey2bd5KVFQk1Xo9Dg4OBtcqLy/nxxU/cP+DDzDjkUdZuvQ7iouK0Ol0ePv4cvzYUQBWLF/Bi6+8zEuvvsqPP6wgNjaOkuJi/AMCqCgv5/wVhTDLy8t59plnuPfe+/D197vu90qlUtG3X18cdDr69O1LQsJ59u7Zy2effoyFhYXSLqVOwcCMjAxsLjRc5V6j0fDA1Kn07NmTqMhI3p8//6p9sbWzJSwsDLM62TOmzH//fYPt1JRUrKyseGTmY+zbuxdrayumPzyDObNnG8zVh5oioBMmTWLosGEcPHiQzz75pM7R+p+Iu7rU1J5Ir7OiQOvWrXl05kzWrP6J6KgooqOi6NK1C+PuvJPw8HD0ej2jRo/mzgkTOHTwEDk5Odw5cQIJCQmcPn05eFFeVoZOpyMzI4OjR48SFRlJbFwcBZcCiSXFxZhZWNC9R3eOHD5s8Jo8PT0ByEi/ekHG66LXY1anpkbNLj0qtbqeE4QQQgghhKhRb4DAlOzsbGJj44iJieFcZKTRIO12k5ubx+/ra54Muri60qdPH+4YPJgWLVoobTp27IiFmTmHDx/i1KlTV336fCM1sbSkiUbDyNGjaNMmhJYtW/LjDz8w7913eOM//2Huiy/yw4oV7Ni+/aqpxDvCtzN+wp3s2GFYqX/b1i0MHDSInTt2GNU+qKyq4j9vvsl9903hwakPYm4+HahJ03/nrbeJjDzHXxEREcHGP/9k+IgRHD1yFDc3V6bPmGHQpry8vCZ1+opASGFhIc/Pma1M8yguKmLtr7/yy88/K22OHjnC+nXrGDZ8OJ07d1H2V1dXG02l2bp1C4VFhYwbN44vvry87GV5eTmPTJ9OZVUVMTHR/PeNN5hw911GKd/bw8MNlkisZWdnR7fu3diwYYPRscYwNzPjkZkz6dy5M1u3biE2OoYp9z9A127d2bRxI5HnIk0+ya8b5NJoNLi6uXEhoSZg4OjkRK9evejVqxeOTk7s37eXxYsWU1FRoZxTWVGBta0NKpVK+WyZm5kRGhZGfn5+vXUyai1etIjc3MsZLlpLK1574w0KC4v4eskSrKyseHf+fF5++RXefvutS4UjVfTq3ZsxY8Yo/fr8s88NPts5Obmo1WruuXcyVRWGdQI6dulMYmIiZWVlmJuZMXL0aEaNHs22rVsNCjR++cWXWFlZYWlpydRp0+jSpQuR5yL5/NOaQIRfgD9PP/sMC+bPVzKGKisreeH550wu2wk1tTXmvvQillotv/9mmH3QoWNH9Ho9CQk3ZlnCw4cPc9+UKSQnJ5OQkMDgwUPQ6/UkSfaAEEIIIYS4igYDBBcvXiQyMoozERGcPXvWoHDe7axV61b4ePvQzNEJJydHmjdvjrOLi/Ileu++vezeuQtfPz9C27Wjbdu29OnXl8LCQg4dPMSunTuJibm5wRGtVkvbtm0xMzPjjjsGExkZyddLvubA/n2Ul5fz7ttvM3XaNKY+9BC2trasX7euweutXfsra9f+arQ/KekiU+69t97zysvLWbLkK5YtW4qnpxeZmRn1fi6efeaZeq+zI3w7O8K3G+3/YcUKJWU+Jiaa/fv3E+AfgJmFGVVV1cTHxZkcjG7cuJEjhw/j5+9HYWEhCecTTAZzVq9axZ9//om3t5fyeuJi4wwGw7UO7N/Pgf37adqsGa6uruTl5irZCLVi4+J4579vodFo8PbxQaPREBsTY7KPfn5+PDxjBmVlZWzauLHe96YhAS1b0rlzZ5YvW8qmjZsAOHXqFP0HDODee+/Fzd3NYEWG+uzft4/PPv0Ub29v3vjPm6hUaqKjoli8aBFnz541ah8XF0efvn35btkyo2O7du6s9z6lpaXExERz+tQpZQqMn68vzz73PImJibw/bx4VFRXk5eXx0cKFPPv880ybNo2FCxYwYMAApjzwAAUFBSxZ8pXJz8uunTvo2bMnQ4YMNdiv1+u5eDFZKSCoUqsJCg5i8+ZNrFj+vUHbzIwMXFxd+e9bb+Ho5MTu3btZ/t13VF4qTPjRwoU8/cyzvDBnDqtXrVIG/PUFBwA6de5MYGAr1qz+ifPnzzPz8cfp0rWrcnzf3n037O/psaNHcXJy5O577qGZoyOR5yL54IOFtzS4KYQQQgghbg+q8PBwk4+az52LJCEhgcKiy1Xl9dV6cvNyyc7KJis7m+zsbIquSAf+KxwcHBo9j/6vGD5yBCNHjSY/L4/srCzi4+OJjo7mXEREvXPJ/f396d6zJx07dmTDH3+w4Y8/jNqo1WpefvUVtmzZwt7de4yOv/jSy5w6fZL1axsevEPNU945L85l1cqVnDtb82R+wICBJCYlEh0VVW+GQIeOHeudN/+/yNHJiQULF7J40aIGB6r/BLPnzEGv17Poyy//0uAwoGVLoqOiTB6zsLDAy8sbrZWl0bHS0jLSUlON6mn07NmT+Ph4o+BHXeZmZrRsGYiZheFUAr2+puhn3cFn7Wf35zU/c/qU6ZoCo0aP5s8NG4yyHdq0CaGwsECZntGrVy/27dunFJX8K6ytrev9e6VSqbh78j0cOnDQZGaUWq1m3PjxbPjjj0b/zRs46A62btmMXq/H0dERbx9vdPY6YuNi613BYuCgQXTs1Il33nrL5PGHH3mEgvx8g7oTQgghhBBC/B0cHBxMBwhyc/NQq1XY2V19HfczERF8/vkXZF9nwa0rO3QzAgTif8ftFCCwsbExWsZRCCGEEEIIIf4JHBwcMHvggQdev/JAZWUlb7z5HzZv2UJycgoVFRU4NHUwKIBWy9nJicBWgYSHhxsdu1ZarZbS0tK/fB3x76JCxZmIM+T9w6fAXE+VfyGEEEIIIYS4GbRabf1TDHJyc3njjTcNKoD7+fnSunUQIW2C8ff3V5bi2759O4sWf/WXOyQZBEIIIYQQQgghxM1X7xSDWoWFhcx/fwFR9cx3dnFxobS0lLy8+ot1XWuHJEAghBBCCCGEEELcXA4ODqgB9u3bR2rq5TXTo6OjuXjxIjY2Nrz+2qv06d3b5AXS0tL+tuCAEEIIIYQQQgghbh01QHp6BkuXXl6+LCAggOXff8+5c5EAzJjxMJMmTrw1PRRCCCGEEEIIIcQNpwbIzcvl+IkTnKqzJNmDDzzAvPnz2bV7NwCjR49i6tQHb00vhRBCCCGEEEIIcUOpAfJya6YJfP31N0qldWdnZ+4cP57PP/+C5cu/B2DggAE8PvOxW9RVIYQQQgghhBBC3ChKBgFAWno6q1atVg4OHTqEli1b8seGDbz8yitkZWXRvXt35syefWt6K4QQQgghhBBCiBuiJkCQe7nQ4B8bNhATG6tsP/boI2g0GuLi4pkz90VOnDhJ27YhuDg73/zeCiGEEEIIIYQQ4oa4FCDINdj52WefG0w1mDDhTgCKiop49733WPjBB6Slp9/krgohhBBCCCGEEOJGUZeXl1NaWmqwMzU1lSVLvla22wQHGxw/dOjwTemcEEIIIYQQQgghbg513ekFde3avZu9e/cBYG5ufjP7JIQQQgghhBBCiJvMPC8vt96Dn3z6KWvXrSUzM+smdkkIIYQQQgghhBA3m3l9GQS1EhOTblJXhBBCCCGEEEIIcauof1279lb3QQghhBBCCCGEELeYOj4+/lb3QQghhBBCCCGEELeY+lZ3QAghhBBCCCGEELeeBAiEEEIIIYQQQgghAQIhhBBCCCGEEEJIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgEAIIYQQQgghhBBIgOCWGTFiOD179LjV3WhQ9+7dGDlyxK3uxj+Oo6MjPXv2oEmTJre6KwA4OTnRs2cPNBqNss/a2ppWrQIxMzO7hT0TQgghhBBC3E7Mb3UHbgV7e3sCA1vWezw3J5eo6Ogb2gedvT35+fk39B5/lZ2dHebm1/YRsdJqcXd3JyY29gb16tbTaDQGg/EboXnz5pSUlJCdnX1d/fH29sbFxZn8/AKSk5MbfV+tVktFRQWVlZXX3GchhBBCCCHE7e3fGSCwsyMsLIycnBzKy8qMjqtVklhxPbRaLaHt2qHVWv5PBwhuND9fX7y8vbiQcKFRAQJTEhISKCkpJi0trdHn2NnZ0bFjB06cOElWVtZ13VcIIYQQQghx+/pXBghq7d27l6Ski7e6G7c9MzMzPDw88PbyApXqVnfntmVvb4+fry/2Ovu/fK3CwkIKCwv/hl4JIYQQQggh/i3+1QGCv5tarcbR0RGNxoLqqmoyMjOpqKi4rmvZ2tpiaWlJdlYWVdXV9bazsLDA3t4OS0tLsrKyKSkpMTiuUqlwdHSksLDQ6NiVzM3NcXRsRlFRMQUFBY3ua8cOHbDUaomJiaFJEw3e3t6NPhdqAgzW1taYm5uj1z1PAFwAACAASURBVOvJzc1Fr9cbtdNoNGi1WvLz800eh8vTIqqqqhpsV/d6arWa/Px8qqqqDI6r1Wrs7GxRq80oKyujqKjI6LiDgwNQMyC/3p81gIuzM8FtgsnOyubA/gN07tL5uq/VELVajVarVeon5Ofny3QCIYQQQgghBCABgqsaOXIEDjodP65cRXl5ubK/f79+NG/uzspVqykvL8fLy4tBAwdgdsWc/bzcXI4dP0FkZGSj7telc2dat25FE0tLAPTV1aSkphIevt3gibCnpyfdu3XFXqczOP+bb76lvLwcc3Nz+vTpjbeXN+YWNX3Kz8tjz959XLhwweCc5s2b07NHD3QOl69VXFREVXX1VYMKADGxsRQUFFBeXo6Pj0+jXmetwMBAmjd3N9iXnJzMuXOX3y9HR0f8/f2xstICUF1dTWZmFhEREVTXCZ60aOFBQECAsl1drScnJ5uIiLMGg3dvb29aeHhgobFQ9hUVFnHg4EGgJsgQGNgSGxsbVHUyIq5Mve/UsaPy3gKUlZURFRVNRkbGNb0HADm5uX9rar+7uzutWgWyZ89eyi5No3FzcyMwMBC1+vJrKi4qZv+BAwbnhoa2Ndg+duw4OTk5f0u/hBBCCCGEEP9cEiC4ii1btnL3XZPo1asnW7duA8DLy4uAAH/WrlunBA0sLS0xMzfnl19+JT09HY1GQwsPD9q0CaZv3z6UlpaSkJDQ4L3ahoTQLqwdZ8+eJTIyioKCAry9vOjcuRPDhw1j5apVSlutVou9TseaNWvIzKwZVFpZWSn9uWPQINzd3Thy9CjRUdGYmZsTGtqWwXcM4te165RBrFarZciQwWRnZ7Nu3XoyMzPRarW4urjQtWuXRr1H1zuodXFxoXlzdyIjo0hPT6eiogIzMzO0Wq3SRqezJySkDXl5+URHR5Obm4tOpyOodWvatAnm5MlTStu0tHRSU9OU69jb29OmTRv8/f04e/ac0s7ayopqfTU7duxUsgYsLwVkzM3NaRcaSrW+mjNnzpCdnUNlZSV2dnYUFxcb9D8mNpbU1NSaa1pb4+frS1Dr1uzOzjbKRria8vLyGzrvv0mTJrRqFUhKSgoXLiRSXFyMSqXC3t7OqG1cXLxBAc1rySYRQgghhBBC3L7+1QGC7t27GxUp1Ov1rF23XtkuKSlh+/YdDLpjELGxcaSkpNCvbx+OHT9OaqpxAbjaNPTy8nJi4+KIi49n3NixdOvapcEAgZlaTfv2YURHRbNz5y5lf8TZs2TnZDN69GgCAvyJjo4xOK+kpFT5/9oBbIsWHrTwbMG2bdsM2u/evQdfHx/ahrRh67ZwANq0CcZMrSY8fDu5ubkAVFRUkJ+fj6+fr8Fg/e9mZ2tLdbWeixcv14GoqqoyyJTw8/OnuKiY48ePK9kCWVlZJFy4gL+/H5aWlpSW1rwHdTM8qqqqyM7OJj09jWbNmhndu7pabzCIr71GixYtMLcw5+DBQwb9MLXiRFZWltKngoICoqKj6dq1C66urgav6Z+gNhsiLTVN+ZzUTOfIM2pbUFBw3cURhRBCCCGEELevf3W5/qysLJJTUoz+u1JcfDzR0dH07dObAf37k5efz6FDhxt1D71ez5mIM9jrdDg6Gg9UazX38KCJpaXJ5RVTU9PIz8vD19e3Uff08vSiorzcKJhQXV3NxYvJNG3atE5bT9LT05XgwM2Uk5uLWq2iZcuWmJmZGR03NzfH3t6O1LQ0g6kEcDlrwc7O+Al4XaWlZde0JKGjYzPy8wuuq8BfcXExlZVVWFtbX/O5N1pBQQGVlVX4+Pjc0KCPEEIIIYQQ4vb1r84giIyMbPQqBrt27WbyPXfTwrMFv//+R4PF765UO/i2t9cp0wGuVDvQzcszfqILNQO8qw2Ga1lbW2Gh0TBjxsMmjxfVGfxaW9sY1SS4WTIzM4mOjsbX1w83Nzeys7OJi4tTsjBqC+n5+fni52c6OGJhcbmOgK2tLX6+vlhZW9GkSRNUKhUV5RUGdQSuxrKJJdl/Yb59RUUF5iaCHbdaeXk5p06eJCg4iG7dupKbm0diYuJ11UsQQgghhBBC/G/6VwcIroWvry8WFhbk5+XRpXNnkpKSGn1ubSzhWgaqf1VZaSlbtm41eayqsu78eD1qs1uXSJKYmERKSiq+vr44OzvRqVMnzpw5Q0ZGhvJ+JSVdJDMz0+T5tcEEW1tbOnbsQE52DufORZKXl4eVlRU+Pt44OjpeW6euIfhjys38OV+LnNxc9u7dh6enJ26uroSEtOFCwgViYmNvddeEEEIIIYQQ/wASIGgEnb09vXv1ZO++/aSkpDB+3Fjatg0xKJDX4PmXVhpoqNhb7Rx3B53OZDudTkdGPYPkKxUWFdHE0pL09AyDefmm5OXnm5yjfzNVVlYSFRVFTEwMbUNCaN26NVlZWUpdAJVKddU58a4uLuj1ek6eOmVQFyAvL/+aAgRl5WVYWVtd/4v5h9Pr9SQkJJCQkICPjzc+Pj5kZmWSm5unvG9q9b965pEQQgghhBD/WjISaIRevXuRlHTxUlX7bA4ePESnTp2Ugf/VtApsSXFREWlpl4sa6vWGA7GLSUmUl5XRvn2Y8fmtWmFtY0NcXFyj7ldbDDG0bdurtITY2FgcHBwIDAxs1LVvpOrqalLT0jA3r1nJoLKykry8fFxdXEzWKKir9pn/tUz9MCUjIwNbW9tG/2z/CWpf87VmLiQn19TbsLGxAVCWgmzSpPE1G4QQQgghhBD/O/7VAQLHZo54eDQ3+V/t3HYfH2/c3dzYt3+/ct6p06cpKy1l4ID+RtccOKA/tra2AGg0Gvr364ermxt79+03aJefn4+7uzturq4AVFVXs2PnLlxdXRk4cAAuLi5YWVkRHBREj+7dSUpKMio6WJ+kpIucP3+esLB2dOzQAZ1Oh0qlQqvV4uvjY9D29OkzpKak0Kd3L9q1a4dOp6NJkya4u7srS//dKDY2Nuh09srAVqPR4O7mRkV5BSUlJQDExMSgNlPTsUMHmjZtirl5TdKLTmevvM9QU7tBrVYTHBysvF6VSqXUMWismiUAS2gbEmJQzFGns7+u4oMWFhb079+PgICAaz63sWpXJfBs0aLeQEqTJk1q3r9Lx83MzPBs0QKAvLya7JWysjJKSkrxaO6BlVVNFoWdnZ3B+yyEEEIIIYT43/WvnmLQpWuXeo+tWbOGvLx8unXr9v/s3XlwI9l9J/hvIoHEfRIEeB8gq3gWWWd3V6u7qw/ZY2vUtmRLliw7ZiPs/cOe3fWuNDsRY8/84Yn9d8ceyZ5Z78xujDbW987OSJbH65Y0alVf1VVdJKuKdfAsnsUbIA7iRiL3DxSymARIgncd309ERxeTicyXiYck3u+993t4MDKiyfKfz+fx8Sef4Gd/9mdLlh4UdDp84xu/qv4cjUTw4x/9GJNbev8HBgbwhS/8PN5994v49//H/wlFUfDw4UP83d/9f7h8+RV86Uu/CADIZbN48OABPr1+fU/X9sMf/giXLl1Ed083Lly88OQXioL/+0//TLPU3Q/+9r/g0qWLONvfh5dffklznKNcrq+xoQG1dbWabel0GsObpglEIhEMDAyiva0NZ8/2a/ZdWFjAyMgogELP/8z0DJqam+DzVav7KIqCRDxRcZlkWcbNmzfRcfo0+vrOaEZ5TE1NY2pqak/XWAx+5HK5Pb1uL8LhMILBIBqbGpHP50vqGgC43W50d3dptslyHmNjY5opLSMPHqCzqwuvPP5sKIqCiYmJHafHEBERERHR80Goras/2JjsQ+R2u7F+gAzyJ6mjowNvvnkFf/qnfwadTgen04FoNKbmFijHbDbD5XJhsczSina7HSaTCaFgEPKWJf72yuFwwOFwIJ1K7ZjHQBAEuFwuWK0WKEphlYF0On2gc+/GYDCoPdT5vIxIJLrtNAFRFGG326DTiYhEIpBluWQfzfFkGdFYrGSJxErpdDrYbDaIoohoNFr2fLtxOp24cOE8hofvHvmKAS6XE8lkatv3TKfTweFwQKfTQVGUHa9JkiTYbDbEYjF16gERERERET2/3G439OfOnsXQrVsnXZbnSiwWq6jHNZlMqkPp93uMSkSj0R0DFUWKomB9ff1YgzTZbHbXBIRFsiwjHC6/DOR+jrebfD5f0X3bidPpRC4nH1qZdrLbvcnn85qRMDvJZDLHUmYiIiIiInp66P7pP/2f8Yd/+Afo6ztz0mUheu7Y7XY8ejS/r9EHREREREREx0l4//331fHcV69ehdVqRUNDA2pqahAMBrGysoqf/vSn+PCjj468MM/yFINqrxetgVYMDd3ikGwiIiIiIiJ6prjdbghffPcXlH/yrW/uuvPKygq+/Z0/2nOStr0W6FkNEBARERERERE9q9xuN4S//Ku/Uvw+n7pxdHQU848eIZVKwe/zobOzU10nHQC+850/2nNG/b0UiAECIiIiIiIiouPldruhLwYH/v7v38P3/+ZvEIloE51JkoS33noTX//a12A0GvE7v/M/IPy/hNXl5YiIiIiIiIjo2Sf8m3/7b5X/8B++u+t6936fD//iX/xzVFVVIR6P45vf+ifY2Ng41MJwBAERERERERHR8XO73RCHhm79/ubl9MxmMy5duoT5+XnNzvF4HLMzs3jjjdchSRISyQRGRw93FIHZbEYqlTrUYxIRERERERHRzsxmM0S73fH7ACAIAt54/XV861vfxJU33sDFixcwNzePYDCovmB1dRWBtgBqa2rQ2NCAv//796AoynbH31eBGCAgIiIiIiIiOl5qgKC9rQ3f+uY38TM/83mYTSYAgMvlwptvXkFroBXT0zMojjJIp9N49fJlmEwmXLv2KTaPPjiMAjFAQERERERERHS8zGYz9L/9W7+F119/Td04OfkQV69exS/+4i+gqqoK58+dw/lz5/CTn7yP/+c//kcsLi6q+9rt9pMoNxEREREREREdMn0xOLC2toa//Ku/xieffAIAuPrBB/jCF34eX/7SlyBJEt5++y289trn8N4Pf6i+WBBOpMxEREREREREdMh0m3/I5/Pqv7PZLL7//b/B7/yP/xN+/F//K4DCkofvfvGL6j6hYOiYiklERERERERER0l0uly/39XZCYvFgpdffgmdXZ2Ynp5CNBoFUMg5MDR0Czc+u4Eavx9+v1998X/5u79DMpk8tMIwBwERERERERHR8TObzRBn5+Z///2f/hRWmxUtzc3wVVfjZz7/eTicDoyPjyObzQIAotEoPvroY8zOzeLc2bPQ6/Xo7urChx99BFmWD61ADBAQERERERERHS91FYNkMomBgQHcHBjA5995BwDQFgjgnbffRiaTweTkpPqihYUF3BwYwJtXrqCqqgpmsxm3b98+tAIxQEBERERERER0vNQAQXFDJBLBF77w8zAYDAAKOQfO9vfj1Vcv49GjBayurgIAYrEYkqkkzvb3o729DTc+u6FOSThogRggICIiIiIiIjpeZrNZm6QQAJTH///zv/hLfPDhhwCA2tpa/PPf+128++6TBIXvvfdDxONxAMDrr79+9KUlIiIi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NjTAX1ODeHwD2UwWDocDDqcTggDcu3ePwQEiIiI6dAwQED0j8hU0BhQODa3YRjy+p+kCiqIgFAody3JmT5vVtVXEGADBejgMayYDk9kMyWBAIp7A0tIyFhYW1Ma6LMuYnHx4KEPbYxsxGNeNkCQJDocDmWwWc3OzePRo4bkdrUJEREQni1MMiIiIiIiIiF5wbrebyxwSERERERERERggICIiIiIiIiIGCIiIiIiIiIgIDBAQERERERERERggICIiIiIiIiIwQEBEREREREREYICAiIiIiIiIiMAAARERERERERGBAQIiIiIiIiIiAgMERERERERERAQGCIiIiIiIiIgIDBAQERERERERERggICIiIiIiIiIwQEBEREREREREYICAiIiIiIiIiMAAARERERERERGBAQIiIiIiIiIiAgMERERERERERAQGCIiIiIiIiIgIDBAQERERERERERggICIiIiIiIiIwQEBEREREREREYICAiIiIiIiIiMAAARERERERERGBAQIiIiIiIiIiAgMERERERERERAQGCIiIiIiIiIgIDBAQERERERERERggICIiIiIiIiIwQEBEREREREREYICAiIiIiIiIiMAAARERERERERGBAQIiIiIiIiIiAgMERERERERERAQGCJ4qgUArXnrp0kkXo6wLFy6gLRA46WI8lSRJwuXLl1FTU3PSRdmXc2fPoqOj40jPcZB7VF1djUCg9QhKRUREREREm+lPugAnwWQyoaWlGXa7A4IAyDkZ8UQcsdgGFhYWoCjKiZTLbDJDFJ/Ot8RqsSAaiRzKsURRRHNzExKJJJaWlkp+bzQaUV9fB4fDgUQiidWVFayHw4dy7qMgiiLMZhMEQTjpouyLxWpBMpU80nMc5B6dPn0KRqMRi4tLSCaPtpxERERERC+yp7M1esR81dW4cuUK1tfXkUmnodOJsNltMJvNSMTjuH79BsbGx0+6mM8lnU6H/v5+uFxOrK2tlQQIrFYrLl64ADkvY21tDVaLBQ3nz2FiYhKzs7MnVGo6SWNj4zAZjQwOEBEREREdsRcyQFD0ySefYH7+kfqzyWTCK6+8jLfffgupdJoN0iPQ09MDq8WCZDJV9vcdp09Dzsv49NPryOVyAIDm5ma0t7dhfX0dsVjsOItLT4HV1dWTLgIRERER0QuBOQg2SaVSuHr1AyQSCXR1audkS5KEhoZ6NDQ0wGw273gcQRBgt9tRU+OH0+ncczn0ej08Hg9cLidEUdx1f1EU4XQ6od9hX7PZ/PiYroqOaTAY4HI6odcfXgypoaEeVVVVuDM8jEwmU7aMLrcLs7NzanAAAObm5pDLyaivr9v1HHpRhMvlgsfjgcPhgE63fRUXRREejwcejweSJO16bJ1OB5fLCY/HA6vVuuv+h3VOh8MBu90Og8Gw476CIMDhcFR03L0wGAwV18XDuEeVEkURDocDHo8Hbrd72+kLxc9HJfdFkiT1fj+rU0aIiIiIiPbrhR5BUI6iKIhEInC5XJrtX//61zSBgWg0ips3b2J8fELdZjab8XM/9w9Q7fVC2NQwzWYyuHNnGDcHBnY9v9FoxGuvfU7TsI3FYrh37z4SiYRmX7PZjK6uLjidDrUxE4lEcf/+fc1wbKvVipdffklzjRsbGxgZGS3pkW9ubkZjY4OmMbWxsQFdBY3DnZjNZrS1tWNqagqRbXIZuN1uAEAoFNJsz+fziEYjcDld5V6mam9rQ2NTo6ZhV7zWsbHxkvNevvyK5jqTyRSmpqZKpj04HA50dJyGzWbTHPv27TsIBoNlyyKKIs6fPwdR1GNgYADZbHZP57Tb7ejoOF3SUE2n05ibm9eMbhFFEZ0dHfBWV0MUdepxx8fHsba2tuM924nf50OgrQ1ms0ndlognMDY+XvIeHeY92qqnuxsOpxPXrl1Tt3V0dJQEjBYWFjAyMqr+bLPZ0NXVqSlTJpPB3Nw8ZmZmNK/t7e2Fx+OBXv+kniuKgrW1IO7du4d8Pl+2bEREREREzxMGCLbQ6XRwOp0ljZof/OBvsb6+DgBwOp24cP483nrzTSwtLauNbFEU4fP5cOPGDQwN3YIgCHA6Hejs6MSFixewsLiIhYWFHc+fl2UMDw9jPRxWe0i7ujrR2dmJwcFBdT+9Xo/z588BAEZGRrG6ugqz2YyOjtM4d+4srl+/AVmWAQCJRAKfXvsUiWQSOp0OVqsV3V1d6Onpwaeffqoe0+v1oq0tgKXFJczNzyORSMBsNqOqqurAWeS7OjsRDq+XNMw2s9lsUBQF8Xi85HfxjTgcjp1HYxiNRmQyGVy/fgO5XA4mkwlerxdNTY3o7+/H9evXkU6n1f2Hhm6p57KYzWhpaUFXVycikYgaYNHr9Tjb34+8kse9e/cQCq0jl8s9TqCYKFsOg8GA/v5+ZDIZ3L07pL4PlZ5TFEX09/dBURSMj09gdXUV6XQaBoMBr7z8MiwW7QiWM2d64XK5MT09jaWlJegEAU3NTejt7cHg4BCi0eiO960cu92Ont4ehMMRjI+PIxwOw+Nxo7m5GX19fbh+/fqR3aPd+P1+1NfXYXR0DCsrK8hms4+TID65LzqdDmfP9iOfz2NkZLSQz8JqRUNDA9raAkilUlheXtZcbyQSxvDwXeTzeRiNRrjdbnR1daK1tQWTkw/3fA+JiIiIiJ41nGKwiV6vx5tXrsBssWBkdEzzu2JwAAAikQg+uXYNgk6HxsaGkuPENwoNQEVREA5H8On160gmEuju7tq1DNlcTs3YL8sy1tfXMTs7C5fLCcumBlBLczMkScLg4BAWFxeRy+UQi8UwOjoGk8kEv9+v7qsoChKPG3P5fB6xWAzTMzOwWMyaYzY1NSGZTGFktDCyQJZlbGxsYGZmBnKu8gbcVn6fD3aHQ9O7W45Br0dezpddRSKTzUKvF3ed8pDPK+r0hFQqhfn5eQwMFAIrra0tmn03ByISySQmJichCAI8Ho+6vbGxEXqDHrdu3cbKyqp67Gg0qpkGUWQymXDx4kUoioI7d4ZLGr6VntNgMODWrduYn59XgxrZbBZ5RduTXZyuMDIygunpaaRSKSSSSYyNjUPOyWhsKK2flQgEWpFMpnDr1i2sra0hl8thZWUVg4NDkHM5tLQ0H9k92o3Dbkc+r+DRo0fqqINiXS1qaGgo3MOhW1hcXEQ2m0U4HMbdu3exHlpHoMySndlMVh0pkE6nsbS0hNXVNdTW1u6pfEREREREz6oXegTBq6++isymHmWH0wmT0YjBgUFMTk7u+NpUKoVcLger1VbRuVZX10qmLVQqGi2MULDZbWpDv6qqCuuh9ZLM7rFYDJlMBjbbzuVKpQpJAo0mExLJJERRhMvlxMz0zL6GU9fU1GimRSQScYTDEQiCgEBbGxYePdL03pclCNueu7jdYDCUbXTuJJ1OI7i2Bl+1b8cgRSaTgSznYTIZ1W1ebxWi0Zim8bkdg8GA/r4+yHIOt2/frmi5zHLnrK72IhqNlR1JsZXX60UuJ5dMUcjn81gPh9U8ADabDQ6HQ7PP8vJy2ca5XhRRVVVVti7IcmF1ierqajzAyOMyHO092mo9HEZjUyNOnz6NycnJstfg9/sQjcbUz8tmi0tL6O7ugs1m27XMsVgMPl81RFHccyCDiIiIiOhZ80IHCILBoDo9wOPxwG8247333sP0tHYYvCAIuHjhAloDrbBZrTBIErKZDPSiiErzmKXTabg97n2Vs9hLqtc/SVInSRKsNivefvutsq/ZnNBOkiS0t7XB6XJBkiSIok5NElicm20yFeaZl2tQVaKjo0Od/w4AiwuLCIcjaGiohyRJmKlgRQhZzkHcZoRAceRAueSGlUgkk/Ab9DAYDMhmsxAEAa2tLaj2VsNkMkHUi5BzMnQ6AQKevKkmowmhTaNHdhIItEKn0+HGjc/KBjH2cs7t5u1vZTQaodeL29aDYlDG6/WWTBMJBoNlG73Gx3UhmSq/0kQylYLeoIdery9M5TjEe1SJtbU1jI+PIxBoQ21tLUKhEB4+fKgJqJiMppI8CUWpVKGOWyzmXQMExc+ewWBggICIiIiInnsvdIBgdHRUXeZQp9Phl3/py7jyxhtYXPxrTW/35995B42NDRgYGMTM7AySyRTq6mrx+XfeqfhcB0lytt1rw+EIpqeny/6uWH5RFHHx4gXk8wqmpqYQCoUgiiJqavxobS3NK7BT1v+dXL16tez2mpoaKPk8zpw5o9lutVoBxYILFy6o5Uql0tDpBEiSVBIIMJtMkHPyvhtpxZ7qYkCkp6cHVR4Ppqansba2hkwmA4/HU34aSIW93LOzs/B6vTh/7hzuDN9BOKxNiljpOfNKXpPkcje5bA53790r+7v84/s1PT29bV05FId0jyo1NzePxcUlBAIB+HzVuHTpEu7du3foSyJurTdERERERM8z5iB4LJ/P44c/+jEkScIbr7+ubtfpdGhrC2D47j3cvlNo0KTTaUxNTZ9oj2I6k4ZBr0coFCr7X7E31eVywWQyYXR0FEtLS8hkMkgmkwgGtb2rqVQKiqIc+tJ0S0vLWFhYQCQc1vwnyzlksllEwmE1GFBMplduachCErn9NSaBQkJAWc4jk8lAp9PB56vG/KPCagCJROLxHPsV5PPahm46k4bFaqnoHMlkCoMDg4jFYug706e5l3s5ZyKeKJkOsJ10Og29QY9oNFq2HoT3cc/SxbpgKX/dVosFuWxOHQFwWPdor3K5HMbGxvDJJ9cQXl9HV1eXGuBKpVMwb1N+i6VwzkRif6NliIiIiIieVwwQbBKJRDA4OIRAWwCBrb3r+5grfZRCwRCsNuuueQ0q7fkszC0Poq6uDpZtGlb7MTc3h4nJyZL/Uqk0Eok4JiYn1WHe4XAY2UxWk2ARKMyft9qsWNqUdX4vJEmC1+tVh+0Xe4Urmf++uroKu91ecf6InCzj9p07iCcS6O/vU6d67OWcjxYWYDab0NXVCXHT8pJ2ux16UTvop7iMYVNTY0XlK0dRFM0Uh5wsIxQKoa6uTrMkIwBYLBZU+6qxuvakp/6w7tF+5fN5LC0vQ69/spLB8vIKHA67JvkjUAjUNDc3I5lMVZQzgYiIiIjoRcIAwRaDQ0MIr4fx0kuXIDxOmre6uobu7m50dnaojWeLxQJBOLnbNz09jWQyhTO9vaitrVVzCFitVrjdT3IdhMNh5PN5tLe3wefzqQ3OzWvbF42PjyOXy+Ls2X7U1NRAkiRYLJZCI+sYhljn83nMzs3B56tGS0sLJEmCy+VEd1cXkslURcPHzWYTTp8+vek6zTh7th8QBDx8nHhSURTEYjHU19VrGsGSJJUEVGZn55BIJNF35oymselyObft/c7n8xgeHoZOp0NfX9+ez7mysoLZmVnU1NTgc5/7HC5cuIBXXn4ZFy9eKJl6EAqFsLa2hubmZrS2tsBqtUIQCtM0fL7qXe8XUOjVd7ndcG0auTE+PgEA6O/vg9frhV6vh89Xjb6+PmSzWUxMPEnieVj3CADa29tx5Y03UFcJ97jZAAAgAElEQVRXt215bTYbXC6net8kSUJdbS2ymayatHNubg6xWAy9PT2ora2FwWCAy+VCX98ZmExGjIyMVHRviIiIiIheJC90DoJyFEXBhx9+iHd/4V30nTmD23fu4Cfvv4+f/ZnP48qVK5p9s5kM4vHy67wftZwsY2BgAKfa29HR0QGd7kkjMxFP4NPr1wv75XK4f/8BOjs60NvbozlGOp3WJIpLpVK4ceMznDp1Cl1dnSUN19wxTKmYmZmBZDCgpaUFgUArFEVBJBzB0K1bFU3pyGVz8Hg8aGioV7eFwxEMDg5qEjDev/8AZ3p70dnZAaBD3S7nZKTST5LzybKMmzdvouP0afT1ndHkaJiamsbU1FTZcmQyGTx4MIL+/j40NzdjZmam4nMCwMTkJB4tLKCmxg9RJyIny1hZWUF/f3/JYJbh4bsItLaivr5ek1eisMzmJ7smdpyemkJffz/OnT+Hn/70amFZzEQCNz77DB0dhesuHm9tLYjR0VE1ed9h36Ni8satIxc2j7xobGhAbZ126cF0Oo3h4WE1X4eiKBgcHMKpU4XPR1dXJ4DCqgRDQ0P7zn1ARERERPQ8E2rr6p+asfNutxvrFWZDP26CIKC6uhqSVBgOHY3G1DnzJ00QBFitVkiShGQyWbL0IVBIVmi329XGWywW0zTyttLpdLDb7RBFEfm8jFhs41hzLhTPv7FR+Xl7urvhcDpx7dq1wnB8vR6xWGzbbPmCIMDhcKijDba7d5vLZLPZIIoiotHovu7HXs+59bVvvPEGpqamMLvNqhAWiwUmkwnZbFZdoaMSxdEi4XC45HcGgwFWq6WiOnDQe6TX6+FwONSRLwDQ29sLs9mEzz67qSmT3W4HAOTzMiKR6LbTN0RRhM1mQzKZ3PcqGEREREREzzu3280RBJVSFAUrKysnXYyyFEXZdT61LMtlG3/byefzB0oKeFAHPX8ljWNFUfZ0jnw+f+Cg0F7PWSQIAlpaWiCKOkSj278+kUggkdj7qJZMJrNt4zmbzVbc437Qe5TL5TTLE4qiCIfDjkhEe8xsNrvtMoZbybJ8onWZiIiIiOhZwQAB0VPGbreju6sLiWQSyUQCFqsFLqcLeoMes7NzL8Tw+KqqKtT4/XC5XZAkCY8ezZ90kYiIiIiInnsMENBzY3VtFbHnIDN9NpvFWnANen1hGL2cl7G4tISVlZUXpidclmXoRBHLyytYXl7e03QJIiIiIiLaH+YgICIiIiIiInrBud1uLnNIRERERERERGCAgIiIiIiIiIgYICAiIiIiIiIiMEBARERERERERGCAgIiIiIiIiIjAAAERERERERERgQECIiIiIiIiIgIDBEREREREREQEBgiIiIiIiIiICAwQEBEREREREREYICAiIiIiIiIiMEBARERERERERGCAgIiIiIiIiIjAAAERERERERERgQECIiIiIiIiIgIDBEREREREREQEBgiIiIiIiIiICAwQEBEREREREREYICAiIiIiIiIiMEBARERERERERGCAgIiIiIiIiIjAAAERERERERERgQECIiIiIiIiIgIDBEREREREREQEBgiIiIiIiIiICAwQEBEREREREREYICAiIiIiIiIiMEBARERERERERGCAgIiIiIiIiIjAAAERERERERERgQECIiIiIiIiIgIDBEREREREREQEBgiIiIiIiIiICAwQ7Kq3pwfvvvvFivZtbGzAP/7t34bFYtnXuV69fBlf+5Vf2XU/j8eDpsbGfZ2DiIiIiIiIqBz9SRfguHm9Xlit1h330el0iITDCK2vw2q1QjIYKjr2V375K/iH//ALGJ8Yx49+9OM9l83pdEKSpF33a29vg9lsxvLKCtLp9J7PQ0RERERERLTVCxcgsFgscDoc6s+SJMFqsyIajULOyQAAURSRTqeA9b0d+//9T/8Jy8vL+PjjTw6zyCUmJiZhMhoZHCAiIiIiIqJD88IFCGZnZzG76efq6mp0d3dhbGwc8Xj8wMf+89nZ3Xc8oFAodOTnICIiIiIiohfLCxcg2C9JklBT40cuJ2NlZQW5XG5Pr7darfB6vTAajZienkYmkzmyckqSBEVRkEgkoCjKjvsbjUYYjUZkMhmk0+ld9yciIiIiIqLnEwMEFbDb7fjN3/gN2O02GI0mmExGLC4t4S/+/C8Q3NSb//bbb+P3fvef4Ve+9nW1l/+rX/0KvvCFL6CxoQEAkEylkIjHMXz3Lv7kf/sTrAWDO567pqYGfWfO4O7du1hYXAQAnD59Cg67AzcHBtT9Ojs74fG4YdiULyEv57EWXMPo6Bjy+by6XRAEtLQ0o8ZfA8n4JOeBks8jGo3h7r17ew6AEBERERER0bONAYIKZHM5/P33vofl5WUYDAbU1dXi61/7Or7xjW/gj/74j3d8bX1dPUSdDr/4pS8jHo/DZDKhpaUFv/mbv4Ff//Vfx7/+9re3fW1LczO6urtw/fqNXacVOOx2hMNhjIyMIp/Pw2g0wuFwoLOjA/GGBGY3TX2or69DU1MTFhcXsbK8gkg0CkVR0NTYiNZAKwRB2NsNIiIiIiIiomcelzmsQCqZxNLSEhRFQSaTwfT0DN5//31093TD6/Wq+203PD+dTqv5DVKpFEZGRvCDH/wtenp6UOXxlH1Nb08PXnnlZdy8OVBxzoFsNquOFEin01hdXcXK6ipqa2vUfXQ6HRobG7G0tIyxsXGEIxG13Jns0Ux7ICIiIiIioqcfAwT7NDc/DwCor69Xt+2l531mZgYAUFNbW/K7rs5OXLx4EZ9+eh2rq6sHKufGxgZMJhNEUQTwZCnF5aWlAx2XiIiIiIiIni8MEOxTIpEAAJjNpn29PhaLPX69WbPd5XLizbfexPT0NKYfBxEOIpvNAgD0jwMEJlOhvMlU6sDHJiIiIiIioucHAwT7JMsyAEDA/ubrF1+/VSQSxccff4yWlha8dOnSvstXtHXag6IUpiAwzwARERERERFtxiSFh+gwlghUFAV37gwjFFrH5cuvIJlMYvju3UMoXUE8Xhj54HA4kOIoAiIiIiIiInqMIwgO0WH2ys/Pz+Ojjz5Ca6AVzc1Nh3bcWCyGWCyGQGsrnE6nul3U6WCz2Q/tPERERERERPRs4QiCp9ji4hIePBhBZ0cnIpEowuHwoRx3ZGQU3V1dOHu2H9FoFHJOht3B4AAREREREdGLjCMInnLj4+MIrYfQ0XEaou5w3q5EIoGBwUHcu3cfodA6ItEoRkZGMTExCeBwpkoQERERERHRs0Worat/alqDbrcb6+vrJ12MF1ZjQwMamxrxySfXTrooREREREREdIzcbjdHEFCB0WhETU0NopHoSReFiIiIiIiITgBzELyATp1qh9lsxkZsAwDw/7N33/FNlfsfwD/nZLTpSPekQAfQ0tKWLQgyFGSrbFDAgeAWHIhXvXrBexWv4+LPwRJFUdwTN6goKKCyZwe0ZbWle48k5/z+SHPaNOmkaQN83q8XpOecJ+c8SdMkz/f5Ps/j5eUFT70nDNUGpKSmdnDtiIiIiIiIqCMwQHAZKiwsgsFghIuLCzQaDcrKy5CVnY3z2dkwSVJHV4+IiIiIiIg6AAMEl6GcnBzk5OR0dDWIiIiIiIjIiXAOAiIiIiIiIiJigICIiIiIiIiIGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQERERERERERggICIiIiIiIiIwAABEREREREREYEBAiIiIiIiIiICAwREREREREREBAYIiIiIiIiIiAgMEBARERERERERGCAgIiIiIiIiIjBAQEREREREREQA1B1dAbowgiBAFESIKhGiKEIUBAiCAAgCBAgdXT0iIiIiIrrMyZABWYYsy5BkGZIkQTJJkGQJsix3dPWoDgYILkKiKEAU1dCoVRBFFUwmE0ySCSaTCcaaPzj+oRERERERkbMQBAGiKEKo6dDUaDRQqVSQJBMMRhMkyQhJYhumozFAcBFRiSpoNBqIogij0YhqgwEmU2VHV4uIiIiIiKhRsizDZDIp2wYYAAAqlQoqUQWtiw6SJMFgMMAkmRo6DTkYAwQXAZWoglarASDAaDKisopBASIiIiIiuviZTOZM6GpDNTQaDbRaLQAZ1dUMFHQEBgicmCgK0Gi0EAURBqMBRqOxo6tERERERETkEAaDAQaDAWq1GlqtFpIswWCo5tCDdsQAgZNSqzTQumhgMBhQZajq6OoQERERERG1C6PRCKPRCI1GA1dXHaqrDDCaDB1drcsCAwROyEWrhSiqUFVVZTVOh4iIiIiI6HJhMBggSRK0Gi1UKgFV1dUdXaVLntjRFaBagiDA1cUFgICKygoGB4iIiIiI6LJmMplQUVkBwNxWEgQu5e5IDBA4CUEQ4eriApMkoaqaQwqIiIiIiIgsqqqrYJKkmiABm7GOwmfWCZgzB7Qw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)

所以我现在在插件该怎么使用这个workflow，我可以直接用自然语言对话吗，然后codex插件自己识别我的意图，走对应的workflow

8:35 AM

可以直接用自然语言对话，但要分清两层：

1. 自然语言理解
2. workflow 执行/推进

你现在这套代码已经把第 2 层接好了，但第 1 层还不是“自动拥有”的，它取决于插件宿主怎么接。

更直白一点说：

- 你可以在 Codex 插件里直接说人话  
    比如“帮我改这个 API”“检查这次改动下一步该走什么流程”
- 但插件要想“自己识别意图并走对应 workflow”，还需要宿主在收到这句话后，调用我们现在做好的 runtime SDK/bridge
- 也就是说，现在已经有了“可执行的大脑后半段”，插件只差把自然语言请求接到这套 runtime 上

你现在在插件里最推荐的接法是：

- 用户自然语言发起请求
- 插件先结合 IDE context 推断当前关注文件/changed paths
- 宿主调用 @enterprise-rag/agent-runtime-host
- 先跑 resolveWorkflow(...)
- 如果返回
    - should_continue === true
    - ui_intent === "auto_execute_recommended"  
        再自动跑 driveAgentWorkflow(...) 或 advanceWorkflow(...)
- 否则把结果展示给用户
    - 比如缺 task contract
    - 比如需要人工 review
    - 比如应该先 preflight

现在可直接用的入口就是：

- SDK: sdk/agent-runtime-host/index.mjs (line 1)
- 类型: sdk/agent-runtime-host/index.d.mts (line 1)
- bridge: scripts/agent_runtime_bridge.py (line 1)

如果你问的是“当前这个 Codex 插件，不额外接宿主逻辑，能不能光靠聊天就自动走这套 workflow”：

- 不能完全指望它天然做到
- 现在这套代码提供的是一个稳定的 workflow runtime
- 还需要插件宿主把“自然语言意图 -> changed paths/task context -> runtime 调用”这一步接起来

所以结论是：

- 可以自然语言对话
- 可以让插件自动推进 workflow
- 但前提是插件宿主接入我们这套 SDK/bridge
- 不是只把代码放仓库里，Codex 插件就会自动学会调用

如果你愿意，我下一步可以直接给你写一版“插件宿主最小伪代码”：  
输入是用户自然语言 + IDE 当前文件，输出是自动调用 resolveWorkflow/driveAgentWorkflow 的完整流程。

8:35 AM

那你之前指的插件和codex的插件有什么区别

8:38 AM

我前面说的“插件”，更接近“你准备接入这套 runtime 的宿主/扩展层”，不一定特指你截图里这个官方 Codex IDE 插件本身。

区别可以这样看：

**1. Codex 插件**  
就是你现在在 IDE 里聊天的这个现成产品。  
它负责：

- 聊天
- 读 IDE 上下文
- 展示执行过程
- 帮你改代码、跑命令

但它不会因为你仓库里多了 agent_runtime_bridge 和 SDK，就自动知道：

- 什么时候该调 resolveWorkflow
- 怎么把 changed paths/task contract 映射进去
- 什么时候自动 advanceWorkflow

也就是说，官方 Codex 插件是“通用 agent 宿主”，不是你仓库专属的 workflow runtime consumer。

**2. 我前面说的“插件”**  
更像是“接你这套 runtime 的定制宿主层”或者“扩展层”。  
它可以是：

- 一个你自己写的 IDE extension
- 一个包在 Codex/VS Code 外面的 sidecar
- 一个在现有插件宿主里加的 adapter 层
- 一个内部工具面板

它的职责是把：

- 自然语言请求
- IDE 当前文件/changed paths
- task contract
- 你仓库的 workflow runtime

真正连起来。

所以我前面说“插件里 agent 可以自己识别并走 workflow”，说的不是“Codex 官方插件天然已经支持这个”，而是“如果有一层接入你现在这套 SDK/bridge 的宿主逻辑，就能做到”。

**一句话区别**

- Codex 插件：通用聊天/执行产品
- 我说的插件：你这套 runtime 的定制接入层

所以你刚才那个疑惑是对的。更准确的说法应该是：

不是“Codex 插件已经会用这套 workflow”，  
而是“你现在已经把这套 workflow 做成了一个 Codex 插件或其他宿主都能接的 runtime”。

如果你愿意，我下一条可以直接把这两个概念画成一张很短的“分层图”，你一眼就能看出：  
Codex IDE 插件 / 你的接入层 / runtime SDK / bridge / repo workflow 各自在哪一层。

8:38 AM

可以

8:39 AM

可以，把它看成 5 层最清楚：

**1. Codex IDE 插件**  
你现在在用的聊天界面。  
负责：

- 接收自然语言
- 读取 IDE context
- 展示执行过程
- 让 agent 改代码、跑命令

它是通用宿主，不懂你仓库私有 workflow 语义。

**2. 你的接入层**  
这是我前面说的“插件”更准确的意思。  
负责把：

- 用户自然语言
- 当前文件 / changed paths
- task contract
- runtime SDK

串起来。

这一层可以是：

- Codex 插件未来支持的一段定制逻辑
- 你自己写的 IDE extension
- 一个 sidecar/service adapter

**3. Runtime SDK**  
就是现在这层：

- sdk/agent-runtime-host/index.mjs (line 1)
- sdk/agent-runtime-host/index.d.mts (line 1)

负责提供稳定调用面：

- resolveWorkflow
- advanceWorkflow
- driveAgentWorkflow

**4. Local Bridge**  
就是：

- scripts/agent_runtime_bridge.py (line 1)

负责把外部调用转成仓库内结构化 capability。

**5. Repo Workflow 真源**  
就是你仓库里真正决定规则的地方：

- backend/app/services/agent_runtime_service.py (line 1)
- scripts/agent_harness.py (line 1)
- scripts/agent_flow.py (line 1)
- AGENTS.md (line 1)

它们才真的定义：

- 怎么 inspect
- 什么时候 preflight
- 什么时候 flow
- 什么时候该停下来 review

**一句话图**

Codex IDE 插件  
→ 你的接入层  
→ Runtime SDK  
→ Local Bridge  
→ Repo Workflow 真源

所以区别就是：

- Codex 插件 是通用聊天宿主
- 你的接入层 是把这个宿主接到你仓库 workflow 的那一层
- 现在我们已经把下面三层基本搭好了
- 还差的是把 Codex 插件上的自然语言入口，正式接到这套 runtime 上

如果你愿意，我下一步可以直接给你写一版“从用户发一句自然语言，到宿主如何调用 driveAgentWorkflow”的时序图。

8:39 AM

可以。时序图写完然后把 Codex 插件上的自然语言入口，正式接到这套 runtime 上

8:41 AM

Worked for 15s

已经按你说的“轻正式化”接好了，现在它是一个仓库内的正式 workspace 包，但还没有进入外部发布流程。

改动在 package.json (line 1)，根 workspace 已挂上 sdk/*；SDK 自身还是 private，包名固定为 sdk/agent-runtime-host/package.json (line 1) 里的 @enterprise-rag/agent-runtime-host。仓库根其他 Node 宿主现在可以稳定直接写：

`import { driveAgentWorkflow } from "@enterprise-rag/agent-runtime-host";`

我还补了根级消费验证 scripts/agent_runtime_workspace_import.test.mjs (line 1)，并把 README/接入文档同步成 workspace 用法：sdk/agent-runtime-host/README.md (line 45)、docs/agent-runtime-plugin-reference.md (line 35)。

验证是新鲜跑过的：

- npm install
- node --test scripts/agent_runtime_workspace_import.test.mjs -> 1 passed
- node --test sdk/agent-runtime-host/index.test.mjs -> 7 passed
- frontend/node_modules/.bin/tsc -p sdk/agent-runtime-host/tsconfig.json -> 通过
- git diff --check -- ... -> 通过

有一个非阻塞提示：npm install 输出了现有依赖里的 4 high severity vulnerabilities，我这轮没有展开处理它们。

8:41 AM

你的意思是必须发布到npm才能给codex接上是吗

8:44 AM

不是，完全不是必须发到 npm。

给 Codex 接上这套 runtime，核心需要的是“Codex 所在的宿主/接入层能调用这套 SDK 或 bridge”，而不是“这个包一定要公开发布到 npm”。

你现在其实有 3 种接法：

- 直接调本地 bridge  
    用 scripts/agent_runtime_bridge.py (line 1) 的 HTTP 接口  
    这根本不需要 npm 发布
    
- 在同一个仓库里直接 import workspace 包  
    用 @enterprise-rag/agent-runtime-host  
    这也不需要发 npm，只要 workspace 装好就行
    
- 如果 Codex 的接入层在另一个仓库/另一个项目里  
    才会开始考虑：
    
    - 用相对源码拷过去
    - git submodule / subtree
    - 私有 registry
    - 公网 npm

所以“发到 npm”只是“跨仓库、跨项目复用时的一种分发方式”，不是“接上 Codex 的前提条件”。

更准确地说：

- Codex 接 runtime 的必要条件是：有一个宿主层能调用 SDK 或 bridge
- 发布到 npm 只是：让另一个项目更容易安装这个 SDK

如果你现在说的还是“当前这个仓库里的 Codex 使用场景”，那答案就是：

- 不需要发 npm
- 现在这套 workspace 包 + bridge 就够接

如果你愿意，我下一条可以直接给你画一个非常具体的判断表：  
“同仓库接 Codex / 跨仓库接 Codex / 给别人复用”分别该用哪种接法。

8:44 AM

我就是要在这个仓库的codex接runtime现在实现了吗

8:45 AM

严格说，**还没有完全接上**。

现在已经实现的是下面这几层：

- runtime 真源已经有了  
    backend/app/services/agent_runtime_service.py (line 1)
- 本地 bridge 已经有了  
    scripts/agent_runtime_bridge.py (line 1)
- 宿主 SDK 已经有了  
    sdk/agent-runtime-host/index.mjs (line 1)

但**还没实现**的是这一层：

- “这个仓库里的 Codex 插件，在你发自然语言后，自动把 IDE context / changed paths / task contract 收集起来，再调用 resolveWorkflow 或 driveAgentWorkflow”

也就是说，当前状态是：

- runtime 已经做好
- Codex 插件自动接 runtime 这一步，还没真正落到插件宿主里

所以答案分两句最准确：

1. **底层已经实现了，可以被接**
2. **Codex 插件当前并没有被正式改造成自动走这套 runtime**

如果你现在问的是“我在这个仓库里和 Codex 聊天，它会不会已经自动走这套 workflow”  
答案是：**不会自动**。

如果你问的是“这仓库里有没有足够的代码让它被接上”  
答案是：**有，已经基本齐了**。

现在缺的就是最后那一层“Codex 宿主接线”。  
如果你愿意，我下一步可以直接给你做一个**仓库内可执行的接线方案**，明确到：

- Codex 输入什么
- 宿主怎么提取 changedPaths
- 什么时候调 resolveWorkflow
- 什么时候调 driveAgentWorkflow
- 什么情况停下来等你输入

这样我们就不是在讨论概念，而是在补“最后一跳”。

8:45 AM

直接补

8:46 AM

Worked for 2m 48s

已经直接补上了，但要说准确一点: 我接的是“这个仓库里的 Codex 仓库侧入口”，不是去改官方 Codex 插件二进制本体。

现在这条链已经有正式入口了：

- Codex 技能入口: .codex/skills/runtime-workflow/SKILL.md (line 1)
- 自然语言宿主入口: scripts/codex_runtime_nl_entry.mjs (line 1)
- SDK 新增自然语言分流: sdk/agent-runtime-host/index.mjs (line 61)
- 时序图和接法说明: docs/agent-runtime-plugin-reference.md (line 17)

现在仓库内可用的方式是：

`node scripts/codex_runtime_nl_entry.mjs --prompt "下一步该走哪个 workflow" node scripts/codex_runtime_nl_entry.mjs --prompt "直接补" --paths backend/app/api/v1/endpoints/chat.py`

或者宿主直接：

`import { runWorkflowFromPrompt } from "@enterprise-rag/agent-runtime-host";`

它会先做很薄的一层自然语言判断，只决定两件事：

- resolve
- drive

真正的 lane / preflight / flow 还是交给 runtime，所以不会越权乱跑。我还顺手修了一个关键问题：宿主不传 changedPaths 时，现在会正确回退到 runtime 的 repo 改动检测，不再错误传空数组。

验证都跑过了：

- node --test sdk/agent-runtime-host/index.test.mjs -> 12 passed
- node --test scripts/agent_runtime_workspace_import.test.mjs -> 1 passed
- frontend/node_modules/.bin/tsc -p sdk/agent-runtime-host/tsconfig.json -> 通过
- node --check scripts/codex_runtime_nl_entry.mjs
- 真实 smoke: 起 python scripts/agent_runtime_bridge.py --host 127.0.0.1 --port 8022 后，
    - --prompt "下一步该走哪个 workflow" 返回 workflow.resolve
    - --prompt "直接补" 在当前 API 例子里仍返回 needs_input / collect_task_contract，这是 runtime 正常阻断，不是入口失效

所以现在如果你的 Codex 会读仓库里的 .codex/skills，这条“自然语言 -> runtime”已经在仓库侧正式接上了。还没做的只有一件事：如果你想让 IDE 里的官方 Codex UI 无感自动触发这个 skill，而不是靠显式调用/仓库 skill 匹配，那就得再看官方宿主是否支持更强的 repo-level auto hook。



我记得codex本来就会自动调用skill，这个skill应该也会吧

8:58 AM

对，**原则上会**。

Codex 本来就会对**匹配到的本地 skill**做自动调用；你这个 runtime-workflow (line 1) 也是同类 skill，**不是必须手敲命令才能用**。

但有个关键区别：

- investigate 这类 skill 已经是明显高频、强路由词命中的技能，所以你截图里它很容易被自动触发
- runtime-workflow 也能自动触发，但前提是你的自然语言更明确地像“让 runtime 来判定/推进 workflow”

也就是说，它不是“不会自动调”，而是**匹配信号要更像它的描述**。

更容易命中它的说法：

- 下一步该走哪个 workflow
- 用 runtime 帮我判断下一步
- 直接补，让 runtime 自动推进
- 不要手动 inspect/preflight，走 runtime workflow

不太容易命中它、反而会被别的 skill 抢走的说法：

- 帮我修这个 bug
- 看看这个报错
- 分析一下为什么不行

这些通常会先落到 investigate、plan、review 之类。

所以结论是：

- **会自动调**
- **你显式点名它时一定更稳**
- **不点名时也可能自动触发，但取决于 prompt 是否像“runtime workflow 路由/推进”而不是别的技能**

如果你想把它调得更“稳触发”，下一步最有效的不是再加底层代码，而是把


















这几个触发口令写进 skill 描述和参考文档里，让你自己和团队都用同一套说法。