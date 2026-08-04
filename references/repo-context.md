# Repository and Controller Context

Use repository evidence to confirm whether Jenkins is relevant and to identify the intended controller. Do not infer a company-specific mapping from a repository name.

## Detect Jenkins

Common Jenkins signals include:

- `Jenkinsfile` files in the root or subdirectories.
- `jenkins/`, `.jenkins/`, or `jenkins-config/` directories.
- Pipeline shared-library declarations in Groovy.
- Makefile, shell, TOML, YAML, or deployment configuration that contains a Jenkins URL or job path.
- Documentation or badges that link to Jenkins builds.

Other CI signals include:

- GitHub Actions: `.github/workflows/*.yml` or `.github/workflows/*.yaml`.
- GitLab CI: `.gitlab-ci.yml`.
- CircleCI: `.circleci/config.yml`.
- Buildkite: `.buildkite/`.
- Azure Pipelines: `azure-pipelines.yml`.

The presence of another CI system does not prove Jenkins is unused.

## Resolve the Controller

Use signals in this order:

1. A controller name, URL, MCP server, or Jeeves profile explicitly supplied by the user.
2. A controller URL or identifier in the relevant `Jenkinsfile`, pipeline library configuration, or repository documentation.
3. An MCP server description that unambiguously matches the repository or URL.
4. An active Jeeves profile or `JENKINS_URL`/`JEEVES_URL` value that unambiguously matches the request.

Repository-name patterns and shared-library names are supporting evidence only. Prefer explicit URLs and configuration when signals conflict.

Do not print environment values that may contain credentials. Controller URLs are normally safe to report, but honor any user instruction that treats them as sensitive.

## Decision Matrix

| Evidence | User intent | Action |
|---|---|---|
| Jenkins only | Jenkins or general CI operation | Use this skill |
| Another CI only | General CI operation | Use tooling for that CI system |
| Jenkins and another CI | Request names a Jenkins job, build, controller, console, or `Jenkinsfile` | Use this skill |
| Jenkins and another CI | Request clearly names the other CI system | Use that system's tooling |
| Jenkins and another CI | Operation ownership is ambiguous | Ask which system runs that operation |
| No CI configuration | Jenkins request names a controller or job | Continue with the supplied Jenkins context |
| No CI configuration | General CI request | Ask which CI system and target to use |

## Disambiguation Prompt

```text
This repository contains both Jenkins and {OTHER_CI} configuration.
Is {OPERATION} handled by Jenkins, or by {OTHER_CI}?
```

When the CI system is clear but the controller is not:

```text
Which Jenkins controller should I use for {JOB_OR_OPERATION}?
I found these configured candidates: {CONTROLLER_NAMES_OR_URLS}.
```
