# G7 : General Hygiene (pass implementation)

**Reference (the why):** project rules + practitioner field notes. Not from a single canonical framework.
**Applies to:** every `.md` file in the vault.

## Fonctionnement de cette passe

Agentique. La détection des tirets cadratin et la règle H1=nom de fichier sont surtout mécaniques, mais l'agent lit chaque candidat pour confirmer que la substitution ou la suppression est sûre compte tenu du contexte, et pour rédiger un raisonnement expliquant pourquoi ce cas précis viole la règle du projet. Voir l'introduction de F1 pour le pattern complet.

## Sommaire

1. [G7.1 : Tirets cadratins](#g71--tirets-cadratins)
2. [G7.2 : Conformité du frontmatter](#g72--conformité-du-frontmatter)
3. [G7.3 : H1 dupliquant le nom de fichier](#g73--h1-dupliquant-le-nom-de-fichier)
4. [G7.4 : Hygiène des README de projet](#g74--hygiène-des-readme-de-projet)
5. [Schéma de constat](#schéma-de-constat)

---

## G7.1 : Tirets cadratins

**Framework rule (project Rule 14):** never use em dashes; use periods, commas, colons, or restructure.

**Heuristique de déclenchement :** retirer les zones protégées (blocs de code, code inline, URLs, frontmatter, wikilinks, chemins). Faire un grep sur `—` (U+2014) et `–` (U+2013).

**Jugement de l'agent :** pour chaque candidat, lire la phrase environnante et choisir le bon remplacement :
- Tiret cadratin séparant deux propositions, où la seconde développe la première → `: ` (deux-points).
- Tiret cadratin encadrant une incise → `, … ,` (virgules).
- Tiret cadratin en fin de proposition jouant le rôle d'un point → `. ` (point).
- Tiret demi-cadratin dans une plage numérique (`100–200`) → garder le tiret demi-cadratin : c'est l'usage canonique, pas une violation de la Rule 14.
- Tiret demi-cadratin dans une plage de dates → garder.

Le raisonnement choisit la bonne substitution pour chaque occurrence et explique pourquoi.

**Faux positifs à ignorer :**
- Plages numériques avec tiret demi-cadratin (`100–200`, `5–10 minutes`).
- Plages de dates (`2026–2027`).
- Tirets cadratin à l'intérieur de paroles citées (préserver le texte original).

**Severity:** warn.

**Format du constat (un par fichier, agrégé) :**
```
{path} — {n} em-dash uses
Examples:
  L{line}: "{full sentence}" → suggested: "{rewritten sentence}"
  …
Reasoning: {how the em dashes are used in this file — clause separators, parentheticals, etc.}
Action: replace each per the suggestion (substitutions vary by use)
```

**Auto-fix :** **corrigeable sur acceptation de l'utilisateur** : l'agent applique la substitution par occurrence qu'il a confirmée (pas une regex en masse).

Après substitution, retirer à nouveau les zones protégées et vérifier que rien à l'intérieur du code, des URLs ou des wikilinks n'a été modifié. Si une sous-chaîne protégée est manquante ou abîmée → annuler la correction de ce fichier et le signaler.

---

## G7.2 : Conformité du frontmatter

**Framework rule (project):** content notes need `status:` and 2+ specific `tags:`. Recommended: `type:`, `date:`, `project:`, `department:`.

**Heuristique de déclenchement :** parser le frontmatter pour les fichiers dans le périmètre (`note`, `context`, `decision`, `meeting`, `daily`).

**Jugement de l'agent :** pour chaque champ manquant :
- Lire le fichier. L'agent peut-il inférer la bonne valeur ?
  - `status:` → chercher des signaux d'achèvement (un "Done", "Shipped", "Decided" quelconque) → suggérer `done` ; sinon `active` pour un travail en cours.
  - `tags:` → inférer 2 tags ou plus à partir du H1 du fichier, du contexte projet et du contenu.
  - `type:` → inférer à partir du pattern de nom de fichier et du contenu (`meeting`, `decision`, `note`, `reference`).
  - `date:` → chercher des dates dans le corps.
  - `project:` / `department:` → inférer à partir de l'emplacement du dossier.
- Le raisonnement fournit la valeur inférée avec un niveau de confiance : élevé (clairement visible) ou faible (meilleure estimation).

**Faux positifs à ignorer :**
- `CLAUDE.md`, `README.md`, `index.md`, `MEMORY.md` (méta).
- `.trash/`, `Onboarding/templates/`.
- Fichiers de tâches (format différent).

**Severity:** warn (par champ manquant).

**Format du constat (un par fichier, agrégé) :**
```
{path} — missing frontmatter: {fields}
Reasoning: inferable values — {field: inferred-value (high/low confidence)}; remaining need human input
Suggested frontmatter:
  ---
  status: {value}
  tags: [{tag1}, {tag2}]
  type: {value}
  ---
Action: add the suggested frontmatter
```

**Auto-fix :** aucun en v0 : confiance trop incertaine pour écrire automatiquement les valeurs de frontmatter.

---

## G7.3 : H1 dupliquant le nom de fichier

**Framework rule:** don't put a `# Title` heading that duplicates the filename.

**Heuristique de déclenchement :** lire la première ligne de contenu hors frontmatter. Si `# {Title}`, slugifier les deux et comparer.

**Jugement de l'agent :** confirmer :
- S'agit-il d'un CLAUDE.md / README.md ? Recouper avec F1.10 pour éviter de compter deux fois.
- Pour les README à la racine d'un dépôt git, le H1 peut être intentionnel pour l'affichage GitHub : signaler avec une sévérité faible, recommander de le garder si l'utilisateur préfère le rendu GitHub.

**Severity:** warn.

**Format du constat :**
```
{path} — H1 "{title}" duplicates filename
Reasoning: Obsidian/Claude already shows the filename as the title; H1 is redundant
Action: remove the H1 and any blank line below
```

**Auto-fix :** **corrigeable**.

---

## G7.4 : Hygiène des README de projet

**Framework rule:** project READMEs should be the entry point with overview/status/next-steps; subtopics route to subdir files.

**Heuristique de déclenchement :** fichiers correspondant à `Projects/*/README.md` ou `Projects/*/*/README.md`. Vérifier les sections (Overview/What, Status, Next/Roadmap) et la taille (<200B = clairsemé, >8KB = surchargé).

**Jugement de l'agent :** pour chaque candidat :
- Lire le README. Qu'y a-t-il, que manque-t-il ?
- Pour un README clairsemé : le raisonnement liste ce que dit réellement le README par rapport à ce dont un point d'entrée de projet a besoin.
- Pour un README surchargé : le raisonnement identifie les sous-sujets qui devraient être extraits (par ex. research/, specs/, notes/).
- Le raisonnement décrit la bonne structure pour CE projet précis.

**Severity:** warn.

**Format du constat (clairsemé) :**
```
{path} — sparse README ({bytes}B)
Reasoning: README contains {what's there}; missing {Overview / Status / Next steps}
Action: add overview, current status, next steps
```

**Format du constat (surchargé) :**
```
{path} — bloated README ({bytes}B)
Reasoning: README contains {N} distinct subtopics that should live in subdir files: {list}
Action: extract {subtopic-A} → research/{slug}.md, {subtopic-B} → specs/{slug}.md; keep README as the entry index
```

**Auto-fix :** aucun.

---

## Schéma de constat

Même forme que F1 : chaque constat possède un `reasoning`. Voir SKILL.md, étape 2.4.
