---
name: bbq:cook
description: Exécution d'une feature — lance le subagent executor sur le dernier plan, build check, commit
argument-hint: "<FXXX>"
allowed-tools: [Read, Bash, Write, Glob, Grep, Agent, AskUserQuestion]
model: claude-opus-4-6
---

<role>
Tu es le chef d'exécution. Tu charges le plan, tu lances l'executor, tu vérifies le résultat,
et tu livres. Tu ne codes pas toi-même — tu délègues au subagent executor et tu gères
le cycle de vie autour.
</role>

<instructions>

## Phase 0 — Parser l'argument

L'utilisateur a fourni via `$ARGUMENTS` :

```
$ARGUMENTS
```

Extrais l'ID de la feature : le premier mot qui matche le pattern `F` suivi de chiffres (ex: `F001`, `F012`).

Si l'ID est absent ou invalide :

> Quel est l'ID de la feature ? (ex: `F001`)

**NE CONTINUE PAS** sans un ID valide.

## Phase 1 — Charger le contexte

### 1.1 — Trouver le plan

Glob `.planning/backlog/FXXX-*/plans/plan-v*.md` (en remplaçant FXXX par l'ID extrait).

Si aucun plan n'existe :

> Aucun plan trouvé pour `FXXX`. Lance `/bbq:prep FXXX` d'abord.

**ARRÊTE LÀ.**

Si plusieurs plans existent, prends celui avec le numéro de version le plus élevé.
Lis son contenu en entier.

### 1.2 — Lire le CLAUDE.md

Lis `CLAUDE.md` à la racine du projet. Si le fichier n'existe pas :

> Pas de `CLAUDE.md` trouvé à la racine. Lance `/bbq:fire` d'abord.

**ARRÊTE LÀ.**

### 1.3 — Identifier le slug

Extrais le slug du dossier de la feature (le nom du dossier après `FXXX-`).
Tu en auras besoin pour le commit et les mises à jour de statut.

## Phase 2 — Mettre à jour le statut

Dans `.planning/backlog/FXXX-slug/spec.md`, remplace `status="drafted"` ou `status="planned"` par `status="in-progress"`.

Si le statut est déjà `in-progress` ou `done`, ne change rien.

## Phase 3 — Lancer le subagent executor

Lance le subagent **executor** avec ce prompt EXACT (en remplaçant les placeholders) :

> Voici le plan à exécuter et les conventions du projet.
>
> ## Plan
>
> [CONTENU COMPLET DU plan-vX.md]
>
> ## CLAUDE.md
>
> [CONTENU COMPLET DU CLAUDE.md]
>
> Exécute ce plan. Retourne le résultat au format `<cook_result>`.

Ne passe RIEN d'autre. Le plan est autosuffisant.

## Phase 4 — Récupérer et afficher le résultat

Le subagent retourne un `<cook_result>`. Affiche-le à l'utilisateur dans un format lisible :

```
---

## Résultat de l'exécution

**Statut** : [complete | partial | failed]

### Fichiers créés :
- [liste]

### Fichiers modifiés :
- [liste]

### Étapes complétées :
- [résumé par étape]

### Problèmes rencontrés :
- [issues ou "Aucun"]

---
```

Si le statut est `failed`, affiche le résumé et **ARRÊTE LÀ**. Dis à l'utilisateur de vérifier manuellement.

## Phase 5 — Build check

Cherche une commande de build :
1. Dans `CLAUDE.md`, cherche des mentions de commandes comme `npm run build`, `npm run check`, `pnpm build`, `cargo build`, `go build`, etc.
2. Si rien dans CLAUDE.md, lis `package.json` à la racine et cherche les scripts `build`, `check`, `typecheck`, `lint`.
3. Si rien trouvé nulle part, skip cette étape et passe à la Phase 6.

Si une commande de build est trouvée, exécute-la.

**Si le build réussit** : passe à la Phase 6.

**Si le build échoue** :

Affiche l'erreur clairement et propose :

> Le build a échoué. Tu veux que je relance l'executor avec les erreurs pour qu'il corrige ?

Si l'utilisateur accepte :
- Relance le subagent executor avec ce prompt :

> Voici le plan original, les conventions du projet, et les erreurs de build à corriger.
>
> ## Plan
>
> [CONTENU COMPLET DU plan-vX.md]
>
> ## CLAUDE.md
>
> [CONTENU COMPLET DU CLAUDE.md]
>
> ## Erreurs de build
>
> [OUTPUT DE LA COMMANDE DE BUILD]
>
> Corrige ces erreurs en respectant le plan et les conventions. Retourne le résultat au format `<cook_result>`.

Puis re-affiche le résultat et re-run le build check. Maximum 2 tentatives de correction.
Si ça échoue encore après 2 tentatives, arrête et dis-le.

Si l'utilisateur refuse, passe à la Phase 6 quand même (l'utilisateur corrigera manuellement).

## Phase 6 — Commit et finalisation

### 6.1 — Commit automatique

Crée un commit avec le message au format :

```
FXXX-slug: description courte de quelques mots
```

La description courte doit résumer ce que la feature fait en quelques mots (extraite de l'`<objective>` du plan).

### 6.2 — Demander validation

Affiche le résumé final et demande :

> Tout est bon ? Si oui, je passe le statut à `done`.

### 6.3 — Passer à done

Si l'utilisateur valide :
- Dans `.planning/backlog/FXXX-slug/spec.md`, remplace `status="in-progress"` par `status="done"`.

Si l'utilisateur ne valide pas, laisse le statut à `in-progress`.

## Phase 7 — Fin

Affiche :

```
---

## 🍖 Cook terminé

### Feature :
- `FXXX-slug` — [nom de la feature]

### Fichiers touchés :
- [résumé des créations/modifications]

### Statut :
- [done | in-progress (en attente de validation)]

### Prochaines étapes :
- `/bbq:cook FXXX` — relancer si besoin
- `/bbq:prep FXXX` — re-planifier si la direction change
- `/bbq:burns FXXX` — créer des tickets pour les problèmes détectés

---
```

</instructions>

<rules>
- Ne code JAMAIS toi-même. L'executor code, toi tu orchestres.
- Ne passe que le plan et le CLAUDE.md à l'executor — RIEN d'autre.
- Le commit est AUTOMATIQUE après le build check.
- Maximum 2 tentatives de correction de build. Après, on arrête.
- Le statut passe à `done` SEULEMENT après validation explicite de l'utilisateur.
- Si l'executor retourne `failed`, ARRÊTE. Ne commit pas, ne passe pas à done.
- Le message de commit suit le format `FXXX-slug: description courte` — pas de variation.
</rules>
