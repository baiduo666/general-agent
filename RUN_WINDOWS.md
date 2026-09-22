# Windows 本地运行指南

> 这份文档记录了在 **中文 Windows** 上把本项目跑起来时踩到的坑和对应解法。
> Linux / macOS 或 Docker 部署请直接看主 README。

## 一、环境要求

| 组件 | 版本 | 说明 |
| --- | --- | --- |
| Python | 3.11 | 与后端 Docker 镜像 `langchain/langgraph-api:3.11` 对齐 |
| Node.js | 20+ | 仅前端需要 |
| MinIO | 任意版本 | **必须在启动 Agent 前就绪**, 见下文说明 |
| PostgreSQL + Redis | 可选 | `langgraph dev` 用内存存储; 二者只在容器化部署或需要持久化时需要 |

## 二、MinIO 是硬前置(最容易踩的一个坑)

`content/middles/file_manager_middle.py` 在**构造函数**里就创建了 `MinioConn`, 而该中间件在
`AllAgent.__init__` 中被实例化, `AllAgent` 又在 `agent.py` **模块级**执行 —— 也就是说:

```
agent.py  →  AllAgent()  →  FileMiddleware()  →  MinioConn()  →  bucket_exists()  [真实网络请求]
```

**MinIO 没起, `langgraph dev` 连 import 都过不去**, 报的是连接被拒而不是「功能不可用」。
所以务必先确认 `http://127.0.0.1:9000/minio/health/live` 返回 200, 再启动服务。

## 三、中文 Windows 的两个编码坑

### 3.1 `.env` 不能带中文注释

`langgraph_api` 用 `DotEnv(dotenv_path=env).dict()` 读 `.env`, 走的是**系统默认编码(GBK)**。
`.env` 里只要有中文, 就会抛:

```
UnicodeDecodeError: 'gbk' codec can't decode byte 0xac
```

**解法**: `.env` 只用 ASCII(注释也写英文)。

### 3.2 必须开启 Python UTF-8 模式

`langgraph_api/validation.py` 用 `open(...).read()` 读包内 JSON 时未指定编码, 在中文 Windows 上同样会 GBK 解码失败。
**解法**: 启动前设置

```powershell
$env:PYTHONUTF8 = '1'
$env:PYTHONIOENCODING = 'utf-8'
```

## 四、其他必须处理的点

### 4.1 缺 `colorama` 会导致启动失败

```
SystemError: ConsoleRenderer with `colors=True` on Windows requires the colorama package installed
```

`pip install colorama` 即可(已加入 requirements.txt)。

### 4.2 `pnpm` 可能被其他工具的同名 shim 覆盖

如果 `pnpm install` 报 `MODULE_NOT_FOUND ... pnpm.cjs`, 说明 PATH 里的 `pnpm` 不是真正的 pnpm。
解法: 用 `npm install -g pnpm`, 然后**用绝对路径**调用, 例如:

```powershell
& "$env:APPDATA\npm\pnpm.cmd" install
```

### 4.3 `langgraph dev` 会监听 `node_modules`, 装前端依赖会把后端搞死

`watchfiles` 默认监听整个工程目录。执行 `pnpm install` 时会产生成百上千次文件变更,
触发无限重载, 最终后端自己退出。

**解法**: 用 `--no-reload` 启动后端:

```powershell
langgraph dev --port 2024 --no-browser --no-reload
```

## 五、完整启动步骤

```powershell
# 1) 依赖
pip install -r requirements.txt

# 2) 配置(注意 .env 只能是 ASCII)
copy .env.example .env
# 填写 HOST_IP / OPENAI_API_KEY / MODEL_API_BASE_URL / BASE_LLM

# 3) 先起 MinIO 并确认健康
#    http://127.0.0.1:9000/minio/health/live  ->  {"status":"ok"}

# 4) 启动后端(必须带 UTF-8 模式与 --no-reload)
$env:PYTHONUTF8 = '1'
$env:PYTHONIOENCODING = 'utf-8'
langgraph dev --port 2024 --no-browser --no-reload

# 5) 启动前端(另开一个终端)
cd sub_projects/agent-chat-ui
& "$env:APPDATA\npm\pnpm.cmd" install
& "$env:APPDATA\npm\pnpm.cmd" dev
```

打开 `http://127.0.0.1:3000` 即可对话。前端 `.env` 需要:

```
NEXT_PUBLIC_API_URL=http://127.0.0.1:2024
NEXT_PUBLIC_ASSISTANT_ID=agent
LANGGRAPH_API_URL=http://127.0.0.1:2024
```

## 六、验证清单

- [ ] `http://127.0.0.1:2024/ok` 返回 `{"ok":true}`
- [ ] `http://127.0.0.1:3000/api/env` 返回正确的 `API_URL` 与 `ASSISTANT_ID`
- [ ] 对话能正常回复
- [ ] 让它「创建一个 hello.txt」能成功, 并返回 MinIO 下载链接
- [ ] `agent_files/<thread_id>/` 下能看到该会话的独立目录
