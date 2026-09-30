<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.2** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags instead of full 40-character commit SHAs, making the action vulnerable to supply-chain attacks if those tags are moved:
- `google-github-actions/setup-gcloud@v2` (line 68)
- `google-github-actions/upload-cloud-storage@v2` (line 71)
Note: `actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea` is correctly pinned.

Locations:

- `action.yml:68`
- `action.yml:71`

### script-injection (severity: high)

Sub-rule (b): Multiple env vars holding workflow-controllable values (`inputs.*`, `github.*`) are expanded unquoted inside `run:` shell scripts, allowing shell metacharacter injection.

**`stage-files` step (lines 60–65):**
- `mkdir -p ${dir};` — `$dir` holds `'./${{ github.sha }}'`, unquoted
- `for file in ${allowlist};` — `$allowlist` holds `'${{ inputs.allowlist }}'`, unquoted; word-splitting on this value is intentional but the unquoted expansion still allows glob and metacharacter injection
- `cp ${file} ${dir}/${f2};` — both `${file}` and `${dir}` are unquoted

**`vertex-execution` step (lines 91–106):**
- `for file in ${notebooks};` — `$notebooks` holds `${{ inputs.allowlist }}`, unquoted
- `--region=${region}` — `$region` holds `${{ inputs.region }}`, unquoted
- `--display-name="${job_name}"` — `$job_name` is derived from `$commit_sha` and `$file`, both from inputs/github context
- `--labels=commit_sha=${commit_sha}` — `$commit_sha` unquoted
- `container-image-uri="${container}"` — `$container` holds `${{ inputs.vertex_container_name }}`
- `--kernel-name="${kernel}"` — `$kernel` holds `${{ inputs.kernel_name }}`

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:91`
- `action.yml:96`

### github-env-injection (severity: high)

Two `run:` blocks write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`):

1. **`stage-files` step (line 65):** `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` — `${dir}` is derived from `${{ github.sha }}` (env var, not sanitized). A newline embedded in the value could inject arbitrary key=value pairs into GITHUB_OUTPUT.

2. **`vertex-execution` step (line 107):** `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` — the jobs.json content is derived from gcloud output that incorporates `${{ inputs.* }}` and `${{ github.sha }}` values (via `$commit_sha`, `$source_location`, `$output_location`). No sanitization is applied before the write.

Locations:

- `action.yml:65`
- `action.yml:107`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/action.yml:
1. unpinned-uses: Pinned google-github-actions/setup-gcloud@v2 to SHA e427ad8a34f8676edf47cf7d7925499adf3eb74f and google-github-actions/upload-cloud-storage@v2 to SHA c0f6160ff80057923ff50e5e567695cea181ec23.
2. script-injection: Quoted all unquoted variable expansions in stage-files (${dir}, ${file}) and vertex-execution (${file}, ${region}, ${commit_sha}, $output) steps. The for-loop expansions of ${allowlist} and ${notebooks} remain unquoted as they are intentionally word-split file lists.
3. github-env-injection: Added sanitization with tr -d '\n\r' before writing to $GITHUB_OUTPUT in both stage-files (safe_notebooks) and vertex-execution (safe_jobs) steps, and quoted the $GITHUB_OUTPUT variable itself.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed both script injection findings in action.yml:
1. 'stage-files' step (line ~61): Added `id: 'stage-files'` to the step. Replaced unquoted `for file in ${allowlist}` with a safe xargs-based tokenization: `while IFS= read -r -d '' t; do files+=("$t"); done < <(printf '%s' "$allowlist" | xargs printf '%s\0')` followed by `for file in "${files[@]}"`.
2. 'vertex-execution' step (line ~97): Changed `notebooks` env var from raw `${{ inputs.allowlist }}` to the sanitized step output `${{ steps.stage-files.outputs.notebooks }}` (produced by `ls` of the staged directory, not user input). Replaced unquoted `for file in ${notebooks}` with the same safe xargs-based tokenization pattern using `nb_files` array.
Both fixes prevent shell metacharacter injection (`;`, `|`, `&`, `$(...)`, etc.) via the `allowlist` input while preserving the intended file iteration behavior.

