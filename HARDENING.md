<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.1** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml use mutable version tags (@v2) instead of pinned full 40-character SHA commit hashes, making the action vulnerable to supply-chain attacks if those tags are moved:
- `google-github-actions/setup-gcloud@v2` (line 68)
- `google-github-actions/upload-cloud-storage@v2` (line 71)
Note: `actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea` (line 113) is correctly pinned.

Locations:

- `action.yml:68`
- `action.yml:71`

### script-injection (severity: high)

Rule (b): Multiple shell variable expansions of untrusted workflow-controllable data are unquoted inside `run:` blocks, allowing shell metacharacter injection.

In the `stage-files` step (line 58–65):
- `mkdir -p ${dir};` — `dir` holds `inputs`-derived value, unquoted (line 61)
- `for file in ${allowlist};` — `allowlist` holds `inputs.allowlist`, unquoted (line 62); word-splitting is intentional but the value is attacker-controlled
- `cp ${file} ${dir}/${f2};` — unquoted path components (line 64)

In the `vertex-execution` step (line 91–109):
- `for file in ${notebooks};` — `notebooks` holds `inputs.allowlist`, unquoted (line 94)
- `--region=${region}` — `region` holds `inputs.region`, unquoted (line 101)
- `--labels=commit_sha=${commit_sha}` — unquoted (line 103)
- `container-image-uri="${container}"` — `container` holds `inputs.vertex_container_name`, partially quoted but embedded in a comma-separated string without full quoting (line 104)
- `--kernel-name="${kernel}"` — `kernel` holds `inputs.kernel_name` (line 105)

All these env vars are sourced from `inputs.*` (attacker-controlled) and must be double-quoted.

Locations:

- `action.yml:61`
- `action.yml:62`
- `action.yml:94`
- `action.yml:101`
- `action.yml:103`

### github-env-injection (severity: high)

The `stage-files` step writes a value derived from untrusted input to `$GITHUB_OUTPUT` without sanitization. Specifically, `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` (line 65) uses `${dir}`, which is set from `inputs`-derived data (`./${{ github.sha }}`). An attacker could craft a value containing newlines to inject arbitrary key=value pairs into `$GITHUB_OUTPUT`. The required sanitization step (`printf '%s' ... | tr -d '\n\r'`) is absent before the write.

Similarly, the `vertex-execution` step writes `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` (line 109); while `jobs.json` is computed, its content is derived from Vertex AI job output that was constructed using attacker-controlled inputs (`${commit_sha}`, `${notebooks}`, etc.) without sanitization before the write.

Locations:

- `action.yml:65`
- `action.yml:109`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/action.yml:

1. unpinned-uses: Pinned google-github-actions/setup-gcloud@v2 to @e427ad8a34f8676edf47cf7d7925499adf3eb74f and google-github-actions/upload-cloud-storage@v2 to @c0f6160ff80057923ff50e5e567695cea181ec23, preserving the original tag in comments.

2. script-injection: Added double-quotes around all unquoted shell variable expansions: ${dir}, ${file}, ${region}, ${commit_sha}, and ${container}. The for-loop iterators ${allowlist} and ${notebooks} remain intentionally unquoted since word-splitting is required for iteration.

3. github-env-injection: Sanitized both GITHUB_OUTPUT writes by capturing output into intermediate variables, stripping newlines with tr -d '\n\r', and using printf instead of echo to write the sanitized values. This prevents newline injection attacks that could inject arbitrary key=value pairs into GITHUB_OUTPUT.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in action.yml:
1. 'stage-files' step (line 62): Replaced unquoted `for file in ${allowlist}` with xargs-based safe tokenization into a bash array `files=()`, then iterated with `for file in "${files[@]}"`.
2. 'vertex-execution' step (line 98): Replaced unquoted `for file in ${notebooks}` with xargs-based safe tokenization into a bash array `nb_files=()`, then iterated with `for file in "${nb_files[@]}"`.

Both fixes use the guard pattern (`if [ -n "${VAR}" ]`) to prevent xargs from emitting an empty token on empty input, and use NUL-delimited output (`printf '%s\0'`) with `read -d ''` to correctly handle file paths with spaces or special characters. The `inputs.allowlist` value is a list input so xargs tokenization (rather than single-value quoting) is the correct approach.

