# Pi Monorepo - Code Wiki

## 目录

- [项目概述](#项目概述)
- [整体架构](#整体架构)
- [主要包说明](#主要包说明)
  - [pi-ai](#pi-ai)
  - [pi-agent-core](#pi-agent-core)
  - [pi-coding-agent](#pi-coding-agent)
  - [pi-tui](#pi-tui)
  - [pi-mom](#pi-mom)
  - [pi-pods](#pi-pods)
  - [pi-web-ui](#pi-web-ui)
- [关键类与函数](#关键类与函数)
- [依赖关系](#依赖关系)
- [项目运行方式](#项目运行方式)

---

## 项目概述

Pi是一个用于构建AI代理和管理LLM部署的工具集，采用Monorepo架构组织代码。项目核心是一个交互式编码代理CLI，建立在多个可重用的基础包之上。

**核心设计理念**：
- 高度可扩展性：通过Extensions、Skills、Prompt Templates和Themes适应各种工作流
- 模块化设计：各包职责清晰，可独立使用
- 最小化核心：核心功能精简，通过扩展增加功能
- 多模式运行：支持交互式、打印、JSON、RPC和SDK嵌入等多种使用方式

---

## 整体架构

Pi采用分层架构设计，从底层到上层依次为：

```
┌─────────────────────────────────────────────────────────┐
│                    应用层 (Applications)                  │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │  pi-coding-agent │  │     pi-mom       │            │
│  │  (CLI编码代理)   │  │  (Slack机器人)    │            │
│  └──────────────────┘  └──────────────────┘            │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │    pi-pods       │  │   pi-web-ui      │            │
│  │  (vLLM部署管理)  │  │  (Web组件)       │            │
│  └──────────────────┘  └──────────────────┘            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                    代理层 (Agent Layer)                   │
│              ┌──────────────────┐                       │
│              │  pi-agent-core   │                       │
│              │  (代理运行时)     │                       │
│              └──────────────────┘                       │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   基础设施层 (Infrastructure)             │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │      pi-ai       │  │      pi-tui      │            │
│  │  (LLM API统一层)  │  │  (终端UI框架)    │            │
│  └──────────────────┘  └──────────────────┘            │
└─────────────────────────────────────────────────────────┘
```

### 架构特点

1. **依赖方向明确**：上层包依赖下层包，下层包不依赖上层
2. **职责分离**：
   - 基础设施层：提供LLM访问和UI组件
   - 代理层：提供代理运行时和工具执行
   - 应用层：具体的应用程序实现

---

## 主要包说明

### pi-ai

**包路径**: [packages/ai](file:///workspace/packages/ai)

**核心职责**: 统一多提供商LLM API，提供模型发现、提供商配置、Token和成本跟踪、上下文持久化以及跨提供商会话切换功能。

**主要模块**:

1. **Providers ([src/providers/](file:///workspace/packages/ai/src/providers))**
   - 各LLM提供商的实现（OpenAI、Anthropic、Google、Mistral等）
   - 每个提供商文件包含stream和complete函数
   - 消息和工具转换逻辑
   - 响应解析和标准化事件发射

2. **API Registry ([src/api-registry.ts](file:///workspace/packages/ai/src/api-registry.ts))**
   - 注册和查询可用的API提供商
   - 管理模型元数据

3. **模型系统 ([src/models.ts](file:///workspace/packages/ai/src/models.ts))**
   - 模型发现和查询
   - 支持自定义模型和提供商

4. **OAuth支持 ([src/utils/oauth/](file:///workspace/packages/ai/src/utils/oauth))**
   - 支持Anthropic、OpenAI Codex、GitHub Copilot等OAuth认证
   - Token刷新和管理

**关键接口**:
- `Model`: 模型元数据
- `Context`: 对话上下文
- `Tool`: 工具定义
- `AssistantMessage`: 助手消息

---

### pi-agent-core

**包路径**: [packages/agent](file:///workspace/packages/agent)

**核心职责**: 提供有状态的代理运行时，包括工具执行和事件流，建立在pi-ai之上。

**主要模块**:

1. **Agent类 ([src/agent.ts](file:///workspace/packages/agent/src/agent.ts))**
   - 状态管理和事件订阅
   - 消息队列处理（steering和follow-up）
   - 工具执行管理
   - 生命周期控制

2. **Agent Loop ([src/agent-loop.ts](file:///workspace/packages/agent/src/agent-loop.ts))**
   - 低级代理循环实现
   - 工具调用和执行
   - 事件序列编排

3. **类型定义 ([src/types.ts](file:///workspace/packages/agent/src/types.ts))**
   - AgentMessage: 代理消息类型
   - AgentState: 代理状态
   - AgentTool: 代理工具定义
   - AgentEvent: 代理事件

**核心特性**:
- 并行/顺序工具执行模式
- 动态API密钥解析
- 工具调用前置/后置钩子
- 自定义消息类型支持
- 上下文转换和过滤

---

### pi-coding-agent

**包路径**: [packages/coding-agent](file:///workspace/packages/coding-agent)

**核心职责**: 交互式编码代理CLI，是整个项目的主要应用。

**主要模块**:

1. **Core ([src/core/](file:///workspace/packages/coding-agent/src/core))**
   - **Agent Session**: 会话管理和运行时
   - **Compaction**: 上下文压缩和总结
   - **Tools**: 内置工具实现（read、write、edit、bash等）
   - **Extensions**: 扩展加载和运行
   - **Settings**: 配置管理
   - **Model Resolver**: 模型解析和选择

2. **Modes ([src/modes/](file:///workspace/packages/coding-agent/src/modes))**
   - **Interactive**: 交互式终端UI模式
   - **Print**: 打印响应模式
   - **RPC**: 远程过程调用模式
   - **JSON**: JSON输出模式

3. **CLI ([src/cli/](file:///workspace/packages/coding-agent/src/cli))**
   - 参数解析
   - 会话选择器
   - 模型列表

4. **Utils ([src/utils/](file:///workspace/packages/coding-agent/src/utils))**
   - Git操作
   - 图像处理
   - 剪贴板操作
   - 路径处理

**核心功能**:
- 会话树结构和分支
- 自动上下文压缩
- 扩展系统
- Skills集成
- Prompt Templates
- 主题定制
- Pi Packages（npm/git包管理）

---

### pi-tui

**包路径**: [packages/tui](file:///workspace/packages/tui)

**核心职责**: 最小化终端UI框架，提供差分渲染和同步输出，用于无闪烁的交互式CLI应用。

**主要模块**:

1. **Components ([src/components/](file:///workspace/packages/tui/src/components))**
   - Text, TruncatedText: 文本显示
   - Input, Editor: 文本输入
   - Markdown: Markdown渲染
   - SelectList, SettingsList: 选择列表
   - Loader, CancellableLoader: 加载动画
   - Image: 终端图像显示
   - Box, Container: 容器组件

2. **Core ([src/](file:///workspace/packages/tui/src))**
   - **TUI**: 主容器，管理组件和渲染
   - **Terminal**: 终端接口
   - **Key Detection**: 键盘输入检测
   - **Autocomplete**: 自动完成支持

**核心特性**:
- 三策略差分渲染系统
- 同步输出（CSI 2026）实现无闪烁
- 组件化架构
- 主题支持
- 覆盖层（Overlay）系统
- IME（输入法编辑器）支持
- 内联图像（Kitty/iTerm2协议）

---

### pi-mom

**包路径**: [packages/mom](file:///workspace/packages/mom)

**核心职责**: Slack机器人，将消息委托给pi编码代理处理。

**主要模块**:
- **Agent**: 代理集成
- **Slack**: Slack API集成
- **Store**: 数据存储
- **Tools**: 工具实现
- **Sandbox**: 沙箱执行

---

### pi-pods

**包路径**: [packages/pods](file:///workspace/packages/pods)

**核心职责**: CLI工具，用于管理GPU Pod上的vLLM部署。

**主要模块**:
- **Commands**: 模型、pod、prompt命令
- **Config**: 配置管理
- **SSH**: SSH连接管理
- **Model Configs**: 模型配置

---

### pi-web-ui

**包路径**: [packages/web-ui](file:///workspace/packages/web-ui)

**核心职责**: AI聊天界面的Web组件。

**主要模块**:
- **Tools**: 工具组件
- **Artifacts**: 制品组件

---

## 关键类与函数

### pi-ai 关键API

#### `getModel(provider, modelId)`
获取特定提供商的模型实例。
- **参数**: provider（提供商名称）, modelId（模型ID）
- **返回**: Model实例
- **用途**: 初始化LLM交互

#### `stream(model, context, options)`
流式获取助手响应。
- **参数**: model（模型实例）, context（对话上下文）, options（流选项）
- **返回**: AssistantMessageEventStream
- **用途**: 实时流式获取LLM响应

#### `complete(model, context, options)`
获取完整的助手响应（非流式）。
- **参数**: 同stream
- **返回**: Promise<AssistantMessage>
- **用途**: 获取完整响应

#### `validateToolCall(tools, toolCall)`
验证工具调用参数。
- **参数**: tools（工具列表）, toolCall（工具调用）
- **返回**: 验证后的参数
- **用途**: 工具执行前的参数验证

---

### pi-agent-core 关键类

#### `Agent`
有状态的代理类。
- **主要方法**:
  - `prompt(input, images?)`: 发送提示
  - `continue()`: 继续当前上下文
  - `steer(message)`: 注入引导消息
  - `followUp(message)`: 队列后续消息
  - `subscribe(listener)`: 订阅事件
  - `abort()`: 中止当前操作
- **用途**: 构建有状态的AI代理应用

#### `AgentTool`
代理工具定义接口。
- **属性**: name, label, description, parameters, execute
- **用途**: 定义可供代理使用的工具

---

### pi-tui 关键类

#### `TUI`
终端UI主容器。
- **主要方法**:
  - `addChild(component)`: 添加组件
  - `removeChild(component)`: 移除组件
  - `start()`: 启动UI
  - `stop()`: 停止UI
  - `showOverlay(component, options)`: 显示覆盖层
  - `hideOverlay()`: 隐藏覆盖层
- **用途**: 构建交互式终端应用

#### `Editor`
多行文本编辑器组件。
- **特性**: 自动完成、文件路径补全、粘贴处理、垂直滚动
- **用途**: 用户输入

---

## 依赖关系

### 包依赖图

```
pi-coding-agent
├── pi-agent-core
│   └── pi-ai
├── pi-tui
└── pi-ai

pi-mom
├── pi-agent-core
│   └── pi-ai
└── pi-ai

pi-web-ui
└── pi-ai

pi-pods
(无内部依赖)
```

### 关键外部依赖

- **@sinclair/typebox**: 类型安全的模式定义和验证
- **vitest**: 测试框架
- **typescript**: 类型系统
- **biome**: 代码格式化和 linting
- **husky**: Git hooks

---

## 项目运行方式

### 环境要求

- Node.js >= 20.0.0
- npm

### 安装依赖

```bash
npm install
```

### 构建项目

```bash
npm run build
```

构建顺序：
1. tui
2. ai
3. agent
4. coding-agent
5. mom
6. web-ui
7. pods

### 开发模式

```bash
npm run dev
```

同时启动所有包的开发模式。

### 代码检查

```bash
npm run check
```

执行 lint、格式化和类型检查。

### 运行测试

```bash
npm test
```

运行所有包的测试（跳过需要API密钥的测试）。

### 从源码运行pi

```bash
./pi-test.sh
```

从源码运行pi编码代理（可在任何目录执行）。

### 特定包测试

从包根目录运行特定测试：

```bash
cd packages/agent
npx tsx ../../node_modules/vitest/dist/cli.js --run test/agent.test.ts
```

---

## 贡献指南

请参考 [CONTRIBUTING.md](file:///workspace/CONTRIBUTING.md) 和 [AGENTS.md](file:///workspace/AGENTS.md) 获取详细的贡献指南。

---

## 许可证

MIT
