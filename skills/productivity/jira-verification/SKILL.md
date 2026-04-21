---
name: jira-verification
description: >
  Generate verification steps for Jira issues (bugs, features, epics). Analyzes the Jira,
  its parent epic, linked issues, and sub-tasks. Identifies PRs and code changes across
  repos. Checks for documentation, molecule tests, and produces structured manual QA
  verification steps. Focused on OSPRH/EDPM but works for any Jira project.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [jira, verification, qa, testing, openstack, edpm]
    related_skills: [edpm-ansible, quarterly-review-generator, github-code-review]
trigger: >
  User asks to verify a Jira, create verification steps, generate test steps for
  an issue, review what a Jira implements, or anything like "verify OSPRH-XXXX",
  "create verification steps for XXXX", "how to test XXXX".
---

# Jira Verification Steps Generator

Generate structured, manual verification steps for Jira issues by analyzing the
issue, its epic, sub-tasks, linked PRs, code changes, documentation, and tests.

## Prerequisites

- Jira API access: `$JIRA_API_TOKEN` env var, user's Jira instance (redhat.atlassian.net)
- GitHub CLI (`gh`) authenticated — to inspect PRs and code changes
- Knowledge of the project structure (load relevant skills like `edpm-ansible`)

## When to Use

- User says "verify PROJ-XXXX", "create verification steps", "how do I test this Jira"
- User wants to understand what a Jira implements and how to validate it
- QA verification for bugs, features, or epics before release

## Step 1: Gather Jira Context

### 1a: Get the target issue details

Use the Jira MCP tools to get the full issue:

```
mcp_jira_jira_get_issue(issueIdOrKey="OSPRH-XXXX")
```

Extract: summary, description, acceptance criteria, issue type, status, fix versions,
components, labels.

### 1b: Get the parent epic (if any)

If the issue has a parent/epic link, fetch the epic to understand the broader scope:

```
mcp_jira_jira_get_issue(issueIdOrKey="<epic-key>")
```

### 1c: Get sub-tasks and linked issues

```
mcp_jira_jira_get_issue_links(issueIdOrKey="OSPRH-XXXX")
```

For epics, get all child issues:
```
mcp_jira_jira_get_epic_issues(epicIdOrKey="OSPRH-XXXX")
```

### 1d: Get comments for implementation context

```
mcp_jira_jira_get_issue_comments(issueIdOrKey="OSPRH-XXXX")
```

Comments often contain design decisions, review feedback, and testing notes.

## Step 2: Identify PRs and Code Changes

### 2a: Search for PRs referencing the Jira key

Search across the relevant repos for PRs mentioning the Jira key:

```bash
# Search GitHub for PRs mentioning the Jira key
gh search prs "OSPRH-XXXX" --json title,repository,url,state,number,mergedAt

# Also search in specific repos if the Jira mentions them
gh pr list --repo openstack-k8s-operators/<repo> --search "OSPRH-XXXX" \
  --json title,url,state,number,mergedAt
```

Common OSPRH repos to check:
- openstack-k8s-operators/edpm-ansible
- openstack-k8s-operators/openstack-operator
- openstack-k8s-operators/dataplane-operator
- openstack-k8s-operators/heat-operator
- openstack-k8s-operators/install_yamls
- openstack-k8s-operators/ci-framework

### 2b: Analyze the code changes

For each PR found, examine the diff to understand what changed:

```bash
# Get the diff for a PR
gh pr diff <pr-number> --repo <org>/<repo>

# Or view the files changed
gh pr view <pr-number> --repo <org>/<repo> --json files
```

Key things to identify:
- **New roles/playbooks** — new functionality added
- **Modified tasks** — changed behavior
- **New variables** — configurable features (check defaults/main.yml)
- **New modules/plugins** — custom logic
- **Config changes** — new config file templates, paths, formats

### 2c: Check for linked PRs in Jira description/comments

Parse the Jira description and comments for GitHub PR URLs like:
- `https://github.com/openstack-k8s-operators/*/pull/*`
- `https://github.com/*/commit/*`

Also look for Gerrit review links if the project uses OpenDev.

## Step 3: Check Documentation

### 3a: Look for doc changes in the PRs

```bash
# Check if any PRs touch documentation
gh pr diff <pr-number> --repo <org>/<repo> | grep -E '^\+\+\+ .*(docs|README|\.rst|\.md)'
```

### 3b: Check for new/updated docs in the repo

```bash
# For edpm-ansible, check docs/source/roles/
gh api repos/<org>/<repo>/contents/docs --jq '.[].name'
```

### 3c: Note if documentation is missing

If a feature adds new roles, variables, or behavior but has no documentation
updates, flag this in the verification steps as a gap.

## Step 4: Check for Tests

### 4a: Molecule tests (for Ansible roles)

```bash
# Check if PRs include molecule test changes
gh pr diff <pr-number> --repo <org>/<repo> | grep -E '^\+\+\+ .*molecule/'

# Check if the role has molecule tests
gh api repos/<org>/<repo>/contents/roles/<role>/molecule --jq '.[].name'
```

### 4b: Unit tests

```bash
# Check for pytest/unittest changes
gh pr diff <pr-number> --repo <org>/<repo> | grep -E '^\+\+\+ .*(tests/|test_)'
```

### 4c: Integration/CI tests

Check the Zuul or GitHub Actions CI definitions for new test jobs:
```bash
gh pr diff <pr-number> --repo <org>/<repo> | grep -E '^\+\+\+ .*(zuul\.d/|\.github/workflows/)'
```

### 4d: Note missing test coverage

If new functionality lacks tests, flag it. For EDPM roles, every role should
have molecule tests — this is mandatory per the project rules.

## Step 5: Synthesize Verification Steps

### Output Format

Generate a markdown document following this structure (see reference example):

```markdown
# Verification Steps for <JIRA-KEY>

**Epic:** [<EPIC-KEY>](<epic-url>) — <epic-summary>

## Background

<2-3 paragraph summary of what this Jira implements, derived from the epic
description, sub-tasks, and code changes. Explain the feature/fix in plain
language. List the main components of the implementation.>

## Pre-requisites

<What environment/setup is needed to test. E.g., deployed RHOSO, specific
services running, specific configuration, etc.>

---

## 1. Verify <First Testable Aspect>

**Objective:** <What you're confirming works>

**Steps:**

1. <Step with specific action>
2. <Step with specific command>
   ```bash
   <actual command to run>
   ```
3. **Expected:** <What the result should be, with specific values/formats>

---

## 2. Verify <Second Testable Aspect>
...

## N. Negative Test — <Edge Case>

**Objective:** <What should fail gracefully>

**Steps:**
...
```

### Guidelines for Writing Steps

1. **Be specific** — Include exact file paths, command names, config values
2. **Include expected output** — Show what correct results look like (YAML
   snippets, file listings, command output)
3. **Test the happy path first** — Verify basic functionality works before
   edge cases
4. **Test cleanup/removal** — If the feature adds something, verify removing it
   also works correctly
5. **Test idempotency** — Running the same operation twice should produce the
   same result (especially for Ansible roles)
6. **Test upgrade path** — If this changes existing behavior, verify migration
   from the old state works
7. **Include negative tests** — What happens with bad input, missing files,
   wrong configuration?
8. **Derive steps from code** — Read the actual task names, variable names,
   and file paths from the code changes. Don't guess.
9. **Note version caveats** — If a feature may not be in all versions, note
   it (e.g., "This may not be in the version you're using")
10. **Cross-reference sub-tasks** — Each sub-task in an epic often maps to
    one or more verification sections

### Section Mapping

Map code changes to verification sections:
- New Ansible role → verify the role's primary function works
- New variable in defaults/main.yml → verify the variable controls behavior
- New container management → verify containers are created/removed
- Config file changes → verify config files exist at expected paths
- State tracking → verify state files are created and updated
- Cleanup logic → verify cleanup removes the right things
- Error handling → negative test for graceful failure

## Step 6: Save the Output

Save the verification steps as `<JIRA-KEY>-verification-steps.md` in the
current directory or a location the user specifies.

## Pitfalls

1. **Jira API**: Use `/rest/api/3/search/jql` GET endpoint (old `/search` POST
   is deprecated). Use MCP tools when available.
2. **Multiple repos**: Features often span multiple repos (e.g., ansible role +
   operator changes + CI updates). Check all relevant repos.
3. **PR search limitations**: `gh search prs` may not find all PRs if the Jira
   key isn't in the title. Also check the Jira description/comments for PR links.
4. **Acceptance criteria**: Some Jiras have explicit acceptance criteria in the
   description — incorporate these directly into verification steps.
5. **Don't fabricate steps**: Every verification step must trace back to actual
   code changes or documented requirements. If you can't find what a feature
   does, say so rather than guessing.
6. **EDPM-specific paths**: Common EDPM paths to verify:
   - `/var/lib/edpm-config/` — EDPM configuration
   - `/var/lib/kolla/config_files/` — Kolla container configs
   - `/var/lib/openstack/` — OpenStack service data
   - Container startup configs under `/var/lib/edpm-config/container-startup-config/`
7. **Load project skills**: Always load the relevant project skill (e.g.,
   `edpm-ansible`) for repo structure and conventions before analyzing code.
