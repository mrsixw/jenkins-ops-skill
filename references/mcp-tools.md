# Jenkins MCP Capability Guide

Jenkins MCP implementations do not share a guaranteed namespace, tool count, or naming scheme. Discover the tools that the current runtime actually exposes and match capabilities from their descriptions and input schemas.

## Discover the Server and Tools

Use the discovery surface provided by the host agent:

- **Codex and ChatGPT:** inspect the tools already exposed in the session. If tool search is available, search for `Jenkins`, the controller name, and the configured MCP server name. Tool identifiers commonly include an MCP server namespace, but do not depend on one exact prefix.
- **Claude Code:** inspect the MCP tools and server namespaces surfaced in the current session. Match the configured server to the target controller before calling a tool.
- **Other Agent Skills clients:** use the runtime's tool catalogue or MCP server list. If the client provides no way to surface a configured Jenkins tool, treat MCP as unavailable rather than guessing a call name.

Do not install, reconfigure, or authenticate an MCP server unless the user explicitly requests that configuration change.

When several Jenkins MCP servers are present, resolve the target using the user's controller name or URL, repository configuration, and server descriptions. Ask when those signals do not identify one server.

## Verify Access Safely

Prefer a read-only identity or controller-status tool for one health/auth check. Common semantic names include `whoAmI`, `getStatus`, `status`, or `getVersion`, but names vary.

If no dedicated probe exists, call the smallest read-only tool relevant to the request, such as listing one folder or reading one known job. Distinguish these outcomes:

| Outcome | Interpretation |
|---|---|
| Tool succeeds | MCP is usable for capabilities exposed by that server |
| Tool is absent | MCP lacks that capability; evaluate Tier 2 |
| Connection/authentication fails | MCP is unavailable until its configuration changes |
| Permission is denied | MCP is unavailable for that protected operation, not necessarily for all reads |
| Job or build is not found | Recheck the controller and path; do not classify the MCP transport as down |

Do not repeat the health probe for every operation after it succeeds or fails conclusively.

## Match Operations to Capabilities

Use descriptions and schemas as the contract. The names below are examples seen in Jenkins MCP implementations, not required identifiers.

| Operation | Matching capability | Common example names | Risk |
|---|---|---|---|
| Identify current user | Return authenticated Jenkins identity | `whoAmI` | Read-only |
| Read controller status | Return health, mode, or version | `getStatus`, `status` | Read-only |
| List jobs or folders | Enumerate jobs, optionally below a folder | `getJobs`, `listJobs` | Read-only |
| Read job details | Return job metadata or configuration | `getJob` | Read-only |
| Find a job by repository | Search job SCM remotes | `findJobsWithScmUrl` | Read-only |
| Read build metadata | Return a numbered build, result, parameters, causes, and URL | `getBuild` | Read-only |
| Read changes or SCM | Return commits, branch, revision, and repository | `getBuildChangeSets`, `getBuildScm` | Read-only |
| Read tests | Return test suites and failures | `getTestResults` | Read-only |
| Read or search logs | Return console text or matching lines | `getBuildLog`, `searchBuildLog` | Read-only |
| Inspect queue | List queue items or fetch one by ID | `getQueue`, `getQueueItem` | Read-only |
| Inspect agents | List nodes and health, labels, executors, or launcher address | `getNodes`, `listAgents` | Read-only |
| List or download artifacts | Return artifacts or artifact content | `getArtifacts`, `downloadArtifact` | Read-only local write for downloads |
| Read Pipeline stages | Return stage names, states, and timings | `getPipelineStages` | Read-only |
| Trigger a build | Start a job with optional parameters | `triggerBuild` | Mutating |
| Rebuild a build | Repeat a prior build, normally with its parameters | `rebuildBuild` | Mutating |
| Replay a Pipeline | Read replay scripts, then run edited or unchanged scripts | `getReplayScripts`, `replayBuild` | Read then mutating |
| Cancel work | Cancel a queue item or running build | `cancelQueueItem`, `cancelBuild` | Mutating |

Before declaring a capability gap:

1. Check all Jenkins tools exposed for the selected server.
2. Compare required inputs and returned data with the operation.
3. Combine read-only MCP calls when that fully answers the request.
4. Do not substitute a similarly named tool whose description targets another controller or object type.

## Fall Back Per Operation

Evaluate the fallback chain independently for each operation. Examples:

- If MCP can read build metadata but cannot stream logs, use MCP for metadata and evaluate Jeeves for log streaming.
- If MCP can trigger a build but cannot list the full queue, trigger through MCP and evaluate Jeeves for queue inspection.
- If one MCP call fails because a job path is wrong, correct the path before moving tiers.
- If a mutating MCP call times out after dispatch, inspect queue and build history. Never repeat the mutation merely because the response was lost.

MCP capabilities evolve. Always prefer a newly exposed MCP capability over the lower-tier examples documented in this skill.
