---
name: rfe.speedrun
description: Run the full RFE pipeline (create, review, submit) with minimal interaction. Makes reasonable defaults, auto-revises if review fails.
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, Skill, mcp__atlassian__jira_create_issue, mcp__atlassian__jira_search, mcp__atlassian__jira_get_issue
---

You are running the full RFE pipeline in speedrun mode. Your goal is to go from a problem statement to submitted Jira tickets with minimal user interaction.

## Defaults

When the user doesn't specify, use these defaults:
- **Priority**: Normal
- **Size**: S or M (unless the input clearly describes a large initiative)
- **RFE count**: Single RFE, unless the input describes multiple distinct business needs
- **Labels**: None unless the user specifies

## Pipeline

### Phase 1: Create

Run `/rfe.create` with the user's input. Ask only the questions you cannot reasonably infer:
- **Always ask**: Who are the affected customers? What is the business justification?
- **Ask if unclear**: Is this one need or multiple? What does success look like?
- **Never ask**: Implementation details, architecture, technology choices

Limit to 2-5 questions total across the entire run.

### Phase 2: Review

Run `/rfe.review` on the produced artifacts.

**If all RFEs pass** (rubric >= 7/10 with no zeros, feasibility is feasible/conditional): proceed to Phase 3.

**If any RFE fails**: Auto-revise the failing RFEs using the review feedback. Read the review report, identify the specific issues, edit the RFE artifact files to address them, then re-run `/rfe.review`.

**Revision limits**:
- Maximum 2 revision cycles
- If RFEs still fail after 2 cycles, stop and present the review report to the user
- Tell them: "These RFEs need manual attention. Run `/rfe.review` after editing to continue."

### Phase 3: Submit

Run `/rfe.submit`. Present the confirmation table to the user before creating tickets — this is the one mandatory interaction point.

## Output

At the end, summarize what was created:

```
## Speedrun Complete

Created N RFEs:
- RHAIRFE-NNNN: <title> (Priority: Normal)
- RHAIRFE-NNNN: <title> (Priority: Critical)

Review cycles: N
Artifacts: artifacts/rfes.md, artifacts/rfe-tasks/, artifacts/rfe-review-report.md, artifacts/jira-tickets.md
```

$ARGUMENTS
