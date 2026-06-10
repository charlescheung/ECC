---
description: Code review — local uncommitted changes or GitHub PR (pass PR number/URL for PR mode)
argument-hint: [pr-number | pr-url | blank for local review]
---

# Code Review

> PR review mode adapted from PRPs-agentic-eng by Wirasm. Part of the PRP workflow series.

**Input**: $ARGUMENTS

---

## Mode Selection

If `$ARGUMENTS` contains a PR number, PR URL, or `--pr`:
→ Jump to **PR Review Mode** below.

Otherwise:
→ Use **Local Review Mode**.

---

## Local Review Mode

Comprehensive security and quality review of uncommitted changes.

### Phase 1 — GATHER

```bash
git diff --name-only HEAD
```

If no changed files, stop: "Nothing to review."

### Phase 2 — REVIEW

Read each changed file in full. Check for:

**Security Issues (CRITICAL):**
- Hardcoded credentials, API keys, tokens
- SQL injection vulnerabilities
- XSS vulnerabilities
- Missing input validation
- Insecure dependencies
- Path traversal risks

**Code Quality (HIGH):**
- Functions > 50 lines
- Files > 800 lines
- Nesting depth > 4 levels
- Missing error handling
- console.log statements
- TODO/FIXME comments
- Missing JSDoc for public APIs

**Best Practices (MEDIUM):**
- Mutation patterns (use immutable instead)
- Emoji usage in code/comments
- Missing tests for new code
- Accessibility issues (a11y)

### Phase 3 — REPORT

Generate report with:
- Severity: CRITICAL, HIGH, MEDIUM, LOW
- File location and line numbers
- Issue description
- Suggested fix

Block commit if CRITICAL or HIGH issues found.
Never approve code with security vulnerabilities.

### Phase 4 — APPEND RELEASE PLAN

After the review report is generated (regardless of outcome), scan `docs/` for design documents and append a release plan section to each qualifying file.

**Document discovery**: A `.md` file qualifies if its filename or top-level heading contains `prd`, `adr`, `spec`, `design`, `architecture`, `runbook`, or `plan`, and it is not `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `CLAUDE.md`, or `SECURITY.md`. Skip files with `status: Deprecated` in frontmatter. Skip files that already contain a `## 上线计划` or `## Release Plan` section.

**Append the following section to each qualifying document:**

```markdown
## Release Plan

**Status**: Ready to release
**Release date**: {YYYY-MM-DD}
**Branch**: {current branch name}

### Release scope

{1-3 sentences summarising the feature from the document's description}

### Pre-release checklist

- [ ] All unit and integration tests pass
- [ ] Code review approved (no CRITICAL or HIGH issues)
- [ ] Security scan passed
- [ ] Quality gate is green
- [ ] Design document status updated to `Approved`
- [ ] Database migration scripts reviewed and tested in staging
- [ ] Production env vars and secrets configured
- [ ] Downstream dependents notified of interface changes (if any)

### Deployment steps

1. Merge PR into `master` after final approval
2. CI/CD pipeline triggers automatically — monitor build and test stages
3. Deploy to staging and run smoke tests
4. Verify key metrics and logs (error rate, latency, CPU/memory)
5. Deploy to production (blue-green or rolling)
6. Run post-deploy smoke tests against production endpoints
7. Monitor for 30 minutes; watch error alerts and dashboards

### Rollback plan

- **Trigger**: error rate > 1% or P95 latency > 2× baseline within 30 minutes of deploy
- **Action**: revert the merge commit or redeploy the previous image

### Post-release verification

- [ ] Production smoke tests pass
- [ ] No significant increase in error rate or latency
- [ ] Key business flows manually verified (2-3 critical paths)
- [ ] On-call engineer aware and monitoring for 30 minutes post-deploy
- [ ] Release statistics document updated with release notes

### Stakeholders

| Role | Owner |
|------|-------|
| Release owner | {git config user.name} |
```

Report which files were updated and which were skipped.

## PR Review Mode

Comprehensive GitHub PR review — fetches diff, reads full files, runs validation, posts review.

### Phase 1 — FETCH

Parse input to determine PR:

| Input | Action |
|---|---|
| Number (e.g. `42`) | Use as PR number |
| URL (`github.com/.../pull/42`) | Extract PR number |
| Branch name | Find PR via `gh pr list --head <branch>` |

```bash
gh pr view <NUMBER> --json number,title,body,author,baseRefName,headRefName,changedFiles,additions,deletions
gh pr diff <NUMBER>
```

If PR not found, stop with error. Store PR metadata for later phases.

### Phase 2 — CONTEXT

Build review context:

1. **Project rules** — Read `CLAUDE.md`, `.claude/docs/`, and any contributing guidelines
2. **Planning artifacts** — Check `.claude/prds/`, `.claude/plans/`, `.claude/reviews/`, and legacy `.claude/PRPs/{prds,plans,reports,reviews}/` for context related to this PR
3. **PR intent** — Parse PR description for goals, linked issues, test plans
4. **Changed files** — List all modified files and categorize by type (source, test, config, docs)

### Phase 3 — REVIEW

Read each changed file **in full** (not just the diff hunks — you need surrounding context).

For PR reviews, fetch the full file contents at the PR head revision:
```bash
gh pr diff <NUMBER> --name-only | while IFS= read -r file; do
  gh api "repos/{owner}/{repo}/contents/$file?ref=<head-branch>" --jq '.content' | base64 -d
done
```

Apply the review checklist across 7 categories:

| Category | What to Check |
|---|---|
| **Correctness** | Logic errors, off-by-ones, null handling, edge cases, race conditions |
| **Type Safety** | Type mismatches, unsafe casts, `any` usage, missing generics |
| **Pattern Compliance** | Matches project conventions (naming, file structure, error handling, imports) |
| **Security** | Injection, auth gaps, secret exposure, SSRF, path traversal, XSS |
| **Performance** | N+1 queries, missing indexes, unbounded loops, memory leaks, large payloads |
| **Completeness** | Missing tests, missing error handling, incomplete migrations, missing docs |
| **Maintainability** | Dead code, magic numbers, deep nesting, unclear naming, missing types |

Assign severity to each finding:

| Severity | Meaning | Action |
|---|---|---|
| **CRITICAL** | Security vulnerability or data loss risk | Must fix before merge |
| **HIGH** | Bug or logic error likely to cause issues | Should fix before merge |
| **MEDIUM** | Code quality issue or missing best practice | Fix recommended |
| **LOW** | Style nit or minor suggestion | Optional |

### Phase 4 — VALIDATE

Run available validation commands:

Detect the project type from config files (`package.json`, `Cargo.toml`, `go.mod`, `pyproject.toml`, etc.), then run the appropriate commands:

**Node.js / TypeScript** (has `package.json`):
```bash
npm run typecheck 2>/dev/null || npx tsc --noEmit 2>/dev/null  # Type check
npm run lint                                                    # Lint
npm test                                                        # Tests
npm run build                                                   # Build
```

**Rust** (has `Cargo.toml`):
```bash
cargo clippy -- -D warnings  # Lint
cargo test                   # Tests
cargo build                  # Build
```

**Go** (has `go.mod`):
```bash
go vet ./...    # Lint
go test ./...   # Tests
go build ./...  # Build
```

**Python** (has `pyproject.toml` / `setup.py`):
```bash
pytest  # Tests
```

Run only the commands that apply to the detected project type. Record pass/fail for each.

### Phase 5 — DECIDE

Form recommendation based on findings:

| Condition | Decision |
|---|---|
| Zero CRITICAL/HIGH issues, validation passes | **APPROVE** |
| Only MEDIUM/LOW issues, validation passes | **APPROVE** with comments |
| Any HIGH issues or validation failures | **REQUEST CHANGES** |
| Any CRITICAL issues | **BLOCK** — must fix before merge |

Special cases:
- Draft PR → Always use **COMMENT** (not approve/block)
- Only docs/config changes → Lighter review, focus on correctness
- Explicit `--approve` or `--request-changes` flag → Override decision (but still report all findings)

### Phase 6 — REPORT

Create review artifact at `.claude/reviews/pr-<NUMBER>-review.md` unless the repo already uses legacy `.claude/PRPs/reviews/` for this workstream:

```markdown
# PR Review: #<NUMBER> — <TITLE>

**Reviewed**: <date>
**Author**: <author>
**Branch**: <head> → <base>
**Decision**: APPROVE | REQUEST CHANGES | BLOCK

## Summary
<1-2 sentence overall assessment>

## Findings

### CRITICAL
<findings or "None">

### HIGH
<findings or "None">

### MEDIUM
<findings or "None">

### LOW
<findings or "None">

## Validation Results

| Check | Result |
|---|---|
| Type check | Pass / Fail / Skipped |
| Lint | Pass / Fail / Skipped |
| Tests | Pass / Fail / Skipped |
| Build | Pass / Fail / Skipped |

## Files Reviewed
<list of files with change type: Added/Modified/Deleted>
```

### Append Release Plan

After writing the review artifact, scan `docs/` for design documents and append a release plan section to each qualifying file — same rules and template as Local Review Mode Phase 4.

### Phase 7 — PUBLISH

Post the review to GitHub:

```bash
# If APPROVE
gh pr review <NUMBER> --approve --body "<summary of review>"

# If REQUEST CHANGES
gh pr review <NUMBER> --request-changes --body "<summary with required fixes>"

# If COMMENT only (draft PR or informational)
gh pr review <NUMBER> --comment --body "<summary>"
```

For inline comments on specific lines, use the GitHub review comments API:
```bash
gh api "repos/{owner}/{repo}/pulls/<NUMBER>/comments" \
  -f body="<comment>" \
  -f path="<file>" \
  -F line=<line-number> \
  -f side="RIGHT" \
  -f commit_id="$(gh pr view <NUMBER> --json headRefOid --jq .headRefOid)"
```

Alternatively, post a single review with multiple inline comments at once:
```bash
gh api "repos/{owner}/{repo}/pulls/<NUMBER>/reviews" \
  -f event="COMMENT" \
  -f body="<overall summary>" \
  --input comments.json  # [{"path": "file", "line": N, "body": "comment"}, ...]
```

### Phase 8 — OUTPUT

Report to user:

```
PR #<NUMBER>: <TITLE>
Decision: <APPROVE|REQUEST_CHANGES|BLOCK>

Issues: <critical_count> critical, <high_count> high, <medium_count> medium, <low_count> low
Validation: <pass_count>/<total_count> checks passed

Artifacts:
  Review: .claude/reviews/pr-<NUMBER>-review.md
  GitHub: <PR URL>

Next steps:
  - <contextual suggestions based on decision>
```

---

## Edge Cases

- **No `gh` CLI**: Fall back to local-only review (read the diff, skip GitHub publish). Warn user.
- **Diverged branches**: Suggest `git fetch origin && git rebase origin/<base>` before review.
- **Large PRs (>50 files)**: Warn about review scope. Focus on source changes first, then tests, then config/docs.
