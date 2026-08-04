---
name: jenkins-ops
description: Operate Jenkins controllers with a strict capability-aware fallback chain. Prefer any configured Jenkins MCP server, fall back to the Jeeves CLI, and use Jenkins HTTP APIs only as a last resort. Use for jobs, builds, console logs, queues, nodes or agents, artifacts, pipeline stages, replays, rebuilds, cancellations, and build triggers on any Jenkins instance.
---

# Jenkins Ops

Operate one Jenkins controller at a time and apply the fallback chain separately to every requested operation.

## Establish the Target

1. Confirm Jenkins is the relevant CI system. Inspect the repository for a `Jenkinsfile`, Jenkins configuration, and other CI configuration when the request is ambiguous.
2. Resolve the controller from, in descending confidence order:
   - A controller, URL, MCP server, or Jeeves profile named by the user.
   - A Jenkins URL or controller identifier in repository configuration.
   - An unambiguous configured Jenkins MCP server, `JEEVES_PROFILE`, or `JENKINS_URL`/`JEEVES_URL` value.
3. If multiple controllers remain plausible, ask which one to use. Never guess the target of a mutation.
4. Identify the exact operation and whether it is read-only, mutating Jenkins, or changing local Jeeves configuration.

Use the [repository context guide][repo-context] for CI and controller discovery.

## Enforce the Fallback Chain

Use the highest available tier that supports the current operation:

1. Jenkins MCP
2. Jeeves CLI
3. Jenkins HTTP API

Do not choose a lower tier because it is shorter, faster, or more familiar. Using a lower tier for one capability gap does not waive MCP-first behaviour for later operations.

A tier is unavailable for an operation only when one of these conditions is established:

- The runtime exposes no matching tool or executable.
- A single safe health or read probe establishes a connection or authentication failure.
- The exposed inputs and descriptions do not cover the required capability.
- The authenticated identity lacks permission for that operation.

State the reason before falling back. Do not repeatedly probe a known outage. Treat an uncertain result from a mutating call as potentially successful and never retry it automatically.

## Tier 1: Jenkins MCP

Inspect the tools surfaced by the current runtime for Jenkins MCP servers. MCP namespaces and tool names vary, so match by server and tool descriptions rather than assuming a prefix, fixed tool count, or exact names.

1. Select the MCP server associated with the resolved controller. Ask if multiple servers remain ambiguous.
2. Prefer a read-only identity or status tool as the health/auth probe. If neither exists, use the smallest read-only tool needed for the request.
3. Match the operation to the MCP capability before declaring a gap:

| Intent | Required MCP capability |
|---|---|
| Check access or health | Identify the user or read controller status |
| Discover jobs | List, fetch, or search jobs |
| Inspect a build | Read build metadata, changes, SCM data, or test results |
| Read logs | Fetch, search, or stream console output |
| Inspect queues | List queue items or fetch a queue item |
| Inspect agents | List nodes or read node health and launcher details |
| Start or repeat work | Trigger, rebuild, or replay a build |
| Cancel work | Cancel a queue item or running build |
| Retrieve outputs | List or download artifacts, or read Pipeline stages |

4. Combine MCP read tools when that satisfies the operation. Fall back only for the missing operation, not the whole session.

See the [MCP capability guide][mcp-capabilities] for runtime discovery and common capability mappings.

## Tier 2: Jeeves CLI

Use Jeeves only after Tier 1 is unavailable for the operation.

1. Verify availability and inspect the installed command surface:

   ```bash
   command -v jeeves
   jeeves --version
   jeeves --no-update-check --help
   ```

2. Check the relevant group help, such as `jeeves job --help` or `jeeves build --help`. Do not assume a command exists from the version string alone.
3. Prefer an existing named profile or the `JEEVES_URL`, `JEEVES_USER`, and `JEEVES_TOKEN` environment variables. If only `JENKINS_URL`, `JENKINS_USER`, and `JENKINS_TOKEN` exist, pass their values to the corresponding Jeeves variables for that invocation without persisting them.
4. Never print a token, place it in command-line arguments, or store it in a Jeeves profile. If no usable authentication is configured, request it or continue only when Tier 3 has independently configured credentials.
5. Do not install Jeeves or change profiles unless the user explicitly requests that local change.

If Jeeves is absent, report the [public releases page][jeeves-releases] as the setup path and continue to Tier 3 when it is configured and capable.

See the [Jeeves reference][jeeves-reference] for current commands, structured output, and version-sensitive behaviour.

## Tier 3: Jenkins HTTP API

Use HTTP only after both higher tiers are unavailable for the operation. Require an explicit controller URL and usable authentication; never infer credentials.

- Prefer `JENKINS_URL`, `JENKINS_USER`, and `JENKINS_TOKEN` environment variables.
- Keep TLS verification enabled. Use a user-provided CA bundle for private certificate authorities; never add `curl -k` or `--insecure`.
- Prefer API tokens. Modern Jenkins versions normally exempt API-token-authenticated requests from CSRF crumbs; handle a crumb only when the controller requires one.
- Reuse exact job and build URLs returned by Jenkins, MCP, or Jeeves. For folders, encode each item name and insert `/job/` between path segments.
- Use form data or URL encoding for parameters instead of interpolating untrusted values into a URL.

See the [HTTP API reference][http-reference] for last-resort recipes.

If no tier has the required capability and valid access, stop with an actionable summary of what is missing. Do not improvise a fourth access path or weaken authentication or TLS settings.

## Apply Safety Gates

Run read-only operations immediately when the controller is unambiguous. Before a mutation that the user has not already authorised, confirm the controller, job, action, and parameters once for that exact scope.

Require explicit user direction before changing Jeeves profiles or persistent defaults. Never treat broad authorisation as permission for another controller or unrelated job.

Read the [safety guide][safety-guide] before triggering, rebuilding, replaying, cancelling, changing local configuration, or using a raw HTTP `POST`.

## Verify Mutations

After a mutation:

1. Capture any queue item, build number, or URL returned by the call.
2. Check status with the highest available tier for that follow-up operation.
3. Account for concurrent builds when a trigger does not return a build number.
4. Report the final known state and canonical Jenkins URL.
5. If dispatch outcome is uncertain, report that uncertainty and inspect the queue/build history. Do not re-trigger automatically.

---

[http-reference]: references/raw-api.md
[jeeves-reference]: references/jeeves.md
[jeeves-releases]: https://github.com/mrsixw/jeeves/releases
[mcp-capabilities]: references/mcp-tools.md
[repo-context]: references/repo-context.md
[safety-guide]: references/safety.md
