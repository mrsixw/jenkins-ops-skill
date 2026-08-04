# Jenkins Operation Safety

## Classify the Operation

Classify tools by behaviour, not by an assumed tool name.

| Class | Examples | Default rule |
|---|---|---|
| Read-only | Controller status, user identity, job/build metadata, logs, tests, queue inspection, node health, artifact listing | Run immediately when the controller is unambiguous |
| Read with local output | Artifact or log download | Confirm the destination if it could overwrite a file; otherwise use a new explicit path |
| Jenkins mutation | Trigger, rebuild, replay, cancel a queued item, abort a running build | Confirm the exact scope unless the user already authorised it |
| Local configuration | Add, edit, remove, or select a persistent Jeeves profile | Perform only when explicitly requested |
| Ambiguous controller | Any operation whose target controller is uncertain | Stop and ask before proceeding |

## Confirm Mutations

Confirm the controller, job, action, and material parameters:

```text
I'm about to {ACTION} `{JOB_PATH}` on {CONTROLLER}.
Parameters: {PARAMETERS_OR_NONE}
Proceed?
```

One approval covers only the stated action, job, controller, and parameters during the current task. A request such as "go ahead" or "do whatever you need" can authorize the current defined task, but it is not a standing unlock for unrelated jobs or controllers.

Do not re-confirm when the user's request already clearly authorizes the same mutation and scope.

## Handle Uncertain Mutation Outcomes

Network timeouts and connection resets can occur after Jenkins accepted a mutation.

1. Treat the operation as potentially successful.
2. Inspect the returned queue URL, queue contents, job history, or build causes.
3. Account for concurrent builds and matching parameters.
4. Report uncertainty if the result cannot be distinguished safely.
5. Never retry a trigger, rebuild, replay, or cancellation automatically.

## Protect Secrets

- Prefer credentials managed by the MCP runtime or environment variables supplied by the user.
- Never echo, log, commit, or include token values in the final response.
- Disable shell tracing before commands that expand credentials.
- Never type a token literal into a command or place one in a Jeeves profile. Expand only established credential variables where a client requires command-line authentication.
- Do not source an arbitrary secrets file. Use only credential setup the user has identified as trusted.
- Redact accidental credentials before surfacing command output.
- Treat console logs, build parameters, artifacts, replay scripts, and job configuration as potentially sensitive.
- Do not publish controller URLs, repository URLs, usernames, or job data outside the user's requested scope.

## Preserve Transport Security

- Keep TLS certificate verification enabled for MCP, Jeeves, and HTTP.
- For a private certificate authority, use the organisation's configured trust store or a user-provided CA bundle.
- Never add `curl -k`, `curl --insecure`, or an equivalent TLS bypass.
- Do not change proxy or certificate configuration merely to make a failing request succeed without user authorisation.

## Handle Local Configuration

Named Jeeves profiles may safely store a controller URL and username when the user requests that configuration. Keep the token in `JEEVES_TOKEN` at runtime.

Prefer `--profile NAME` for one invocation. Commands such as `profile use` or `profile add --default` change the persistent default and require explicit direction. Treat overwriting an existing profile URL as a controller-target change and confirm the new target.

## Handle Long-Running Reads

Live log following is read-only but may block until a build finishes. Use a snapshot by default. Follow a live log only when it helps the task, and use a bounded wait when the selected tool supports one.
