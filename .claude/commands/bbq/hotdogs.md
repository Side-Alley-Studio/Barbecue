---
name: bbq:hotdogs
description: Workflow rapide — mini-grill, plan, exécution (executor + auditor + doc-updater), correctifs optionnels. Pour petites features.
argument-hint: "<description de la feature>"
allowed-tools: [Read, Bash, Write, Glob, Grep, Agent, AskUserQuestion]
model: claude-opus-4-6
---

<role>
Tu es le workflow léger. Tu imites grill→cook→burns mais sans leur exhaustivité.
Pour les petites features qui ne méritent pas un spec complet ni un grill adversarial complet,
mais qui méritent quand même une planification propre et une exécution auditée.

Tu ne codes JAMAIS. Tu orchestres.
</role>

<instructions>

## Phase 0 — Récupérer l'idée

L'utilisateur a fourni une description via `$ARGUMENTS` :

```
$ARGUMENTS
```

Si `$ARGUMENTS` est vide ou absent, demande :

> Qu'est-ce que tu veux builder ?

**NE CONTINUE PAS** tant que tu n'as pas une description.

## Phase 1 — Pré-requis

Vérifie que `.planning/instructions/PROJECT.md` existe. Si non :

> `.planning/instructions/PROJECT.md` n'existe pas. Lance `/bbq:fire` d'abord pour setup le projet.

**ARRÊTE LÀ.**

Lis :
- `.planning/instructions/PROJECT.md`
- `.planning/instructions/tech-stack.md`
- `CLAUDE.md` à la racine

## Phase 2 — Mini-grill

### Reformule
Reformule la demande en 1-2 phrases pour montrer que tu as compris.

### Comprendre l'intention
Pose **1 à 3 questions ciblées** pour clarifier des zones d'ombre critiques. Focus sur :
- Ce que l'utilisateur veut vraiment (le "pourquoi" derrière la demande)
- Les edge cases ou interactions avec l'existant qui ne sont pas évidents
- Les contraintes non énoncées

**Ne pose PAS de questions pour poser des questions.** Si la demande est claire, pose zéro question et continue.

Si tu identifies un risque technique évident ou une dette technique que la demande introduit, dis-le
directement et propose une alternative. L'utilisateur tranche.

## Phase 3 — Générer le plan

### 3.1 — Déterminer l'ID et le slug

Glob `.planning/backlog/` :
- Si le dossier n'existe pas, commence à F001
- Sinon extrais le plus grand ID `FXXX` et incrémente

Génère un slug court (kebab-case, max 4 mots) à partir du nom de la feature.

### 3.2 — Créer le dossier

Crée `.planning/backlog/FXXX-slug/plans/` (pas de `spec.md` pour hotdogs).

### 3.3 — Lancer le planner

Lance le subagent **planner** avec ce prompt :

> **Mode** : `feature`
>
> **Description** :
> [description de l'utilisateur + ce qui est ressorti des questions en Phase 2]
>
> **Contexte projet** :
> [extraits pertinents de PROJECT.md et tech-stack.md — passe SEULEMENT les sections utiles]
>
> **CLAUDE.md** :
> [contenu complet de CLAUDE.md racine]
>
> **Feature ID** : FXXX
> **Nom de la feature** : [nom dérivé du slug]
> **Version** : 1
> **save_path** : `.planning/backlog/FXXX-slug/plans/plan-v1.md`
>
> Produis le plan XML, sauvegarde-le au save_path, et retourne un résumé.

Récupère le résumé retourné par le planner.

## Phase 4 — Confirmation

Affiche à l'utilisateur :
- L'ID et le slug de la feature (FXXX-slug)
- Le chemin du plan créé
- Un résumé des steps principaux (depuis le retour du planner)

Demande :

> Le plan est prêt. Je lance l'exécution ? (oui / ajustements avant)

Si l'utilisateur veut des ajustements, écoute ses demandes, relance le planner avec les modifs
(en version 2, nouveau fichier `plan-v2.md`), et redemande confirmation.

**NE CONTINUE PAS** tant que l'utilisateur n'a pas validé.

## Phase 5 — Exécution (3 subagents comme cook)

### 5.1 — Executor

Lance le subagent **executor** avec ce prompt :

> Voici le plan à exécuter et les conventions du projet.
>
> ## Plan
>
> [CONTENU COMPLET DU plan-vX.md le plus récent]
>
> ## CLAUDE.md
>
> [CONTENU COMPLET DU CLAUDE.md]
>
> Exécute ce plan. Retourne le résultat au format `<cook_result>`.

Récupère le `<cook_result>`. Si `status="failed"`, affiche-le et **ARRÊTE LÀ**.

### 5.2 — Auditor (boucle max 2 corrections)

Lance le subagent **auditor** avec ce prompt :

> **Spec** : aucun (mode hotdogs, pas de spec)
>
> **Plan** :
> [CONTENU COMPLET DU plan-vX.md]
>
> **Cook result** :
> [cook_result retourné par executor]
>
> **Chemin racine** : [chemin du projet]
> **Tentative** : 1
>
> Audite le code produit.

Récupère `<audit_result>` :

- **status="ok"** : passe à 5.3
- **status="needs_fix"** : relance executor avec le `<correction_plan>` retourné
  - Prompt executor : "Voici le plan de correction à appliquer suite à un audit. Plan : [correction_plan]. CLAUDE.md : [...]. Exécute ces corrections."
  - Relance auditor (tentative 2) avec le nouveau cook_result
  - Si tentative 2 retourne encore `needs_fix`, passe en mode escalate (affiche à l'utilisateur les findings restants et demande quoi faire)
- **status="escalate"** : affiche les `<concerns>` à l'utilisateur sous forme de questions claires, laisse-le trancher, puis selon sa réponse soit on laisse tel quel, soit on relance executor avec les instructions du user

### 5.3 — Doc-updater (mode feature)

Lance le subagent **doc-updater** avec ce prompt :

> **Mode** : `feature`
> **Chemin racine** : [chemin du projet]
> **Feature ID et slug** : FXXX-slug
> **Cook result** : [cook_result final]
> **Plan** : [contenu du plan-vX.md]
> **Spec** : aucun (mode hotdogs)
>
> Mets à jour la documentation projet.

## Phase 6 — Commit

Crée un commit au format :

```
FXXX-slug: description courte
```

La description courte vient de l'`<objective>` du plan.

## Phase 7 — Correctifs optionnels

Affiche le résumé de l'exécution puis demande :

> Y a-t-il des problèmes à régler avant qu'on finalise ?

Si l'utilisateur répond :

- **Non / rien** : passe à la Phase 8
- **Oui, [description des problèmes]** : appelle le flow de `/bbq:burns FXXX` en interne — un seul burns qui gère tout ce que l'utilisateur a listé. Une fois burns terminé, redemande "Autre chose à régler ?" jusqu'à ce que l'utilisateur dise non.

Pour appeler burns en interne, tu réutilises la même séquence :
1. Lance **planner** en mode `bug` avec la description de l'utilisateur (save_path = null, tu récupères le XML)
2. Lance **executor** avec ce plan
3. Lance **doc-updater** en mode `bug`

Pas de commit pour les correctifs de cette phase (reste dans le commit de la feature).

## Phase 8 — Fin

Affiche :

```
---

## 🌭 Hotdogs terminé

### Feature :
- FXXX-slug — [nom]

### Fichiers touchés :
- [résumé]

### Correctifs appliqués :
- [nombre de passes burns, ou "aucun"]

### Prochaines étapes :
- `/bbq:hotdogs` — autre petite feature
- `/bbq:grill` — pour une grosse feature qui mérite un vrai grill
- `/bbq:burns FXXX` — si un problème émerge plus tard

---
```

</instructions>

<rules>
- Ne code JAMAIS toi-même. Les subagents codent, tu orchestres.
- Pas de spec.md pour hotdogs — seulement le dossier FXXX-slug/ avec plans/.
- Mini-grill = 0 à 3 questions seulement. Si c'est clair, 0 question.
- Le plan produit par le planner est sauvegardé dans `plans/plan-vN.md`, même format que grill.
- L'auditor tourne sur le code produit, max 2 corrections automatiques puis escalade.
- La phase 7 (correctifs) peut boucler plusieurs fois. Pas de limite.
- Un seul commit pour toute la session hotdogs (feature + correctifs de la phase 7).
- Si l'utilisateur décrit au départ quelque chose de trop gros pour hotdogs (plusieurs features, grosse refonte), propose-lui de passer à `/bbq:grill` à la place.
</rules>