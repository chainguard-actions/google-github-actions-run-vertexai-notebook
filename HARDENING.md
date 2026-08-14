<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.0** was hardened automatically. 6 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation in the 'stage-files' step: env vars `${dir}` (sourced from `${{ github.sha }}`) and `${allowlist}` (sourced from `${{ inputs.allowlist }}`) are expanded unquoted in shell commands (`mkdir -p ${dir}`, `for file in ${allowlist}`, `cp ${file} ${dir}/${f2}`). An attacker-controlled allowlist value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) can break out of the intended command and execute arbitrary code.

Locations:

- `action.yml:53`

### script-injection (severity: high)

Rule (b) violation in the 'vertex-execution' step: multiple env vars sourced from user inputs are expanded unquoted in shell commands — `${notebooks}` (from `inputs.allowlist`), `${region}` (from `inputs.region`), `${machine_type}` (from `inputs.vertex_machine_type`), `${container}` (from `inputs.vertex_container_name`), and `${kernel}` (from `inputs.kernel_name`). For example: `for file in ${notebooks}`, `--region=${region}`, `container-image-uri="${container}"`. Unquoted expansion allows shell metacharacter injection.

Locations:

- `action.yml:83`

### github-env-injection (severity: high)

The 'stage-files' step writes `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` where `${dir}` is derived from `${{ github.sha }}` (an untrusted input). The value is not sanitized with `printf '%s' ... | tr -d '\n\r'` before the write. A newline embedded in the directory listing could inject additional key=value pairs into GITHUB_OUTPUT.

Locations:

- `action.yml:67`

### unpinned-uses (severity: high)

Two composite action steps in action.yml reference external actions by mutable version tags instead of full 40-character commit SHAs, making the action vulnerable to supply-chain attacks if those tags are moved: `google-github-actions/setup-gcloud@v2` and `google-github-actions/upload-cloud-storage@v2`.

Locations:

- `action.yml:71`
- `action.yml:74`

### unpinned-uses (severity: high)

Workflow files reference reusable workflows by mutable version tags (`@v1`) instead of full 40-character commit SHAs: `google-github-actions/.github/.github/workflows/draft-release.yml@v1` in draft-release.yml and `google-github-actions/.github/.github/workflows/release.yml@v1` in release.yml.

Locations:

- `.github/workflows/draft-release.yml:17`
- `.github/workflows/release.yml:12`

### missing-permissions (severity: medium)

Neither `draft-release.yml` nor `release.yml` define a top-level `permissions:` key, and their single jobs also lack a job-level `permissions:` block. Without explicit permissions, the GITHUB_TOKEN is granted its default (potentially broad) permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/draft-release.yml:1`
- `.github/workflows/release.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all 6 findings across action.yml and the two workflow files:

1. **script-injection (stage-files)**: Moved all ${{ }} expressions to env vars with INPUT_ prefix. Tokenized the allowlist input using xargs+printf NUL-delimited pipeline into a bash array to handle quoted arguments correctly. All variable expansions are now double-quoted.

2. **script-injection (vertex-execution)**: Same approach — all inputs moved to env vars, notebooks list tokenized via xargs into an array, all expansions double-quoted (--region="${INPUT_REGION}", container-image-uri="${INPUT_CONTAINER}", etc.).

3. **github-env-injection (stage-files)**: Sanitized the GITHUB_OUTPUT write with `tr -d '\n\r'` before writing: `safe_notebooks=$(ls "./${INPUT_DIR}" | xargs | tr -d '\n\r')` then `echo "notebooks=${safe_notebooks}" >> "$GITHUB_OUTPUT"`.

4. **unpinned-uses (action.yml)**: Pinned setup-gcloud@v2 → @e427ad8a34f8676edf47cf7d7925499adf3eb74f and upload-cloud-storage@v2 → @c0f6160ff80057923ff50e5e567695cea181ec23.

5. **unpinned-uses (workflows)**: Pinned both reusable workflow references (draft-release.yml@v1 and release.yml@v1) to @6900f1ed495961bca1d6c2e6cb679e7ce7e23a88.

6. **missing-permissions**: Added `permissions: {}` top-level block to both draft-release.yml and release.yml.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed two issues in the 'vertex-execution' step of action.yml: (1) Quoted `$output` in `echo "$output" | jq -c > training.json` to prevent shell metacharacter injection from unquoted variable expansion. (2) Sanitized the jobs.json content before writing to $GITHUB_OUTPUT by using `safe_jobs=$(printf '%s' "$(cat jobs.json)" | tr -d '\n\r')` and then `echo "training_jobs=${safe_jobs}" >> "$GITHUB_OUTPUT"` to strip newlines that could inject arbitrary key=value pairs.

### Iteration 3

**Fixes applied:** github-env-injection

**Notes:**

Fixed the github-env-injection finding in the `stage-files` step of action.yml. The original code computed `safe_notebooks` using `ls "./${INPUT_DIR}" | xargs | tr -d '\n\r'` inline in the pipeline, which did not match the required sanitization pattern. The fix splits the computation into two steps: (1) `raw_notebooks=$(ls "./${INPUT_DIR}" | xargs)` to capture the raw listing, and (2) `safe_notebooks=$(printf '%s' "$raw_notebooks" | tr -d '\n\r')` to sanitize using the required `printf '%s' "$VAR" | tr -d '\n\r'` form immediately before writing to $GITHUB_OUTPUT.

