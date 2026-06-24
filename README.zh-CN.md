# Maintainer Workflows（维护工作流）

[English](README.md)

用于维护长期 Fork 和逆向分析现有代码库的 pi prompt template 命令集合。**init-fork** 负责初始化 Fork 基础设施，**upstream-sync** 负责持续同步上游发布版本，**codebase-archaeologist** 负责从陌生代码库提取可重建规格。

## 命令

### `/upstream-sync`

将 Fork 与上游仓库的最新标签发布版本同步。

- 从上游获取最新的语义化版本标签
- 通过隔离的 git worktree 合并到开发分支
- 智能解决合并冲突，以 `DOWNSTREAM_CHANGES.md` 作为 Fork 特定行为的真实性来源
- 检测上游是否已取代某个下游变更，并提议更新账本
- 打印拟定的审阅说明和上游变更日志供审阅
- 经确认后本地合并，或推送远程审阅分支并附带上游发布说明
- 在 `.upstream-version` 中追踪状态——无需外部数据库

在 pi 中通过 `/upstream-sync [URL]` 使用。

### `/init-fork`

初始化一个用于长期下游维护的软 Fork。克隆 Fork 后运行一次即可。

- 配置 `upstream` git 远程仓库
- 创建 `.upstream-version`，记录发现的基线标签
- 创建 `DOWNSTREAM_CHANGES.md`——结构化的 Fork 独有修改账本
- 在 `AGENTS.md` 中追加 Fork 维护章节
- 检测已有的 Fork 独有提交，提供预填充账本选项

与 `/upstream-sync` 配合用于持续维护。

### `/codebase-archaeologist`

将现有代码库逆向分析为可用于重建的规格文档包。

- 梳理项目结构、入口点、领域逻辑、数据模型和接口
- 生成 `spec.md`、`plan.md`、`data-model.md`、`research.md`、`completeness-report.md` 和 `contracts/`
- 引用源码证据，并区分已观察到的行为与推断
- 根据小型、中型、大型和企业级代码库调整分析深度

在 pi 中通过 `/codebase-archaeologist [CODEBASE_PATH]` 使用。

## 快速开始

```bash
# 1. 克隆 Fork
git clone https://git.example.com/your-group/your-fork.git
cd your-fork

# 2. 在 pi 中初始化 Fork（一次性）
/init-fork https://github.com/upstream/project.git

# 3. 在 pi 中与上游同步（定期运行或交互式运行）
/upstream-sync
```

## 文件结构

```
commands/
  codebase-archaeologist.md  # 代码库重建规格的 pi prompt template
  init-fork.md               # 初始化工作流的 pi prompt template
  upstream-sync.md           # 同步工作流的 pi prompt template
.pi/prompts -> ../commands   # 项目 prompt template 发现用符号链接
```

## 维护的产物文件

Fork 维护命令会创建和管理以下文件：

| 文件 | 用途 |
|------|------|
| `.upstream-version` | 追踪上游仓库 URL 和最后合并的标签 |
| `DOWNSTREAM_CHANGES.md` | 所有 Fork 独有修改的账本 |
| `AGENTS.md`（Fork 维护章节） | 声明仓库为 Fork 并记录账本格式 |

代码库考古命令会将重建规格写入 `codebase-archaeology/[project-name-slug]/`。

## 许可证

内部使用。
