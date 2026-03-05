---
name: pr-triage
description: >
  3-phase PR backlog management: audit open PRs, deep review selected ones, draft and post
  review comments with mandatory validation. Args: "all" to review all, PR numbers to focus
  (e.g. "42 57"), "en"/"fr" for language, no arg = audit only in French.
tags: [github, pr, triage, review, maintainer, multi-agent]
---

# PR Triage

3-phase workflow for maintainers: automated audit of all open PRs, opt-in deep review via parallel agents, and validated comment posting.

## When to Use This Skill

| Skill | Usage | Output |
|-------|-------|--------|
| `/pr-triage` | Sort, review, and comment on a PR backlog | Triage table + reviews + posted comments |
| `/review-pr` | Review a single PR in depth | Inline PR review |

**Triggers**:
- Manually: `/pr-triage` or `/pr-triage all` or `/pr-triage 42 57`
- Proactively: when >5 PRs open without review, or stale PR >14 days detected

---

## Language

- Check the argument passed to the skill
- If `en` or `english` → tables and summary in English
- If `fr`, `french`, or no argument → French (default)
- Note: GitHub comments (Phase 3) are ALWAYS in English (international audience)

---

## Configuration

Thresholds used throughout the workflow. Edit to match your project:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `staleness_days` | 14 | Days without activity before flagging as stale |
| `overlap_threshold` | 50% | Shared files % to flag as overlapping |
| `cluster_min_prs` | 3 | Author PR count to trigger cluster suggestion |
| `xl_cutoff_additions` | 1000 | Additions above which a PR is classified XL |
| `xl_cutoff_files` | 10 | Changed files above which a PR is "too large" |

---

## Preconditions

```bash
git rev-parse --is-inside-work-tree
gh auth status
```

If either fails, stop and explain what is missing.

---

## Phase 1 — Audit (always executed)

### Data Gathering (parallel commands)

```bash
# Repo identity
gh repo view --json nameWithOwner -q .nameWithOwner

# Open PRs with full metadata (body for cross-reference issues)
gh pr list --state open --limit 50 \
  --json number,title,author,createdAt,updatedAt,additions,deletions,changedFiles,isDraft,mergeable,reviewDecision,statusCheckRollup,body

# Collaborators (to distinguish internal from external PRs)
gh api "repos/{owner}/{repo}/collaborators" --jq '.[].login'
```

**Collaborators fallback**: if `gh api .../collaborators` returns 403/404:
```bash
# Extract authors from last 10 merged PRs
gh pr list --state merged --limit 10 --json author --jq '.[].author.login' | sort -u
```
If still ambiguous, ask via `AskUserQuestion`.

For each PR, fetch existing reviews AND changed files:

```bash
gh api "repos/{owner}/{repo}/pulls/{num}/reviews" \
  --jq '[.[] | .user.login + ":" + .state] | join(", ")'

# Changed files (required for overlap detection)
gh pr view {num} --json files --jq '[.files[].path] | join(",")'
```

**Rate-limiting note**: fetching files requires N API calls (1 per PR). For repos with 20+ PRs, prioritize candidates for overlap detection (same functional domain, same author).

**Note**: `author` is an object `{login: "..."}` — always extract `.author.login`.

### Analysis

**Size classification**:
| Label | Additions |
|-------|-----------|
| XS | < 50 |
| S | 50–200 |
| M | 200–500 |
| L | 500–1000 |
| XL | > 1000 |

Size format: `+{additions}/-{deletions}, {files} files ({label})`

**Detections**:
- **Overlaps**: compare file lists across PRs — if >50% files in common → cross-reference
- **Clusters**: author with 3+ open PRs → suggest review order (smallest first)
- **Staleness**: no activity for >14 days → flag "stale"
- **CI status**: via `statusCheckRollup` → `clean` / `unstable` / `dirty`
- **Reviews**: approved / changes_requested / none

**PR ↔ Issue linking**:
- Scan each PR `body` for `fixes #N`, `closes #N`, `resolves #N` (case-insensitive)
- If found, display in the table: `Fixes #42` in the Action/Status column

**Categorization**:

_Internal PRs_: author in collaborators list

_External — Ready_: additions ≤ 1000 AND files ≤ 10 AND `mergeable` ≠ `CONFLICTING` AND CI clean/unstable

_External — Problematic_: any of:
- additions > 1000 OR files > 10
- OR `mergeable` == `CONFLICTING` (merge conflict)
- OR CI dirty (statusCheckRollup contains failures)
- OR overlap with another open PR (>50% shared files)

### Output — Triage Table

```
## Open PRs ({count})

### Internal PRs
| PR | Title | Size | CI | Status |
| -- | ----- | ---- | -- | ------ |

### External — Ready for Review
| PR | Author | Title | Size | CI | Reviews | Action |
| -- | ------ | ----- | ---- | -- | ------- | ------ |

### External — Problematic
| PR | Author | Title | Size | Problem | Recommended Action |
| -- | ------ | ----- | ---- | ------- | ------------------ |

### Summary
- Quick wins: {XS/S PRs ready to merge}
- Risks: {overlaps, XL sizes, CI dirty}
- Clusters: {authors with 3+ PRs}
- Stale: {PRs with no activity >14d}
- Overlaps: {PRs touching the same files}
```

0 PRs → display `No open PRs.` and stop.

### Automatic Copy

After displaying the triage table, copy to clipboard using platform-appropriate command:

```bash
# Detect platform and copy
UNAME=$(uname -s)
if [ "$UNAME" = "Darwin" ]; then
  pbcopy <<'EOF'
{full triage table}
EOF
elif command -v xclip &>/dev/null; then
  echo "{full triage table}" | xclip -selection clipboard
elif command -v wl-copy &>/dev/null; then
  echo "{full triage table}" | wl-copy
elif command -v clip.exe &>/dev/null; then
  echo "{full triage table}" | clip.exe
fi
```

Confirm: `Triage table copied to clipboard.` (EN) / `Tableau copié dans le presse-papier.` (FR)

---

## Phase 2 — Deep Review (opt-in)

### PR Selection

**If argument passed**:
- `"all"` → all external PRs
- Numbers (`"42 57"`) → only those PRs
- No argument → propose via `AskUserQuestion`

**If no argument**, display:

```
question: "Which PRs do you want to review in depth?"
header: "Deep Review"
multiSelect: true
options:
  - label: "All external"
    description: "Review {N} external PRs with parallel code-reviewer agents"
  - label: "Problematic only"
    description: "Focus on {M} risky PRs (CI dirty, too large, overlaps)"
  - label: "Ready only"
    description: "Review {K} PRs ready to merge"
  - label: "Skip"
    description: "Stop here — audit only"
```

**Draft PR behavior**:
- Draft PRs are EXCLUDED from "All external" and "Ready only"
- Draft PRs are INCLUDED in "Problematic only" (they need attention)
- To review a draft: type its number explicitly (e.g. `42`)

If "Skip" → end workflow.

### Executing Reviews

For each selected PR, launch a `code-reviewer` agent via **Task tool in parallel**:

```
subagent_type: code-reviewer
model: sonnet
prompt: |
  Review PR #{num}: "{title}" by @{author}

  **Metadata**: +{additions}/-{deletions}, {changedFiles} files ({size_label})
  **CI**: {ci_status} | **Reviews**: {existing_reviews} | **Draft**: {isDraft}

  **PR Body**:
  {body}

  **Diff**:
  {gh pr diff {num} output}

  Apply your security and architecture expertise. Use the project-specific checklist
  from the SKILL.md Configuration section if available.

  Return structured review:
  ### Critical Issues
  ### Important Issues
  ### Suggestions
  ### What's Good

  Be specific: quote file:line, explain the issue, suggest the fix.
```

**Fallback if parallel agents unavailable**: run reviews sequentially, one PR at a time. Notify user: `Running sequential review (parallel agents not available).`

Fetch diff via:
```bash
gh pr diff {num}
gh pr view {num} --json body,title,author -q '{body: .body, title: .title, author: .author.login}'
```

Aggregate all reports. Display a summary after all reviews complete.

---

## Phase 3 — Comments (mandatory validation)

### Draft Generation

For each reviewed PR, generate a GitHub comment using the template `templates/review-comment.md`.

**Rules**:
- Language: **English** (international audience)
- Tone: professional, constructive, factual
- Always include at least 1 positive point
- Quote code lines when relevant (format `file:42`)

### Display and Validation

**Display ALL drafted comments** in format:

```
---
### Draft — PR #{num}: {title}

{full comment}

---
```

Then request validation via `AskUserQuestion`:

```
question: "These comments are ready. Which ones do you want to post?"
header: "Post Comments"
multiSelect: true
options:
  - label: "All ({N} comments)"
    description: "Post on all reviewed PRs"
  - label: "PR #{x} — {title_truncated}"
    description: "Post only on this PR"
  - label: "None"
    description: "Cancel — post nothing"
```

(Generate one option per PR + "All" + "None")

### Posting

For each validated comment:

```bash
gh pr comment {num} --body-file - <<'REVIEW_EOF'
{comment}
REVIEW_EOF
```

Confirm each post: `Comment posted on PR #{num}: {title}`

If "None" → `No comments posted. Workflow complete.`

---

## Project-Specific Checklist

Add your stack's checklist to the agent prompt in Phase 2. Examples by stack:

**Node.js / TypeScript**:
- No `any` type without explicit justification
- `async/await` error handling (try/catch or `.catch()`)
- No unhandled promise rejections
- Input validation at API boundaries

**Python**:
- Type hints on all public functions
- Exception specificity (no bare `except:`)
- Resource cleanup (`with` statements, context managers)
- No mutable default arguments

**Rust**:
- `Result<T, E>` with `.context()` for error chain (no `.unwrap()` in production code)
- No `clone()` on hot paths without justification
- `lazy_static!` or `once_cell` for static regex
- Lifetime annotations where ownership is non-obvious

**Go**:
- Explicit error handling (no `_` discard without comment)
- `defer` for resource cleanup
- Context propagation in concurrent code
- No goroutine leaks

**Generic** (stack-agnostic):
- No secrets or hardcoded credentials
- New public functions have tests
- Breaking changes documented in PR body
- Dependencies added have clear justification

---

## Edge Cases

| Situation | Behavior |
|-----------|----------|
| 0 open PRs | Display `No open PRs.` + stop |
| Draft PR | Show in table, skip for review unless explicitly selected |
| Unknown CI | Display `?` in CI column |
| Review agent timeout | Show partial error, continue with others |
| `gh pr diff` empty | Skip this PR, notify user |
| Very large PR (>5000 additions) | Warn: "Partial review, diff truncated" |
| Collaborators API 403/404 | Fallback to last 10 merged PR authors |
| Parallel agents unavailable | Run sequential reviews, notify user |

---

## Notes

- Always derive owner/repo via `gh repo view`, never hardcode
- Use `gh` CLI (not `curl` GitHub API) except for collaborators list
- `statusCheckRollup` can be null → treat as `?`
- `mergeable` can be `MERGEABLE`, `CONFLICTING`, or `UNKNOWN` → treat `UNKNOWN` as `?`
- Never post without explicit user validation in chat
- Drafted comments must be visible BEFORE any `gh pr comment`

---

## Related: /review-pr

| | `/pr-triage` | `/review-pr` |
|--|-------------|--------------|
| **Scope** | Full PR backlog | Single PR |
| **Use when** | Catching up after accumulation, periodic triage | Reviewing a specific incoming PR |
| **Phases** | 3 (audit + deep review + comments) | 1 (review only) |
| **Agents** | Parallel sub-agents per PR | Single session |
| **Output** | Triage table + review reports + GitHub comments | Inline review |
| **Validation** | AskUserQuestion before posting | Manual decision |

**Decision rule**: use `/pr-triage` for backlog triage (5+ PRs), `/review-pr` for focused review of a single PR.
