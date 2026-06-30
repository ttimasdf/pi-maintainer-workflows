---
description: Sync a fork with the latest tagged upstream release, review the changelog, then merge locally or open a review request
argument-hint: "[UPSTREAM_URL]"
---

Execute the upstream-sync workflow for the current repository. Treat this prompt template's argument as an optional upstream repository URL:

```bash
ARG_URL="$ARGUMENTS"
```

If no argument was provided, infer the URL from the existing `upstream` git remote. Work pragmatically and perform the sync unless a step fails or the user chooses to stop.

# Upstream Sync Command

Keep a long-lived fork in sync with tagged releases from an upstream repository. The workflow:

1. Determines the target development branch from the currently checked-out branch.
2. Fetches tags from the upstream repository.
3. Compares the latest upstream tag with `.upstream-version`.
4. Creates an isolated worktree and sync branch.
5. Merges the upstream tag, resolving conflicts with awareness of `DOWNSTREAM_CHANGES.md`.
6. Updates `.upstream-version` and any confirmed downstream ledger changes.
7. Prints the proposed review description and upstream changelog.
8. Asks whether to merge locally, push a remote review branch, or stop for manual inspection.

## Invariants

- Do not modify the current dev branch during preparation. All sync work happens inside a git worktree on `upstream-sync/<tag>`.
- The local dev branch may only be updated in step 10 after the user explicitly chooses the local merge path.
- If `DOWNSTREAM_CHANGES.md` exists, treat it as the source of truth for intentional fork-only behavior.
- If `.upstream-version` is missing, bootstrap carefully by discovering the fork's nearest upstream ancestor tag.
- Prefer tag-based releases over upstream branches. Tags give reviewers a discrete unit of change.
- In non-interactive or CI-style runs, do not merge locally. Default to preparing and pushing a remote review branch when credentials are available.

## 1. Validate Environment

Run these checks from the repository root:

```bash
REPO_ROOT="$(git rev-parse --show-toplevel)"
cd "$REPO_ROOT"
DEV_BRANCH="$(git rev-parse --abbrev-ref HEAD)"
```

Abort if `DEV_BRANCH` is `HEAD`; the workflow needs a real branch as the review target or local merge target.

Confirm the main worktree is clean before preparation:

```bash
if [[ -n "$(git status --porcelain)" ]]; then
  echo "Error: working tree is dirty; refusing upstream sync." >&2
  exit 1
fi
```

## 2. Resolve Upstream URL

The prompt argument wins when provided. Otherwise use the existing `upstream` remote.

```bash
EXISTING_URL="$(git remote get-url upstream 2>/dev/null || true)"

if [[ -n "$ARG_URL" ]]; then
  UPSTREAM_URL="$ARG_URL"
  if [[ -n "$EXISTING_URL" ]]; then
    if [[ "$EXISTING_URL" != "$UPSTREAM_URL" ]]; then
      git remote set-url upstream "$UPSTREAM_URL"
    fi
  else
    git remote add upstream "$UPSTREAM_URL"
  fi
elif [[ -n "$EXISTING_URL" ]]; then
  UPSTREAM_URL="$EXISTING_URL"
else
  echo "Error: no upstream URL provided and no 'upstream' git remote configured." >&2
  echo "Pass a URL to /upstream-sync or run 'git remote add upstream <url>' first." >&2
  exit 2
fi

git fetch upstream --tags --prune
```

## 3. Find Latest Upstream Tag

Prefer the highest semver-style tag, not the most recently created tag.

```bash
LATEST_TAG="$(git -c versionsort.suffix=-pre \
  for-each-ref --sort=-v:refname --format='%(refname:short)' \
  'refs/tags/v[0-9]*' 'refs/tags/[0-9]*' \
  --count=1)"
```

Use `v[0-9]*`, not `v*`, to avoid auxiliary tags such as `vscode-v0.0.13`. If the upstream uses a different release-tag scheme, adapt the pattern deliberately.

If no semver-like tag is found, fall back to:

```bash
LATEST_TAG="${LATEST_TAG:-$(git describe --tags "$(git rev-list --tags --max-count=1)" 2>/dev/null || true)}"
```

If still empty, report `upstream has no tags` and exit successfully. A tagless upstream is not a sync error.

Filter out pre-release tags containing `-alpha`, `-beta`, `-rc`, or `-pre` unless the existing `.upstream-version` already points at a pre-release.

## 4. Compare Against `.upstream-version`

Read `.upstream-version` when present:

- Current format:

  ```dotenv
  UPSTREAM_REPO=<repo-url.git>
  UPSTREAM_VERSION=<vN.NN.N>
  ```

- Legacy format: first line is the upstream version.

Rules:

- If the file is current-format, read `UPSTREAM_VERSION` as `LAST_TAG`. If `UPSTREAM_REPO` differs from `UPSTREAM_URL`, prefer `UPSTREAM_URL` and mention the URL change in the review body.
- If the file is legacy one-line format, set `LEGACY_UPSTREAM_VERSION=1`, read the first line as `LAST_TAG`, and rewrite the file in current format during step 8.
- If the file is missing, run the first-run bootstrap below.
- If `LAST_TAG == LATEST_TAG` and the tracking file is already current with the resolved upstream URL, print `Already at latest upstream tag: $LATEST_TAG` and stop.
- If `LAST_TAG == LATEST_TAG` but the tracking file is missing, legacy-formatted, or has a different upstream URL, create the sync branch and make only the bookkeeping update. Do not merge upstream again.
- Otherwise continue to merge `LAST_TAG..LATEST_TAG`.

### First-Run Bootstrap

The bootstrap discovers which upstream tag the fork is already based on. Do not blindly record the latest tag, because that can hide unmerged upstream history from future syncs.

Find the nearest version-like ancestor tag and compare it with the highest version-like ancestor tag:

```bash
TAG_PATTERNS=("refs/tags/v[0-9]*" "refs/tags/[0-9]*")
DESCRIBE_MATCHES=(--match "v[0-9]*" --match "[0-9]*")

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

LAST_TAG="$NEAREST_ANCESTOR_TAG"
BOOTSTRAP_BASELINE_SOURCE="nearest ancestor"
BOOTSTRAP_BASELINE_WARNING=""
BOOTSTRAPPED=1
```

If `UPSTREAM_SYNC_BASELINE_TAG` is set, use it as `LAST_TAG` and record `BOOTSTRAP_BASELINE_SOURCE="UPSTREAM_SYNC_BASELINE_TAG override"`.

If nearest and highest ancestor tags differ, explain the mismatch clearly. In an interactive run, ask the user to choose the nearest ancestor, highest version-like ancestor, or a manual tag. In a non-interactive run, default to the nearest ancestor and mention the warning in the review body.

Validate any chosen baseline tag exists locally. If no baseline tag is found, set `LAST_TAG="$LATEST_TAG"`, record `BOOTSTRAP_BASELINE_SOURCE="no shared ancestor fallback: latest tag"`, and prominently warn that no shared upstream tag was found in fork ancestry.

## 5. Create Worktree and Sync Branch

```bash
SYNC_BRANCH="upstream-sync/$LATEST_TAG"
WORKTREE_DIR="$REPO_ROOT/.git-upstream-sync-worktree"

if [[ -d "$WORKTREE_DIR" ]]; then
  git worktree remove --force "$WORKTREE_DIR" 2>/dev/null || true
fi

git worktree add -b "$SYNC_BRANCH" "$WORKTREE_DIR" HEAD
cd "$WORKTREE_DIR"
```

If the branch already exists locally, it is probably from a prior failed run. Remove the stale worktree/branch and recreate it:

```bash
git branch -D "$SYNC_BRANCH" 2>/dev/null || true
git worktree remove --force "$WORKTREE_DIR" 2>/dev/null || true
git worktree add -b "$SYNC_BRANCH" "$WORKTREE_DIR" HEAD
cd "$WORKTREE_DIR"
```

If the branch already exists on `origin`, a remote review request may already exist. Prefer updating the existing request when the hosting provider's CLI or API makes that straightforward; otherwise push the refreshed branch and report that the user should update the existing request manually.

## 6. Merge Upstream Tag

```bash
git merge --no-ff --no-edit "refs/tags/$LATEST_TAG" \
  -m "Merge upstream tag $LATEST_TAG into $DEV_BRANCH"
```

If histories are unrelated, retry with `--allow-unrelated-histories` only when `.upstream-version` does not exist. Otherwise stop and report the problem.

## 7. Resolve Conflicts Intelligently

If the merge reports conflicts, do not blindly accept one side. For each conflicted file:

- Read `DOWNSTREAM_CHANGES.md` first, when present. Search for the conflicted path, related feature names, config keys, and behavior from the fork-side diff.
- Inspect fork-side changes with `git log --oneline $LAST_TAG..HEAD -- <path>` and `git diff $LAST_TAG..HEAD -- <path>`.
- Inspect upstream changes with `git log --oneline $LAST_TAG..$LATEST_TAG -- <path>` and `git diff $LAST_TAG..$LATEST_TAG -- <path>`.

Resolution guidance:

- Pure additions on both sides: keep both.
- Upstream refactored something the fork edited: prefer upstream's structure and port the fork's intent into it.
- Documented downstream-only behavior: preserve it unless upstream now implements the same feature or policy.
- Both sides implemented the same feature differently: prefer upstream as canonical, then preserve any downstream-specific configuration/defaults that still matter.
- Upstream deleted a file the fork modified: lean toward deletion, but flag it prominently in the review body.
- Upstream renamed a file the fork modified: apply the rename and carry fork edits into the renamed file.
- Lockfiles/generated files: take upstream's version, then regenerate with the project's normal install/build command when obvious.
- Formatting-only conflicts: take upstream.

After each resolution, verify the file parses or at least reads sensibly, then `git add` it. When all conflicts are resolved, run:

```bash
git commit --no-edit
```

Keep a concise conflict log for the review body. Separate ordinary conflict notes from downstream supersession warnings.

## 8. Update Tracking File

```bash
cat > .upstream-version <<EOF
UPSTREAM_REPO=$UPSTREAM_URL
UPSTREAM_VERSION=$LATEST_TAG
EOF

git add .upstream-version
git commit -m "chore: record upstream sync to $LATEST_TAG"
```

Keep this as a separate commit from the merge so reviewers can distinguish bookkeeping from upstream changes.

## 9. Review `DOWNSTREAM_CHANGES.md`

Skip this step if `DOWNSTREAM_CHANGES.md` is missing or has no `Status: active` entries.

For each active entry, check whether upstream changes in `LAST_TAG..LATEST_TAG` affect it:

1. Read the entry's scope and affected files.
2. Check whether upstream touched those files.
3. Inspect upstream diffs to classify overlap as superseded, partially superseded, unrelated, or removed.
4. Also look for removed entries when upstream deleted a file or conflict resolution took upstream entirely.

Present findings before modifying the ledger:

```text
The following DOWNSTREAM_CHANGES.md entries may need updating after this upstream sync:

1. [slug-id]: Short description
   -> Superseded by upstream (v<LATEST_TAG>): upstream now implements <summary>
   Proposed: Status -> superseded, Superseded by upstream -> <LATEST_TAG>

Apply these updates? (all / none / select specific entries)
```

Only edit `DOWNSTREAM_CHANGES.md` for entries the user confirms. In non-interactive mode, do not modify the ledger automatically; include proposed updates in the review description instead.

For confirmed updates:

- Superseded: set `Status` to `superseded` and `Superseded by upstream` to `$LATEST_TAG`.
- Partially superseded: update the description/scope to show what remains fork-specific; keep `Status: active`.
- Removed: set `Status` to `removed` and note when/why the change disappeared.

Commit confirmed edits:

```bash
git add DOWNSTREAM_CHANGES.md
git commit -m "chore: update DOWNSTREAM_CHANGES.md for upstream $LATEST_TAG sync"
```

## 10. Compose Review Material and Choose Merge Path

Before pushing or merging, compose the proposed review description and upstream changelog locally.

Generate a git diff patch file in `/tmp` for each upstream tag interval included in `LAST_TAG..LATEST_TAG`, then write a changelog grounded in those patch files. Use hosted upstream release notes only as supporting material below the patch-grounded changelog.

```bash
UPSTREAM_GH_REPO="$(printf '%s\n' "$UPSTREAM_URL" | sed -E 's#^https://github.com/##; s#^git@github.com:##; s#\.git$##')"
PATCH_DIR="/tmp/upstream-sync-patches"
PATCH_CHANGELOG_FILE="/tmp/upstream-sync-patch-changelog.md"
UPSTREAM_PROVIDED_CHANGELOG_FILE="/tmp/upstream-release-notes.md"
UPSTREAM_CHANGELOG_FILE="/tmp/upstream-sync-changelog.md"
mkdir -p "$PATCH_DIR"

mapfile -t INCLUDED_TAGS < <(git -c versionsort.suffix=-pre tag \
  --merged "refs/tags/$LATEST_TAG" \
  --no-merged "refs/tags/$LAST_TAG" \
  --sort=v:refname 'v[0-9]*' '[0-9]*')

PREV_TAG="$LAST_TAG"
for tag in "${INCLUDED_TAGS[@]}"; do
  PATCH_FILE="$PATCH_DIR/${PREV_TAG}..${tag}.patch"
  git diff --binary "refs/tags/$PREV_TAG" "refs/tags/$tag" > "$PATCH_FILE"
  PREV_TAG="$tag"
done
```

Write `$PATCH_CHANGELOG_FILE` from the generated patch files before consulting hosted release notes:

- For each tag interval, read the corresponding patch file from `$PATCH_DIR` and summarize the actual code, docs, tests, build, dependency, and config changes visible in the diff.
- Ground every changelog item in the patch content. Do not invent changes from tag names, commit subjects, or hosted release notes.
- Group related changes by impact area when useful, and call out potentially breaking behavior, migrations, removed APIs/files, dependency updates, and security-sensitive changes that are visible in the patch.
- If a patch is too large to inspect fully, say which patch file was generated and summarize only the portions inspected.

Fetch or compose hosted upstream release notes for every included tag, then attach them under a subheading below the patch-grounded changelog:

```bash
{
  echo "## Upstream-provided changelog: $LAST_TAG -> $LATEST_TAG"
  echo
  for tag in "${INCLUDED_TAGS[@]}"; do
    echo "### $tag"
    if [[ "$UPSTREAM_URL" == *"github.com"* ]] && gh release view "$tag" -R "$UPSTREAM_GH_REPO" --json body,url --jq 'if .body then .body + "\n\n" + .url else .url end' 2>/dev/null; then
      true
    else
      echo "No hosted release notes found for this tag."
      echo "Tag: $tag"
    fi
    echo
  done
} > "$UPSTREAM_PROVIDED_CHANGELOG_FILE"
```

After `$PATCH_CHANGELOG_FILE` and `$UPSTREAM_PROVIDED_CHANGELOG_FILE` are ready, combine them into `$UPSTREAM_CHANGELOG_FILE` with the patch-grounded changelog first and the upstream-provided changelog under `### Upstream-provided changelog`.

If hosted release notes cannot be fetched from the provider, fall back to compare/release/tag links where possible. Do not fail the sync because hosted release notes cannot be fetched.

The final upstream changelog should have this structure:

```markdown
## Upstream changelog: <LAST_TAG> -> <LATEST_TAG>

### Patch-grounded changelog
<Changelog written from /tmp/upstream-sync-patches/*.patch.>

### Upstream-provided changelog
<Contents of /tmp/upstream-release-notes.md, or fallback links.>
```

Write `/tmp/upstream-sync-review.md` with this structure:

```markdown
## Upstream sync: <LAST_TAG> -> <LATEST_TAG>

Automated sync of upstream tag `<LATEST_TAG>` into `<DEV_BRANCH>`.

### Bootstrap notes
<Only include when BOOTSTRAPPED=1. Explain baseline source and warnings.>

### Upstream changelog
Patch-grounded changelog is printed for review before integration and attached to the remote review request when one is created. Summary/link: <compare URL or tag/release URL if derivable>

### Conflicts resolved
<Conflict log, or "Clean merge, no conflicts.">

### Downstream changes superseded by upstream
<Supersession warnings, or "None.">

### DOWNSTREAM_CHANGES.md updates
<Applied or proposed ledger updates, or "No ledger updates needed.">

### Reviewer checklist
- [ ] Conflict resolutions preserve fork-specific behavior
- [ ] CI passes
- [ ] `.upstream-version` records `UPSTREAM_REPO=<UPSTREAM_URL>` and `UPSTREAM_VERSION=<LATEST_TAG>`
- [ ] Upstream release notes/changelog were reviewed or attached
- [ ] `DOWNSTREAM_CHANGES.md` entries are accurate

Generated by /upstream-sync.
```

Print both files before asking what to do:

```bash
echo "=== UPSTREAM SYNC REVIEW DESCRIPTION ==="
cat /tmp/upstream-sync-review.md
echo
echo "=== UPSTREAM CHANGELOG ==="
cat "$UPSTREAM_CHANGELOG_FILE"
```

In an interactive run, ask the user to choose:

1. **Merge locally**: fast-forward the local dev branch to the prepared sync branch.
2. **Push for remote review**: push the sync branch and create or update a provider-native review request if project tooling is available.
3. **Stop without merging**: leave the worktree and sync branch in place for manual inspection.

In non-interactive mode, choose the remote-review path.

### Local Merge Path

Use this only after explicit user approval:

```bash
cd "$REPO_ROOT"
if [[ -n "$(git status --porcelain)" ]]; then
  echo "Error: dev worktree is dirty; refusing local merge." >&2
  exit 1
fi

git merge --ff-only "$SYNC_BRANCH"
```

If fast-forward fails, stop. Recommend pushing for remote review or rerunning from the updated dev branch. Do not reopen conflict resolution in the dev worktree.

### Remote Review Path

Push the prepared sync branch:

```bash
git -C "$WORKTREE_DIR" push --set-upstream origin "$SYNC_BRANCH" --force-with-lease
```

Then create or update a review request using the repository's normal hosting workflow. Detect and use available project tooling when it is already configured, for example a repository-specific CLI command, project script, or documented contribution command. Use `/tmp/upstream-sync-review.md` as the description body and `$UPSTREAM_CHANGELOG_FILE` as the changelog attachment/comment content when the provider supports it.

If no hosting workflow is obvious, do not guess. Report the pushed branch, the target branch, and the prepared files:

```text
Pushed branch: <origin>/<SYNC_BRANCH>
Target branch: <DEV_BRANCH>
Review description: /tmp/upstream-sync-review.md
Upstream changelog: <UPSTREAM_CHANGELOG_FILE>
```

## 11. Cleanup

After a successful local merge or after the remote review branch has been pushed and no local inspection is needed, remove the worktree:

```bash
cd "$REPO_ROOT"
git worktree remove --force "$WORKTREE_DIR"
```

If the user chose to stop for manual inspection, leave the worktree in place and print `$WORKTREE_DIR`.

For dry runs or early exits, clean up stale worktrees unless the user asked to inspect them:

```bash
git worktree remove --force "$(git rev-parse --show-toplevel)/.git-upstream-sync-worktree" 2>/dev/null || true
```

## Dry-Run Mode

If `UPSTREAM_SYNC_DRY_RUN=1`, do the local preparation through review body/changelog composition, but skip the interactive prompt, local merge, push, remote review creation, and remote comments. Print:

```text
=== DRY RUN REVIEW BODY ===
<contents of /tmp/upstream-sync-review.md>

=== DRY RUN UPSTREAM CHANGELOG ===
<contents of /tmp/upstream-sync-changelog.md>
```

Then clean up the worktree and exit successfully.

## Report

Finish with one clear summary line:

- `Merged locally into <branch> at <tag>`
- `Pushed <branch> for remote review of upstream sync <tag>`
- `Stopped before merge; sync branch left at <branch>`
- `Already up to date at <tag>`

Also mention any unresolved concerns, skipped release notes, unsafe conflicts, or ledger updates that still need human review.
