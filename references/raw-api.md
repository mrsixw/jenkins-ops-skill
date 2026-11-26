# Jenkins HTTP API Reference

Use Jenkins HTTP endpoints only when the selected Jenkins MCP server and Jeeves are both unavailable for the required operation.

The Jenkins remote API is REST-like rather than one uniform REST API. Controller versions and installed plugins determine the exact endpoints. Check the controller's own `.../api/` pages when a documented endpoint is unavailable.

## Require Explicit Connection Details

Use API-token authentication and keep credentials in environment variables:

```bash
: "${JENKINS_URL:?Set the Jenkins controller URL}"
: "${JENKINS_USER:?Set the Jenkins username}"
: "${JENKINS_TOKEN:?Set the Jenkins API token}"
JENKINS_URL="${JENKINS_URL%/}"
```

Do not print these variables or run credential-bearing commands with shell tracing enabled. The examples expand the token from the environment; never replace the expansion with a literal token.

Keep TLS verification enabled. If the controller uses a private certificate authority, add `--cacert "$JENKINS_CA_BUNDLE"` to requests only after the user or environment provides that bundle. Never use `-k` or `--insecure`.

## Use Canonical Object URLs

Prefer exact URLs returned by Jenkins, MCP, or Jeeves. Jenkins folder paths repeat `/job/` for every item level. For example:

```bash
JOB_URL="$JENKINS_URL/job/Team/job/Service"
BUILD_URL="$JOB_URL/142"
```

Percent-encode each folder, job, branch, and artifact path segment when constructing a URL. Do not append an untrusted slash-delimited job name directly.

## Run Read-Only Requests

Controller metadata:

```bash
curl -fsS --globoff --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$JENKINS_URL/api/json?tree=nodeDescription,mode,useSecurity"
```

Build metadata:

```bash
curl -fsS --globoff --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$BUILD_URL/api/json?tree=number,result,building,url,queueId,actions[parameters[*]]" \
  | jq '{number, result, building, url, queueId}'
```

Build history past the newest 100 (`builds` stops there; see the MCP capability guide for paging rules):

```bash
curl -fsS --globoff --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$JOB_URL/api/json?tree=allBuilds[number,timestamp,result,url]{100,125}" \
  | jq '.allBuilds'
```

Console snapshot:

```bash
curl -fsS --globoff --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$BUILD_URL/consoleText"
```

Queue inventory:

```bash
curl -fsS --globoff --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$JENKINS_URL/queue/api/json?tree=items[id,why,stuck,task[name,url],actions[parameters[*]]]" \
  | jq '.items[] | {id, job: .task.name, why, stuck}'
```

Artifact inventory:

```bash
curl -fsS --globoff --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$BUILD_URL/api/json?tree=artifacts[fileName,relativePath]" \
  | jq '.artifacts[]'
```

Pipeline stages, when the [Pipeline REST API plugin][pipeline-rest-api] is installed:

```bash
curl -fsS --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$BUILD_URL/wfapi/describe" \
  | jq '.stages[] | {name, status, durationMillis}'
```

For progressive console output, read the `X-Text-Size` and `X-More-Data`
response headers and keep the polling window bounded:

```bash
headers_file="$(mktemp)"
trap 'rm -f "$headers_file"' EXIT
offset=0
more_data=true
attempts=0

while [ "$attempts" -lt 60 ] && [ "$more_data" = true ]; do
  curl -fsS --globoff \
    --dump-header "$headers_file" \
    --user "$JENKINS_USER:$JENKINS_TOKEN" \
    "$BUILD_URL/logText/progressiveText?start=$offset"

  next_offset="$(awk 'tolower($1) == "x-text-size:" {
    sub(/\r$/, "", $2); print $2
  }' "$headers_file" | tail -n 1)"
  more_data="$(awk 'tolower($1) == "x-more-data:" {
    sub(/\r$/, "", $2); print tolower($2)
  }' "$headers_file" | tail -n 1)"

  case "$next_offset" in
    ''|*[!0-9]*)
      printf '%s\n' 'Jenkins returned no valid X-Text-Size header' >&2
      exit 1
      ;;
  esac

  offset="$next_offset"
  attempts=$((attempts + 1))
  [ "$more_data" = true ] && sleep 2
done

if [ "$more_data" = true ]; then
  printf '%s\n' 'Progressive log polling reached its time limit' >&2
  exit 124
fi
```

Prefer a console snapshot when live following is unnecessary. Add the same
user-provided `--cacert` option used for other requests when the controller has
a private certificate authority.

## Download an Artifact

Confirm an explicit destination and avoid overwriting an existing file without authorisation:

```bash
ARTIFACT_URL="$BUILD_URL/artifact/path/to/artifact"
curl -fsSL --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$ARTIFACT_URL" \
  --output ./artifact-name
```

Use the artifact URL returned by Jenkins when available. Otherwise percent-encode each artifact path segment.

## Handle CSRF Correctly

API tokens are preferred. Jenkins 2.96 and later normally exempt requests authenticated with an API token from CSRF crumbs. Do not fetch a crumb pre-emptively.

If a controller explicitly rejects an authenticated `POST` because a crumb is required, obtain the configured field and value:

```bash
CRUMB_JSON=$(curl -fsS --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$JENKINS_URL/crumbIssuer/api/json")
CRUMB_FIELD=$(printf '%s' "$CRUMB_JSON" | jq -r '.crumbRequestField')
CRUMB_VALUE=$(printf '%s' "$CRUMB_JSON" | jq -r '.crumb')
```

Add `--header "$CRUMB_FIELD: $CRUMB_VALUE"` to that controller's mutating requests. Older password-authenticated controllers may bind crumbs to a web session; retain the response cookie when the controller requires it. See the [Jenkins CSRF guidance][jenkins-csrf].

## Run Mutations Once

Confirm the controller, job, action, and parameters before every not-yet-authorised mutation. The examples omit an optional crumb header; add it only when the controller requires one.

Trigger without parameters and retain response headers, including a queue `Location` when supplied:

```bash
curl -fsS --request POST \
  --user "$JENKINS_USER:$JENKINS_TOKEN" \
  --dump-header - --output /dev/null \
  "$JOB_URL/build"
```

Trigger with parameters using encoded form data:

```bash
curl -fsS --request POST \
  --user "$JENKINS_USER:$JENKINS_TOKEN" \
  --data-urlencode "ENV=stage" \
  --data-urlencode "REGION=eu-west-1" \
  --dump-header - --output /dev/null \
  "$JOB_URL/buildWithParameters"
```

Cancel a selected queue item:

```bash
curl -fsS --request POST \
  --user "$JENKINS_USER:$JENKINS_TOKEN" \
  --data-urlencode "id=$QUEUE_ID" \
  "$JENKINS_URL/queue/cancelItem"
```

Stop a selected running build:

```bash
curl -fsS --request POST \
  --user "$JENKINS_USER:$JENKINS_TOKEN" \
  "$BUILD_URL/stop"
```

After any timeout or lost response, inspect the queue and build history before taking another action. Never repeat a mutation automatically.

## Upstream References

- [Jenkins Remote Access API][jenkins-remote-api]
- [Jenkins CSRF API-token guidance][jenkins-csrf]
- [Pipeline REST API plugin][pipeline-rest-api]

---

[jenkins-csrf]: https://www.jenkins.io/doc/developer/security/misc/
[jenkins-remote-api]: https://www.jenkins.io/doc/book/using/remote-access-api/
[pipeline-rest-api]: https://plugins.jenkins.io/pipeline-rest-api/
