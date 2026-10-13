<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.1** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags instead of immutable 40-character commit SHAs, making them vulnerable to supply-chain attacks if the upstream tag is moved:
- `google-github-actions/setup-gcloud@v2` (line 69)
- `google-github-actions/upload-cloud-storage@v2` (line 72)

The third reference (`actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea`) is correctly pinned to a SHA.

Locations:

- `action.yml:69`
- `action.yml:72`

### script-injection (severity: high)

Sub-rule (b): Multiple env vars holding workflow-controllable (untrusted) data are expanded without double-quoting inside `run:` shell commands, allowing shell metacharacter injection.

In the `stage-files` step (env vars `allowlist` from `inputs.allowlist` and `dir` from `github.sha`):
- `mkdir -p ${dir};` — `${dir}` unquoted
- `for file in ${allowlist};` — `${allowlist}` unquoted; word-splitting is intentional but still allows glob/metachar injection
- `f2=$(echo ${file}|tr '/' '_');` — `${file}` unquoted
- `cp ${file} ${dir}/${f2};` — `${file}` and `${dir}` unquoted

In the `vertex-execution` step (env vars `notebooks` from `inputs.allowlist`, `region` from `inputs.region`, `commit_sha` from `github.sha`):
- `for file in ${notebooks};` — `${notebooks}` unquoted
- `--region=${region}` — `${region}` unquoted
- `--labels=commit_sha=${commit_sha}` — `${commit_sha}` unquoted

All of these should use double-quoted expansions (e.g. `"${region}"`, `"${commit_sha}"`) to prevent shell metacharacter interpretation.

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:63`
- `action.yml:64`
- `action.yml:85`
- `action.yml:88`
- `action.yml:89`

### github-env-injection (severity: high)

Two `run:` steps write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

1. `stage-files` step (line 66): `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` — `${dir}` is set from `github.sha` (untrusted). A newline embedded in the value could inject arbitrary key=value pairs into GITHUB_OUTPUT.

2. `vertex-execution` step (line 97): `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` — `jobs.json` is built from user-controlled inputs (`inputs.allowlist`, `inputs.gcs_output_bucket`, `inputs.gcs_source_bucket`, etc.). Content with embedded newlines could inject additional output variables.

Both writes must be preceded by sanitization, e.g.:
```bash
safe=$(printf '%s' "$value" | tr -d '\n\r')
echo "key=$safe" >> "$GITHUB_OUTPUT"
```

Locations:

- `action.yml:66`
- `action.yml:97`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/action.yml:

1. **unpinned-uses**: Pinned `google-github-actions/setup-gcloud@v2` to SHA `e427ad8a34f8676edf47cf7d7925499adf3eb74f` and `google-github-actions/upload-cloud-storage@v2` to SHA `c0f6160ff80057923ff50e5e567695cea181ec23`, preserving the original tags in comments.

2. **script-injection**: 
   - In `stage-files`: Added double-quotes around `${dir}`, `${file}`, `${f2}`. Replaced unquoted `for file in ${allowlist}` with xargs-based tokenization into a bash array (`files=()`), then iterated with `"${files[@]}"`.
   - In `vertex-execution`: Added double-quotes around `${region}`, `${commit_sha}`. Replaced unquoted `for file in ${notebooks}` with xargs-based tokenization into a bash array (`nb_files=()`), then iterated with `"${nb_files[@]}"`.

3. **github-env-injection**: 
   - In `stage-files`: Captured `ls` output into `raw`, sanitized with `printf '%s' "$raw" | tr -d '\n\r'` into `safe`, then wrote `notebooks=${safe}` to `$GITHUB_OUTPUT`.
   - In `vertex-execution`: Captured `jobs.json` content into `raw`, sanitized with `printf '%s' "$raw" | tr -d '\n\r'` into `safe`, then wrote `name=training_jobs=${safe}` to `$GITHUB_OUTPUT`.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed two bare assignments that would abort the step under errexit:
1. action.yml line 69: Added `|| true` to `raw="$(ls "${dir}" | xargs)"` so a non-zero exit from ls doesn't abort the step, matching upstream behavior.
2. action.yml line 124: Added `|| true` to `raw="$(cat jobs.json)"` so a non-zero exit from cat doesn't abort the step, matching upstream behavior.

