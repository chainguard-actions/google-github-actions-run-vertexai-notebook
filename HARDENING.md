<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.1** was hardened automatically. 9 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation in the 'stage-files' step: env vars sourced from untrusted inputs are expanded unquoted in the run: shell script. Specifically, `${dir}` (from `inputs`/`github.sha`) and `${allowlist}` (from `inputs.allowlist`) are used unquoted in `mkdir -p ${dir}`, `for file in ${allowlist}`, `cp ${file} ${dir}/${f2}`, and `ls ${dir}`. An attacker-controlled value containing shell metacharacters (`;`, `|`, `&`, glob chars, whitespace) can break out of the intended command context.

Locations:

- `action.yml:55`

### script-injection (severity: high)

Rule (b) violation in the 'vertex-execution' step: multiple env vars sourced from untrusted inputs are expanded unquoted in the run: shell script. Offending expansions include `${notebooks}` (from `inputs.allowlist`), `${region}` (from `inputs.region`), `${commit_sha}` (from `github.sha`), `${machine_type}` (from `inputs.vertex_machine_type`), `${container}` (from `inputs.vertex_container_name`), and `${kernel}` (from `inputs.kernel_name`). These appear in loop headers, gcloud flag values, and label assignments without double-quoting, allowing shell metacharacter injection.

Locations:

- `action.yml:80`

### github-env-injection (severity: high)

The 'stage-files' step writes to $GITHUB_OUTPUT using `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`. The value of `${dir}` is derived from `github.sha` (an untrusted input) and is not sanitized with `printf '%s' ... | tr -d '\n\r'` before the write. A newline embedded in the value could inject additional key=value pairs into GITHUB_OUTPUT.

Locations:

- `action.yml:63`

### github-env-injection (severity: high)

The 'vertex-execution' step writes to $GITHUB_OUTPUT using `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT`. The content of jobs.json is built from user-controlled inputs (notebook names, commit SHA, bucket paths) without sanitization via `printf '%s' ... | tr -d '\n\r'` before the write. A newline in any of those values could inject additional entries into GITHUB_OUTPUT.

Locations:

- `action.yml:101`

### unpinned-uses (severity: high)

Two composite action steps in action.yml reference external actions by mutable version tags rather than full 40-character SHA digests, making them vulnerable to supply-chain attacks: `google-github-actions/setup-gcloud@v2` (marked `# ratchet:exclude`) and `google-github-actions/upload-cloud-storage@v2` (marked `# ratchet:exclude`).

Locations:

- `action.yml:68`
- `action.yml:71`

### unpinned-uses (severity: high)

The draft-release workflow references an external reusable workflow by a mutable tag `@v3` instead of a full SHA digest: `google-github-actions/.github/.github/workflows/draft-release.yml@v3` (marked `# ratchet:exclude`).

Locations:

- `.github/workflows/draft-release.yml:17`

### unpinned-uses (severity: high)

The release workflow references an external reusable workflow by a mutable tag `@v3` instead of a full SHA digest: `google-github-actions/.github/.github/workflows/release.yml@v3` (marked `# ratchet:exclude`).

Locations:

- `.github/workflows/release.yml:11`

### missing-permissions (severity: medium)

The draft-release.yml workflow has no top-level `permissions:` key and its single job (`draft-release`) also has no job-level `permissions:` key. Without explicit permissions, the workflow runs with the default (potentially broad) token permissions.

Locations:

- `.github/workflows/draft-release.yml:1`

### missing-permissions (severity: medium)

The release.yml workflow has no top-level `permissions:` key and its single job (`release`) also has no job-level `permissions:` key. Without explicit permissions, the workflow runs with the default (potentially broad) token permissions.

Locations:

- `.github/workflows/release.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all 9 findings across 3 files:

**action.yml**:
- script-injection (stage-files): Moved ${{ inputs.allowlist }} and ${{ github.sha }} into env vars (INPUT_ALLOWLIST, INPUT_DIR). All shell expansions are now double-quoted. The allowlist list input is tokenized into a bash array using xargs (quote-aware) to correctly handle whitespace/quoted values.
- script-injection (vertex-execution): Moved all ${{ ... }} expressions into INPUT_-prefixed env vars. All shell variable expansions are now double-quoted.
- github-env-injection (stage-files): Sanitized ls output with tr -d '\n\r' before writing to $GITHUB_OUTPUT.
- github-env-injection (vertex-execution): Sanitized jobs.json content with tr -d '\n\r' before writing to $GITHUB_OUTPUT.
- unpinned-uses: Pinned setup-gcloud@v2 to @e427ad8a34f8676edf47cf7d7925499adf3eb74f and upload-cloud-storage@v2 to @c0f6160ff80057923ff50e5e567695cea181ec23.

**draft-release.yml**:
- unpinned-uses: Pinned draft-release.yml@v3 to @29c6d38eeb974133b4b66401985f7c70cf4a6681.
- missing-permissions: Added permissions: {} at top level.

**release.yml**:
- unpinned-uses: Pinned release.yml@v3 to @29c6d38eeb974133b4b66401985f7c70cf4a6681.
- missing-permissions: Added permissions: {} at top level.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in the 'vertex-execution' step of action.yml. The --worker-pool-spec and --args gcloud CLI arguments that embedded user-controlled env vars (INPUT_MACHINE_TYPE, INPUT_CONTAINER, INPUT_KERNEL) were unquoted at the outer shell level. Wrapped both composite arguments in outer double-quotes (e.g. "--worker-pool-spec=machine-type=${INPUT_MACHINE_TYPE},replica-count=1,container-image-uri=${INPUT_CONTAINER}") so each is treated as a single shell word, preventing word-splitting and shell metacharacter injection. Also consistently applied outer quoting to the --labels argument.

