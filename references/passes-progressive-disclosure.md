# F6 : Progressive Disclosure (implémentation de la passe)

**Référence (le pourquoi) :** `references/progressive-disclosure.md`.
**S'applique à :** chaque `SKILL.md`. F6.4–F6.5 parcourent le dossier `references/` du skill.

## Fonctionnement de cette passe

Agentique. La validation du frontmatter est essentiellement mécanique, mais les jugements de qualité du contenu (déclencheurs de description, conception des références, cohérence terminologique, constantes vaudou) exigent que l'agent lise le SKILL.md et ses références, et raisonne pour déterminer si la règle du framework est réellement enfreinte. Voir l'introduction de F1 pour le pattern complet.

## Sommaire

1. [F6.1 : Taille du SKILL.md](#f61--taille-du-skillmd)
2. [F6.2 : Frontmatter `name`](#f62--frontmatter-name)
3. [F6.3 : Frontmatter `description`](#f63--frontmatter-description)
4. [F6.4 : Profondeur des références](#f64--profondeur-des-références)
5. [F6.5 : TOC des références](#f65--toc-des-références)
6. [F6.6 : Chemins de style Windows](#f66--chemins-de-style-windows)
7. [F6.7 : Langage sensible au temps](#f67--langage-sensible-au-temps)
8. [F6.8 : Constantes vaudou](#f68--constantes-vaudou)
9. [F6.9 : Terminologie incohérente](#f69--terminologie-incohérente)
10. [F6.10 : Préfixe MCP `ServerName:`](#f610--préfixe-mcp-servername)
11. [F6.11 : Duplication skill-vault](#f611--duplication-skill-vault)
12. [F6.12 : Description à la première/deuxième personne](#f612--description-à-la-premièredeuxième-personne)
13. [Schéma de constat](#schéma-de-constat)

---

## F6.1 : Taille du SKILL.md

**Règle du framework :** corps du SKILL.md sous les 500 lignes.

**Heuristique de déclenchement :** `wc -l` sur le corps (après le frontmatter).

**Jugement de l'agent :** pour chaque SKILL.md surdimensionné :
- Le lire. Identifier les sections riches en détails qui pourraient migrer vers `references/`.
- Le raisonnement nomme des sections spécifiques et un découpage réaliste.

**Sévérité :** warn 400–500, fail >500.

**Format de constat :**
```
{path} — {N} lines (target <500)
Reasoning: sections that look extractable: {names with line counts}
Action: extract {specific section} → references/{topic}.md; replace with a 1-line reference
```

**Correction automatique :** aucune.

---

## F6.2 : Frontmatter `name`

**Règle du framework :** ≤64 caractères, `[a-z0-9-]` uniquement, aucun mot réservé (`anthropic`, `claude`), aucune balise XML.

**Heuristique de déclenchement :** parser le `name` du frontmatter. Appliquer les contraintes ci-dessus.

**Jugement de l'agent :** essentiellement mécanique. L'agent lit pour suggérer un meilleur nom lorsque le nom actuel enfreint la règle :
- Trop long → suggérer un slug plus serré en forme de gérondif.
- Mauvais caractères → proposer la version nettoyée.
- Mot réservé → proposer une alternative qui capture la même intention.

**Sévérité :** fail.

**Format de constat :**
```
{path} — frontmatter name "{value}" {issue}
Reasoning: {what the issue is and why it breaks skill loading}
Suggested name: {alternative}
Action: rename
```

**Correction automatique :** aucune.

---

## F6.3 : Frontmatter `description`

**Règle du framework :** ≤1024 caractères, non vide, troisième personne, inclut les déclencheurs, aucune balise XML.

**Heuristique de déclenchement :** parser la `description` du frontmatter. Vérifier la longueur, la présence, la personne, les mots-clés déclencheurs.

**Jugement de l'agent :** lire la description et le corps du SKILL.md :
- Pour les dépassements de longueur → le raisonnement identifie le surplus (scalaire replié gaspillant de l'indentation, formulation hésitante, répétition redondante) ; suggère une version resserrée.
- Pour les déclencheurs manquants → le raisonnement liste 3 à 5 formulations que l'utilisateur emploierait réellement pour invoquer ce skill (tirées de la lecture du corps du SKILL.md).
- Pour la première/deuxième personne → suggérer la réécriture à la troisième personne.
- Pour les descriptions trop vagues → signaler même si la longueur est correcte ; le raisonnement explique ce que fait le skill que la description ne capture pas.

**Faux positifs à ignorer :**
- Les descriptions qui incluent de vrais noms de produits ressemblant à des mots réservés (ex. : « Claude API » : c'est le produit, pas une violation de mot réservé).

**Sévérité :**
- longueur > 1024, manquante, vide, balises XML → fail
- première/deuxième personne, déclencheurs manquants → warn

**Format de constat :**
```
{path} — description {issue}
Reasoning: {specific to the description content}
Suggested rewrite: {tightened/third-person/triggers-added version}
Action: replace
Citation: progressive-disclosure.md → Description authoring
```

**Correction automatique :** aucune.

---

## F6.4 : Profondeur des références

**Règle du framework :** garder les références à un seul niveau de profondeur depuis le SKILL.md.

**Heuristique de déclenchement :** parser le SKILL.md pour les liens markdown → marquer le saut 1. Récursion sur le saut 2.

**Jugement de l'agent :** pour chaque référence à 2 sauts :
- Lire la chaîne. Le fichier le plus profond est-il un détail réellement secondaire (bon cas pour une fusion dans le saut 1) ou une préoccupation distincte (bon cas pour un lien direct depuis le SKILL.md) ?
- Le raisonnement recommande la bonne restructuration.

**Sévérité :** fail.

**Format de constat :**
```
{skill}/SKILL.md → {ref-1}.md → {ref-2}.md
Reasoning: {what {ref-2} contains and whether it should merge into {ref-1} or get a direct link from SKILL.md}
Action: {specific restructure}
```

**Correction automatique :** aucune.

---

## F6.5 : TOC des références

**Règle du framework :** les fichiers de référence de plus de 100 lignes ont besoin d'une TOC en tête.

**Heuristique de déclenchement :** pour chaque `references/*.md` lié depuis un SKILL.md, compter les lignes. >100 → vérifier les 30 premières lignes pour une TOC (section Contents / Table of Contents avec liens d'ancre).

**Jugement de l'agent :** pour chaque candidat sans TOC :
- Lire la liste des H2 du fichier. Générer une TOC suggérée.
- Le raisonnement fournit les ancres H2 sous forme de liste copiable-collable.

**Sévérité :** warn.

**Format de constat :**
```
{path} — {N} lines, no TOC
Reasoning: file has {n} H2 sections; TOC helps Claude do whole-file reads (no `head -100` truncation)
Suggested TOC: {generated list of [Section](#section) links}
Action: add a "## Contents" section at the top
```

**Correction automatique :** aucune en v0 (auto-génération possible mais l'ordre de la TOC nécessite une revue de l'utilisateur).

---

## F6.6 : Chemins de style Windows

**Règle du framework :** barres obliques uniquement.

**Heuristique de déclenchement :**
```
\b[A-Z]:\\
\\\\
\\(?![nrt"\\])
```

**Jugement de l'agent :** pour chaque correspondance, lire la ligne :
- Est-ce réellement un chemin Windows ou une barre inverse faisant partie d'une syntaxe d'échappement / regex ?
- Le raisonnement confirme lequel.

**Faux positifs à ignorer :**
- Les barres inverses à l'intérieur de littéraux regex ou de séquences d'échappement dans des blocs de code (déjà protégées).
- La documentation sur le comportement spécifique à Windows dans des `<details>`.

**Sévérité :** fail.

**Format de constat :**
```
{path}:{line} — Windows-style path
Excerpt: "{matched line}"
Reasoning: {what the path resolves to; that forward slashes work on Windows too}
Action: convert to forward-slash path
```

**Correction automatique :** aucune.

---

## F6.7 : Langage sensible au temps

**Règle du framework :** aucune information sensible au temps en ligne en dehors de `<details>`.

**Heuristique de déclenchement :**
```
\b(After|Before|Since|Until|As of|Starting in|From)\s+(January|February|March|April|May|June|July|August|September|October|November|December|Q[1-4]|\d{4})
\b(Last|This|Next)\s+(year|quarter|sprint|week)\b
```

**Jugement de l'agent :** pour chaque correspondance, lire :
- La date est-elle ancrée à un événement stable (ex. : « After v2.0 release », « After feature X ships ») → pas sensible au temps au sens du pourrissement ; ignorer.
- Est-ce réellement du pourrissement temporel (« After August 2025 », « Starting in Q3 ») → signaler.
- Est-ce déjà à l'intérieur d'un `<details>` ? → ignorer.

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — time-sensitive language
Excerpt: "{matched line}"
Reasoning: this date will rot; replace with a stable anchor (version, feature flag, event)
Suggested replacement: {a stable anchor the agent infers from context}
Action: move into <details> or replace with the stable anchor
```

**Correction automatique :** aucune.

---

## F6.8 : Constantes vaudou

**Règle du framework :** aucun nombre magique sans commentaire `# why` (dans les scripts référencés depuis le SKILL.md).

**Heuristique de déclenchement :** dans les scripts (pas le markdown), grep des littéraux numériques non suivis d'un commentaire explicatif. Filtrer les indices de tableau, les codes HTTP, les constantes courantes (60, 1000, 1024, 3600).

**Jugement de l'agent :** pour chaque correspondance :
- Le nombre s'explique-t-il de lui-même par le contexte (nom de fonction, nom de variable) ?
- Ou est-ce une vraie constante vaudou (ex. : `30` dans `if x > 30: …` sans signification évidente) ?
- Le raisonnement fournit la signification probable si elle est déductible, ou la demande.

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — magic number {value}
Excerpt: "{matched line}"
Reasoning: {what the number probably means based on surrounding code; or "meaning unclear from context"}
Action: add `# why ...` comment explaining the constant
```

**Correction automatique :** aucune.

---

## F6.9 : Terminologie incohérente

**Règle du framework :** terminologie cohérente : choisir un terme, l'utiliser partout.

**Heuristique de déclenchement :** rechercher les paires de synonymes dans un même SKILL.md :

| Groupe | Si les deux apparaissent → signaler |
|---|---|
| API endpoint / API path / endpoint | en choisir un |
| user / customer / client / account | en choisir un |
| repo / repository / project | en choisir un |
| folder / directory | en choisir un |
| node / step / stage | en choisir un |
| Casse de CLAUDE.md | en choisir une |
| Casse de skill / Skill / SKILL | en choisir une |

**Jugement de l'agent :** pour chaque paire de groupe, lire les deux occurrences :
- Les termes sont-ils réellement synonymes dans ce skill, ou désignent-ils des choses différentes (ex. : `customer` = utilisateur payant, `user` = utilisateur final en général) ?
- Si synonymes → signaler ; le raisonnement choisit le terme canonique selon la première utilisation ou la majorité.
- Si choses différentes → ignorer ; le skill fait ce qu'il faut.

**Sévérité :** warn.

**Format de constat :**
```
{path} — inconsistent terminology: "{term-a}" ({n}×) and "{term-b}" ({m}×)
Reasoning: {whether they're true synonyms here; which term is canonical}
Action: replace the non-canonical term throughout
```

**Correction automatique :** aucune.

---

## F6.10 : Préfixe MCP `ServerName:`

**Règle du framework :** les outils MCP toujours pleinement qualifiés.

**Heuristique de déclenchement :** détecter les noms d'outils nus qui ressemblent à des outils MCP : références à `slack_send`, `posts_create`, `vault_read`, etc. sans préfixe de serveur.

**Jugement de l'agent :** pour chaque candidat :
- L'outil est-il réellement un outil MCP ? (Recoupement avec des patterns connus : `mcp__server__tool`, ou le contexte implique MCP.)
- Ou le nom nu décrit-il un concept générique (« the search tool ») et non une invocation MCP spécifique ?
- Le raisonnement identifie à quel serveur appartient l'outil et propose la forme qualifiée.

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — possibly unqualified MCP tool reference
Excerpt: "{matched line}"
Reasoning: {whether this is an MCP tool, and if so which server}
Suggested form: {ServerName:tool_name} or {mcp__server__tool}
Action: qualify the tool reference
```

**Correction automatique :** aucune.

---

## F6.11 : Duplication skill-vault

**Règle du framework :** les skills ne devraient pas embarquer leurs propres copies de contenu que le vault possède déjà dans `Context/`.

**Heuristique de déclenchement :** correspondance de noms de fichiers entre le `references/` du skill et le `Context/` du vault :
- `icp*` / `ideal-customer*` / `customer-profile*` / `audience*` → `Context/icp.md`
- `voice*` / `tone*` / `brand*` → `Context/brand.md`
- `offers*` / `services*` / `what-we-do*` / `products*` → `Context/services.md`
- `me.md` / `profile*` / `operator*` / `background*` → `Context/operator.md`
- `strategy*` / `goals*` / `okrs*` → `Context/strategy.md`
- `team*` / `org*` → `Context/team.md` ou `Context/organization.md`

**Jugement de l'agent :** pour chaque paire candidate, **lire les deux fichiers** :
- Y a-t-il réellement duplication, ou la référence du skill fournit-elle un complément spécifique au skill ?
- Confirmer que le SKILL.md du skill référence le fichier dupliqué (sinon la référence est un orphelin obsolète, un problème différent).
- Le raisonnement cite 2 à 3 phrases qui se recoupent entre les deux fichiers.

**Sévérité :** warn.

**Format de constat :**
```
Skill: {skill-name}
Duplicate: {skill}/references/{file} ({bytes}B)
Vault file: Context/{vault-file}
Reasoning: {overlap evidence — quote 2-3 overlapping claims}
Action: rewrite SKILL.md to read Context/{vault-file}; delete the duplicate ref file (after grepping the skill folder)
```

**Correction automatique : corrigeable** : l'agent réécrit le SKILL.md pour pointer vers le chemin du vault, puis greppe le dossier du skill ; si la référence n'est pas référencée ailleurs, il la supprime. Si elle est encore référencée → fait remonter le conflit, ignore la suppression.

---

## F6.12 : Description à la première/deuxième personne

**Règle du framework :** les descriptions devraient être à la troisième personne.

**Heuristique de déclenchement :**
```
\b(I can|I will|I'll|I help|Use me|I'm|I am)\b
\b(you can|you will|you'll)\b
\b(we can|we will|we'll)\b
```

**Jugement de l'agent :** lire la description :
- La formulation à la première/deuxième personne dirige-t-elle réellement l'utilisateur, ou fait-elle partie d'un exemple cité ?
- Le raisonnement explique pourquoi la troisième personne se lit mieux pour la sélection de skill (Claude parcourt plus de 100 descriptions ; la troisième personne est moins ambiguë).
- Suggérer une réécriture.

**Sévérité :** warn.

**Format de constat :**
```
{path} — description in first/second person
Excerpt: "{matched substring}"
Reasoning: {why third person reads better for skill selection}
Suggested rewrite: {third-person version}
Action: rewrite
```

**Correction automatique :** aucune.

---

## Schéma de constat

Même forme que F1 : chaque constat possède un `reasoning`. Voir l'étape 2.4 du SKILL.md.
