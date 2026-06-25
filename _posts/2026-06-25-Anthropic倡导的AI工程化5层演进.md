---
layout: post
title: Anthropic 倡导的 AI 工程化五层演进
date: 2026-06-25 11:00:00
categories: 
- AI
tags:
- AI
---  

随着大语言模型（LLM）从最初的“对话助手”演变为如今能够自主编写代码、执行复杂逻辑的 **AI Agent（智能体）**，开发者与 AI 交互的模式正在发生一场深刻的范式转移。

在这一进程中，Anthropic 团队（特别是 Claude Code 团队负责人 Boris Cherny 及其行业探索者）总结并推广了 AI 工程化的几个核心阶段。而在业界实践中，AI 工程化逐步收敛为五个层层递进的学科：**提示词工程（Prompt Engineering）**、**上下文工程（Context Engineering）**、**沙箱/脚手架工程（Harness Engineering）**、**循环工程（Loop Engineering）** 和 **评估工程（Evaluation Engineering）**。

本文将为您深入拆解这五个层级，介绍它们各自解决的核心问题，以及如何通过这些工程化实践构建真正稳定、自主的 AI 系统。

---

## 🚀 AI 工程化的演进鸟瞰

AI 系统的开发在过去不到两年的时间里，经历了从“手工作坊”到“自动化流水线”的转变。这五个层次并非孤立存在，而是层层递进、环环相扣，最终形成开发与评估的闭环：

```text
[1. Prompt Engineering] (关注: 表达与指令)
         │
         ▼
[2. Context Engineering] (关注: 召回与过滤)
         │
         ▼
[3. Harness Engineering] (关注: 安全与边界)
         │
         ▼
[4. Loop Engineering] (关注: 自动化与闭环)
         │
         ▼
[5. Evaluation Engineering] (关注: 评测与质量保障)
         │
         └───────── 反馈与优化 ─────────► (回到第 1 步循环迭代)
```

下面我们逐一剖析每个层级所解决的痛点及核心工程实践。

---

## 1. 提示词工程 (Prompt Engineering)

*   **核心问题**：“我该如何表达和撰写这条指令？” (*"How should I phrase this?"*)
*   **关注对象**：优化单次查询（Single Query）的词汇结构与模板，以提升单次响应的质量。

### 解决的问题与背景
在 LLM 发展早期，模型对输入的指令极为敏感。同样的诉求，如果换一种表达方式、加上角色扮演（Persona）或提供几个示例（Few-Shot），模型输出的质量就会天差地别。
提示词工程解决的是**模型“理解意图”的不确定性**，通过制定明确的结构、约束与示例，引导模型给出结构化、符合预期的回答。

### 核心实践
*   **角色定义 (Persona)**：如 “你是一位拥有 10 年经验的鸿蒙技术专家...”。
*   **少样本提示 (Few-Shot Prompting)**：在输入中塞入多组“输入-输出”对，训练模型进行模式匹配。
*   **链式思考 (CoT)**：引导模型“一步步思考 (Think step-by-step)”，提高逻辑推理的准确率。

### 局限性
提示词工程是静态的、手动的。当任务复杂度上升，需要执行多步骤的复杂工作流时，仅仅依靠一段精雕细琢的提示词，模型依然容易出现幻觉，且维护成本呈指数级上升。

---

## 2. 上下文工程 (Context Engineering)

*   **核心问题**：“模型在做决策时，需要看到哪些核心信息？” (*"What information does the model need to see?"*)
*   **关注对象**：在有限且昂贵的 Token 窗口中，精细化管理、检索和筛选输入模型的数据。

### 解决的问题与背景
随着模型支持的上下文窗口越来越大（例如从 4K 提升到 200K 甚至百万级别），“大海捞针 (Needle in a Haystack)” 的问题开始凸显。如果把整个代码库或几百页文档直接塞给模型：
1.  **注意力分散**：模型容易忽略中间的关键细节，产生幻觉或偏移主题。
2.  **成本与延迟攀升**：Token 消耗巨大，首字延迟（TTFT）显著增加。

上下文工程解决的是**信息的精准供给与 Token 预算控制**。

### 核心实践
*   **检索增强生成 (RAG) 优化**：使用语义搜索（Embedding）和混合检索，仅将最相关的代码片段或文档召回。
*   **动态上下文装配**：根据当前任务的类型，动态构建系统提示词（System Prompt）、历史会话（Memory）、运行中的工具输出（Tool Outputs）等。
*   **上下文修剪 (Pruning)**：压缩多余的代码行，甚至只提供函数签名而非完整实现，在不失真前提下降低 Token 体积。

---

## 3. 脚手架/沙箱工程 (Harness Engineering)

*   **核心问题**：“整个系统如何在无人值守的情况下安全可靠地运行？” (*"How does the whole system operate reliably?"*)
*   **关注对象**：设计包围在自主 Agent 周围的执行环境、护栏和检测机制。

### 解决的问题与背景
当 AI 被允许使用工具（如修改代码、执行终端命令、访问数据库）时，它不再仅仅是一个“打字机”，而是一个具备现实破坏力的“执行体”。如果 Agent 陷入死循环、不小心执行了危险命令、或者陷入重复错误的死结，系统将遭受重大损失。

Harness Engineering 的核心思想是：**不能只靠提示词约束模型“不要做什么”，而要靠代码和物理环境强行限制模型“不能做什么”。** 它是将 Agent 作为一个不可信的组件，嵌入到一个高度可控、可验证的安全网中。

### 核心实践
*   **隔离沙箱 (Sandboxing)**：Agent 生成的代码或运行的指令必须在 Docker 容器等隔离的沙箱环境中执行，防止危害宿主机。
*   **权限审计 (Permissions & Gates)**：拦截高危命令，并由“人机协同 (Human-in-the-loop)”进行审批确认。
*   **防死循环与资源上限**：对 Agent 执行的单次任务限制最大时间、最大 Token 消耗以及最大 API 调用次数。
*   **断言与容错恢复 (Assertions & Recovery)**：在 Harness 中加入测试套件。如果 Agent 的修改导致编译失败或 Lint 报错，Harness 会捕捉到该错误，并作为反馈输入（Harness Feedback）回传给 Agent，让其自己调整。

---

## 4. 循环工程 (Loop Engineering)

*   **核心问题**：“如何将 Agent 的决策、行动、验证和纠错周期完全程序化？” (*"How can I automate the agent's decision-making cycle?"*)
*   **关注对象**：编写一个外层的控制流程序，作为驱动 Agent 自主循环运转的“发条”。

### 解决的问题与背景
在传统的 AI 开发中，人是循环的一部分（Human-in-the-loop）：人输入指令，AI 输出，人看了一眼觉得不对，再手动敲提示词让 AI 修正。
**Loop Engineering** 旨在将这种手动的“提示-观察-再提示”模式升级为由代码驱动的“自主闭环（Autonomous Loop）”。

正如 Anthropic 团队负责 Claude Code 的 **Boris Cherny** 所言：
> *"I don't prompt Claude anymore. I have loops running that prompt Claude and figuring out what to do. My job is to write loops."*
> （我不再直接提示 Claude 了。我写循环代码去提示它并分析它下一步该做什么。我的工作就是写循环。）

### 核心实践
*   **递归目标分解**：外层 Loop 接收到一个宏观目标（例如“重构整个登录模块”），Loop 负责维护一个全局任务队列，并分配给子 Agent 去逐步完成。
*   **主动观察与决策循环 (ReAct / ODAF Loop)**：
    1.  **观察 (Observe)**：Loop 检测到代码库中存在一个编译错误。
    2.  **决策 (Decide)**：Loop 封装这个状态，调用 LLM 决策下一步动作（是查文件，还是修改代码）。
    3.  **行动 (Act)**：Agent 调用 Harness 提供的工具（如 grep、replace_file）修改代码。
    4.  **反馈 (Feedback)**：Harness 运行编译器，将结果返回给 Loop。若未通过，Loop 触发下一次迭代，直到任务完成或触发熔断。
*   **状态持久化与检查点 (Checkpoints & Persistence)**：在漫长的闭环执行中，Loop 会自动保存 Agent 的中间状态（比如 git stash 或状态机），如果某条路径尝试失败，Loop 可以自动回滚并让 Agent 尝试其他方案。

---

## 5. 评估工程 (Evaluation Engineering)

*   **核心问题**：“我们如何持续、客观地验证 Agent 系统的质量与安全性？” (*"How do we continuously measure and validate agent quality?"*)
*   **关注对象**：设计基准数据集（Benchmarks）、定义多维度指标、构建评估流水线与轨迹（Trajectory）分析工具。

### 解决的问题与背景
在传统软件开发中，我们依靠断言语句来编写单元测试（输入 $A$ 必须得到 $B$）。然而，Agent 系统输出是**非确定性（Probabilistic）**的，且生成代码或执行动作的中间路径（例如用 grep 还是 view_file）极其多样。传统的回归测试无法满足复杂 Agent 评估的需求。

评估工程解决的是**从“拍脑袋凭感觉（Vibe Check）”到“科学量化（Metrics-Driven）”的飞跃**。只有当系统的每一个微小修改（如调优了一个 Prompt 或优化了检索策略）都可以被精确评估时，复杂的 Agent 系统才具备生产落地的可能。

### 核心实践
*   **LLM-as-a-judge（大模型裁判）**：编写特定的评估模型和 Prompt 指引，充当裁判对 Agent 最终交付的业务代码或回答的语义准确性、安全性进行自动化打分。
*   **运行轨迹评估 (Trajectory Evaluation)**：不仅看最终结果是否正确，还要量化评估 Agent 在 Loop 期间执行了多少次不必要的工具调用（效率）、消耗了多少 Token（成本），以及是否有高危行为（安全）。
*   **自动化回归评估流**：在 CI/CD 流程中自动执行测试数据集，当系统检测到优化某处 Prompt 反而降低了另一场景的准确率（能力退化）时，及时阻断代码发布。

---

## ⚖️ 五个层级的对比

| 工程层级 | 关注焦点 | 解决的典型问题 | 核心产出物 | 范式转变 |
| :--- | :--- | :--- | :--- | :--- |
| **Prompt Engineering** | 指令语法与表达 | “模型听不懂我的要求” | System Prompt, 模版, Few-shot | 怎么问模型 |
| **Context Engineering** | 信息筛选与组装 | “模型漏掉了关键信息” | RAG 检索器, Context 组装逻辑 | 模型看什么 |
| **Harness Engineering** | 环境限制与验证 | “Agent 把系统搞崩溃了” | 沙箱环境, Lint 检查器, 权限审批流 | Agent 能干什么 |
| **Loop Engineering** | 自主流程与自愈 | “每次出错都需要我手动重新提示” | 任务分发器, 状态机, 自动纠错循环 | 怎么自动执行 |
| **Evaluation Engineering** | 科学评测与质量 | “我不知道修改 Prompt 后系统变好还是变差” | 测试集, LLM 裁判, 轨迹评估分析器 | 怎么量化质量 |

---

## 💡 总结：AI 工程师的角色质变

从这五层演进中我们可以看到，AI 工程师的职责正在从 **“提示词撰写者（Prompt Writer）”** 迅速退化，并同时向 **“系统架构师（System Architect）”** 进化。

1.  **精雕细琢的提示词越来越不重要**：随着基座模型理解能力的提升，提示词工程的边际效用正在递减。
2.  **工程化的软件系统越来越重要**：开发出色的 AI 应用，核心在于设计一套强大的 **Harness（护栏）** 来隔离风险，编写稳健的 **Loop（循环）** 来驱动自愈和递归执行，以及构建客观的 **Evaluation（评估）** 体系来确保整个迭代不会失控。

未来的软件工程，将是人类工程师设计**“产生提示词的软件系统（Harness & Loop）”** 并配套 **“自动化质检流水线（Evaluation）”**，再由该系统驱动 AI 自主完成具体业务代码的编写。
