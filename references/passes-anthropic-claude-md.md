# F1 : Anthropic CLAUDE.md (implémentation de la passe)

**Référence (le pourquoi) :** `references/anthropic-claude-md.md`.
**S'applique à :** chaque fichier classé `root-claude` ou `folder-claude` (et `claude-rules` là où c'est précisé).

## Fonctionnement de cette passe

Cette passe est **agentique, pas pilotée par regex**. Pour chaque vérification, le fichier liste :

- **Règle du framework** : ce qu'Anthropic affirme et pourquoi c'est important.
- **Heuristique de déclenchement** : un motif rapide qui fait remonter les correspondances candidates. Le déclencheur est un point de départ, pas un verdict. Certaines vérifications n'ont pas de déclencheur : le fichier lui-même est le candidat.
- **Jugement de l'agent** : ce que l'agent doit lire et analyser avant de produire un constat.
- **Faux positifs à ignorer** : cas courants qui ressemblent au déclencheur mais ne constituent pas réellement des violations.
- **Format de constat** : chaque constat inclut `reasoning` (1 à 2 phrases spécifiques à ce cas).

L'agent lit le fichier en entier, applique chaque vérification, et **ne produit que les constats confirmés par l'étape de raisonnement**. Si le déclencheur se déclenche 47 fois mais que seuls 5 cas violent réellement la règle en contexte, la passe produit 5 constats, pas 47.

## Sommaire

1. [F1.1 : Vérification de taille](#f11--vérification-de-taille)
2. [F1.2 : Heuristique de spécificité](#f12--heuristique-de-spécificité)
3. [F1.3 : Règles de style de code](#f13--règles-de-style-de-code)
4. [F1.4 : Descriptions fichier par fichier](#f14--descriptions-fichier-par-fichier)
5. [F1.5 : Platitudes évidentes](#f15--platitudes-évidentes)
6. [F1.6 : Ratio d'emphase générique](#f16--ratio-demphase-générique)
7. [F1.7 : Effet de position (amorce en tête de fichier)](#f17--effet-de-position-amorce-en-tête-de-fichier)
8. [F1.8 : Parcours de profondeur des `@import`](#f18--parcours-de-profondeur-des-import)
9. [F1.9 : Recommandation `.claude/rules/`](#f19--recommandation-clauderules)
10. [F1.10 : Le titre duplique le nom de fichier](#f110--le-titre-duplique-le-nom-de-fichier)
11. [F1.11 : Règles contradictoires](#f111--règles-contradictoires)
12. [F1.12 : Reformulations de conventions standard](#f112--reformulations-de-conventions-standard)
13. [Schéma de constat](#schéma-de-constat)

---

## F1.1 : Vérification de taille

**Règle du framework :** garder CLAUDE.md sous 200 lignes (plafond communautaire de 300). Au-delà d'environ 200, la ligne marginale retranche du signal.

**Heuristique de déclenchement :** `wc -l` et `stat`. >200 lignes = candidat. >300 lignes = candidat fort.

**Jugement de l'agent :**
- Lire le fichier. Déterminer si la taille est le problème, ou si le fichier est surtout une table de routage / des tableaux structurés (qui se compressent bien en attention).
- Un CLAUDE.md de 247 lignes composé à 80 % de règles à puces et de prose → vrai poids de spécificité, à signaler.
- Un CLAUDE.md de 247 lignes composé à 70 % de tables de routage → moins un problème d'attention ; signaler avec une sévérité moindre et recommander d'extraire seulement les sections de prose.
- Pour **folder-claude**, le budget est plus serré (les CLAUDE.md de dossier se superposent à la racine). Signaler à >120 lignes.

**Sévérité :**
- root : warn 200–300, fail >300
- folder : warn 120–200, fail >200

**Format de constat :**
```
{path} — {N} lines, {bytes}B (~{tokens}t)
Reasoning: {why this size is hurting THIS file's compliance — cite the structure (prose vs table) you observed}
Action: extract {specific section name} to references/, or split by domain
Citation: anthropic-claude-md.md → Hard rules (200-line target)
```

**Correction auto :** aucune.

---

## F1.2 : Heuristique de spécificité

**Règle du framework :** les règles spécifiques obtiennent environ 89 % de conformité ; les règles vagues environ 35 %. C'est le plus gros levier de qualité dans CLAUDE.md.

**Heuristique de déclenchement :** repérer les termes vagues (insensible à la casse) :
```
\b(properly|correctly|clean|good|appropriate|reasonable|sensible|nicely|carefully|thoughtfully|as needed|when appropriate|as you see fit)\b
```
Plus les expressions connues comme vagues : "be careful", "follow best practices", "write clean code", "test your changes", "format code properly", "keep things organized", "use good judgment".

**Jugement de l'agent :** pour chaque ligne candidate, déterminer s'il s'agit réellement d'une règle vague :
- La ligne est-elle une règle numérotée, une puce ou un impératif ? (Pas un titre, une citation ou une référence.)
- La règle inclut-elle un ancrage concret : chemin de fichier, nom de fonction, commande, seuil, système nommé ? Si oui → pas vague.
- La vague formulation est-elle délibérée ? Par exemple, les tâches créatives ("Write in a thoughtful tone") peuvent être irréductiblement fondées sur le jugement → signaler avec une sévérité moindre.
- La règle est-elle un rappel de clôture ("Be careful with auth") à l'intérieur d'un fichier par ailleurs spécifique ? Sévérité moindre : c'est un écho léger, pas l'instruction principale.
- Une règle comme "Be careful with payments — see `payments/README.md`" est **spécifique** parce qu'elle cite l'ancrage. Ne pas signaler.

**Faux positifs à ignorer :**
- Titres, citations, références à des docs externes.
- Règles qui citent déjà un fichier/fonction/seuil spécifique.
- Fichiers de voix de marque où la vague formulation reflète une intention subjective.
- Le mot "carefully" à l'intérieur d'un bloc de code ou d'un message utilisateur cité.

**Sévérité :** warn (un par règle réellement vague). Plafonner à 25 par fichier.

**Format de constat :**
```
{path}:{line} — vague rule
Excerpt: "{exact line}"
Reasoning: {why this rule is vague in THIS file — what context it's missing, what Claude couldn't determine from the rule alone}
Suggested rewrite: {a concrete version using anchors visible in the project (file path, function name, command)}
Citation: anthropic-claude-md.md → Specificity beats vagueness
```

**Correction auto :** aucune.

---

## F1.3 : Règles de style de code

**Règle du framework :** les règles de style de code ont leur place dans la config du linter + `.claude/rules/`, pas dans CLAUDE.md (les linters sont déterministes ; CLAUDE.md ne l'est pas).

**Heuristique de déclenchement :**
```
\b(use|prefer)\s+\d+[- ]?(space|tab)
(single|double)\s+quotes
\b(camelCase|snake_case|kebab-case|PascalCase)\b
\b(semicolon|trailing comma|line break)\b
\b(eslint|prettier|biome|ruff|black|rustfmt|gofmt)\b
```

**Jugement de l'agent :** lire la ligne candidate en contexte.
- Est-ce réellement une règle de style de code (formatage, nommage) ou une contrainte de fond (par exemple, "API responses must be JSON" : c'est un contrat, pas du style) ?
- Le projet a-t-il déjà une config de linter (`.eslintrc`, `.prettierrc`, `pyproject.toml` avec `[tool.ruff]`, etc.) ? Si oui → signaler, recommander la suppression. Si non → signaler, recommander de créer une config de linter + de déplacer la règle.
- La règle est-elle limitée à un chemin (par exemple, "TS files use 2 spaces, JSON uses 4") ? C'est un candidat pour `.claude/rules/{language}.md` → signaler avec cette recommandation.

**Faux positifs à ignorer :**
- Mentions de convention de nommage dans de la *prose* à propos d'un motif de la base de code ("the API uses snake_case keys" : cela documente un comportement externe, ça n'impose pas un style).
- Chaînes de directive de linter dans des blocs de code (`/* eslint-disable */`, etc.).

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — code-style rule
Excerpt: "{matched line}"
Reasoning: {why this is style-not-substance, and what the project's linter status is}
Action: {move to {tool} config OR create .claude/rules/{topic}.md with paths: scope}
Citation: anthropic-claude-md.md → Don'ts ("don't include code style rules")
```

**Correction auto :** aucune.

---

## F1.4 : Descriptions fichier par fichier

**Règle du framework :** ne pas inclure de descriptions fichier par fichier de la base de code : Claude lit le code.

**Heuristique de déclenchement :** détecter des séries de ≥6 items de liste ou lignes de tableau contigus où ≥4 items contiennent une référence de chemin/dossier suivie d'une prose descriptive.

**Jugement de l'agent :**
- Lire la section. Est-ce une table de routage (cellules courtes, ≤1 ligne chacune) ou une longue liste descriptive ?
- Une table de routage est correcte : c'est un index de navigation, pas de la documentation de fichiers. Ignorer.
- Une liste où chaque item fait ">40 chars of description per file" est l'anti-motif. Signaler.
- Le fichier décrit-il des surfaces d'API **externes** (par exemple, un SKILL de référence listant des endpoints) ? C'est de la documentation, pas une description fichier par fichier du projet. Ignorer.

**Faux positifs à ignorer :**
- Tables de routage (cellules courtes).
- Listes de référence d'API dans les skills.
- Listes où chaque item décrit un *concept* (pas un fichier).

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line-start}-{line-end} — file-by-file description block ({N} items)
Excerpt: first 2 items
Reasoning: {why these descriptions duplicate what reading the code would tell Claude}
Action: collapse to a {N}-line routing table, or delete entirely
```

**Correction auto :** aucune.

---

## F1.5 : Platitudes évidentes

**Règle du framework :** ne pas inclure de pratiques évidentes ("write clean code", "follow best practices", "be careful").

**Heuristique de déclenchement :** motifs sur ligne entière :
```
^\s*[-*0-9.]*\s*(write clean code|follow best practices|be careful|use common sense|do your best|test your changes|review your work|think before|stay focused|be thorough|be thoughtful|pay attention)\b
```

**Jugement de l'agent :**
- Pour chaque candidat, est-ce la règle entière, ou est-elle qualifiée par un ancrage ? "Be thorough — run all 4 tests in `tests/integration/`" → pas une platitude, ignorer.
- La platitude est-elle dans un encadré ou un résumé délibérément de haut niveau ? Sévérité moindre.
- Beaucoup de platitudes recoupent F1.2 ; dédupliquer : si les deux se déclenchent, préférer F1.5 car c'est un signal plus fort (supprimer vs réécrire).

**Faux positifs à ignorer :**
- Platitudes qui incluent un ancrage spécifique sur la même ligne.
- Citations de sources externes (citations de Karpathy, citations d'Anthropic).
- Platitudes à l'intérieur d'un bloc `<details>` conservé pour raisons historiques.

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — platitude
Excerpt: "{matched line}"
Reasoning: {why this rule has zero operational value in THIS file's context}
Action: delete (preferred) or replace with a concrete imperative tied to a project anchor
```

**Correction auto :** aucune.

---

## F1.6 : Ratio d'emphase générique

**Règle du framework :** "If everything is IMPORTANT, nothing is."

**Heuristique de déclenchement :** compter les marqueurs d'emphase autonomes :
```
\b(IMPORTANT|YOU MUST|CRITICAL|NEVER|ALWAYS|MUST|REQUIRED)\b
```
Puis compter le total des règles numérotées/à puces. Calculer le ratio.

**Jugement de l'agent :** lire le fichier.
- Un ratio élevé dans un fichier court et entièrement critique (politique de sécurité, garde-fou de déploiement) est *mérité* : chaque règle est réellement à ne pas rater. Sévérité moindre ou ignorer.
- Un ratio élevé dans un CLAUDE.md de 250 lignes est l'anti-motif → signaler.
- Pour chaque règle `IMPORTANT`, se demander : l'échec de Claude sur ce point provoquerait-il un véritable incident de production ? Si oui pour la plupart, l'emphase est méritée. Si non, l'emphase est de l'inflation.

**Faux positifs à ignorer :**
- Fichiers où chaque règle est véritablement critique (sécurité, paiements, opérations irréversibles).
- Emphase à l'intérieur d'encadrés (`> [!important]`) : l'encadré effectue déjà le marquage.

**Sévérité :**
- ratio > 0.5 → fail
- ratio > 0.3 → warn

**Format de constat :**
```
{path} — {n} emphasis markers across {m} rules ({pct}%)
Reasoning: {why the emphasis is inflated here — sample 2-3 rules that are NOT mission-critical but use IMPORTANT}
Action: reserve emphasis for rules where failure is genuinely costly; demote the rest
```

**Correction auto :** aucune.

---

## F1.7 : Effet de position (amorce en tête de fichier)

**Règle du framework :** les LLM sur-attendent au début et à la fin. Commencer par la règle porteuse.

**Heuristique de déclenchement :** lire les 30 premières lignes (ou les 30 % premiers, selon ce qui est le plus grand). Évaluer la présence de "load-bearing".

**Jugement de l'agent :** l'agent lit le haut du fichier et se demande :
- Si Claude n'avait que les 30 premières lignes, connaîtrait-il les choses les plus importantes que ce fichier cherche à imposer ?
- Le haut est-il une table de routage, une liste numérotée de règles, ou un encadré ? → bonne amorce.
- Le haut est-il un long bloc de prose descriptive, une introduction ("This document explains…"), ou un préambule de personnalité ? → amorce enfouie.
- Parfois le préambule est nécessaire (par exemple, le SKILL.md d'un skill a besoin de la description et du frontmatter). Juger ce qui est porteur **pour le rôle de ce fichier**.

**Faux positifs à ignorer :**
- Fichiers où le préambule est imposé (blocs de frontmatter, en-têtes de licence).
- Fichiers où le H1 + un unique résumé d'une ligne en tête EST l'affirmation porteuse.

**Sévérité :** warn.

**Format de constat :**
```
{path} — top-of-file is preamble, not load-bearing rules
Excerpt: "{first 100 chars}…"
Reasoning: {what's actually at the top vs what should be there given the file's role}
Action: lead with {specific section name observed deeper in the file} that's actually load-bearing
```

**Correction auto :** aucune.

---

## F1.8 : Parcours de profondeur des `@import`

**Règle du framework :** imports limités à 5 sauts ; au-delà ils sont abandonnés. Les imports n'économisent PAS de tokens.

**Heuristique de déclenchement :** grep `^@(\S+)`. Résoudre chacun, récurser, suivre la profondeur.

**Jugement de l'agent :** essentiellement mécanique : le compte de profondeur est la règle. Mais l'agent lit le fichier pour :
- Confirmer que la ligne d'import n'est pas à l'intérieur d'une clôture de code (exemples de code à propos des imports).
- Décider de signaler un fichier à profondeur 5 en warn ou en fail, sachant qu'un saut de plus est un abandon définitif.
- Détecter les cycles en suivant les chemins visités.

**Faux positifs à ignorer :**
- Chaînes préfixées par `@` à l'intérieur de clôtures de code ou de code inline.
- Tags Git / noms scopés npm qui commencent par hasard par `@` mais ne sont pas des imports markdown.

**Sévérité :**
- depth > 5 → fail (will be dropped)
- depth = 5 → warn (at ceiling)
- cycle detected → fail
- import target missing → fail

**Format de constat :**
```
{path}:{line} — @import {target}
Reasoning: depth {n} from root via {path-1} → {path-2} → … (or "cycle: {path} re-imports itself")
Action: flatten the chain or remove the deepest import; imports don't save tokens
```

**Correction auto :** aucune.

---

## F1.9 : Recommandation `.claude/rules/`

**Règle du framework :** les règles limitées à un chemin ont leur place dans `.claude/rules/*.md` avec un frontmatter `paths:`.

**Heuristique de déclenchement :**
1. F1.3 s'est-il déclenché ≥3 fois dans l'arborescence CLAUDE.md ?
2. `.claude/rules/` existe-t-il ? Si oui, ses fichiers ont-ils un frontmatter `paths:` ?

**Jugement de l'agent :**
- Si F1.3 s'est déclenché et que `.claude/rules/` est absent → recommandation forte. Le raisonnement liste les règles de style spécifiques qui devraient y être déplacées.
- Si `.claude/rules/` existe avec des fichiers dépourvus de `paths:` → ces fichiers se chargent inconditionnellement et agissent comme CLAUDE.md. Signaler chacun avec un raisonnement expliquant la perte de divulgation progressive.
- Si `.claude/rules/` existe avec un frontmatter `paths:` correct → aucun constat.

**Faux positifs à ignorer :**
- Vaults sans règles de style de code dans CLAUDE.md → aucune recommandation nécessaire.

**Sévérité :** warn.

**Format de constat (pas de dossier rules) :**
```
.claude/rules/ does not exist
Reasoning: {n} code-style rules found in CLAUDE.md ({list 2-3 with line numbers}) — they should be path-scoped
Action: create .claude/rules/{topic}.md per topic with paths: frontmatter; move the listed rules there
```

**Format de constat (fichier de règle sans paths) :**
```
.claude/rules/{file} — missing paths: frontmatter
Reasoning: this file currently loads unconditionally on every session, defeating the path-scoping benefit
Action: add a paths: array so the rule only loads when matching files are read
```

**Correction auto :** aucune.

---

## F1.10 : Le titre duplique le nom de fichier

**Règle du framework :** ne pas mettre un titre `# Title` qui duplique le nom de fichier.

**Heuristique de déclenchement :** lire la première ligne de contenu hors frontmatter. Si c'est `# {Title}`, slugifier à la fois `{Title}` et le nom de fichier ; s'ils correspondent → candidat.

**Jugement de l'agent :** essentiellement mécanique, mais l'agent lit pour confirmer :
- Le H1 est-il réellement un titre dupliquant le nom de fichier, ou un titre de section (par exemple, `# Overview`) qui se trouve simplement coïncider ?
- Pour les fichiers `CLAUDE.md`, le H1 est généralement `# CLAUDE.md` littéralement : c'est le doublon à supprimer.
- Pour `README.md`, un H1 de `# Project Name` peut être un affichage GitHub intentionnel → signaler avec une sévérité moindre.

**Faux positifs à ignorer :**
- READMEs à la racine du dépôt où le H1 s'affiche dans l'UI GitHub (cas de jugement → signaler, mais recommander de garder si l'utilisateur préfère le rendu GitHub).

**Sévérité :** warn.

**Format de constat :**
```
{path} — H1 "{title}" duplicates filename
Reasoning: Obsidian/Claude already shows the filename as the title; the H1 is redundant
Action: remove the H1 line and any blank line below
```

**Correction auto :** **corrigible** (dédupliqué avec G7.3 : s'exécute une fois par fichier).

---

## F1.11 : Règles contradictoires

**Règle du framework :** deux règles qui se contredisent → Claude en choisit une arbitrairement. Résoudre les conflits.

**Heuristique de déclenchement :** pour chaque ligne contenant `Always|Never|Must|Don't|Do not`, parcourir le reste du fichier à la recherche d'une autre ligne au modal opposé qui partage ≥3 tokens de contenu (après suppression des mots vides).

**Jugement de l'agent :** lire les deux lignes candidates.
- Se contredisent-elles réellement, ou sont-elles de portées différentes ? "Always commit before pushing" + "Never commit on main" : portées différentes, pas de conflit. Ignorer.
- Sont-elles séquencées ("first do X; never do X without Y") ? → pas un conflit.
- S'opposent-elles véritablement ? → signaler, avec un raisonnement montrant la contradiction.

**Faux positifs à ignorer :**
- Règles de portées différentes.
- Règles où l'une est un défaut et l'autre une exception ("Always run tests; never run tests on the deploy branch").

**Sévérité :** warn.

**Format de constat :**
```
{path} — rules at L{a} and L{b} appear to conflict
Excerpt:
  L{a}: "{text-a}"
  L{b}: "{text-b}"
Reasoning: {why these contradict — what action a confused Claude would take based on which rule it gives weight to}
Action: resolve which rule wins; remove or qualify the other
```

**Correction auto :** aucune.

---

## F1.12 : Reformulations de conventions standard

**Règle du framework :** ne pas reformuler les conventions de langage que Claude connaît déjà.

**Heuristique de déclenchement :**
```
\b(Use|Prefer)\s+(camelCase|snake_case|kebab-case|PascalCase)\b
\b(JS|JavaScript|Python|Go|Rust|Ruby)\s+uses\s+\w+
\bAlways use semicolons in (JS|JavaScript)\b
\bIndent with (2|4) spaces in (JS|JavaScript|Python|TypeScript)\b
```

**Jugement de l'agent :** lire la ligne.
- Reformule-t-elle une convention par défaut que le langage impose déjà ? → signaler.
- Surcharge-t-elle une convention (par exemple, "We use snake_case in JS for legacy reasons") ? → pas une reformulation ; règle de projet valide. Ignorer.
- Est-ce de la documentation sur les conventions d'une API externe ? → ignorer.

**Faux positifs à ignorer :**
- Surcharges spécifiques au projet.
- Documentation de systèmes externes.

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — restates a standard language convention
Excerpt: "{matched line}"
Reasoning: {language} already enforces this; Claude knows it without being told
Action: delete
```

**Correction auto :** aucune.

---

## Schéma de constat

```json
{
  "framework": "F1",
  "check_id": "F1.x",
  "check_name": "name",
  "path": "./CLAUDE.md",
  "line": 42,
  "severity": "fail|warn",
  "excerpt": "matched line or surrounding context",
  "reasoning": "1-2 sentences specific to this case",
  "action": "remediation",
  "fixable": false,
  "citation": "anthropic-claude-md.md → section"
}
```

Le champ `reasoning` est obligatoire : chaque constat le possède. Un constat sans raisonnement est un bug, pas un constat.
