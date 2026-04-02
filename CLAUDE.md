# BBQ — Système de gestion de contexte pour Claude Code

## Ce que c'est

BBQ est un système de commandes Claude Code pour la gestion de contexte, le brainstorming adversarial, et l'exécution structurée de fonctionnalités. C'est un outil, pas un framework. Il doit rester simple, sans friction, et adaptable à tout type de projet.

Le livrable est un dossier `.claude/` portable (commands + agents) accompagné d'une structure `.planning/` et `.docs/` qui se crée dans le projet cible.

## Philosophie

- **Anti-vibecode** : chaque feature passe par un grill adversarial, une planification avec questions, et une exécution structurée. Rien n'est flou.
- **Context engineering** : les subagents reçoivent un contexte minimal et ciblé. Le thread principal reste propre. Les plans sont des prompts exécutables, pas des documents.
- **Zéro friction** : l'utilisateur ne gère jamais manuellement les fichiers `.planning/`. Le système les crée, les update, et les maintient.
- **Simplicité d'abord** : commencer petit, itérer. Pas de 40 commandes dès le jour 1.

## Stack du projet

Ce projet n'a pas de stack traditionnelle. C'est une collection de fichiers Markdown (.md) qui servent de commandes et d'agents pour Claude Code.

- **Commandes** : `.claude/commands/bbq/*.md` — chaque fichier = une commande slash
- **Agents** : `.claude/agents/*.md` — subagents réutilisables avec frontmatter
  - `scanner.md` — scan codebase + WebSearch technos → rapport structuré
  - `doc-writer.md` — rédige tech-stack.md, PROJECT.md, CLAUDE.md (Opus 4.6)
  - `doc-reviewer.md` — review la doc vs le codebase, identifie les manques
  - `doc-scanner.md` — scanne .docs/ pour trouver les fichiers pertinents à une feature
- **Structure cible** : `.planning/` et `.docs/` créés dans le projet de l'utilisateur

## Conventions

### Format des commandes

Chaque commande utilise le frontmatter Claude Code :

```markdown
---
name: bbq:nom-commande
description: Description courte
argument-hint: "[arguments optionnels]"
allowed-tools: [Read, Bash, Write, Task]
---
```

### Format XML des plans d'exécution (produits par /bbq:prep)

Les plans sont des prompts autonomes en XML — le subagent de /bbq:cook les exécute sans contexte additionnel :

```xml
<plan id="FXXX" version="N" feature="nom" date="YYYY-MM-DD">
  <objective>Ce que le plan accomplit en 1-2 phrases.</objective>
  <context>Stack, patterns, fichiers pertinents — minimum nécessaire.</context>
  <steps>
    <step n="1">
      <name>Nom descriptif</name>
      <what>Ce qui doit être fait — haut niveau, concret.</what>
      <files>Fichiers à créer ou modifier</files>
    </step>
  </steps>
  <success_criteria>Critères vérifiables concrètement.</success_criteria>
  <out_of_scope>Ce que le cook ne doit PAS faire.</out_of_scope>
</plan>
```

### Structure `.planning/` dans le projet cible

```
.planning/
  PROJECT.md              # Contexte riche du projet (but, architecture, conventions, décisions)
  tech-stack.md           # Référence technique (stack, versions, bonnes pratiques, mises en garde)
  ROADMAP.md              # Backlog ordonné (optionnel, pour projets avec livrable)
  config.json             # Type de projet, préférences
  backlog/
    FXXX-slug/
      spec.md             # Spécification (créée par /bbq:grill)
      plans/
        plan-v1.md        # Plan d'exécution (créé par /bbq:prep)
      issues/
        ISS-XXX.md        # Tickets de problèmes (créés par /bbq:burns)

.docs/
  # Ressources du projet : images, PDFs, markdown de référence
  # Les commandes scannent ce dossier pour du contexte pertinent
```

### Statuts de cycle de vie

Features : `drafted` → `planned` → `in-progress` → `done`
Issues : `open` → `in-progress` → `resolved`

### Écriture des commandes — bonnes pratiques

- Les instructions sont en langage naturel, structurées avec des sections claires
- Tout ce qui est interactif (questions à l'utilisateur) reste sur le thread principal
- Tout ce qui est travail silencieux (scan, exécution) est délégué à un subagent
- Un subagent ne peut PAS interagir avec l'utilisateur — il reçoit un prompt, travaille, retourne un résultat
- Passer le minimum de contexte aux subagents — seulement ce dont ils ont besoin
- Les subagents ont accès au filesystem et au web search
- Les subagents n'ont PAS accès aux images du thread principal (ils doivent les lire depuis le filesystem)

### Commandes BBQ (roadmap)

| Commande | Rôle | Statut |
|----------|------|--------|
| `bbq:update` | Met à jour le système BBQ lui-même | 🔨 En cours |
| `bbq:fire` | Setup projet (scan + interview + tech-stack.md + PROJECT.md + CLAUDE.md) | ✅ Fait |
| `bbq:grill` | Brainstorm adversarial → spec.md | ✅ Fait |
| `bbq:marinade` | Recherche/collecte de contexte (optionnel) | À faire |
| `bbq:prep` | Planification itérative → plan-vX.md | ✅ Fait |
| `bbq:cook` | Exécution via subagent → code + commit | ✅ Fait |
| `bbq:burns` | Création de tickets d'issues | ✅ Fait |
| `bbq:status` | Vue d'ensemble du projet | À faire |
| `bbq:menu` | Roadmap / backlog global (optionnel) | À faire |

## Règles critiques

- **Ne jamais vibecoder** : chaque décision doit être explicite et validée
- **XML pour les plans** : les plans sont des prompts exécutables en XML structuré
- **Subagents = fire-and-forget** : instructions claires, contexte minimal, pas d'interaction utilisateur
- **`.planning/` est auto-géré** : l'utilisateur ne touche jamais ces fichiers manuellement
- **`.docs/` est géré par l'utilisateur** : il y met ses ressources, les commandes y cherchent du contexte