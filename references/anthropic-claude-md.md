# Anthropic : bonnes pratiques CLAUDE.md

## Sommaire

1. [Thèse centrale](#thèse-centrale)
2. [Règles strictes](#règles-strictes)
3. [Le test d'élagage](#le-test-délagage)
4. [Hiérarchie et précédence des CLAUDE.md](#hiérarchie-et-précédence-des-claudemd)
5. [Les trois blocs de construction essentiels](#les-trois-blocs-de-construction-essentiels-dont-chaque-claudemd-a-besoin)
6. [La précision l'emporte sur le flou : exemples](#la-précision-lemporte-sur-le-flou--exemples)
7. [Règles à portée de chemin dans `.claude/rules/`](#règles-à-portée-de-chemin-dans-clauderules)
8. [Le motif `@import`](#le-motif-import)
9. [`claudeMdExcludes` et la politique gérée](#claudemdexcludes-et-la-politique-gérée)
10. [Effet de position : pourquoi l'ordre compte](#effet-de-position--pourquoi-lordre-compte)
11. [Marqueurs d'emphase](#marqueurs-demphase)
12. [Le système `MEMORY.md`](#le-système-memorymd)
13. [Comportement de compaction](#comportement-de-compaction)
14. [Les commentaires HTML sont retirés](#les-commentaires-html-sont-retirés)
15. [À faire](#à-faire)
16. [À ne pas faire](#à-ne-pas-faire)
17. [Citations textuelles à préserver](#citations-textuelles-à-préserver)
18. [Signaux auditables](#signaux-auditables)
19. [Sources](#sources)

---

## Thèse centrale

CLAUDE.md est chargé comme **message utilisateur au début de chaque session**, et non comme partie du prompt système. Claude le lit et tente de le suivre, mais il n'y a **aucune garantie de conformité stricte**, en particulier pour les instructions vagues ou contradictoires. Le bon CLAUDE.md est le plus petit ensemble concret d'instructions qui survit au test d'élagage.

La façon dont Anthropic formule l'ingénierie de contexte est sans détour : *"trouver le plus petit ensemble possible de tokens à fort signal qui maximise la probabilité d'un résultat souhaité."* CLAUDE.md, c'est le même problème appliqué à la couche d'instructions de votre projet.

## Règles strictes

| # | Règle | Auditable |
|---|---|---|
| 1 | Maintenir CLAUDE.md sous les **200 lignes** (plafond communautaire : 300) | ✅ Pass 1 |
| 2 | Les règles spécifiques obtiennent **89 % de conformité** ; les règles vagues obtiennent **35 %** | ✅ Pass 2 (heuristique de précision) |
| 3 | Un CLAUDE.md surchargé → Claude **ignore les instructions entièrement** (pas seulement celles en excès) | ✅ Pass 1 (taille) |
| 4 | Les LLM préfèrent les instructions au **début et à la fin** d'un prompt ; le milieu est négligé | ✅ Pass 9 (heuristique de position) |
| 5 | Les modèles suivent raisonnablement ~150 à 200 instructions (Opus 4.5) ; moins pour les modèles plus petits. Le prompt système de Claude Code à lui seul utilise ~50 instructions | ⚠️ Information seule |
| 6 | Les imports via `@path/to/import` se résolvent relativement au fichier importateur ; **5 sauts maximum** | ✅ Pass 9 |
| 7 | Les imports n'économisent **pas** de tokens, ils ne font qu'organiser. Le contenu importé se charge entièrement au lancement | ⚠️ Information seule |
| 8 | `MEMORY.md` charge les **200 premières lignes ou 25 Ko**, selon ce qui survient en premier | ⚠️ Information seule |
| 9 | Les commentaires HTML de niveau bloc `<!-- ... -->` sont retirés avant l'injection dans le contexte | — |
| 10 | Le CLAUDE.md à la racine du projet est **réinjecté après `/compact`**. Les CLAUDE.md imbriqués ne sont **pas** réinjectés | — |
| 11 | Deux règles contradictoires → Claude peut en choisir une **arbitrairement**. Résolvez les conflits ; ne supposez pas que la précision l'emporte | ⚠️ Pass 6 (duplication) |
| 12 | Les règles à portée de chemin dans `.claude/rules/` ne se chargent que lorsque Claude lit des fichiers correspondants (via le frontmatter `paths:`) | ✅ Pass 9 |
| 13 | Le CLAUDE.md d'un sous-dossier n'est **pas auto-chargé au lancement**, seulement quand Claude lit des fichiers dans ce dossier | ⚠️ Information seule |

## Le test d'élagage

Pour chaque ligne de CLAUDE.md, demandez-vous :

> **"Le retrait de cette ligne pousserait-il Claude à faire des erreurs ? Si non, supprimez-la."**

Appliquez ce test sans pitié. L'erreur que commettent la plupart des équipes est l'inverse : elles continuent d'ajouter des lignes en espérant que davantage de directives aide. Au-delà de ~200 lignes, la ligne marginale **soustrait** du signal, car l'adhésion de Claude au fichier entier se dégrade.

> *"Si Claude continue de faire quelque chose que vous ne voulez pas malgré une règle qui l'interdit, le fichier est probablement trop long et la règle se perd."*

## Hiérarchie et précédence des CLAUDE.md

Tous les fichiers découverts se **concatènent** (ils ne s'écrasent pas). Les portées plus spécifiques l'emportent en cas de conflit. Les fichiers remontent l'arborescence des dossiers à partir du répertoire de travail.

| Portée | Chemin (par défaut) | Partagé avec | Notes |
|---|---|---|---|
| Politique gérée | `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS) | Toute l'organisation | **Ne peut pas être exclu** même via `claudeMdExcludes` |
| Politique gérée | `/etc/claude-code/CLAUDE.md` (Linux/WSL) | Toute l'organisation | Idem |
| Politique gérée | `C:\Program Files\ClaudeCode\CLAUDE.md` (Windows) | Toute l'organisation | Idem |
| Projet | `./CLAUDE.md` ou `./.claude/CLAUDE.md` | Équipe via git | Le fichier principal que maintiennent la plupart des équipes |
| Utilisateur | `~/.claude/CLAUDE.md` | Vous seul, tous projets | Préférences personnelles (style de réponse, langue) |
| Local | `./CLAUDE.local.md` (gitignored) | Vous seul, ce projet | URLs de bac à sable, identifiants personnels, particularités locales à la machine |

### Comportement en monorepo

Si vous exécutez Claude Code dans `root/foo/` :
- `root/foo/CLAUDE.md` est chargé (cwd)
- `root/CLAUDE.md` est également chargé (remontée vers le parent)
- Si Claude lit des fichiers dans `root/foo/bar/`, alors `root/foo/bar/CLAUDE.md` se charge à la demande

Cela permet à la racine du monorepo d'appliquer des règles à tous les sous-projets, tandis que chaque sous-projet ajoute les siennes.

## Les trois blocs de construction essentiels dont chaque CLAUDE.md a besoin

Source : camelCase, "Stop Writing Bad CLAUDE.md Files." Appuyé par les recommandations d'Anthropic sur la précision.

### 1. Une phrase décrivant le projet

Une seule phrase qui donne à Claude le framework, le langage, le public et le domaine.

✅ Bon :
- *"Ceci est notre portail de paiement orienté client, construit avec Next.js 15 et Stripe."*
- *"Ceci est un site de documentation construit avec Astro pour une CLI open source."*
- *"Ceci est un site portfolio basé sur Angular, sans backend."*

❌ Mauvais :
- *"Ceci est un projet."*
- (Aucune description du tout)

### 2. Les commandes bash clés

Les commandes que Claude ne peut pas deviner et qui servent dans le travail **quotidien**.

✅ Bon :
- `npm run build` : construit le bundle de production
- `npm run typecheck` : exécute les vérifications TypeScript
- `pytest -k "not integration"` : exécute les tests unitaires en ignorant l'intégration

❌ Mauvais :
- Lister chaque script npm, même ceux utilisés une fois par mois
- Lister des commandes standard que Claude connaît (`git log`, `cd`)

### 3. Mises en garde : avertissements non évidents du projet

Des éléments invisibles depuis le code mais qui feront trébucher Claude.

✅ Bon :
- *"Ne modifiez jamais `schema.prisma` directement, exécutez plutôt `npm run db:generate`."*
- *"Le webhook de l'API attend un corps brut, n'utilisez pas le middleware `body-parser`."*
- *"Les images dans `/public` doivent être optimisées, tout fichier au-delà de 200 Ko fait échouer la CI."*
- *"Le dossier `legacy/` est en lecture seule et programmé pour suppression. N'y ajoutez pas de nouveau code."*

❌ Mauvais :
- Avertissements génériques ("attention à la base de données")
- Avertissements que le linter détecterait de toute façon

## La précision l'emporte sur le flou : exemples

Les règles spécifiques obtiennent ~89 % de conformité. Les règles vagues chutent à ~35 %. Exemples documentés par Anthropic :

| ❌ Vague (faible conformité) | ✅ Spécifique (forte conformité) |
|---|---|
| "Formater le code correctement" | "Utiliser une indentation de 2 espaces" |
| "Tester vos changements" | "Exécuter `npm test` avant de committer" |
| "Garder les fichiers organisés" | "Les handlers d'API résident dans `src/api/handlers/`" |
| "Écrire du code propre" | (supprimez simplement ceci) |
| "Faire attention à l'authentification" | "Toutes les routes `/api/admin/*` doivent appeler `requireAdmin()` depuis `src/auth/middleware.ts`" |
| "Documenter le code complexe" | "Les fonctions avec >3 paramètres nécessitent un bloc JSDoc listant chaque paramètre" |

## Règles à portée de chemin dans `.claude/rules/`

Pour les projets plus grands, découpez les instructions en fichiers spécifiques à un sujet dans `.claude/rules/`. Chaque fichier `.md` couvre un sujet. Les règles sont limitées à des chemins de fichiers via le frontmatter YAML :

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "src/handlers/**/*.ts"
---

# API Handler Rules

- Always wrap handlers in `withAuth()`.
- Return errors as `{ error: { code, message } }`.
- Use the shared logger from `src/lib/logger.ts`.
```

Les fichiers **sans** frontmatter `paths:` se chargent inconditionnellement au lancement (ils agissent comme un CLAUDE.md). Les fichiers **avec** `paths:` ne se chargent que lorsque Claude lit des fichiers correspondant au glob : une véritable divulgation progressive.

C'est le bon endroit pour les règles de style de code, les conventions de test, les patterns de sécurité. **Sortez-les de CLAUDE.md.**

## Le motif `@import`

Syntaxe : `@path/to/file.md` à l'intérieur d'un CLAUDE.md.

- Les chemins relatifs se résolvent **relativement au fichier importateur**.
- Profondeur de récursion maximale : **5 sauts**. Au-delà, l'import est abandonné.
- Le contenu importé **se charge entièrement** au lancement : les imports n'économisent pas de tokens, ils ne font qu'organiser.

Cas d'usage : règles partagées entre plusieurs dépôts.

```markdown
# Project CLAUDE.md

@~/.claude/shared-rules/typescript.md
@~/.claude/shared-rules/security.md

## Project-specific
- ...
```

## `claudeMdExcludes` et la politique gérée

Dans `.claude/settings.local.json`, vous pouvez ignorer les fichiers CLAUDE.md ancêtres par glob :

```json
{
  "claudeMdExcludes": [
    "../../legacy-monorepo/**"
  ]
}
```

Utile lorsque vous êtes à l'intérieur d'un gigantesque monorepo d'organisation et que la plupart du contenu CLAUDE.md ancêtre est du bruit non pertinent.

**Le CLAUDE.md de politique gérée ne peut pas être exclu.** Les règles à l'échelle de l'organisation s'appliquent toujours. C'est intentionnel : les règles de sécurité ou de conformité ne peuvent pas être désactivées par un développeur.

## Effet de position : pourquoi l'ordre compte

Les LLM sur-pondèrent **le début et la fin** d'un prompt et sous-pondèrent **le milieu**. C'est une propriété structurelle de l'attention des transformeurs, documentée dans de multiples études.

Pour CLAUDE.md :
- Placez les **règles les plus importantes en haut**.
- Placez les **blocs de construction** (description du projet, commandes, mises en garde) tôt.
- Placez les **cas limites et les directives de moindre priorité** au milieu (ou déplacez-les vers `.claude/rules/`).
- La toute fin est aussi une zone de forte attention : utilisez-la pour les rappels finaux `IMPORTANT`.

Le propre rappel système d'Anthropic est révélateur :

> *"important: This context may or may not be relevant to your tasks. You should not respond to this context unless it is highly relevant to your task."*

On *dit* au modèle que le contenu de CLAUDE.md peut ne pas être pertinent. Les règles vagues ou enfouies au milieu sont traitées comme du bruit de fond. **La précision et la position surmontent cela.**

## Marqueurs d'emphase

Anthropic confirme que les marqueurs d'emphase améliorent l'adhésion pour les règles critiques :

- `IMPORTANT:`
- `YOU MUST`
- `CRITICAL:`
- LES MAJUSCULES pour la règle elle-même
- Les points d'exclamation (avec parcimonie)

Réservez-les aux règles dont l'échec coûte véritablement cher. Si tout est `IMPORTANT`, rien ne l'est.

## Le système `MEMORY.md`

Mémoire automatique sous `~/.claude/projects/<project>/memory/` :

- `MEMORY.md` est l'index. **Les 200 premières lignes ou 25 Ko**, selon ce qui survient en premier, se chargent au démarrage.
- Les fichiers de sujets dans le même dossier se chargent à la demande.
- Claude décide de ce qui vaut la peine d'être retenu en fonction de l'utilité future.
- La mémoire est locale à la machine et partagée entre les worktrees d'un même dépôt git.

Traitez `MEMORY.md` comme CLAUDE.md : élaguez agressivement, gardez des entrées spécifiques.

## Comportement de compaction

Lorsque `/compact` s'exécute :

- Le **CLAUDE.md à la racine du projet est réinjecté** automatiquement. Vos règles survivent.
- **Les CLAUDE.md imbriqués ne sont pas réinjectés.** Si les règles d'un sous-dossier importaient pour la conversation, elles disparaissent après la compaction.
- Personnalisez la compaction en ajoutant à CLAUDE.md : *"Lors de la compaction, préservez toujours la liste complète des fichiers modifiés et toutes les commandes de test."*

## Les commentaires HTML sont retirés

Les commentaires HTML de niveau bloc sont supprimés de l'injection dans le contexte :

```markdown
<!-- This note is for the human maintainer only.
     Claude never sees it. -->
```

Utilisez ceci pour les notes du mainteneur, la piste d'audit, l'historique du "pourquoi cette règle existe", sans dépenser de tokens. Les commentaires en bloc de code (à l'intérieur des barrières ` ``` `) sont **préservés**.

## À faire

- N'exécutez `/init` que comme point de départ, puis élaguez agressivement. Un CLAUDE.md généré par IA est verbeux par défaut.
- Traitez CLAUDE.md comme du code : révisez-le quand les choses tournent mal, élaguez régulièrement, testez les changements en observant le comportement.
- Utilisez les titres et les listes à puces markdown pour regrouper les instructions liées et améliorer la lisibilité.
- Utilisez `IMPORTANT` / `YOU MUST` pour les règles dont l'échec coûte cher.
- Déplacez les procédures multi-étapes vers des Skills.
- Déplacez les règles spécifiques à un chemin vers `.claude/rules/` avec un frontmatter `paths:`.
- Ajoutez à CLAUDE.md quand Claude commet la même erreur **deux fois**.
- Placez les règles les plus importantes en **haut** du fichier.
- Programmez des audits mensuels : CLAUDE.md est un document vivant, le code dérive, les règles deviennent obsolètes.
- Utilisez `<!-- HTML comments -->` pour les notes du mainteneur qui ne dépensent pas de tokens.
- Utilisez l'intégration GitHub d'Anthropic pour demander à Claude de mettre à jour CLAUDE.md à partir des observations de PR.

## À ne pas faire

- N'incluez pas de règles de style de code (`use 2 spaces`, `single quotes`). Utilisez des linters et des formateurs avec le hook `post tool use`.
- N'incluez rien que Claude puisse déduire en lisant le code.
- N'incluez pas de conventions de langage standard que Claude connaît déjà.
- N'incluez pas de documentation d'API détaillée : créez plutôt un lien vers la documentation.
- N'incluez pas d'informations qui changent fréquemment (URLs de déploiement, sprint en cours, qui est d'astreinte).
- N'incluez pas de longues explications ou de tutoriels.
- N'incluez pas de descriptions fichier par fichier de la base de code.
- N'incluez pas de pratiques évidentes ("écrire du code propre", "suivre les bonnes pratiques", "faire attention").
- Ne mettez pas un titre `# Title` qui duplique le nom du fichier.
- Ne comptez pas sur les `@imports` pour "économiser des tokens" : ils ne le font pas.
- Ne laissez pas CLAUDE.md dériver au-delà de 200 lignes sans le découper en Skills, `.claude/rules/`, ou imports.
- Ne mettez pas d'avertissements génériques : soyez précis sur ce qui échoue et comment.
- N'écrivez pas de règles qui se contredisent ; résolvez-les ou l'une sera choisie arbitrairement.

## Citations textuelles à préserver

> *"CLAUDE.md content is delivered as a user message after the system prompt, not as part of the system prompt itself. Claude reads it and tries to follow it, but there's no guarantee of strict compliance, especially for vague or conflicting instructions."*

> *"If Claude keeps doing something you don't want despite having a rule against it, the file is probably too long and the rule is getting lost."*

> *"Treat CLAUDE.md like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts."*

> *"Treating context as a precious, finite resource will remain central to building reliable, effective agents."*

## Signaux auditables

Lorsque ce skill exécute la Pass 1 (vérification de taille) et la Pass 2 (élagage) pour les fichiers CLAUDE.md, cherchez :

- **Nombre total de lignes** par CLAUDE.md (>200 = warn, >300 = fail).
- **Règles vagues** : règles qui n'incluent pas de valeur concrète, de commande, de chemin de fichier, ou de seuil quantifiable. Heuristique : la règle contient `properly`, `correctly`, `clean`, `good`, `appropriate` → à signaler pour revue.
- **Règles de style de code** : détecter `\b(use|prefer)\s+\d+\s+(space|tab)`, les règles de guillemets simples, les opinions de formatage. Suggérer de les déplacer vers la configuration du linter + `.claude/rules/`.
- **Descriptions fichier par fichier** : longues sections décrivant chaque fichier d'un dossier. Suggérer de les convertir en une courte table de routage ou de les supprimer.
- **Reformulations de conventions standard** : règles qui ne font que répéter ce que le langage impose (`Use camelCase for JS`, `Use snake_case for Python`).
- **Platitudes évidentes** : `write clean code`, `follow best practices`, `be careful`, `test your changes`. À couper.
- **Emphase générique** : chaque règle marquée `IMPORTANT`. Suggère une absence de priorisation réelle. À signaler.
- **Contenu en haut du fichier** : les 20 premières lignes devraient contenir la description du projet, les commandes clés, ou les mises en garde critiques. S'il s'agit de préambule ou de justification, suggérer de remonter le contenu utile.
- **Imports** : détecter les `@imports`. Parcourir les imports jusqu'à 5 sauts. Signaler si la profondeur est >5 (l'import sera abandonné).
- **Duplication de titre** : un `# H1` qui correspond au nom de fichier → à couper.
- **Contenu importé jamais référencé** par le reste de CLAUDE.md → suggérer de le supprimer.

## Sources

- https://code.claude.com/docs/en/best-practices
- https://code.claude.com/docs/en/memory
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- camelCase, "Stop Writing Bad CLAUDE.md Files" (2026-02-04) : notes de terrain de praticien
