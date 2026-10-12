<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.2** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two composite action steps reference external actions pinned to mutable version tags instead of full 40-character commit SHAs. An attacker who compromises the upstream repository could push malicious code to the tag and have it execute in any workflow using this action.

Failing references:
- `google-github-actions/setup-gcloud@v2` (line 70) — mutable tag, not a SHA
- `google-github-actions/upload-cloud-storage@v2` (line 73) — mutable tag, not a SHA

The third step (`actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea`) is correctly pinned.

Locations:

- `action.yml:70`
- `action.yml:73`

### script-injection (severity: high)

Rule (b) violation: Multiple env vars sourced from `inputs.*` and `github.*` are expanded unquoted inside `run:` shell scripts, allowing an attacker to inject shell metacharacters (`;`, `|`, `&`, `$(...)`, glob chars, whitespace word-splitting).

**`stage-files` step (lines 61–67):**
- `mkdir -p ${dir};` — `$dir` holds `'./${{ github.sha }}'`, unquoted
- `for file in ${allowlist};` — `$allowlist` holds `${{ inputs.allowlist }}`, unquoted; word-splitting is intentional but metacharacters in filenames can still be injected
- `cp ${file} ${dir}/${f2};` — both `$file` and `$dir` are unquoted
- `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT` — `$dir` unquoted inside command substitution

**`vertex-execution` step (lines 93–106):**
- `for file in ${notebooks};` — `$notebooks` holds `${{ inputs.allowlist }}`, unquoted
- `--region=${region}` — `$region` holds `${{ inputs.region }}`, unquoted
- `--display-name="${job_name}"` — `$job_name` is derived from unquoted `$commit_sha` and `$file`
- `--labels=commit_sha=${commit_sha}` — `$commit_sha` unquoted
- `machine-type="${machine_type}"` — `$machine_type` holds `${{ inputs.vertex_machine_type }}`, unquoted outside the quoted segment
- `container-image-uri="${container}"` — `$container` holds `${{ inputs.vertex_container_name }}`, unquoted outside the quoted segment
- `--kernel-name="${kernel}"` — `$kernel` holds `${{ inputs.kernel_name }}`, unquoted outside the quoted segment

Locations:

- `action.yml:61`
- `action.yml:62`
- `action.yml:65`
- `action.yml:67`
- `action.yml:93`
- `action.yml:97`
- `action.yml:99`
- `action.yml:100`
- `action.yml:101`

### github-env-injection (severity: high)

Two `run:` steps write values derived from untrusted inputs to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

**`stage-files` step (line 67):**
```
echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT
```
`$dir` is set from `'./${{ github.sha }}'` (caller-controlled via `github.sha`). The output of `ls ${dir}` is written directly to `$GITHUB_OUTPUT` with no newline stripping, allowing a crafted directory name to inject additional key=value pairs into the output file.

**`vertex-execution` step (line 107):**
```
echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT
```
`jobs.json` is populated by a loop over `$notebooks` (which holds `${{ inputs.allowlist }}`). Attacker-controlled notebook filenames flow into `jobs.json` content and then into `$GITHUB_OUTPUT` without sanitization, enabling newline injection to set arbitrary output variables.

Locations:

- `action.yml:67`
- `action.yml:107`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/action.yml:

1. unpinned-uses: Pinned google-github-actions/setup-gcloud@v2 to SHA e427ad8a34f8676edf47cf7d7925499adf3eb74f and google-github-actions/upload-cloud-storage@v2 to SHA c0f6160ff80057923ff50e5e567695cea181ec23, preserving the original tags in comments.

2. script-injection: Quoted all variable expansions in both stage-files and vertex-execution steps. Variables used in for-loop iteration (${allowlist}, ${notebooks}) remain unquoted as word-splitting is intentional for list iteration. All other usages of $dir, $file, $region, $commit_sha, $machine_type, $container, $kernel, etc. are now properly double-quoted.

3. github-env-injection: Both GITHUB_OUTPUT writes now use a two-step sanitization: capture the value into a raw variable, then pipe through `printf '%s' ... | tr -d '\n\r'` to strip newlines before writing to $GITHUB_OUTPUT, preventing newline injection attacks.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in action.yml:
1. In the 'stage-files' step (line 57): Replaced unquoted `for file in ${allowlist}` with a safe xargs-based bash array tokenization. The allowlist input is now tokenized via `printf '%s' "$allowlist" | xargs printf '%s\0'` into a `files` array, then iterated with `"${files[@]}"`.
2. In the 'vertex-execution' step (line 83): Replaced unquoted `for file in ${notebooks}` with a safe xargs-based bash array tokenization. The notebooks (allowlist) input is now tokenized via `printf '%s' "$notebooks" | xargs printf '%s\0'` into a `nb_files` array, then iterated with `"${nb_files[@]}"`.
Both fixes include the required empty-value guard (`if [ -n "$VAR" ]`) to prevent xargs from emitting an empty token on empty input. The steps both use `shell: 'bash'` so bash arrays and process substitution are valid.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed two bare assignments that would abort the step under errexit: (1) line 71: `raw_notebooks=$(ls "${dir}" | xargs) || true` and (2) line 128: `raw_jobs=$(cat jobs.json) || true`. Both now match upstream behavior where the commands were run inside echo/printf whose exit status hid the command's, allowing the step to continue even on non-zero exit.

