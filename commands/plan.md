---
description: Restate requirements, assess risks, and create step-by-step implementation plan. WAIT for user CONFIRM before touching any code.
argument-hint: "[feature description | path/to/*.prd.md]"
---

# Plan Command

This command creates a comprehensive implementation plan before writing any code. It accepts either free-form requirements or a PRD markdown file.

Run inline by default. Do not call the Task tool or any subagent by default. This keeps `/plan` usable from plugin installs that ship commands without agent files.

## What This Command Does

1. **Restate Requirements** - Clarify what needs to be built
2. **Identify Risks** - Surface potential issues and blockers
3. **Create Step Plan** - Break down implementation into phases
4. **Generate Design Doc** - Generate a design doc under the project `docs/` directory
5. **Generate Release Plan** - Generate a qualifying release plan section
6. **Wait for Confirmation** - MUST receive user approval before proceeding

## When to Use

Use `/plan` when:
- Starting a new feature
- Making significant architectural changes
- Working on complex refactoring
- Multiple files/components will be affected
- Requirements are unclear or ambiguous

## How It Works

The assistant will:

1. **Analyze the request** and restate requirements in clear terms
2. **Ground the plan** in relevant codebase patterns when the repo is available
3. **Break down into phases** with specific, actionable steps
4. **Identify dependencies** between components
5. **Assess risks** and potential blockers
6. **Estimate complexity** (High/Medium/Low)
7. **Generate release plan** — generate a qualifying release plan section (see template below)
8. **Present the plan** and WAIT for your explicit confirmation

## Input Modes

| Input | Mode | Behavior |
|---|---|---|
| `path/to/name.prd.md` | PRD artifact mode | Read the PRD, pick the next pending delivery milestone or implementation phase, and write `.claude/plans/{name}.plan.md` |
| Any other markdown path | Reference mode | Read the file as context and produce an inline plan |
| Free-form text | Conversational mode | Produce an inline plan |
| Empty input | Clarification mode | Ask what should be planned |

In PRD artifact mode, create `.claude/plans/` if needed. If the PRD contains a `Delivery Milestones` table, update only the selected row from `pending` to `in-progress` and set its `Plan` cell to the generated plan path. If the PRD uses the legacy `.claude/PRPs/prds/` format with `Implementation Phases`, read it without migrating paths.

## Pattern Grounding

Before writing the plan, search the codebase for conventions the implementation should mirror. Capture the top example for each relevant category with file references:

| Category | What to capture |
|---|---|
| Naming | File, function, type, command, or script naming in the affected area |
| Error handling | How failures are raised, returned, logged, or handled gracefully |
| Logging | Levels, format, and what gets logged |
| Data access | Repository, service, query, or filesystem patterns |
| Tests | Test file location, framework, fixtures, and assertion style |

If no similar code exists, state that explicitly. Do not invent a pattern.

## PRD Artifact Output

When called with a `.prd.md` file, write the plan to `.claude/plans/{kebab-case-name}.plan.md` using this structure:

````markdown
# Plan: {Feature Name}

**Source PRD**: {path}
**Selected Milestone**: {milestone or phase name}
**Complexity**: {Small | Medium | Large}

## Summary
{2-3 sentences}

## Patterns to Mirror
| Category | Source | Pattern |
|---|---|---|
| Naming | `path:line` | {short description} |
| Errors | `path:line` | {short description} |
| Tests | `path:line` | {short description} |

## Files to Change
| File | Action | Why |
|---|---|---|
| `path` | CREATE / UPDATE / DELETE | {reason} |

## Tasks
### Task 1: {name}
- **Action**: {what to do}
- **Mirror**: {pattern to follow}
- **Validate**: {command that proves correctness}

## Validation
```bash
{project-specific validation commands}
```

## Risks
| Risk | Likelihood | Mitigation |
|---|---|---|

## Acceptance
- [ ] All tasks complete
- [ ] Validation passes
- [ ] Patterns mirrored, not reinvented
````

After writing the artifact, report its path and WAIT for confirmation before writing code.

## Example Usage

```
User: /plan I need to add real-time notifications when markets resolve

Assistant:
# Implementation Plan: Real-Time Market Resolution Notifications

## Requirements Restatement
- Send notifications to users when markets they're watching resolve
- Support multiple notification channels (in-app, email, webhook)
- Ensure notifications are delivered reliably
- Include market outcome and user's position result

## Implementation Phases

### Phase 1: Database Schema
- Add notifications table with columns: id, user_id, market_id, type, status, created_at
- Add user_notification_preferences table for channel preferences
- Create indexes on user_id and market_id for performance

### Phase 2: Notification Service
- Create notification service in lib/notifications.ts
- Implement notification queue using BullMQ/Redis
- Add retry logic for failed deliveries
- Create notification templates

### Phase 3: Integration Points
- Hook into market resolution logic (when status changes to "resolved")
- Query all users with positions in market
- Enqueue notifications for each user

### Phase 4: Frontend Components
- Create NotificationBell component in header
- Add NotificationList modal
- Implement real-time updates via Supabase subscriptions
- Add notification preferences page

## Dependencies
- Redis (for queue)
- Email service (SendGrid/Resend)
- Supabase real-time subscriptions

## Risks
- HIGH: Email deliverability (SPF/DKIM required)
- MEDIUM: Performance with 1000+ users per market
- MEDIUM: Notification spam if markets resolve frequently
- LOW: Real-time subscription overhead

## Estimated Complexity: MEDIUM
- Backend: 4-6 hours
- Frontend: 3-4 hours
- Testing: 2-3 hours
- Total: 9-13 hours

**WAITING FOR CONFIRMATION**: Proceed with this plan? (yes/no/modify)
```

## Important Notes

**CRITICAL**: This command will **NOT** write any code until you explicitly confirm the plan with "yes" or "proceed" or similar affirmative response.

If you want changes, respond with:
- "modify: [your changes]"
- "different approach: [alternative]"
- "skip phase 2 and do phase 3 first"

## Release Plan Template

**Document discovery**: A `.md` file qualifies if its filename or top-level heading contains `prd`, `adr`, `spec`, `design`, `architecture`, `runbook`, or `plan`. Exclude `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `CLAUDE.md`, `SECURITY.md`, and files with `status: Deprecated` in frontmatter.

```markdown
**Status**: Ready to release
**Release date**: {YYYY-MM-DD}
**Branch**: {current branch name}

### 1. Release scope

{1-3 sentences summarising the feature from the document's description}

### 2. Pre-release checklist

- [ ] All unit and integration tests pass
- [ ] Code review approved (no CRITICAL or HIGH issues)
- [ ] Security scan passed
- [ ] Quality gate is green
- [ ] Design document status updated to `Approved`
- [ ] Database migration scripts reviewed and tested in staging
- [ ] Production env vars and secrets configured
- [ ] Downstream dependents notified of interface changes (if any)

### 3. Deployment steps

1. Merge PR into `master` after final approval
2. CI/CD pipeline triggers automatically — monitor build and test stages
3. Deploy to staging and run smoke tests
4. Verify key metrics and logs (error rate, latency, CPU/memory)
5. Deploy to production (blue-green or rolling)
6. Run post-deploy smoke tests against production endpoints
7. Monitor for 30 minutes; watch error alerts and dashboards

### 4. Rollback plan

- **Trigger**: error rate > 1% or P95 latency > 2× baseline within 30 minutes of deploy
- **Action**: revert the merge commit or redeploy the previous image

### 5. Post-release verification

- [ ] Production smoke tests pass
- [ ] No significant increase in error rate or latency
- [ ] Key business flows manually verified (2-3 critical paths)
- [ ] On-call engineer aware and monitoring for 30 minutes post-deploy
- [ ] Release statistics document updated with release notes

### 6. Stakeholders

| Role | Owner |
|------|-------|
| Release owner | {git config user.name} |
```

Report which files were updated and which were skipped.

## Integration with Other Commands

After planning:
- Use the `tdd-workflow` skill to implement with test-driven development
- Use `/build-fix` if build errors occur
- Use `/code-review` to review completed implementation (updates release plan status in design docs)
- Use `/pr` or `/prp-pr` to open a pull request

> **Need requirements first?** Use `/plan-prd` for a lean PRD at `.claude/prds/{name}.prd.md`.
>
> **Need the legacy PRP flow?** Use `/prp-plan` for deep PRP planning with `.claude/PRPs/` artifacts. Use `/prp-implement` to execute those plans with rigorous validation loops.

## Optional Planner Agent

ECC also provides a `planner` agent for manual installs that include agent files. Use it only when the local runtime already exposes that subagent and the user explicitly asks you to delegate planning.

If the `planner` subagent is unavailable, continue planning inline instead of surfacing an "Agent type 'planner' not found" error.

For manual installs, the source file lives at:
`agents/planner.md`
