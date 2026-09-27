<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.3** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation: Multiple unquoted shell variable expansions of attacker-controlled env vars in run: blocks.

In the 'stage-files' step (line ~57), `${dir}` (from `${{ github.sha }}`) and `${allowlist}` (from `${{ inputs.allowlist }}`) are expanded unquoted:
  - `mkdir -p ${dir};`
  - `for file in ${allowlist};`
  - `cp ${file} ${dir}/${f2};`
An attacker-controlled `allowlist` value containing shell metacharacters (`;`, `|`, `&`, `$(...)`) can cause command injection.

In the 'vertex-execution' step (line ~80), the following env vars sourced from inputs are used unquoted:
  - `for file in ${notebooks};`
  - `--region=${region}`
  - `--labels=commit_sha=${commit_sha}`
  - `echo $output | jq -c > training.json`
All of `${notebooks}`, `${region}`, `${commit_sha}`, `${machine_type}`, `${container}`, and `${kernel}` are derived from `inputs.*` and must be double-quoted.

Locations:

- `action.yml:57`
- `action.yml:80`

### github-env-injection (severity: high)

The 'stage-files' step writes to $GITHUB_OUTPUT using a value derived from an unquoted, unsanitized env var:
  `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`
Here `${dir}` is set from `${{ github.sha }}` without sanitization (no `printf '%s' ... | tr -d '\n\r'` applied before the write). While `github.sha` is typically safe, the pattern is non-compliant and `${allowlist}` (from `${{ inputs.allowlist }}`) is also used unquoted in the same step, making the overall output tainted.

The 'vertex-execution' step writes to $GITHUB_OUTPUT:
  `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT`
The jobs.json content is built from a loop over `${notebooks}` (from `${{ inputs.allowlist }}`), an attacker-controlled value, without sanitization before the write.

Locations:

- `action.yml:70`
- `action.yml:110`

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags rather than full 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved:
  - `uses: 'google-github-actions/setup-gcloud@v2'` (marked `# ratchet:exclude` but still unpinned)
  - `uses: 'google-github-actions/upload-cloud-storage@v2'` (marked `# ratchet:exclude` but still unpinned)
These should be pinned to immutable commit SHAs, e.g. `google-github-actions/setup-gcloud@<40-char-sha> # v2`.

Locations:

- `action.yml:75`
- `action.yml:78`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses

**Notes:**

Fixed all three findings in action.yml:

1. **unpinned-uses**: Pinned `google-github-actions/setup-gcloud@v2` to `@e427ad8a34f8676edf47cf7d7925499adf3eb74f # v2` and `google-github-actions/upload-cloud-storage@v2` to `@c0f6160ff80057923ff50e5e567695cea181ec23 # v2`.

2. **script-injection**: 
   - In `stage-files`: Quoted `${dir}` as `"${dir}"`, tokenized the `allowlist` list input via xargs into a bash array (`files=()`), and iterated with `"${files[@]}"`; all file references are now double-quoted.
   - In `vertex-execution`: Tokenized the `notebooks` list input via xargs into a bash array (`notebooks=()`), iterated with `"${notebooks[@]}"`; all single-value inputs (`region`, `commit_sha`, `machine_type`, `container`, `kernel`) are now double-quoted. Also quoted `$output` as `"$output"`.
   - Renamed env vars to use `INPUT_` prefix to avoid naming conflicts.

3. **github-env-injection**: Both GITHUB_OUTPUT writes now sanitize values with `tr -d '\n\r'` before writing:
   - `stage-files`: `safe_notebooks=$(ls "${dir}" | xargs | tr -d '\n\r')` then `echo "notebooks=${safe_notebooks}" >> "$GITHUB_OUTPUT"`
   - `vertex-execution`: `safe_jobs=$(cat jobs.json | tr -d '\n\r')` then `echo "name=training_jobs=${safe_jobs}" >> "$GITHUB_OUTPUT"`

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed two github-env-injection findings in action.yml:

1. **stage-files step (lines 75-76)**: Replaced `safe_notebooks=$(ls "${dir}" | xargs | tr -d '\n\r')` with a two-step approach: first capture the raw value with `raw_notebooks=$(ls "${dir}" | xargs)`, then sanitize using the required pattern `safe_notebooks=$(printf '%s' "${raw_notebooks}" | tr -d '\n\r')`.

2. **vertex-execution step (lines 132-133)**: Replaced `safe_jobs=$(cat jobs.json | tr -d '\n\r')` with a two-step approach: first capture the raw value with `raw_jobs=$(cat jobs.json)`, then sanitize using the required pattern `safe_jobs=$(printf '%s' "${raw_jobs}" | tr -d '\n\r')`.

Both fixes now use `printf '%s'` to safely pass the value to `tr`, preventing potential injection via values that could bypass the previous `xargs`/`cat` piping approach.

