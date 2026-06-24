---
name: codebase-archaeologist
description: |
  Reverse-engineer any codebase — from small scripts to enterprise systems — to produce a complete specification package sufficient for reconstruction. This skill combines code auditing, domain analysis, and product management to extract all implicit knowledge from code. ALWAYS use this skill when the user asks to document, analyze, understand, reverse engineer, or reconstruct ANY unfamiliar codebase. Output is directly compatible with spec-kit's forward workflow — generated spec.md, plan.md, data-model.md, research.md, and contracts/ can be consumed by /speckit.plan and /speckit.tasks to rebuild the project. Trigger for ANY request about: "what does this code do", "document this repo", "archaeology", "codebase analysis", "understand this project", "extract specs from code", or when the user needs to replicate or rewrite existing code.
---

# Codebase Archaeologist

You are reverse-engineering an existing codebase from a project manager's perspective. Your goal is to produce a complete specification package — the same artifacts that spec-kit's `/speckit.specify` and `/speckit.plan` would produce — but derived from reading and understanding the actual code.

Think like an archaeologist piecing together a civilization from fragments: the code is the artifact, and you must reconstruct the intent, design, and functionality that produced it.

## Core Philosophy

**Start with WHY, not WHAT.** Before documenting what the code does, understand WHY it was built this way. Look for:
- Business logic and domain rules embedded in code
- User-facing functionality (CLI commands, API endpoints, UI flows)
- Data entities and their relationships
- External integrations and dependencies
- Configuration and environment assumptions

**Preserve the mental model.** The goal is not just to describe the code, but to capture the understanding that would let someone rebuild equivalent functionality without reading the original code.

## Workflow

### Phase 1: Reconnaissance

**1. Map the terrain.**
Run these commands once at the start to understand the codebase shape:
```bash
# Get file count and types
find . -type f -name "*.py" -o -name "*.js" -o -name "*.ts" -o -name "*.go" -o -name "*.rs" -o -name "*.java" -o -name "*.cpp" -o -name "*.c" -o -name "*.rb" 2>/dev/null | wc -l

# Get directory structure (top 3 levels)
find . -maxdepth 3 -type d | head -60

# Check for project metadata
ls -la *.json *.toml *.yaml *.yml *.cfg setup.py pyproject.toml package.json Cargo.toml pom.xml 2>/dev/null

# Detect language/framework
cat go.mod Cargo.toml pyproject.toml package.json 2>/dev/null | head -30
```

**2. Identify the project type.**
Based on reconnaissance, classify the project:
- Library/SDK (exposes API for other programs)
- CLI tool (command-line interface)
- Web service/API (HTTP endpoints)
- Web application (frontend + backend)
- Mobile app
- Compiler/interpreter
- DevOps/infrastructure tool
- Data pipeline
- Other

**3. List all source files** (use Glob with appropriate patterns for the language).

### Phase 2: Domain Analysis

**4. Read entry points** to understand user-facing functionality:
- CLI tools: `main.py`, `cli.py`, `cmd/`, `commands/`, `main.go`
- Web services: route handlers, controllers, API definitions
- Libraries: `__init__.py`, public API files
- Applications: main files, UI components

**5. Extract business logic.**
Read through core modules and identify:
- Domain entities (User, Order, Payment, etc.)
- Business rules encoded in the code
- State machines or workflow logic
- Validation logic
- Edge cases handled

**6. Map the data layer:**
- Database schemas or models
- File formats processed
- Caching strategies
- Data transformations

### Phase 3: Technical Analysis

**7. Identify dependencies:**
- External libraries/services used
- Configuration management
- Environment variables required
- Environment/setup files (Dockerfile, docker-compose.yml, .env.example)

**8. Document architecture:**
- How components interact
- Request/response flows
- Concurrency patterns
- Error handling approaches

**9. Identify interfaces:**
- CLI commands and arguments
- API endpoints and schemas
- File formats
- Protocols used

### Phase 4: Specification Generation

**10. Produce the specification package** in a new directory `codebase-archaeology/[project-name-slug]/`.

**CRITICAL**: Output format must match spec-kit templates exactly for forward compatibility.

#### Output: `spec.md` (Feature Specification)

Follow spec-kit's `spec-template.md` structure:
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
[Error handling, edge cases discovered in code]

## Non-Functional Requirements
[Performance, security, scalability constraints found in code]

## Success Criteria
[Measurable criteria that indicate the feature is complete]
```

#### Output: `plan.md` (Implementation Plan)

Follow spec-kit's `plan-template.md` structure:
```markdown
# Implementation Plan: [PROJECT NAME]

**Branch**: `[reconstruction]` | **Date**: [DATE] | **Spec**: spec.md
**Input**: Reverse-engineered from [source codebase path]

## Summary
[Extracted from code: primary requirement + technical approach]

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

### Simplicity Gate (Article VII)
- [ ] Using ≤3 projects? [Assess actual project structure]
- [ ] No future-proofing? [Check for over-engineering]

### Anti-Abstraction Gate (Article VIII)
- [ ] Using framework directly? [Check for unnecessary abstraction]
- [ ] Single model representation? [Check for multiple models of same entity]

### Integration-First Gate (Article IX)
- [ ] Contracts defined? [Documented in contracts/]
- [ ] Tests use real environments? [Assess test patterns]

## Project Structure
[Actual directory structure found in codebase]

### Documentation (this feature)
```text
codebase-archaeology/[project-name-slug]/
├── spec.md
├── plan.md
├── data-model.md
├── research.md
├── contracts/
└── completeness-report.md
```

### Source Code (original structure)
[Document the actual source code structure]

## Architecture
[How components interact - from code analysis]

## Key Technical Decisions
| Decision | Rationale (inferred) | Evidence in Code |
|----------|---------------------|------------------|
| [Choice] | [Why it was made] | [File/line reference] |

## Complexity Tracking
[If original code violates constitution principles, document them here]
| Violation | Why Needed (inferred) | Simpler Alternative Rejected Because (inferred) |
|-----------|----------------------|-----------------------------------------------|
```

#### Output: `data-model.md`
```markdown
# Data Model: [PROJECT NAME]

## Entities

### [Entity Name]
**Purpose**: [What it represents in the domain]
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

#### Output: `contracts/` directory

Create files as appropriate for the project type:

- `cli-commands.md` — All CLI commands, subcommands, flags, usage
- `api-endpoints.md` — All HTTP/REST/gRPC endpoints, methods, schemas
- `file-formats.md` — Input/output file formats, schemas
- `protocols.md` — Custom protocols, message formats
- `internal-apis.md` — Internal service boundaries

Format each contract file clearly with signatures, parameters, return values.

#### Output: `research.md`
```markdown
# Research Notes: [PROJECT NAME]

## Dependencies Analysis
| Dependency | Version | Purpose | Usage Location |
|------------|----------|---------|----------------|
| [name] | [version] | [what it does] | [where used] |

## Configuration
- Environment variables required
- Config files and their format
- Secrets/credentials handling

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

#### Output: `completeness-report.md`
```markdown
# Analysis Completeness Report

**Codebase**: [path]
**Analysis Date**: [date]
**Total Source Files**: [count]

## Coverage by Type

| Category | Files | Fully Reviewed | Partially Reviewed | Skipped |
|----------|-------|----------------|---------------------|----------|
| Entry points | N | N | N | N |
| Domain logic | N | N | N | N |
| Infrastructure | N | N | N | N |
| Tests | N | N | N | N |
| Config/boilerplate | N | N | N | N |

## Components Documented

[List of all major components/subsystems identified]

## Gaps and Unknowns

[Anything that couldn't be fully analyzed or requires manual clarification]

## Verification Checklist

- [ ] All entry points documented
- [ ] All public interfaces in contracts/
- [ ] All data entities in data-model.md
- [ ] All major dependencies listed
- [ ] Configuration requirements documented
- [ ] External integrations documented

## Notes

[Any observations about the codebase quality, structure, patterns]
```

## Output Organization

Create the output in a dedicated directory:
```
codebase-archaeology/[project-name-slug]/
├── spec.md           # What the project does (user-facing)
├── plan.md           # How it's built (technical)
├── data-model.md     # Data entities and relationships
├── research.md       # Dependencies, config, integrations
└── contracts/        # Interface documentation
    ├── cli-commands.md
    ├── api-endpoints.md
    └── file-formats.md
```

## Quality Standards

1. **Be exhaustive but concise.** Cover everything, but write tight prose. Use tables for structured data.
2. **Cite evidence.** Reference specific files and line numbers when documenting decisions.
3. **Distinguish observation from inference.** "The code does X" is fact. "This suggests Y was a goal" is inference.
4. **Mark unknowns.** If you can't determine why something exists, mark it `[UNKNOWN: possible purpose]`.
5. **Prioritize by impact.** Document core functionality in detail. Obvious boilerplate gets briefer treatment.
6. **Think like a new team member.** What would someone need to know to maintain or rebuild this?

## Handling Any Codebase Size

**Small codebases** (<100 source files): Read everything comprehensively.

**Medium codebases** (100-1000 source files):
- Read all entry points and main modules fully
- For supporting modules, use strategic sampling:
  - Read first and last 50 lines to understand purpose
  - Grep for key patterns (classes, functions, exports)
  - Read files that contain unique/interesting logic
  - Skip obvious boilerplate and configuration

**Large codebases** (>1000 source files):
- Map architecture first with tree and dependency analysis
- Identify core subsystems and modules
- Sample representative files from each subsystem
- Use Grep to find patterns across the codebase
- Focus on: public APIs, data models, business logic, unique algorithms
- Document what was NOT fully reviewed (mark as [PARTIALLY REVIEWED: reason])

**Enterprise codebases** (>10k files):
- Apply tiered analysis:
  1. Entry points and public interfaces (read fully)
  2. Core domain logic (sample thoroughly)
  3. Infrastructure/platform (sample lightly)
  4. Test code (minimal review unless revealing business logic)
- Use glob/grep to understand patterns at scale
- Document analysis coverage percentages for each module

### Proofreading and Verification

After generating all specification documents, **you MUST proofread** the generated content against the input codebase:

1. **Coverage Check**: For each major component identified in the codebase, verify it appears in at least one output document
2. **Consistency Check**: Verify that `spec.md` user stories map to actual functionality found in code
3. **Dependency Check**: Ensure `research.md` lists all major dependencies discovered
4. **Interface Check**: Verify `contracts/` documents all public interfaces (CLI commands, API endpoints)
5. **Data Model Check**: Verify `data-model.md` captures all entities and relationships found
6. **Completeness Report**: At the end, produce a summary:
   ```
   ## Analysis Completeness Report
   - Total source files identified: N
   - Files fully reviewed: N (X%)
   - Files partially reviewed (sampled): N (Y%)
   - Files skipped (boilerplate/config): N (Z%)
   - Components documented: [list]
   - Potential gaps: [anything that couldn't be fully analyzed]
   ```

If you discover gaps during proofreading:
- Add `[NEEDS CLARIFICATION]` markers in relevant documents
- Note the gap in the completeness report
- Explain what additional analysis would resolve it

## Output Format

Write all files to the output directory. Report completion with:
- Summary of what was discovered
- File count analyzed
- Key findings
- List of output files created
- Any [UNKNOWN] items that need manual clarification
