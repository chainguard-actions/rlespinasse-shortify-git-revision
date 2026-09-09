<!-- markdownlint-disable -->

# Hardening Report: rlespinasse--shortify-git-revision/v1.6.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **rlespinasse--shortify-git-revision/v1.6.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference actions by mutable tag instead of a full 40-character commit SHA, making them vulnerable to supply-chain attacks if the tag is moved.

Failing references:
- `.github/workflows/linter.yml`: `actions/checkout@v7` (line 17), `super-linter/super-linter@v8` (line 22)
- `.github/workflows/shortify-git-revision.yaml`: `actions/checkout@v7` (lines 18, 165, 220), `rlespinasse/release-that@v1` (line 224)

All should be pinned to a full SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/linter.yml:17`
- `.github/workflows/linter.yml:22`
- `.github/workflows/shortify-git-revision.yaml:18`
- `.github/workflows/shortify-git-revision.yaml:165`
- `.github/workflows/shortify-git-revision.yaml:220`
- `.github/workflows/shortify-git-revision.yaml:224`

### broad-permissions (severity: medium)

Both workflow files set `permissions: read-all` at the top level. `read-all` is an overly broad permission grant that gives every job read access to all scopes. It should be replaced with specific minimal permissions per job.

Locations:

- `.github/workflows/linter.yml:6`
- `.github/workflows/shortify-git-revision.yaml:8`

### github-env-injection (severity: high)

In `shortify.sh`, the variables `${REVISION}` and `${SHORT_VALUE}` are written to `$GITHUB_OUTPUT` and `$GITHUB_ENV` without the required newline-stripping sanitization (`printf '%s' "$VAR" | tr -d '\n\r'`).

`REVISION` is derived directly from `INPUT_REVISION`, which is set to `${{ inputs.revision }}` in `action.yml`. An attacker-controlled value containing newlines could inject arbitrary key=value pairs into the runner's environment or output context.

Failing lines in shortify.sh:
```
echo "revision=${REVISION}" >> "$GITHUB_OUTPUT"   # line ~44
echo "short=${SHORT_VALUE}" >> "$GITHUB_OUTPUT"    # line ~45
echo "${PREFIX}${NAME}=${REVISION}"               # line ~49 (>> $GITHUB_ENV)
echo "${PREFIX}${NAME}_SHORT=${SHORT_VALUE}"       # line ~50 (>> $GITHUB_ENV)
```
Fix: sanitize each value with `safe=$(printf '%s' "$VAR" | tr -d '\n\r')` before writing.

Locations:

- `shortify.sh:44`
- `shortify.sh:45`
- `shortify.sh:49`
- `shortify.sh:50`

### script-injection (severity: high)

Multiple `run:` blocks in `.github/workflows/shortify-git-revision.yaml` directly interpolate `${{ env.* }}` and `${{ steps.*.outputs.* }}` expressions inside shell commands (sub-rule a). These values flow from the action's outputs, which are themselves derived from user-controlled inputs (`inputs.revision`, `inputs.name`, etc.). GitHub Actions substitutes these expressions into the shell script text before the shell executes it, allowing an attacker to inject shell metacharacters.

Examples of failing lines:
```
[[ "${{ env.ROOT_COMMIT }}" == "88428f56bd9d2751c47106bedfd148162dfa50b8" ]]
[[ "${{ env.ROOT_COMMIT_SHORT }}" == "88428f5" ]]
[[ "${{ steps.shortify-an-existing-git-revision.outputs.revision }}" == ... ]]
[[ "${{ steps.short-on-error.outputs.short }}" == "88428f5" ]]
```
Fix: move the values into `env:` variables and reference them as `"$ENV_VAR"` in the shell script.

Locations:

- `.github/workflows/shortify-git-revision.yaml:27`
- `.github/workflows/shortify-git-revision.yaml:40`
- `.github/workflows/shortify-git-revision.yaml:52`
- `.github/workflows/shortify-git-revision.yaml:62`
- `.github/workflows/shortify-git-revision.yaml:77`
- `.github/workflows/shortify-git-revision.yaml:89`
- `.github/workflows/shortify-git-revision.yaml:103`
- `.github/workflows/shortify-git-revision.yaml:117`
- `.github/workflows/shortify-git-revision.yaml:133`
- `.github/workflows/shortify-git-revision.yaml:147`
- `.github/workflows/shortify-git-revision.yaml:183`
- `.github/workflows/shortify-git-revision.yaml:199`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, broad-permissions, github-env-injection, script-injection

**Notes:**

Fixed all four findings:

1. **unpinned-uses**: Pinned all action references to full SHAs with tag comments:
   - actions/checkout@v7 → @3d3c42e5aac5ba805825da76410c181273ba90b1 # v7 (4 occurrences across both files)
   - super-linter/super-linter@v8 → @4ce20838b8ab83717e78138c5b3a1407148e0918 # v8
   - rlespinasse/release-that@v1 → @f4912d4053839003bb368e9c7067b071ccb1c146 # v1

2. **broad-permissions**: Removed top-level `permissions: read-all` from both workflow files. Added specific `contents: read` permissions to os-testing and error-os-testing jobs in shortify-git-revision.yaml. The release job already had specific permissions. linter.yml's build job already had specific job-level permissions.

3. **github-env-injection**: In shortify.sh, sanitized REVISION, SHORT_VALUE, PREFIX, and NAME with `printf '%s' "${VAR}" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT and $GITHUB_ENV.

4. **script-injection**: In shortify-git-revision.yaml, moved all ${{ env.* }} and ${{ steps.*.outputs.* }} expressions from run: shell scripts into env: blocks, then referenced them as plain environment variables (e.g., $ROOT_COMMIT instead of ${{ env.ROOT_COMMIT }}).

