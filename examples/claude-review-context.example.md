# Review context example

A real filled-in context file, from a repo whose only artifacts are Docker images built in CI.
Names and paths are changed. Use it to see the level of detail that works, then write your own
from the template.

## About this repo

`example-org/build-images` publishes Docker images that self-hosted GitHub Actions runners
consume. There is no application code — the artifacts are the images themselves. The `dev` image
is a composition: `docker/dev.dockerfile` does `FROM ${UBUNTU_BASE}` then
`COPY --from=${CUDA_BASE}` of the CUDA tree and `/usr/local/{include,lib,share}`, with no `RUN`
steps of its own. Most C++ libraries arrive as pre-built tarballs from an internal artifact
store — this repo doesn't own that source. The real surface area for review is `docker/` and
`.github/workflows/`.

## Do flag (repo-specific pitfalls)

These are illustrative pattern families, not an exhaustive checklist.

- **Build composition integrity.** Installs in the base images must not overwrite each other or
  break linkage in the composed `dev` image. The dev image does a blanket
  `COPY --from=${CUDA_BASE} /usr/local/{include,lib,share}`, so anything the CUDA base writes
  there silently wins over the Ubuntu base's copy.
  *Examples:* dropping the `--exclude='libfmt.a'` guard on the libtorch `tar -xz` (libtorch's
  bundled fmt would overwrite the standalone fmt, breaking spdlog's external-fmt linkage); a new
  install step in the CUDA base writing into `/usr/local` without checking for collisions.

- **A version duplicated with no cross-check.** `CUDA_VERSION` is repeated as an independent
  literal in the CUDA dockerfile, the dev dockerfile's ARG default, and the build-args of two
  workflows. Nothing enforces that these stay in sync.
  *Examples:* bumping the dockerfile without updating the workflows, producing a
  `COPY --from=cuda /usr/local/cuda-${CUDA_VERSION}` path that was never built.

- **Dependency hygiene.** Every external download must be version-pinned in the URL (no
  `latest`), and credentials must only come via BuildKit
  `--mount=type=secret` — never `ARG` or `ENV`.
  *Examples:* a new `curl` line with no pinned version; `ARTIFACT_STORE_PASSWORD` referenced as an `ARG` and
  baked into a layer.

- **CI lockstep across workflows.** Paths-filters, concurrency groups and the fork-vs-same-repo
  conditions in the validate and build workflows must stay synchronized; breaking the lockstep
  produces races, missing rebuilds, or fork PRs that try to push without credentials.
  *Examples:* the `detect-changes` filter listing a dockerfile path in one workflow but not the
  other; the mirrored fork-PR conditions (login `if:`, build-push `push:`, build-push `load:`)
  moved apart.

- **Tag scheme assumptions.** The shell that computes the next tag encodes a specific scheme.
  *Examples:* `dev-vN` is integer-bumped via `git tag -l | sort -n | tail -n 1` + 1; switching it
  to semver breaks the tag computation.

## Don't flag

- Style, structure or internals of vendored upstream code — this repo doesn't own that source.
- Missing tests. There is no test suite here; nothing in CI compiles or runs code against the
  published images.
- Requests to add comments or docstrings to dockerfiles unless the absence is genuinely
  confusing.
- Praise, summaries, or restating what the diff does.
- Hypothetical issues not tied to a specific line in this PR.
