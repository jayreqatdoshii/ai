---
name: jira-branch-pr
description: Create git branches and GitHub PRs that are linked to a Jira ticket. Use whenever the user is starting work on / creating a branch for a Jira ticket, or opening a pull request that should reference a Jira ticket. Triggers on a Jira ticket key (e.g. ENG-377) or Jira URL (e.g. https://doshii.atlassian.net/browse/ENG-377) together with a request to create a branch, start work, or open/raise a PR.
---

# Jira-linked branches & PRs

Enforce a consistent link between git work and its Jira ticket. Every branch name, PR title, and PR description created through this skill must reference the ticket.

## 1. Resolve the ticket key

The user supplies either:
- a bare key: `ENG-377`
- a full URL: `https://doshii.atlassian.net/browse/ENG-377`

Extract the **ticket key** (`ENG-377`) and the **ticket URL**. If given only the key, build the URL as `https://doshii.atlassian.net/browse/<KEY>`. If given only the URL, extract the key from the `/browse/<KEY>` segment.

The key format is `<PROJECT>-<NUMBER>` (uppercase letters, hyphen, digits). Uppercase the project part if the user typed it lowercase (`eng-377` → `ENG-377`).

If no ticket key can be determined, ask the user for it before doing anything else — do not create an unlinked branch or PR.

## 2. Fetch ticket details (for naming)

Use the Atlassian MCP `getJiraIssue` tool (via ToolSearch: `select:mcp__atlassian__getJiraIssue`) to fetch the issue by its key. Use the returned **issue type** and **summary** to derive:

- **Semantic prefix** from the issue type:
  - Story / Task / Sub-task / Improvement → `feature`
  - Bug / Defect → `fix`
  - Chore / maintenance / dependency work → `chore`
  - Spike / research → `spike`
  - If unclear, default to `feature`.
- **Branch details**: kebab-cased short slug from the summary (lowercase, alphanumeric + hyphens, ~3-6 words, no trailing hyphen).

If the MCP fetch fails or is unavailable, ask the user for a short description and the change type instead of guessing.

## 3. Create the branch

Branch naming — **semantic, ticket-key second**:

```
<prefix>/<KEY>-<branch-details>
```

Example: `feature/ENG-377-partner-sdk-order-status`

Steps:
1. Confirm the current repo is the right one (`git remote get-url origin`) and the working tree state.
2. Create the branch off the default/base branch unless the user specifies another base:
   ```
   git switch -c feature/ENG-377-partner-sdk-order-status
   ```
   (Fetch/update the base branch first if needed.)

Show the proposed branch name and let the user adjust the slug before creating it if there's any ambiguity.

## 4. Open the PR

Use `gh pr create`. The PR must satisfy all three rules:

- **Title** is prefixed with the key + colon:
  ```
  ENG-377: <concise feature description>
  ```
- **Description** references the Jira ticket as the very first line:
  ```
  Jira ticket: https://doshii.atlassian.net/browse/ENG-377
  ```
- Body then continues with the usual sections (Summary / Changes / Testing / etc.).

Example:

```bash
gh pr create \
  --title "ENG-377: expose received order status in partner SDK" \
  --body "$(cat <<'EOF'
Jira ticket: https://doshii.atlassian.net/browse/ENG-377

## Summary
<what changed and why>

## Changes
- <bullet>

## Testing
- <how it was verified>
EOF
)"
```

Never create the PR unless the branch has been pushed; push with `git push -u origin <branch>` first if the upstream isn't set.

## Checklist before finishing

- [ ] Branch name matches `<prefix>/<KEY>-<slug>`
- [ ] PR title starts with `<KEY>: `
- [ ] PR body's first line is `Jira ticket: <URL>`
- [ ] Report the created branch name and PR URL back to the user
