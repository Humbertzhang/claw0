# Section 02: Tool Use 学习笔记

## 1. Tools 定义格式：是 Anthropic 规范，源自 JSON Schema

`TOOLS` 列表中每个工具对象由三个核心字段组成：
- `name`: 工具的唯一名称，LLM 靠它决定调用哪个工具。
- `description`: **最重要的字段**，LLM 全靠这段自然语言来理解工具用途，决定何时使用。
- `input_schema`: 参数格式，遵循业界标准 **JSON Schema**（包含 `type`, `properties`, `required` 等字段）。

**JSON Schema** 是一种用于描述和约束 JSON 数据结构的规范。（如：参数必须是字符串 `type: string`，哪些必填 `required: [...]`）。它早已广泛用于 REST API 定义（OpenAPI/Swagger）等场景，被 Anthropic 和 OpenAI 一同借用，是当前的事实行业标准。

**与其他 Agent 的兼容性：** 不同厂商（Anthropic/OpenAI/Google）的工具定义高度相似，只有个别字段名不同（如 Anthropic 用 `input_schema`, OpenAI 用 `parameters`）。内部 JSON Schema 内容完全通用，也是 LangChain 等框架能自动转换的原因。

---

## 2. `bash` 工具：名字是 Bash，但能兼容 Windows

工具名叫 `bash` 只是一个**面向 LLM 的语义标签**，底层实际执行的是：

```python
subprocess.run(command, shell=True, ...)
```

`shell=True` 让 Python 在 Windows 上自动调用 `cmd.exe`，在 Linux/Mac 上调用 `/bin/sh`，实现了跨平台自适应。

**LLM 如何知道生成 PowerShell/CMD 命令而非 Bash 命令？**
1. **试错与纠错**：LLM 先尝试 Linux 命令，若执行失败返回 Windows 风格报错（如 `'cat' is not recognized...`），就能即时推断出是 Windows 环境并切换语法。
2. **上下文线索**：对话中出现 `C:\` 路径、`PowerShell` 字样等，LLM 可立刻识别操作系统。
3. **System Prompt 预埋信息（生产级最佳实践）**：框架启动时用 `platform.system()` 获取 OS 并注入 System Prompt，一次到位，无需试错。

---

## 3. `read_file` 工具：生产级应加行范围参数

`s02` 的实现是读取**文件全部内容**并截断。生产级代码会增加：
- `start_line` / `end_line`：精准读取指定行范围，节约 Context Window。
- 关键词/正则搜索（`grep_file`）：只返回匹配行及其上下文。
- 文件大纲（`outline_file`）：先看函数/类列表，再按需定位展开。

> **核心思想**：LLM 的 Context Window 是有限稀缺资源，要精准投喂"刚好够用的信息"，而不是无脑全量塞入。

---

## 4. 工具的并发执行：生产级应并行，而非串行循环

`s02` 中处理一批工具调用时使用了顺序 `for` 循环。但 Claude 支持在单次响应中同时返回多个 `tool_use` block（即发起并行调用），串行处理会白白浪费时间。

**生产级做法**：使用 `asyncio.gather()` 或 `ThreadPoolExecutor` 并发执行同批次的所有工具调用。

**边界条件（并非所有场景都能并发）：**
| 场景                                 | 是否可并发               |
| ------------------------------------ | ------------------------ |
| 同时读取多个独立文件                 | ✅ 可以                   |
| 有依赖关系的命令（先建目录再写文件） | ❌ 不行                   |
| 同时修改同一文件                     | ❌ 危险（Race Condition） |

> Claude 自身在决策时也会判断工具间是否有依赖，独立的操作会在同一响应中并行发出，有依赖的会分多轮顺序请求。执行层应忠实地尊重这个语义。
