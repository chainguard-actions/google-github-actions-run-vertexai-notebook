<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.2** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags instead of immutable 40-character commit SHAs, making the action vulnerable to supply-chain attacks if those tags are moved:
- `google-github-actions/setup-gcloud@v2` (line 68)
- `google-github-actions/upload-cloud-storage@v2` (line 71)
These are marked `# ratchet:exclude` but remain unpinned. They should be replaced with full SHA digests.

Locations:

- `action.yml:68`
- `action.yml:71`

### script-injection (severity: high)

Rule (b) violation — unquoted shell variable expansions of env vars holding untrusted workflow-controllable data.

**`stage-files` step (lines 60–65):** `${dir}` (derived from `${{ github.sha }}`), `${allowlist}` (derived from `${{ inputs.allowlist }}`), and `${file}` are all expanded unquoted in shell commands:
  - `mkdir -p ${dir};`
  - `for file in ${allowlist};`
  - `f2=$(echo ${file}|tr '/' '_');`
  - `cp ${file} ${dir}/${f2};`

**`vertex-execution` step (lines 92–100):** Multiple env vars holding untrusted inputs are expanded unquoted:
  - `for file in ${notebooks};` — `notebooks` holds `${{ inputs.allowlist }}`
  - `--region=${region}` — `region` holds `${{ inputs.region }}`
  - `--labels=commit_sha=${commit_sha}` — `commit_sha` holds `${{ github.sha }}`
  - `--worker-pool-spec=machine-type="${machine_type}",...,container-image-uri="${container}"` — `machine_type` and `container` hold user inputs; the outer `--worker-pool-spec=` value is unquoted
  - `--kernel-name="${kernel}"` — `kernel` holds `${{ inputs.kernel_name }}`

Unquoted expansions allow shell metacharacter injection (`;`, `|`, `&`, `$(...)`, glob chars, whitespace splitting).

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:63`
- `action.yml:64`
- `action.yml:92`
- `action.yml:95`
- `action.yml:97`
- `action.yml:98`

### github-env-injection (severity: high)

The `stage-files` step writes a value derived from an untrusted input to `$GITHUB_OUTPUT` without sanitization.

The env var `dir` is set to `'./${{ github.sha }}'` (an attacker-controllable value via the calling workflow's context). Inside the `run:` block, `${dir}` is used unquoted and its contents are written directly to `$GITHUB_OUTPUT`:

```
echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT
```

The required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`) is absent before the write. A newline injected into `dir` could allow an attacker to inject arbitrary key=value pairs into the GitHub output context, potentially overwriting subsequent step outputs.

Locations:

- `action.yml:65`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/action.yml:

1. unpinned-uses: Pinned google-github-actions/setup-gcloud@v2 to SHA e427ad8a34f8676edf47cf7d7925499adf3eb74f and google-github-actions/upload-cloud-storage@v2 to SHA c0f6160ff80057923ff50e5e567695cea181ec23, preserving version tags as comments.

2. script-injection: Quoted all unquoted variable expansions in both the stage-files step (${dir}, ${file}, ${dir}/${f2}) and the vertex-execution step (${file}, and the gcloud flag arguments --region, --labels, --worker-pool-spec, --args are now double-quoted as whole arguments). The for-loop iterators ${allowlist} and ${notebooks} remain unquoted as they are intentionally word-split lists.

3. github-env-injection: Added sanitization before writing to $GITHUB_OUTPUT in the stage-files step: the notebooks list is captured into a variable and passed through 'tr -d '\n\r'' to strip newlines before writing, preventing newline injection attacks.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in action.yml:
1. 'stage-files' step (line 62): Replaced unquoted `for file in ${allowlist};` with a safe xargs-based bash array tokenization. The allowlist value is now tokenized using `printf '%s' "$allowlist" | xargs printf '%s\0'` read into a bash array, then iterated with `"${files[@]}"`.
2. 'vertex-execution' step (line 99): Same fix applied to `for file in ${notebooks};` using a `nb_files` array. Both fixes include the required `if [ -n "$VAR" ]` guard to prevent xargs from emitting an empty token on empty input.

### Iteration 3

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed two high-severity findings in hardened/action/action.yml:
1. script-injection (line 122): Quoted `$output` variable in `echo "$output" | jq -c > training.json` to prevent shell metacharacter interpretation from the gcloud JSON output.
2. github-env-injection (line 127): Replaced direct `cat jobs.json` write to GITHUB_OUTPUT with sanitized form: `safe_jobs=$(printf '%s' "$(cat jobs.json)" | tr -d '\n\r')` followed by `echo "training_jobs=${safe_jobs}" >> "$GITHUB_OUTPUT"`, stripping newlines to prevent injection of arbitrary key=value pairs into the GitHub output context.

