# Codex Agent Team 能力增强蓝图

## Enhanced Agent Team: Creation, Management, Visualization & Human-Agent Interaction

> 基于对 Claude Code Agent Teams、OpenAI Codex Multi-Agent、Karpathy AgentHub/AutoResearch、
> 以及社区 Issues/PRs 的深度研究而设计的全面方案。

---

## 目录

1. [现状分析与痛点总结](#1-现状分析与痛点总结)
2. [设计原则](#2-设计原则)
3. [核心架构：嵌套 Agent Team 模型](#3-核心架构嵌套-agent-team-模型)
4. [Agent 生命周期管理](#4-agent-生命周期管理)
5. [Team 配置与声明系统](#5-team-配置与声明系统)
6. [通信与协调协议](#6-通信与协调协议)
7. [可视化系统](#7-可视化系统)
8. [人机交互层](#8-人机交互层)
9. [容错与恢复机制](#9-容错与恢复机制)
10. [实现路线图](#10-实现路线图)
11. [附录：参考来源](#11-附录参考来源)

---

## 1. 现状分析与痛点总结

### 1.1 Claude Code Agent Teams 现状

Claude Code 的 Agent Teams 是一个实验性功能（需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`），
采用 **Team Lead + Teammates** 模型：

- 一个 session 作为 team lead，协调工作和综合结果
- Teammates 在独立 context window 中工作
- 支持 peer-to-peer 消息传递和共享任务列表

**已知限制：**
- Session 重启后 team 无法恢复
- 任务可能永久停留在 "in progress" 状态
- 会话结束后孤儿 team 配置残留在磁盘上，阻塞后续 team 创建
- 多 teammate 编辑同一文件导致覆盖冲突
- Subagent 可以 `TeamCreate` 但缺少 `Task` tool，只能创建空壳 team
- **不支持嵌套 Agent Team**

### 1.2 OpenAI Codex Multi-Agent 现状

Codex 已实现 `spawn_subagents`（并行，最多 24 个）和 `chain_subagents`（串行），
自动处理 spawning、routing、waiting、collecting。

**已知问题（来自 GitHub Issues）：**

| Issue | 问题 | 严重性 |
|-------|------|--------|
| #13947 | Spawn slot 泄漏：完成的子 agent 永久占用 slot | Critical |
| #13947 | 缺乏级联取消：父 session 中断后子 agent 继续消耗 token | Critical |
| #14318 | 子 agent 异步通知与父 agent turn 生命周期的竞态条件 | High |
| #14194 | 子 agent 崩溃/挂起时不可靠的生命周期信号 | High |
| #9723 | Orchestrator 频繁中断子 agent，造成协调开销 | Medium |
| #9607 | 主 agent 未收到子 agent 完成信号，需手动查询 | Medium |
| #13947 | `wait` tool 是 wait-any 而非 wait-all | Medium |

### 1.3 社区提案摘要

| Issue | 提案 | 状态 |
|-------|------|------|
| #12047 | Multi-agent TUI 大改：命名 agent、per-agent config、async 编排、@mention 消息 | Open (8 👍) |
| #9902 | 持久 "Agents" 侧面板：活跃线程概览 + 快捷操作 | Open |
| #11815 | CLI 操作行添加 agent/thread 归属标识 | Open |
| #9846 | 高质量 Sub-Agent 协作（零配置 team mode） | Closed (dup) |
| #10067 | `--agents` flag 支持命名 AGENTS.md 变体 | Open |
| #3280 | 多 agent 编排：跨 agent 通信 | Closed (dup) |
| #2771 | Orchestrator agent 委托式多 agent 任务执行 | Closed |

### 1.4 Karpathy 的关键洞察

来自 AutoResearch（8-agent 实验）和 AgentHub 项目：

1. **"Org Code" 概念**：prompts + tools + processes = 组织代码（`program.md`），
   定义自治研究组织的结构
2. **扁平 peer 协调在 20-30 agents 时退化**：冗余探索、重复发现，
   显式层级结构优于 peer-based 方法
3. **Context 管理是核心瓶颈**：实验历史超过 ~5K tokens 后 agent 丧失连贯性，
   重新发现 dead ends 而非构建在前人工作之上
4. **人类 PI 监督不可或缺**：agent 善于执行明确定义的任务，
   但在假设生成和实验设计方面挣扎
5. **轻量级基础设施优先**：git branches per agent、file-based communication、
   tmux sessions，而非 Docker/VM
6. **DAG > 线性分支**：AgentHub 用 commit DAG 替代 main branch + PR 的传统模型

---

## 2. 设计原则

基于以上分析，提出以下核心设计原则：

### P1: 嵌套优先（Nesting-First）
Agent team 是一等公民，支持任意深度嵌套。一个 team 的成员可以是另一个 team。

### P2: 声明式配置（Declarative Configuration）
Team 拓扑通过声明式配置文件定义，而非命令式 API 调用。配置即文档即代码。

### P3: 渐进式复杂度（Progressive Complexity）
从单 agent → 简单 team → 嵌套 team → swarm，复杂度递增但 API 保持一致。

### P4: 人机协同（Human-in-the-Loop by Design）
人类可以在任何层级介入：观察、指导、批准、修正、接管。不是事后添加，而是架构核心。

### P5: 可观测性（Observability-First）
每个 agent 的状态、进度、通信、资源消耗必须实时可见、可追溯。

### P6: 优雅降级（Graceful Degradation）
子 agent 失败不应导致整个 team 崩溃。级联取消、超时、重试必须内建。

### P7: Context 经济性（Context Economy）
最小化 context 传递，agent 间通信使用摘要而非原始数据。类比 Karpathy 的发现：
context rot 是多 agent 系统的首要敌人。

---

## 3. 核心架构：嵌套 Agent Team 模型

### 3.1 实体模型

```
┌─────────────────────────────────────────────────────┐
│                    Agent Universe                     │
│                                                       │
│  ┌─────────────────────────────────────────────┐     │
│  │              Root Team (Project)              │     │
│  │                                               │     │
│  │  ┌──────────┐  ┌──────────────────────────┐  │     │
│  │  │ Human PI │  │     Orchestrator Agent    │  │     │
│  │  │ (Owner)  │  │     (@lead)               │  │     │
│  │  └──────────┘  └──────────────────────────┘  │     │
│  │                         │                     │     │
│  │         ┌───────────────┼───────────────┐     │     │
│  │         ▼               ▼               ▼     │     │
│  │  ┌─────────────┐ ┌──────────┐ ┌────────────┐ │     │
│  │  │ Sub-Team A  │ │ Agent C  │ │ Sub-Team B │ │     │
│  │  │ (Frontend)  │ │ (@tester)│ │ (Backend)  │ │     │
│  │  │             │ │          │ │            │ │     │
│  │  │ @fe-lead    │ └──────────┘ │ @be-lead   │ │     │
│  │  │ @fe-impl    │              │ @be-impl   │ │     │
│  │  │ @fe-style   │              │ @be-api    │ │     │
│  │  └─────────────┘              │ @be-db     │ │     │
│  │                               └────────────┘ │     │
│  └───────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### 3.2 核心实体定义

```rust
/// 一个 Agent 是最小执行单元
struct Agent {
    id: AgentId,
    name: String,              // 人类可读名称，如 "@fe-lead"
    role: AgentRole,           // orchestrator | worker | monitor | reviewer
    config: AgentConfig,       // model, instructions, tools, permissions
    state: AgentState,         // idle | running | waiting | blocked | done | failed
    parent_team: Option<TeamId>,
    metrics: AgentMetrics,     // tokens, cost, duration, tool_calls
}

/// 一个 Team 是 Agent 的有组织集合，可嵌套
struct Team {
    id: TeamId,
    name: String,              // 如 "frontend-team"
    orchestrator: AgentId,     // 该 team 的协调者
    members: Vec<TeamMember>,  // Agent 或嵌套 Team
    task_board: TaskBoard,     // 共享任务看板
    message_bus: MessageBus,   // 内部通信通道
    policy: TeamPolicy,        // 协调策略、权限、资源限制
    parent_team: Option<TeamId>,
    state: TeamState,
}

/// Team 成员可以是 Agent 或嵌套 Team
enum TeamMember {
    Agent(AgentId),
    SubTeam(TeamId),
}

/// 协调策略
enum OrchestrationPattern {
    Supervisor,    // 中心化：orchestrator 分配所有任务
    Hierarchy,     // 多级：orchestrator → sub-team leads → workers
    Pipeline,      // 流水线：顺序传递
    Swarm,         // 去中心化：peer-to-peer 自协调
    Hybrid,        // 混合模式
}
```

### 3.3 嵌套 Team 的关键语义

1. **作用域隔离**：子 team 的内部通信对父 team 不可见（除非显式 escalate）
2. **接口抽象**：父 team 将子 team 视为单个"黑盒" agent，仅通过子 team 的 orchestrator 交互
3. **资源继承与限制**：子 team 继承父 team 的资源配额，但可设置更严格的限制
4. **故障隔离**：子 team 内部故障通过其 orchestrator 汇报，不直接传播到父 team

```
消息流示意：

Human → Root.orchestrator → SubTeamA.orchestrator → SubTeamA.worker
                                                  ← SubTeamA.worker (result)
                          ← SubTeamA.orchestrator (summary)
       ← Root.orchestrator (synthesized answer)
← Human
```

---

## 4. Agent 生命周期管理

### 4.1 状态机

```
                ┌──────────┐
                │ Created  │
                └────┬─────┘
                     │ initialize()
                     ▼
              ┌──────────────┐
         ┌────│    Idle      │◄──────────────────┐
         │    └──────┬───────┘                    │
         │           │ assign_task()              │
         │           ▼                            │
         │    ┌──────────────┐                    │
         │    │   Running    │──── complete() ────┘
         │    └──────┬───────┘
         │           │
         │     ┌─────┼──────────┐
         │     │     │          │
         │     ▼     ▼          ▼
         │  ┌──────┐ ┌───────┐ ┌──────────┐
         │  │Wait  │ │Blocked│ │  Failed   │
         │  │(I/O) │ │(dep)  │ │           │
         │  └──┬───┘ └───┬───┘ └─────┬────┘
         │     │         │           │
         │     └────┬────┘     retry?│
         │          │ resume()  ┌────┘
         │          ▼           ▼
         │    ┌──────────────┐
         │    │   Running    │
         │    └──────────────┘
         │
         │ cancel() / timeout()
         ▼
  ┌──────────────┐
  │  Terminated  │
  └──────────────┘
```

### 4.2 级联生命周期管理

解决 Codex #13947 中 spawn slot 泄漏和缺乏级联取消的问题：

```rust
/// 级联取消策略
enum CancellationPolicy {
    /// 父 agent 取消时，所有子 agent 立即取消
    Immediate,
    /// 给子 agent 一个宽限期完成当前工作
    Graceful { timeout: Duration },
    /// 子 agent 独立运行，不受父 agent 取消影响
    Detached,
}

/// 资源回收保证
trait LifecycleManager {
    /// 确保 slot 在 agent 终止时必定释放（RAII 模式）
    fn release_on_drop(&self);
    /// 周期性健康检查，清理僵尸 agent
    fn health_check_sweep(&self, interval: Duration);
    /// 带有 parent_turn_id 的完成通知，解决 #14318 竞态条件
    fn notify_completion(&self, result: AgentResult, parent_turn_id: TurnId);
}
```

### 4.3 资源预算与限制

```toml
[team.budget]
max_total_tokens = 500_000
max_cost_usd = 5.00
max_agents = 12
max_nesting_depth = 3
max_agent_runtime = "30m"
max_team_runtime = "2h"

[team.budget.per_agent]
max_tokens = 50_000
max_cost_usd = 0.50
max_tool_calls = 200
```

---

## 5. Team 配置与声明系统

### 5.1 配置文件层次

受 Codex #12047 提案和 Karpathy "Org Code" 概念启发，设计三层配置：

```
~/.codex/
├── global_config.toml          # 全局默认配置
├── agents/                     # 全局 agent 模板库
│   ├── code-reviewer.toml
│   ├── architect.toml
│   ├── debugger.toml
│   └── test-writer.toml
└── teams/                      # 全局 team 模板库
    ├── fullstack.toml
    └── research.toml

<project>/
├── AGENTS.md                   # 项目级 agent 指令（已有）
├── .codex/
│   ├── team.toml               # 项目级 team 定义 ← 核心配置文件
│   ├── agents/                 # 项目特定 agent 配置
│   │   ├── frontend-lead.toml
│   │   └── backend-lead.toml
│   └── programs/               # Karpathy 式 "org code"
│       ├── feature-dev.md      # 功能开发流程
│       ├── bug-fix.md          # Bug 修复流程
│       └── code-review.md      # 代码评审流程
```

### 5.2 team.toml 核心格式

```toml
[team]
name = "project-alpha"
description = "Full-stack feature development team"
pattern = "hierarchy"               # supervisor | hierarchy | pipeline | swarm | hybrid
max_nesting_depth = 3

[team.orchestrator]
name = "@lead"
model = "gpt-5.3-codex"
reasoning_effort = "high"
instructions = "programs/feature-dev.md"
tools = ["spawn_agent", "create_team", "assign_task", "broadcast", "escalate"]

# 直属 agent 成员
[[team.agents]]
name = "@tester"
model = "gpt-5.3-codex-spark"
reasoning_effort = "medium"
instructions = "agents/test-writer.toml"
tools = ["shell", "read", "write"]
auto_launch = "on_task"             # manual | on_task | always

[[team.agents]]
name = "@reviewer"
model = "gpt-5.3-codex"
reasoning_effort = "high"
instructions = "agents/code-reviewer.toml"
tools = ["read", "grep", "glob"]
auto_launch = "on_task"

# 嵌套子 team
[[team.sub_teams]]
name = "frontend"
pattern = "supervisor"

[team.sub_teams.orchestrator]
name = "@fe-lead"
model = "gpt-5.3-codex"
instructions = "agents/frontend-lead.toml"

[[team.sub_teams.agents]]
name = "@fe-impl"
model = "gpt-5.3-codex-spark"
count = 2                           # 可以有同类型多个实例
tools = ["read", "write", "shell"]

[[team.sub_teams.agents]]
name = "@fe-style"
model = "gpt-5.3-codex-spark"
file_ownership = ["src/styles/**", "src/components/**/*.css"]

# 另一个嵌套子 team
[[team.sub_teams]]
name = "backend"
pattern = "supervisor"

[team.sub_teams.orchestrator]
name = "@be-lead"
instructions = "agents/backend-lead.toml"

[[team.sub_teams.agents]]
name = "@be-api"
file_ownership = ["src/api/**", "src/routes/**"]

[[team.sub_teams.agents]]
name = "@be-db"
file_ownership = ["src/models/**", "migrations/**"]

# Team 间通信规则
[team.communication]
# 哪些 team 可以互相发消息
allowed_channels = [
    { from = "frontend", to = "backend", via = "orchestrator" },
    { from = "@tester", to = "*", via = "direct" },
    { from = "@reviewer", to = "*", via = "direct" },
]

# 资源预算
[team.budget]
max_total_tokens = 1_000_000
max_cost_usd = 10.00
max_concurrent_agents = 8
```

### 5.3 "Org Code" — 程序化流程定义

受 Karpathy `program.md` 启发，定义结构化的执行流程：

```markdown
<!-- .codex/programs/feature-dev.md -->

# Feature Development Program

## Phase 1: Analysis (Team Lead)
- Analyze the feature request
- Break down into frontend and backend tasks
- Create task board entries with clear acceptance criteria

## Phase 2: Parallel Implementation
- Assign frontend tasks to `frontend` sub-team
- Assign backend tasks to `backend` sub-team
- Assign integration test specs to `@tester`

## Phase 3: Integration Gate
- **HUMAN CHECKPOINT**: Review task board, approve to proceed
- Run integration tests
- If failures: assign fix tasks, return to Phase 2

## Phase 4: Review
- `@reviewer` performs code review on all changes
- Address review comments
- **HUMAN CHECKPOINT**: Final approval

## Escalation Rules
- If any agent is blocked for > 5 minutes: escalate to team lead
- If team lead is blocked for > 10 minutes: escalate to human
- If budget exceeds 80%: pause and notify human
```

---

## 6. 通信与协调协议

### 6.1 消息类型系统

```rust
enum Message {
    /// 任务分配
    TaskAssignment {
        from: AgentId,
        to: AgentId,
        task: Task,
        priority: Priority,
        deadline: Option<Duration>,
    },

    /// 任务结果（含摘要，而非原始 context）
    TaskResult {
        from: AgentId,
        to: AgentId,
        task_id: TaskId,
        summary: String,           // 摘要（context 经济性原则）
        artifacts: Vec<Artifact>,  // 文件变更、测试结果等
        full_log: Option<LogRef>,  // 完整日志的引用（按需拉取）
    },

    /// Peer 消息（@mention）
    DirectMessage {
        from: AgentId,
        to: AgentId,
        content: String,
        reply_to: Option<MessageId>,
    },

    /// 广播（team 范围）
    Broadcast {
        from: AgentId,
        scope: BroadcastScope,  // team-local | parent-team | global
        content: String,
    },

    /// 升级（向上传递问题）
    Escalation {
        from: AgentId,
        reason: EscalationReason,
        context: String,
        suggested_action: Option<String>,
    },

    /// 人类干预请求
    HumanIntervention {
        from: AgentId,
        checkpoint_id: CheckpointId,
        question: String,
        options: Vec<String>,
        context_summary: String,
    },
}
```

### 6.2 共享任务看板（Task Board）

类似物理看板，但支持嵌套 team 的任务可见性控制：

```
┌─────────────────────────────────────────────────────────────┐
│                     Root Team Task Board                     │
├──────────┬──────────┬──────────┬──────────┬────────────────┤
│ Backlog  │ Claimed  │ In Prog  │ Review   │ Done           │
├──────────┼──────────┼──────────┼──────────┼────────────────┤
│          │          │ FE-001   │          │ BE-001 ✓       │
│ INT-002  │          │ (@fe)    │ FE-002   │ (@be-api)      │
│          │ TEST-001 │          │ (@revie) │                │
│          │ (@tester)│ BE-002   │          │ TEST-002 ✓     │
│          │          │ (@be)    │          │ (@tester)      │
└──────────┴──────────┴──────────┴──────────┴────────────────┘

Sub-team "frontend" 内部看板（父 team 看到的是聚合状态）:
┌─────────────────────────────────────────┐
│         Frontend Sub-Team Board          │
├──────────┬──────────┬──────────┬────────┤
│ Todo     │ Working  │ Review   │ Done   │
├──────────┼──────────┼──────────┼────────┤
│ FE-001-c │ FE-001-a │          │FE-001-b│
│(@fe-impl)│(@fe-impl)│          │(@style)│
└──────────┴──────────┴──────────┴────────┘
```

### 6.3 文件所有权与冲突预防

解决 Claude Code Agent Teams 中多 agent 编辑同文件导致覆盖的问题：

```rust
struct FileOwnership {
    /// 显式文件所有权声明（在 team.toml 中定义）
    explicit_owners: HashMap<GlobPattern, AgentId>,
    /// 运行时锁：agent 开始编辑时获取，完成后释放
    active_locks: HashMap<FilePath, Lock>,
    /// 冲突解决策略
    conflict_policy: ConflictPolicy,
}

enum ConflictPolicy {
    /// 锁定：先到先得，后来者等待
    Lock,
    /// 合并：使用 3-way merge 自动合并
    AutoMerge,
    /// 升级：冲突时升级给 orchestrator 或人类
    Escalate,
    /// 分支：每个 agent 在独立 git worktree 中工作（Karpathy 模式）
    GitWorktree,
}
```

### 6.4 通信拓扑

根据 `OrchestrationPattern` 自动配置通信规则：

```
Supervisor 模式:          Hierarchy 模式:           Swarm 模式:
                          
   ┌──────┐                  ┌──────┐              ┌──────┐
   │ Lead │                  │ Lead │              │  A   │◄────►┌──────┐
   └──┬───┘                  └──┬───┘              └──┬───┘      │  B   │
      │                         │                     │          └──┬───┘
   ┌──┴──┬──┐             ┌────┴────┐                │              │
   ▼     ▼  ▼             ▼         ▼             ┌──┴───┐    ┌────┴──┐
  [A]   [B] [C]       ┌──────┐  ┌──────┐         │  C   │◄──►│  D    │
                       │TeamA │  │TeamB │         └──────┘    └───────┘
  (star topology)      │Lead  │  │Lead  │
                       └──┬───┘  └──┬───┘         (mesh topology)
                          │         │
                        [a1][a2]  [b1][b2]

                       (tree topology)
```

---

## 7. 可视化系统

### 7.1 TUI（终端用户界面）

#### 7.1.1 Agent 树视图（主界面）

受 Codex #9902 和 #12047 启发，设计持久化侧边栏：

```
┌─────────────────────────────────────────────────────────────────────┐
│ Codex Agent Team Dashboard                              [?] Help   │
├──────────────────────┬──────────────────────────────────────────────┤
│ 🏠 TEAM: project-a  │  @fe-lead [Frontend Team]                   │
│                      │  ─────────────────────────────────────────── │
│ ▼ @lead ● RUNNING   │  > Implementing responsive layout for the   │
│   ├─ @tester ◌ IDLE  │    dashboard component. Split into 3 sub-  │
│   ├─ @reviewer ○ WAIT│    tasks assigned to @fe-impl instances.   │
│   │                  │                                              │
│   ├─▼ frontend ● RUN │  Tasks:                                     │
│   │  ├─ @fe-lead ●   │  ☑ FE-001-a: Header component              │
│   │  ├─ @fe-impl.1 ● │  ▶ FE-001-b: Sidebar navigation (78%)     │
│   │  ├─ @fe-impl.2 ● │  ☐ FE-001-c: Main content grid            │
│   │  └─ @fe-style ◌  │                                              │
│   │                  │  Files modified:                              │
│   └─▼ backend ● RUN  │  + src/components/Sidebar.tsx               │
│      ├─ @be-lead ●   │  ~ src/styles/layout.css                   │
│      ├─ @be-api ●    │                                              │
│      └─ @be-db ◌     │  Tokens: 12,450 / 50,000  Cost: $0.18     │
│                      │  Duration: 4m 32s                            │
│ ───────────────────  │──────────────────────────────────────────────│
│ Budget: $2.14/$10.00 │  [m]essage  [t]ask  [p]ause  [k]ill       │
│ Tokens: 89K/1M      │  [e]scalate [h]uman  [f]ocus  [l]ogs       │
│ Agents: 9/12 active │                                              │
└──────────────────────┴──────────────────────────────────────────────┘
```

#### 7.1.2 实时状态指示器

```
● RUNNING    ◌ IDLE       ○ WAITING
◉ BLOCKED    ✕ FAILED     ✓ DONE
⊘ CANCELLED  ⏸ PAUSED     ⟳ RETRYING
```

#### 7.1.3 消息流视图

```
┌─────────────────────── Message Stream ──────────────────────────────┐
│ [14:32:01] @lead → @fe-lead: Implement dashboard responsive layout │
│ [14:32:05] @fe-lead → @fe-impl.1: Handle header component FE-001-a │
│ [14:32:05] @fe-lead → @fe-impl.2: Handle sidebar nav FE-001-b      │
│ [14:35:12] @fe-impl.1 → @fe-lead: ✓ FE-001-a complete (summary...) │
│ [14:36:44] @be-api → @be-lead: ⚠ Need DB schema for /api/users     │
│ [14:36:45] @be-lead → @be-db: Priority: define users table schema   │
│ [14:37:01] @lead → HUMAN: 🔔 Budget at 80%. Continue? [Y/n]        │
│ [14:37:15] HUMAN → @lead: Y, increase budget to $15                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Web Dashboard（可选扩展）

对于更复杂的 team 或远程监控场景，提供 Web 界面：

```
┌─────────────────────────────────────────────────────────────────────┐
│  Agent Team Dashboard                    🔴 Live    [Export] [Share]│
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─── Team Topology Graph ───┐  ┌─── Resource Usage ────────────┐  │
│  │                           │  │                                │  │
│  │    ┌──────┐               │  │  Tokens ████████░░ 78%         │  │
│  │    │ @lead│               │  │  Cost   █████░░░░░ 48%         │  │
│  │    └──┬───┘               │  │  Time   ██████░░░░ 55%         │  │
│  │   ┌───┼────────┐         │  │                                │  │
│  │   ▼   ▼        ▼         │  │  Per-agent breakdown:          │  │
│  │  [FE] [BE]  [@tester]    │  │  @fe-impl.1: 15K tokens $0.22 │  │
│  │  /  \  / \               │  │  @fe-impl.2: 12K tokens $0.18 │  │
│  │ a1  a2 b1 b2             │  │  @be-api:    18K tokens $0.27 │  │
│  │                           │  │  ...                           │  │
│  └───────────────────────────┘  └────────────────────────────────┘  │
│                                                                     │
│  ┌─── Task Board (Kanban) ───────────────────────────────────────┐  │
│  │ Backlog │ In Progress    │ In Review      │ Done              │  │
│  │         │                │                │                   │  │
│  │ INT-002 │ FE-001 (78%)  │ FE-002 @review │ BE-001 ✓         │  │
│  │         │ BE-002 (45%)  │                │ TEST-002 ✓        │  │
│  └─────────┴────────────────┴────────────────┴───────────────────┘  │
│                                                                     │
│  ┌─── Timeline / Gantt ─────────────────────────────────────────┐  │
│  │ @lead     ██████████████████████████████████████████████████  │  │
│  │ @fe-lead  ░░░░████████████████████████████░░░░░░░░░░░░░░░░  │  │
│  │ @fe-impl1 ░░░░░░██████████████████░░░░░░░░░░░░░░░░░░░░░░░  │  │
│  │ @fe-impl2 ░░░░░░░████████████████████████░░░░░░░░░░░░░░░░  │  │
│  │ @be-lead  ░░░░████████████████████████████░░░░░░░░░░░░░░░░  │  │
│  │ @be-api   ░░░░░░████████████████████████████████░░░░░░░░░░  │  │
│  │ @tester   ░░░░░░░░░░░░░░░░░░░░░░░░░███████████████████████  │  │
│  │ @reviewer ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████████████  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.3 DAG 提交视图

受 Karpathy AgentHub 启发，展示代码变更的 DAG 结构：

```
┌─── Commit DAG ────────────────────────────────────────┐
│                                                        │
│  main ─── a1 ─── a2 ─── a3 ──────────── MERGE ── m1  │
│            \                              /            │
│             └── @fe-impl.1: b1 ── b2 ───┘             │
│            \                            /              │
│             └── @fe-impl.2: c1 ── c2 ─┘               │
│            \                          /                │
│             └── @be-api: d1 ── d2 ──┘                  │
│                                                        │
│  Legend: [agent color] ■ commit  ─ parent              │
└────────────────────────────────────────────────────────┘
```

---

## 8. 人机交互层

### 8.1 交互模式谱

从 Karpathy "post-AGI" 洞察出发，设计多级人类参与模式：

```
Full Auto          Supervised          Collaborative         Manual
  ◄──────────────────────────────────────────────────────────────►

  Agent swarm       Agent team with     Human as team         Human does
  runs fully        human checkpoints   member alongside      everything,
  autonomous        & approval gates    agents                agent assists
```

```rust
enum HumanInLoopMode {
    /// 全自动：仅在错误时通知人类
    FullAuto {
        notify_on: Vec<NotifyTrigger>,  // error, budget_threshold, completion
    },
    /// 监督模式：关键检查点需要人类批准
    Supervised {
        checkpoints: Vec<Checkpoint>,   // phase transitions, file changes, etc.
        auto_approve_timeout: Option<Duration>,
    },
    /// 协作模式：人类是 team 的一个活跃成员
    Collaborative {
        human_role: AgentRole,          // reviewer, architect, etc.
        can_assign_tasks: bool,
        can_claim_tasks: bool,
    },
    /// 手动模式：agent 仅提供建议，人类执行所有操作
    Manual,
}
```

### 8.2 人类交互点

#### 8.2.1 检查点（Checkpoints）

在关键节点自动暂停，等待人类决策：

```
╔══════════════════════════════════════════════════════╗
║  🔔 CHECKPOINT: Phase 2 → Phase 3 Transition        ║
║                                                      ║
║  Frontend team completed 3/3 tasks                   ║
║  Backend team completed 2/3 tasks (1 in review)      ║
║  Integration tests: 14 passing, 2 failing            ║
║                                                      ║
║  @lead recommends: Proceed to Phase 3 after          ║
║  fixing the 2 failing integration tests              ║
║                                                      ║
║  Options:                                            ║
║  [1] Approve: proceed to Phase 3                     ║
║  [2] Fix first: assign fix tasks, stay in Phase 2    ║
║  [3] Modify plan: (opens editor)                     ║
║  [4] Take over: switch to manual mode                ║
║                                                      ║
║  Auto-approve in: 5:00 (press any key to decide)     ║
╚══════════════════════════════════════════════════════╝
```

#### 8.2.2 实时干预命令

在任何时刻，人类可以通过命令直接干预：

```bash
# 发送消息给特定 agent
codex team msg @fe-lead "优先完成 sidebar，先跳过动画"

# 暂停/恢复特定 agent 或 team
codex team pause @be-db
codex team resume @be-db

# 将任务重新分配给其他 agent
codex team reassign TASK-005 @fe-impl.2

# 将自己加入 team 作为成员
codex team join --role reviewer

# 查看特定 agent 的完整 context
codex team inspect @be-api --show-context

# 调整预算
codex team budget --add 50000 tokens

# 强制 merge 某个 agent 的工作
codex team merge @fe-impl.1

# 创建临时 agent 处理紧急任务
codex team spawn "hotfix-agent" --task "fix critical CSS regression in header"

# 查看 team 整体状态
codex team status --tree
codex team status --gantt
codex team status --board
```

#### 8.2.3 @Human 消息通道

Agent 可以主动与人类通信：

```rust
/// Agent 发起的人类交互
enum HumanRequest {
    /// 需要决策
    Decision {
        question: String,
        options: Vec<String>,
        context: String,
        urgency: Urgency,      // low, medium, high, critical
    },
    /// 需要信息
    Information {
        what_i_need: String,
        why: String,
        what_i_tried: Vec<String>,
    },
    /// 需要批准
    Approval {
        action: String,
        impact: String,
        reversible: bool,
    },
    /// 状态更新
    StatusUpdate {
        progress: f32,
        summary: String,
        blockers: Vec<String>,
    },
}
```

### 8.3 人类作为 Team 成员

在协作模式下，人类是 task board 的参与者：

```
┌── Task Board ──────────────────────────────────────────────┐
│                                                             │
│  Available (unclaimed):                                     │
│  ☐ ARCH-001: Design API schema for notifications  [claim]  │
│  ☐ ARCH-002: Review database migration strategy    [claim]  │
│                                                             │
│  Claimed by you (@human):                                   │
│  ▶ ARCH-003: Define team communication protocol             │
│                                                             │
│  Claimed by agents:                                         │
│  ▶ FE-001: Implement notification UI    (@fe-impl.1)       │
│  ▶ BE-001: Build notification service   (@be-api)          │
│                                                             │
│  [c]laim task  [d]elegate  [r]eview  [a]dd task             │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. 容错与恢复机制

### 9.1 故障分类与响应

```rust
enum FailureType {
    /// Agent 自身崩溃
    AgentCrash {
        agent_id: AgentId,
        error: String,
        recovery: RecoveryAction,
    },
    /// Context window 耗尽
    ContextExhaustion {
        agent_id: AgentId,
        action: ContextAction,  // compact | summarize_and_restart | escalate
    },
    /// 通信超时
    CommunicationTimeout {
        from: AgentId,
        to: AgentId,
        retry_count: u32,
    },
    /// 资源预算耗尽
    BudgetExhausted {
        scope: BudgetScope,    // agent | team | global
        resource: Resource,    // tokens | cost | time
    },
    /// 文件冲突
    FileConflict {
        path: FilePath,
        agents: Vec<AgentId>,
    },
    /// 任务死锁（循环依赖）
    TaskDeadlock {
        cycle: Vec<TaskId>,
    },
}

enum RecoveryAction {
    Retry { max_attempts: u32, backoff: Duration },
    Reassign { to: Option<AgentId> },
    Escalate { to: EscalationTarget },
    Abort,
    SpawnReplacement,
}
```

### 9.2 Session 恢复（解决 Claude Code 的关键限制）

```rust
/// Team 状态快照，支持 session 恢复
struct TeamSnapshot {
    timestamp: DateTime,
    team_config: TeamConfig,
    agent_states: HashMap<AgentId, AgentState>,
    task_board: TaskBoard,
    message_history: Vec<Message>,
    file_changes: Vec<FileChange>,
    git_state: GitState,        // 当前分支、commit、worktree 状态
    budget_consumed: BudgetUsage,
}

trait SessionPersistence {
    /// 定期自动保存 team 快照
    fn auto_checkpoint(&self, interval: Duration);
    /// 从快照恢复 team
    fn restore_from_snapshot(&self, snapshot: TeamSnapshot) -> Team;
    /// 清理孤儿 team 配置（解决 Claude Code #32730）
    fn cleanup_orphaned_teams(&self);
    /// 列出可恢复的历史 session
    fn list_recoverable_sessions(&self) -> Vec<TeamSnapshot>;
}
```

### 9.3 "死信队列"（Dead Letter Queue）

处理无法投递的消息和永久失败的任务：

```
┌── Dead Letter Queue ──────────────────────────────┐
│                                                    │
│ [14:32:15] MSG to @be-db: agent crashed (retry 3x)│
│            → Action: [R]etry [D]rop [E]scalate    │
│                                                    │
│ [14:35:01] TASK FE-003: stuck 10min (agent hung)  │
│            → Action: [R]eassign [K]ill [I]nspect   │
│                                                    │
│ [14:36:22] MERGE conflict: src/utils.ts            │
│            @fe-impl.1 vs @be-api                   │
│            → Action: [M]anual merge [P]ick one     │
└────────────────────────────────────────────────────┘
```

---

## 10. 实现路线图

### Phase 0: 基础设施强化（解决现有 Bug）
**优先级：P0 | 估计：2-3 周**

- [ ] 修复 spawn slot 泄漏（Codex #13947）：实现 RAII-style slot management
- [ ] 实现级联取消：父 agent 终止时正确清理所有子 agent
- [ ] 修复竞态条件（Codex #14318）：带 `parent_turn_id` 的完成通知
- [ ] 修复子 agent 生命周期信号（Codex #14194）：崩溃/挂起的可靠检测
- [ ] 实现 wait-all 语义（替代当前 wait-any）
- [ ] 添加 per-agent timeout 配置

### Phase 1: 命名 Agent 与基础可视化
**优先级：P0 | 估计：3-4 周**

- [ ] 实现 Agent 命名系统：`@name` handle 替代 UUID
- [ ] TUI 侧边栏：agent 树视图 + 状态指示器
- [ ] Agent/thread 归属标识（Codex #11815）
- [ ] 基础消息流视图
- [ ] `codex team status` 命令

### Phase 2: 声明式 Team 配置
**优先级：P1 | 估计：4-5 周**

- [ ] 设计并实现 `team.toml` 格式
- [ ] Team 模板系统（全局 + 项目级）
- [ ] `AGENTS.md` 变体支持（Codex #10067）
- [ ] "Org Code" / program.md 流程定义
- [ ] 文件所有权声明与运行时锁
- [ ] `codex team init` / `codex team create` 命令

### Phase 3: 嵌套 Team 支持
**优先级：P1 | 估计：5-6 周**

- [ ] 嵌套 Team 实体模型与生命周期
- [ ] 子 team 作为黑盒 agent 的接口抽象
- [ ] 跨层级通信路由
- [ ] 嵌套 task board（聚合 + 展开视图）
- [ ] 嵌套资源预算继承与限制
- [ ] 作用域隔离与故障隔离

### Phase 4: 高级通信与协调
**优先级：P1 | 估计：4-5 周**

- [ ] 完整消息类型系统（任务、结果、@mention、广播、升级）
- [ ] 多种编排模式（Supervisor, Hierarchy, Pipeline, Swarm, Hybrid）
- [ ] 基于 git worktree 的文件隔离（Karpathy 模式）
- [ ] 3-way merge 自动冲突解决
- [ ] 通信拓扑自动配置

### Phase 5: 人机交互层
**优先级：P1 | 估计：4-5 周**

- [ ] 检查点（Checkpoint）系统
- [ ] 实时干预命令集（`codex team msg/pause/resume/reassign/inspect`）
- [ ] @Human 消息通道
- [ ] 人类作为 Team 成员的协作模式
- [ ] 可配置的 Human-in-the-Loop 策略

### Phase 6: 高级可视化
**优先级：P2 | 估计：5-6 周**

- [ ] 完整 TUI Dashboard（树视图 + 任务看板 + 消息流 + 资源面板）
- [ ] Gantt 时间线视图
- [ ] DAG commit 视图
- [ ] 可选 Web Dashboard
- [ ] 实时指标与告警

### Phase 7: 容错与恢复
**优先级：P2 | 估计：3-4 周**

- [ ] Session 持久化与恢复
- [ ] 自动 checkpoint 与快照
- [ ] 死信队列
- [ ] 智能重试与重新分配
- [ ] 孤儿 team 清理

### Phase 8: 高级功能
**优先级：P3 | 估计：持续迭代**

- [ ] Agent 模板市场（社区共享）
- [ ] 跨项目 Team（多 repo 协同）
- [ ] Agent 性能分析与优化建议
- [ ] A/B 测试不同编排策略
- [ ] 与 AgentHub 式 DAG 平台集成
- [ ] MCP Server 模式：作为外部编排系统的执行后端

---

## 11. 附录：参考来源

### Claude Code Agent Teams
- [Official Docs: Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)
- [Claude Code Subagents](https://docs.anthropic.com/en/docs/claude-code/subagents)
- [BUG: Subagent-created teams persist on disk (anthropics/claude-code#32730)](https://github.com/anthropics/claude-code/issues/32730)

### OpenAI Codex Issues & PRs
- [#2604: Subagent Support (219+ upvotes, closed as completed)](https://github.com/openai/codex/issues/2604)
- [#2771: Orchestrator agent for delegated multi-agent execution](https://github.com/openai/codex/issues/2771)
- [#3280: Orchestration of multiple agents](https://github.com/openai/codex/issues/3280)
- [#3655: Implement (multi) subagent orchestration system](https://github.com/openai/codex/issues/3655)
- [#8664: Native Subagent System (spawn_subagents / chain_subagents)](https://github.com/openai/codex/issues/8664)
- [#9607: Main agent not receiving subagent completion signal](https://github.com/openai/codex/issues/9607)
- [#9723: Multi-agent collab: sub-agent interruptions + orchestrator prompting](https://github.com/openai/codex/issues/9723)
- [#9846: High-Quality Sub-Agent Collaboration Built into Codex](https://github.com/openai/codex/issues/9846)
- [#9902: Persistent "Agents" sidepanel (active threads overview)](https://github.com/openai/codex/issues/9902)
- [#10067: --agents flag to switch between named AGENTS.md variants](https://github.com/openai/codex/issues/10067)
- [#11815: Show agent/thread attribution on action status rows](https://github.com/openai/codex/issues/11815)
- [#12047: Multi-agent TUI overhaul: named agents, per-agent config, async orchestration](https://github.com/openai/codex/issues/12047)
- [#12460: (referenced agent team implementation)](https://github.com/openai/codex/issues/12460)
- [#13947: Collab subagents leak spawn slots, lack cascading cancellation](https://github.com/openai/codex/issues/13947)
- [#14194: Sub-agents do not reliably signal lifecycle state](https://github.com/openai/codex/issues/14194)
- [#14318: Race condition between subagent async notification and main agent turn](https://github.com/openai/codex/issues/14318)

### OpenAI Codex Official Docs
- [Multi-agents Concept](https://developers.openai.com/codex/concepts/multi-agents/)
- [Multi-agents Guide](https://developers.openai.com/codex/multi-agent/)
- [AGENTS.md Custom Instructions](https://developers.openai.com/codex/guides/agents-md)
- [Customization](https://developers.openai.com/codex/concepts/customization/)

### Karpathy Projects & Commentary
- [AgentHub: Agent-first collaboration platform](https://github.com/karpathy/agenthub)
- [AutoResearch: Autonomous LLM research agent loop](https://github.com/karpathy/autoresearch)
- [Karpathy on "post-AGI" agent workflows (March 2026)](https://xcancel.com/karpathy/status/2004607146781278521)
- [8-Agent Nanochat Research Org analysis](https://blockchain.news/ainews/karpathy-tests-8-agent-nanochat-research-org-claude-and-codex-struggle-with-experiment-design-analysis-and-lessons-for-2026)

### Multi-Agent Architecture Patterns
- [Multi-Agent Orchestration Patterns 2026](https://amirbrooks.com.au/guides/multi-agent-orchestration-patterns)
- [LangGraph Multi-Agent Supervisor Pattern Guide](https://myengineeringpath.dev/genai-engineer/langgraph-multi-agent/)
- [Building Multi-Agent AI Systems 2026](https://aiworkflowlab.dev/article/building-multi-agent-ai-systems-2026-architecture-patterns-mcp-production-orchestration)

---

## 核心创新点总结

本蓝图相比现有方案的关键差异化：

| 维度 | 现状 | 本方案 |
|------|------|--------|
| **嵌套** | 单层 subagent 或扁平 team | 任意深度嵌套 Team，子 team 作为黑盒 agent |
| **配置** | 命令式 API / 简单 AGENTS.md | 声明式 `team.toml` + "Org Code" program.md |
| **可视化** | UUID 标识，无全局视图 | 命名 @handle、树形面板、Kanban、Gantt、DAG |
| **人机交互** | 事后查看结果 | 实时检查点、@Human 通道、人类作为 team 成员 |
| **通信** | 简单 parent↔child | 多模式（star/tree/mesh）、类型化消息、作用域隔离 |
| **容错** | Slot 泄漏、无级联取消 | RAII slot、级联取消、session 恢复、死信队列 |
| **文件管理** | 覆盖冲突 | 所有权声明、运行时锁、git worktree 隔离 |
| **编排模式** | 仅 Supervisor | Supervisor + Hierarchy + Pipeline + Swarm + Hybrid |
| **Context 管理** | Context rot 导致性能退化 | 摘要式通信、按需 context 拉取、分层 context 隔离 |
