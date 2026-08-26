# Pi Agent Tutor

> 一个面向中文学习者的 Pi Agent 源码教学 Skill：不是把文档一次性塞给你，而是按课程、源码路径和理解程度，带你真正读懂 Agent 的运行逻辑。

## 这是什么？

`pi-agent-tutor` 是一个围绕 [earendil-works/pi](https://github.com/earendil-works/pi) 构建的源码陪练 Skill。

它把 Pi 的 Agent 架构拆成一套可持续学习的课程，并提供：

- 中文优先的源码讲解
- 从运行逻辑到代码实现的分层教学
- 课程地图与多条学习路线
- 分课练习与理解检查
- 术语表，降低源码里的英文门槛
- 学习进度卡，方便中断后继续
- 从 Pi 设计迁移到自己的 Agent / Agent 工作台

它尤其适合：

- 想系统理解 Coding Agent（编程智能体）底层架构的人
- 看得懂一点代码，但容易被大型源码仓库绕晕的人
- 想研究 Agent Loop（智能体循环）、工具调用、Session（会话持久化）、Compaction（上下文压缩）、Provider（模型提供商）、Extension（扩展）和 Skill（技能）机制的人
- 想参考 Pi 的设计做自己 Agent 工作台的人

## 它会怎么教？

这个 Skill 不以“把答案讲完”为目标，而是维护学习状态并循环进行：

1. 先用人话建立直觉
2. 再讲运行时的数据流和状态变化
3. 对应到具体类型、函数和文件
4. 解释为什么这样设计，以及它避免了什么问题
5. 用一个小例子帮助迁移理解
6. 用一道判断题或推理题检查是否真的懂了

如果你只说“继续”，它会尽量沿着当前学习进度往下讲，而不是每次从头开始。

## 课程内容

当前课程共 14 个阶段：

| 课次 | 主题 |
| --- | --- |
| 0 | 包结构与依赖方向 |
| 1 | 类型系统：`AgentMessage` / `AgentTool` / `AgentEvent` |
| 2 | Agent 状态机与双队列 |
| 3 | `agentLoop` 主循环 |
| 4 | 工具执行流水线 |
| 5 | 内置工具 |
| 6 | Session 持久化 |
| 7 | Compaction 上下文压缩 |
| 8 | `pi-ai` 模型层 |
| 9 | 扩展系统 |
| 10 | Skills 与 System Prompt（系统提示词） |
| 11 | `AgentSession` |
| 12 | Durable Harness（可持久运行框架） |
| 13 | 总复习与架构迁移 |

完整先修关系和不同目标的推荐路线见 [`references/curriculum-map.md`](references/curriculum-map.md)。

## 仓库结构

```text
pi-agent-tutor/
├─ SKILL.md                     # Skill 的核心教学规则
├─ agents/
│  └─ openai.yaml               # OpenAI 产品界面配置
├─ assets/
│  └─ icon.svg                  # Skill 图标
└─ references/
   ├─ curriculum-map.md         # 课程地图
   ├─ exercise-bank.md          # 分课练习库
   ├─ glossary.md               # 术语表
   └─ pi-course-source.md       # 创建 Skill 时使用的课程源码快照
```

## 使用方式

这是一个标准的文件型 Skill。将整个 `pi-agent-tutor` 目录导入或放入你所使用产品支持的 Skill 目录中即可。

导入后可以直接尝试：

```text
从头教我 Pi Agent，我对源码不熟。
```

```text
继续讲第三课。
```

```text
解释 prepareArguments 到 schema 校验之间发生了什么。
```

```text
我想参考 Pi 做一个自己的 Agent 工作台，给我安排学习路线。
```

不同产品的 Skill 导入位置和方式可能变化，请以对应产品当前版本的说明为准。

## 关于课程快照

`references/pi-course-source.md` 是创建本 Skill 时使用的源码课程快照，用来提供稳定的教学上下文。

需要注意：

- Pi 仍在持续迭代，快照中的具体实现可能与最新 `main` 分支不同。
- 快照中的历史行号只用于定位当时的课程内容，不代表当前源码行号。
- 当用户提供当前 Pi 仓库，或明确询问“最新版现在怎么实现”时，Skill 应优先检查当前源码，而不是把快照当成最新版事实。

## 与 Pi 官方项目的关系

本项目是社区制作的独立教学 Skill，**不是 Pi 官方项目，也不代表 Earendil Works**。

课程内容基于公开的 `earendil-works/pi` 项目进行学习、整理和解释。Pi 项目采用 MIT License（MIT 开源许可）。相关上游版权与许可信息见 [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md)。

## License

本仓库中由本项目贡献者原创的内容采用 [MIT License](LICENSE) 发布。

上游项目、引用代码与第三方内容仍遵循各自原有许可证与版权声明。
