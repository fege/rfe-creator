---
name: rfe.review
description: Review and improve RFEs. Accepts a Jira key (e.g., /rfe.review RHAIRFE-1234) to fetch and review an existing RFE, or reviews local artifacts from /rfe.create. Runs rubric scoring, technical feasibility checks, and auto-revises issues it finds.
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, AskUserQuestion, mcp__atlassian__jira_get_issue
---

You are an RFE review orchestrator. Your job is to review RFEs for quality and technical feasibility, and auto-revise issues when possible.

## Step 0: Resolve Input

Check if `$ARGUMENTS` contains a Jira key (e.g., `RHAIRFE-1234`).

**If a Jira key is provided**: Fetch the RFE from Jira using `mcp__atlassian__getJiraIssue` with `fields: ["comment"]` to get both the description and comment history. Write the RFE to `artifacts/rfe-tasks/` as a local artifact using the RFE template format (read `${CLAUDE_SKILL_DIR}/../rfe.create/rfe-template.md` for the format). Update `artifacts/rfes.md` with the RFE summary. Record the Jira key in the artifact metadata so `/rfe.submit` knows to update rather than create.

**Also write a separate comments file** to `artifacts/rfe-tasks/RFE-NNN-comments.md` with the Jira comment history. Format each comment as:

```markdown
# Comments: RHAIRFE-NNNN

## <Author Name> — <date>
<comment body>

## <Author Name> — <date>
<comment body>
```

This file provides stakeholder context to the review forks. It is NOT part of the RFE content and must NOT be pushed back to Jira during submission.

**If no Jira key**: Proceed with existing local artifacts.

## Step 1: Verify Artifacts Exist

Read `artifacts/rfes.md` and list files in `artifacts/rfe-tasks/`. If no RFE artifacts exist and no Jira key was provided, tell the user to run `/rfe.create` first or provide a Jira key (e.g., `/rfe.review RHAIRFE-1234`) and stop.

Check if a prior review report exists at `artifacts/rfe-review-report.md`. If it does, read it — this is a re-review after revisions.

## Step 1.5: Fetch Architecture Context

```bash
bash scripts/fetch-architecture-context.sh
```

The architecture context path for the feasibility fork is `.context/architecture-context/architecture/$LATEST`.

If the fetch fails (network issue, repo unavailable, API rate limit), proceed without architecture context. Note it in the review report.

## Step 2: Run Reviews

Run two independent reviews. These assessments must remain separate — "this RFE is poorly written" is a different concern from "this RFE is technically infeasible."

### Review 1: Rubric Validation

<!-- TEMPORARY: This bootstrap approach clones assess-rfe from GitHub and copies
     the skill locally because the Claude Agent SDK doesn't yet support marketplace
     plugin resolution. Once the SDK or ambient runner adds plugin support, this
     can be replaced with a direct /assess-rfe:assess-rfe plugin invocation. -->

Bootstrap the assess-rfe skill by running:

```bash
bash scripts/bootstrap-assess-rfe.sh
```

This clones the assess-rfe repo into `.context/assess-rfe/` and copies the skills into `.claude/skills/`. If the clone already exists, it reuses it.

When any assess-rfe skill resolves its `{PLUGIN_ROOT}`, it should use the absolute path of `.context/assess-rfe/` in the project working directory.

**If the bootstrap succeeded**: Invoke `/assess-rfe` to score each RFE against the rubric. The plugin owns the scoring logic, criteria, and calibration. Do not reimplement or second-guess its scores.

**If the bootstrap failed** (network issue, git unavailable): Skip rubric validation. Note in the review report that rubric validation was skipped because assess-rfe could not be fetched. Perform a basic quality check instead:
- Does each RFE describe a business need (WHAT/WHY), not a task or technical activity?
- Does each RFE avoid prescribing architecture, technology, or implementation?
- Does each RFE name specific affected customers?
- Does each RFE include evidence-based business justification?
- Is each RFE right-sized for a single strategy feature?

### Stakeholder Context

Both review forks should read any `artifacts/rfe-tasks/RFE-NNN-comments.md` files that exist. Comments from stakeholders provide context about what is intentional in the RFE, what has already been discussed, and what related work exists. This context should inform the review — e.g., if a stakeholder has already confirmed a technology choice is deliberate, the rubric should not penalize it.

### Review 2: Technical Feasibility (Forked)

Invoke the `rfe-feasibility-review` skill on the RFE artifacts. This runs in a forked context with architecture context (if available) to assess whether each RFE is technically feasible without the business context influencing the assessment. If comments files exist in `artifacts/rfe-tasks/`, include them in the feasibility reviewer's context.

## Step 3: Combine Results

Write `artifacts/rfe-review-report.md` with the following structure:

```markdown
# RFE Review Report

**Date**: <date>
**RFEs reviewed**: <count>
**Rubric validation**: <pass/fail/skipped>
**Technical feasibility**: <pass/conditional/fail>

## Summary
<Overall assessment: are these RFEs ready for submission?>

## Per-RFE Results

### RFE-001: <title>

**Rubric score**: <score>/10 <PASS/FAIL> (or "skipped — plugin not installed")
<Rubric feedback details if available>

**Technical feasibility**: <feasible / infeasible / needs RFE revision>
**Strategy considerations**: <none / list of items flagged for /strat.refine>

**Recommendation**: <submit / revise / split / reject>
<Specific actionable suggestions if revision needed>

### RFE-002: <title>
...

## Changes vs. Original
<For Jira-sourced RFEs only: summarize what was modified compared to the original Jira description. List sections added, removed, or edited so the user can see what will change if they submit.>

## Revision History
<If this is a re-review, note what changed since the prior review:>
- What concerns from the prior review were addressed
- What concerns remain
- What new issues the revisions introduced
```

## Step 4: Auto-Revise

Always attempt at least one auto-revision cycle when any criterion scores below full marks. Improve what you can with available information. If a revision requires information you don't have (e.g., named customer accounts), make the best improvement possible and note the gap in Revision Notes for the user. Only skip auto-revision entirely if the RFE is technically infeasible or the problem statement needs to be rethought from scratch.

### Revision Principles

**Only edit sections that directly caused a rubric failure.** If the rubric didn't flag a section, don't touch it. If you're unsure whether a section contributed to a score, leave it alone. Never rewrite the entire artifact from scratch — this destroys author context that wasn't scored.

**When a section mixes WHAT and HOW, annotate — don't delete.** If a section contains both business need and implementation detail, annotate the HOW portions inline with a review note rather than removing or rewriting the section:

```markdown
> *Review note: The implementation detail above may be more appropriate in the strategy phase (/strat.refine). Preserved for continuity.*
```

This flags the content for strategy refinement without destroying it. The HOW details are useful — they just belong in the STRAT, not the RFE. Labeling them lets the content carry forward naturally.

**Right-sizing is a recommendation, never auto-applied.** If the rubric scores Right-sized at 0 or 1, report the recommendation to split in the review report. Do NOT remove acceptance criteria, scope items, or capabilities from the artifact to force a different shape. Splitting an RFE is a structural decision that changes what the RFE *is* — only the author can make that call.

**Do not invent missing evidence.** If the rubric flags weak business justification due to missing named customers or revenue data, flag the gap in Revision Notes for the author to fill. Do not fabricate evidence.

### Revision Steps

1. Read the review feedback for each failing RFE
2. Read the comments file (`artifacts/rfe-tasks/RFE-NNN-comments.md`) if it exists — stakeholder comments may explain why certain content is intentional
3. Make targeted edits to the artifact files using the Edit tool to address specific rubric failures
4. Annotate (don't delete) any author context sections that lean into HOW
5. Add a `### Revision Notes` section at the end of each revised RFE documenting what changed and what gaps remain for the user to fill
6. Re-run the review (go back to Step 2) on the revised artifacts

**Revision limits**:
- Maximum 2 auto-revision cycles
- If RFEs still fail after 2 cycles, stop and present the review report to the user

## Step 5: Advise the User

Based on the results:
- **All pass**: Tell the user RFEs are ready for `/rfe.submit`.
- **Some need revision after auto-revise failed**: List the remaining issues. Tell the user to edit the artifact files and re-run `/rfe.review`.
- **Fundamental problems**: Recommend re-running `/rfe.create` if the RFEs need to be rethought entirely.

$ARGUMENTS
