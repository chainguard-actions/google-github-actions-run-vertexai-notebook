<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.1** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags instead of full 40-character commit SHAs, making the action vulnerable to supply-chain attacks if the upstream tag is moved or compromised. Failing references: `google-github-actions/setup-gcloud@v2` and `google-github-actions/upload-cloud-storage@v2`. Note: `actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea` is correctly pinned.

Locations:

- `action.yml:71`
- `action.yml:74`

### script-injection (severity: high)

Sub-rule (b): Multiple env vars holding workflow-controllable (untrusted) data are expanded unquoted inside `run:` shell commands, allowing shell metacharacter injection.

**`stage-files` step (lines ~61-67):** `${dir}` (from `inputs.*`/`github.sha`) and `${allowlist}` (from `inputs.allowlist`) are used unquoted:
  - `mkdir -p ${dir};`
  - `for file in ${allowlist};`
  - `cp ${file} ${dir}/${f2};`
  - `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`

**`vertex-execution` step (lines ~93-118):** `${notebooks}`, `${region}`, `${commit_sha}`, `${machine_type}`, `${container}` are all used unquoted:
  - `for file in ${notebooks};`
  - `--region=${region}`
  - `--display-name="${job_name}"` (job_name built from unquoted vars)
  - `--labels=commit_sha=${commit_sha}`
  - `machine-type="${machine_type}"` and `container-image-uri="${container}"`

All of these should be double-quoted (e.g. `"${dir}"`, `"${allowlist}"`, `"${region}"`) to prevent word-splitting and glob expansion of attacker-controlled values.

Locations:

- `action.yml:61`
- `action.yml:62`
- `action.yml:93`
- `action.yml:100`

### github-env-injection (severity: high)

Two `run:` steps write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

**`stage-files` step:** `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` — `${dir}` is set from `github.sha` (untrusted). A newline embedded in the SHA could inject additional key=value pairs into GITHUB_OUTPUT.

**`vertex-execution` step:** `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` — `jobs.json` is constructed from unsanitized user-controlled inputs (`inputs.allowlist`, `inputs.gcs_output_bucket`, etc.). Newlines in those values could corrupt the GITHUB_OUTPUT file.

Both writes must be preceded by sanitization, e.g.:
```bash
safe=$(printf '%s' "$(ls ${dir} | xargs)" | tr -d '\n\r')
echo "notebooks=$safe" >> "$GITHUB_OUTPUT"
```

Locations:

- `action.yml:67`
- `action.yml:118`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in action.yml: (1) Pinned google-github-actions/setup-gcloud@v2 to SHA e427ad8a34f8676edf47cf7d7925499adf3eb74f and google-github-actions/upload-cloud-storage@v2 to SHA c0f6160ff80057923ff50e5e567695cea181ec23. (2) Added double-quotes around all unquoted variable expansions in both run: steps (${dir}, ${file}, ${f2}, ${region}, ${commit_sha}, ${machine_type}, ${container}, ${source_file}, ${output_file}, ${kernel}, ${job_name}); the for-loop iterators over ${allowlist} and ${notebooks} are intentionally left unquoted to preserve upstream's whitespace-splitting behavior for the list input. (3) Sanitized both GITHUB_OUTPUT writes using `safe=$(printf '%s' "$(cmd)" | tr -d '\n\r')` before writing to prevent newline injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed three script-injection issues in action.yml:
1. 'stage-files' step: Replaced unquoted `for file in ${allowlist}` with a safe xargs-based tokenization into a bash array (`files`), guarded with `if [ -n "${allowlist}" ]` and using `xargs -r printf '%s\0'` with a null-delimited read loop. The array is then iterated with `"${files[@]}"`.
2. 'vertex-execution' step: Applied the same xargs-based tokenization for `${notebooks}` into `nb_files` array, iterated with `"${nb_files[@]}"`.
3. 'vertex-execution' step: Fixed unquoted `echo $output` to `echo "$output"` to prevent word splitting and glob expansion of gcloud output derived from attacker-controlled inputs.
All fixes preserve the upstream whitespace-splitting behavior while preventing shell metacharacter injection.

