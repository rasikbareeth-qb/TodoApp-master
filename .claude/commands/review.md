---
description: "Read-only iOS code review. /review = issues in changed lines only; /review all = changed lines (section 1) + whole existing codebase (section 2)"
argument-hint: "[all | file/path | branch | PR number]"
---

Launch the `ios-code-reviewer` subagent (Agent tool, `subagent_type: "ios-code-reviewer"`). Put the mode rules below into its prompt verbatim — they override any conflicting scope or output rules in the agent file or the skill.

**Argument:** $ARGUMENTS

## Changed lines (used by every mode)
"Changed lines" = uncommitted working-tree changes (`git diff HEAD` + untracked Swift files) plus the current branch's commits vs `main` (`git diff main...HEAD`). Ignore `xcuserdata/` and `.DS_Store`.

## Modes
- **Empty argument → diff-only review.** Flag issues only in changed lines (the skill's "changed lines only" rule applies). Output a single section `### 1. Issues in changed lines`. Do NOT review or report pre-existing code; omit section 2 entirely.
- **`all` → full review, two sections, in this order:**
  - `### 1. Issues in changed lines` — same as diff-only mode.
  - `### 2. Issues in existing codebase` — every Swift file under `Today/` plus `Info.plist`/pbxproj privacy & deployment settings, as currently on disk. The skill's "changed lines only" rule does NOT apply to this section. Do not repeat anything already reported in section 1.
- **A number → PR review.** Changed lines = `gh pr diff <n>`; single section 1 only.
- **Anything else → file, directory or branch.** For a branch, changed lines = `git diff main...<branch>`. For a file/directory, report only section 1 for changed lines inside it; if it has no changes, review the path as existing code under section 2.

## Rules for every mode
- Follow `docs/ios-code-review-guidelines.md` and apply the review criteria in `.claude/skills/ios-review/skill.md` (how to judge code). Skip skill steps that need `mcp__cupertino__*` tools, `gh pr comment`, preflight, or writing output files; list skipped steps briefly.
- Strictly read-only.
- In each section, group findings by severity (**Critical**, **High**, **Medium**, **Low**, **Nit**) with `file:line` and a one-line suggested fix. Write "No findings" if a section is empty.
- Start with a 2–3 line summary and a verdict per section; end with good practices observed and the top 3 fixes.

When the agent finishes, relay its findings to the user unchanged in substance. Do not apply any fixes unless the user asks for them afterwards.
