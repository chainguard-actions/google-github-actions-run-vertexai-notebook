<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.3** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Composite action steps point to version tags instead of 40-character commit SHAs: line 70 `uses: 'google-github-actions/setup-gcloud@v2'` and line 73 `uses: 'google-github-actions/upload-cloud-storage@v2'`.

Locations:

- `action.yml:70`
- `action.yml:73`

### script-injection (severity: high)

Rule (b) broken in step 'stage-files'. Shell variables holding untrusted data are used without double quotes. `allowlist` comes from inputs.allowlist and `dir` from github.sha. Offending lines: `mkdir -p ${dir};`, `for file in ${allowlist};`, `f2=$(echo ${file}|tr '/' '_');`, `cp ${file} ${dir}/${f2};`, `echo "notebooks=$(ls ${dir} | xargs)"`.

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:63`
- `action.yml:64`
- `action.yml:66`

### script-injection (severity: high)

Rule (b) broken in step 'vertex-execution'. Env vars set from inputs are expanded without double quotes. Offending lines: `for file in ${notebooks};`, `file=$(echo ${file}|tr '/' '_');`, `--region=${region} \`, `--labels=commit_sha=${commit_sha} \`, `echo $output | jq -c > training.json` (the output includes input-derived data), and `jq '.jobs[.jobs | length] |= . + '$(cat training.json) jobs.json` (unquoted command substitution of input-derived data).

Locations:

- `action.yml:96`
- `action.yml:98`
- `action.yml:105`
- `action.yml:107`
- `action.yml:111`
- `action.yml:112`

### github-env-injection (severity: high)

Step 'stage-files' runs `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`. The file names come from inputs.allowlist, and the value is written to $GITHUB_OUTPUT without first sanitizing it with `printf '%s' ... | tr -d '\n\r'`.

Locations:

- `action.yml:66`

### github-env-injection (severity: high)

Step 'vertex-execution' runs `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT`. jobs.json is built from gcloud output that contains input-controlled values (display name, args, container, kernel), and it is written to $GITHUB_OUTPUT without first sanitizing it with `printf '%s' ... | tr -d '\n\r'`.

Locations:

- `action.yml:116`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Pinned setup-gcloud@v2 to e427ad8a34f8676edf47cf7d7925499adf3eb74f and upload-cloud-storage@v2 to c0f6160ff80057923ff50e5e567695cea181ec23. In stage-files and vertex-execution, every variable is now double-quoted. allowlist is split into an array on whitespace with `read -r -d '' -a ... || true`, guarded so an empty value adds nothing. That matches how upstream's unquoted `for file in ${allowlist}` split it, except that it no longer expands globs. The gcloud output is passed to jq quoted with printf. The new job is appended with `jq --slurpfile` instead of pasting `$(cat training.json)` unquoted into the jq program; the resulting JSON is the same. Both $GITHUB_OUTPUT writes now capture the value first (with `|| true`, because upstream ran the command inside echo's arguments), strip it with `tr -d '\n\r'`, and write to a quoted "$GITHUB_OUTPUT".

