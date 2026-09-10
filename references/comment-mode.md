# GitHub comment mode

Use this mode only when `--comment` is present or the user otherwise explicitly requests PR comments.

## Preconditions

Resolve one open GitHub PR and record its repository, number, base SHA, and full head SHA. Confirm each finding still applies to that head before posting. Read existing review comments on the same head and skip an equivalent root-cause comment.

If the environment has a GitHub inline-comment connector, use it. Otherwise use `gh api` with the GitHub pull-request review-comments endpoint. Do not approve the PR, request changes, submit an overall review state, or post comments to issues or unrelated PRs.

## Findings

Post one inline comment per unique retained finding on a changed line. Each body contains:

- A concise `[P0]` through `[P3]` title.
- The concrete trigger and observable impact in one short paragraph.
- A permanent GitHub link using the real repository, full head SHA, path, and accurate line range.
- For a project-rule finding, a second permanent link to the exact applicable rule.

Use the PR's `RIGHT` side for added or modified code and `LEFT` only for removed code. Anchor the comment to the narrowest changed line range that demonstrates the problem.

For a self-contained fix of at most five lines, a GitHub `suggestion` block is allowed only when accepting it completely fixes the issue. The block replaces the entire selected range, so include syntactically complete replacement text. Do not use a suggestion for structural, multi-location, generated-file, or partially corrective changes.

## No findings

When no finding survives filtering, post one top-level PR comment:

```markdown
## Code review

No issues found. Checked for bugs and project-rule compliance.
```

## Completion report

Return the number of inline comments posted, duplicates skipped, and failures. Include direct links returned by GitHub when available. Never claim a comment was posted when the API call failed.
