# F3 : Caveman Compression (implémentation de la passe)

**Référence (le pourquoi) :** `references/caveman-compression.md`.
**S'applique à :** la couche d'instruction, soit chaque `root-claude`, `folder-claude`, `claude-rules`, `skill` (SKILL.md), et chaque `.md` à l'intérieur de n'importe quel dossier `references/` d'un skill. NE s'applique PAS à : notes, dailies, réunions, transcriptions, décisions, contexte (couche de connaissances destinée à l'humain).

## Fonctionnement de cette passe

Agentique. **Crucial pour caveman** : l'heuristique de déclenchement seule produit un déluge de faux positifs parce que des mots comme `just`, `really`, `simply` font souvent un véritable travail dans leur contexte ("Just run X", par opposition à "run all of them"). L'agent lit chaque candidat **dans sa phrase environnante** et juge si le mot est du remplissage ou structurant.

Les constats incluent un champ `reasoning` expliquant pourquoi cette occurrence précise est du remplissage dans cette phrase précise. Sans ce jugement, cette passe serait inutile.

Avant l'exécution de toute vérification, l'orchestrateur retire les **zones protégées** afin que l'agent ne raisonne pas sur du code/des URLs/des chemins/des wikilinks. Voir la liste en bas de page.

## Sommaire

1. [F3.1 : Densité de mots de remplissage](#f31--densité-de-mots-de-remplissage)
2. [F3.2 : Densité de langage évasif](#f32--densité-de-langage-évasif)
3. [F3.3 : Préambules de politesse](#f33--préambules-de-politesse)
4. [F3.4 : Connecteurs verbeux](#f34--connecteurs-verbeux)
5. [F3.5 : Densité d'articles](#f35--densité-darticles)
6. [F3.6 : Top-3 des cibles de réduction](#f36--top-3-des-cibles-de-réduction)
7. [F3.7 : Candidats prose-vs-table-de-routage](#f37--candidats-prose-vs-table-de-routage)
8. [Zones protégées](#zones-protégées)
9. [Table de substitution (chemin de correction opt-in)](#table-de-substitution-chemin-de-correction-opt-in)
10. [Schéma de constat](#schéma-de-constat)

---

## F3.1 : Densité de mots de remplissage

**Règle du framework :** `just`, `really`, `basically`, `simply`, `please`, `actually`, `definitely`, `literally`, `very`, `quite` n'ajoutent typiquement aucune information.

**Heuristique de déclenchement :** compter les occurrences (après retrait des zones protégées) :
```
\b(just|really|basically|simply|please|actually|definitely|literally|very|quite|truly|essentially)\b
```
Calculer la densité pour 100 mots.

**Jugement de l'agent, c'est le cœur de la vérification :** pour chaque occurrence candidate, lire la phrase environnante et décider :

| Mot | Remplissage en contexte | Structurant en contexte |
|---|---|---|
| just | "It's just a quick check" | "Just run X (not all of them)" |
| really | "This is really important" | "Does it really fail when {condition}?" |
| simply | "Simply add the file" | "The function simply returns the input" (sens mathématique/algorithmique) |
| basically | "Basically, you ship it" | "It's basically O(n) but with a constant factor" |
| actually | "It's actually fine" | "Does X actually run? (vs. is supposed to)" |
| literally | "Literally just delete it" | "Literally as in the L-flag, not figurative" |

La règle empirique : si retirer le mot ne change rien sur le plan opérationnel, c'est du remplissage. S'il porte un contraste, une insistance sur la valeur de vérité, ou une précision technique, il est structurant.

Chaque constat signale UNE occurrence précise (ou agrège un petit lot d'occurrences clairement de remplissage dans le même fichier, avec un raisonnement). Le raisonnement cite la phrase et explique pourquoi ce `just` est du remplissage.

**Faux positifs à ignorer :**
- Discours rapporté (les mots d'un tiers entre backticks ou dans des blockquotes).
- Cas où le mot fait un travail technique/contrastif.
- Échantillons de code (déjà protégés).

**Sévérité :** warn (un constat par occurrence véritablement de remplissage ; plafonner à 25 par fichier avec "…and N more flagged in JSON sidecar").

**Format de constat :**
```
{path}:{line} — filler word "{word}"
Excerpt: "{full sentence}"
Reasoning: {why removing "{word}" doesn't change the meaning of THIS sentence}
Suggested edit: "{sentence with the word removed/replaced}"
Citation: caveman-compression.md → Compression rules
```

**Auto-correction :** **corrigeable sur opt-in de l'utilisateur, par fichier** : l'agent applique les substitutions uniquement aux occurrences qu'il a confirmées (pas à tous les résultats de la regex).

---

## F3.2 : Densité de langage évasif

**Règle du framework :** le langage évasif fait chuter la conformité. Utiliser des impératifs.

**Heuristique de déclenchement :**
```
\b(I think|I believe|I guess|maybe|perhaps|might be|could be|kind of|sort of|in a way|I'd suggest|you might want|it could|it might|seems like|appears to|tends to)\b
```

**Jugement de l'agent :** pour chaque candidat, lire la phrase environnante :
- "I think X is best" → retirer l'atténuation pour donner "X is best" (ou quelle que soit l'affirmation structurante). Signaler.
- "It might be that the API rate-limits us" → incertitude réelle sur un fait. Ignorer ; c'est une atténuation honnête à propos d'un comportement externe.
- "Maybe we should check the logs" : selon le cas, dans un CLAUDE.md/SKILL.md procédural c'est une atténuation ; dans des notes narratives, c'est une discussion.

La ligne de jugement : dans les fichiers de **couche d'instruction** (auxquels cette passe s'applique), l'atténuation affaiblit l'instruction. Dans le narratif/la discussion (que cette passe exclut), l'atténuation est honnête.

**Faux positifs à ignorer :**
- Incertitude factuelle honnête au sujet de systèmes externes.
- Contenu d'utilisateur/de conversation cité.
- Section qui traite *de quand* atténuer (méta).

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — hedge "{phrase}"
Excerpt: "{sentence}"
Reasoning: this is an instruction file; the hedge weakens the rule's compliance from ~89% to ~35%. Strip and use an imperative.
Suggested edit: "{imperative version}"
```

**Auto-correction :** **corrigeable sur opt-in de l'utilisateur**.

---

## F3.3 : Préambules de politesse

**Règle du framework :** les formules de politesse gaspillent des tokens.

**Heuristique de déclenchement :**
```
^\s*[-*0-9.]*\s*(Sure!?|I'd be happy to|Let me explain|Let me start by|Great question!?|Thanks for asking|Hope (this|that) helps|Let me know if|Before we dive in|First, let me|It's worth noting|To start)\b
```
Plus les clôtures : `\b(hope (this|that) helps|let me know if you (need|have)|happy to (help|elaborate|clarify))\b`.

**Jugement de l'agent :** pour chaque candidat, lire la ligne et les 1 à 2 lignes suivantes.
- Est-ce un préambule littéral qui ne dit rien d'opérationnel ? → signaler. ("Let me explain how this works." → supprimer ; la ligne suivante l'explique vraisemblablement.)
- Est-ce une partie d'une conversation citée ? → ignorer.
- Est-ce à l'intérieur d'un `<details>` ou d'un bloc de citation préservant un exemple ? → ignorer.

**Faux positifs à ignorer :**
- Discours rapporté, exemples de ce qu'il NE faut PAS écrire.
- À l'intérieur de blocs `<details>` ou de citations `> ` utilisés comme exemples.

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — pleasantry preamble
Excerpt: "{first 80 chars}"
Reasoning: this line says nothing operational; the next line(s) carry the actual content
Action: delete the matched span
```

**Auto-correction :** **corrigeable sur opt-in de l'utilisateur**, supprimer uniquement la portion correspondante.

---

## F3.4 : Connecteurs verbeux

**Règle du framework :** "in order to" → "to". "due to the fact that" → "because". Etc.

**Heuristique de déclenchement :** faire correspondre à la table :

| Verbeux | Remplacement |
|---|---|
| `\bin order to\b` | `to` |
| `\bdue to the fact that\b` | `because` |
| `\bat this point in time\b` | `now` |
| `\bin the event that\b` | `if` |
| `\bwith regards to\b` | `for` (ou `on`) |
| `\bwith respect to\b` | `for` |
| `\bin spite of the fact that\b` | `although` |
| `\bin light of the fact that\b` | `because` |
| `\bin the process of\b` | (supprimer) |
| `\bfor the purpose of\b` | `to` |
| `\ba large number of\b` | `many` |
| `\ba small number of\b` | `few` |
| `\bthe majority of\b` | `most` |
| `\bin terms of\b` | `for` |

**Jugement de l'agent :** pour chaque candidat, lire la phrase.
- La plupart de ces substitutions sont sûres ; l'agent confirme que la substitution préserve le sens.
- "with respect to" signifie parfois un "with respect to" mathématique/juridique (calcul différentiel, réglementaire) : ignorer ces cas.
- "in terms of" signifie parfois "expressed as" (math) : ignorer si c'est technique.

**Faux positifs à ignorer :**
- Usage technique/mathématique/juridique où la forme la plus longue a un sens précis.

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line} — verbose connector "{phrase}" → "{replacement}"
Excerpt: "{sentence}"
Reasoning: {why the substitution preserves meaning here}
Suggested edit: "{sentence after substitution}"
```

**Auto-correction :** **corrigeable sur opt-in de l'utilisateur**.

---

## F3.5 : Densité d'articles

**Règle du framework :** supprimer les articles là où le sens tient.

**Heuristique de déclenchement :** compter `\b(a|an|the)\b` après retrait des zones protégées ; calculer la densité par rapport à la médiane des fichiers pairs.

**Jugement de l'agent :** simple signalement, cette passe est une mesure, pas une correction. Les articles ne sont pas sûrs à corriger automatiquement (glissement de sens : "the API" vs "an API"). L'agent lit brièvement le fichier et raisonne sur le fait que la densité d'articles est inhabituellement élevée *compte tenu de la structure du fichier* (riche en prose → attendu ; riche en table de routage → inhabituel).

**Faux positifs à ignorer :**
- Fichiers majoritairement composés de prose (essais, récits).

**Sévérité :**
- densité > médiane des pairs × 2 → fail
- densité > médiane des pairs × 1.5 → warn

**Format de constat :**
```
{path} — article density {density} (peer median: {median})
Reasoning: this file is mostly {structure type} but uses articles {N}× the peer median; likely contains stripable "the/a/an"
Action: manual rewrite or run a caveman-lite pass; do NOT auto-substitute (meaning shifts)
```

**Auto-correction :** aucune.

---

## F3.6 : Top-3 des cibles de réduction

**Règle du framework :** identifier les plus grandes sections H2/H3 du CLAUDE.md racine comme candidates à la compression.

**Heuristique de déclenchement :** analyser tous les H2/H3 du CLAUDE.md racine, les classer par taille en octets décroissante.

**Jugement de l'agent :** pour le top 3, lire chaque section et raisonner sur une compression réaliste :
- Section riche en prose → ~50 à 70 % compressible.
- Mixte (prose + listes) → ~25 à 35 %.
- Riche en code / riche en tables → ~5 à 15 %.
- Le raisonnement explique la structure de chaque section du top et une estimation réaliste des gains (pas un 25 % fixe).

**Faux positifs à ignorer :**
- Sections qui sont intentionnellement un gros bloc de code / une grande table.

**Sévérité :** info (pas de sévérité ; c'est un panneau de mesure).

**Format de constat (un par section du top 3) :**
```
./CLAUDE.md → ## {section-name} — {bytes}B (~{tokens}t)
Reasoning: section is {structure breakdown}; realistic caveman savings ~{est}t after compressing the prose lines
Action: candidate for caveman compression pass
```

**Auto-correction :** aucune.

---

## F3.7 : Candidats prose-vs-table-de-routage

**Règle du framework :** ≥3 correspondances catégorielles → basculer vers une table markdown.

**Heuristique de déclenchement :** séquences consécutives de listes à puces/numérotées de ≥3 items où ≥75 % des items correspondent à `^[-*0-9.]+\s+([A-Z][\w\s]+)\s*[:→\-]+\s+(.+)$`.

**Jugement de l'agent :** pour chaque séquence candidate :
- Lire les items. Sont-ils vraiment des correspondances catégorielles (X correspond à Y) ou une liste d'items reliés mais non tabulaires ?
- Correspondances → signaler, suggérer une table avec des colonnes d'en-tête dérivées des items.
- Pas des correspondances → ignorer.

**Faux positifs à ignorer :**
- Étapes d'une procédure (1. Do X. 2. Do Y. : ce sont des étapes séquencées, pas catégorielles).
- Listes à puces de synonymes ou d'exemples.

**Sévérité :** warn.

**Format de constat :**
```
{path}:{line-start}-{line-end} — {n} categorical bullets — table candidate
Excerpt:
  L{a}: "{bullet 1}"
  L{b}: "{bullet 2}"
  L{c}: "{bullet 3}"
Reasoning: each bullet maps a {category} to {value}; a 2-column table compresses ~50% and is faster to scan
Suggested table headers: | {col1} | {col2} |
Action: convert to a markdown table
```

**Auto-correction :** aucune (le choix des en-têtes de colonnes requiert le jugement de l'utilisateur).

---

## Zones protégées

Retirer de la prise en compte avant que toute vérification F3 ne compte un résultat OU que toute correction ne s'exécute :

| Zone | Détection |
|---|---|
| Code fences | between ` ``` ` and ` ``` ` |
| Inline code | between `` ` `` and `` ` `` |
| URLs | `https?://\S+` |
| File paths | `[/\w.-]+\.(md\|ts\|js\|py\|sh\|json\|yaml\|yml\|html\|css)\b` |
| Frontmatter | between leading `---` and closing `---` |
| Wikilinks | `\[\[[^\]]+\]\]` |
| Markdown table delimiters | lines starting with `\|` or `\|---` |
| Headings | lines starting with `#` |
| Dates | `\d{4}-\d{2}-\d{2}` |
| Version numbers | `v?\d+\.\d+(\.\d+)?` |

Les constats rapportent toujours les numéros de ligne d'origine du fichier (pas les décalages du corps retiré).

---

## Table de substitution (chemin de correction opt-in)

Utilisée à l'étape 5 de SKILL.md lorsque l'utilisateur opte pour les corrections caveman sur un fichier précis. L'agent applique les substitutions **uniquement aux occurrences qu'il a confirmées via l'étape de jugement**, pas à chaque résultat de la regex.

| Pattern (regex, case-insensitive, after protected-zone stripping) | Replacement |
|---|---|
| `\bjust\s+\b` | (delete the word) |
| `\breally\s+\b` | (delete) |
| `\bbasically\s+\b` | (delete) |
| `\bsimply\s+\b` | (delete) |
| `\bplease\s+\b` | (delete) |
| `\bactually\s+\b` | (delete) |
| `\bdefinitely\s+\b` | (delete) |
| `\bliterally\s+\b` | (delete) |
| `\bvery\s+\b` | (delete) |
| `\bquite\s+\b` | (delete) |
| `\bin order to\b` | `to` |
| `\bdue to the fact that\b` | `because` |
| `\bat this point in time\b` | `now` |
| `\bin the event that\b` | `if` |
| `\bwith regards to\b` | `for` |
| `\bwith respect to\b` | `for` |
| `\bin spite of the fact that\b` | `although` |
| `\bin light of the fact that\b` | `because` |
| `\bfor the purpose of\b` | `to` |
| `\ba large number of\b` | `many` |
| `\ba small number of\b` | `few` |
| `\bthe majority of\b` | `most` |
| `\bin terms of\b` | `for` |
| `\bI think\s+\b` | (delete) |
| `\bI believe\s+\b` | (delete) |
| `\bI guess\s+\b` | (delete) |
| `\bmaybe\s+\b` | (delete) |
| `\bperhaps\s+\b` | (delete) |
| `\bkind of\s+\b` | (delete) |
| `\bsort of\s+\b` | (delete) |
| Pleasantry preambles per F3.3 | delete the matched span |

Après substitution, retirer de nouveau les zones protégées et vérifier qu'aucun code/chemin/URL/wikilink n'a été modifié. Si une sous-chaîne protégée est manquante/altérée dans le diff → abandonner la correction sur ce fichier et le signaler.

---

## Schéma de constat

Même forme que F1, chaque constat possède un `reasoning`. Voir SKILL.md, étape 2.4.
