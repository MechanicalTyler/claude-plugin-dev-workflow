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
