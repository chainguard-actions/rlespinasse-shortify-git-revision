<!-- markdownlint-disable -->

# Hardening Report: rlespinasse--shortify-git-revision/v1.6.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **rlespinasse--shortify-git-revision/v1.6.0** was hardened automatically. 1 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

shortify.sh writes values derived from untrusted inputs directly to $GITHUB_OUTPUT and $GITHUB_ENV without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). Specifically:

1. `echo "revision=${REVISION}" >> "$GITHUB_OUTPUT"` — REVISION comes from INPUT_REVISION (mapped from inputs.revision) with no newline stripping.
2. `echo "short=${SHORT_VALUE}" >> "$GITHUB_OUTPUT"` — SHORT_VALUE is derived from git operations on the attacker-controlled REVISION.
3. `echo "${PREFIX}${NAME}=${REVISION}" >> "$GITHUB_ENV"` and `echo "${PREFIX}${NAME}_SHORT=${SHORT_VALUE}" >> "$GITHUB_ENV"` — PREFIX and NAME come from INPUT_PREFIX/INPUT_NAME (inputs.prefix, inputs.name), and REVISION/SHORT_VALUE are input-derived.

An attacker who controls any of these inputs could inject newlines to set arbitrary environment variables or outputs, potentially hijacking subsequent steps in the calling workflow.

Locations:

- `shortify.sh:47`
- `shortify.sh:48`
- `shortify.sh:53`
- `shortify.sh:54`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed shortify.sh by sanitizing all untrusted input-derived values before writing to $GITHUB_OUTPUT and $GITHUB_ENV. Added four sanitization lines using `printf '%s' "${VAR}" | tr -d '\n\r'` to create SAFE_REVISION, SAFE_SHORT_VALUE, SAFE_PREFIX, and SAFE_NAME variables. All four writes to $GITHUB_OUTPUT and $GITHUB_ENV now use these sanitized variables instead of the raw input-derived ones, preventing newline injection attacks that could allow an attacker to set arbitrary environment variables or outputs.

