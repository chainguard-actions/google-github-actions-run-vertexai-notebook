<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.2** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags instead of immutable 40-character commit SHAs, making the action vulnerable to supply-chain attacks if the upstream tag is moved or compromised:
- `google-github-actions/setup-gcloud@v2` (line 72)
- `google-github-actions/upload-cloud-storage@v2` (line 75)

The third reference (`actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea`) is correctly pinned to a SHA.

Locations:

- `action.yml:72`
- `action.yml:75`

### script-injection (severity: high)

Sub-rule (b): Multiple `run:` blocks expand env vars that hold untrusted workflow-controllable values without double-quoting them, allowing shell metacharacter injection.

**`stage-files` step (lines ~62–68):** `${dir}` (sourced from `${{ github.sha }}`), `${allowlist}` (sourced from `${{ inputs.allowlist }}`), and `${file}` are all used unquoted:
  - `mkdir -p ${dir};`
  - `for file in ${allowlist};`
  - `cp ${file} ${dir}/${f2};`
  - `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`

**`vertex-execution` step (lines ~95–111):** `${notebooks}` (from `inputs.allowlist`), `${region}` (from `inputs.region`), `${machine_type}` (from `inputs.vertex_machine_type`), `${container}` (from `inputs.vertex_container_name`), and `${kernel}` (from `inputs.kernel_name`) are all used unquoted in shell commands and gcloud arguments:
  - `for file in ${notebooks};`
  - `--region=${region}`
  - `--labels=commit_sha=${commit_sha}`
  - `machine-type="${machine_type}",...,container-image-uri="${container}"`
  - `--kernel-name="${kernel}"`

An attacker controlling any of these inputs can inject shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) to execute arbitrary commands.

Locations:

- `action.yml:62`
- `action.yml:95`

### github-env-injection (severity: high)

Two `run:` blocks write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

**`stage-files` step:** `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` — `${dir}` is derived from `${{ github.sha }}` (untrusted). A newline embedded in the value could inject additional key=value pairs into GITHUB_OUTPUT.

**`vertex-execution` step:** `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` — `jobs.json` is populated with data derived from untrusted `inputs.*` values (allowlist, gcs buckets, etc.) flowing through the gcloud command output. Unsanitized newlines in this content could inject additional entries into GITHUB_OUTPUT.

Locations:

- `action.yml:68`
- `action.yml:111`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/action.yml:

1. **unpinned-uses**: Pinned `google-github-actions/setup-gcloud@v2` to SHA `e427ad8a34f8676edf47cf7d7925499adf3eb74f` and `google-github-actions/upload-cloud-storage@v2` to SHA `c0f6160ff80057923ff50e5e567695cea181ec23`, preserving the original tag in comments.

2. **script-injection**: Double-quoted all unquoted variable expansions in both `stage-files` and `vertex-execution` steps: `${dir}`, `${file}`, `${region}`, `${commit_sha}`, `${machine_type}`, `${container}`, `${kernel}`, `${source_file}`, `${output_file}`, and `$output`. The `${allowlist}` and `${notebooks}` variables in `for` loop headers are intentionally left unquoted as they are space-separated lists requiring word-splitting.

3. **github-env-injection**: Sanitized both GITHUB_OUTPUT writes by stripping newlines with `tr -d '\n\r'` before writing: `safe_notebooks=$(ls "${dir}" | xargs | tr -d '\n\r')` in `stage-files`, and `safe_jobs=$(cat jobs.json | tr -d '\n\r')` in `vertex-execution`.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in action.yml:

1. `stage-files` step (line 59): Replaced `for file in ${allowlist};` with a safe `while IFS= read -r -d ',' file || [ -n "$file" ]; do ... done < <(printf '%s,' "${allowlist}")` loop. This properly splits on commas (the actual delimiter) without allowing shell metacharacter injection from the unquoted expansion.

2. `vertex-execution` step (line 90): Replaced `for file in ${notebooks};` with the same safe while-read loop pattern: `while IFS= read -r -d ',' file || [ -n "$file" ]; do ... done < <(printf '%s,' "${notebooks}");`.

Both fixes: use `IFS= read -r -d ','` to split on commas, strip leading/trailing spaces from each token, skip empty tokens, use `printf '%s'` instead of `echo` for safe handling, and use process substitution to feed the comma-terminated list. The `|| [ -n "$file" ]` handles the last element which may lack a trailing comma.

