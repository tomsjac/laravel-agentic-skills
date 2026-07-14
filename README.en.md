# Laravel agentic skills

[Français](README.md) | **English**

Agentic skills for AI-assisted **PHP / Laravel** development.

This repository contains the `laravel-tools` plugin: a set of reusable skills to bootstrap, test,
analyze and maintain Laravel projects.

It complements [`workflow-agentic-skills`](https://github.com/tomsjac/workflow-agentic-skills),
which contains generic workflow skills such as work plans, commits, PRs/MRs and changelogs. This
repository intentionally stays focused on Laravel and the PHP ecosystem.

## Installation

### Claude Code

```shell
/plugin marketplace add tomsjac/laravel-agentic-skills
/plugin install laravel-tools@laravel-agentic-skills
```

### Other agents

Skills are standalone Markdown files in `plugin/skills/<skill>/SKILL.md`. Any agent able to load
that format can use them directly from this repository or from a local copy of the `plugin/`
directory.

## Compatibility

- **Claude Code**: marketplace and manifest are provided in `.claude-plugin/`.
- **Codex / `.agents`-compatible agents**: marketplace and manifest are provided in `.agents/plugins/` and `plugin/.codex-plugin/`.
- **OpenCode, pi and other agents**: portable Markdown skills, with no dependency on a private project.

## Skills

### Laravel tooling

| Skill | Use it to |
|---|---|
| `init-laravel` | Bootstrap a Laravel project: monolith, API or microservice |
| `write-tests` | Write PHPUnit / Pest tests for a Laravel feature |
| `fix-phpstan` | Analyze and fix PHPStan errors |
| `fix-tests-php` | Diagnose and repair failing PHPUnit / Pest tests |
| `audit-packages` | Audit Composer / npm dependencies, updates and vulnerabilities |

### Daily watch

| Skill | Use it to |
|---|---|
| `tech-news` | Generate a PHP, Laravel and web development digest, with AI applied to development as a complement |

## Included agent

| Agent | Role |
|---|---|
| `laravel-expert` | Architecture, security, convention and quality review for Laravel projects |

## Structure

```text
.claude-plugin/marketplace.json   # Claude Code marketplace
.agents/plugins/marketplace.json  # Codex / .agents-compatible marketplace
plugin/
  .claude-plugin/plugin.json      # Claude Code manifest
  .codex-plugin/plugin.json       # Codex manifest
  agents/                         # specialized agents
  skills/                         # Laravel skills
```

## Scope

This repository focuses on Laravel needs: project bootstrap, tests, PHPStan, dependencies and
PHP/Laravel monitoring. For cross-project development workflows, use
[`workflow-agentic-skills`](https://github.com/tomsjac/workflow-agentic-skills).

## Contributing

Contributions are welcome. See [CONTRIBUTING.en.md](CONTRIBUTING.en.md).

## License

[MIT](LICENSE)
