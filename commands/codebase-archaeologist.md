---
description: Reverse-engineer a codebase into a reconstruction-ready specification package
argument-hint: "[CODEBASE_PATH]"
---

Analyze an existing codebase and produce a complete reconstruction-oriented specification package. Use the optional argument as the codebase path; if omitted, analyze the current repository.

```bash
TARGET_CODEBASE="${ARGUMENTS:-.}"
```

# Codebase Archaeologist Command

You are reverse-engineering an existing codebase from a project manager's perspective. Produce the same kinds of artifacts that spec-kit's forward workflow would need, but derive them from the actual code.

Your goal is not just to describe files. Capture the intent, design, domain model, user-facing behavior, operational assumptions, and interfaces well enough that another engineer could rebuild equivalent functionality without reading the original code.

## Core Principles

- Start with why, not just what. Look for business rules, product decisions, and operational constraints embedded in code.
- Preserve the mental model. Explain how the system is meant to be used and maintained.
- Cite evidence. Reference concrete files and line numbers for important claims.
- Separate observation from inference. Mark uncertain conclusions as `[UNKNOWN]` or `[NEEDS CLARIFICATION]`.
- Scale the depth of review to codebase size while documenting what was sampled or skipped.

## Phase 1: Reconnaissance

Begin from `TARGET_CODEBASE`:

```bash
cd "$TARGET_CODEBASE"
CODEBASE_ROOT="$(pwd)"
```

Run a quick terrain scan:

```bash
# Count common source files
find . -type f \( -name "*.py" -o -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.go" -o -name "*.rs" -o -name "*.java" -o -name "*.cpp" -o -name "*.c" -o -name "*.rb" -o -name "*.php" -o -name "*.cs" \) 2>/dev/null | wc -l

# Directory structure, top levels
find . -maxdepth 3 -type d | sort | head -80

# Project metadata
ls -la package.json pnpm-lock.yaml yarn.lock package-lock.json pyproject.toml setup.py requirements.txt Cargo.toml go.mod pom.xml build.gradle settings.gradle composer.json Gemfile Dockerfile docker-compose.yml *.yaml *.yml *.toml *.json 2>/dev/null
```

Classify the project:

- Library/SDK
- CLI tool
- Web service/API
- Web application
- Mobile app
- Compiler/interpreter
- DevOps/infrastructure tool
- Data pipeline
- Other or hybrid

List source files with `rg --files` where available. Prefer `rg` for searching.

## Phase 2: Domain Analysis

Read entry points first:

- CLI: `main.*`, `cli.*`, `cmd/`, `commands/`, executable scripts
- Web services: route handlers, controllers, API definitions
- Libraries: package exports, public API files, `__init__.py`, `index.ts`, `lib.rs`
- Applications: app bootstrap files, UI route/component roots
- Jobs/pipelines: workflow definitions, DAGs, workers, queue consumers

Extract:

- Domain entities and relationships
- User-facing capabilities
- Business rules and validations
- State machines and workflow transitions
- Error handling and edge cases
- Permissions, authentication, authorization, and security-relevant checks

## Phase 3: Technical Analysis

Identify and document:

- Runtime languages and versions
- Primary dependencies and their purpose
- Storage systems, schemas, models, migrations, and file formats
- External services and protocols
- Configuration files and environment variables
- Build, test, deployment, and local development workflows
- Architecture and component boundaries
- Concurrency, background jobs, caching, and performance-sensitive paths
- Public interfaces: CLI commands, API endpoints, events, file formats, protocols, and exported library APIs

## Phase 4: Generate the Specification Package

Create output under:

```text
codebase-archaeology/[project-name-slug]/
```

Use a stable slug derived from the repository or package name.

Required output files:

```text
codebase-archaeology/[project-name-slug]/
├── spec.md
├── plan.md
├── data-model.md
├── research.md
├── completeness-report.md
└── contracts/
    ├── cli-commands.md        # when applicable
    ├── api-endpoints.md       # when applicable
    ├── file-formats.md        # when applicable
    ├── protocols.md           # when applicable
    └── internal-apis.md       # when applicable
```

Create only relevant contract files, but make sure every public interface is documented somewhere in `contracts/`.

## `spec.md` Structure

```markdown
# Feature Specification: [PROJECT NAME]

**Type**: [CLI tool / Web service / Library / etc.]
**Analyzed**: [DATE]
**Source**: [path to analyzed codebase]

## Summary

[What this project does in 2-3 sentences]

## User Scenarios & Testing

### User Story 1 - [Capability] (Priority: P1)
[What users can do with this software]

**Independent Test**: [How to verify this works]

**Acceptance Scenarios**:
1. **Given** [initial state], **When** [action], **Then** [outcome]

### User Story 2 - [Capability] (Priority: P2)
[...]

### Edge Cases
[Error handling and edge cases discovered in code]

## Non-Functional Requirements
[Performance, security, scalability, compatibility, and operational constraints]

## Success Criteria
[Measurable criteria that indicate the rebuilt feature is complete]
```

## `plan.md` Structure

````markdown
# Implementation Plan: [PROJECT NAME]

**Branch**: `[reconstruction]` | **Date**: [DATE] | **Spec**: spec.md
**Input**: Reverse-engineered from [source codebase path]

## Summary
[Primary requirement and technical approach]

## Technical Context
**Language/Version**: [detected from code]
**Primary Dependencies**: [main packages/libraries]
**Storage**: [database/file storage if applicable]
**Testing**: [test framework used]
**Target Platform**: [where it runs]
**Project Type**: [library/cli/web-service/mobile-app/etc.]
**Performance Goals**: [observed or inferred]
**Constraints**: [inferred from code patterns]
**Scale/Scope**: [estimated size/complexity]

## Constitution Check
*GATE: Would this pass spec-kit's constitution gates?*

### Simplicity Gate
- [ ] Using <=3 projects? [Assess actual project structure]
- [ ] No future-proofing? [Check for over-engineering]

### Anti-Abstraction Gate
- [ ] Using framework directly? [Check for unnecessary abstraction]
- [ ] Single model representation? [Check for multiple models of same entity]

### Integration-First Gate
- [ ] Contracts defined? [Documented in contracts/]
- [ ] Tests use real environments? [Assess test patterns]

## Project Structure

### Documentation
```text
codebase-archaeology/[project-name-slug]/
├── spec.md
├── plan.md
├── data-model.md
├── research.md
├── contracts/
└── completeness-report.md
```

### Source Code
[Document the actual source code structure]

## Architecture
[How components interact]

## Key Technical Decisions
| Decision | Rationale (observed or inferred) | Evidence in Code |
|----------|----------------------------------|------------------|
| [Choice] | [Why it was made] | [File/line reference] |

## Complexity Tracking
| Violation | Why Needed (inferred) | Simpler Alternative Rejected Because (inferred) |
|-----------|----------------------|-----------------------------------------------|
````

## `data-model.md` Structure

```markdown
# Data Model: [PROJECT NAME]

## Entities

### [Entity Name]
**Purpose**: [What it represents]
**Location**: [File path where defined]

**Fields**:
| Field | Type | Description |
|-------|------|-------------|
| name | string | [description] |

**Relationships**: [how it relates to other entities]

## Data Flows
[How data moves through the system]

## State Machines
[Any stateful entities with transitions]

## Constraints
[Business rules enforced on data]
```

## `research.md` Structure

```markdown
# Research Notes: [PROJECT NAME]

## Dependencies Analysis
| Dependency | Version | Purpose | Usage Location |
|------------|---------|---------|----------------|
| [name] | [version] | [what it does] | [where used] |

## Configuration
- Environment variables required
- Config files and formats
- Secrets and credential handling

## External Integrations
| Service | Protocol | Purpose | Authentication |
|---------|----------|---------|----------------|
| [name] | [type] | [why] | [how] |

## Performance Characteristics
[Observed or inferred performance characteristics]

## Security Considerations
[Security patterns, vulnerabilities, or concerns discovered]

## Known Issues / Technical Debt
[Code quality issues discovered during analysis]
```

## Contracts

Create contract files appropriate to the project:

- `cli-commands.md`: commands, subcommands, flags, examples, exit behavior
- `api-endpoints.md`: HTTP/RPC endpoints, methods, paths, schemas, auth, responses
- `file-formats.md`: input/output files, schemas, examples, validation rules
- `protocols.md`: custom protocols, events, queues, message formats
- `internal-apis.md`: important internal module/service boundaries

Each contract should include signatures, parameters, return values, side effects, errors, and evidence links.

## `completeness-report.md` Structure

```markdown
# Analysis Completeness Report

**Codebase**: [path]
**Analysis Date**: [date]
**Total Source Files**: [count]

## Coverage by Type

| Category | Files | Fully Reviewed | Partially Reviewed | Skipped |
|----------|-------|----------------|---------------------|---------|
| Entry points | N | N | N | N |
| Domain logic | N | N | N | N |
| Infrastructure | N | N | N | N |
| Tests | N | N | N | N |
| Config/boilerplate | N | N | N | N |

## Components Documented
[List of all major components/subsystems identified]

## Gaps and Unknowns
[Anything that could not be fully analyzed or requires clarification]

## Verification Checklist
- [ ] All entry points documented
- [ ] All public interfaces in contracts/
- [ ] All data entities in data-model.md
- [ ] All major dependencies listed
- [ ] Configuration requirements documented
- [ ] External integrations documented

## Notes
[Any observations about code quality, structure, or patterns]
```

## Depth by Codebase Size

Small codebases (<100 source files): read everything comprehensively.

Medium codebases (100-1000 source files): read all entry points and main modules fully. For supporting modules, sample strategically, search for key patterns, and read files with unique business logic.

Large codebases (>1000 source files): map architecture first, identify core subsystems, sample representative files, and focus on public APIs, data models, business logic, and unique algorithms. Mark partially reviewed areas clearly.

Enterprise codebases (>10k files): apply tiered analysis:

1. Entry points and public interfaces: read fully.
2. Core domain logic: sample thoroughly.
3. Infrastructure/platform: sample lightly.
4. Test code: review only where it reveals behavior not obvious from implementation.

## Proofreading and Verification

After generating all documents, proofread them against the codebase:

1. Coverage check: every major component appears in at least one output document.
2. Consistency check: `spec.md` user stories map to actual functionality found in code.
3. Dependency check: `research.md` lists all major dependencies discovered.
4. Interface check: `contracts/` documents all public interfaces.
5. Data model check: `data-model.md` captures all important entities and relationships.
6. Completeness report: coverage numbers and gaps are honest.

If you discover gaps during proofreading, add `[NEEDS CLARIFICATION]` markers in relevant documents, note the gap in `completeness-report.md`, and explain what additional analysis would resolve it.

## Final Response

Report completion with:

- Summary of what was discovered
- File count analyzed
- Key findings
- List of output files created
- Any `[UNKNOWN]` or `[NEEDS CLARIFICATION]` items
