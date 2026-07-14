# Contributing to Laravel agentic skills

[Français](CONTRIBUTING.md) | **English**

Thanks for your interest! This marketplace bundles skills for PHP / Laravel development.

## Structure

```
.claude-plugin/marketplace.json   # plugin catalog
plugin/
  .claude-plugin/plugin.json      # plugin manifest
  skills/<skill>/SKILL.md         # one skill = one folder with SKILL.md (frontmatter + instructions)
  agents/<agent>.md               # optional agents
```

## Conventions

- **Names** in `kebab-case` (plugins, skills, marketplace).
- A `SKILL.md` frontmatter `name` must match its folder name.
- **Generic** content: no reference to a company, private URL, proprietary tool or internal
  project. The toolkit is multi-forge (GitHub & GitLab) and has no proprietary MCP dependency.
- User-facing text is in **French** by default; internal skill instructions may be in English or French.

## Before opening a PR

1. Validate the syntax:
   ```bash
   claude plugin validate .
   claude plugin validate ./plugin
   ```
2. Make sure your additions contain no internal/proprietary references.
3. Bump the `version` of the modified `plugin.json` (users only receive updates when the version
   changes).

## License

By contributing, you agree that your contribution is published under the [MIT](LICENSE) license.
