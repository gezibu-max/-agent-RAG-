# Ragent 项目技术文档

## 一、项目概述

**Ragent** 是一个基于 Spring Boot 的 RAG（Retrieval-Augmented Generation，检索增强生成）综合智能体系统。它是一个企业级智能文档处理与检索平台，集成向量数据库，拥有智能问答、知识库管理、会话记忆、深度思考、多通道检索等功能。

- **项目名称**：ragent-ai
- **GroupId**：com.nageoffer.ai
- **版本**：0.0.1-SNAPSHOT
- **Java 版本**：17
- **Spring Boot 版本**：3.5.7
- **构建工具**：Maven（多模块）
- **前端**：React 18 + TypeScript + Vite
- **数据库**：PostgreSQL（含 pgvector 扩展）
- **许可协议**：Apache 2.0

---

## 二、项目模块架构

```
agent-main/
├── bootstrap/          # 主启动模块 + 全部业务代码
├── framework/          # 共享基础设施层
├── infra-ai/           # AI 基础设施层（LLM/Embedding/Rerank）
├── mcp-server/         # MCP 工具服务器（独立进程）
├── frontend/           # 前端应用（React 18 + TypeScript）
├── resources/          # 数据库脚本、Docker 编排文件、示例文档
├── docs/               # 架构文档（快速开始、多通道检索说明）
└── pom.xml             # 父 POM
```

### 模块依赖关系

```
mcp-server ← (HTTP JSON-RPC) → bootstrap
                                   ├── framework
                                   └── infra-ai
                                         └── framework
```

- **bootstrap**：主应用，依赖 framework + infra-ai
- **framework**：无内部依赖
- **infra-ai**：依赖 framework
- **mcp-server**：独立进程，与 bootstrap 通过 HTTP/JSON-RPC 通信
- **frontend**：独立前端应用，与 bootstrap 通过 HTTP REST API + SSE 通信

---

## 三、技术栈全景图

### 3.1 后端技术栈

| 类别 | 技术 | 版本 | 用途 |
|------|------|------|------|
| **框架** | Spring Boot | 3.5.7 | 应用框架 |
| **ORM** | MyBatis-Plus | 3.5.14 | 数据库访问 + 分页 |
| **数据库** | PostgreSQL + pgvector | - | 关系数据 + 向量存储 |
| **向量数据库** | Milvus（可选） | SDK 2.6.6 | 向量检索（支持 pgvector/Milvus 双模式） |
| **缓存/分布式** | Redis + Redisson | 7.x / 4.0.0 | 缓存、分布式锁、信号量、Snowflake ID |
| **消息队列** | RocketMQ | 5.3.2 | 异步消息（反馈收集、文档分块） |
| **认证** | Sa-Token | 1.43.0 | 无状态认证 + Redis 持久化 |
| **文档解析** | Apache Tika | 3.2.3 | 多格式文档解析（PDF/Word/Excel/HTML 等） |
| **对象存储** | AWS S3 SDK | 2.40.2 | S3 兼容存储（RustFS） |
| **HTTP 客户端** | OkHttp | 4.12.0 | HTTP 调用 LLM API |
| **工具库** | Hutool | 5.8.37 | 通用工具（Snowflake、日期、JSON 等） |
| **JSON** | Fastjson2 / Gson | 2.0.43 / 2.13.2 | JSON 序列化 |
| **线程增强** | Transmittable-Thread-Local | 2.14.5 | 线程池上下文传递 |
| **AOP** | AspectJ | - | 切面编程（幂等、限流、追踪） |
| **代码格式化** | Spotless | 2.22.1 | 代码格式统一 + 版权头 |
| **测试** | JUnit + Mockito | 5.20.0 | 单元测试 |

### 3.2 前端技术栈

| 类别 | 技术 | 用途 |
|------|------|------|
| **框架** | React 18 + TypeScript 5.5 | UI 框架 |
| **构建工具** | Vite 5 | 构建 + HMR 开发服务器 |
| **路由** | React Router v6 | 客户端路由 |
| **状态管理** | Zustand | 轻量状态管理 |
| **HTTP** | Axios | API 请求 + 拦截器 |
| **UI 原语** | Radix UI | 无障碍 UI 组件 |
| **样式** | Tailwind CSS 3.4 | 原子化 CSS |
| **Markdown** | react-markdown + remark-gfm | 消息渲染 |
| **代码高亮** | react-syntax-highlighter | 代码块着色 |
| **图表** | Recharts 3 | 管理后台图表 |
| **虚拟列表** | react-virtuoso | 消息列表虚拟化 |
| **表单** | react-hook-form + zod | 表单管理与校验 |
| **图标** | lucide-react | 图标库 |
| **提示** | sonner | Toast 通知 |
| **文件上传** | react-dropzone | 拖拽上传 |
| **日期** | date-fns | 日期格式化 |

### 3.3 AI 模型支持

| 提供商 | 支持能力 | 说明 |
|--------|----------|------|
| **阿里云百炼 (Bailian)** | Chat / Embedding / Rerank | 默认提供商 |
| **硅基流动 (SiliconFlow)** | Chat / Embedding | 备用提供商 |
| **Ollama** | Chat / Embedding | 本地部署模型 |
| **NOOP** | Rerank | 占位/测试 |

支持的模型路由特性：
- 多候选模型优先级排序
- 断路器模式（失败 2 次打开，30s后半开）
- 流式对话支持
- 首包探测（Probe Buffering）

---

## 四、core RAG 引擎核心流程

### 4.1 整体对话流程

```
用户提问
    ↓
【问题重写】QueryRewriteService → 多问题拆分
    ↓
【意图识别】IntentResolver → 意图树匹配 → 知识库定向
    ↓
【多通道检索】MultiChannelRetrievalEngine
    ├─ IntentDirectedSearchChannel（意图定向，始终执行）
    └─ VectorGlobalSearchChannel（全局兜底，条件触发）
    ↓
【后置处理链】
    ├─ DeduplicationPostProcessor（多通道结果去重）
    └─ RerankPostProcessor（Rerank 重排序）
    ↓
【Prompt 构建】RAGPromptService + ContextFormatter
    ↓
【LLM 生成】RoutingLLMService → 流式 SSE 输出
    ↓
【会话记忆】ConversationMemoryService → 摘要 + 上下文窗口管理
```

### 4.2 多通道检索架构

```
                         用户问题
                             ↓
                   【意图识别：IntentResolver】
                     /                    \
          意图置信度 ≥ 0.6            意图置信度 < 0.6
                  /                              \
      仅意图定向检索                    意图定向 + 全局向量检索
      (IntentDirected)                 (IntentDirected + VectorGlobal)
                  \                              /
                   └──────────┬─────────────────┘
                              ↓
                   【去重：Deduplication】
                              ↓
                   【Rerank 重排序】
                              ↓
                         Top-K 结果
```

### 4.3 意图树系统

- 树形结构，支持无限层级嵌套
- 叶子节点关联具体知识库（Knowledge Base）
- 节点携带示例问题，用于 LLM 意图匹配
- 支持启用/禁用、批量操作
- 缓存管理（IntentTreeCacheManager）

---

## 五、数据流与存储架构

### 5.1 数据库表结构

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| `t_user` | 系统用户 | id, username, password, role(admin/user) |
| `t_conversation` | 会话列表 | conversation_id, user_id, title, last_time |
| `t_conversation_summary` | 会话摘要 | conversation_id, content |
| `t_message` | 消息记录 | conversation_id, role(user/assistant), content |
| `t_message_feedback` | 消息反馈 | message_id, vote(like/dislike), reason |
| `t_sample_question` | 示例问题 | title, question |
| `t_intent_node` | 意图树节点 | parent_id, name, level, vector_space_id |
| `t_query_term_mapping` | 查询词映射 | source_term, target_term |
| `t_knowledge_base` | 知识库 | name, collection_name, embedding_model |
| `t_knowledge_document` | 文档 | kb_id, file_name, status, chunk_count |
| `t_knowledge_chunk` | 文档分块 | doc_id, content, embedding(vector), index |
| `t_knowledge_document_chunk_log` | 分块日志 | doc_id, status, message |
| `t_knowledge_document_schedule` | 定时刷新任务 | doc_id, cron_expr, source_url |
| `t_knowledge_document_schedule_exec` | 刷新执行记录 | schedule_id, status, result |
| `t_ingestion_pipeline` | 摄取管道 | name, pipeline_config(JSON) |
| `t_ingestion_pipeline_node` | 管道节点 | pipeline_id, node_type, config |
| `t_ingestion_task` | 摄取任务 | pipeline_id, status, source_type |
| `t_ingestion_task_node` | 任务节点执行记录 | task_id, node_type, status |
| `t_rag_trace_run` | RAG 追踪运行 | user_id, question, answer, duration |
| `t_rag_trace_node` | RAG 追踪节点 | trace_id, node_type, input, output, cost_ms |

### 5.2 基础设施依赖

| 组件 | 端口 | 用途 |
|------|------|------|
| **PostgreSQL** | 5432 | 业务数据 + 向量数据（pgvector） |
| **Redis** | 6379 | 缓存/分布式锁/信号量/Snowflake ID/Sa-Token 持久化 |
| **RocketMQ** | 9876 | 异步消息队列 |
| **Milvus**（可选） | 19530 | 向量检索引擎 |
| **RustFS**（S3） | 9000 | 对象存储 |
| **Ollama**（可选） | 11434 | 本地 LLM 模型 |
| **后端服务** | 9090 | Spring Boot API（上下文路径 /api/ragent） |
| **MCP 服务** | 9099 | MCP JSON-RPC 工具服务 |
| **前端** | 5173 | Vite 开发服务器 |

---

## 六、API 端点总览

### 6.1 认证与用户
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/auth/login` | 用户登录 |
| POST | `/auth/logout` | 用户登出 |
| GET | `/user/me` | 当前用户信息 |
| GET | `/users` | 用户列表（管理员） |
| POST | `/users` | 创建用户（管理员） |
| PUT | `/users/{id}` | 更新用户（管理员） |
| DELETE | `/users/{id}` | 删除用户（管理员） |
| PUT | `/user/password` | 修改密码 |

### 6.2 RAG 对话（核心）
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/rag/v3/chat` | SSE 流式对话 |
| POST | `/rag/v3/stop` | 停止流式任务 |
| GET | `/conversations` | 会话列表 |
| PUT | `/conversations/{id}` | 重命名会话 |
| DELETE | `/conversations/{id}` | 删除会话 |
| GET | `/conversations/{id}/messages` | 消息列表 |
| POST | `/conversations/messages/{id}/feedback` | 消息反馈 |
| GET | `/rag/settings` | 系统配置 |
| GET | `/rag/sample-questions` | 随机示例问题 |
| GET | `/rag/traces/runs` | RAG 追踪列表 |
| GET | `/rag/traces/runs/{id}` | 追踪详情 |

### 6.3 知识库管理
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/knowledge-base/{kbId}/docs/upload` | 上传文档 |
| GET | `/knowledge-base/{kbId}/docs` | 文档列表 |
| GET | `/knowledge-base/docs/search` | 全局文档搜索 |
| GET | `/knowledge-base/docs/{id}/chunks` | 文档分块列表 |
| POST | `/knowledge-base/docs/{id}/chunks` | 创建分块 |
| DELETE | `/knowledge-base/docs/{id}/chunks/{id}` | 删除分块 |
| POST | `/knowledge-base/docs/{id}/chunks/rebuild` | 重建向量 |

### 6.4 意图树
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/intent-tree/trees` | 完整意图树 |
| POST | `/intent-tree` | 创建节点 |
| PUT | `/intent-tree/{id}` | 更新节点 |
| DELETE | `/intent-tree/{id}` | 删除节点 |
| POST | `/intent-tree/batch/enable` | 批量启用 |
| POST | `/intent-tree/batch/disable` | 批量禁用 |

### 6.5 摄取管道
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/ingestion/pipelines` | 创建管道 |
| GET | `/ingestion/pipelines` | 管道列表 |
| POST | `/ingestion/tasks` | 创建执行任务 |
| POST | `/ingestion/tasks/upload` | 上传文件触发摄取 |

### 6.6 管理后台
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/admin/dashboard/overview` | 仪表盘概览 |
| GET | `/admin/dashboard/performance` | 性能指标 |
| GET | `/admin/dashboard/trends` | 趋势数据 |
| GET | `/mappings` | 查询词映射列表 |
| POST | `/mappings` | 创建映射规则 |
| GET | `/sample-questions` | 示例问题管理 |

### 6.7 MCP Server（端口 9099）
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/mcp` | JSON-RPC 入口 |
| - | `initialize` | MCP 协议握手 |
| - | `tools/list` | 列出可用工具 |
| - | `tools/call` | 调用工具 |

---

## 七、关键技术设计

### 7.1 AI 模型路由与容错

```
ModelSelector → 候选模型排序（优先级）
    ↓
ModelRoutingExecutor → 遍历候选
    ↓
ModelHealthStore → 断路器检查（CLOSED/OPEN/HALF_OPEN）
    ↓ 失败次数 ≥ 2 → OPEN（30s 后转 HALF_OPEN）
    ↓
ChatClient → HTTP 调用 LLM
    ↓
流式场景：ProbeBufferingCallback → 首包确认后提交
```

### 7.2 会话记忆管理

- **滑动窗口**：保留最近 N 轮对话历史
- **自动摘要**：超过 N 轮后，对早期对话自动生成摘要
- **TTL 过期**：可配置会话记忆过期时间
- **摘要独立存储**：摘要与原始消息分离存储
- **可配置参数**：`historyKeepTurns`、`summaryStartTurns`、`summaryMaxChars`

### 7.3 流式对话处理

```
GET /rag/v3/chat?question=xxx&conversationId=xxx&deepThinking=true
    ↓
SSE 事件流：
  event: meta       → 会话 ID、任务 ID
  event: message    → {type: "think"}  推理过程
  event: message    → {type: "response"} 回答内容
  event: finish     → 消息完成
  event: done       → 连接结束
  event: cancel     → 用户取消
  event: error      → 发生错误
```

- 前端使用 EventSource / ReadableStream 解析 SSE
- 支持取消（AbortController + 后端任务管理）
- 深度思考模式（显示推理过程）

### 7.4 文档摄取管道

```
Fetcher → Parser → Chunker → Enricher → Enhancer → Indexer
   ↓         ↓        ↓          ↓          ↓          ↓
 获取文档   解析    分块      扩充元数据   AI增强    写入向量库
```

支持的摄取来源：
- 本地文件上传（multipart form）
- HTTP URL 远程文件
- S3 对象存储
- 飞书文档

支持的节点类型（IngestionNodeType）：
- `FETCHER`：文件获取
- `PARSER`：文档解析（Tika、Markdown）
- `CHUNKER`：文本分块（固定大小 / 结构感知）
- `ENRICHER`：元数据扩充
- `ENHANCER`：AI 增强（LLM 提炼）
- `INDEXER`：向量嵌入 + 写入

### 7.5 分布式并发控制

| 场景 | 技术方案 |
|------|----------|
| **表单幂等提交** | Redisson 分布式锁 + 自定义注解 `@IdempotentSubmit` |
| **MQ 消费幂等** | Redis Lua SET NX + `@IdempotentConsume` |
| **聊天并发限流** | 全局分布式信号量 `@ChatRateLimit` |
| **文档上传限流** | Redisson `RPermitExpirableSemaphore` |
| **定时任务锁** | Redis 互斥锁（ScheduleLockManager） |
| **分布式 ID** | Redis Lua 脚本分配 WorkerId + Hutool Snowflake |

### 7.6 MCP 工具集成

MCP（Model Context Protocol）是 AI 模型调用外部工具的标准化协议：

```
bootstrap（主应用）
    ↓ HTTPClient 调用 MCP
mcp-server（端口 9099）
    ↓ 注册工具执行器
├── WeatherMCPExecutor（天气查询 - 模拟20个城市）
├── SalesMCPExecutor（销售数据查询 - 模拟）
└── TicketMCPExecutor（工单查询 - 模拟）
```

流程：
1. 主应用启动时连接 MCP Server → `tools/list` 获取工具列表 → 注册到 `MCPToolRegistry`
2. 对话中识别用户需要工具 → LLM 提取参数 → 调用 `tools/call` → 结果注入 Prompt

### 7.7 RAG 全链路追踪

- `@RagTraceRoot` / `@RagTraceNode` 注解驱动
- 记录每个节点的输入、输出、耗时、错误
- 支持按用户/时间范围查询
- 前端提供 Trace 详情页（瀑布图、节点卡片）

---

## 八、部署架构

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   前端 (5173) │────▶│ 后端 (9090)   │────▶│ PostgreSQL   │
│ React + Vite  │     │ Spring Boot  │     │    (5432)    │
└──────────────┘     └──────┬───────┘     └──────────────┘
                            │                    │
                     ┌──────┼──────┐      ┌──────┴───────┐
                     ▼      ▼      ▼      │  pgvector 扩展 │
              ┌────────┐ ┌────┐ ┌──────┐ └──────────────┘
              │ Redis  │ │ MQ │ │Milvus│
              │ (6379) │ │9876│ │(可选)│
              └────────┘ └────┘ └──────┘
                     │
              ┌──────┴───────┐
              │  MCP Server  │
              │   (9099)     │
              └──────────────┘
```

---

## 九、线程池设计

系统定义了 8 个专用线程池，确保不同业务场景的线程隔离：

| 线程池名称 | 用途 |
|------------|------|
| `mcpBatch` | MCP 工具并行调用 |
| `ragContext` | 上下文构建 |
| `ragRetrieval` | 知识库检索 |
| `ragInnerRetrieval` | 通道内并行检索 |
| `intentClassify` | 意图分类 |
| `memorySummary` | 会话摘要生成 |
| `modelStream` | 流式对话 |
| `chatEntry` | 聊天入口 |
| `knowledgeChunk` | 文档分块处理 |

---

## 十、项目启动指南

### 前置依赖
1. JDK 17+
2. PostgreSQL（需启用 pgvector 扩展）
3. Redis
4. RocketMQ
5. RustFS（或兼容 S3 的存储）
6. （可选）Milvus、Ollama

### 数据库初始化
```bash
# 执行建表脚本
psql -U postgres -d ragent -f resources/database/schema_pg.sql
# 执行初始数据
psql -U postgres -d ragent -f resources/database/init_data_pg.sql
```

### Docker 启动中间件
```bash
docker compose -f resources/docker/rocketmq-stack-5.2.0.compose.yaml up -d
docker compose -f resources/docker/lightweight/milvus-stack-2.6.6.compose.yaml up -d
```

### 后端启动
```bash
./mvnw spring-boot:run -pl bootstrap
# 或
mvn clean package -pl bootstrap -DskipTests
java -jar bootstrap/target/bootstrap-0.0.1-SNAPSHOT.jar
```

### 前端启动
```bash
cd frontend
npm install
npm run dev
```

### 默认账号
- 用户名：`admin`
- 密码：`admin`

---

## 十一、目录结构速查

```
bootstrap/src/main/java/com/nageoffer/ai/ragent/
├── RagentApplication.java          # 启动类
├── admin/                           # 管理后台（仪表盘）
├── core/                            # 核心引擎（分块、文档解析）
├── ingestion/                       # 文档摄取管道
│   ├── controller/                  # 摄取 API
│   ├── engine/                      # 摄取引擎
│   ├── node/                        # 管道节点（Fetch/Parse/Chunk/Enrich/Enhance/Index）
│   ├── strategy/fetcher/            # 文档获取策略（Local/HTTP/S3/Feishu）
│   └── service/                     # 摄取服务
├── knowledge/                       # 知识库管理
│   ├── controller/                  # 知识库/文档/分块 API
│   ├── schedule/                    # 定时刷新任务
│   └── service/                     # 知识库服务
├── rag/                             # RAG 核心
│   ├── controller/                  # 对话/意图树/反馈/设置/追踪 API
│   ├── core/                        # 核心引擎
│   │   ├── intent/                  # 意图识别
│   │   ├── memory/                  # 会话记忆
│   │   ├── prompt/                  # Prompt 构建
│   │   ├── retrieve/                # 多通道检索
│   │   ├── rewrite/                 # 问题重写
│   │   ├── vector/                  # 向量存储（pgvector/Milvus）
│   │   ├── mcp/                     # MCP 客户端
│   │   └── guidance/                # 对话引导
│   └── service/                     # RAG 服务
└── user/                            # 用户认证

framework/src/main/java/.../framework/
├── cache/           # Redis 序列化
├── config/          # 数据库/MQ/Web 自动配置
├── context/         # 用户上下文（TTL）、应用上下文
├── convention/      # 共享 DTO（ChatMessage, Result, RetrievedChunk）
├── database/        # MyBatis-Plus 元数据自动填充
├── distributedid/   # 雪花 ID 生成
├── errorcode/       # 错误码接口
├── exception/       # 统一异常层次
├── idempotent/      # 幂等注解 + AOP
├── mq/              # RocketMQ 生产者抽象
├── trace/           # RAG 追踪注解 + 上下文
└── web/             # 全局异常处理、SSE 发送器

infra-ai/src/main/java/.../infra/
├── chat/            # LLM 对话客户端（Bailian/Ollama/SiliconFlow）
├── config/          # AI 模型配置属性
├── embedding/       # 向量嵌入客户端
├── model/           # 模型路由、选择器、断路器
├── rerank/          # Rerank 客户端
├── token/           # Token 计数
└── util/            # 响应清理工具

mcp-server/src/main/java/.../mcp/
├── core/            # MCP 核心接口与注册
├── endpoint/        # MCP JSON-RPC 端点
├── executor/        # 工具执行器（天气/销售/工单）
└── protocol/        # JSON-RPC 消息定义

frontend/src/
├── components/      # 可复用组件（chat/admin/common/layout/ui）
├── hooks/           # 自定义 Hooks（useChat, useStreamResponse, useAuth）
├── lib/             # 工具函数
├── pages/           # 页面（ChatPage, LoginPage, admin/*）
├── services/        # API 服务层（13个服务文件）
├── stores/          # Zustand 状态（auth, chat, theme）
├── styles/          # 全局 CSS 设计系统
├── types/           # TypeScript 类型定义
└── utils/           # 工具函数
```
