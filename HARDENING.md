<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags (`@v2`) rather than immutable full-length commit SHAs. This means a supply-chain compromise of those actions could silently inject malicious code into any workflow that uses this action. Failing references:
- `google-github-actions/setup-gcloud@v2` (line 71) — marked `# ratchet:exclude` but still unpinned
- `google-github-actions/upload-cloud-storage@v2` (line 74) — marked `# ratchet:exclude` but still unpinned

These should be pinned to their full 40-character commit SHA, e.g. `google-github-actions/setup-gcloud@<sha> # v2`.

Locations:

- `action.yml:71`
- `action.yml:74`

### script-injection (severity: high)

Rule (b) violation — multiple unquoted shell variable expansions of env vars that hold workflow-controllable (untrusted) input values. Unquoted expansions allow shell metacharacter injection (`;`, `|`, `&`, `$(...)`, glob chars, whitespace word-splitting).

**`stage-files` step (lines 61–67):**
- `mkdir -p ${dir};` — `$dir` holds `'./${{ github.sha }}'`, unquoted (line 61)
- `for file in ${allowlist};` — `$allowlist` holds `${{ inputs.allowlist }}`, unquoted (line 62); word-splits on whitespace/globs
- `cp ${file} ${dir}/${f2};` — `$file` and `$dir` are unquoted (line 65)

**`vertex-execution` step (lines 97–110):**
- `for file in ${notebooks};` — `$notebooks` holds `${{ inputs.allowlist }}`, unquoted (line 97)
- `--region=${region}` — `$region` holds `${{ inputs.region }}`, unquoted (line 106)
- `--labels=commit_sha=${commit_sha}` — `$commit_sha` holds `${{ github.sha }}`, unquoted (line 108)
- `--worker-pool-spec=machine-type="${machine_type}",…,container-image-uri="${container}"` — `$machine_type` and `$container` hold user inputs, unquoted outside the quoted substrings (line 109)

All these variables should be double-quoted: `"${dir}"`, `"${allowlist}"`, `"${notebooks}"`, `"${region}"`, etc.

Locations:

- `action.yml:61`
- `action.yml:62`
- `action.yml:65`
- `action.yml:97`
- `action.yml:106`
- `action.yml:108`
- `action.yml:109`

### github-env-injection (severity: high)

Two `run:` steps write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). An attacker-controlled newline in these values can inject arbitrary key=value pairs into the output environment, potentially hijacking subsequent steps.

**`stage-files` step (line 67):**
```
echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT
```
`${dir}` is set from `'./${{ github.sha }}'` (env var, no sanitization). The output of `ls ${dir}` is written directly to `$GITHUB_OUTPUT`.

**`vertex-execution` step (line 117):**
```
echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT
```
The `jobs.json` file is populated using data derived from `${notebooks}` (which holds `${{ inputs.allowlist }}`), `${commit_sha}` (`${{ github.sha }}`), and other user-controlled inputs — none of which are sanitized before being written to `$GITHUB_OUTPUT`.

Fix: apply `safe=$(printf '%s' "$VALUE" | tr -d '\n\r')` before each write.

Locations:

- `action.yml:67`
- `action.yml:117`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in action.yml:

1. **unpinned-uses**: Pinned `google-github-actions/setup-gcloud@v2` to SHA `e427ad8a34f8676edf47cf7d7925499adf3eb74f` and `google-github-actions/upload-cloud-storage@v2` to SHA `c0f6160ff80057923ff50e5e567695cea181ec23`, preserving `# v2` comments.

2. **script-injection**: Quoted all unquoted variable expansions (`${dir}`, `${file}`, etc.). For the `allowlist`/`notebooks` list inputs (which upstream iterated with `for file in ${allowlist}` — whitespace splitting), replaced with the xargs tokenization idiom into bash arrays with the required empty-value guard, then iterated with `"${files[@]}"`.

3. **github-env-injection**: In `stage-files`, captured `ls "${dir}" | xargs` into `safe_notebooks` with `tr -d '\n\r'` before writing to `$GITHUB_OUTPUT`. In `vertex-execution`, captured `cat jobs.json` into `raw_jobs`, sanitized into `safe_jobs` with `tr -d '\n\r'`, then wrote to `$GITHUB_OUTPUT`. Also quoted `$GITHUB_OUTPUT` references.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed two bare assignments that would abort the step under errexit:
1. action.yml line 71: Added `|| true` to `safe_notebooks=$(ls "${dir}" | xargs | tr -d '\n\r')` to match upstream behavior where ls was inside echo/printf.
2. action.yml line 127: Added `|| true` to `raw_jobs="$(cat jobs.json)"` to match upstream behavior where cat was inside echo/printf.
Both changes preserve the captured output while preventing errexit from aborting the step on non-zero exit.

