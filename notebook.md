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




  这是从 OpenClaw 43万行源码中提炼的核心设计，适合学习 AI Agent 的系统级架构。
