# GeneralAgent · 通用智能体平台

> 基于 **LangChain 1.0 + deepagents** 构建的可扩展通用 Agent 平台 —— 支持多用户多会话、文件上传下载、联网搜索、图文生成、Excel / PPT 子代理与 MCP 工具接入。

---

## 一、项目简介

通用智能体(General Agent)与普通问答机器人的区别在于:**能自主规划任务、调用工具、读写文件并交付产物**。本项目实现了一个开箱即用的通用 Agent 平台,核心难点不在算法,而在工程:

- **长链路任务出错概率指数级上升** → 需要任务规划与状态跟踪;
- **多会话文件互相污染** → 需要会话级文件系统隔离;
- **Agent 视角的路径与工具视角的路径不一致** → 需要路径归一化;
- **产物无法交付给用户** → 需要对象存储与下载链接回传。

## 二、核心能力

| 能力 | 说明 |
| --- | --- |
| 智能代理 | 工具调用、自主任务规划、多轮对话与上下文理解 |
| 多模态文件 | 接收与生成图片、文档、PDF、Word、Excel、PPT |
| 联网搜索 | 集成 Tavily,实时信息获取与结果整合 |
| 子代理 | Excel 子代理、PPT 子代理,把专业任务下发给专职 Agent |
| MCP 工具 | 通过 Model Context Protocol 接入外部工具服务 |
| 平台能力 | 多用户、多会话窗口,会话间完全隔离 |

## 三、架构设计

### 3.1 Agent 组装

平台入口是 `deepagents` 提供的 `create_deep_agent`,把模型、工具、子代理、文件后端、中间件与系统提示词组装成一个 Agent。相比基础 `create_agent`,它预置了任务规划、文件系统、工具修复、摘要与记忆管理等能力。

### 3.2 六大中间件时机

中间件是在 Agent 工作流中插入自定义逻辑的组件,共有六个时机:

| 时机 | 作用 |
| --- | --- |
| before_agent | Agent 开始工作前 |
| after_agent | Agent 结束工作后 |
| before_model | 每次调用模型前 |
| after_model | 每次模型返回后 |
| wrap_model_call | 包裹模型调用(可改写提示词) |
| wrap_tool_call | 包裹工具调用(可改写参数与结果) |

本项目利用 `wrap_model_call` 拼接任务规划提示词,用 `before_model` 注入目录结构,用 `wrap_tool_call` 做路径归一化,用 `after_agent` 回传产物。

### 3.3 会话级文件后端隔离

**问题**:不同会话窗口共用同一个文件根目录,互相污染。

**方案**:用会话 ID(thread_id)动态生成根目录 —— `/agent_files/{thread_id}`。

**难点与解法**:

1. `create_deep_agent` 创建时 `root_dir` 就被固定,无法运行中改变 → 改用**后端工厂(BackendFactory)**,在运行时按当前会话动态返回 `FilesystemBackend`;
2. 引入工厂后出现**线程阻塞**(os 级建目录操作) → 改为**懒加载**:自定义类继承 `BackendProtocol` 并重写相关方法,把建目录推迟到真正写文件时。

### 3.4 文件路径归一化

**问题**:Agent 视角下根目录是 `/agent_files/{thread_id}`,而其他工具没有这个概念,导致绝对路径与相对路径混用。

**方案**:

- `before_model` 时机注入当前目录结构(用文件更新时间判断是否有新文件,无变化则不注入,避免浪费 token);
- `wrap_tool_call` 时机把工具返回结果中的绝对路径改写为相对路径,同时兼容两种返回形态。

### 3.5 用户端文件收发(MinIO 对象存储)

**下载**:在 `after_agent` 时机检测本次工作是否产生新文件 → 有则打包工作目录为 zip → 上传 MinIO → 生成下载链接 → 以 AIMessage 追加进消息列表,前端即可展示可点击的下载链接。

**上传**:前端把用户上传的文件以 base64 放进 `upload_files` 字段 → Agent 在 `before_agent` 时机取出并落盘到工作目录。

文件类型转换使用 **markitdown**(微软开源),可读取大量文本格式。

### 3.6 MCP 工具接入

通过 MCP 协议接入外部工具服务,包括 Tavily 联网搜索、Excel MCP 与 PPT MCP。MCP 服务独立部署(见 `sub_projects/ppt-mcp`),Agent 侧统一走 `langchain-mcp-adapters` 调用。

### 3.7 子代理

把专业任务下发给专职子代理:Excel 数据处理交给 `excel-agent`,演示文稿生成交给 `ppt-agent`。主 Agent 只负责路由与结果汇总,避免单个 Agent 的提示词与工具集过度膨胀。

### 3.8 限流保护

大模型接口通常有速率限制。`wait_rate_limit` 中间件在触发限流时按固定间隔等待并重试,避免任务中途失败。

## 四、技术栈

| 层次 | 技术 |
| --- | --- |
| 语言 | Python 3.11+ |
| Agent 框架 | LangChain 1.0、LangGraph、deepagents |
| 工具协议 | MCP(Model Context Protocol) |
| 对象存储 | MinIO |
| 数据库 | PostgreSQL(pgvector) |
| 缓存 | Redis |
| 文档处理 | markitdown、python-pptx、pypandoc、weasyprint |
| 前端 | Next.js(基于 Agent Chat UI) |
| 部署 | Docker、Docker Compose |

## 五、目录结构

~~~
general-agent/
├── agent.py                    # LangGraph 入口(导出 agent 变量)
├── langgraph.json              # LangGraph CLI 配置
├── base/configs.py             # 全局配置(读取 .env)
├── conn/                       # 外部连接:llms / minio_conn / gen_img
├── content/
│   ├── all_agent.py            # 主 Agent 组装
│   ├── middles/                # 中间件:文件管理 / 限流
│   ├── mytools/                # 自定义工具:生图 / 读写文档 / VLM 读图
│   ├── mcps/                   # MCP 接入:tavily / excel / ppt
│   ├── sub_agents/             # 子代理:excel-agent / ppt-agent
│   └── others/mybackend.py     # 会话级文件后端(BackendFactory + 懒加载)
├── utils/                      # 文档转换 / zip 打包 / 流式输出 / 日志
├── sub_projects/
│   ├── agent-chat-ui/          # 前端(Next.js)
│   └── ppt-mcp/                # PPT MCP 服务
├── docker/                     # Dockerfile 与 compose
└── study/                      # 框架各组件的练习代码
~~~

## 六、快速开始

### 6.1 准备环境变量

~~~bash
cp .env.example .env
# 编辑 .env, 至少填写 HOST_IP / OPENAI_API_KEY / MODEL_API_BASE_URL / BASE_LLM
~~~

> 注意:`HOST_IP` 必须是外部可访问的 IP 或域名,不能填 `localhost` 或 `127.0.0.1`,因为前端需要用它拼接后端地址。

### 6.2 方式一:本地开发

~~~bash
pip install -r requirements.txt
langgraph dev          # 开发模式启动, 自动重载
~~~

### 6.3 方式二:Docker Compose 一键部署

~~~bash
cd docker
cp ../.env.example .env
docker compose -f docker-compose.build.yml up --build -d
~~~

启动后访问:

- `http://你的HOST_IP:3000` —— 前端操作界面
- `http://你的HOST_IP:8000/docs` —— 后端接口文档

### 6.4 配置要求

| 配置级别 | CPU | 内存 | 适用场景 |
| --- | --- | --- | --- |
| 最低配置 | 2 核 | 4 GB | 个人测试 |
| 生产环境 | 4 核+ | 8 GB+ | 高并发场景 |

## 七、扩展方式

- **新增工具**:在 `content/mytools/` 下按函数 + 描述的方式注册,加入 `_get_tools()` 列表;
- **新增中间件**:继承 `AgentMiddleware` 并重写目标时机的方法,加入 `_get_middlewares()` 列表;
- **新增子代理**:在 `content/sub_agents/` 下实现 `get_agent()`,加入 `_get_subagent()` 列表;
- **新增 MCP**:在 `content/mcps/` 下实现调用封装,按需在 `_get_tools()` 中挂载。
