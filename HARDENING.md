<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.3** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags instead of full 40-character commit SHAs, making the action vulnerable to supply-chain attacks if those tags are moved:
- `google-github-actions/setup-gcloud@v2` (line 68)
- `google-github-actions/upload-cloud-storage@v2` (line 71)
These should be pinned to specific commit SHAs (e.g. `google-github-actions/setup-gcloud@<40-hex-sha> # v2`).

Locations:

- `action.yml:68`
- `action.yml:71`

### script-injection (severity: high)

Rule (b) violation — unquoted shell variable expansions of untrusted/workflow-controlled data in two `run:` blocks:

**stage-files step (lines 60–65):** `${dir}` (sourced from `inputs.allowlist` and `github.sha`) and `${allowlist}` (sourced from `inputs.allowlist`) are expanded without double-quotes throughout the loop body:
  - `mkdir -p ${dir};`
  - `for file in ${allowlist};`
  - `f2=$(echo ${file}|tr '/' '_');`
  - `cp ${file} ${dir}/${f2};`

Unquoted expansions allow shell metacharacters (`;`, `|`, `&`, `$(...)`, glob chars, whitespace) in attacker-controlled input values to be interpreted by the shell.

**vertex-execution step (lines 94–103):** Multiple env vars sourced from `inputs.*` are expanded without double-quotes:
  - `for file in ${notebooks};` — `notebooks` from `inputs.allowlist`
  - `--region=${region}` — `region` from `inputs.region`
  - `--labels=commit_sha=${commit_sha}` — `commit_sha` from `github.sha`
  - `--kernel-name="${kernel}"` — `kernel` from `inputs.kernel_name` (quoted, but others in the same command are not)

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:94`
- `action.yml:98`

### github-env-injection (severity: high)

Two `run:` steps write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`):

1. **stage-files step (line 66):** `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` — the directory listing is derived from `${dir}` which is set from `inputs.allowlist` and `github.sha`. No newline sanitization is applied before the write.

2. **vertex-execution step (line 107):** `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` — the content of `jobs.json` is built from unsanitized input-controlled values (`notebooks`/`inputs.allowlist`, `commit_sha`/`github.sha`, etc.). An attacker-controlled notebook filename containing a newline could inject arbitrary key=value pairs into `$GITHUB_OUTPUT`.

Locations:

- `action.yml:66`
- `action.yml:107`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in action.yml:
1. unpinned-uses: Pinned google-github-actions/setup-gcloud@v2 to SHA e427ad8a34f8676edf47cf7d7925499adf3eb74f and google-github-actions/upload-cloud-storage@v2 to SHA c0f6160ff80057923ff50e5e567695cea181ec23.
2. script-injection: In stage-files, tokenized the comma-separated allowlist input via xargs (replacing commas with spaces first) into a bash array and double-quoted all variable expansions. In vertex-execution, similarly tokenized notebooks into nb_files array and double-quoted all gcloud flag expansions.
3. github-env-injection: Both GITHUB_OUTPUT writes now capture the value into a raw variable, sanitize with `printf '%s' "$raw" | tr -d '\n\r'`, and write the sanitized value. Also quoted $GITHUB_OUTPUT references.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed all four publish-gate issues in action.yml: (1) Line 64: removed comma substitution from allowlist xargs pipeline (${allowlist//,/ } → ${allowlist}) to split on whitespace only, matching upstream; (2) Line 71: added '|| true' to ls capture assignment to preserve upstream behavior where non-zero exit was hidden; (3) Line 106: removed comma substitution from notebooks xargs pipeline (${notebooks//,/ } → ${notebooks}) to split on whitespace only, matching upstream; (4) Line 128: added '|| true' to cat capture assignment to preserve upstream behavior where non-zero exit was hidden.

