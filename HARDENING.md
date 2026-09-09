<!-- markdownlint-disable -->

# Hardening Report: rlespinasse--shortify-git-revision/v1.6.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **rlespinasse--shortify-git-revision/v1.6.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

shortify.sh writes unsanitized, attacker-controlled values to $GITHUB_OUTPUT and $GITHUB_ENV without the required `printf '%s' ... | tr -d '\n\r'` sanitization step. Specifically: (1) `echo "revision=${REVISION}" >> "$GITHUB_OUTPUT"` and `echo "short=${SHORT_VALUE}" >> "$GITHUB_OUTPUT"` — REVISION is derived from INPUT_REVISION (inputs.revision) or from ${!NAME} (an env var named by inputs.name), both untrusted; (2) `echo "${PREFIX}${NAME}=${REVISION}"` and `echo "${PREFIX}${NAME}_SHORT=${SHORT_VALUE}"` piped to $GITHUB_ENV — PREFIX and NAME come from inputs.prefix and inputs.name respectively. A newline injected into any of these values can add arbitrary key=value pairs to the runner's environment or output context.

Locations:

- `shortify.sh:44`
- `shortify.sh:45`
- `shortify.sh:51`
- `shortify.sh:52`

### script-injection (severity: high)

Multiple `run:` blocks in the workflow directly interpolate GitHub Actions expressions inside shell command strings (sub-rule a). Examples include: `[[ "${{ env.ROOT_COMMIT }}" == "88428f56bd9d2751c47106bedfd148162dfa50b8" ]]`, `[[ "${{ env.ROOT_COMMIT_SHORT }}" == "88428f5" ]]`, `[[ "${{ steps.shortify-an-existing-git-revision.outputs.revision }}" == ... ]]`, `[[ "${{ steps.test-shortify-wrong-git-revision.outcome }}" == "failure" ]]`, etc. These expressions are expanded by the Actions template engine before the shell ever sees them, allowing injection of shell metacharacters if the values are attacker-influenced.

Locations:

- `.github/workflows/shortify-git-revision.yaml:25`
- `.github/workflows/shortify-git-revision.yaml:40`
- `.github/workflows/shortify-git-revision.yaml:55`
- `.github/workflows/shortify-git-revision.yaml:68`
- `.github/workflows/shortify-git-revision.yaml:82`
- `.github/workflows/shortify-git-revision.yaml:95`
- `.github/workflows/shortify-git-revision.yaml:108`
- `.github/workflows/shortify-git-revision.yaml:122`
- `.github/workflows/shortify-git-revision.yaml:136`
- `.github/workflows/shortify-git-revision.yaml:150`

### unpinned-uses (severity: high)

The workflow references actions by mutable tag/version refs instead of full 40-character SHA commit digests, making the workflow vulnerable to supply-chain attacks if those tags are moved or hijacked. Failing references: `actions/checkout@v3` (appears 3 times) and `rlespinasse/release-that@v1` (appears once).

Locations:

- `.github/workflows/shortify-git-revision.yaml:16`
- `.github/workflows/shortify-git-revision.yaml:168`
- `.github/workflows/shortify-git-revision.yaml:204`
- `.github/workflows/shortify-git-revision.yaml:211`

### missing-permissions (severity: medium)

The workflow file has no top-level `permissions:` key and none of its three jobs (os-testing, error-os-testing, release) define a job-level `permissions:` block. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (write access to contents, packages, etc.).

Locations:

- `.github/workflows/shortify-git-revision.yaml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings: (1) shortify.sh now sanitizes REVISION, SHORT_VALUE, PREFIX, and NAME with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_OUTPUT and $GITHUB_ENV; (2) all ${{ }} expressions in workflow run: blocks moved to env: blocks with plain shell variable references; (3) actions/checkout@v3 pinned to SHA a37ce9120846195fa4ece8f58b268e6043cb2f26 (3 occurrences) and rlespinasse/release-that@v1 pinned to SHA f4912d4053839003bb368e9c7067b071ccb1c146; (4) added top-level `permissions: {}` and per-job permissions blocks (os-testing and error-os-testing get `{}`, release gets `contents: write` for release creation).

