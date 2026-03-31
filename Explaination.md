# Claude Code 框架阅读笔记

## 一、按顺序阅读源文件

### 1. `src/models.py` — 数据结构

轻量级项目管理模型，专门用于记录"把某个系统从 A 移植到 B"的工作清单。

| 类 | 职责 |
|----|------|
| `Subsystem` | 描述原系统的一个子模块（名字、路径、文件数） |
| `PortingModule` | 每一项具体移植任务（名字、职责、来源、状态） |
| `PortingBacklog` | 把所有任务汇总在一起，可以输出摘要行 |

---

### 2. `src/port_manifest.py` — 工作区自描述

项目结构自动探针，为移植工作提供起点概览。

```
给定源码根目录
  → 自动扫描 .py 文件
  → 按顶层模块分组计数
  → 附上人工说明
  → 输出结构化清单 (PortManifest)
```

---

### 3. `src/commands.py` — 命令移植清单

负责加载、检索和展示已移植的命令列表。

| 函数 | 用途 |
|------|------|
| `command_names()` | 返回所有命令的名称列表 |
| `get_command(name)` | 按名称精确查找（大小写不敏感），找不到返回 `None` |
| `find_commands(query, limit)` | 模糊搜索，匹配名称或来源字段，最多返回 `limit` 条 |
| `render_command_index(...)` | 生成可读的文本索引，支持过滤和限制条数 |

---

### 4. `src/tools.py` — 工具移植清单

与 `commands.py` 是同一模式的平行实现，API 完全对称。

> 注：`commands` 和 `tools` 高度重复。如果项目继续扩展，可以抽出一个通用的 `SnapshotRegistry` 基类或工厂函数消除重复。

---

### 5. `src/runtime.py` — 路由核心

根据用户输入的自然语言，从命令和工具两个库里找出最相关的匹配项。

```
用户输入 prompt
    ↓ 分词（按空格、/ 、- 切分，转小写）
tokens 集合
    ↓ 对每个 command / tool 打分（token 命中 name/source_hint/responsibility 各加1分）
commands 匹配列表 + tools 匹配列表
    ↓ 平衡选取（各取至少1个代表，再按分数补充到 limit）
最终 RoutedMatch 列表（≤ limit 条）
```

---

### 6. `src/main.py` — CLI 入口

`argparse` 搭建的命令行，把所有子命令串联起来。没有业务逻辑，只做参数解析和模块调用的分发。

---

## 二、Agent Harness 三大概念

### 概念一：Tool Wiring（工具注册与调用）

**核心问题：** Agent 如何知道自己有哪些工具可以用？

```
reference_data/tools_snapshot.json     ← 原始快照，184 条工具记录
         ↓ json.loads()
load_tool_snapshot()                   ← 反序列化成 PortingModule 对象
@lru_cache(maxsize=1)                  ← 第一次读盘解析，之后直接返回缓存
         ↓
PORTED_TOOLS: tuple[PortingModule]     ← 全局注册表，所有模块共享
         ↓
get_tool(name) / find_tools(query)     ← 对外查询接口
```

---

### 概念二：Task Orchestration（任务调度）

**核心问题：** 谁决定做什么、按什么顺序做？

- `task.py` → `tasks.py` 定义了任务列表，每个任务有名字、描述、状态
- `runtime.py` 的 `PortRuntime.route_prompt()` 是调度器核心：接收用户意图，路由到最匹配的 command 或 tool

```
用户输入 (prompt)
     │
     ▼
PortRuntime.route_prompt()        ← 决定用哪个工具/命令
     │
     ├── PORTED_COMMANDS           ← 已注册的命令表
     └── PORTED_TOOLS              ← 已注册的工具表
              │
              ▼
         RoutedMatch[]             ← 返回匹配结果（kind / name / score）
```

---

### 概念三：Runtime Context（运行时上下文）

**核心问题：** Agent 如何知道自己"在哪里"、"有什么"、"做了多少"？

| 层级 | 文件 | 职责 |
|------|------|------|
| 层 1 — 路径上下文 | `context.py` | `PortContext`：存 `source_root / tests_root / assets_root`，`frozen=True` 不可变 |
| 层 2 — 工作区清单 | `port_manifest.py` | `PortManifest`：实时扫描 `src/` 目录，动态构建当前文件树——Agent 的自我感知 |
| 层 3 — 汇总引擎 | `query_engine.py` / `QueryEngine.py` | 把 manifest + commands + tools 聚合成完整视图 |

**两个引擎的区别：**

- `QueryEnginePort`（`query_engine.py`）— 静态汇总，渲染 Markdown 摘要
- `QueryEngineRuntime`（`QueryEngine.py`）— 继承 `QueryEnginePort`，加入 `route()` 方法，支持动态路由

```
QueryEnginePort.render_summary()    ← Runtime Context：汇总当前状态
     │
     ├── PortManifest               ← 文件系统上下文
     ├── PORTED_COMMANDS            ← 命令上下文
     └── PORTED_TOOLS               ← 工具上下文
```

---

## 三、各文件夹职责

### 核心运行时

| 文件夹 | 职责 | 核心内容 |
|--------|------|---------|
| `state/` | 全局应用状态管理 | `AppStateStore`：Agent 运行时的"内存"，所有模块共享同一状态树 |
| `coordinator/` | 任务协调器 | `coordinatorMode.ts`：决定当前是单 Agent 还是多 Agent 协作模式 |
| `assistant/` | 对话历史 | `sessionHistory.ts`：管理发给 Claude API 的 `messages` 数组 |
| `bootstrap/` | 启动初始化 | `state.ts`：App 启动时初始化全局状态 |

### 工具系统

`tools/` — 每个子目录是一个独立工具，结构固定：

```
tools/BashTool/
├── BashTool.tsx       ← 工具主逻辑：定义 name / description / input_schema，实现 execute()
├── UI.tsx             ← 工具结果在终端的渲染方式
├── prompt.ts          ← 给 Claude 看的工具描述（写入 system prompt）
└── bashPermissions.ts ← 权限检查：这条命令允许执行吗？
```

### 其他层

| 文件夹 | 职责 |
|--------|------|
| `services/` | 业务服务层（130个模块），如 AgentSummary（/compact）、SessionMemory（记忆读写） |
| `components/` | TUI 界面组件（389个），如 `App.tsx`、`AgentProgressLine.tsx` |
| `screens/` | 顶级页面：`REPL.tsx`（主界面）、`Doctor.tsx`、`ResumeConversation.tsx` |
| `hooks/` | React hooks：通知系统（104个），toolPermission 权限拦截 |
| `commands/` | 斜杠命令实现，每个命令一个子目录 |
| `constants/` | 常量：API 限制、错误码、终端图标等 |
| `types/` | 全局类型定义 |
| `schemas/` | 配置文件格式校验（Zod schema） |
| `memdir/` | `~/.claude/memory/` 目录的读写和相关性查找 |
| `migrations/` | 版本升级时的数据迁移逻辑 |
| `skills/` | 内置技能（`/commit`、`/review-pr` 等 slash 命令实现） |
| `utils/` | 工具函数（564个模块），按需取用 |

---

## 四、实现顺序（如果要自己填充这个框架）

```
第1步: types/                ← 先定好数据结构，后面都依赖它
第2步: constants/            ← 常量
第3步: state/                ← 状态管理（其他所有模块都读/写这里）
第4步: tools/ 选1个工具实现  ← 推荐 BashTool，最简单
第5步: assistant/            ← 接通 Claude API，管理消息历史
第6步: hooks/toolPermission/ ← 给工具加权限控制
第7步: coordinator/          ← 实现多 Agent 调度
```

---

## 五、Mini Agent 实现

基于这个框架，从零实现一个真正可运行的 Agent。

### 整体数据流

```
用户输入
  → [1] 构建 messages，调用 Claude API
  → [2] Claude 返回 tool_use（要调用哪个工具）
  → [3] 执行工具，得到结果
  → [4] 把结果塞回 messages，再次调用 API
  → [5] Claude 返回 end_turn，输出最终回复
  → 循环结束
```

---

### Step 1：实现工具（对应 `tools/`）

每个工具分两部分：**告诉 Claude 怎么调用**（definition）和**真正执行逻辑**（execute）。

```python
# src/tools/bash_tool.py
import subprocess

class BashTool:
    definition = {
        "name": "Bash",
        "description": "在终端执行 shell 命令",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {"type": "string", "description": "要执行的 shell 命令"}
            },
            "required": ["command"]
        }
    }

    def execute(self, command: str) -> str:
        result = subprocess.run(command, shell=True, capture_output=True, text=True)
        return result.stdout or result.stderr
```

```python
# src/tools/file_read_tool.py
class FileReadTool:
    definition = {
        "name": "FileRead",
        "description": "读取文件内容",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "文件路径"}
            },
            "required": ["path"]
        }
    }

    def execute(self, path: str) -> str:
        try:
            return open(path).read()
        except Exception as e:
            return f"Error: {e}"
```

---

### Step 2：实现消息历史（对应 `assistant/`）

```python
# src/assistant/session_history.py
class SessionHistory:
    def __init__(self):
        self.messages = []

    def add_user(self, text: str):
        self.messages.append({"role": "user", "content": text})

    def add_assistant(self, content):
        self.messages.append({"role": "assistant", "content": content})

    def add_tool_result(self, tool_use_id: str, result: str):
        # 工具结果以 user 角色返回（Claude API 协议规定）
        self.messages.append({
            "role": "user",
            "content": [{"type": "tool_result", "tool_use_id": tool_use_id, "content": result}]
        })
```

---

### Step 3：实现 Agentic Loop（对应 `coordinator/`）

这是整个 Agent 的核心，只有一个关键判断：

```python
# src/agent.py
import anthropic
from .assistant.session_history import SessionHistory
from .tools.bash_tool import BashTool
from .tools.file_read_tool import FileReadTool

client = anthropic.Anthropic()

TOOL_REGISTRY = {
    "Bash": BashTool(),
    "FileRead": FileReadTool(),
}

TOOL_DEFINITIONS = [t.definition for t in TOOL_REGISTRY.values()]


def run(user_input: str):
    history = SessionHistory()
    history.add_user(user_input)

    while True:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            tools=TOOL_DEFINITIONS,
            messages=history.messages
        )

        # Claude 的回复存入历史
        history.add_assistant(response.content)

        if response.stop_reason == "end_turn":
            # 没有更多工具调用，输出最终回复
            for block in response.content:
                if hasattr(block, "text"):
                    print(block.text)
            break

        if response.stop_reason == "tool_use":
            # 逐个执行 Claude 要求的工具
            for block in response.content:
                if block.type != "tool_use":
                    continue

                print(f"[调用工具] {block.name}({block.input})")

                tool = TOOL_REGISTRY[block.name]
                result = tool.execute(**block.input)

                history.add_tool_result(block.id, result)
            # 把工具结果发回给 Claude，继续循环


if __name__ == "__main__":
    run("帮我列出当前目录的文件，并告诉我有多少个 Python 文件")
```

---

### Step 4：运行

```bash
pip install anthropic
export ANTHROPIC_API_KEY="your-key"
python3 -m src.agent
```

---

### 执行时序

```
你的代码                         Claude API
   │                                 │
   │  messages=[{user: "列出文件"}]   │
   │  tools=[Bash, FileRead]         │
   │ ──────────────────────────────► │
   │                                 │  决定用 BashTool
   │ ◄────────────────────────────── │
   │  stop_reason="tool_use"         │
   │  content=[tool_use{ls}]         │
   │                                 │
   │  执行 ls → 得到文件列表           │
   │                                 │
   │  messages=[..., tool_result]    │
   │ ──────────────────────────────► │
   │                                 │  看到结果，生成自然语言回复
   │ ◄────────────────────────────── │
   │  stop_reason="end_turn"         │
   │  content=[text: "目录包含..."]  │
   │                                 │
   │  打印，结束                      │
```

---

### 在这个框架上扩展的方向

| 功能 | 需要实现的模块 |
|------|--------------|
| 权限控制（执行前询问用户） | `hooks/toolPermission/` |
| 跨会话记忆 | `memdir/` |
| 多 Agent 并行 | `coordinator/` + `AgentTool` |
| 上下文超长时压缩 | `services/AgentSummary/` |
| 流式输出 | `client.messages.stream()` |
