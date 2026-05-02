# React-Loop Function-Call 工程

> 基于 Claude Code Function-Call 机制拆解，实现歌曲创作流程的 AI 工作流引擎

---

## 项目概述

本工程参考 Claude Code 的 function-call 机制，设计并实现一套基于 React 的歌曲创作 AI 工作流引擎。

### 核心目标

1. 拆解 Claude Code 的 Tool/Function Calling 机制
2. 基于 react-loop 架构重新实现 function-call 循环
3. 开发面向歌曲创作场景的 AI 工具集

---

## Claude Code Function-Call 机制拆解

### 核心架构

Claude Code 采用 MCP (Model Context Protocol) 作为 function-call 的基础协议。

```
User Message → Claude API → Assistant Message (tool_use) → Execute Tools → Tool Result → Claude API → ...
```

### 消息类型

| 类型 | 角色 | 内容 |
|------|------|------|
| `user` | user | 原始用户消息 |
| `assistant` | assistant | Claude 回复（可能含 tool_use 块） |
| `tool_use` | assistant | 工具调用请求 |
| `tool_result` | user | 工具执行结果 |

### Tool Definition 格式

```typescript
{
  name: "bash",
  description: "Execute shell commands",
  input_schema: {
    type: "object",
    properties: {
      command: { type: "string", description: "The shell command" },
      timeout: { type: "number" }
    },
    required: ["command"]
  }
}
```

### 完整执行循环

```
1. 构建 Messages 数组 (包含历史)
2. 构建 Tools 数组 (可用工具)
3. POST /v1/messages (Anthropic API)
4. 解析 Response:
   - 若 content[0].type === "text": 输出文本
   - 若 content[0].type === "tool_use": 
     a. 提取所有 tool_use 块
     b. 并行执行所有工具调用
     c. 收集结果，构建 tool_result 消息
     d. 将结果追加到 Messages
     e. 继续请求
5. 重复直到无 tool_use
```

### 关键设计模式

| 模式 | 说明 |
|------|------|
| **Tool as First-Class** | 工具是独立对象，有定义、执行、结果 |
| **Parallel Execution** | 多个 tool_use 可并行执行 |
| **Result Injection** | 工具结果作为 user 消息注入 |
| **Streaming Parse** | 流式解析响应 |
| **Stateful Loop** | 维护 messages 数组，形成完整对话历史 |

---

## react-loop 架构设计

### 核心组件

```
┌─────────────────────────────────────────┐
│              LoopProvider                │
│  ┌─────────────────────────────────┐    │
│  │         Message State            │    │
│  │  - messages[]                    │    │
│  │  - toolCalls[]                   │    │
│  │  - songState                     │    │
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │         Tool Registry            │    │
│  │  - Map<string, ToolDef>         │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│              Hooks                       │
│  - useLoop()                            │
│  - useTool()                            │
└─────────────────────────────────────────┘
```

### 状态类型

```typescript
interface SongState {
  phase: 'idle' | 'lyrics' | 'melody' | 'chord' | 'arrangement' | 'mixing' | 'complete';
  lyrics?: string;
  melody?: string;
  chordProgression?: string;
  arrangement?: string;
  mixedResult?: string;
  metadata: { tempo?: number; key?: string; genre?: string; mood?: string; };
}
```

---

## 歌曲创作工具集

| 工具名 | 分类 | 功能 | 图标 |
|--------|------|------|------|
| `lyrics_generation` | 歌词 | 根据主题和风格生成歌词 | 🎤 |
| `melody_composition` | 旋律 | 根据歌词创作旋律 | 🎹 |
| `chord_progression` | 和弦 | 生成和弦进行 | 🎸 |
| `arrangement` | 编曲 | 乐器分配和声部编排 | 🎻 |
| `mixing` | 混音 | 混音和母带处理 | 🔊 |

### 工具调用示例

```
User: 帮我写一首关于春天的流行歌曲

Assistant (tool_use):
  - lyrics_generation({ theme: "春天", mood: "欢快", style: "pop" })
  
→ Tool Result: 歌词内容

Assistant (tool_use):
  - melody_composition({ lyrics: "...", tempo: 120, key: "C" })

→ Tool Result: 旋律数据

Assistant: 完成！以下是歌曲《春天》
```

---

## 工程结构

```
assets/song-loop/
├── src/
│   ├── types/
│   │   └── index.ts           # 类型定义
│   ├── providers/
│   │   └── LoopProvider.tsx   # 核心 Provider (约 250 行)
│   ├── components/
│   │   └── LoopUI.tsx         # 对话界面组件
│   ├── tools/
│   │   └── index.ts           # 5 个歌曲创作工具 Mock 实现
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── package.json
└── vite.config.ts
```

### 核心文件说明

| 文件 | 行数 | 说明 |
|------|------|------|
| `LoopProvider.tsx` | ~250 | 核心状态管理和 AI loop 执行 |
| `types/index.ts` | ~100 | 所有 TypeScript 类型定义 |
| `LoopUI.tsx` | ~100 | React 对话界面组件 |
| `tools/index.ts` | ~250 | 5 个工具的 Mock 实现 |

---

## 技术栈

| 层级 | 技术选择 |
|------|---------|
| 核心框架 | React 18 + TypeScript |
| 状态管理 | React Context + useReducer |
| 构建工具 | Vite |
| 样式 | Tailwind CSS (CSS-in-JS) |
| LLM 调用 | Anthropic SDK / Mock Provider |
| 开发模式 | 内置 Mock Provider |

---

## 下一步计划

- [ ] 实现 Streaming Response (SSE)
- [ ] 集成真实歌词生成 API
- [ ] 添加多轨编辑支持
- [ ] 实现 MIDI 导出
- [ ] 添加版本历史记录

---

## 相关资源

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [MCP 协议文档](https://modelcontextprotocol.io)
- [Anthropic API](https://docs.anthropic.com)

---

*本文档由 AI 生成，最后更新于 2026-05-02*