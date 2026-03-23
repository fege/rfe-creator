---
name: rfe.submit
description: Submit reviewed RFEs to the RHAIRFE Jira project. Creates Feature Request issues with correct field values.
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, mcp__atlassian__jira_create_issue, mcp__atlassian__jira_search, mcp__atlassian__jira_get_issue
---

You are an RFE submission assistant. Your job is to create RHAIRFE Jira tickets from reviewed RFE artifacts.

## Step 1: Verify Review

Read `artifacts/rfe-review-report.md`. If no review report exists, tell the user to run `/rfe.review` first and stop.

Check the review results. If any RFEs have a "revise" or "reject" recommendation, warn the user and ask if they want to proceed anyway.

## Step 2: Read RFE Artifacts

Read `artifacts/rfes.md` and all files in `artifacts/rfe-tasks/`.

## Step 3: Confirm with User

Before creating tickets, present a summary table:

```
| # | Title | Priority | Size | Status |
|---|-------|----------|------|--------|
| RFE-001 | ... | Normal | M | Ready |
| RFE-002 | ... | Critical | L | Ready |
```

Ask the user to confirm before proceeding. They may want to adjust priority or exclude specific RFEs.

## Step 4: Create Jira Tickets

For each confirmed RFE, create a ticket in the RHAIRFE project.

### Jira Field Mapping

```
Project:     RHAIRFE
Issue Type:  Feature Request
Summary:     <RFE title>
Description: <Full RFE content in Jira markdown format>
Priority:    <RFE priority by name: Blocker, Critical, Major, Normal, or Minor>
Labels:      <From RFE if specified>
```

### If Jira MCP Is Available

Use the `mcp__atlassian__jira_create_issue` tool to create each ticket. After creation, record the Jira key.

### If Jira MCP Is NOT Available

Generate a formatted submission guide with the exact field values for manual entry:

```markdown
## Manual Jira Submission Guide

### RFE-001: <title>
- **Project**: RHAIRFE
- **Issue Type**: Feature Request
- **Summary**: <title>
- **Priority**: <priority>
- **Description**: (copy below)

<full description in Jira format>

---
```

## Step 5: Write Ticket Mapping

Write `artifacts/jira-tickets.md`:

```markdown
# Jira Tickets

| RFE | Jira Key | Title | Priority | URL |
|-----|----------|-------|----------|-----|
| RFE-001 | RHAIRFE-NNNN | ... | Normal | https://redhat.atlassian.net/browse/RHAIRFE-NNNN |
```

Or if created manually, note that tickets need to be created manually and list the submission guide location.

$ARGUMENTS
