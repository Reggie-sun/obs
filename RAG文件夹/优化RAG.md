
请为当前项目补最小主链路烟测，优先覆盖：  
  
1. 上传文档 -> 入库任务创建  
2. 检索接口返回结果结构稳定  
3. 问答接口返回 answer + citations 等主骨架字段稳定  
4. 至少补一个权限相关回归测试，验证跨文档/跨部门访问限制  
5. 测试重点是守住主契约，不追求全量覆盖



请对当前 backend/app/core/config.py 做温和拆分重构：

1. 按 app / model / retrieval / storage / auth 等维度拆分 settings
2. 保持 get_settings() 对外调用方式尽量兼容
3. 不改变现有配置语义
4. 不做激进重构，只做可维护性收口
5. 不影响现有测试通过




请不要直接重写当前鉴权系统，而是先基于现状输出一个 auth 演进方案文档，并补最小回归测试：

1. 说明当前 auth_service 的边界和适用范围
2. 指出多实例、重启后 revoke 丢失、自定义 token 非标准化等限制
3. 给出 V1.1/V2 的演进路线
4. 如有必要，只补最小代码注释和测试，不做大改


请为当前仓库补统一启动/测试入口：

1. 不改变现有 frontend 和 backend 自己的脚本
2. 在根目录增加聚合脚本（Makefile 或 package.json scripts 都可以）
3. 至少提供 dev / test / build 的统一入口
4. 更新 README 中的命令说明


请整理当前项目里的可观测性基础设施：

1. 统一关键日志里的 request_id / trace_id 语义
2. 梳理 request_trace_dir / request_snapshot_dir / event_log_dir / rerank_canary_dir 的职责
3. 只做最小增量整理，不做大规模 logging 重构
4. 补文档说明这些诊断产物分别解决什么问题


请对当前仓库做一次代码卫生整理：  
  
1. 删除明显未使用代码和重复逻辑  
2. 缩短过长函数，但不要做大规模架构重写  
3. 优化注释密度，保留真正有价值的解释  
4. 不修改主契约字段和响应骨架  
5. 输出一份“本次整理了哪些技术债”的总结