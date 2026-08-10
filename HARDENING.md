<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.3** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

action.yml references two external actions using mutable version tags instead of full 40-character SHA commit hashes, making the action vulnerable to supply-chain attacks if those tags are moved. Failing references: `google-github-actions/setup-gcloud@v2` and `google-github-actions/upload-cloud-storage@v2`. (Note: `actions/github-script` is correctly pinned to a SHA.) Additionally, .github/workflows/draft-release.yml and .github/workflows/release.yml use `@v3` tag references for reusable workflows.

Locations:

- `action.yml:70`
- `action.yml:73`
- `.github/workflows/draft-release.yml:16`
- `.github/workflows/release.yml:10`

### script-injection (severity: high)

Rule (b) violation — unquoted shell variable expansions of env vars that hold workflow-controllable (untrusted) data. In the `stage-files` step, `${dir}` (from `github.sha`) and `${allowlist}` (from `inputs.allowlist`) are expanded unquoted: `mkdir -p ${dir}`, `for file in ${allowlist}`, `cp ${file} ${dir}/${f2}`, and `ls ${dir}`. An attacker-controlled value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, glob chars, whitespace) would be parsed by the shell before quoting takes effect, enabling command injection.

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:63`
- `action.yml:64`
- `action.yml:66`

### script-injection (severity: high)

Rule (b) violation — unquoted shell variable expansions of env vars that hold workflow-controllable (untrusted) data in the `vertex-execution` step. `${notebooks}` (from `inputs.allowlist`), `${region}` (from `inputs.region`), `${commit_sha}` (from `github.sha`), `${machine_type}` (from `inputs.vertex_machine_type`), `${container}` (from `inputs.vertex_container_name`), and `${kernel}` (from `inputs.kernel_name`) are all used unquoted in shell commands including `for file in ${notebooks}`, `--region=${region}`, `--labels=commit_sha=${commit_sha}`, and `--worker-pool-spec=machine-type="${machine_type}",...,container-image-uri="${container}"`. Shell metacharacters in any of these values would be interpreted by the shell.

Locations:

- `action.yml:96`
- `action.yml:105`
- `action.yml:107`
- `action.yml:108`

### github-env-injection (severity: high)

In the `stage-files` step, the value written to `$GITHUB_OUTPUT` is derived from `${dir}` (which is set from `github.sha` via the `env:` block) without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). The command `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` could allow newline injection into the output file if the directory name contains newlines, enabling an attacker to inject arbitrary key-value pairs into `$GITHUB_OUTPUT`.

Locations:

- `action.yml:66`

### github-env-injection (severity: high)

In the `vertex-execution` step, the value written to `$GITHUB_OUTPUT` is derived from `jobs.json`, which is built using data from `${commit_sha}` (from `github.sha`) and other user-controlled inputs, without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). The command `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` could allow newline injection into the output file, enabling injection of arbitrary key-value pairs into `$GITHUB_OUTPUT`.

Locations:

- `action.yml:116`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all 5 findings across action.yml and two workflow files:

1. unpinned-uses: Pinned setup-gcloud@v2 to SHA e427ad8a34f8676edf47cf7d7925499adf3eb74f, upload-cloud-storage@v2 to SHA c0f6160ff80057923ff50e5e567695cea181ec23, and both reusable workflow @v3 references to SHA 29c6d38eeb974133b4b66401985f7c70cf4a6681.

2. script-injection (stage-files): Quoted ${dir} in mkdir/cp/ls calls; tokenized ${allowlist} using xargs+NUL-delimited read loop into a bash array to safely iterate over whitespace-separated filenames.

3. script-injection (vertex-execution): Tokenized ${notebooks} using xargs+NUL-delimited read loop into a bash array; quoted all gcloud flag expansions (--region, --labels, --worker-pool-spec, --args) as double-quoted strings.

4. github-env-injection (stage-files): Captured ls output into safe_notebooks variable with tr -d '\n\r' sanitization before writing to $GITHUB_OUTPUT via printf.

5. github-env-injection (vertex-execution): Captured jobs.json content into safe_jobs variable with tr -d '\n\r' sanitization before writing to $GITHUB_OUTPUT via printf. Also corrected the output key name (removed erroneous 'name=' prefix from original).

