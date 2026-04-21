---
name: gerrit-code-review
description: Review Gerrit code changes (OpenDev, GerritHub, Google, etc.) using the REST API. Fetches patches, metadata, and file context via curl — no authentication needed for public instances. Use when the user shares a Gerrit review URL.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [Gerrit, Code-Review, OpenStack, OpenDev]
    related_skills: [github-code-review]
---

# Gerrit Code Review

Review code changes on Gerrit instances (OpenDev, GerritHub, Android, etc.) using the REST API. No authentication needed for public reviews.

## When to Use

- User shares a URL like `https://review.opendev.org/c/openstack/heat/+/983528`
- User asks to review a Gerrit change by number
- Any URL containing `/c/` and a change number on a Gerrit instance

## URL Parsing

Gerrit URLs follow the pattern:
```
https://{host}/c/{project}/+/{change_number}
```

Extract:
- `HOST` — e.g., `review.opendev.org`
- `PROJECT` — e.g., `openstack/heat` (URL-encode as `openstack%2Fheat`)
- `CHANGE_ID` — e.g., `983528`

## Approach: Use REST API, NOT the Browser

**IMPORTANT**: Do NOT try to read Gerrit diffs via browser_snapshot. Gerrit's web UI
uses a complex Shadow DOM and the accessibility tree is extremely verbose, truncated,
and difficult to parse. The REST API is far more reliable.

## Step 1: Fetch the Patch (Most Important)

```bash
# Get the unified diff — Gerrit returns base64-encoded patch
curl -s "https://{HOST}/changes/{PROJECT_ENCODED}~{CHANGE_ID}/revisions/current/patch" | base64 -d
```

This returns a complete git-format patch with:
- Commit message (subject, body, bug references, Change-Id)
- Full unified diff for all files
- File paths, line numbers, additions, deletions

This single call gives you everything needed to review the code.

## Step 2: Fetch Change Metadata

```bash
# Gerrit REST API prefixes JSON with )]} for XSSI protection — strip with tail
curl -s "https://{HOST}/changes/{PROJECT_ENCODED}~{CHANGE_ID}/detail" | tail -c +5 | python3 -m json.tool
```

Useful fields:
- `subject` — change title
- `status` — NEW, MERGED, ABANDONED
- `owner.name` — author
- `branch` — target branch
- `project` — repository
- `labels` — Code-Review, Verified, Workflow votes
- `messages` — reviewer comments
- `contains_git_conflicts` — merge conflict status

## Step 3: Fetch Surrounding Code Context

When the diff alone isn't enough, fetch the full file from the repo:

**For OpenDev (Gitea-backed):**
```bash
curl -sL "https://opendev.org/{PROJECT}/raw/branch/{BRANCH}/{FILE_PATH}" | sed -n '{START},{END}p'
```

**For GitHub-mirrored repos:**
```bash
curl -sL "https://raw.githubusercontent.com/{PROJECT}/{BRANCH}/{FILE_PATH}" | sed -n '{START},{END}p'
```

**Via Gerrit API (works on any instance):**
```bash
curl -s "https://{HOST}/changes/{PROJECT_ENCODED}~{CHANGE_ID}/revisions/current/files/{FILE_PATH_ENCODED}/content" | base64 -d
```

## Step 4: Check for Related Information

```bash
# Get reviewer comments/messages
curl -s "https://{HOST}/changes/{PROJECT_ENCODED}~{CHANGE_ID}/detail" | tail -c +5 | \
  python3 -c "
import sys, json
d = json.load(sys.stdin)
for m in d.get('messages', []):
    print(f\"{m.get('author',{}).get('name','?')}: {m.get('message','')[:200]}\")
    print('---')
"

# Get related changes (dependencies)
curl -s "https://{HOST}/changes/{PROJECT_ENCODED}~{CHANGE_ID}/related" | tail -c +5
```

## Review Output Format

Present the review in a structured terminal-friendly format:

```
========================================================================
  CODE REVIEW: {Subject}
  Change: {URL}
  Author: {Owner}
  Branch: {branch}
  Bug: {bug reference if any}
  Status: {status}
========================================================================

SUMMARY
-------
{Brief description of what the change does and why}

CHANGES ({N} files, +{added}/-{removed})
-----------------------------------------
{For each file: path, what changed, key lines}

ASSESSMENT
----------
{Analysis of correctness, edge cases, code quality}

ISSUES / SUGGESTIONS
--------------------
{Numbered list with severity: CRITICAL / MEDIUM / MINOR / NOTE}

VERDICT
-------
{Overall assessment and rating}
```

## Pitfalls

1. **XSSI prefix**: Gerrit REST JSON responses start with `)]}'` — always strip with `tail -c +5` before parsing
2. **URL encoding**: Project names with `/` must be URL-encoded (e.g., `openstack%2Fheat`)
3. **OpenDev raw files**: The `opendev.org` Gitea instance often times out (even at 15s+ timeout) — prefer GitHub raw mirrors for OpenStack projects: `https://raw.githubusercontent.com/openstack/{repo}/{branch}/{path}`
4. **Browser approach fails**: Gerrit's web UI renders diffs in Shadow DOM components. The accessibility tree is huge, truncated, and unusable for extracting actual code changes. Always use the REST API.
5. **No auth needed**: Public Gerrit instances (OpenDev, Android, Chromium) allow anonymous read access to the REST API
