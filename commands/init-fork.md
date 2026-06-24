---
description: Initialize a fork for long-term downstream maintenance with upstream tracking and a downstream change ledger
argument-hint: "<UPSTREAM_URL>"
---

Execute the init-fork workflow for the current repository. Treat this prompt template's argument as the required upstream repository URL:

```bash
UPSTREAM_URL="$ARGUMENTS"
```

If no upstream URL was provided, stop with: `No upstream URL provided. Usage: /init-fork <upstream-url>`.

# Init Fork Command

Set up a freshly cloned or long-lived fork for downstream maintenance with a structured change ledger and upstream tracking. Run this once, right after creating or cloning the fork. After that, `/upstream-sync` handles ongoing upstream release syncs.

The workflow:

1. Configures the `upstream` git remote.
2. Discovers the upstream baseline tag that best represents where the fork started.
3. Creates or updates `.upstream-version`.
4. Creates `DOWNSTREAM_CHANGES.md`, the source of truth for fork-only modifications.
5. Optionally pre-populates the ledger from existing fork-only commits.
6. Appends a Fork Maintenance section to `AGENTS.md`.
7. Commits the fork maintenance infrastructure.

## Inputs and Invariants

- Required argument: upstream repo URL, HTTPS or SSH.
- Current directory: a checked-out clone of the fork.
- `HEAD` must be on a real development branch, not detached.
- The working tree must be clean before initialization.
- Ask before overwriting existing `.upstream-version` or materially rewriting an existing ledger.
- Keep the Fork Maintenance section at the end of `AGENTS.md`.

## 1. Validate Environment

```bash
if [[ -z "$UPSTREAM_URL" ]]; then
  echo "No upstream URL provided. Usage: /init-fork <upstream-url>" >&2
  exit 2
fi

REPO_ROOT="$(git rev-parse --show-toplevel)"
cd "$REPO_ROOT"
DEV_BRANCH="$(git rev-parse --abbrev-ref HEAD)"

if [[ "$DEV_BRANCH" == "HEAD" ]]; then
  echo "Error: detached HEAD; check out the fork's development branch first." >&2
  exit 1
fi

if [[ -n "$(git status --porcelain)" ]]; then
  echo "Error: working tree is dirty; commit or stash changes before initializing fork maintenance." >&2
  exit 1
fi
```

## 2. Set Up the Upstream Remote

```bash
if git remote get-url upstream >/dev/null 2>&1; then
  EXISTING_URL="$(git remote get-url upstream)"
  if [[ "$EXISTING_URL" != "$UPSTREAM_URL" ]]; then
    echo "Updating 'upstream' remote: $EXISTING_URL -> $UPSTREAM_URL"
    git remote set-url upstream "$UPSTREAM_URL"
  else
    echo "Remote 'upstream' already points to $UPSTREAM_URL"
  fi
else
  git remote add upstream "$UPSTREAM_URL"
  echo "Added 'upstream' remote: $UPSTREAM_URL"
fi

git fetch upstream --tags
```

## 3. Discover the Baseline Upstream Tag

The fork was created from upstream at some point. Find the upstream tag that best represents the fork's actual starting point, so `.upstream-version` starts from a truthful baseline rather than blindly recording the latest tag.

Prefer the nearest version-like ancestor tag to `HEAD`, not the numerically largest reachable tag. Some upstream projects have backport tags, branch-specific tags, calendar versions, or irregular numbering where the largest visible tag is not the closest historical baseline.

```bash
TAG_PATTERNS=("refs/tags/v[0-9]*" "refs/tags/[0-9]*")
DESCRIBE_MATCHES=(--match "v[0-9]*" --match "[0-9]*")

LATEST_TAG="$(git -c versionsort.suffix=-pre \
  for-each-ref --sort=-v:refname --format='%(refname:short)' \
  "${TAG_PATTERNS[@]}" \
  --count=1)"

NEAREST_ANCESTOR_TAG="$(git describe --tags --abbrev=0 \
  "${DESCRIBE_MATCHES[@]}" HEAD 2>/dev/null || true)"

HIGHEST_ANCESTOR_TAG=""
while read -r tag; do
  [[ -z "$tag" ]] && continue
  if git merge-base --is-ancestor "refs/tags/$tag" HEAD 2>/dev/null; then
    HIGHEST_ANCESTOR_TAG="$tag"
    break
  fi
done < <(git -c versionsort.suffix=-pre \
  for-each-ref --sort=-v:refname --format='%(refname:short)' \
  "${TAG_PATTERNS[@]}")

BASELINE_TAG="$NEAREST_ANCESTOR_TAG"
```

If `NEAREST_ANCESTOR_TAG` and `HIGHEST_ANCESTOR_TAG` both exist and differ, pause and ask the user which baseline to record:

```text
Warning: the nearest ancestor tag differs from the highest version-like ancestor tag.

  Nearest ancestor tag:      <NEAREST_ANCESTOR_TAG>
  Highest version-like tag:  <HIGHEST_ANCESTOR_TAG>

The nearest ancestor is usually safest because it reflects where this fork sits in the commit graph.
Choose the baseline to record:
  1. nearest ancestor (recommended)
  2. highest version-like ancestor
  3. another tag/manual value
```

Do not continue until the user chooses. Record the selected value in `BASELINE_TAG`.

If no baseline tag is found:

- If `LATEST_TAG` exists, warn that no upstream tag was found in fork ancestry and set `BASELINE_TAG="$LATEST_TAG"`. Tell the user to verify the baseline before the first `/upstream-sync` run.
- If upstream has no version-like tags, set `BASELINE_TAG="none"`. The first `/upstream-sync` run will handle future tags when they appear.

## 4. Create `.upstream-version`

If `.upstream-version` already exists, show its current contents and ask before overwriting it. Explain that `/upstream-sync` relies on this file.

Write the current format:

```bash
cat > .upstream-version <<EOF
UPSTREAM_REPO=$UPSTREAM_URL
UPSTREAM_VERSION=$BASELINE_TAG
EOF
```

## 5. Create `DOWNSTREAM_CHANGES.md`

If `DOWNSTREAM_CHANGES.md` already exists, do not overwrite it. Verify that it is recognizable as the downstream change ledger and continue.

If it does not exist, create it with this header:

```markdown
# Downstream Changes

This file is the source of truth for all intentional fork-only modifications.
The upstream-sync command reads it during merge conflict resolution to preserve
fork behavior. Update this file every time you add, modify, or remove a
fork-only change.

---

<!-- Add new entries below using the format described in AGENTS.md. -->
```

## 5b. Detect and Populate Existing Downstream Changes

The fork may already contain commits that do not exist upstream. Discover those commits, present them to the user, and offer to generate ledger entries. This is especially useful for forks maintained informally before adopting this workflow.

Find fork-only commits:

```bash
if [[ -n "$BASELINE_TAG" && "$BASELINE_TAG" != "none" ]] && \
   git rev-parse -q --verify "refs/tags/$BASELINE_TAG" >/dev/null; then
  FORK_COMMITS="$(git log --format='%H %s' refs/tags/"$BASELINE_TAG"..HEAD --no-merges)"
else
  FORK_COMMITS="$(git log --format='%H %s' --no-merges)"
fi
```

If `BASELINE_TAG` is `none` or does not resolve to a local tag, explain that all commits appear fork-only and the user should review carefully.

If fork-only commits exist, display them:

```text
Found N fork-only commits (not present upstream at BASELINE_TAG):

  abc1234  Add custom telemetry opt-out
  def5678  Patch config defaults for internal deployment
  9012345  Remove upstream analytics tracking

Would you like to populate DOWNSTREAM_CHANGES.md from these commits?
Choose all, none, or specific commits by number/hash.
```

For each selected commit:

1. Inspect `git show --stat <sha>` and `git show <sha>`.
2. Classify the change as `patch`, `feature`, `config`, `override`, or `removal`.
3. Generate a slug from the purpose, such as `feat-telemetry-optout`.
4. Determine scope from the file paths.
5. Group closely related commits into one logical ledger entry when appropriate.

Append generated entries before the HTML comment. Use this format:

```markdown
## [slug-id]: Short description

- **Scope**: `path/to/file`
- **Type**: patch | feature | config | override | removal
- **Status**: active
- **Introduced**: <sha> (or multiple SHAs for grouped entries)
- **Superseded by upstream**: N/A

### What this changes

<description inferred from the commit message and diff>

### Files affected

- `path/to/file`: <summary of what changed>
```

If no fork-only commits are found or the user declines, leave the ledger empty and ready for future entries.

## 6. Append Fork Maintenance to `AGENTS.md`

Read `AGENTS.md` if it exists. If a `## Fork Maintenance` section already exists, verify it is compatible and skip appending. Otherwise append this section at the very end of the file. If `AGENTS.md` does not exist, create it with only this section.

````markdown

---

## Fork Maintenance

This repository is a **soft fork** of an upstream project. It tracks upstream
tagged releases and carries a small number of intentional downstream changes.

### Maintenance files

| File | Purpose |
|------|---------|
| `.upstream-version` | Tracks the upstream repo URL and the last merged upstream tag. Written by `/upstream-sync` on every sync. |
| `DOWNSTREAM_CHANGES.md` | Ledger of all fork-only modifications. Read by `/upstream-sync` during conflict resolution to preserve downstream behavior. |

### DOWNSTREAM_CHANGES.md format

Every entry in `DOWNSTREAM_CHANGES.md` follows this structure:

```markdown
## [slug-id]: Short description

- **Scope**: `path/to/file` (or comma-separated paths)
- **Type**: patch \| feature \| config \| override \| removal
- **Status**: active \| superseded \| removed
- **Introduced**: <commit-sha, tag, or date>
- **Superseded by upstream**: <upstream-version or N/A>

### What this changes

Plain-English description of what the fork does differently from upstream and why.

### Files affected

- `path/to/file`: what was changed (function names, config keys, line ranges)
```

**Type values:**
- `patch` - bug fix applied downstream ahead of upstream
- `feature` - new functionality not present upstream
- `config` - configuration changes, default values, feature flags
- `override` - behavior replacement where the fork supersedes upstream
- `removal` - upstream code intentionally removed or disabled in the fork

**Status values:**
- `active` - currently in effect
- `superseded` - upstream now implements equivalent functionality; entry kept for history
- `removed` - change reverted; entry kept for history

### When making downstream changes

Every time you make a fork-only modification, add or update an entry in
`DOWNSTREAM_CHANGES.md`. Without it, `/upstream-sync` has no way to know which
changes to preserve during upstream merges.

When `/upstream-sync` detects that an upstream release implements the same
feature or fix as a downstream change, it will update the entry's status to
`superseded` and note the upstream version.

<!-- IMPORTANT: This section must remain at the end of AGENTS.md. Do not move it or add content after it. -->
````

## 7. Commit the Fork Infrastructure

Stage only the files that were created or changed:

```bash
git add .upstream-version DOWNSTREAM_CHANGES.md AGENTS.md
git commit -m "chore: initialize fork maintenance infrastructure

- Add .upstream-version tracking upstream at $BASELINE_TAG
- Add DOWNSTREAM_CHANGES.md ledger (<N> entries pre-populated from fork history / empty, ready for entries)
- Add Fork Maintenance section to AGENTS.md

Paired with /upstream-sync for ongoing upstream release tracking."
```

Adjust the commit body based on what happened. If a file was already present and unchanged, do not stage it and do not claim it was created.

## 8. Report

Print a concise summary:

```text
Fork initialized for downstream maintenance.

  Upstream remote:  <UPSTREAM_URL>
  Baseline tag:     <BASELINE_TAG>
  Dev branch:       <DEV_BRANCH>

Created or updated:
  .upstream-version       - tracking file read by /upstream-sync
  DOWNSTREAM_CHANGES.md   - change ledger (<N> entries from existing fork history / empty)
  AGENTS.md               - Fork Maintenance section appended or verified

Next steps:
  1. Review pre-populated entries in DOWNSTREAM_CHANGES.md, if any.
  2. Document every future fork-only change in DOWNSTREAM_CHANGES.md.
  3. Run /upstream-sync periodically to pull upstream releases.
```

## Edge Cases

- Existing `AGENTS.md`: append the Fork Maintenance section at the end unless it already exists.
- Repository not pushed to a remote host: continue. This command only touches local files and remote config.
- Existing `.upstream-version`: ask before overwriting.
- Nearest ancestor tag differs from highest numbered ancestor tag: warn and ask the user which baseline to record.
- No upstream tags: record `UPSTREAM_VERSION=none` and continue.
- No shared history with upstream: warn prominently; the first `/upstream-sync` run will also detect this.
- Many fork-only commits: if there are more than about 30, warn that the list is large and group related commits aggressively.
- Merge commits in fork history: exclude merge commits from the fork-only list; non-merge commits represent intentional downstream changes.
