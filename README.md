# Maintainer Workflows

[中文](README.zh-CN.md)

Prompt-template commands for maintaining long-lived forks and reverse-engineering existing codebases. **init-fork** sets up fork infrastructure, **upstream-sync** keeps forks in sync with upstream releases, and **codebase-archaeologist** extracts reconstruction-ready specifications from unfamiliar code.

## Commands

### `/upstream-sync`

Sync a fork with the latest tagged release from an upstream repository.

- Fetches the newest semver tag from upstream
- Merges into the dev branch via an isolated git worktree
- Resolves merge conflicts intelligently, using `DOWNSTREAM_CHANGES.md` as the source of truth for fork-specific behavior
- Detects when upstream supersedes a downstream change and proposes ledger updates
- Prints the proposed review description and upstream changelog for review
- Either merges locally after approval or pushes a branch for remote review with upstream release notes
- Tracks state in `.upstream-version` — no external database

Use from pi as `/upstream-sync [URL]`.

### `/init-fork`

Initialize a soft fork for long-term downstream maintenance. Run once after cloning the fork.

- Configures the `upstream` git remote
- Creates `.upstream-version` with the discovered baseline tag
- Creates `DOWNSTREAM_CHANGES.md` — the structured ledger of fork-only modifications
- Appends a Fork Maintenance section to `AGENTS.md`
- Detects existing fork-only commits and offers to pre-populate the ledger

Pairs with `/upstream-sync` for ongoing maintenance.

### `/codebase-archaeologist`

Reverse-engineer an existing codebase into a reconstruction-ready specification package.

- Maps the project structure, entry points, domain logic, data model, and interfaces
- Produces `spec.md`, `plan.md`, `data-model.md`, `research.md`, `completeness-report.md`, and `contracts/`
- Cites evidence from source files and distinguishes observed behavior from inference
- Scales analysis depth for small, medium, large, and enterprise codebases

Use from pi as `/codebase-archaeologist [CODEBASE_PATH]`.

## Quick Start

```bash
# 1. Clone the fork
git clone https://git.example.com/your-group/your-fork.git
cd your-fork

# 2. Initialize the fork from pi (one-time)
/init-fork https://github.com/upstream/project.git

# 3. Sync with upstream from pi (run periodically or interactively)
/upstream-sync
```

## File Structure

```
commands/
  codebase-archaeologist.md  # pi prompt template for codebase reconstruction specs
  init-fork.md               # pi prompt template for fork initialization
  upstream-sync.md           # pi prompt template for the sync workflow
.pi/prompts -> ../commands   # project prompt-template discovery symlink
```

## Maintained Artifacts

The fork-maintenance commands create and manage:

| File | Purpose |
|------|---------|
| `.upstream-version` | Tracks the upstream repo URL and last merged tag |
| `DOWNSTREAM_CHANGES.md` | Ledger of all fork-only modifications |
| `AGENTS.md` (Fork Maintenance section) | Declares the repo as a fork and documents the ledger format |

The archaeology command writes reconstruction specs under `codebase-archaeology/[project-name-slug]/`.

## License

Internal use.
