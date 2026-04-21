 BBQ — Système de gestion de contexte pour Claude Code

## Ce que c'est

BBQ est un système de commandes Claude Code pour la gestion de contexte, le brainstorming adversarial, la planification, et l'exécution auditée de fonctionnalités. C'est un outil, pas un framework. Il doit rester simple, sans friction, et adaptable à tout type de projet.

Le livrable est un dossier `.claude/` portable (commands + agents) accompagné d'une structure `.planning/` et `.ressources/` qui se crée dans le projet cible.

## Philosophie

- **Anti-vibecode** : chaque feature passe par un grill adversarial ou un mini-grill, une planification, et une exécution auditée. Rien n'est flou.
- **Context engineering** : les subagents reçoivent un contexte minimal et ciblé. Le thread principal reste propre. Les plans sont des prompts exécutables, pas des documents.
- **Zéro friction côté `.planning/`** : l'utilisateur ne gère jamais manuellement ces fichiers. Le système les crée, les update, et les maintient via le subagent `doc-updater`.
- **`.ressources/` est la zone utilisateur** : les agents y accèdent en lecture seule. Jamais d'écriture.
- **Simplicité d'abord** : peu de commandes, chacune avec un rôle clair.

## Stack du projet

Ce projet n'a pas de stack traditionnelle. C'est une collection de fichiers Markdown (.md) qui servent de commandes et d'agents pour Claude Code.

- **Commandes** : `.claude/commands/bbq/*.md` — chaque fichier = une commande slash
- **Agents** : `.claude/agents/*.md` — subagents réutilisables avec frontmatter
  - `scanner.md` — scan codebase + WebSearch technos → rapport structuré (opus)
  - `doc-writer.md` — rédige les 5 fichiers de doc (PROJECT, tech-stack, technique, non-technique, CLAUDE.md) (opus)
  - `doc-reviewer.md` — review la doc vs codebase (opus)
  - `doc-scanner.md` — scanne `.ressources/` pour trouver les fichiers pertinents (haiku)
  - `planner.md` — transforme une description en plan XML (opus)
  - `executor.md` — exécute un plan XML, code, retourne un résumé (opus)
  - `auditor.md` — audite le code contre spec/plan, sort un plan de correction si besoin (opus)
  - `doc-updater.md` — maintient la doc (mode feature) ou l'historique des bugs (mode bug) (opus)
- **Structure cible** : `.planning/` + `.ressources/` + `CLAUDE.md` racine dans le projet de l'utilisateur

## Conventions

### Format des commandes

Chaque commande utilise le frontmatter Claude Code :

```markdown
---
name: bbq:nom-commande
description: Description courte
argument-hint: "[arguments optionnels]"
allowed-tools: [Read, Bash, Write, Task]
model: claude-opus-4-6
---
```

### Format XML des plans d'exécution (produits par le subagent `planner`)

Les plans sont des prompts autonomes en XML — l'executor les exécute sans contexte additionnel :

```xml
<plan id="FXXX" version="N" feature="nom" date="YYYY-MM-DD" mode="feature|bug">
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
  <out_of_scope>Ce que l'executor ne doit PAS faire.</out_of_scope>
</plan>
```

### Structure cible dans le projet de l'utilisateur

```
.planning/
  instructions/
    PROJECT.md          # créé par fire, maintenu par doc-updater
    tech-stack.md       # créé par fire, maintenu par doc-updater
  documentations/
    technique.md        # doc pour devs — créée par fire, maintenue par doc-updater
    non-technique.md    # doc pour utilisateurs — créée par fire, maintenue par doc-updater
    [dynamiques.md]     # créés par doc-updater si nouveau concept (ex: stripe.md)
  backlog/
    FXXX-slug/
      spec.md           # créé par /bbq:grill (pas par /bbq:hotdogs)
      history.md        # créé/maintenu par doc-updater en mode bug (via /bbq:burns)
      plans/
        plan-vN.md      # créé par le subagent planner (via grill ou hotdogs)

CLAUDE.md               # racine, compact (<80 lignes), maintenu par doc-updater
.ressources/            # géré par l'utilisateur, agents LECTURE SEULE
```

### Statuts de cycle de vie

Features : `drafted` → `planned` → `in-progress` → `done`

### Écriture des commandes — bonnes pratiques

- Les instructions sont en langage naturel, structurées avec des sections claires
- Tout ce qui est interactif (questions à l'utilisateur) reste sur le thread principal
- Tout ce qui est travail silencieux (scan, rédaction, exécution, audit) est délégué à un subagent
- Un subagent ne peut PAS interagir avec l'utilisateur — il reçoit un prompt, travaille, retourne un résultat
- Passer le minimum de contexte aux subagents — seulement ce dont ils ont besoin
- Les subagents ont accès au filesystem et au web search
- Les subagents n'ont PAS accès aux images du thread principal (ils doivent les lire depuis le filesystem)

### Commandes BBQ

| Commande | Rôle | Subagents utilisés |
|----------|------|-------------------|
| `bbq:update` | Met à jour le système BBQ lui-même | — |
| `bbq:fire` | Setup projet (scan + interview + rédaction de 5 fichiers de doc) | scanner, doc-writer, doc-reviewer |
| `bbq:grill` | Brainstorm adversarial → spec.md + plan-v1.md | doc-scanner, planner |
| `bbq:cook` | Exécution auditée → code + commit | executor, auditor, doc-updater (feature) |
| `bbq:burns` | Correction directe d'un problème → code + history | planner, executor, doc-updater (bug) |
| `bbq:hotdogs` | Workflow rapide grill→cook→burns pour petites features | planner, executor, auditor, doc-updater |

### Flux standard

**Nouveau projet :**
```
/bbq:fire "description"  →  instructions/ + documentations/ + CLAUDE.md
```

**Grosse feature :**
```
/bbq:grill "description"  →  spec.md + plan-v1.md
/bbq:cook FXXX            →  code + audit + doc auto + commit
```

**Petite feature :**
```
/bbq:hotdogs "description"  →  mini-grill + plan + code + audit + doc + commit
```

**Bug / correctif :**
```
/bbq:burns FXXX "description"  →  plan interne + code + history.md
```

## Règles critiques

- **Ne jamais vibecoder** : chaque décision doit être explicite et validée
- **XML pour les plans** : les plans sont des prompts exécutables en XML structuré
- **Subagents = fire-and-forget** : instructions claires, contexte minimal, pas d'interaction utilisateur
- **`.planning/` est auto-géré** : l'utilisateur ne touche jamais ces fichiers manuellement
- **`.ressources/` est lecture seule pour les agents** : géré uniquement par l'utilisateur
- **CLAUDE.md racine reste compact** : < 80 lignes, sinon c'est du context rot
