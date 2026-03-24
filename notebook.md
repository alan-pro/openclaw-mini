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






  这是从 OpenClaw 43万行源码中提炼的核心设计，适合学习 AI Agent 的系统级架构。
