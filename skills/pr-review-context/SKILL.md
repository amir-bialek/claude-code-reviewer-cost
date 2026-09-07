---
name: pr-review-context
description: Use when the user wants to write or update the repo-specific `.github/claude-review-context.md` file that the automated `claude-code-review` GitHub Action reads, so the review comments point at this repo's actual pitfalls instead of generic advice. Reads the repo — build files, CI workflows, recent merged PRs — and writes exactly one file. Does not touch any GitHub Actions workflow.
---

You are writing the repo-specific context file that the `claude-code-review` GitHub Action reads on every pull request. It is what turns a generic review into one that knows this repo's actual pitfalls. This skill produces exactly one file, `.github/claude-review-context.md`, and nothing else. It does not create or edit any GitHub Actions workflow.

The workflow fails the job if the file is missing or empty, and it inlines the whole file into the reviewer's prompt — so everything you write here is read on every PR, and length costs money on every PR.

## Step 1 — Read the repo before writing anything

Spend a few minutes reading:

- `README.md` and any `CLAUDE.md` — what the repo is, what's vendored vs in-house, what the deploy path looks like. Read these yourself; you need the same understanding to write the "About this repo" paragraph.
- `Dockerfile*`, build files and `.github/workflows/*` — what gets built, how, and what footguns the build has. If the repo has more than ~3 build files or the workflow files are long/complex, **delegate this to a sub-agent** with the Agent tool. The sub-agent's job is to surface **concrete tripwires grouped under invariant families** — both halves matter. Brief it to return **3–5 invariant families**, each with: a headline naming the invariant, one or two sentences on the mechanism that makes it fragile in this repo, and **2–3 concrete real tripwires from the code** (what the change would look like, what would break) as anchors under the family. Returning only invariants (no examples) or only a flat bug list (no grouping) are both failures — you need both together, because the examples are what you will paraphrase into the *Examples:* sub-list of each final Do-flag bullet.
- Recent commits or PRs (`gh pr list --state merged --limit 10`) — what kinds of mistakes have been caught recently. **Delegate this scan to a sub-agent.** Pulling 10–30 PRs with their diffs and review comments will balloon the main context for a small synthesized output. Same shape as above: the sub-agent should return **3–5 invariant families** that the historical bugs cluster under, each with a headline, the mechanism, and **2–3 concrete real bugs** (what was changed, what broke, what the fix was) as examples beneath it. The content of each bug is what matters — PR numbers are optional and only useful for your own sanity check, not as deliverables. Don't return only invariants (no examples) and don't return only a flat bug list (no grouping); you need both, because the bugs become the *Examples:* sub-list under each invariant headline.
- The directory structure — where the real surface area is vs vendored/generated code. Do this yourself.

## Step 2 — Write the required sections

Write `.github/claude-review-context.md` with these three sections, in this order. The heading text matters: the workflow's own error message names them, and the reviewer's prompt is written around them.

```markdown
## About this repo

<3-6 sentences>

## Do flag (repo-specific pitfalls)

These are illustrative pattern families, not an exhaustive checklist.

- **<Invariant name>.** <One-line statement of the invariant.>
  <Sentence or two on *why this matters in this repo* — the mechanism that would break.>
  *Examples:* <1-3 concrete repo-specific tripwires.>

## Don't flag

- <bullet>
```

### About this repo

3–6 sentences in plain English. What this repo *is*. What's vendored from upstream vs original work (if applicable). Where the deploy goes. Which directories are the actual surface area for review. This grounds the reviewer so it doesn't waste comments on out-of-scope files. Keep it tight.

### Do flag (repo-specific pitfalls)

**3–6 themed pattern families**, not a fingerprint list of individual bugs. Each family states an underlying invariant of the repo, explains the mechanism that makes it fragile here, and ends with 1–3 concrete examples as anchors. The reviewer is an LLM; if you give it literal strings it will anchor on them and miss semantically equivalent regressions, so the bullet's *headline* must be the invariant, not the tripwire. But an invariant on its own is too vague to act on — the concrete examples underneath are what give the reviewer traction. Both halves are required.

Your sub-agents from Step 1 already did most of the synthesis: each returned invariant families with mechanism + 2–3 concrete bugs/tripwires under each. Your job here is to polish that into the final bullet shape — keep their invariant headlines (tighten the wording if needed), keep their mechanism sentences, and paste/paraphrase their concrete bugs into the *Examples:* sub-list. Don't strip the examples down to abstractions, and don't promote any single example to the headline.

Format each bullet as:

- **Pattern name.** One-line statement of the invariant.
  Sentence or two on *why this matters in this repo* — the mechanism that would break.
  *Examples:* 1–3 concrete repo-specific tripwires (from the sub-agent output, paraphrased).

Aim for 3–6 families. If you have 8–10 candidate tripwires, group them — most cluster under 3–4 invariants (composition integrity, dependency hygiene, CI lockstep, contract preservation, etc.). Keep the total length of this section comparable to §About this repo, not 3× it — it is read on every PR and its volume dominates attention.

Here is one fully-worked family, from the example context file shipped with this repository (`examples/claude-review-context.example.md`) — note the invariant headline, the mechanism sentence, and the examples sub-list:

> - **CI lockstep across workflows.** Paths-filters, concurrency groups and the fork-vs-same-repo
>   conditions in the validate and build workflows must stay synchronized; breaking the lockstep
>   produces races, missing rebuilds, or fork PRs that try to push without credentials.
>   *Examples:* the `detect-changes` filter listing a dockerfile path in one workflow but not the
>   other; the mirrored fork-PR conditions (login `if:`, build-push `push:`, build-push `load:`)
>   moved apart.

Read the rest of that example file for what a finished set of families looks like. For the repo you're currently writing for, find the equivalent invariants by reading its build files, CI and `CLAUDE.md`. If you genuinely can't find any repo-specific patterns, ask the user before falling back to a generic list.

### Don't flag

3–6 bullets. Things that waste the author's time in *this* repo. Common ones:

- Style/structure of vendored upstream code — only flag if the diff introduces a real bug there.
- Missing tests, when the repo has no test suite wired into CI.
- Type-checking or strictness gaps inherited from upstream.
- Praise, summaries, or restating the diff.
- Hypothetical issues not tied to a specific line.
- "Please add a comment/docstring" unless the absence is genuinely confusing.

## Step 3 — Add the optional sections the workflow acts on

Three further sections are optional, but the workflow reads them mechanically, so the heading wording has to be exact. Add one only when the repo actually needs it — an empty or invented section is worse than none.

### `## Skip review if only these paths changed`

A precheck step in the workflow extracts this section and skips the whole review when a push touches nothing outside these globs. This is the one section that directly saves money, so it is worth filling in for any repo with a lot of docs or generated churn.

How it is parsed, and what that means for how you write it:

- The heading must be a level-2 heading with this exact text (matched case-insensitively). The section ends at the next `##` heading.
- One glob per list item, starting with `-` or `*`. Only the first whitespace-delimited token of each item is used, so a trailing note after the glob is harmless. Backticks are stripped.
- A glob must be made only of letters, digits and `_ . / * ? { } + -`; anything else is dropped silently.
- A catch-all (`*`, `**`, `*/*`) is rejected on purpose — it would disable review entirely.
- The workflow reads this section from the file **on the base branch**, so a change to the list only takes effect once it is merged.

```markdown
## Skip review if only these paths changed

- docs/**
- *.md
```

### `## Coupled files`

On large PRs the reviewer splits the diff into bins by directory. Files listed here are always put in the same bin instead, so a change and its counterpart get reviewed together. Use it for pairs that must change in lockstep across directory boundaries.

```markdown
## Coupled files

- src/api/schema.py and src/client/types.ts
```

### `## Exclude from size calculation`

The reviewer routes small and large PRs down different paths based on changed-line count. Paths listed here don't count towards that threshold — lockfiles, snapshots, generated output, anything that inflates the number without being real review surface.

```markdown
## Exclude from size calculation

- package-lock.json
```

## Step 4 — Sanity-check before handing back

- `.github/claude-review-context.md` exists, is non-empty, and has all three required sections (`## About this repo`, `## Do flag (repo-specific pitfalls)`, `## Don't flag`) in that order.
- Any optional section you added uses the exact heading text from Step 3, and its list items are plain `-` bullets. A typo in one of these headings fails silently — the workflow simply behaves as if the section were absent.
- The `Do flag` list is genuinely repo-specific **and shaped as pattern families, not fingerprints**. **Delegate this check to a fresh sub-agent** with the Agent tool — you wrote the list, so you are the worst judge of whether it reads well. Pass the sub-agent the final Do-flag list plus a one-line summary of the repo's purpose, and ask all three:
  1. "Do these bullets read as genuinely specific to this repo, or could they apply to any GitHub project?"
  2. "Does each bullet lead with an invariant/pattern name, or does it lead with a literal file/flag? If the latter, the headline should be lifted to the invariant and the literal demoted to an example."
  3. "Does each bullet have 1–3 concrete examples under *Examples:*, or is it invariant-only? Invariant-only bullets are too vague to act on and should have concrete tripwires added back."

  If the sub-agent flags any bullet on any of the three, fix it before handing back. Note: the sub-agent should not be asked "is this too specific?" — concrete examples are required, and the invariant headline is what generalizes the pattern.

Do not run `git` commands or commit the context file — leave that to the user.
