<!-- markdownlint-disable -->

# Hardening Report: google-github-actions--run-vertexai-notebook/v1.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **google-github-actions--run-vertexai-notebook/v1.1.1** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation in the 'stage-files' step. Shell variables that hold input data are expanded without quotes. allowlist comes from inputs.allowlist and dir comes from github.sha. Offending lines: `mkdir -p ${dir};`, `for file in ${allowlist};`, `f2=$(echo ${file}|tr '/' '_');`, `cp ${file} ${dir}/${f2};` and `echo "notebooks=$(ls ${dir} | xargs)"`.

Locations:

- `action.yml:60`
- `action.yml:61`
- `action.yml:63`
- `action.yml:64`
- `action.yml:66`

### script-injection (severity: high)

Rule (b) violation in the 'vertex-execution' step. Input-derived env vars are expanded without quotes. Offending lines: `for file in ${notebooks};` (from inputs.allowlist), `file=$(echo ${file}|tr '/' '_');`, `--region=${region} \` (from inputs.region), `--labels=commit_sha=${commit_sha} \` and `echo $output | jq -c > training.json`.

Locations:

- `action.yml:96`
- `action.yml:98`
- `action.yml:105`
- `action.yml:107`
- `action.yml:111`

### github-env-injection (severity: high)

The 'stage-files' step writes to $GITHUB_OUTPUT a value built from file names taken from inputs.allowlist: `echo "notebooks=$(ls ${dir} | xargs)" >> $GITHUB_OUTPUT`. The value is not passed through `printf '%s' ... | tr -d '\n\r'` before the write.

Locations:

- `action.yml:66`

### github-env-injection (severity: high)

The 'vertex-execution' step writes jobs.json to $GITHUB_OUTPUT: `echo "name=training_jobs=$(cat jobs.json)" >> $GITHUB_OUTPUT`. jobs.json contains values derived from the inputs (display names from allowlist file names, plus the region, kernel and container inputs). The value is not passed through `printf '%s' ... | tr -d '\n\r'` before the write.

Locations:

- `action.yml:116`

### unpinned-uses (severity: high)

Two composite steps in action.yml reference actions by mutable tags instead of 40-character commit SHAs: `uses: 'google-github-actions/setup-gcloud@v2'` and `uses: 'google-github-actions/upload-cloud-storage@v2'`. The step using actions/github-script is pinned to a SHA and passes.

Locations:

- `action.yml:70`
- `action.yml:73`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses

**Notes:**

All five findings are fixed in action.yml. Nothing has been run or tested yet.

- **Pinned actions:** setup-gcloud@v2 is now pinned to e427ad8a34f8676edf47cf7d7925499adf3eb74f and upload-cloud-storage@v2 to c0f6160ff80057923ff50e5e567695cea181ec23. I got both SHAs from lookup_action_sha.
- **Allowlist split:** in the stage-files and vertex-execution steps, the allowlist is still split on whitespace only, as upstream's unquoted for-loop did, using `read -r -d '' -a`. It is not split on commas, even though the input's description says "comma separated".
- **Glob patterns:** upstream's unquoted loop also expanded patterns like `somedir/**.ipynb`. Each item is now expanded with `compgen -G`, and kept as written if nothing matches, which is what bash does by default.
- **Quoting:** all paths and variables are now double-quoted, including dir, file, region and commit_sha. `echo` is replaced with `printf '%s'`.
- **jq call:** the job JSON is now passed to jq with `--argjson` instead of being pasted unquoted into the jq expression. The resulting jobs list should be the same.
- **Outputs:** both values written to `$GITHUB_OUTPUT` are captured first with `|| true` and then have newlines stripped with `tr -d '\n\r'`. The `|| true` keeps upstream's behaviour, where echo's exit status hid a failing `ls` or `cat`.
- **Upstream quirk kept:** the `name=training_jobs=...` output key is left exactly as upstream wrote it, odd as it looks.

