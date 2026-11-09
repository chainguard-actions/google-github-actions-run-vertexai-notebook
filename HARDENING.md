<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.3** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation in step 'stage-files'. Env vars set from workflow inputs are expanded without quotes. `allowlist` comes from inputs.allowlist and `dir` from github.sha. Offending lines: `mkdir -p ${dir};`, `for file in ${allowlist};`, `f2=$(echo ${file}|tr '/' '_');`, `cp ${file} ${dir}/${f2};`, `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`. None of these are the exempt NAME=($VAR) array form with a shellcheck disable comment.

Locations:

- `action.yml:59`

### script-injection (severity: high)

Rule (b) violation in step 'vertex-execution'. Env vars set from workflow inputs are expanded without quotes. Offending lines: `for file in ${notebooks};` (from inputs.allowlist), `file=$(echo ${file}|tr '/' '_');`, `--region=${region} \` (from inputs.region), `--labels=commit_sha=${commit_sha} \`, `echo $output | jq -c > training.json`, `jq '.jobs[.jobs | length] |= . + '$(cat training.json) jobs.json`.

Locations:

- `action.yml:95`

### github-env-injection (severity: high)

Step 'stage-files' writes a value built from inputs to $GITHUB_OUTPUT without sanitizing it first. The file names come from inputs.allowlist and the directory from github.sha. Line: `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`. No `printf '%s' ... | tr -d '\n\r'` step runs before the write.

Locations:

- `action.yml:66`

### github-env-injection (severity: high)

Step 'vertex-execution' writes jobs.json to $GITHUB_OUTPUT without sanitizing it first. jobs.json holds gcloud job output built from inputs (notebook names, region, machine type, container, kernel). Line: `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT`. No `printf '%s' ... | tr -d '\n\r'` step runs before the write.

Locations:

- `action.yml:122`

### unpinned-uses (severity: high)

Composite action steps reference actions by mutable tag instead of a full 40-character commit SHA: `uses: 'google-github-actions/setup-gcloud@v2' # ratchet:exclude` and `uses: 'google-github-actions/upload-cloud-storage@v2' # ratchet:exclude`.

Locations:

- `action.yml:70`
- `action.yml:73`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses

**Notes:**

stage-files and vertex-execution: allowlist/notebooks are now split with the exempt unquoted array form (arr=($VAR) plus a shellcheck disable comment). It splits on whitespace and globs exactly as upstream's 'for file in ${allowlist}' did. Every other expansion is now double-quoted: dir, file, region, commit_sha and output. The jq append that pasted an unquoted $(cat training.json) into the filter now passes the job via --argjson, with the same result. Both $GITHUB_OUTPUT writes now capture the value first ('|| true' keeps upstream's echo-wrapped exit-status behaviour), strip CR/LF with printf | tr -d '\n\r', and write to a quoted "$GITHUB_OUTPUT". setup-gcloud@v2 is pinned to e427ad8a34f8676edf47cf7d7925499adf3eb74f and upload-cloud-storage@v2 to c0f6160ff80057923ff50e5e567695cea181ec23.

