# Caveman : discipline de compression

## Sommaire

1. [Thèse centrale](#thèse-centrale)
2. [L'article Brevity Constraints](#larticle-brevity-constraints)
3. [Règles de compression : ce qu'il faut retirer](#règles-de-compression--ce-quil-faut-retirer)
4. [Structures autorisées](#structures-autorisées)
5. [Zones strictement protégées : ne jamais compresser](#zones-strictement-protégées--ne-jamais-compresser)
6. [Table de substitution par symboles](#table-de-substitution-par-symboles)
7. [Niveaux d'intensité](#niveaux-dintensité)
8. [Benchmark réel : verbatim](#benchmark-réel--verbatim)
9. [Exemples avant / après](#exemples-avant--après)
10. [Skills compagnons](#skills-compagnons)
11. [Où appliquer caveman dans un vault](#où-appliquer-caveman-dans-un-vault)
12. [Où NE PAS appliquer caveman dans un vault](#où-ne-pas-appliquer-caveman-dans-un-vault)
13. [À faire](#à-faire)
14. [À ne pas faire](#à-ne-pas-faire)
15. [Citations verbatim](#citations-verbatim)
16. [Signaux auditables](#signaux-auditables)
17. [Sources](#sources)

---

## Thèse centrale

> *"why use many token when few token do trick"*

Éliminer le superflu tout en préservant l'exactitude technique. Le coût en tokens se cumule : chaque article, chaque "vraiment", chaque formule de précaution est multiplié à travers des milliers de conversations. La bonne réponse : retirer tout ce qui ne justifie pas ses tokens.

Compression moyenne sur le benchmark publié : **65 %**. Plage : **22 à 87 %**. Les gains les plus importants concernent les explications riches en prose ; les plus faibles concernent les revues de code et les comparaisons d'architecture (où le raisonnement structurel constitue la charge).

## L'article Brevity Constraints

Un article de mars 2026, *"Brevity Constraints Reverse Performance Hierarchies in Language Models"*, a constaté que les contraintes de concision amélioraient la précision de **+26 points de pourcentage** sur certains benchmarks. Le mécanisme : imposer la concision force le modèle à identifier et faire ressortir l'affirmation porteuse au lieu de l'enfouir sous des réserves.

C'est le fondement académique de l'intuition de caveman : la concision n'est pas seulement une économie de tokens, c'est un levier de qualité.

## Règles de compression : ce qu'il faut retirer

### Articles
Supprimer "a", "an", "the" partout où le sens est préservé.

✅ Retirer : *"the README"* → *"README"*
✅ Retirer : *"a function that returns"* → *"function returns"*
❌ Garder lorsque le sens change : *"the API"* (spécifique) vs *"an API"* (n'importe lequel)

### Mots de remplissage
- "just"
- "really"
- "basically"
- "simply"
- "please"
- "actually"
- "definitely"
- "literally"

Ils n'ajoutent aucune information. Les retirer sans exception.

### Formules de politesse
- "Sure!"
- "I'd be happy to..."
- "Let me explain..."
- "Great question!"
- "Thanks for asking..."
- Conclusions "Hope that helps!" / "Let me know if..."
- Excuses préliminaires ("Sorry for the confusion...")

### Formules de précaution
- "I think"
- "I believe"
- "I guess"
- "maybe"
- "perhaps"
- "might be"
- "could be"
- "kind of"
- "sort of"
- "in a way"

Utiliser des impératifs à la place. *"I think you should run npm test"* → *"Run `npm test`"*.

### Préambules d'introduction
- "Before we dive in..."
- "First, let me explain..."
- "It's worth noting..."
- "Let's start by..."

Dire la chose directement.

### Connecteurs verbeux
- "in order to" → "to"
- "due to the fact that" → "because"
- "at this point in time" → "now"
- "in the event that" → "if"
- "with regards to" → "for" / "on"

## Structures autorisées

- **Les fragments sont bons.** "Node by node." "In real time." Les lignes à une seule idée frappent le plus fort.
- **Modèle :** `[chose] [action] [raison]. [étape suivante].`
  - Exemple : *"Add `withAuth()`. Wraps handler in JWT check. Then redeploy."*
- **Impératifs plutôt que conditionnels.** "Run X" l'emporte sur "You might want to run X."
- **Tables de routage plutôt que prose.** Dès que vous avez ≥3 correspondances catégorielles, passez à un tableau markdown.

## Zones strictement protégées : ne jamais compresser

Ces éléments se cassent s'ils sont compressés :

| Zone | Pourquoi |
|---|---|
| **Blocs de code** (délimiteurs ` ``` `) | Les espaces/la syntaxe comptent |
| **URLs** | Un seul caractère modifié = lien cassé |
| **Chemins de fichiers** | La résolution de chemin est exacte |
| **Commandes et numéros de version** | `npm@9.5.1` ≠ `npm 9.5` |
| **Titres** | Les liens d'ancre se cassent |
| **Dates** | `2026-04-30` est canonique |
| **Clés de frontmatter** (YAML) | Validées par schéma en aval |
| **Code inline** (` `code` `) | Comme les blocs de code |
| **Wikilinks** `[[Target]]` | La correspondance de nom de fichier est exacte |
| **Délimiteurs de tableaux markdown** | Structurels |
| **Noms d'identifiants** | Contrats d'API |

En cas de doute : si un outil le lit, ne le compressez pas. Si un humain le lit, compressez.

## Table de substitution par symboles

Lorsque le contexte est sans ambiguïté, remplacez les connecteurs verbeux par des symboles. **À utiliser avec parcimonie** : une prose trop symbolique devient illisible pour les humains.

| Verbeux | Symbole | Quand c'est sûr |
|---|---|---|
| "leads to" / "results in" / "produces" | `→` | Chaînes de cause/effet, tables de routage |
| "and" | `&` | Sujets/objets composés, jamais comme conjonction en prose |
| "or" | `\|` | Alternatives dans les tableaux/options |
| "approximately" / "about" | `~` | Estimations numériques |
| "less than" / "greater than" | `<` / `>` | Seuils numériques |
| "equals" | `=` | Définitions/affectations |
| "increase" / "decrease" | `↑` / `↓` | Tableaux de tendance |
| "implies" / "therefore" | `⇒` / `∴` | Chaînes logiques (rares dans le contenu d'un vault) |

## Niveaux d'intensité

| Niveau | Déclencheur (dans le repo caveman) | Approche | Quand l'utiliser |
|---|---|---|---|
| **Lite** | `/caveman lite` | Retirer le remplissage, préserver la grammaire | Docs destinées aux clients qui doivent rester fluides à lire |
| **Full** *(par défaut)* | `/caveman full` | Retirer les articles, fragments OK, caveman complet | CLAUDE.md interne, fichiers d'instruction, `references/*.md` |
| **Ultra** | `/caveman ultra` | Télégraphique ; abréviation agressive | Contextes critiques en budget de tokens ; audience experte uniquement |
| **文言文 (wenyan)** | `/caveman wenyan` | Compression littéraire en chinois classique | Même philosophie, langue différente |

Pour un audit de vault typique, **Full** est le niveau par défaut. Lite est réservé aux docs de surface destinées aux humains.

## Benchmark réel : verbatim

| Tâche | Tokens normaux | Tokens caveman | Économie |
|---|---|---|---|
| Expliquer un bug de re-render React | 1,180 | 159 | **87%** |
| Configurer un pool de connexions PostgreSQL | 2,347 | 380 | **84%** |
| Corriger l'expiration de token du middleware d'auth | 704 | 121 | **83%** |
| Déboguer une race condition PostgreSQL | 1,200 | 232 | **81%** |
| Build Docker multi-stage | 1,042 | 290 | **72%** |
| Expliquer git rebase vs merge | 702 | 292 | **58%** |
| Relire une PR pour des problèmes de sécurité | 678 | 398 | **41%** |
| Architecture : microservices vs monolithe | 446 | 310 | **30%** |
| Refactorer un callback en async/await | 387 | 301 | **22%** |

Constat : les tâches riches en explication compressent de 70 à 87 %. Les tâches riches en raisonnement (comparaisons d'architecture, revues de code) compressent de 22 à 41 % car l'argument structurel constitue la charge.

## Exemples avant / après

### Exemple 1 : débogage React

> **Normal (69 tokens) :**
> *"The reason your React component is re-rendering is likely because you're creating a new object reference every time the component renders. When you pass an inline object as a prop, React sees a new reference each render and treats it as a prop change."*
>
> **Caveman (19 tokens) :**
> *"New object ref each render. Inline object prop = new ref = re-render. Wrap in `useMemo`."*

**Réduction de 73 %.** Même valeur opérationnelle. Le correctif est plus facile à trouver dans la seconde version.

### Exemple 2 : instruction CLAUDE.md (pertinent pour un vault)

> **Normal (38 tokens) :**
> *"It's generally a good idea to make sure you always run the test suite before committing any changes, especially if you've modified files in the API directory."*
>
> **Caveman (14 tokens) :**
> *"Run `npm test` before commit. Required for `src/api/**` changes."*

**Réduction de 63 %.** Le vague "generally a good idea" est devenu un impératif spécifique, ce qui, selon la recherche d'Anthropic, fait passer la conformité de 35 % à 89 %.

### Exemple 3 : règle de routage

> **Normal (32 tokens) :**
> *"When the user shares a meeting transcript or any kind of meeting note, you should put it in the Intelligence folder under meetings, and pick the right subfolder based on the meeting type."*
>
> **Caveman (16 tokens) :**
> *"Meeting → `Intelligence/meetings/{type}/`"*

**Réduction de 50 %.** Une table de routage remplace un paragraphe.

## Skills compagnons

Le repo caveman livre plusieurs skills qui appliquent la discipline à différentes couches :

| Skill | Ce qu'il compresse |
|---|---|
| **caveman-commit** | Messages de commit Git : commits conventionnels concis, sujet ≤50 caractères |
| **caveman-review** | Commentaires de revue de PR : une ligne par problème (ex. `L42: 🔴 bug: user null. Add guard.`) |
| **caveman-compress** | Fichiers de mémoire (CLAUDE.md, MEMORY.md) : ~46 % d'économie en entrée, originaux préservés |

Pour les audits de vault, l'équivalent serait une passe ciblée qui exécute caveman sur la hiérarchie CLAUDE.md. C'est dans la roadmap ; pas dans la v0.

## Où appliquer caveman dans un vault

✅ **Hiérarchie CLAUDE.md** : racine + par dossier. ROI le plus élevé ; chargée à chaque session.
✅ **Fichiers `.claude/rules/`** : même logique.
✅ **`description` du frontmatter SKILL.md des skills** : limite stricte de 1024 caractères ; la concision est obligatoire.
✅ **Index `MEMORY.md`** : les 200 premières lignes sont une taxe par session.
✅ **Tables de routage et résumés de décision** : gains de compression faciles.
✅ **Documentation interne** que seul l'agent lit.

## Où NE PAS appliquer caveman dans un vault

❌ **Notes destinées aux utilisateurs** que les humains lisent pour comprendre.
❌ **Transcriptions de réunion** : préserver la voix de chaque intervenant.
❌ **Notes quotidiennes** : la réflexion en langage naturel a de la valeur telle quelle.
❌ **Voix de marque / échantillons d'écriture** : la voix compte.
❌ **Contenu en brouillon** encore en cours d'élaboration.
❌ **Récits de décision** où la *chaîne de raisonnement* est la valeur.
❌ **Tout ce qui relève des sources brutes / du canon immuable.**
❌ **Commentaires de code** : de nombreux outils/coéquipiers en dépendent.

La règle empirique : **compresser la couche d'instruction destinée à l'agent, laisser tranquille la couche de connaissance destinée aux humains.**

## À faire

- Retirer les articles, le remplissage, les précautions, les politesses de la prose dans les fichiers d'instruction.
- Utiliser des fragments là où la grammaire n'est pas porteuse.
- Utiliser des impératifs plutôt que des conditionnels.
- Utiliser le modèle `[chose] [action] [raison]. [étape suivante].` pour les procédures.
- Convertir la prose comportant ≥3 correspondances catégorielles en tables de routage.
- Utiliser `→` pour la cause/effet ; `&` pour les sujets composés uniquement lorsque c'est sans ambiguïté.
- Appliquer Caveman Full à CLAUDE.md et `.claude/rules/`.
- Appliquer Caveman Lite aux docs de surface qui doivent rester fluides à lire.
- Relancer la compression comme une corvée périodique : la dérive revient toujours.

## À ne pas faire

- **Ne jamais** compresser les blocs de code, URLs, chemins de fichiers, commandes, numéros de version, titres, dates, frontmatter, wikilinks ou code inline.
- Ne pas retirer les termes techniques ou les identifiants.
- Ne pas compresser les tokens de raisonnement/réflexion (ce sont des sorties, pas des instructions).
- Ne pas compresser les fichiers que les coéquipiers lisent manuellement pour la compréhension humaine.
- Ne pas pousser jusqu'au niveau Ultra les fichiers d'instruction partagés en équipe : la lisibilité compte pour la revue.
- Ne pas trop symboliser : `→` et `&` sont acceptables ; enchaîner 5+ symboles par phrase devient illisible.
- Ne pas compresser les docs de voix de marque, les échantillons d'écriture, les transcriptions.
- Ne pas compresser les sources brutes / le canon immuable.

## Citations verbatim

> *"why use many token when few token do trick"*

> *"free like mass mammoth on open plain"* (licence)

## Signaux auditables

Lorsque ce skill exécute la Pass 4 (estimation de tokens) pour les candidats à la compression :

- **Densité de remplissage** : compter les occurrences de `\b(just|really|basically|simply|please|actually|definitely|literally)\b` pour 100 mots. Signaler les fichiers >2 pour 100.
- **Densité de précaution** : compter `\b(I (think|believe|guess)|maybe|perhaps|might be|could be|kind of|sort of)\b` pour 100 mots. Signaler les fichiers >1 pour 100.
- **Densité d'articles** dans les fichiers d'instruction (CLAUDE.md, skills) : compter les articles par phrase non protégée. Signaler si nettement plus élevée que les fichiers comparables.
- **Préambule de politesse** : détecter les débuts de note commençant par "Sure!", "I'd be happy to...", "Let me...", "Great question!". Presque toujours retirable.
- **Usage de connecteurs verbeux** : détecter `\b(in order to|due to the fact that|at this point in time|in the event that|with regards to)\b`. Suggérer des remplacements.
- **Longue prose vs candidats à une table de routage** : détecter les paragraphes comportant ≥3 items de liste qui pourraient être un tableau.
- **Taille de section dans le CLAUDE.md racine** : identifier les 3 plus grandes sections h2/h3. Estimer la compression Caveman à ~25 % par section. Faire ressortir comme cibles de réduction de la Pass 4.
- **Violations de zones protégées Caveman** : signaler si un bloc de code, une URL, un chemin de fichier ou un wikilink est manquant/abîmé d'une manière qui suggère une compression trop agressive.

## Sources

- https://github.com/JuliusBrussee/caveman (canonique)
- https://raw.githubusercontent.com/JuliusBrussee/caveman/main/README.md (README complet)
- *"Brevity Constraints Reverse Performance Hierarchies in Language Models"* : article de mars 2026 cité par le README de caveman
