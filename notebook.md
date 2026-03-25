# 系统框架

● 根据我对项目代码的检索，从存储、搜索、分块、来源四个方面来解析 openclaw-mini 的实现：                                                         
                                                                                                                                                
  ## 1. 存储 (Storage)                                                                                                                             

  使用 JSONL (JSON Lines) 格式进行会话持久化，核心设计在 src/session.ts：                                                                          
  双写策略：                                                                                                                                    
  - 内存缓存：快速读取，避免重复 IO
  - 磁盘持久化：.openclaw-mini/sessions/{sessionKey}.jsonl

  JSONL 格式：
  - 每行一条 entry，支持追加写入 O(1)
  - 损坏行自动跳过，容错性好
  - 包含 header (会话元数据) + message/compaction entries

  关键实现：
  ```ts
  // 追加消息时双写
  async append(sessionKey: string, message: Message): Promise<void> {
    const state = await this.ensureState(sessionKey);
    // 1. 先更新内存缓存
    state.entries.push(entry);
    // 2. 再追加写入磁盘
    await this.persistEntry(state, entry);
  }
  ```

  使用json格式进行记忆持久化，存储位置在.mini-agent\memory\index.json

  ---
  ## 2. 搜索 (Search)
  
  长期记忆搜索在 src/memory.ts，采用 BM25 风格关键词评分：

  评分算法：
  - 词频 (TF)：统计查询词在文档中的出现次数
  - 饱和函数：tf / (tf + k1) 防止高频词过度加权
  - 文档长度惩罚：短文档匹配权重更高
  - 查询覆盖率：匹配了多少查询词

  无时间衰减（对齐 openclaw 设计）：
  ```ts
  function computeKeywordScore(content: string, queryTerms: string[]): number {
    // 纯关键词相关性，不考虑时间因素
    return (coverage * totalTf) / lengthPenalty;
  }
  ```

  搜索流程：
  1. 提取查询词（字母数字 token，去重）
  2. 遍历所有记忆条目计算得分
  3. 按得分降序截断返回

  ---
  ## 3. 分块 (Chunking)

  上下文管理

  Token 估算（底层支持）：context/tokens.ts 提供了简单的字符数/4 估算，是分块和裁剪的基础。
  
  裁剪 (Pruning) 是另一个核心机制，与分块/压缩不同：

  // 三层递进策略 (pruning.ts)
  // Layer 1: Soft Trim - 工具结果保留 head + tail
  // Layer 2: Hard Clear - 用占位符替换内容
  // Layer 3: Message Drop - 丢弃整条旧消息，保护最近 N 条

  区别：
  - 裁剪：直接删除/截断内容
  - 分块：将内容压缩为摘要


  上下文压缩/分块在 src/context/compaction.ts，使用 自适应 Token 分块：

  触发条件：
  ```ts
  // 当 contextTokens > contextWindow - reserveTokens 时触发
  const shouldCompact = totalTokens > contextWindowTokens - settings.reserveTokens;
  ```

  分块策略：
  1. 自适应比率：computeAdaptiveChunkRatio() 根据消息数量和平均长度计算
  2. 按 Token 均分：splitMessagesByTokenShare() 将消息分成 N 份
  3. 单块上限：chunkMessagesByMaxTokens() 每块不超过 maxChunkTokens

  摘要生成：
  - 分阶段摘要：长消息分多块分别摘要，再合并
  - 支持增量摘要：基于之前的摘要追加新内容
  - 结构化格式：目标、约束、进展、关键决策、下一步

  ---
  ## 4. 来源 (Sources)

  引导文件加载在 src/context/bootstrap.ts 和 src/context/loader.ts：

  支持的 Bootstrap 文件：

  这里为你将表格转换为标准的 Markdown 格式，同时保留原有的排版结构：

| 文件         | 用途           |
| ------------ | -------------- |
| AGENTS.md    | Agent 角色定义 |
| SOUL.md      | 人格与语气指引 |
| TOOLS.md     | 工具配置       |
| IDENTITY.md  | 身份信息       |
| USER.md      | 用户偏好       |
| HEARTBEAT.md | 主动唤醒任务   |
| BOOTSTRAP.md | 通用引导       |
| MEMORY.md    | 长期记忆       |

  截断策略：
  - 默认最大 20,000 字符
  - 超过时保留头部 70% + 尾部 20%
  - 添加截断标记提示

  ```ts
  function trimBootstrapContent(content: string, maxChars: number) {
    if (content.length <= maxChars) return content;
    const headChars = Math.floor(maxChars * 0.7);
    const tailChars = Math.floor(maxChars * 0.2);
    // 添加截断标记
    return head + `[...truncated...]` + tail;
  }
  ```
  实际上还有两个重要来源：
  - 工具执行结果：这是对话的主要内容来源
  - 会话历史：Session 中的历史消息

  ---
  
  ## 总结

| 方面 | 实现方式                          | 对应文件                        |
| ---- | --------------------------------- | ------------------------------- |
| 存储 | JSONL 追加 + 内存缓存双写         | session.ts                      |
| 搜索 | BM25 关键词评分，无时间衰减       | memory.ts                       |
| 分块 | 自适应 Token 分块 + 分阶段摘要    | context/compaction.ts           |
| 来源 | Bootstrap 文件加载 + 截断         | context/bootstrap.ts, loader.ts |


# TypeScript：是什么及核心特性

TypeScript（TS）是什么？

TypeScript（简称 TS）是由微软开发的强类型超集，完全兼容 JavaScript（JS），并在其基础上增加了静态类型系统——简单来说，TS = JavaScript + 类型系统，最终会被编译成纯 JS 运行。

你可以把 TS 理解为“带说明书的 JS”：JS 是动态类型语言（运行时才检查类型，容易出隐蔽 bug），而 TS 让你在编写代码时就定义变量、函数、对象的类型，提前暴露类型错误，大幅提升代码的可维护性和可读性，尤其适合中大型项目/团队协作。


---
## 一、核心特性（新手易懂版）

1. 静态类型系统（最核心）

- 变量类型约束：声明变量时指定类型，TS 编译器会检查类型是否匹配，避免“字符串加减数字”这类低级错误。

```ts
// TS 写法（指定类型）
let age: number = 25; 
age = "25"; // 报错：不能将字符串赋值给数字类型

// 对比 JS 写法（无类型约束，运行时才可能发现问题）
let age = 25;
age = "25"; // 不报错，但后续逻辑可能出错
```

- 函数类型约束：定义参数和返回值类型，明确函数的输入输出。

```ts
// 定义：接收两个数字，返回数字
function add(a: number, b: number): number {
  return a + b;
}
add(1, 2); // 正常
add(1, "2"); // 报错：参数2不是数字
```

2. 完全兼容 JavaScript

- 所有 JS 代码可以直接在 TS 文件中运行，TS 不会破坏现有逻辑；

- 支持 ES6+ 所有特性（箭头函数、解构、Promise 等），还扩展了接口、泛型等高级特性。

3. 接口（Interface）—— 定义数据结构

用于约束对象的形状，让复杂数据结构更清晰（比如接口返回的 JSON 格式）。

```ts
// 定义用户接口
interface User {
  name: string;
  age: number;
  isVip?: boolean; // 可选属性
}

// 使用接口约束对象
const user: User = {
  name: "张三",
  age: 30
};
```

4. 编译时检查，运行时纯 JS

TS 不会直接运行，需要通过 tsc（TS 编译器）编译成 JS，编译过程中会检查类型错误，最终在浏览器/Node.js 中运行的还是纯 JS，无运行时性能损耗。


---
## 二、TS 解决了什么问题？

1. 提前发现 bug：把 JS “运行时的类型错误”提前到“编译时”，比如传错参数类型、访问不存在的对象属性，写代码时就会提示；

2. 提升代码可读性：类型注解相当于“自文档”，别人看代码时能快速知道变量/函数的用途，比如 age: number 一眼就知道是数字；

3. 适配大型项目：JS 无类型约束，项目越大越容易出隐蔽 bug，TS 的类型系统能规范代码，降低维护成本（React/Vue/Angular 等框架都推荐用 TS）；

4. 更好的开发体验：VS Code 等编辑器对 TS 有完美支持，能提供智能提示、自动补全、重构建议，写代码效率更高。


---
## 三、TS 的使用场景

- 前端框架开发：React、Vue 3、Next.js、Nuxt.js 等主流框架的官方推荐语言；

- 后端开发：Node.js 项目（Express/Nest.js），替代纯 JS 提升后端代码稳定性；

- 大型团队协作项目：类型约束能统一代码规范，减少沟通成本；

- 工具库开发：比如 axios、lodash 等库的 TS 版本，方便使用者获得类型提示。


---
## 四、简单上手步骤（新手友好）

1. 安装 Node.js（TS 编译依赖 Node）；

2. 全局安装 TS 编译器：

npm install -g typescript

3. 创建 demo.ts 文件，写 TS 代码；

4. 编译成 JS：

tsc demo.ts # 生成 demo.js 文件

5. 运行 JS 文件：

node demo.js


---
## 总结

1. TS 是 JS 的强类型超集，兼容所有 JS 语法，核心是静态类型系统；

2. TS 解决了 JS 动态类型导致的“隐蔽 bug 多、维护成本高”问题，适合中大型项目/团队协作；

3. TS 最终编译为纯 JS 运行，无运行时损耗，是前端/Node.js 开发的主流选择。

如果需要，我可以给你一份 TS 入门的核心代码示例（变量、函数、接口、泛型），帮你快速上手。




# Mini Agent 核心架构 超详细解析（agent.ts）

我用通俗、结构化、新手能懂的方式，把这套 Mini Agent 智能体的 5 大子系统 + 核心循环彻底讲清楚，你可以直接对照源码理解。


---
## 一、先搞懂：Mini Agent 是什么？

它是一个轻量级智能体框架，专门用来做：

- 能记住你、长期记忆

- 能调用工具（查资料、写代码、发消息等）

- 能主动唤醒自己

- 能管理多轮对话

- 能动态加载上下文

一句话：一个小型但完整的“AI 助手大脑”。


---
## 二、5 大核心子系统（最关键）

我会用生活类比 + 技术解释讲透。


---
1. Session Manager —— 会话管理器

作用：管理你和 AI 的每一轮对话，并且永久保存。

功能

- 记录每一次提问、AI 回答

- 区分不同用户、不同对话

- 持久化存储：JSONL 格式

  - JSONL = 一行一条 JSON，轻量、易读、适合日志/对话

类比

= 你的聊天记录抽屉，每次对话都存进去，永不丢失。


---
2. Memory Manager —— 长期记忆管理器

作用：让 AI 记住“很久之前”的事情，而不是只记住当前对话。

功能

- 存储长期记忆（用户习惯、偏好、重要事件、历史结论）

- 关键词检索：你问问题时，AI 自动搜相关记忆

- 不依赖上下文长度，记忆可以无限大

类比

= AI 的长期记忆笔记本，不是临时记忆，是永久记忆。


---
3. Context Loader —— 按需上下文加载器

核心作用：AI 思考前，自动把“需要用到的所有信息”拼进提示词里。

这是 Agent 最核心的能力之一：知道该带什么资料去思考。

它会加载 9 类上下文：

1. AGENTS：其他智能体信息

2. SOUL：AI 的性格、灵魂设定

3. TOOLS：可用工具列表

4. IDENTITY：AI 身份

5. USER：用户信息

6. HEARTBEAT：主动唤醒机制状态

7. BOOTSTRAP：启动初始化指令

8. MEMORY：长期记忆

9. CONTEXT：当前对话上下文

类比

= AI 的随身文件夹

你问问题 → AI 自动打开相关资料 → 再给你回答


---
4. Skill Manager —— 可扩展技能系统

作用：让 AI 拥有“能做什么”的能力，并且可以随时加技能。

特点

- 插件化

- 可热拔插

- 一个技能 = 一个工具/能力

例如：

- 搜索技能

- 写代码技能

- 发邮件技能

- 查天气技能

类比

= AI 的技能面板，想加新能力直接装插件就行。


---
5. Heartbeat Manager —— 主动唤醒机制

作用：让 AI 不用等你提问，自己主动做事。

功能

- 定时唤醒

- 事件触发唤醒

- 后台运行

- 主动推送消息、主动执行任务

类比

= AI 的闹钟 + 生物钟

到点自动醒、自动干活。


---
## 三、核心循环：agent-loop.ts（AI 的思考运行逻辑）

这是AI 真正运行的大脑流程。

结构分为两层：

- OUTER LOOP：外层循环（追问问答）

- INNER LOOP：内层循环（工具调用 + 转向控制）


---
图解核心循环
```ts
OUTER LOOP 外层循环（追问）
 ├─ INNER LOOP 内层循环（工具 + 控制）
 │   ├─ 注入待处理消息
 │   ├─ LLM 流式调用（AI 思考）
 │   ├─ 执行工具
 │   ├─ 检查 steering（是否要中断）
 │   └─ 循环：还有工具？还有消息？
 ├─ 检查是否需要追问
 └─ 有追问 → 继续外层循环
```

---
逐行解释

- 1. OUTER LOOP (follow-ups)

外层循环 = 多轮对话 / 追问循环

只要 AI 觉得“还没说完”“需要继续问你”，就会重新跑一轮外层。


---
- 2. INNER LOOP (tools + steering)

内层循环 = 工具调用 + 流程控制

AI 在这里：思考 → 调用工具 → 再思考 → 再调用工具……


---
内层循环步骤详解

① 注入 pendingMessages

把等待处理的消息放进上下文，比如：

- 系统指令

- 转向指令

- 用户追问


---
② LLM 流式调用

让大模型（GPT / 通义千问 / 文心一言）开始流式输出思考。


---
③ 执行工具（一个一个执行）

AI 思考完 → 决定调用工具 → 执行。

每执行完一个，就检查一次 steering。


---
④ 若 steering = true：立即跳过剩余工具

steering = 强制转向/中断

比如：

- 用户说“停止”

- 系统说“不需要继续查了”

- 出现错误

AI 会立刻停手，不再执行后面的工具。


---
⑤ 内层循环继续的条件

只要满足任意一个，就继续循环：

- 还有工具要调用（hasMoreToolCalls）

- 还有消息没处理（pendingMessages.length > 0）


---
回到外层循环

内层结束后 → 检查是否需要follow-up（追问）

- 需要 → 再跑一次外层

- 不需要 → 结束对话


---
## 四、用生活例子完整跑一遍

你说：“帮我查北京明天天气，然后发给我朋友。”

执行流程

1. Session：记录这条请求

2. Context Loader：加载身份、工具、记忆

3. Memory：查找你朋友的昵称、习惯

4. 外层循环开始

5. 内层循环

  - 注入消息

  - LLM 思考：需要查天气 + 发消息

  - 调用天气工具

  - 检查 steering：无中断 → 继续

  - 调用发消息工具

  - 工具执行完

6. 内层结束

7. 检查 follow-up：无需追问

8. 外层结束

9. Heartbeat：保持待命


---
## 五、最精简总结（方便记忆）

5 大子系统

1. Session：聊天记录管理

2. Memory：长期记忆 + 搜索

3. Context：自动加载所有需要的信息

4. Skill：可插拔技能

5. Heartbeat：主动唤醒

核心循环

- 外层：多轮追问

- 内层：工具调用 + 随时中断

- LLM：流式思考

- steering：紧急停止开关


---

## 六、补充

### Steering 超直白解释

Steering = 智能体的「方向控制器 / 紧急刹车 / 强制转向开关」

我用最简单的话给你讲清楚：


---
1. 官方定义

Steering = 对智能体执行流程的实时控制与干预

- 不让 AI 继续做无用的工具调用

- 不让 AI 跑偏

- 强制 AI 切换任务

- 强制 AI 停止


---
2. 最简单理解

AI 正在执行一连串工具，你突然喊停 / 让它换方向 → 这就是 steering


---
3. 生活类比

你让 AI：

先查天气 → 再查机票 → 再订酒店

你突然说：

不用查酒店了，直接告诉我结果就行

AI 收到这句话，立刻停止后续工具 → 这就是 steering = 中断/转向


---
4. 在你的代码里是什么作用？

你给的核心循环里写：

若 steering: 跳过剩余工具

意思就是：

只要触发 steering，AI 马上停止执行后面的所有工具

不管还有多少个没调用，全部放弃。


---
5. 什么场景会触发 steering？

最常见 4 种：

1. 用户手动中断

  - 停止

  - 取消

  - 不用继续了

2. 系统强制中断

  - 出错

  - 超时

  - 安全限制

3. AI 自己判断不需要继续

  - 信息已经足够

  - 工具结果没用

4. 切换任务方向

  - 从“查资料”变成“写总结”

  - 从“执行”变成“回答”


---
6. 一句话终极总结

Steering = 让 AI 立刻停手、换方向、不执行后面工具的控制信号


---
最精简记忆

- steering = 中断 / 刹车 / 转向

- 触发 = 跳过所有剩余工具

- 目的 = 不让 AI 瞎执行、浪费资源



### 事件广播到所有监听者：解耦与高效响应

我用最直白、最通俗的方式给你讲清楚，你马上就能彻底明白：

是一个事件被所有监听者监听吗，为什么要所有的，监听者对应哪个功能

是的，一个事件会被所有监听者同时收到。

不是“只给一个”，而是全部都发一遍。


---
1. 为什么要发给“所有”监听者？

因为一个事件可能触发多个不同的业务功能。

举个生活例子：

事件 = 手机收到一条新消息

这个事件要同时通知：

- 通知栏显示消息

- 消息图标小红点+1

- 声音/震动提醒

- 桌面组件刷新

- 聊天页面未读计数更新

这些功能互不相关，但都关心“新消息”这件事。

所以：

一个事件触发 → 所有关心它的功能都执行自己的逻辑

这就是发布-订阅模式的核心：

一件事发生，所有关心这件事的模块自动响应。


---
2. 监听者（listener）对应什么功能？

监听者 = 一个函数 = 一个业务功能

在你代码里：

listener(event)

每一个 listener 都是一个提前注册好的回调函数。

比如：

// 监听者1：更新UI
function updateUI(event) { ... }

// 监听者2：记录日志
function logEvent(event) { ... }

// 监听者3：发送数据到后端
function sendToServer(event) { ... }

当你调用 emit(事件)：

- updateUI 执行

- logEvent 执行

- sendToServer 执行

三个功能互不干扰，各自做自己的事。


---
3. 用你的业务场景解释

你这段代码来自 pi-agent（一个代理/事件驱动框架）：

当某个事件发生（比如：连接成功、状态变化、收到消息），

系统需要：

- 更新状态

- 打印日志

- 通知上层业务

- 触发监控

- 做数据上报

- 触发钩子函数

这些功能都在监听者列表里。

所以：

emit 就是把事件广播给所有这些功能。


---
4. 最关键的设计好处：解耦

如果不发给所有监听者，你就要写成：

if(需要更新UI) updateUI()
if(需要日志) log()
if(需要上报) send()
...

代码会越来越乱、越来越难维护。

而发布-订阅模式：

- 事件发送方不知道谁在监听

- 监听方不知道谁在发送

- 新增功能只需要加一个监听者，不用改原有代码

这就是解耦，大型项目必须这么做。


---
极简总结（你只需要记住这3句）

1. 一个事件 → 发给所有监听者

2. 每个监听者 = 一个独立功能

3. 目的：一件事发生，所有相关功能自动响应，代码不乱


---

### 总结 - 创建 SummarizeFn（用于 compaction 内容压缩）
把已经生成的 AI 内容 / 长对话，再丢给 AI 重新总结 → 压缩文字数量 → 确保不超过 params.maxTokens 长度限制。
通过 pi-ai 的 completeSimple 实现，与 Agent 当前的 model/apiKey 绑定
- pi-ai：你们内部封装的 AI 调用工具库
- completeSimple：库里最简单、直接调用 AI 生成文本的方法
- 在你代码里：用来调用 AI 做内容总结、压缩长度




  这是从 OpenClaw 43万行源码中提炼的核心设计，适合学习 AI Agent 的系统级架构。
