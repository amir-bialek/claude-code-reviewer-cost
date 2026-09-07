# Review context template

Copy this to `.github/claude-review-context.md` in the repo you want reviewed and fill it in.
The workflow fails if the file is missing or empty. Only the first three sections are required.

## About this repo

What this repo produces, what runs in CI, and which directories hold the code that matters.
Say what the repo does not own (vendored code, generated files, another pipeline's scripts),
so the reviewer does not waste the run on it.

## Do flag (repo-specific pitfalls)

Three to six pitfalls that are real in this repo, each with a concrete example of a change that
would trigger it. Write them as pattern families, not as a checklist — the reviewer applies its
own judgement too, and these are the anchors for what is easy to miss here.

- **Name of the pitfall.** Why it breaks, and which files it lives in.
  *Examples:* a specific line or change that would be wrong, and why.

## Don't flag

- Style, naming, formatting, comment wording.
- Vendored or generated code this repo does not own.
- Missing tests, if the repo has no test suite.
- Praise, summaries, restating what the diff does.
- Hypothetical issues not tied to a line in this PR.

## Skip review if only these paths changed

Optional. One glob per line. If a PR touches nothing else, the review is skipped.

- docs/**
- *.md

## Coupled files

Optional. Files that must change together, so the reviewer puts them in the same slice
instead of splitting them by directory.

- src/api/schema.py and src/client/types.ts

## Exclude from size calculation

Optional. Paths that should not count towards the small-PR / large-PR routing threshold,
such as lockfiles or generated output.

- package-lock.json
