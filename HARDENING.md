<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.0** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags rather than full 40-character commit SHAs, making the action vulnerable to supply-chain attacks if the upstream tag is moved or hijacked:
- `google-github-actions/setup-gcloud@v2` (line 72)
- `google-github-actions/upload-cloud-storage@v2` (line 75)
Note: `actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea` is correctly pinned.

Locations:

- `action.yml:72`
- `action.yml:75`

### script-injection (severity: high)

Rule (b) violation — env vars holding workflow-controllable input values are expanded unquoted inside `run:` shell commands in two steps.

**`stage-files` step (lines 63–69):** `${allowlist}` (from `inputs.allowlist`) and `${dir}` (from `github.sha`) are used unquoted:
- `mkdir -p ${dir};`
- `for file in ${allowlist};`
- `f2=$(echo ${file}|tr '/' '_');`
- `cp ${file} ${dir}/${f2};`
An attacker-controlled `allowlist` value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, whitespace, globs) will be parsed by the shell before quoting takes effect.

**`vertex-execution` step (lines 96–107):** `${notebooks}`, `${region}`, `${machine_type}`, `${container}`, `${kernel}`, and `${commit_sha}` (all sourced from `inputs.*`) are used unquoted:
- `for file in ${notebooks};`
- `--region=${region}`
- `--labels=commit_sha=${commit_sha}`
- `machine-type="${machine_type}"` (partially quoted but the outer assignment is unquoted)
- `container-image-uri="${container}"`
- `--kernel-name="${kernel}"`
All of these must be double-quoted: `"${region}"`, `"${notebooks}"`, etc.

Locations:

- `action.yml:63`
- `action.yml:64`
- `action.yml:96`

### github-env-injection (severity: high)

Two `run:` blocks write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

**`stage-files` step (line 69):** `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` — `${dir}` is derived from `${{ github.sha }}` (an untrusted context value). The output of `ls` on an attacker-influenced directory name is written directly to GITHUB_OUTPUT with no newline stripping, enabling header injection.

**`vertex-execution` step (line 108):** `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` — `jobs.json` is populated using values from `inputs.allowlist`, `inputs.gcs_output_bucket`, `inputs.gcs_source_bucket`, and other inputs. The content is written to GITHUB_OUTPUT without sanitization, allowing a crafted input to inject additional key=value pairs into the output file.

Locations:

- `action.yml:69`
- `action.yml:108`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in action.yml:

1. **unpinned-uses**: Pinned `google-github-actions/setup-gcloud@v2` to SHA `e427ad8a34f8676edf47cf7d7925499adf3eb74f` and `google-github-actions/upload-cloud-storage@v2` to SHA `c0f6160ff80057923ff50e5e567695cea181ec23`, preserving the original tags as comments.

2. **script-injection**: Double-quoted all unquoted variable expansions in both shell steps. In `stage-files`: `"${dir}"`, `"${file}"` in mkdir, echo, and cp. In `vertex-execution`: `"${file}"`, `"${region}"`, `"${commit_sha}"`, `"${machine_type}"`, `"${container}"`, `"${kernel}"`. The `${allowlist}` and `${notebooks}` loop variables are intentionally left unquoted in `for` statements since they are space-separated lists requiring word-splitting.

3. **github-env-injection**: Both GITHUB_OUTPUT writes now sanitize values with `tr -d '\n\r'` before writing: `safe_notebooks=$(ls "${dir}" | xargs | tr -d '\n\r')` in stage-files, and `safe_jobs=$(cat jobs.json | tr -d '\n\r')` in vertex-execution. Also quoted `"$GITHUB_OUTPUT"` for correctness.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in action.yml:
1. stage-files step (line 62): Replaced unquoted `for file in ${allowlist};` with `IFS=', ' read -ra files <<< "${allowlist}"` and `for file in "${files[@]}";` to safely split and iterate the comma-separated allowlist input.
2. vertex-execution step (line 98): Replaced unquoted `for file in ${notebooks};` with `IFS=', ' read -ra notebook_files <<< "${notebooks}"` and `for file in "${notebook_files[@]}";` to safely split and iterate the notebooks input.
Both fixes use bash's read -ra to tokenize the input into an array with IFS set to comma and space (matching the documented 'Comma separated list' format), then iterate with properly double-quoted array expansion to prevent shell metacharacter injection.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed script-injection vulnerabilities in the vertex-execution step of action.yml:
1. Wrapped '--worker-pool-spec=machine-type=${machine_type},replica-count=1,container-image-uri=${container}' in outer double-quotes so the entire compound argument is a single shell word, preventing metacharacter injection from vertex_machine_type and vertex_container_name inputs.
2. Wrapped '--args=nbexecutor,--input-notebook=${source_file},--output-notebook=${output_file},--kernel-name=${kernel}' in outer double-quotes for the same reason, protecting gcs_source_bucket, gcs_output_bucket, and kernel_name inputs.
3. Fixed unquoted 'echo $output' to 'echo "$output"'.
4. Also fixed '--labels=commit_sha="${commit_sha}"' to '--labels="commit_sha=${commit_sha}"' for consistent outer-quoting style.

