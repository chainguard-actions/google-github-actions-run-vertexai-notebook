<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.2** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two `uses:` references in action.yml are pinned to mutable version tags instead of immutable full 40-character SHA commits, making the action vulnerable to supply-chain attacks if those tags are moved:
- `google-github-actions/setup-gcloud@v2` (line 68)
- `google-github-actions/upload-cloud-storage@v2` (line 71)
These should be pinned to full SHA digests (e.g. `google-github-actions/setup-gcloud@<sha> # v2`).

Locations:

- `action.yml:68`
- `action.yml:71`

### script-injection (severity: high)

Sub-rule (b): Multiple env vars holding untrusted input values are expanded unquoted inside `run:` shell commands, allowing shell metacharacter injection.

In the `stage-files` step (lines 60–65), `${dir}` (sourced from `inputs` via `github.sha`) and `${allowlist}` (sourced from `inputs.allowlist`) are used unquoted:
- `mkdir -p ${dir};`  — unquoted
- `for file in ${allowlist};`  — unquoted (word-splits on whitespace/globs)
- `cp ${file} ${dir}/${f2};`  — unquoted

In the `vertex-execution` step (lines 91–101), multiple env vars sourced from `inputs.*` are used unquoted:
- `for file in ${notebooks};`  — unquoted
- `--region=${region} \`  — unquoted
- `--labels=commit_sha=${commit_sha} \`  — unquoted
- `--worker-pool-spec=machine-type="${machine_type}",...,container-image-uri="${container}"` — partially unquoted in the spec string
- `--kernel-name="${kernel}"` — quoted but embedded in a comma-separated arg string that is itself unquoted

An attacker controlling `inputs.allowlist`, `inputs.region`, `inputs.vertex_machine_type`, `inputs.vertex_container_name`, or `inputs.kernel_name` can inject arbitrary shell commands.

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:64`
- `action.yml:91`
- `action.yml:96`
- `action.yml:98`

### github-env-injection (severity: high)

The `stage-files` step writes a value derived from an untrusted input to `$GITHUB_OUTPUT` without the required sanitization (`printf '%s' ... | tr -d '\n\r'`).

Line 66: `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`

Here `${dir}` is set from `'./${{ github.sha }}'`. Although `github.sha` is not directly attacker-controlled in the same way as PR head refs, it is still a `github.*` context value that flows through the env block and is written unsanitized to `$GITHUB_OUTPUT`. A newline embedded in the value could allow injection of additional key=value pairs into the output file, potentially overwriting subsequent outputs read by downstream steps.

Locations:

- `action.yml:66`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/action.yml:

1. unpinned-uses: Pinned google-github-actions/setup-gcloud@v2 to full SHA e427ad8a34f8676edf47cf7d7925499adf3eb74f and google-github-actions/upload-cloud-storage@v2 to full SHA c0f6160ff80057923ff50e5e567695cea181ec23.

2. script-injection: In stage-files step, quoted all variable expansions: mkdir -p "${dir}", echo "${file}", cp "${file}" "${dir}/${f2}". In vertex-execution step, quoted: echo "${file}", --region="${region}", --labels="commit_sha=${commit_sha}", and wrapped the entire --worker-pool-spec and --args values in double quotes.

3. github-env-injection: In stage-files step, captured the ls output into a variable sanitized with tr -d '\n\r' before writing to $GITHUB_OUTPUT, preventing newline injection attacks.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection, unpinned-uses

**Notes:**

Fixed all four findings:
1. stage-files script-injection: Replaced unquoted `for file in ${allowlist}` with IFS-based safe splitting: `IFS=',' read -ra files <<< "${allowlist}"` and `for file in "${files[@]}"`.
2. vertex-execution script-injection: Same IFS-based fix for `${notebooks}` loop. Also fixed `echo $output` to `echo "$output"`.
3. github-env-injection: Added `safe_jobs=$(printf '%s' "$(cat jobs.json)" | tr -d '\n\r')` before writing to GITHUB_OUTPUT to strip newlines. Also fixed the output key (removed spurious `name=` prefix).
4. unpinned-uses: Pinned both `google-github-actions/.github` reusable workflow references from `@v3` to full SHA `@29c6d38eeb974133b4b66401985f7c70cf4a6681` with `# v3` comment in both draft-release.yml and release.yml.

