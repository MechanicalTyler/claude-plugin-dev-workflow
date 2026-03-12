# PM Adapter: GitHub Issues

Story ID format: `#XXX` or numeric `XXX`

## Fetch Story

```bash
gh issue view {number} --json title,body,comments,labels,state
```

Parse the JSON to extract title, description (body), and comments array.

## Post Comment

```bash
gh issue comment {number} --body "{text}"
```

## Update Story

```bash
# Add label
gh issue edit {number} --add-label "{label}"

# Set milestone
gh issue edit {number} --milestone "{milestone-name}"

# Close issue
gh issue close {number}
```

## Story Reference in PRs

Format: `Closes #XXX` (auto-closes issue on merge)

Or: `Issue: #XXX` (if you don't want auto-close)

Include this in the PR body so reviewers can find the original requirements.

## Create Story

Use the `gh` CLI to create a new issue. Write the body to a temp file first to avoid shell interpolation issues with multi-line markdown:

```bash
# Write body to temp file
# Then create the issue using --body-file
gh issue create --title "{title}" --body-file .scratch/tmp/gh-issue-body.md
```

Write the body content to `.scratch/tmp/gh-issue-body.md` using the Write tool before running the command.

### Body format

Construct the body as:

```
## Original Request
{originalRequest}

---

## Story

{description}

**Repo to modify:** {repoToModify}

**Repos to reference:** {reposToReference joined with ", " or "(none)" if empty}

**Acceptance Criteria**
- [ ] {ac item 1}
- [ ] {ac item 2}
...

**Testing Instructions**
1. {step 1}
2. {step 2}
...
```

Return: the created issue number (e.g., `#42`) and URL for confirmation.
