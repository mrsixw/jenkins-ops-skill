# Jeeves CLI Reference

Jeeves is the Tier 2 Jenkins CLI. Use it only when the selected Jenkins MCP server is unavailable for the required operation.

This reference describes the v1.8.1 command surface. Treat installed help as authoritative because older or newer versions may differ.

## Discover Jeeves

```bash
command -v jeeves
jeeves --version
jeeves --no-update-check --help
```

If the executable is absent, do not install it automatically. Report the [Jeeves releases page][jeeves-releases] and evaluate Tier 3.

Before using a documented capability, inspect its help:

```bash
jeeves job --help
jeeves build --help
jeeves node --help
jeeves profile --help
```

Do not use deprecated flat aliases or substitute an unverified repository-local build.

## Supply Connection Details

Jeeves accepts one controller per invocation through a named profile, its native environment variables, or global options. Prefer profiles for non-secret connection metadata and environment variables for credentials.

Native variables:

```bash
export JEEVES_URL="https://jenkins.example.com"
export JEEVES_USER="jenkins-username"
export JEEVES_TOKEN="$JENKINS_API_TOKEN"
export JEEVES_NO_UPDATE_CHECK=1
```

When generic Jenkins variables are already configured, map them for one invocation without persisting values:

```bash
JEEVES_URL="$JENKINS_URL" \
JEEVES_USER="$JENKINS_USER" \
JEEVES_TOKEN="$JENKINS_TOKEN" \
JEEVES_NO_UPDATE_CHECK=1 \
jeeves status
```

Never enter a real token as a literal, print it, pass it through `--token`, or save it in a profile. Do not run commands with shell tracing enabled when credentials expand.

## Use Named Profiles

```bash
jeeves profile list
jeeves --profile PROFILE status
jeeves profile add PROFILE --url URL --username USER
jeeves profile add PROFILE --url URL --username USER --default
jeeves profile add PROFILE --url URL --force
jeeves profile use PROFILE
jeeves profile use --clear
jeeves profile remove PROFILE
```

`--profile PROFILE` and `JEEVES_PROFILE` select a configured controller; `--profile` wins. Prefer `--profile` for a one-off selection.

`profile use`, `profile add --default`, `profile add`, and `profile remove` modify local configuration. Run them only when explicitly requested. Never add `--token`. Treat `profile add PROFILE --force --url URL` as a controller-target change because it retargets later uses of that profile.

## Command Catalogue

Confirm each group against installed help before use.

```bash
# Controller
jeeves status
jeeves whoami

# Jobs
jeeves job list
jeeves job list --folder FOLDER
jeeves job list --expand
jeeves job params JOB
jeeves job trigger JOB --param KEY=VALUE
jeeves job trigger JOB --wait --timeout 600 --poll-interval 10

# Builds
jeeves build summary JOB
jeeves build list JOB
jeeves build list JOB --limit 50 --result FAILURE
jeeves build list JOB --param ENV=prod --param REGION=eu-west-1
jeeves build show JOB BUILD
jeeves build blame JOB BUILD
jeeves build log JOB BUILD
jeeves build log JOB BUILD --follow
jeeves build rebuild JOB
jeeves build rebuild JOB --build BUILD
jeeves build rebuild JOB --param ENV=stage
jeeves build cancel JOB BUILD

# Queue and nodes
jeeves queue
jeeves node list
jeeves node list --stats
jeeves node list --address
jeeves node list --stats --address
```

`build show`, `build blame`, and `build log` accept a build number or permalink such as `lastBuild` or `lastFailedBuild`. `build cancel` requires an integer build number and cannot cancel a queued item.

`build log --follow` has no timeout option in v1.8.1 and streams until completion; prefer a snapshot unless following is explicitly useful. `node list --address` makes another Jenkins request per node, so avoid unnecessarily broad repeated calls.

## Prefer Structured Output

Place global output options before the command group:

```bash
jeeves --format json job list
jeeves --format json build show JOB BUILD
jeeves --format ndjson build list JOB
jeeves --format csv node list
jeeves --format tsv job params JOB
jeeves --format markdown build summary JOB
jeeves --format tree job list --expand
jeeves --format template --template "{name}: {status}" job list
```

In v1.8.1, structured JSON results are top-level arrays, including `build show`. Empty results return `[]` with exit status 0, so check the array length rather than relying on status alone.

| Command | Common JSON record keys |
|---|---|
| `profile list` | `name`, `url`, `username`, `token_set`, `default` |
| `job list` | `name`, `type`, `color`, `status`, `health`, `url` |
| `job params` | `name`, `type`, `default`, `choices`, `description` |
| `build summary` | `permalink`, `number`, `result`, `building`, `timestamp`, `duration`, `url` |
| `build list`, `build show` | `number`, `result`, `building`, `timestamp`, `duration`, `url`, `params`, `causes` |
| `build blame` | `commit`, `author`, `author_url`, `email`, `timestamp`, `message` |
| `queue` | `name`, `reason`, `stuck`, `url` |
| `node list` | `name`, `status`, `executors`, `labels`, `url` |
| `node list --stats` | Node keys plus `disk`, `temp`, `swap`, `response_ms`, `clock_ms`, `architecture` |
| `node list --address` | Node keys plus `address` |

Decorative output goes to stderr while structured data and logs go to stdout. Do not parse decorative headers.

## Script Read-Only Queries

```bash
# Last build result
jeeves --format json build summary MY/JOB/PATH \
  | jq -r '.[] | select(.permalink == "last") | .result'

# One build with an explicit not-found check
jeeves --format json build show MY/JOB/PATH 142 \
  | jq 'if length == 0 then error("build not found")
        else .[0] | {result, params, causes, url}
        end'

# Failed builds with matching parameters
jeeves --format json build list MY/JOB/PATH \
  --result FAILURE --param ENV=prod \
  | jq '.[] | {number, params, causes, url}'

# Node health
jeeves --format json node list --stats \
  | jq '.[] | {name, disk, temp, swap, response_ms, clock_ms, architecture}'
```

## Read Build History Past the Newest 100

`build list` reads the job API's `builds` field, which Jenkins limits to the
newest 100 builds. In v1.12.0 a larger `--limit` returns 100 builds without a
warning. Check the count: when exactly 100 come back and the oldest started at
or after the beginning of the period you need, the list is truncated.

Paged history past the newest 100 is a Jeeves capability gap, so apply the
fallback chain to it:

1. Use the MCP server's `allBuilds` range when MCP is available (see the MCP
   capability guide).
2. Otherwise use the Tier 3 `allBuilds` recipe in the HTTP API reference when
   its URL and credentials are configured.
3. Otherwise read older builds one at a time with `build show JOB NUMBER`,
   working down from the oldest number returned, in parallel batches. Skip
   numbers that return not found, because discarded builds leave gaps. Stop at
   build 1 or once a whole batch of found builds started before the period.

## Handle Triggers Carefully

Without `--wait`, `job trigger` confirms dispatch but does not return a queue URL or build number in v1.8.1. Inspect `queue`, then `build summary` or `build list`, while accounting for concurrent builds.

With `--wait`, progress goes to stderr and a human-readable build number and verdict go to stdout; `--format` does not make this result structured. Always pass `--timeout SECONDS` for unattended use. The timeout bounds only the local wait and does not cancel the Jenkins build.

Exit statuses are `0` for success, `1` for failure, unknown verdict, or command error, `2` for unstable, `3` for aborted or cancelled, and `124` for timeout. A non-zero status never authorizes an automatic retry because the original build may have been dispatched.

## Evaluate Remaining Gaps

In v1.8.1, common Jeeves gaps include artifact listing/download, Pipeline stage breakdown through `wfapi`, and queued-item cancellation. Verify installed help first because these capabilities may be added later. Use Tier 3 only if the selected Jenkins MCP server and installed Jeeves both lack the operation.

---

[jeeves-releases]: https://github.com/mrsixw/jeeves/releases
