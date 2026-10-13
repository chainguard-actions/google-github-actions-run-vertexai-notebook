<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.3** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags rather than immutable 40-character commit SHAs, making the action vulnerable to supply-chain attacks if those tags are moved:
- `google-github-actions/setup-gcloud@v2` (line 68)
- `google-github-actions/upload-cloud-storage@v2` (line 71)
The `# ratchet:exclude` comments are ratchet tool annotations and do not constitute a security control. The third step (`actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea`) is correctly pinned.

Locations:

- `action.yml:68`
- `action.yml:71`

### script-injection (severity: high)

Rule (b) violation: Multiple `run:` blocks expand env vars sourced from `inputs.*` and `github.*` without double-quoting, allowing shell metacharacter injection.

In the `stage-files` step (run block ~line 60):
- `mkdir -p ${dir};` — `dir` is set from `'./${{ github.sha }}'`
- `for file in ${allowlist};` — `allowlist` is set from `'${{ inputs.allowlist }}'`
- `cp ${file} ${dir}/${f2};` — both variables unquoted
- `echo "notebooks=$(ls ${dir} | xargs)"` — `dir` unquoted inside command substitution

In the `vertex-execution` step (run block ~line 93):
- `for file in ${notebooks};` — `notebooks` from `inputs.allowlist`
- `--region=${region}` — `region` from `inputs.region`
- `--display-name="${job_name}"` — derived from `commit_sha` and `file`
- `--labels=commit_sha=${commit_sha}` — unquoted
- `machine-type="${machine_type}"` — from `inputs.vertex_machine_type`
- `container-image-uri="${container}"` — from `inputs.vertex_container_name`
- `--kernel-name="${kernel}"` — from `inputs.kernel_name`

An attacker-controlled input containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) can break out of the intended command and execute arbitrary code.

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:93`
- `action.yml:98`

### github-env-injection (severity: high)

Two `run:` blocks write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

1. `stage-files` step (~line 65): `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` — `${dir}` is derived from `github.sha` (set via env: `dir: './${{ github.sha }}'`). A newline embedded in the value could inject additional key=value pairs into GITHUB_OUTPUT.

2. `vertex-execution` step (~line 107): `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` — the jobs.json content is built from `inputs.allowlist` and other user-controlled inputs. Without sanitization, a crafted input could inject additional entries into GITHUB_OUTPUT.

Neither write is preceded by the required sanitization pipeline.

Locations:

- `action.yml:65`
- `action.yml:107`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/action.yml:

1. unpinned-uses: Pinned `google-github-actions/setup-gcloud@v2` to `@e427ad8a34f8676edf47cf7d7925499adf3eb74f # v2` and `google-github-actions/upload-cloud-storage@v2` to `@c0f6160ff80057923ff50e5e567695cea181ec23 # v2`.

2. script-injection: In `stage-files`, the `allowlist` env var (a whitespace-separated list from dorny/paths-filter) is now tokenized into a bash array using the xargs/NUL-delimited read loop pattern, and all variable expansions are double-quoted. In `vertex-execution`, the `notebooks` env var is similarly tokenized, and all gcloud flag values (`--region`, `--display-name`, `--labels`, `--worker-pool-spec`, `--args`) are double-quoted. The `dir` variable is used as `./${dir}` with surrounding quotes.

3. github-env-injection: Both GITHUB_OUTPUT writes now capture the value into `raw`, then sanitize with `safe=$(printf '%s' "$raw" | tr -d '\n\r')` before writing `${safe}` to GITHUB_OUTPUT. The `jq -c` already compacts JSON to one line, so this is a safety measure against any embedded newlines.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed two bare assignments that would abort the step under errexit: (1) action.yml line 69: added `|| true` to `raw=$(ls "./${dir}" | xargs)` so a non-zero exit from ls/xargs doesn't abort the step, matching upstream behavior where the exit status was hidden inside echo's argument; (2) action.yml line 124: added `|| true` to `raw=$(cat jobs.json)` for the same reason.

