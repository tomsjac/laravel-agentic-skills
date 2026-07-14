# Contribuer à Laravel agentic skills

**Français** | [English](CONTRIBUTING.en.md)

Merci de votre intérêt ! Ce marketplace regroupe des skills pour le développement PHP / Laravel.

## Structure

```
.claude-plugin/marketplace.json   # catalogue des plugins
plugin/
  .claude-plugin/plugin.json      # manifeste du plugin
  skills/<skill>/SKILL.md         # un skill = un dossier avec SKILL.md (frontmatter + instructions)
  agents/<agent>.md               # agents éventuels
```

## Conventions

- **Noms** en `kebab-case` (plugins, skills, marketplace).
- Le champ `name` du frontmatter d'un `SKILL.md` doit correspondre au nom de son dossier.
- Contenu **générique** : aucune référence à une entreprise, une URL privée, un outil propriétaire
  ou un projet interne. Le toolkit est multi-forge (GitHub & GitLab) et sans dépendance MCP propriétaire.
- Texte visible par l'utilisateur en **français** ; instructions internes de skill en anglais ou français.

## Avant de proposer une PR

1. Valider la syntaxe :
   ```bash
   claude plugin validate .
   claude plugin validate ./plugin
   ```
2. Vérifier l'absence de références internes/propriétaires dans vos ajouts.
3. Bumper la `version` du `plugin.json` modifié (les utilisateurs ne reçoivent les mises à jour
   que si la version change).

## Licence

En contribuant, vous acceptez que votre contribution soit publiée sous licence [MIT](LICENSE).
