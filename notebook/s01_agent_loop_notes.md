# Section 01: Agent 循环 学习笔记

## 1. 关于 `end_turn`
**核心概念**：`end_turn` 是模型表达“我这轮说完了，该轮到你（用户）了”的标准停止状态。
- 它**不代表**所有对话结束或者程序退出。
- 它仅仅表明 LLM (如 Claude) 在当前这一回合（turn）的发言自然结束。
- **在代码实现中**（如 `s01_agent_loop.py`），当 API 返回 `stop_reason == "end_turn"` 时，程序会：
  1. 打印并显示 Assistant 的回复内容。
  2. 将该回复追加到 `messages` 历史记录中。
  3. 继续执行外层的大 `while True:` 循环，重新等待用户的下一轮输入。
- **退出机制**：只有当用户手动输入 `quit`、`exit` 或按下 `Ctrl+C` 时，才会真正退出整个对话脚本当中的循环。

## 2. 常见及其他的 `stop_reason`
除了 `end_turn` 外，Anthropic (Claude) API 中常见的 `stop_reason` 还包括：

### `tool_use`
- **含义**: 模型决定调用你提供给它的一个或多个工具。
- **表现**: 返回的内容中会包含工具块（ToolUseBlock），其中有要调用的工具名称和参数。此时模型在等待**开发者代码（系统）**去执行该工具，并将结果传回给它。对话在此暂时挂起。

### `max_tokens`
- **含义**: 模型生成的 token 数量达到了 API 请求中设置的 `max_tokens` 限制，或者是达到了模型单次输出的硬性最大限制（如 4096 或 8192个 tokens）。
- **表现**: 模型的回复可能是不完整的，话说到一半被截断。通常在 Agent 循环中需要专门处理这个状态，比如提示“回复过长被截断”或者通过代码逻辑让其“继续”。

### `stop_sequence`
- **含义**: 模型的输出匹配到了 API 调用时指定的 `stop_sequences` 列表中的某个字符串。
- **表现**: 比如设置了 `stop_sequences=["\nHuman:"]`，如果模型在生成文本时即将输出 `\nHuman:`，它就会认为这个回合该结束了，从而立即停止生成，并返回这个 `stop_reason`。

> **总结**：`end_turn`, `tool_use`, `max_tokens` 和 `stop_sequence` 这四种状态构成了大多数 LLM API 交互的控制流核心。编写 Agent 框架的重点之一，就是要在代码里精细地处理这四种不同的 `stop_reason`，以维持生生不息的多轮智能对话。
