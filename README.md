# Jenkins Ops Agent Skill

Operate Jenkins through a strict, capability-aware fallback chain:

1. Any configured Jenkins MCP server.
2. The Jeeves CLI.
3. Jenkins HTTP APIs.

The skill is controller-agnostic. It discovers MCP capabilities at runtime, validates Jeeves commands from installed help, and contains no organisation-specific controller hosts, usernames, token names, or MCP namespaces.

## Install

Ask Codex's bundled skill installer to install the repository root under the
skill name `jenkins-ops`:

```text
$skill-installer Install the root skill from https://github.com/mrsixw/jenkins-ops-skill and name it jenkins-ops.
```

Or clone it directly into a user skill directory:

```bash
git clone https://github.com/mrsixw/jenkins-ops-skill \
  ~/.agents/skills/jenkins-ops
```

Skills-compatible clients may use a different user skill directory. Keep the installed folder name as `jenkins-ops` so it matches the skill metadata.

## Requirements

- An Agent Skills-compatible client.
- Access to at least one Jenkins controller.
- A configured Jenkins MCP server for the preferred tier, when available.
- [Jeeves][jeeves] for the second tier, when installed and configured.
- `curl` and `jq` only for raw HTTP fallback operations.

The skill does not install tools, configure credentials, or bundle an MCP server.

## Contents

- [SKILL.md][skill] — fallback workflow and operational rules.
- [agents/openai.yaml][openai-metadata] — Codex and ChatGPT UI metadata.
- [references/mcp-tools.md][mcp-guide] — runtime-neutral MCP capability discovery.
- [references/jeeves.md][jeeves-guide] — Jeeves commands and structured output.
- [references/raw-api.md][http-guide] — last-resort Jenkins HTTP recipes.
- [references/repo-context.md][repo-context] — CI and controller discovery.
- [references/safety.md][safety-guide] — mutation, credential, and TLS safeguards.

## Security

Keep credentials in runtime-managed configuration or environment variables. Never commit tokens, store them in Jeeves profiles, disable TLS verification, or retry an uncertain Jenkins mutation automatically.

## License

Licensed under the [Apache License 2.0][license].

---

[http-guide]: references/raw-api.md
[jeeves]: https://github.com/mrsixw/jeeves
[jeeves-guide]: references/jeeves.md
[license]: LICENSE
[mcp-guide]: references/mcp-tools.md
[openai-metadata]: agents/openai.yaml
[repo-context]: references/repo-context.md
[safety-guide]: references/safety.md
[skill]: SKILL.md
