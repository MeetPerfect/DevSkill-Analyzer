# DevSkill Analyzer

> 基于 Skills 的代码仓库分析 Agent — 挖掘技术亮点，结论有证据。

DevSkill Analyzer 是一个使用声明式 Skills 组织专业分析流程的代码仓库分析工具。它通过代码检索、文件读取、依赖解析、Git 历史分析等只读工具，对代码仓库进行结构化分析，并输出带有证据引用的分析结果。

## 核心特性

- **声明式 Skills 框架** — 通过标准化的 Skill 定义（`SKILL.md`）组织分析方法，支持发现、加载、匹配和版本管理。新增分析能力只需新增 Skill 文件，无需修改 Runtime。
- **受控的 Tool Calling** — 工具注册、Schema 校验、权限控制、超时和调用预算由 Runtime 统一管理，默认只读。
- **证据驱动的分析结果** — 每条发现关联源码文件路径、行号、符号、提交 SHA 和内容哈希，支持从结论回到原始证据。
- **事实与推断分离** — 通过 `verified` / `inferred` / `needs_review` / `contradicted` 状态降低大模型过度推断。
- **显式多 Skill 编排** — 将复杂任务拆解为可追踪、可测试的分析阶段，支持有限并行和部分失败处理。
- **可重复的评估体系** — 使用固定测试仓库、标注结果和量化指标比较不同实现方案。
- **安全边界清晰** — 仓库内容视为不可信输入，限制文件访问范围、工具调用和潜在命令执行。

## 架构概览

```
用户 / API ──> 任务管理器 ──> Orchestrator 编排器
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Skill Registry   Skill Runtime   Evidence Validator
                    │               │               │
                    ▼               ▼               ▼
              Skill Loader    Context Builder   结果持久化
                              LLM Adapter
                              Tool Registry
                                    │
                    ┌───────┬───────┼───────┬───────┐
                    ▼       ▼       ▼       ▼       ▼
                project  code    code    code     git
                .tree    .search  .read  .references .log
```

### 第一批 Skills

| Skill | 目标 |
|---|---|
| `project-profiler` | 建立仓库画像：语言、框架、依赖、模块与基础设施 |
| `architecture-analyzer` | 模块边界、调用关系、关键数据流 |
| `performance-analyzer` | 缓存、批处理、并发控制、性能优化机制 |
| `reliability-analyzer` | 幂等、重试、超时、异常处理、数据一致性 |
| `evidence-validator` | 校验前序 Skill 输出是否有足够证据支持 |

### 第一批工具

| 工具 | 功能 |
|---|---|
| `project.tree` | 获取仓库目录树（排除 .git、构建产物、大型依赖） |
| `project.detect` | 识别语言、框架、依赖与基础设施 |
| `code.search` | 搜索关键词、符号或模式 |
| `code.read` | 读取指定文件或行范围 |
| `code.references` | 查找符号的引用位置 |
| `git.log` | 查看提交记录和文件变更 |

## 技术栈

| 层次 | 技术选型 |
|---|---|
| 后端 | Go + go-zero |
| 前端 | Vue 3 + TypeScript（可选） |
| 数据库 | PostgreSQL |
| 代码分析 | 文件扫描、正则/符号检索、语言专用 AST |
| 模型接入 | 统一 LLM Adapter（支持结构化输出和工具调用） |
| 部署 | 本地运行或 Docker Compose |

## 项目结构

```
devskill-analyzer/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── api/                  # HTTP API
│   ├── task/                 # 任务创建、状态和取消
│   ├── orchestrator/         # 多 Skill 编排
│   ├── skill/
│   │   ├── registry.go       # Skill 发现与索引
│   │   ├── loader.go         # Skill 加载
│   │   ├── parser.go         # SKILL.md 元数据解析
│   │   ├── matcher.go        # Skill 匹配
│   │   └── runtime.go        # Skill 执行生命周期
│   ├── context/              # 上下文构建与 Token 预算
│   ├── llm/                  # 模型适配层
│   ├── tool/
│   │   ├── registry.go
│   │   ├── project_tree.go
│   │   ├── project_detect.go
│   │   ├── code_search.go
│   │   ├── code_read.go
│   │   ├── code_references.go
│   │   └── git_log.go
│   ├── evidence/              # 证据模型与采集
│   ├── validator/             # 输出和证据校验
│   ├── storage/               # 数据持久化
│   └── config/
├── skills/
│   ├── project-profiler/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── architecture-analyzer/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── performance-analyzer/
│   │   └── SKILL.md
│   ├── reliability-analyzer/
│   │   └── SKILL.md
│   └── evidence-validator/
│       └── SKILL.md
├── configs/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── evals/
│       ├── repositories/
│       ├── cases/
│       └── expected/
├── docs/
├── go.mod
└── README.md
```

## 快速开始

> **注意：** 项目当前处于设计阶段，以下为规划中的使用方式。

### 环境要求

- Go 1.22+
- PostgreSQL 15+
- 支持工具调用的 LLM API（如 OpenAI、Claude 等）

### 本地运行

```bash
# 克隆仓库
git clone https://github.com/<your-org>/devskill-analyzer.git
cd devskill-analyzer

# 配置环境变量
cp configs/config.example.yaml configs/config.yaml
# 编辑 config.yaml，填入数据库连接和 LLM API 密钥

# 启动依赖服务
docker compose up -d postgres

# 启动服务
go run cmd/server/main.go -c configs/config.yaml
```

### API 使用示例

```bash
# 1. 登记项目
curl -X POST http://localhost:8080/api/projects \
  -H "Content-Type: application/json" \
  -d '{"name": "my-project", "repoPath": "/srv/repos/my-project"}'

# 2. 创建分析任务
curl -X POST http://localhost:8080/api/projects/{projectId}/analysis-tasks \
  -H "Content-Type: application/json" \
  -d '{"analysisTypes": ["architecture", "performance", "reliability"], "branch": "main"}'

# 3. 查询任务状态
curl http://localhost:8080/api/analysis-tasks/{taskId}

# 4. 查询分析发现
curl http://localhost:8080/api/analysis-tasks/{taskId}/findings

# 5. 查询发现的证据
curl http://localhost:8080/api/findings/{findingId}/evidence
```

## 分析流程

```
扫描仓库 ──> project-profiler
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
 architecture  performance  reliability
       └─────────┼─────────┘
                 ▼
          合并发现与证据
                 │
                 ▼
        evidence-validator
                 │
                 ▼
        去重与冲突处理
                 │
                 ▼
          生成最终报告
```

- 项目画像先执行，为后续 Skill 提供语言、框架和目录上下文。
- 架构、性能、可靠性分析在资源允许时并行执行。
- 证据校验在分析结果合并后执行。
- 证据不足的结论不会被无条件标记为已验证。

## 设计原则

- **Skill 与 Runtime 分离** — Skill 描述"如何分析"，Runtime 负责"如何安全执行"。
- **渐进式加载** — 发现阶段只加载元数据，激活阶段加载完整 Skill，按需加载参考资料。
- **证据优先** — 每条重要发现关联文件路径、行号、符号和提交版本。
- **事实与推断分离** — `verified` / `inferred` / `needs_review` / `contradicted`。
- **默认只读与最小权限** — 仓库内容属于不可信输入，不能通过其中的指令覆盖安全策略。

## 开发路线

| 阶段 | 交付内容 | 验收标准 |
|---|---|---|
| 一 | Skill Registry 与 Loader | 新增 Skill 文件后无需修改 Runtime 即可发现并读取 |
| 二 | Tool Registry 与 Runtime | Runtime 能完成模型—工具循环并拒绝未授权调用 |
| 三 | 项目画像与核心分析 | 对固定仓库生成可追溯的项目画像和技术发现 |
| 四 | 多 Skill 编排与证据校验 | 完整运行"扫描 → 分析 → 校验 → 报告"流程 |
| 五 | 评估与 Web UI | 可重复运行评估并比较不同 Skill 或模型版本 |

## 评估指标

| 指标 | 含义 |
|---|---|
| Skill 匹配准确率 | 是否选择了适合任务的 Skill |
| 发现精确率 | 输出发现中得到人工确认的占比 |
| 发现召回率 | 已标注的重要发现中被识别的占比 |
| 证据覆盖率 | 有有效证据关联的发现占比 |
| 无依据结论率 | 缺乏证据或超出证据范围的结论占比 |
| 任务完成率 | 成功完成的任务比例 |
| 平均耗时 | 单任务平均执行时间 |
| 平均成本 | 单任务 Token 与模型调用成本 |

## 参考资源

- [设计文档](./DevSkill_Analyzer_设计文档.md) — 完整技术设计方案
- [Anthropic Skills](https://github.com/anthropics/skills)
- [OpenAI Skills](https://github.com/openai/skills)
- [VoltAgent Awesome Agent Skills](https://github.com/VoltAgent/awesome-agent-skills)
- [Superpowers](https://github.com/obra/superpowers)

## License

MIT
