# Pi 教学术语表

按“中文概念 → 英文原词 → 在 Pi 中的落点”解释。只在相关课程中读取，不要整表灌给学习者。

| 中文概念 | 英文 | 在 Pi 中的意思 |
| --- | --- | --- |
| 智能体消息 | AgentMessage | 会话记录使用的可扩展消息，范围可能大于模型真正看到的消息。 |
| 模型消息 | Message | 经过转换后发送给大模型的 user、assistant、toolResult 等标准消息。 |
| 声明合并 | declaration merging | TypeScript 允许应用扩展接口，使 AgentMessage 自动加入自定义消息类型。 |
| 流函数 | StreamFn | Agent 调用模型的函数边界；通过函数注入与具体模型提供方解耦。 |
| 控制反转 | inversion of control | 核心只规定函数契约，外层把具体实现传进来。 |
| 单飞行 | single-flight | 同一 Agent 同时只允许一个主运行，插话必须进入队列。 |
| 引导消息 | steering message | 当前运行期间排入队列，在一次工具批结束后、下一次模型调用前注入。 |
| 追问消息 | follow-up message | Agent 原本要停止时才注入，用来续跑。 |
| 判别联合 | discriminated union | 用共同的 `type` 或 `kind` 字段区分不同结构，并让类型自动收窄。 |
| 参数垫片 | prepareArguments shim | 在结构校验前修复旧格式或模型常见格式错误，不等同于校验。 |
| 结构校验 | schema validation | 检查参数是否符合工具声明的形状与类型。 |
| 预检 | preflight | 真正执行前完成工具查找、参数处理、校验和拦截。 |
| 收尾 | finalize | 执行后应用 afterToolCall 等结果覆盖，再生成最终工具结果。 |
| 中止信号 | AbortSignal | 把取消状态传给模型请求、工具和监听器。 |
| 记录稿 | transcript | Agent 持有的完整会话记录，可能包含模型不可见的自定义消息。 |
| 事件溯源 | event sourcing | 保存一串不可变变化记录，通过按序重放重建状态。 |
| 变更原子 | mutation | JSONL 中一行对应的一次最小状态变化，如追加条目或移动 lane。 |
| 会话分支指针 | lane | 指向 Entry 树某个叶节点的命名指针，移动它即可切换分支。 |
| 预写提交 | write-ahead | 先把变化写入持久化介质，再更新内存状态。 |
| 单写队列 | single-writer queue | 把并发写操作排成一条 Promise 链，避免文件内容交错。 |
| 撕裂尾行 | torn tail | 进程在追加最后一行时崩溃，留下不完整 JSON；只安全修复尾部语法损坏。 |
| 原子发布 | atomic publish | 先写临时文件，再用 rename 一次替换正式文件。 |
| 上下文压缩 | compaction | 把较早消息总结成摘要，同时保留近期尾部和必要结构。 |
| 保留尾部 | retained tail | 压缩后原样保留的近期消息，后续压缩时仍参与切点计算。 |
| 合法切点 | valid cut point | 不会拆散工具调用与工具结果等结构关系的压缩边界。 |
| 真值锚点 | ground-truth anchor | 用最近一次模型提供方报告的真实用量作基准，再估算新增消息。 |
| 模型提供方 | provider | 拥有模型目录、鉴权语义和流式调用实现的运行时单元。 |
| 代际失效 | generation invalidation | 新一轮刷新开始后，旧刷新结果不得覆盖新结果。 |
| 虚拟模块 | virtual module | 在单文件二进制中把扩展的 import 名称映射到已打包模块。 |
| 渐进式披露 | progressive disclosure | 系统提示只放 Skill 索引，匹配任务后再读取完整指令。 |
| 持久程序计数器 | durable program counter | 把操作当前状态完整写入寄存器，恢复时直接从该状态继续判断。 |
| 副作用夹心 | effect sandwich | 外部操作前先记意图，执行后再记结果，使崩溃窗口可识别。 |
| 重放策略 | replay policy | 决定中断后的外部操作可安全重跑，还是只能生成“已中断”结果。 |
| 不变量 | invariant | 系统在所有合法状态中必须始终成立的规则。 |
