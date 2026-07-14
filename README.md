# Laravel agentic skills

**Français** | [English](README.en.md)

Skills agentiques pour le développement **PHP / Laravel** assisté par IA.

Ce dépôt contient le plugin `laravel-tools` : un ensemble de skills réutilisables pour
initialiser, tester, analyser et maintenir des projets Laravel.

Il complète [`workflow-agentic-skills`](https://github.com/tomsjac/workflow-agentic-skills), qui
porte les skills génériques de workflow comme la création de plan de travail, les commits, les PR/MR
et les changelogs. Ici, le périmètre reste volontairement centré sur Laravel et l'écosystème PHP.

## Installation

### Claude Code

```shell
/plugin marketplace add tomsjac/laravel-agentic-skills
/plugin install laravel-tools@laravel-agentic-skills
```

### Autres agents

Les skills sont des fichiers Markdown autonomes dans `plugin/skills/<skill>/SKILL.md`. Un agent qui
sait charger ce format peut les utiliser directement depuis ce dépôt ou depuis une copie locale du
dossier `plugin/`.

## Compatibilité

- **Claude Code** : marketplace et manifest fournis dans `.claude-plugin/`.
- **Codex / agents compatibles `.agents`** : marketplace et manifest fournis dans `.agents/plugins/` et `plugin/.codex-plugin/`.
- **OpenCode, pi et autres agents** : contenu portable sous forme de skills Markdown, sans dépendance à un projet privé.

## Skills

### Outils Laravel

| Skill | À utiliser pour |
|---|---|
| `init-laravel` | Initialiser un projet Laravel : monolithe, API ou microservice |
| `write-tests` | Écrire des tests PHPUnit / Pest pour une feature Laravel |
| `fix-phpstan` | Analyser et corriger des erreurs PHPStan |
| `fix-tests-php` | Diagnostiquer et réparer des tests PHPUnit / Pest en échec |
| `audit-packages` | Auditer les dépendances Composer / npm, les mises à jour et les vulnérabilités |

### Veille

| Skill | À utiliser pour |
|---|---|
| `tech-news` | Générer une veille PHP, Laravel et développement web, avec l'IA appliquée au dev en complément |

## Agent inclus

| Agent | Rôle |
|---|---|
| `laravel-expert` | Revue d'architecture, sécurité, conventions et qualité sur un projet Laravel |

## Structure

```text
.claude-plugin/marketplace.json   # marketplace Claude Code
.agents/plugins/marketplace.json  # marketplace Codex / agents compatibles .agents
plugin/
  .claude-plugin/plugin.json      # manifest Claude Code
  .codex-plugin/plugin.json       # manifest Codex
  agents/                         # agents spécialisés
  skills/                         # skills Laravel
```

## Périmètre

Ce dépôt se concentre sur les besoins Laravel : création de projet, tests, PHPStan, dépendances et
veille PHP/Laravel. Pour les workflows transverses de développement, utilisez plutôt
[`workflow-agentic-skills`](https://github.com/tomsjac/workflow-agentic-skills).

## Contribution

Les contributions sont les bienvenues. Voir [CONTRIBUTING.md](CONTRIBUTING.md).

## Licence

[MIT](LICENSE)
