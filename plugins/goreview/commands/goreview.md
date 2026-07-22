---
description: "Run GoLegends' named Go judges for cited deduction scorecards. Read-only by default; --fix deliberates, edits, verifies, and re-reviews."
argument-hint: "[list] [--fix] [--max-rounds N] [judge...] [-- <scope path or pr N>]"
---

Arguments: `$ARGUMENTS`

Read `${CLAUDE_PLUGIN_ROOT}/protocol.md` completely before acting. It is the
host-neutral contract. This command only supplies Claude-specific parsing,
workflow dispatch, locking, and rendering.

## Load the review

Read `${CLAUDE_PLUGIN_ROOT}/review.json` as JSON. Validate that its `id` is
`goreview`; every declared judge path exists under
`${CLAUDE_PLUGIN_ROOT}`, matches the judge's frontmatter `name`, and is listed by
`.claude-plugin/plugin.json`; and `fixer` matches `fixer.md`. Stop on any
missing, malformed, or inconsistent input. Pass the parsed review object to
every Workflow call. Do not load `policy.md` for `list` or read-only review.

If `.goreview.json` exists at the reviewed repository root, parse it as
JSON. It may contain only `judges` and `maxReviewRounds`. Reject unknown fields,
unknown judges, duplicate judges, and invalid round values rather than guessing.

## Parse arguments

Split on the first literal `--`; everything after it is the scope. Before it:

- `list` is metadata-only and cannot be combined with other options.
- `--fix` enables writes.
- `--max-rounds N` requires `--fix`; N must be an integer from 2 through the
  review's `maxAllowedReviewRounds`.
- Remaining tokens are judge labels. Accept and strip only the exact
  `goreview:` namespace.

Read-only judge precedence is explicit labels, repository `judges`, then
`defaultJudges`. Fix mode ignores repository judges: explicit labels win;
otherwise omit `judges` so the workflow selects three. Fix-round precedence is
`--max-rounds`, repository `maxReviewRounds`, then `defaultMaxReviewRounds`.
Read-only always runs one round.

## Dispatch

For `list`, call:

```text
Workflow({
  scriptPath: "${CLAUDE_PLUGIN_ROOT}/workflow.js",
  args: { inspect: true, review: <parsed review.json> }
})
```

For read-only review, call the same workflow with `apply: false`, the review,
scope, and selected `{label}` records. Each judge receives only that scope and
its canonical rubric; never pass repository or plugin house style.

For `--fix`, load `${CLAUDE_PLUGIN_ROOT}/policy.md` once and extract the first
line matching `^Version:[[:space:]]*[0-9]+[[:space:]]*$`. The policy is fixer
guidance only and must never be passed to judges, deliberators, or the chair.
Warn that files will change and ask the user not to edit the scope. Acquire an
atomic repository lock with:

```bash
mkdir "$(git rev-parse --git-path goreview-fix.lock)"
```

Never remove an existing lock. Call the workflow with `apply: true`,
`lockHeld: true`, the resolved `maxReviewRounds`, and the same review, scope,
fixer policy, `policySource: "policy.md@<version>"`, and optional explicit
judges. Await the write-capable fixer without abandonment. After the Workflow
call returns, release only the lock this run acquired with `rmdir`.

## Render

Identify the returned `plugin`, `language`, and selected judges. Print each
`scores[].scorecard` verbatim in selection order; these blocks are already
rendered from canonical deductions by the engine. Report `reviewRounds`,
`fixAttempts`, `maxReviewRounds`, selection provenance, and any deliberation
chair or resolved disagreement. In fix mode, report fixer-policy provenance;
do not describe it as review or score provenance.

Print **ACCEPTED** only for `verdict === "ACCEPTED"`. Handle `INSPECT`,
`INVALID_REQUEST`, `REVIEW_ONLY`, `JUDGES_UNAVAILABLE`, `BUDGET_EXHAUSTED`,
`FIX_FAILED`, `SCOPE_EXPLOSION`, and `STALL` according to `protocol.md`. Print
an unknown verdict and the complete result verbatim and stop; never infer a
pass. `FIX_FAILED` means the working tree may contain partial edits.
