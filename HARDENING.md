<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **google-github-actions--run-vertexai-notebook/v1.1.2** was hardened automatically. 5 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags rather than full 40-character commit SHAs, making them vulnerable to supply-chain attacks. Specifically: `google-github-actions/setup-gcloud@v2` and `google-github-actions/upload-cloud-storage@v2`. (The `# ratchet:exclude` comments indicate intentional exclusion from pinning tooling, but they remain unpinned.)

Locations:

- `action.yml:69`
- `action.yml:72`

### script-injection (severity: high)

Sub-rule (b): The `stage-files` run block expands env vars sourced from `inputs.allowlist` and `github.sha` without double-quoting, allowing shell metacharacter injection. Offending lines include: `mkdir -p ${dir};`, `for file in ${allowlist};`, `cp ${file} ${dir}/${f2};`, and `echo ${file}|tr '/' '_'`. The variables `$dir` and `$allowlist` are set from `inputs.*`/`github.*` and must be double-quoted in all expansions.

Locations:

- `action.yml:60`
- `action.yml:62`
- `action.yml:63`
- `action.yml:64`

### script-injection (severity: high)

Sub-rule (b): The `vertex-execution` run block expands multiple env vars sourced from `inputs.*` and `github.*` without double-quoting, allowing shell metacharacter injection. Offending expansions include: `for file in ${notebooks};` (from `inputs.allowlist`), `--region=${region}` (from `inputs.region`), `--labels=commit_sha=${commit_sha}` (from `github.sha`), `machine-type="${machine_type}"` (from `inputs.vertex_machine_type`), `container-image-uri="${container}"` (from `inputs.vertex_container_name`), and `--kernel-name="${kernel}"` (from `inputs.kernel_name`). All loop-variable and flag expansions must be double-quoted.

Locations:

- `action.yml:90`
- `action.yml:96`
- `action.yml:97`
- `action.yml:100`

### github-env-injection (severity: high)

The `stage-files` step writes `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` where `${dir}` is derived from `github.sha` (untrusted). The value is not sanitized with `printf '%s' ... | tr -d '\n\r'` before the write, allowing newline injection into `$GITHUB_OUTPUT` which could poison subsequent step outputs or environment variables.

Locations:

- `action.yml:65`

### github-env-injection (severity: high)

The `vertex-execution` step writes `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT` without sanitizing the value with `printf '%s' ... | tr -d '\n\r'` before the write. The jobs.json content is built from `inputs.allowlist` (via `${notebooks}`) and other untrusted inputs, so newline characters in those inputs could inject additional key=value pairs into `$GITHUB_OUTPUT`.

Locations:

- `action.yml:103`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all 5 findings in action.yml:
1. unpinned-uses: Pinned setup-gcloud@v2 to SHA e427ad8a34f8676edf47cf7d7925499adf3eb74f and upload-cloud-storage@v2 to SHA c0f6160ff80057923ff50e5e567695cea181ec23.
2. script-injection (stage-files): Added double-quotes around all variable expansions: ${dir}, ${file}, ${dir}/${f2}, and the echo|tr pipeline.
3. script-injection (vertex-execution): Added double-quotes around ${file} in echo|tr pipeline, ${region} in --region flag, and ${commit_sha} in --labels flag.
4. github-env-injection (stage-files): Replaced unsafe echo with safe_notebooks=$(ls "${dir}" | xargs | tr -d '\n\r') followed by printf 'notebooks=%s\n' "${safe_notebooks}" >> "$GITHUB_OUTPUT".
5. github-env-injection (vertex-execution): Replaced unsafe echo with safe_jobs=$(cat jobs.json | tr -d '\n\r') followed by printf 'training_jobs=%s\n' "${safe_jobs}" >> "$GITHUB_OUTPUT".

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in action.yml:

1. stage-files step (line ~61): Replaced `for file in ${allowlist}` (unquoted expansion of attacker-controlled env var) with `IFS=',' read -ra allowlist_files <<< "${allowlist}"; for file in "${allowlist_files[@]}"`. This uses a controlled IFS delimiter (comma) to safely split the comma-separated list into a bash array, then iterates with proper double-quoting, preventing shell word-splitting and glob expansion on attacker-controlled input. Added space-trimming for each element.

2. vertex-execution step (line ~95): Same fix applied - replaced `for file in ${notebooks}` with `IFS=',' read -ra notebook_files <<< "${notebooks}"; for file in "${notebook_files[@]}"`. Also fixed the unquoted `echo $output` to `echo "$output"` to prevent word-splitting on the command output.

