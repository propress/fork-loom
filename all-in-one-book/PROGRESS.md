# 项目实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 1 | 序言：先建立全局地图 | ch01-preface-system-map.md | 项目定位、读者路线图、架构全景图、核心概念词典、代码库地图、一次典型交互极简流程 | ⏳ |
| 2 | 从一次输入开始：主干执行链路总览 | ch02-main-flow-overview.md | CLI 入口、会话初始化、Agent 启动、状态机驱动主循环 | ⏳ |
| 3 | 数据流全景：一次请求如何穿过整个系统 | ch03-dataflow-panorama.md | 典型场景完整数据流，逐步拆解数据形态变化，复杂节点打上“详见第 N 章” | ⏳ |
| 4 | 状态机内核：Agent 为什么可控 | ch04-agent-state-machine.md | AgentState、AgentEvent、AgentAction、状态迁移与重试策略 | ⏳ |
| 5 | 工具系统：模型能力如何落到真实操作 | ch05-tool-system.md | ToolDefinition、ToolRegistry、工具调用协议、执行结果回注会话 | ⏳ |
| 6 | LLM 代理层：为什么要 server-side proxy | ch06-llm-proxy.md | ProxyLlmClient → server proxy → provider，密钥边界与流式返回 | ⏳ |
| 7 | 线程与持久化：对话如何被保存、恢复与同步 | ch07-thread-persistence.md | Thread/ThreadId、LocalThreadStore、SyncingThreadStore、快照模型 | ⏳ |
| 8 | 配置、密钥与安全边界 | ch08-config-and-secrets.md | 分层配置、Secret 类型、redaction、运行期安全取舍 | ⏳ |
| 9 | 项目演进史（上）：从最小可用到服务化 | ch09-evolution-phase-1-2.md | 早期 CLI + core，随后 server/web 进入主架构的转折点 | ⏳ |
| 10 | 项目演进史（中）：多租户与企业能力成形 | ch10-evolution-phase-3.md | auth/租户/组织边界与相关架构重塑 | ⏳ |
| 11 | 项目演进史（下）：可观测性与平台化扩展 | ch11-evolution-phase-4-5.md | analytics/crash/crons/sessions 与平台能力叠加 | ⏳ |
| 12 | 支线系统 1：Web 前端与 API 契约 | ch12-web-and-api-contract.md | Svelte 前端如何消费 server 能力，关键 API 契约与状态流 | ⏳ |
| 13 | 支线系统 2：Weaver 远程执行与运行环境 | ch13-weaver-runtime.md | weaver 生命周期、k8s 编排、与主流程的耦合点 | ⏳ |
| 14 | 端到端追踪：从用户问题到可验证结果 | ch14-end-to-end-trace.md | 选一个关键场景完整追踪，串联前文所有关键节点 | ⏳ |
| 15 | 架构复盘与阅读源码实战法 | ch15-recap-and-reading-playbook.md | 设计取舍复盘、常见误读点、如何继续自主深入源码 | ⏳ |

## 章节规划说明

认知路径按“先知道它是什么，再理解它怎么运作，再打开关键黑盒，再看演进与验收”组织：

1. **全局画面**：先用序言建立地图和词典，避免读者在陌生名词里迷路。  
2. **主干流程**：先走通“输入 → 状态机 → 工具/LLM → 持久化 → 输出”的主链路。  
3. **关键站点深挖**：按学习依赖拆开状态机、工具系统、LLM 代理、线程持久化。  
4. **支线系统补全**：在主干已理解后，再引入 Web、Weaver 等支线。  
5. **演进史独立成组**：本仓库 commit 历史丰富（704 commits），按阶段拆成 3 章，重点讲设计思路如何“长出来”。  
6. **端到端验收收束**：最后用一条完整追踪把全书知识闭环。

## 状态说明
- ✅ 已完成
- 🔄 进行中
- ⏳ 待开始

## 术语约定

### 行业标准术语（直接使用）
- State Machine（状态机）
- SSE（Server-Sent Events，服务端推送事件流）
- Trait（Rust 抽象接口）
- Retry / Exponential Backoff（重试 / 指数退避）
- Proxy（代理层）
- ABAC（属性基访问控制）

### 项目特有术语（首次出现时做类比与区别）
- **Thread**（项目语义：可同步的会话对象；类比一般聊天会话记录，但包含版本/可见性/同步语义）
- **Weaver**（项目语义：远程执行容器会话；类比 remote dev sandbox，但与 Loom 的会话与认证体系强绑定）
- **ProxyLlmClient**（项目语义：CLI 侧 LLM 客户端代理；类比普通 SDK client，但请求经 loom-server 统一转发）
- **ToolRegistry**（项目语义：工具注册与调度中心；类比插件注册表，但绑定统一 Tool 协议）

## 下次续写指引

### 从哪里继续
从 **第 1 章 `ch01-preface-system-map.md`** 开始，完成“序言”整章（含：全景图、概念词典、代码库地图、一次极简交互流程、质检报告）。

### 交接备忘
- `all-in-one-book/PROGRESS.md` 为首次规划稿，尚未开始章节正文。  
- 已确认仓库初始不存在 `all-in-one-book/PROGRESS.md`，本次已创建。  
- 已完成一次基础源码核验（README、specs 索引、核心入口与状态机/线程核心文件）。  
- 已补全 shallow clone 历史，当前可做完整演进分析。

### 待验证项
- Weaver 相关端到端调用链条细节（需在写到第 13 章前补足源码验证）。
- WhatsApp / SCIM 等扩展能力是否纳入正文还是附录（按主干相关性再决策）。
- Web 侧关键交互路径具体入口函数（写第 12 章前补函数级调用路径）。
