二、整体架构一览

Claude Code 分层架构示意图

![Claude Code 分层架构示意图](./images/claude-code-architecture.png)

三、核心系统深度剖析
3.1 工具系统（Tool System）— 一切能力的基石
Claude Code 的所有”能力”都通过工具（Tool）暴露给 LLM。每个工具都是一个自包含的模块，定义了输入 Schema、权限模型和执行逻辑。

src/tools/ 目录下共有 40+ 个工具实现：



架构亮点：每个工具都是声明式的——通过 Schema 定义输入、通过权限模型定义安全边界、通过独立模块实现逻辑。这种设计使得新增工具的成本极低，同时确保了 LLM 能准确理解每个工具的能力边界。

3.2 命令系统（Command System）— 用户交互的前门
src/commands/ 目录下有 50+ 个斜杠命令，涵盖开发工作流的方方面面：

核心开发命令：

/commit — Git 提交
/diff — 查看变更
/review — 代码审查
/compact — 上下文压缩
配置与管理：

/config — 设置管理
/mcp — MCP 服务器管理
/memory — 持久化记忆管理
/skills — 技能管理
/hooks — 钩子配置
高级功能：

/vim — Vim 模式切换
/doctor — 环境诊断
/cost — 使用费用查看
/context — 上下文可视化
/desktop / /mobile — 跨平台交接
有趣的发现：源码中还有一些未公开的命令，比如 good-claude（给 Claude 正向反馈？）、buddy（伴侣精灵，一个彩蛋功能）、bughunter（自动化 Bug 搜索）、chrome（Chrome 扩展集成）等。

3.3 QueryEngine — 对话引擎的心脏
QueryEngine.ts 是整个 Claude Code 最核心的文件，约 46,000 行代码。它负责：

流式响应处理：与 Anthropic API 的流式通信
工具调用循环：LLM 输出 → 解析工具调用 → 执行工具 → 结果回传 → LLM 继续
Thinking 模式：Extended Thinking（扩展思考）的启用与管理
重试逻辑：API 错误的优雅降级与重试
Token 计数：精确的 Token 使用量追踪
这是一个经典的 Agent Loop 实现——LLM 不断”思考-行动-观察”，直到任务完成或需要用户输入。

3.4 权限系统（Permission System）— 安全的守门人
src/hooks/toolPermission/ 实现了一套多层权限模型。每次工具调用都会经过权限检查：


none
工具调用请求 → 权限检查 → [自动通过 | 提示用户 | 拒绝]

支持多种权限模式：

default — 默认模式，危险操作需确认
plan — 计划模式，仅允许只读操作
auto — 自动模式，信任所有操作
bypassPermissions — 跳过权限（开发/测试用）
这解释了为什么 Claude Code 在执行文件写入、Shell 命令等操作前会弹出确认提示——不是 LLM 在”犹豫”，而是权限系统在拦截。

3.5 Bridge 系统 — IDE 集成的桥梁
src/bridge/ 实现了 CLI 与 IDE 扩展之间的双向通信层：

bridgeMain.ts — 桥接主循环
bridgeMessaging.ts — 消息协议
bridgePermissionCallbacks.ts — 权限回调
replBridge.ts — REPL 会话桥接
jwtUtils.ts — JWT 认证
sessionRunner.ts — 会话执行管理
这套 Bridge 系统使得 VS Code 和 JetBrains 插件能够与底层的 Claude Code CLI 共享同一个会话和上下文，实现了“一个引擎，多个前端”的架构。

3.6 Agent 协调系统 — 从单 Agent 到多 Agent
src/coordinator/ 虽然目前只有一个 coordinatorMode.ts 文件，但结合 AgentTool、SendMessageTool、TeamCreateTool 等工具，可以看出 Claude Code 已经构建了一套多 Agent 协调机制：

AgentTool：生成子 Agent 处理独立子任务
SendMessageTool：Agent 间通信
TeamCreateTool：创建 Agent 团队并行处理
useSwarmInitialization.ts / useSwarmPermissionPoller.ts — Swarm 模式初始化与权限轮询
这意味着 Claude Code 可以同时启动多个 Agent 并行工作，类似于一个软件工程团队的分工协作模式。






Agent 多智能体协调系统



四、服务层架构
src/services/ 是外部依赖集成的汇聚点：



服务说明 api/Anthropic API 客户端、文件 API、Bootstrapmcp/Model Context Protocol 服务器连接管理 oauth/OAuth 2.0 认证流程 lsp/Language Server Protocol 管理器 analytics/GrowthBook 特性标志与分析 plugins/插件加载器 compact/对话上下文压缩 extractMemories/自动记忆提取 tokenEstimation.tsToken 数量估算 teamMemorySync/团队记忆同步 policyLimits/组织策略限制 remoteManagedSettings/远程托管设置 MagicDocs/智能文档 PromptSuggestion/提示词建议
服务	说明
api/	Anthropic API 客户端、文件 API、Bootstrap
mcp/	Model Context Protocol 服务器连接管理
oauth/	OAuth 2.0 认证流程
lsp/	Language Server Protocol 管理器
analytics/	GrowthBook 特性标志与分析
plugins/	插件加载器
compact/	对话上下文压缩
extractMemories/	自动记忆提取
tokenEstimation.ts	Token 数量估算
teamMemorySync/	团队记忆同步
policyLimits/	组织策略限制
remoteManagedSettings/	远程托管设置
MagicDocs/	智能文档
PromptSuggestion/	提示词建议
autoDream/	自动梦境（？神秘功能）
autoDream/自动梦境（？神秘功能）


特别值得关注的是 extractMemories 服务——Claude Code 的记忆系统不仅仅是被动存储，还能主动从对话中提取有价值的信息并持久化。这解释了为什么 Claude Code 用久了会越来越”懂你”。

五、UI 组件体系 — 终端里的 React
Claude Code 使用 React + Ink 构建终端 UI，这是整个项目最”意外”的技术选型之一。src/components/ 下有 144 个 UI 组件，这几乎是一个中型 Web 前端应用的规模。

部分关键组件：

App.tsx — 应用根组件
CoordinatorAgentStatus.tsx — 多 Agent 状态展示
ContextVisualization.tsx — 上下文可视化
ContextSuggestions.tsx — 上下文建议
DiagnosticsDisplay.tsx — 诊断信息展示
AutoUpdater.tsx — 自动更新
ConsoleOAuthFlow.tsx — OAuth 认证流程
DevBar.tsx — 开发者工具栏
配套的 80+ React Hooks 覆盖了几乎所有状态管理场景：

useVimInput.ts — Vim 模式输入处理
useVoice.ts / useVoiceIntegration.tsx — 语音输入集成
useSwarmInitialization.ts — Swarm 模式初始化
useScheduledTasks.ts — 定时任务调度
useMemoryUsage.ts — 内存使用监控
useTerminalSize.ts — 终端尺寸自适应
useVirtualScroll.ts — 虚拟滚动（性能优化）
六、技术栈深度解析


类别技术选型理由分析运行时 Bun 极快的启动速度 + 原生 TypeScript 支持语言 TypeScript (strict)类型安全，512K 行代码必须强类型终端 UIReact + Ink 声明式 UI，组件复用，开发效率高 CLI 解析 Commander.js 成熟的 CLI 参数解析框架 Schema 校验 Zod v4 运行时类型校验 + TypeScript 类型推断代码搜索 ripgrep 极快的代码搜索引擎协议 MCP SDK + LSP 标准化的工具/语言服务器协议 APIAnthropic SDK 官方 SDK 遥测 OpenTelemetry + gRPC 标准化可观测性特性标志 GrowthBookA/B 测试与灰度发布认证 OAuth 2.0 + JWT + macOS Keychain
类别	技术	选型理由分析
运行时	Bun	极快的启动速度 + 原生 TypeScript 支持
语言	TypeScript (strict)	类型安全，512K 行代码必须强类型
终端 UI	React + Ink	声明式 UI，组件复用，开发效率高
CLI 解析	Commander.js	成熟的 CLI 参数解析框架
Schema 校验	Zod v4	运行时类型校验 + TypeScript 类型推断
代码搜索	ripgrep	极快的代码搜索引擎
协议	MCP SDK + LSP	标准化的工具/语言服务器协议
API	Anthropic SDK	官方 SDK
遥测	OpenTelemetry + gRPC	标准化可观测性
特性标志	GrowthBook	A/B 测试与灰度发布
认证	OAuth 2.0 + JWT + macOS Keychain	多层认证体系
多层认证体系


为什么选 Bun 而不是 Node.js？
Bun 提供了几个关键优势：

启动速度：比 Node.js 快数倍，CLI 工具对冷启动时间敏感
原生 TypeScript：无需编译步骤
Bundle 特性标志：bun:bundle 的 feature() 机制实现了编译时死代码消除

TypeScript
import { feature } from 『bun:bundle』 // 未启用的功能代码在构建时被完全移除 const voiceCommand = feature(『VOICE_MODE』) ? require(『./commands/voice/index.js』).default : null

目前已知的特性标志：PROACTIVE、KAIROS、BRIDGE_MODE、DAEMON、VOICE_MODE、AGENT_TRIGGERS、MONITOR_TOOL。

为什么用 React 写终端 UI？
这看似反直觉，但有深层考量：

声明式渲染：终端 UI 的状态管理同样复杂（权限弹窗、多 Agent 状态、流式输出），React 的声明式模型大大降低了复杂度
组件复用：144 个组件之间存在大量复用关系
Hooks 生态：80+ 自定义 Hooks 实现了关注点分离
Bridge 复用：同一套组件逻辑可以通过 Bridge 在 IDE 中复用
七、启动优化 — 毫秒级的细节打磨
Claude Code 的启动序列经过精心优化：


TypeScript
// main.tsx — 在其他模块加载之前就作为副作用触发 startMdmRawRead() // 预读 MDM 设置 startKeychainPrefetch() // 预取 Keychain 凭据

并行预取策略：MDM 设置读取、Keychain 访问、API 预连接和 GrowthBook 初始化全部并行执行，避免串行等待。

懒加载策略：重模块（OpenTelemetry ~400KB、gRPC ~700KB）通过动态 import() 延迟加载，只在实际需要时才引入。

八、几个有趣的发现
1. Buddy 系统（彩蛋）
src/buddy/ 目录——一个”伴侣精灵”系统。虽然目前看起来是个彩蛋功能，但谁知道 Anthropic 会不会把它变成一个正式功能？

2. autoDream 服务
src/services/autoDream/ ——“自动梦境”？名字非常神秘。可能与 Claude 的后台思考或记忆整理相关。

3. 隐藏命令
源码中存在许多未公开的命令：

good-claude — 给 Claude 正向反馈
bughunter — 自动化 Bug 猎手
chrome — Chrome 浏览器集成
btw — 顺便说一下（?）
ant-trace — Anthropic 内部追踪
mock-limits — 模拟速率限制（测试用）
heapdump — 堆内存快照
4. KAIROS 特性标志
在特性标志中出现的 KAIROS 是什么？Kairos 在希腊语中意为”恰当的时机”——可能是某个与时间相关的高级 Agent 调度功能。

九、架构设计的核心思想总结
经过深入分析，Claude Code 的架构体现了几个核心设计理念：

1. 工具即能力（Tools as Capabilities）
LLM 的所有交互能力都通过标准化的 Tool 接口暴露，每个工具都是自描述的（Schema + 权限 + 执行逻辑）。这种设计使得：

新增能力 = 新增一个工具模块
LLM 能通过 Schema 精确理解工具的输入输出
权限系统能细粒度控制每个工具的执行
2. Agent 是一等公民
Agent 不是一个事后想法，而是从架构层面就支持的核心概念。子 Agent 生成、Agent 间通信、团队协作都有专门的工具和服务支撑。

3. 声明式优于命令式
从 React/Ink 的 UI 渲染到 Zod Schema 的类型校验，再到工具定义的声明式接口，整个项目都倾向于声明式编程范式。

4. 性能是设计约束
Bun 运行时、并行预取、懒加载、编译时死代码消除——这些优化不是事后补丁，而是从技术选型阶段就考虑在内的设计决策。

十、写在最后
Claude Code 是目前市面上最复杂的 AI 编程 CLI 工具之一。51 万行 TypeScript 代码、40+ 工具、50+ 命令、144 个 UI 组件——这不是一个简单的”API 包装器”，而是一个完整的软件工程平台。

这次源码泄露让我们得以窥见 Anthropic 在 AI 工具链上的工程投入之深。对于开发者而言，这份源码是学习大型 TypeScript 项目架构、Agent 系统设计、终端 UI 开发的绝佳教材。