---
name: os-optimizer
description: "Framework-driven audit and optimizer for any markdown vault. Applies 9 frameworks (F1 Anthropic CLAUDE.md, F2 Karpathy Wiki, F3 Caveman, F4 Chroma Context Rot, F5 Anthropic Memory, F6 Progressive Disclosure, G7 Hygiene, F8 Reflection / Anthropic Dreams, F9 Architecture & Discoverability). F9 walks the actual co-worker-Claude discovery chain (root CLAUDE.md → routing → folder Plot.md → file), audits routing-table truthfulness against folder reality, generates/refreshes per-folder Plot.md indexes, detects navigation orphans, proposes architectural reorganizations grounded in the user's Context/. Every finding ships a concrete fix — nothing is flag-only, nothing is fix-later; user picks apply-now (walk per item) or save-to-plan per finding. Creates visible per-stage tasks via TaskCreate so the user watches the run unfold. TRIGGERS: os optimizer, optimize vault, vault audit, second brain audit, clean up vault, framework audit, discoverability check, architecture audit, reorg vault. Run from vault root."
---

# Optimiseur de vault

Applique 9 frameworks à chaque fichier markdown du vault. Pour chaque framework, lis son fichier d'implémentation de la passe, exécute chaque vérification, journalise les constats, accompagne l'utilisateur dans une revue pas à pas des corrections item par item, applique (ou enregistre dans un plan). Enregistre un rapport HTML complet, groupé par framework. **N'intègre pas le HTML dans le chat : seulement le chemin enregistré et un résumé d'un paragraphe.**

## Philosophie de fonctionnement : à lire attentivement, c'est ce qui distingue ce skill

1. **Chaque constat embarque une correction concrète.** Pas de simple signalement. Pas de « signaler puis oublier ». Pas de tas « à revoir manuellement ». Quand l'utilisateur lance le mode application, chaque constat devient soit une modification appliquée, soit une étape de migration enregistrée dans un plan de réorganisation daté, soit (uniquement si l'utilisateur refuse explicitement ce constat pendant la revue) un refus consigné. Rien ne subsiste comme avertissement ouvert d'une exécution à l'autre.
2. **La sévérité est informative, pas bloquante.** `fail` / `warn` / `info` décrit à quel point le problème est structurant ; cela ne conditionne *pas* le fait qu'une correction soit proposée. Chaque vérification produit des corrections.
3. **Revue pas à pas, l'utilisateur choisit la cible.** L'application en masse est réservée aux corrections purement mécaniques (tirets cadratins, H1 dupliqués). Tout ce qui est sémantique : repointage de wikilink, fusions, réécritures de routage, génération de Plot.md, réorganisations, se fait uniquement en revue pas à pas, l'utilisateur confirmant la destination/le gagnant/la formulation item par item.
4. **Deux modes pour chaque correction :** *appliquer maintenant* (s'exécute dans cette exécution) ou *enregistrer dans le plan* (écrit le changement comme étape de checklist dans `Intelligence/decisions/{date}-reorg.md` pour une exécution échelonnée). L'utilisateur choisit par constat pour les items à fort rayon d'impact. Les corrections plus petites passent par défaut en appliquer-maintenant.
5. **Progression visible, jamais silencieuse.** L'étape 0.5 crée une entrée TaskCreate par stade et par framework ; les tâches passent de `in_progress` à `completed` au fil de l'exécution. L'utilisateur regarde l'audit parcourir le vault au lieu d'attendre un gros rapport à la fin.
6. **Lire et raisonner, pas seulement faire correspondre.** Les déclencheurs de chaque framework font remonter des candidats ; chaque constat exige que l'agent lise le contexte, juge l'alignement avec le monde décrit par l'utilisateur, et produise un raisonnement spécifique au cas. Pas de reformulations paraphrasées de règles présentées comme du « raisonnement ».
7. **Découvrir la structure, ne jamais présumer des noms de dossiers.** Les vaults varient. La couche curatée d'un utilisateur est `Context/` ; pour un autre c'est `About/`, `Me/`, du frontmatter sur la racine, ou des éléments dispersés dans des dossiers thématiques. L'optimiseur exécute l'étape 1.5 (découverte des rôles) avant tout framework, et chaque framework référence des **rôles** (équivalent-contexte, équivalent-décisions, équivalent-quotidien, convention d'index de dossier…) découverts à partir du contenu, jamais codés en dur. Si un rôle est absent, il remonte comme un constat assorti d'une correction proposée (« vous n'avez pas de dossier équivalent-décisions ; voici une recommandation »), et non comme une hypothèse silencieuse. Les références de chemin statiques dans les fichiers de passe sont toujours des abstractions ; l'agent les résout via le registre des rôles au moment de l'exécution.

## Frameworks

| # | Framework | Référence (le *pourquoi*) | Fichier de passe (le *comment*) | S'applique à |
|---|---|---|---|---|
| F1 | Anthropic CLAUDE.md | `references/anthropic-claude-md.md` | `references/passes-anthropic-claude-md.md` | chaque `CLAUDE.md` |
| F2 | Karpathy LLM Wiki | `references/karpathy-llm-wiki.md` | `references/passes-karpathy-wiki.md` | notes de contenu wiki |
| F3 | Caveman compression | `references/caveman-compression.md` | `references/passes-caveman.md` | fichiers de la couche instruction |
| F4 | Chroma context rot | `references/chroma-context-rot.md` | `references/passes-chroma-context-rot.md` | chaque `.md` |
| F5 | Anthropic Memory | `references/anthropic-managed-memory.md` | `references/passes-anthropic-memory.md` | chaque `.md` |
| F6 | Progressive Disclosure | `references/progressive-disclosure.md` | `references/passes-progressive-disclosure.md` | chaque `SKILL.md` |
| G7 | General Hygiene | (règles projet + notes de praticien) | `references/passes-general-hygiene.md` | chaque `.md` |
| F8 | Reflection (Anthropic Dreams) | `references/anthropic-dreams.md` | `references/passes-reflection.md` | couche curatée (`Context/`, `Projects/`, `Intelligence/decisions/`, `Resources/`) ; lit les `Daily/` + `Intelligence/meetings/` récents comme preuves |
| F9 | Architecture & Discoverability | `references/anthropic-architecture.md` | `references/passes-architecture.md` | tout le vault : root CLAUDE.md, routage, le `Plot.md` de chaque dossier, la chaîne de navigation de bout en bout |

Quand tu exécutes une vérification, **lis le fichier d'implémentation de la passe** et suis exactement son format de regex / d'heuristique / de constat. Ne paraphrase pas. Cite la référence du framework dans chaque constat.

## Déroulé

1. **Vérifier que le cwd ressemble à un vault** : vérification légère (étape 0)
2. **Créer la liste de tâches visible** : TaskCreate, une tâche par stade + framework (étape 0.5)
3. **Découvrir et classifier chaque fichier `.md`** : uniquement des exclusions techniques (étape 1)
4. **Itérer les frameworks F1 → F9**, en appliquant la grille de chaque framework avec le jugement de l'agent ; F8 = synthèse trans-vault, F9 = raisonnement structurel sur tout le vault + revue de découvrabilité (étape 2)
5. **Agréger les constats + écrire le paragraphe de lecture architecturale** (étapes 3 et 3.5)
6. **Parcourir chaque constat via appliquer-maintenant / enregistrer-dans-le-plan** : pas d'option de saut pour les corrections ; chaque constat devient une modification appliquée, une étape de migration enregistrée, ou un refus explicite de l'utilisateur item par item (étape 4)
7. **Appliquer les corrections approuvées** (étape 5)
8. **Rendre le tableau de bord HTML, l'enregistrer, l'ouvrir dans le navigateur, l'émettre comme artefact** (étape 6)

---

## Étape 0 : Vérifier que le cwd ressemble à un vault

N'impose aucune disposition de dossiers particulière. Vérifie (l'une des conditions suffit) :

```bash
test -f CLAUDE.md || test -f claude.md
[ "$(find . -maxdepth 4 -name 'CLAUDE.md' | head -1)" ]
[ "$(find . -maxdepth 1 -name '*.md' | wc -l)" -gt 0 ]
```

Si aucune n'est vraie → arrête :

> Cela ne ressemble pas à un vault markdown : aucun fichier `.md` ni `CLAUDE.md` trouvé. Placez-vous (`cd`) dans la racine de votre vault et relancez.

Sinon, indique à l'utilisateur une seule ligne :

> J'audite votre vault contre 9 frameworks. Je vais d'abord découvrir votre structure (étape 1.5) : je ne présumerai pas des noms de dossiers. Ensuite, je parcours chaque correction avec vous. Vous verrez chaque stade sous forme de tâche.

Passe à l'étape 0.5.

---

## Étape 0.5 : Créer la liste de tâches visible (TaskCreate)

C'est obligatoire. Les personnes qui lancent ce skill sur leur second cerveau ont besoin de voir ce qui se passe : le silence est inacceptable pour un audit de vault de longue durée.

Crée une tâche par stade + une tâche par framework, dans l'ordre. Utilise TaskCreate avec des titres courts et explicites. L'utilisateur les regarde s'égrener.

```
[ ] Découvrir et classifier les fichiers .md (Step 1)
[ ] Découverte des rôles : classification sémantique dossier/fichier, aucun nom codé en dur (Step 1.5)
[ ] F1 Anthropic CLAUDE.md : lire les fichiers CLAUDE.md, juger les candidats
[ ] F2 Karpathy Wiki : wikilinks, orphelins, schéma
[ ] F3 Caveman : compression des fichiers de la couche instruction
[ ] F4 Chroma Context Rot : longueur, distracteurs, position
[ ] F5 Anthropic Memory : taille de fichier, nommage, index
[ ] F6 Progressive Disclosure : layering des SKILL.md
[ ] G7 General Hygiene : tirets cadratins, frontmatter, règles H1
[ ] F8 Reflection : clustering + jugement de la synthèse trans-fichiers
[ ] F9 Architecture : véracité du routage, présence de Plot.md, revue de découvrabilité
[ ] Agréger les constats + lecture architecturale (Steps 3 / 3.5)
[ ] Parcourir chaque constat via appliquer / enregistrer-dans-le-plan (Step 4)
[ ] Appliquer les corrections approuvées (Step 5)
[ ] Rendre le tableau de bord + ouvrir (Step 6)
```

Marque `in_progress` à l'entrée d'un stade ; `completed` à la sortie. Pour les frameworks longs (F2, F8, F9), émets une sous-mise à jour en cours d'exécution via TaskUpdate ou une seule ligne de chat, afin que l'utilisateur connaisse la progression.

Sauter la liste de tâches pour « gagner du temps » va à l'encontre de la raison d'être de ce skill. La liste de tâches visible n'est pas optionnelle.

---

## Étape 1 : Découvrir et classifier chaque fichier `.md`

### 1.1 : Glob universel (chaque fichier audité)

```bash
find . -name '*.md' \
  -not -path '*/.git/*' \
  -not -path '*/.obsidian/*' \
  -not -path '*/.trash/*' \
  -not -path '*/.claude/worktrees/*' \
  -not -path '*/node_modules/*' \
  -not -path '*/dist/*' \
  -not -path '*/build/*'
```

**Aucune exclusion fondée sur le rôle.** Pas de « templates exclus », pas de « Daily exclu », pas de « Onboarding exclu ». Chaque `.md` hors de la liste d'exclusion technique ci-dessus est audité contre chaque règle de framework qui s'applique à son rôle. La classification route les bonnes règles vers les bons fichiers ; la classification n'exclut PAS de fichiers.

### 1.2 : Classifier chaque fichier par rôle (la première correspondance l'emporte) : **indices uniquement**

Les motifs ci-dessous sont des *indices* qui orientent le registre des rôles construit à l'étape 1.5. Ils ne font pas autorité : l'étape 1.5 lit un échantillon de contenu pour confirmer les attributions de rôle. Un fichier correspondant à `\d{4}-\d{2}-\d{2}\.md` hors de tout dossier en forme de daily ne sera pas classé `daily` si l'étape 1.5 ne trouve pas de rôle daily ; ce sera une note ordinaire.

Pour la rétrocompatibilité avec les vaults qui *utilisent* bien les noms conventionnels, les motifs s'appliquent toujours ; pour ceux qui ne les utilisent pas, l'étape 1.5 prend le relais.

| Rôle | Détection |
|---|---|
| `root-claude` | `./CLAUDE.md` ou `./claude.md` (racine du cwd uniquement) |
| `folder-claude` | tout autre `CLAUDE.md` / `claude.md` dans les sous-dossiers |
| `claude-rules` | fichiers à l'intérieur de `.claude/rules/` |
| `skill` | fichiers `SKILL.md` (n'importe où) |
| `index` | `index.md` (insensible à la casse) |
| `readme` | `README.md` (insensible à la casse) |
| `daily` | correspond à `\d{4}-\d{2}-\d{2}\.md` à l'intérieur d'un `Daily/` |
| `meeting` | à l'intérieur de `*meetings*/` ou nom de fichier correspondant à `\d{4}-\d{2}-\d{2} - .+\.md` hors de `Daily/` |
| `transcript` | à l'intérieur de `*transcripts*/` ou fichiers >100KB |
| `decision` | à l'intérieur de `*decisions*/` |
| `template` | à l'intérieur de `*templates*/` ou nom de fichier finissant par `-template.md` |
| `context` | à l'intérieur d'un `Context/` (insensible à la casse) |
| `note` | tout le reste |

Construis la carte de classification :

```json
{
  "root_claude": "./CLAUDE.md",
  "folder_claudes": [...],
  "claude_rules": [...],
  "skills": [...],
  "indexes": [...],
  "readmes": [...],
  "dailies": [...],
  "meetings": [...],
  "transcripts": [...],
  "decisions": [...],
  "templates": [...],
  "context_files": [...],
  "notes": [...],
  "by_folder": {...},
  "stats": {"total_files": N, "total_bytes": B, "folders": F}
}
```

### 1.3 : Construire les index de support (utilisés par F2/F4)

| Index | Construit à partir de | Utilisé par |
|---|---|---|
| `vault_filename_index` | chaque nom de base `.md`, en minuscules, avec et sans extension | F2.2, F2.4 |
| `inbound_link_index` | grep sur le vault pour `\[\[name(\||\]|#)` | F2.3, F2.4 |
| `routing_table` | section routing/knowledge-routing du root CLAUDE.md | F2.6 |
| `top_level_entries` | `find . -maxdepth 1` | F2.6 |
| `headers_index` | liste H2/H3 par fichier avec numéros de ligne + tailles en octets | F3.6, F5.2 |
| `protected_zones_map` | carte par fichier des plages code/URL/chemin/frontmatter/wikilink | F3.x, G7.1 |

### 1.4 : Afficher le résumé de classification dans le chat (un seul bloc, avant l'exécution de tout framework)

```markdown
## 📋 Découverte : {N} fichiers markdown sur {F} dossiers, {B-formatted} au total

| Rôle | Nombre |
|---|---:|
| Root CLAUDE.md | 1 |
| CLAUDE.md de dossier | {n} |
| Skills (SKILL.md) | {n} |
| .claude/rules | {n} |
| Index / READMEs | {n} |
| Fichiers de contexte | {n} |
| Notes | {n} |
| Notes quotidiennes | {n} |
| Réunions | {n} |
| Transcriptions | {n} |
| Décisions | {n} |
| Templates | {n} |

**Cibles des frameworks (chaque fichier du périmètre est audité) :**
- F1 Anthropic CLAUDE.md → {n} fichiers CLAUDE.md
- F2 Karpathy Wiki → {n} notes de contenu (notes + contexte + décision + réunion + index + readme) + 1 vérification de doc de schéma
- F3 Caveman → {n} fichiers de la couche instruction (CLAUDE.md + SKILL.md + .claude/rules + références de skill)
- F4 Chroma Context Rot → {N} fichiers (chaque `.md`)
- F5 Anthropic Memory → {N} fichiers (chaque `.md`)
- F6 Progressive Disclosure → {n} skills
- G7 General Hygiene → {N} fichiers
- F8 Reflection → {n} fichiers dans les rôles de couche curatée découverts + {n} fichiers dans les rôles de couche de session découverts (fenêtre {window}) pour la synthèse trans-vault
- F9 Architecture → table de routage ({n} entrées), {n} dossiers pour la présence d'index + la fraîcheur contre la convention découverte, graphe de navigation complet pour la découvrabilité, orientation spécifique au vault construite à partir de la couche identité découverte

Lancement de la découverte des rôles (étape 1.5) maintenant…
```

---

## Étape 1.5 : Découverte des rôles (sémantique, pas fondée sur les noms)

**Cette étape remplace toute hypothèse codée en dur sur les noms de dossiers/fichiers.** Les fichiers de passe référencent des *rôles* (`context`, `projects`, `decisions`, `daily`, `meetings`, `transcripts`, `resources`, `skills`, `archive`, `identity`, `folder_index_convention`). L'étape 1.5 découvre quel dossier/fichier joue chaque rôle dans *ce* vault, ou consigne que le rôle est absent.

### 1.5.1 : Comment fonctionne la découverte

Pour chaque rôle abstrait, l'agent fait ce qui suit, dans l'ordre, jusqu'à ce que quelque chose se résolve :

1. **Lire les noms de dossiers + les fichiers Plot/README/index/CLAUDE au niveau supérieur**, puis jusqu'à deux niveaux de profondeur. Construire une liste de candidats : quels dossiers pourraient jouer ce rôle d'après leur nom + leur propre description ?
2. **Lire 3 à 5 fichiers d'exemple par dossier candidat.** Le contenu correspond-il à la finalité du rôle ?
3. **Scorer et choisir.** Le candidat le plus fiable l'emporte. Si aucun candidat n'atteint une confiance moyenne → le rôle est `missing`.

L'agent ne **privilégie pas** un nom de dossier particulier. `Context/`, `About/`, `Me/`, `Personal/`, `Identity/`, ou une section de frontmatter sur le root CLAUDE.md peuvent tous jouer le rôle `identity`. L'agent décide en lisant.

### 1.5.2 : Rôles standards (motifs que l'agent reconnaît)

Ce sont des *motifs*, pas une taxonomie exhaustive. Chaque dossier/fichier est classifié : ces rôles standards correspondent aux formes courantes ; tout ce qui ne correspond pas devient un **rôle personnalisé** (étape 1.5.3).

| Rôle standard | Couche par défaut | De quoi il s'agit | Comment le reconnaître |
|---|---|---|---|
| `identity` | curated | Fichiers décrivant l'utilisateur/opérateur (qui il est, ce qu'il fait, sa voix, ses préférences) | Contenu biographique à la première personne ; mentions de rôle/titre ; consignes de voix ou de style |
| `context` | curated | Dossier(s) contenant la connaissance canonique du monde de l'utilisateur (business, stratégie, marque, équipe, parties prenantes), plus large que l'identité | Faits déclaratifs au présent sur l'environnement opérationnel ; entités nommées (entreprise, produits, personnes clés) |
| `projects` | curated | Unités de travail actives ou récentes | Noms de dossiers correspondant aux projets mentionnés dans identity/context ; index par dossier décrivant le périmètre, le statut, les échéances |
| `decisions` | curated | Enregistrements de décisions persistants | Fichiers avec préfixes de date contenant un langage de décision (« decided », « chose », « going with ») |
| `daily` | session | Journaux ou logs par jour | Noms de fichiers correspondant à `YYYY-MM-DD\.md` ; dossier organisé par hiérarchie de dates |
| `meetings` | session | Notes de réunion | Noms de fichiers avec date + personne/sujet ; contenu avec listes de participants, actions à mener |
| `transcripts` | session | Transcriptions brutes d'appels/de voix | Fichiers >100KB avec mise en forme monologue/dialogue |
| `resources` | curated | Bibliothèque de référence (prompts, frameworks, swipe files, templates) | Ressources réutilisables, non spécifiques à un projet ; souvent imbriquées par catégorie |
| `skills` | curated | Contenu de skill / SOP / playbook que l'utilisateur possède | Fichiers `SKILL.md`, ou markdown décrivant des processus/playbooks répétables |
| `archive` | archive | Contenu désactivé intentionnellement | Dossier nommé archive/old/deprecated/_archive, ou fichiers marqués archivés dans le frontmatter |
| `folder_index_convention` | (meta) | Le nom de fichier d'index par dossier choisi par l'utilisateur | Nom de fichier le plus fréquent à travers les dossiers qui fonctionne comme un index (`Plot.md`, `README.md`, `index.md`, `_index.md`, `CLAUDE.md`, etc.) |

### 1.5.3 : Rôles personnalisés (tout autre dossier)

Après la découverte des rôles standards, **chaque dossier de premier niveau restant et chaque sous-dossier significatif doivent être classifiés**, pas ignorés. Pour chaque dossier non classifié :

1. **Lire le fichier d'index du dossier** (selon la convention découverte) s'il est présent.
2. **Échantillonner 3 à 5 fichiers** dans le dossier. Lire les 1500 premiers caractères + les en-têtes.
3. **Lire l'index du dossier parent** (le cas échéant) pour voir comment ce dossier est décrit en amont.
4. **Attribuer un rôle personnalisé :**
   - `name` : un slug dérivé du nom du dossier plus du contenu (ex. `Building/` avec du contenu de construction de prototypes → nom de rôle `building` ; `Garden/` avec du contenu d'incubation d'idées → `garden`).
   - `layer` : l'une des valeurs `curated` (canonique, durable), `session` (éphémère, horodaté), `archive` (désactivé), `meta` (outillage, fichiers système), `unknown` (l'agent n'a pas pu classifier avec confiance).
   - `purpose` : description d'une ligne de ce que contient le dossier.
   - `is_standard: false`.
   - `confidence: high | medium | low`.

Si l'agent ne peut pas attribuer une couche avec confiance (confiance faible) → émets un constat F9.0 demandant à l'utilisateur de préciser la finalité du dossier pendant la revue. La réponse persiste désormais dans le registre.

**Les rôles personnalisés sont de premier rang.** F8, F9 et les correcteurs par constat opèrent sur les rôles par *couche*, pas par appartenance à la liste des rôles standards. Un rôle personnalisé `building` avec `layer: curated` participe à la synthèse de la couche curatée de F8 exactement comme le rôle standard `context`.

### 1.5.4 : Sortie : le registre des rôles

Construis-le une seule fois, mets-le en cache pour le reste de l'exécution, et persiste-le dans `.claude/vault-roles.json` à la fin de l'étape 1.5 afin que les exécutions futures démarrent à partir d'attributions confirmées plutôt que de redemander :

```json
{
  "vault_root": "./",
  "discovered_at": "2026-05-08T14:23:00Z",
  "folder_index_convention": {
    "name": "README.md",
    "confidence": "high",
    "evidence": "23 of 31 non-trivial folders have README.md",
    "coverage": 0.74
  },
  "roles": [
    {"name": "identity",  "path": "./About/me.md", "kind": "file",   "layer": "curated", "is_standard": true,  "confidence": "high",   "purpose": "Operator bio + voice"},
    {"name": "context",   "path": "./Knowledge/",  "kind": "folder", "layer": "curated", "is_standard": true,  "confidence": "high",   "purpose": "Org/strategy/brand canonical knowledge"},
    {"name": "projects",  "path": "./Work/",       "kind": "folder", "layer": "curated", "is_standard": true,  "confidence": "high",   "purpose": "Active and recent work units"},
    {"name": "daily",     "path": "./Journal/",    "kind": "folder", "layer": "session", "is_standard": true,  "confidence": "medium", "purpose": "Per-day journal entries"},
    {"name": "resources", "path": "./Library/",    "kind": "folder", "layer": "curated", "is_standard": true,  "confidence": "high",   "purpose": "Prompts, frameworks, templates"},
    {"name": "archive",   "path": "./_archive/",   "kind": "folder", "layer": "archive", "is_standard": true,  "confidence": "high",   "purpose": "Deactivated content"},
    {"name": "building",  "path": "./Building/",   "kind": "folder", "layer": "curated", "is_standard": false, "confidence": "high",   "purpose": "Active prototype builds and experiments"},
    {"name": "garden",    "path": "./Garden/",     "kind": "folder", "layer": "curated", "is_standard": false, "confidence": "medium", "purpose": "Long-form essays in slow incubation"},
    {"name": "inbox",     "path": "./Inbox/",      "kind": "folder", "layer": "session", "is_standard": false, "confidence": "high",   "purpose": "Unprocessed capture; aged out into Garden or Resources"}
  ],
  "missing_standard_roles": ["decisions", "meetings", "transcripts"],
  "low_confidence_roles": ["garden"],
  "unconfirmed_custom_roles": []
}
```

### 1.5.5 : Afficher le résumé de découverte dans le chat (un seul bloc)

Présente cela comme « voici la structure que je vois dans votre vault », et non « voici ce qui manque par rapport à une checklist ». La structure de l'utilisateur est la source de vérité ; la taxonomie standard AI5D n'est qu'une des grilles que l'agent utilise pour reconnaître des motifs.

```markdown
## 🔍 Structure de votre vault : {N_folders} dossiers classifiés

| Dossier | Rôle | Couche | Finalité | Confiance |
|---|---|---|---|---|
| ./About/me.md | identity | curated | Bio + voix de l'opérateur | high |
| ./Knowledge/ | context | curated | Org/stratégie/marque | high |
| ./Work/ | projects | curated | Travail actif et récent | high |
| ./Journal/ | daily | session | Journal par jour | medium |
| ./Library/ | resources | curated | Prompts, frameworks, templates | high |
| ./_archive/ | archive | archive | Contenu désactivé | high |
| ./Building/ | building (personnalisé) | curated | Constructions de prototypes actives | high |
| ./Garden/ | garden (personnalisé) | curated | Essais longs en incubation lente | medium |
| ./Inbox/ | inbox (personnalisé) | session | Capture non traitée | high |

**Convention d'index de dossier détectée :** README.md (74 % de couverture)

J'auditerai cette structure telle quelle. F9.0 pourra suggérer des améliorations structurelles précises là où je vois une preuve concrète qu'elles aideraient votre vault (pas seulement parce que AI5D les utilise) : vous examinerez chaque suggestion en revue. Les attributions confirmées persistent dans `.claude/vault-roles.json`.

Lancement de F1 maintenant…
```

Le résumé met d'abord en avant *ce que possède l'utilisateur*. Les constats F9.0 sur les manques apparaissent plus tard dans la revue, avec un raisonnement ancré dans des problèmes observés, et non comme un tableau de carences au moment de la découverte.

### 1.5.6 : Règles strictes pour les frameworks en aval

- **Les frameworks référencent les rôles par *couche*, pas par nom.** La couche curatée de F8 = chaque rôle avec `layer == 'curated'` (standard ou personnalisé). La couche de session de F8 = chaque rôle avec `layer == 'session'`. F9 parcourt chaque dossier indépendamment de l'appartenance à un rôle.
- **Les rôles personnalisés participent à chaque framework qui opère sur leur couche.** Si `building` est en `layer: curated`, F8 examine `Building/` pour les candidats à la fusion, les contradictions, les thèmes et les promotions exactement comme `Context/`.
- **Les rôles absents ne bloquent jamais un framework.** Si `decisions` est absent, F8 procède sans lui ; F9.0 produit un constat suggérant à l'utilisateur d'en créer un (ou d'attribuer le rôle à un dossier personnalisé existant).
- **La convention d'index de dossier est ce que F9.2 applique**, pas Plot.md par défaut. Si le vault de l'utilisateur utilise README.md, F9.2 génère/rafraîchit README.md. Si aucune convention n'existe, F9.0 propose d'en adopter une.
- **Le registre persisté est lu au début de l'étape 1.5.** Si `.claude/vault-roles.json` existe d'une exécution antérieure, l'agent le charge comme référence de base et ne reclassifie que les *nouveaux* dossiers ou les dossiers signalés par l'utilisateur pour réexamen. Les attributions confirmées ne redemandent pas à chaque exécution.
- **Aucun framework ne code en dur des chemins de dossier ou des noms de fichier.** Les exemples dans les fichiers de passe sont illustratifs ; l'exécution résout via le registre.

---

## Étape 2 : Itérer les frameworks F1 → F9 avec jugement

**Ce n'est pas une passe de regex.** Pour chaque framework, lis son fichier d'implémentation de la passe, puis applique chaque vérification qu'il définit aux fichiers du périmètre de ce framework. Les déclencheurs dans les fichiers de passe font remonter des candidats ; l'agent lit le contexte et juge chaque candidat avant de produire un constat. Chaque constat inclut un `reasoning` spécifique au cas.

Pourquoi : une correspondance regex sur `\bjust\b` signale « just run X » (où « just » fait un vrai travail : contraster avec le fait d'en lancer plusieurs) de la même façon que « It's just a quick check » (où c'est du remplissage). Seul un agent qui lit la ligne en contexte peut distinguer l'un de l'autre. Cela vaut aussi pour « be careful » (parfois un rappel de clôture, parfois une banalité vague), `IMPORTANT:` (parfois mérité, parfois de l'inflation), `voice.md` + `brand.md` (parfois redondants, parfois intentionnellement séparés), et la plupart des autres signaux de framework.

### 2.1 : Pour chaque framework F1, F2, F3, F4, F5, F6, G7, F8, F9 (dans cet ordre)

F8 s'exécute en avant-dernier (synthèse trans-fichiers). F9 s'exécute **en dernier** parce qu'il consomme le tableau structurel produit par F1–G7 et opère sur le plus grand rayon d'impact (raisonnement structurel sur tout le vault, génération de Plot.md, propositions de réorganisation). Les exécuter en dernier garde l'application des corrections ordonnée du plus petit au plus grand à l'étape 5.

**Mets à jour la tâche TaskCreate** du framework courant à `in_progress` avant de le démarrer ; `completed` quand ses constats sont journalisés. Sous-mises à jour en cours de framework autorisées pour les exécutions longues (F2, F8, F9).

Pour chaque framework :

1. **Lis le fichier d'implémentation de la passe** de ce framework (ex. `references/passes-anthropic-claude-md.md` pour F1). Mets-le en cache pour la durée de l'exécution du framework.
2. **Détermine le périmètre de fichiers** à partir du tableau en tête de ce SKILL.md (ex. F1 = chaque CLAUDE.md ; F6 = chaque SKILL.md ; F4/F5/G7 = chaque `.md`).
3. **Pour chaque vérification dans le fichier de passe** :
   - Applique l'**heuristique du déclencheur** (regex / métrique / motif structurel) pour faire remonter rapidement des candidats. Certaines vérifications n'ont pas de déclencheur : le fichier lui-même est le candidat.
   - Pour chaque candidat, **lis les 5 à 15 lignes environnantes** avec l'outil `Read`, puis applique les **critères de jugement de l'agent** que liste le fichier de passe. Lis d'autres fichiers (cibles liées, clusters voisins, l'index du fichier) quand le jugement l'exige.
   - Décide : ce cas viole-t-il réellement la règle du framework **dans le contexte spécifique de ce fichier** ? Ou est-ce un faux positif (le fichier de passe liste les plus courants à ignorer) ?
   - Si réel → produis un constat. Sinon → abandonne-le ; le déclencheur était un candidat, pas un verdict.
   - **Chaque constat inclut un champ `reasoning`** (1 à 2 phrases spécifiques à ce cas, pas une reformulation générique de la règle).
4. **Exécute les vérifications de portée vault** du framework (en utilisant les index de l'étape 1.3) : détection d'orphelins, wikilinks morts, paires de distracteurs, conformité de schéma, etc. : voir les sections « vault-wide » de chaque fichier de passe.
5. **Émets une ligne de progression** quand le framework se termine :

   > F1 Anthropic CLAUDE.md : 14 fichiers CLAUDE.md lus, 312 candidats jugés → 22 constats (5 fail · 17 warn)

   Le ratio `judged` vs `findings` est un contrôle de cohérence : s'ils sont à peu près égaux, l'exécution a été paresseuse (des candidats regex devenus constats sans jugement) ; si `judged` ≫ `findings`, le jugement filtre les faux positifs, ce qui est le comportement attendu.

### 2.2 : Contrôle de cohérence du raisonnement (par framework)

Après avoir terminé l'exécution de chaque framework, échantillonne 5 constats au hasard (ou tous les constats si < 5). Lis le champ `reasoning` de chacun. Si > 40 % des raisonnements échantillonnés sont des paraphrases de la règle plutôt qu'un jugement spécifique au cas (« This file uses too many em dashes » : paraphrase ; « These em dashes appear in callout headers where the writer was substituting for colons; replacing with colons preserves the cadence » : jugement), **arrête et relance ce framework avec des lectures plus approfondies**.

C'est un contrôle bloquant. La valeur du skill, c'est le jugement, pas le regex. Livrer les constats d'un framework sans appliquer ce contrôle est un bug.

### 2.3 : Schéma de constat (chaque framework, chaque chemin)

```json
{
  "framework": "F1",
  "check_id": "F1.2",
  "check_name": "Specificity heuristic",
  "path": "./Projects/foo/CLAUDE.md",
  "line": 42,
  "severity": "warn",
  "excerpt": "Be careful with auth",
  "reasoning": "This rule sits in the top half of a CLAUDE.md that's otherwise a routing index — there's no specific auth boundary, file path, or function named anywhere else. As a primary rule it falls into the 35%-compliance bucket; either anchor it to a specific path/function or remove it.",
  "action": "Either delete or rewrite as 'All /api/admin/* routes must call requireAdmin() from src/auth/middleware.ts'.",
  "fixable": false,
  "fixed": false,
  "citation": "anthropic-claude-md.md → Specificity beats vagueness"
}
```

Le champ `reasoning` est obligatoire. Chaque constat le possède.

**Chaque constat embarque une correction.** Le fichier de passe émet toujours `fixable: true`. Les anciens constats `fixable: false` « à revoir manuellement » ont disparu : les vérifications qui étaient historiquement en simple signalement (F1.x, F4.x, F5.x) embarquent désormais des propositions de correction en revue pas à pas, comme tout autre framework.

Le champ `fix_status` est défini par l'**étape 5** après que l'utilisateur a parcouru chaque constat. Valeurs possibles :
- `applied` → correction exécutée dans cette exécution (pastille verte FIXED)
- `saved_to_plan` → correction écrite dans `Intelligence/decisions/{date}-reorg.md` pour une exécution échelonnée (pastille bleue SAVED)
- `declined` → l'utilisateur a explicitement choisi de ne pas corriger cet item en revue (pastille grise DECLINED)
- `failed` → correction tentée mais échouée au contrôle de sécurité (pastille rouge FAILED, avec `failure_reason`)

Il n'y a pas d'état « skipped » ou « deferred ». Soit la correction a été appliquée, soit enregistrée comme étape planifiée, soit refusée explicitement item par item, soit elle a échoué mécaniquement. Les avertissements ouverts ne survivent pas à l'exécution.

---

## Étape 3 : Agréger les constats + calculer le score

Pour chaque framework F1–F8, compte : total des constats, répartition par sévérité (fail/warn/info), nombre de corrigeables, fichiers touchés.

### Formule de score (pondérée par framework)

```
For each framework F1..G7:
  deduction = (fail_count × 5) + (warn_count × 1)
  capped_deduction = min(deduction, 25)
score = max(0, 100 - sum(capped_deduction for F1..G7))
```

**F8 et F9 n'affectent pas le score.** F8 fait remonter des opportunités de synthèse ; F9 fait remonter des changements structurels/de découvrabilité. Tous deux sont des opportunités d'optimisation, pas des échecs de lint : les compter mélangerait « hygiène du vault » et « le vault pourrait être réorganisé ». Suis F8 dans une tuile « Insights surfaced / applied » et F9 dans une tuile « Architecture changes / proposed / applied / saved-to-plan ».

| Score | Interprétation |
|---|---|
| 90–100 | Bien réglé. Lancer l'audit chaque mois. |
| 70–89 | Dérive visible. Traiter les principaux constats. |
| 50–69 | Le surpoids nuit aux performances. |
| <50 | Pourrissement du vault. Nettoyage majeur nécessaire. |

Après le scoring, **ne rends pas un long résumé markdown dans le chat**. Émets un seul bloc court :

```
✅ Les 9 frameworks appliqués.

| Framework | Fichiers | Vérifications | Constats | Fail | Warn | Corrections proposées |
|---|---:|---:|---:|---:|---:|---:|
| F1 Anthropic CLAUDE.md | … | … | … | … | … | … |
| F2 Karpathy Wiki | … | … | … | … | … | … |
| F3 Caveman | … | … | … | … | … | … |
| F4 Chroma Context Rot | … | … | … | … | … | … |
| F5 Anthropic Memory | … | … | … | … | … | … |
| F6 Progressive Disclosure | … | … | … | … | … | … |
| G7 Hygiene | … | … | … | … | … | … |
| F8 Reflection | … | … | … | … | … | … |
| F9 Architecture | … | … | … | … | … | … |
| **TOTAL** | … | … | … | … | … | … |

Score : {score_before}/100 (F8 + F9 non scorés). Lecture architecturale à suivre…
```

---

## Étape 3.5 : Lecture architecturale (le paragraphe de synthèse)

Avant de parcourir les corrections, écris une courte lecture architecturale du vault : les 1 à 3 principaux problèmes de la structure dans son ensemble, avec un raisonnement ancré dans `Context/`. C'est la couche que l'utilisateur a explicitement demandée : l'agent démontre qu'il a compris ce que le vault signifie, pas seulement ce qu'il a échoué.

Format :

```markdown
### Lecture architecturale

{1 à 3 courts paragraphes. Chaque paragraphe : une observation structurelle, pourquoi elle compte pour le monde de *cet* utilisateur (cite Context/), le ou les constats F9 qui l'ont fait remonter, la direction proposée.}
```

Ce paragraphe va aussi mot pour mot dans le tableau de bord HTML (au-dessus des lignes de framework) afin d'être la première chose que l'utilisateur voit en ouvrant le rapport. Garde-le sous 250 mots au total : de la synthèse, pas un récapitulatif des constats.

Si l'agent ne peut identifier aucune observation structurelle digne d'être remontée (le vault est bien organisé) : écris une phrase le disant, en citant les métriques F9 qui justifient l'évaluation. Ne rembourre pas.

---

## Étape 4 : Parcourir chaque constat via appliquer-maintenant / enregistrer-dans-le-plan

**Chaque constat embarque une proposition de correction.** L'étape 4 guide l'utilisateur à travers elles. Il n'y a pas de « sauter ce framework », pas de « simple signalement », pas de « corriger plus tard ». Chaque constat se termine dans l'un de quatre états explicites : `applied`, `saved_to_plan`, `declined`, ou `failed`.

### 4.1 : Un seul AskUserQuestion d'ouverture (le verrou d'application)

Avant la revue, déclenche un seul `AskUserQuestion` :

> « Audit terminé : {N_total} constats sur 9 frameworks ({fail} fail · {warn} warn · {info} info). Chaque constat a une correction proposée. Choisissez le mode : »

Options :
1. **Application en masse (tout appliquer, aucune échappatoire)** : chaque constat est appliqué. Les invites de revue ne se déclenchent que là où l'agent a vraiment besoin que l'utilisateur choisisse une cible/formulation (ex. « cible de fusion : A ou B ? »). Tout le reste s'applique sans confirmation item par item. Le seul état non-appliqué valide est `failed` (mécanique : verrou OS, conflit de fichier, dépendance manquante). Pas de `saved_to_plan`, pas de `declined`. À utiliser quand vous avez dit « apply everything ».
2. **Revue sélective** : déclenche une invite par constat pour chaque correction ; l'utilisateur choisit appliquer / enregistrer-dans-le-plan / refuser par item. À utiliser pour une revue minutieuse ou une première exécution sur un nouveau vault.
3. **Tout enregistrer dans le plan** : écris toutes les corrections proposées comme étapes de checklist dans le dossier équivalent-décisions découvert ; aucune modification appliquée dans cette exécution.
4. **Annuler** : abandonne avant toute correction.

L'option 1 est le défaut. L'utilisateur ne choisit la 2 que lorsqu'il veut un verrou item par item. **L'instruction préalable de l'utilisateur « apply everything » correspond à l'option 1, pas à l'option 2.**

### 4.1.1 : Règles du mode application en masse (option 1)

Règles strictes quand ce mode est actif :

- **Aucun résultat `saved_to_plan`.** Cet état est retiré de l'ensemble `fix_status` disponible.
- **Aucun résultat `declined`.** Idem.
- **L'agent ne décide pas unilatéralement qu'un constat est « test fixture », « l'utilisateur ne voulait probablement pas ça », « trop complexe à appliquer mécaniquement » ou « jugement requis par section » et ne le route vers rien d'autre que applied/failed.** Si appliquer une correction mécaniquement endommageait le contenu, l'agent tente la version mécanique la plus sûre (ex. découper un fichier de 222KB aux frontières de H2 avec un marqueur `<!-- automated split, review headings -->` ; marquer le découpage comme `applied` avec `applied_with_caveat: true`).
- **Les invites de revue ne se déclenchent que lorsque l'agent ne peut vraiment pas choisir une cible sans l'utilisateur.** Exemple : fusion F8.2 avec trois cibles canoniques plausibles → déclenche l'invite. Exemple : substitution de tiret cadratin G7.1 → applique sans inviter ; le raisonnement par occurrence de l'agent a déjà choisi la substitution.
- **Les échecs mécaniques sont toujours enregistrés comme `failed`** avec `failure_reason` (fichier verrouillé par l'OS, dossier parent manquant, conflit de fichier, contrôle de sécurité du framework qui a annulé la correction). Les corrections échouées sont signalées dans le tableau de bord avec leur raison : elles ne sont pas silencieusement ignorées.
- **L'agent ne doit pas préclasser des constats comme « à sauter en mode application en masse ».** Chaque constat est dans le périmètre. Si appliquer une correction de fixture de test endommage un motif de test délibéré, l'utilisateur peut annuler ce fichier précisément, mais l'agent ne prend pas cette décision.

### 4.2 : Boucle de revue par constat

Ordre : F1 → F2 → F3 → F4 → F5 → F6 → G7 → F8 → F9. Au sein de F8/F9, le plus petit rayon d'impact d'abord (par fichier de passe).

Pour chaque constat :

1. **Montre le constat de façon compacte** dans le chat : chemin, sévérité, extrait d'une ligne, le `proposed_edit` ou l'étape de migration de la correction proposée.
2. **Déclenche AskUserQuestion** avec la sous-invite propre au type de constat :

| Type de constat | Options de la sous-invite |
|---|---|
| **Mécanique** (G7.1 tirets cadratins, G7.3/F1.10 H1 dupliqué) | Appliquer maintenant / Refuser cet item / Appliquer tous les restants de ce type sans demander |
| **Repointage de wikilink** (F2.2) | Appliquer avec la cible [X] / Choisir une autre cible / Refuser |
| **Ajout de référence croisée** (F2.4) | Appliquer / Modifier la liste des fichiers liés / Refuser |
| **Substitution Caveman** (F3.1–F3.4 par fichier) | Tout appliquer dans ce fichier / Parcourir les substitutions de ce fichier / Refuser ce fichier |
| **Déduplication skill-vault** (F6.11) | Appliquer / Refuser |
| **Édition de CLAUDE.md** (F1.x) | Appliquer l'édition rédigée / Modifier le texte d'abord puis appliquer / Enregistrer dans le plan / Refuser |
| **Découpage / renommage mémoire** (F5.1, F5.3) | Appliquer avec les points de découpe ou le nom de fichier proposés / Modifier puis appliquer / Enregistrer dans le plan / Refuser |
| **Réorganisation context-rot** (F4.3, F4.4, F4.6, F4.7) | Appliquer le réordonnancement rédigé / Modifier le texte / Enregistrer dans le plan / Refuser |
| **Reflection : contradiction** (F8.1) | Gagnant : [A] / [B] / [aucun : déférer à une troisième source] → Appliquer la réécriture au perdant / Modifier le texte / Refuser |
| **Reflection : fusion** (F8.2) | Canonique : [A] / [B] / [C] → Confirmer fusion + redirection + archivage / Enregistrer dans le plan / Refuser |
| **Reflection : obsolète** (F8.3) | Appliquer la réécriture / Modifier le texte / Refuser |
| **Reflection : thème** (F8.4) | Créer à [proposed path] / Choisir un autre chemin / Modifier le contenu d'abord / Refuser |
| **Reflection : promotion** (F8.5) | Destination : [proposed] / [choisir autre] → Appliquer l'ajout + laisser un stub wikilink / Refuser |
| **Réécriture de routage** (F9.1) | Appliquer l'entrée rédigée / Modifier le texte / Refuser |
| **Génération de Plot.md** (F9.2) | Appliquer le Plot.md rédigé / Modifier la ligne Purpose + les descriptions d'abord / Refuser |
| **Correction de découvrabilité** (F9.3) | Ajouter au Plot.md / Ajouter une entrée de routage / Déplacer le fichier / Archiver / Refuser |
| **Fichier mal placé** (F9.4) | Déplacer vers [proposed folder] / Choisir un autre dossier / Élargir plutôt le Plot.md du dossier courant / Refuser |
| **Duplication de dossier** (F9.5) | Fusionner dans [A] / Fusionner dans [B] / Clarifier les finalités des Plot.md des deux / Enregistrer dans le plan / Refuser |
| **Proposition de réorganisation** (F9.6) | Appliquer maintenant (parcourir les étapes de migration) / Enregistrer dans le plan (écrire dans {date}-reorg.md) / Refuser |
| **Lacune d'orientation** (F9.7) | Appliquer l'édition CLAUDE.md rédigée / Modifier le texte d'abord / Enregistrer dans le plan / Refuser |

Chaque option atteint un état terminal pour ce constat. **Il n'y a pas d'option « skip » qui laisse le constat ouvert.** « Decline » est enregistré comme une décision explicite de l'utilisateur et montré dans le rapport : c'est une décision, pas un report.

3. **Mets à jour le `fix_status` du constat** (`applied` / `saved_to_plan` / `declined`) et tout `per_item_target` choisi par l'utilisateur.
4. **TaskUpdate** sur la tâche du framework courant avec une note de progression `{i}/{N}` toutes les 10 constats.

### 4.3 : Format du fichier enregistrer-dans-le-plan

Résolution du chemin d'enregistrement dans le plan (utilise le registre des rôles) :
1. Si `roles.decisions.path` est défini → enregistre sous `{decisions.path}/{YYYY-MM-DD}-reorg.md`.
2. Sinon, si un dossier correspond à `*decisions*/`, `*logs*/`, `*records*/` (insensible à la casse) → utilise-le.
3. Sinon, crée `audits/` à la racine du vault et enregistre-y. Note le rôle decisions manquant pour F9.0 : l'utilisateur reçoit une proposition de correction pour formaliser un dossier équivalent-décisions.

Quand des constats sont routés vers enregistrer-dans-le-plan, ajoute des entrées de checklist au chemin résolu :

```markdown
---
status: pending
type: reorg-plan
tags: [optimizer, plan]
date: 2026-05-08
source: os-optimizer run 2026-05-08T14:23:00Z
---

# Plan de réorganisation du vault : 2026-05-08

Généré par os-optimizer. Chaque item est une proposition de correction enregistrée pour une exécution échelonnée. Parcourez-les quand vous êtes prêt.

## F9.6 Propositions de réorganisation

- [ ] **Fusionner `Notes/` dans `Resources/notes/`** : cluster de constats F9.5, les deux dossiers contiennent du brouillon avec 38 % de recouvrement de noms de fichiers. Migration :
  - Déplacer 12 fichiers de `Notes/` vers `Resources/notes/`.
  - Repointer 4 wikilinks entrants.
  - Mettre à jour l'entrée de routage du root CLAUDE.md pour retirer `Notes/`.
  - Supprimer le répertoire `Notes/` vide.
  - Raisonnement : le Plot.md de `Resources/notes/` indique qu'il contient des notes de référence ; le Plot.md de `Notes/` indique la même chose. Selon `Context/me.md`, la convention déclarée par l'utilisateur est une seule bibliothèque de ressources : en garder deux est une dérive structurelle.

## F8.2 Candidats à la fusion

- [ ] **Fusionner `Notes/2026-q1-strategy.md` et `Projects/positioning/research/strategy-thoughts.md` dans `Context/strategy.md`** : constat F8.2. Étapes : …

(se poursuit par constat enregistré)
```

Le fichier de plan est lui-même audité à la prochaine exécution : les conventions F1/G7 s'appliquent.

### 4.4 : Manquement à la revue

Si l'agent saute un constat (oublie de déclencher sa sous-invite, regroupe plusieurs constats dans une seule invite, ou traite « aucune réponse » comme un refus), c'est un bug. Parcours chaque constat individuellement. La liste de tâches visible attrape cela : si la tâche d'un framework passe à `completed` alors que des constats restent non parcourus, l'étape de vérification en 5.7 le signalera.

---

## Étape 5 : Appliquer les corrections approuvées (définir `fix_status` par constat)

Applique dans cet ordre (le plus petit rayon d'impact d'abord). Pour chaque constat, définis `fix_status` à `applied`, `saved_to_plan`, `declined` ou `failed` selon ce qui s'est passé à l'étape 4 + cette étape. Le rendu HTML utilise directement `fix_status`.

**Les items enregistrer-dans-le-plan** sont écrits dans `Intelligence/decisions/{date}-reorg.md` à l'étape 4.3 : cette étape (5) ne traite que les items pour lesquels l'utilisateur a choisi « appliquer maintenant ». (En mode application en masse, il n'y a pas d'items enregistrer-dans-le-plan.)

### 5.0 : Capturer les métriques AVANT (obligatoire, avant toute correction)

Avant d'appliquer toute correction, fais un instantané des métriques par rôle et par framework dans le cache `before_metrics`. Ce sont les valeurs contre lesquelles s'affichent les colonnes avant/après du tableau de bord.

Métriques par rôle (une ligne par rôle dans le registre, plus une ligne TOTAL) :
- `file_count`
- `total_chars` (somme des octets du corps, après frontmatter)
- `total_tokens_est` (caractères / 4)
- `em_dash_count` (sortie du déclencheur G7.1)
- `frontmatter_complete_pct` (fichiers avec `status:` + ≥2 `tags:` / total)
- `wikilink_orphan_count` (aucun wikilink entrant)
- `index_coverage_pct` (dossiers avec le fichier d'index de dossier découvert / nombre de dossiers non triviaux)
- `findings_open` (nombre de constats touchant des fichiers de ce rôle)

Métriques par framework (F1–F9 + TOTAL) :
- `findings_count`
- `fail_count`, `warn_count`, `info_count`
- `applied_count` (commence à 0)
- `failed_count` (commence à 0)
- `score_contribution` (selon la formule de l'étape 3 ; F8/F9 toujours 0)

Mets-les en cache. Applique les corrections (5.1–5.8). Puis exécute 5.9.

### 5.1 : Tirets cadratins (G7.1)
- Pour chaque constat `applied` : remplace `—` → `. `, `–` → `, ` dans le corps détouré uniquement.
- Re-détoure les zones protégées après ; si du code/URL/wikilink a été touché → annule la modification de ce fichier ET définis `fix_status: failed` avec `failure_reason`.
- Utilise `Edit` avec `replace_all: true` par fichier.
- Définis `fix_status: applied` sur chaque constat G7.1 dont le fichier a été corrigé avec succès.

### 5.2 : H1 dupliqué (G7.3 / F1.10)
- Supprime la ligne H1 et toute ligne vide qui la suit immédiatement.
- Utilise `Edit`. Définis `fix_status: applied`.

### 5.3 : Corrections de wikilink (F2.2)
- Pour chaque constat approuvé par l'utilisateur (en masse = suggestion top-1, revue = cible choisie) : remplace `[[Old Target]]` par le remplacement choisi.
- Préserve le texte environnant octet par octet.
- Marque `fixed: true` par constat corrigé ; `fixed: false` pour ceux sautés ou à faible confiance que l'utilisateur a refusés.

### 5.4 : Réécritures skill-vault (F6.11)
- Mets à jour le SKILL.md pour lire le chemin `Context/` du vault au lieu de la référence dupliquée.
- Grep le reste du dossier de skill à la recherche du nom de fichier de la référence dupliquée.
- Si toujours référencée → fais remonter le conflit, saute la suppression, définis `fix_status: failed`.
- Sinon → `rm` le doublon, définis `fix_status: applied`.

### 5.5 : Substitutions Caveman (F3.1, F3.2, F3.3, F3.4) : uniquement si l'utilisateur a opté pour, par fichier
- Applique la table de substitution de `references/passes-caveman.md` au corps détouré.
- Re-détoure les zones protégées ; si l'une a été modifiée → abandonne la correction sur ce fichier, signale, définis `fix_status: failed` pour les constats de ce fichier.
- Marque `fixed: true` sur chaque constat F3.x pour les fichiers où l'agent a substitué avec succès.

### 5.6 : Corrections Reflection (F8.1, F8.3, F8.2, F8.4, F8.5) : revue uniquement, cibles confirmées par l'utilisateur item par item

Applique les corrections F8 **en dernier** (plus grand rayon d'impact). L'ordre au sein de F8 va du plus petit au plus grand :

1. **F8.1 contradictions** : pour chaque constat approuvé, réécris le fichier perdant. Soit remplace la ligne contredite par `See [[Winner]] for current state.` (mode report), soit supprime la ligne entièrement (mode suppression) ; l'utilisateur choisit par item à l'étape 4. Utilise `Edit`. Définis `fix_status: applied`.
2. **F8.3 entrées obsolètes** : pour chaque constat approuvé, réécris l'entrée curatée avec le nouvel état. Ajoute une section `## History` avec l'ancienne formulation si elle a un contexte de décision (selon la règle du fichier de passe). Ajoute un wikilink vers la source qui supplante. Définis `fix_status: applied`.
3. **F8.2 candidats à la fusion** : pour chaque cluster approuvé :
   - Concatène les sections uniques des sources dans la cible canonique (saute les paragraphes dupliqués).
   - **Revérifie la taille contre le budget F5.** Si le fichier fusionné > 10KB → abandonne la fusion, définis `fix_status: failed`, fais remonter en simple signalement avec la raison de l'abandon dans `reasoning_post`.
   - Grep le vault pour les motifs `[[SourceName]]` et `[[SourceName|alias]]` ; remplace chacun par `[[CanonicalName|alias]]` (ou `[[CanonicalName]]`).
   - Vérifie zéro nouveau wikilink mort (exécute le déclencheur F2.2 sur chaque fichier touché). Si un nouveau lien mort apparaît → annule la correction de ce constat, définis `fix_status: failed`.
   - Déplace les fichiers source vers `Intelligence/archive/{YYYY-MM-DD}-merged/`. Utilise `Bash mv`.
   - Définis `fix_status: applied`.
4. **F8.4 thèmes émergents** : pour chaque constat approuvé :
   - Crée le nouveau fichier au chemin choisi par l'utilisateur (par défaut `Context/{theme-slug}.md` ou `Resources/{theme-slug}-MOC.md`).
   - Le frontmatter doit inclure `status:`, `tags:`, `type:`, `date:` (conformité G7.2) et ne contenir aucun tiret cadratin (G7.1).
   - Corps : définition d'une ligne, 3 à 5 points clés, wikilinks de retour vers chaque note source du cluster.
   - Définis `fix_status: applied`.
5. **F8.5 promotions** : pour chaque constat approuvé :
   - Ajoute le contenu durable à la destination choisie (`Context/`, `Resources/`, ou `Intelligence/decisions/`).
   - Dans le fichier source, remplace les lignes originales par `See [[Target#section]]`.
   - Définis `fix_status: applied`.

**Contrôle de sécurité trans-framework après chaque correction F8 :**
- Re-détoure les zones protégées sur chaque fichier écrit. Si du code/URL/wikilink/chemin/frontmatter a été modifié hors de l'édition prévue → annule, définis `fix_status: failed`.
- Si le fichier touché vise un CLAUDE.md ou claude.md → abandonne (cela aurait dû être attrapé à l'étape 4 comme simple signalement, mais défense en profondeur).
- Si la taille du fichier touché dépasse désormais le budget recommandé F5 de 10KB → consigne la violation dans `reasoning_post` ; la prochaine exécution d'audit la fera remonter comme simple signalement F5 (n'annule pas, mais journalise).

### 5.7 : Corrections Architecture (F9.1, F9.2, F9.3, F9.4, F9.5, F9.6, F9.7) : revue uniquement, cibles confirmées par l'utilisateur item par item

Applique les corrections F9 après F8. Ordre : le plus petit rayon d'impact d'abord.

1. **F9.1 réécritures de routage** : pour chaque constat `applied`, `Edit` la table de routage du root CLAUDE.md avec la/les ligne(s) confirmée(s) par l'utilisateur. Item par item uniquement ; pas de réécritures par lot. C'est le seul chemin légitime pour éditer un CLAUDE.md (approbation utilisateur item par item).
2. **F9.2 génération / régénération de Plot.md** : pour chaque constat `applied` :
   - Si absent : `Write` le Plot.md rédigé par l'agent (l'utilisateur a déjà confirmé Purpose + descriptions des enfants en revue).
   - Si obsolète : `Edit` pour mettre à jour la liste Children ; préserve toute édition utilisateur de Purpose si non dépréciée.
   - Vérifie taille ≤ 8KB ; vérifie G7.1 (aucun tiret cadratin) et G7.2 (frontmatter complet).
3. **F9.3 corrections de découvrabilité** : pour chaque constat `applied`, applique la réparation choisie par l'utilisateur :
   - « Add to Plot.md » → `Edit` le Plot.md parent.
   - « Add routing entry » → `Edit` le routage du root CLAUDE.md.
   - « Move file » → `Bash mv` ; repointe les wikilinks entrants (même mécanique que F8.2).
   - « Archive » → `Bash mv` vers `Intelligence/archive/`.
4. **F9.4 fichiers mal placés** : `Bash mv` vers le dossier choisi par l'utilisateur. Mets à jour le Plot.md du dossier source (retire l'enfant) et le Plot.md du dossier cible (ajoute l'enfant). Repointe les wikilinks entrants.
5. **F9.5 duplication de dossier** : pour les constats « merge into », parcours les déplacements de fichiers un par un (chacun est sa propre micro-étape `applied`). Pour les constats « clarify Plot.md purposes » : `Edit` les deux Plot.md.
6. **F9.6 propositions de réorganisation** : présent ici uniquement si l'utilisateur a choisi « apply now » ; ceux en enregistrer-dans-le-plan sont déjà dans le fichier de réorganisation daté. Pour appliquer-maintenant : parcours chaque étape de migration de la proposition comme son propre micro-constat (déplacer X, renommer Y, mettre à jour l'entrée de routage Z), chacun confirmé individuellement avant exécution.
7. **F9.7 lacune d'orientation** : `Edit` le root CLAUDE.md avec la section rédigée par l'agent que l'utilisateur a confirmée. Item par item uniquement.

**Contrôle de sécurité trans-framework après chaque correction F9 :**
- Relance le déclencheur F2.2 sur chaque fichier touché. Si de nouveaux wikilinks morts sont apparus → annule cette correction, définis `fix_status: failed`.
- Revérifie la taille du root CLAUDE.md contre le budget F1.1 après les éditions F9.1 / F9.7. Au-dessus du budget → consigne dans `failure_reason` ; la prochaine exécution d'audit signalera F1.1.
- Les écritures de Plot.md doivent passer G7.1 + G7.2 + F5 (≤ 8KB) par construction ; si une vérification échoue → annule, `fix_status: failed`.

### 5.8 : Lanceur de correction générique item par item (F1.x, F4.x, F5.x : auparavant en simple signalement)

Pour les constats dont les vérifications n'avaient historiquement pas de procédure de correction spécialisée (l'essentiel de F1, F4, F5), le lanceur est générique :

1. Lire le fichier.
2. Appliquer le `proposed_edit` rédigé par l'agent depuis le constat (l'utilisateur l'a déjà confirmé ou édité à l'étape 4.2).
3. Re-détourer les zones protégées ; si du code/URL/chemin/frontmatter hors de l'édition prévue a été modifié → annuler, `fix_status: failed`.
4. Si le fichier est un CLAUDE.md → ne procéder que si `applied` est passé par une revue-avec-confirmation-explicite (étape 4.2). Ne jamais appliquer en masse des éditions de CLAUDE.md via ce lanceur.
5. Définir `fix_status: applied` en cas de succès.

Ce lanceur gère, par exemple :

- F1.2 réécriture ou suppression de règle vague
- F1.5 suppression de banalité
- F1.7 réordonnancement pour l'effet de position
- F4.3 réordonnancement d'information critique
- F4.4 suppression du préambule d'amorce
- F5.1 découpage de fichier (découper le fichier aux en-têtes suggérés par l'agent ; l'utilisateur a confirmé les points de découpe en revue)
- F5.3 renommage de fichier via `Bash mv` + repointage des wikilinks entrants

La sous-procédure de correction peut varier selon le type de constat : ce qui reste constant, c'est : lire → appliquer l'édition confirmée → contrôle de sécurité → statut. La consigne par vérification du fichier de passe contraint toujours ce que l'agent rédige.

### 5.9 : Capturer les métriques APRÈS (obligatoire, après chaque correction)

Une fois les étapes 5.1–5.8 terminées, re-mesure les mêmes métriques capturées en 5.0. C'est la source de vérité pour les colonnes APRÈS du tableau de bord et le score recalculé. Sauter cette étape est le bug qui a produit le rendu 46/100 en AVANT seulement : ne la saute jamais.

Re-mesure :
- Métriques par rôle (relance les mêmes sondes que 5.0 : des fichiers ont pu être découpés, des tirets cadratins supprimés, du frontmatter ajouté).
- Métriques par framework (recompte les constats contre l'état *courant* du vault : de nombreux constats devraient maintenant se résoudre à zéro parce que leur problème sous-jacent a été corrigé).
- **Recalcule le score** avec la formule de l'étape 3, contre le nombre de constats re-mesuré. C'est `{{SCORE_AFTER}}`. Ne réutilise pas le score AVANT.
- Calcule les deltas : `tokens_saved_per_role`, `em_dashes_removed`, `frontmatter_pct_delta`, `findings_resolved_per_framework`.

Si la re-mesure montre une catégorie de constat dont le nombre a *augmenté* (ex. une correction a accidentellement ajouté de nouveaux orphelins), fais remonter cela comme une ligne d'avertissement de régression dans le tableau de bord : la correction a réussi mécaniquement mais a produit un problème en aval.

La re-mesure devrait prendre quelques secondes, pas quelques minutes : mêmes sondes que l'étape 1 + les déclencheurs de framework, mais seulement sur les fichiers touchés par l'étape 5 plus les petits décomptes globaux (tirets cadratins sur tout le vault, couverture du frontmatter). Mets en cache pour l'étape 6.

### 5.10 : Vérifier que les totaux de fix-status s'additionnent

Après application :
- `total_applied` = constats avec `fix_status: applied`.
- `total_saved_to_plan` = constats avec `fix_status: saved_to_plan`.
- `total_declined` = constats avec `fix_status: declined`.
- `total_failed` = constats avec `fix_status: failed`.

Ces quatre devraient égaler `total_findings`. **Aucun `fix_status` `null` ou non défini n'est permis** : chaque constat a atteint l'étape 4 et a été parcouru. Si un constat a un `fix_status` non défini après l'étape 5, l'orchestrateur a sauté une revue ; arrête et fais remonter le bug à l'utilisateur avant le rendu. (Le HTML se rend quand même correctement à partir des flags par constat, mais le constat sauté signale un bug à l'étape 4.4.)

### Gestion des échecs
Si une correction échoue (fichier manquant, conflit d'édition, correspondance ambiguë, violation de zone protégée) → arrête, signale quel item a échoué, demande à l'utilisateur comment procéder. Ne saute jamais silencieusement un échec.

### Toujours préserver
- Les fichiers de la liste d'exclusion technique (`.git`, `.obsidian`, `.trash`, `node_modules`, `dist`, `build`).
- Les blocs de code, le code inline, les URLs, les chemins de fichier, les clés de frontmatter, les wikilinks, les délimiteurs de tableau, les titres, les dates, les numéros de version.

---

## Étape 6 : Rendre, enregistrer, ouvrir et faire remonter le HTML

### 6.1 : Calculer les métriques par framework

Instantané avant/après par framework :

| Variable | Source |
|---|---|
| `{{F1_FILES}}`, `{{F1_CHECKS}}`, `{{F1_FINDINGS_BEFORE}}`, `{{F1_FINDINGS_AFTER}}`, `{{F1_FAIL_BEFORE}}`, `{{F1_FAIL_AFTER}}`, `{{F1_WARN_BEFORE}}`, `{{F1_WARN_AFTER}}`, `{{F1_INFO_BEFORE}}`, `{{F1_INFO_AFTER}}`, `{{F1_APPLIED}}`, `{{F1_FAILED}}`, `{{F1_DETAILS}}` | Bucket F1 : chaque décompte a une paire avant/après pour que le tableau de bord montre le delta |
| idem pour F2, F3, F4, F5, F6, G7, F8, F9 | chaque bucket |
| `{{ARCHITECTURAL_READ}}` | Paragraphe de l'étape 3.5, rendu au-dessus des lignes de framework |
| `{{F8_INSIGHTS_SURFACED}}`, `{{F8_INSIGHTS_APPLIED}}` | F8 uniquement : alimente la tuile « Insights surfaced / applied » |
| `{{F9_PROPOSALS}}`, `{{F9_APPLIED}}`, `{{F9_FAILED}}` | F9 uniquement : alimente la tuile « Architecture changes » (en mode application en masse, il n'y a pas de colonne Saved/Declined) |
| `{{ROLE_METRICS_TABLE}}` | tableau avant/après par rôle (étape 6.1.5) |
| `{{FRAMEWORK_METRICS_TABLE}}` | tableau avant/après par framework (étape 6.1.5) |
| `{{REORG_PLAN_PATH}}` | chemin du plan de réorganisation enregistré s'il existe des items enregistrer-dans-le-plan (revue sélective uniquement) ; vide en mode application en masse |
| `{{TOTAL_FIXABLE}}` | nombre de tous les constats avec `fixable: true` |
| `{{TOTAL_FIXED}}` | nombre de tous les constats avec `fixed: true` |
| `{{TOTAL_FIXABLE_NOT_FIXED}}` | nombre de constats avec `fixable: true && fixed: false` (sautés/refusés) |
| `{{TOTAL_BEFORE}}`, `{{TOTAL_AFTER}}`, `{{TOTAL_SAVED}}`, `{{TOTAL_PCT}}` | somme sur tous les frameworks |
| `{{SCORE_BEFORE}}`, `{{SCORE_AFTER}}`, `{{SCORE_DELTA}}`, `{{SCORE_DELTA_SIGN}}` | formule de score avant et après |
| `{{SESSION_BEFORE}}`, `{{SESSION_AFTER}}`, `{{SESSION_PCT}}` | nombre de tokens du root CLAUDE.md avant et après |
| `{{ANNUAL_SAVINGS}}` | `{{TOTAL_SAVED}} × SESSIONS_PER_WEEK × WEEKS_PER_YEAR`, suffixe K/M |
| `{{SESSIONS_PER_WEEK}}` (défaut 50) | surcharge utilisateur si spécifiée |
| `{{WEEKS_PER_YEAR}}` (défaut 50) | — |
| `{{FILES_SCANNED}}`, `{{FOLDERS_COVERED}}` | depuis l'étape 1 |
| `{{ORG_NAME}}` | titre de `Context/business.md` ou `Context/organization.md` ou frontmatter `name:` ; repli sur le nom du dossier cwd |
| `{{DATE}}`, `{{TIMESTAMP}}` | aujourd'hui (YYYY-MM-DD) et ISO UTC |

Pour le `{{Fx_DETAILS}}` de chaque framework : rends la liste des constats en HTML (pastilles de sévérité, chemins, extraits, actions, citation du framework). Plafonne aux 25 premiers constats par framework : s'il y en a plus, ajoute « et N autres signalés dans {decisions-folder}/{date}-vault-audit-findings.json ».

Formate les entiers avec des séparateurs de milliers. Formate les pourcentages à une décimale.

### 6.1.5 : Tableaux avant/après par rôle et par framework

Ces deux tableaux sont la preuve phare du tableau de bord que l'exécution a changé le vault, pas seulement décrit. Rends les deux au-dessus des sections de détail des frameworks (entre la lecture architecturale et les lignes de framework).

**Tableau par rôle** (`{{ROLE_METRICS_TABLE}}`) : une ligne par rôle dans le registre plus une ligne TOTAL :

| Rôle | Fichiers | Tokens avant → après | Tirets cadratins avant → après | Frontmatter complet avant → après | Couverture d'index avant → après | Constats ouverts avant → après |
|---|---:|---:|---:|---:|---:|---:|
| identity | 6 | 4,200 → 4,150 (-50) | 12 → 0 (-12) | 17% → 100% (+83pp) | — | 6 → 0 |
| context | 10 | 12,400 → 11,890 (-510) | 23 → 0 (-23) | 80% → 100% (+20pp) | 100% → 100% | 5 → 0 |
| projects | 14 | 8,200 → 8,200 (0) | 0 → 0 | 71% → 100% (+29pp) | 0% → 100% (+100pp) | 4 → 0 |
| daily | 241 | 145,000 → 145,000 (0) | 18 → 0 | 95% → 100% | — | 18 → 0 |
| meetings | 611 | 412,000 → 412,000 | 89 → 0 | 92% → 100% | — | 89 → 0 |
| (etc.) | | | | | | |
| **TOTAL** | 1,710 | 2.1M → 1.9M (-200K) | 289 → 0 | 67% → 100% | 12% → 100% | 502 → 12 |

Stylise chaque cellule de delta en vert pour les améliorations, rouge pour les régressions, gris pour l'inchangé. Les colonnes tiret cadratin + frontmatter abandonnent « before → after » quand les deux valent 0.

**Tableau par framework** (`{{FRAMEWORK_METRICS_TABLE}}`) : une ligne par framework F1–F9 plus TOTAL :

| Framework | Constats avant → après | Fail avant → après | Warn avant → après | Info avant → après | Appliqués | Échecs |
|---|---:|---:|---:|---:|---:|---:|
| F1 Anthropic CLAUDE.md | 9 → 0 | 2 → 0 | 6 → 0 | 1 → 0 | 9 | 0 |
| F2 Karpathy Wiki | 4 → 0 | 1 → 0 | 2 → 0 | 1 → 0 | 4 | 0 |
| F3 Caveman | 2 → 0 | 1 → 0 | 1 → 0 | 0 → 0 | 2 | 0 |
| F4 Chroma | 1 → 0 | 0 → 0 | 1 → 0 | 0 → 0 | 1 | 0 |
| F5 Memory | 3 → 1 | 2 → 0 | 1 → 1 | 0 → 0 | 2 | 1 |
| F6 Progressive Disclosure | 3 → 0 | 1 → 0 | 2 → 0 | 0 → 0 | 3 | 0 |
| G7 Hygiene | 3 → 0 | 1 → 0 | 1 → 0 | 1 → 0 | 3 | 0 |
| F8 Reflection | 4 → 0 | 1 → 0 | 2 → 0 | 1 → 0 | 4 | 0 |
| F9 Architecture | 5 → 0 | 1 → 0 | 1 → 0 | 3 → 0 | 5 | 0 |
| **TOTAL** | 34 → 1 | 9 → 0 | 17 → 1 | 8 → 0 | 33 | 1 |

Ligne de score, séparément, avec before → after rendu de façon proéminente :

```
Score (F1–G7) : 46 → 97 (+51) · « Pourrissement du vault » → « Bien réglé »
```

Ces tableaux font la différence entre « voici ce que nous devons faire » (ce qu'auraient été enregistrer-dans-le-plan/refusé) et « voici ce qui a été fait » (ce que produit le mode application en masse). Le tableau de bord présente l'exécution de manière rétrospective.

### 6.2 : Construire le HTML

Templates (lus une fois, substitués plusieurs fois) :

| Fichier | Utilisé comme |
|---|---|
| `references/report-template.html` | coquille principale |
| `references/report-row-template.html` | une ligne par framework dans le tableau de synthèse |
| `references/report-section-template.html` | une section de détail par framework |
| `references/report-finding-template.html` | une carte par constat dans une section de détail |

Étapes :
1. Lis les quatre templates.
2. Pour chaque framework F1..F9 :
   - Rends une ligne depuis `report-row-template.html` avec les métriques du framework → ajoute à l'accumulateur `{{ROWS}}`.
   - Pour chaque constat (plafonné aux 25 premiers par framework, classés : corrigés d'abord, puis fail avant warn, puis par check ID) : rends une carte depuis `report-finding-template.html` → ajoute au `{{FINDINGS_HTML}}` de cette section. **Échappe le HTML** de `EXCERPT`, `REASONING` et `ACTION` (`<`, `>`, `&`, `"`). Le champ `REASONING` est obligatoire : si le constat n'a pas de texte de raisonnement, le skill est buggé ; échoue bruyamment.
   - Calcule `{{STATUS_PILL}}` par constat depuis `fix_status` :
     - `applied` → `<span class="pill fixed">CORRIGÉ CETTE EXÉCUTION</span>`
     - `saved_to_plan` → `<span class="pill saved">ENREGISTRÉ DANS LE PLAN</span>` (lie vers `{{REORG_PLAN_PATH}}`)
     - `declined` → `<span class="pill declined">REFUSÉ</span>`
     - `failed` → `<span class="pill failed">ÉCHEC</span>` (avec `failure_reason` montré en ligne)
   - Si 0 constat : laisse `{{FINDINGS_HTML}}` vide et substitue `{{CLEAN_STATE}}` par `<div class="clean">✅ Toutes les vérifications sont passées. Aucun constat pour ce framework.</div>`.
   - Si total des constats > 25 : substitue `{{MORE_NOTE}}` par `<p class="more">…et {N} autres constats journalisés dans {YYYY-MM-DD}-vault-audit-findings.json.</p>`. Sinon laisse vide.
   - Rends la section depuis `report-section-template.html` → ajoute à l'accumulateur `{{DETAILS}}`.
3. Substitue `{{ROWS}}` et `{{DETAILS}}` dans le template principal.
4. Substitue chaque autre `{{PLACEHOLDER}}` (statistiques d'en-tête, score, impact de session, etc.).
5. **Passe de cohérence :** scanne le HTML rendu à la recherche de tout `{{...}}` ou `{{ }}` restant. S'il en reste → échoue bruyamment, n'enregistre jamais un fichier à moitié rendu.

Le `{{FRAMEWORK_WHY}}` de chaque section provient de la « Core thesis » de la référence du framework correspondant : un court paragraphe au maximum. Chaînes fixes :

- F1 : "Recommandation Anthropic : garder CLAUDE.md comme le plus petit ensemble concret d'instructions qui survit au pruning test. Les règles spécifiques obtiennent ~89 % de conformité ; les règles vagues ~35 %."
- F2 : "Karpathy LLM Wiki : la connaissance est compilée une fois et tenue à jour. Le lint attrape les liens morts, les orphelins, les contradictions, les références croisées manquantes et les sources non digérées."
- F3 : "Caveman compression : chaque token rivalise pour l'attention. Élague le remplissage, les atténuations, les politesses et les connecteurs verbeux de la couche d'instructions destinée à l'agent."
- F4 : "Chroma context rot : tout modèle se dégrade avec la longueur ; les distracteurs nuisent ; la position compte. Commence par la règle porteuse et garde le budget de chargement automatique serré."
- F5 : "Anthropic Memory : par fichier ≤100KB / ~25K tokens (recommandé <10KB). Plusieurs fichiers ciblés valent mieux qu'un méga-fichier. L'agent navigue par le nom."
- F6 : "Progressive Disclosure : la fenêtre de contexte est un bien public. Les métadonnées de skill se chargent toujours ; les corps selon la pertinence ; les références à la demande. Garde les références à un saut de profondeur."
- G7 : "Règles projet et notes de terrain de praticiens : discipline des tirets cadratins, conformité du frontmatter, pas de H1 dupliquant le nom de fichier, hygiène des README."
- F8 : "Reflection (inspiré d'Anthropic Dreams) : le lint par fichier ne peut pas voir les contradictions, les doublons, les hypothèses obsolètes ou les thèmes émergents. La synthèse à travers la couche curatée plus les sessions récentes les fait remonter, et propose des corrections concrètes que l'utilisateur approuve item par item."
- F9 : "Architecture & Discoverability : parcourir le chemin que Claude collègue emprunte réellement (root CLAUDE.md → routing → Plot.md de dossier → fichier). Vérifier que les entrées de routage correspondent à la réalité des dossiers, que chaque dossier a un Plot.md à jour, que chaque fichier est atteignable en ≤ 3 sauts, et que CLAUDE.md oriente réellement un agent dans le monde spécifique de cet utilisateur."

### 6.3 : Enregistrer (utilise le registre des rôles, pas de chemins codés en dur)

Choisis le dossier d'enregistrement via le registre des rôles :
1. `roles.decisions.path` est défini → enregistre là.
2. Sinon `roles.archive.path` est défini + accessible en écriture → enregistre là.
3. Sinon, crée `audits/` à la racine du vault et enregistre là. Note le rôle manquant pour F9.0.

Enregistre le HTML dans `{decisions-folder}/{YYYY-MM-DD}-vault-audit.html`.
Enregistre le JSON brut des constats dans `{decisions-folder}/{YYYY-MM-DD}-vault-audit-findings.json` (pour que les utilisateurs puissent exploiter la liste complète).

Relis le HTML pour confirmer que le contenu est présent (pas seulement que le fichier est présent). En cas d'incohérence → réessaie une fois → échoue bruyamment.

### 6.4 : Ouvrir + rendre + résumer (réponse finale, dans cet ordre)

**Partie 1 : ouvrir dans le navigateur.** `Bash` : `open "{path}"` (macOS) / `xdg-open` (Linux) / `start` (Windows).

**Partie 2 : émettre le HTML enregistré complet dans un bloc de code `html`.** Les runtimes avec support des artefacts (claude.ai, Claude Desktop) le rendent comme un panneau latéral ; le CLI le montre comme un bloc de code (le navigateur est déjà ouvert en partie 1). C'est le seul endroit où le HTML apparaît dans le chat : ne déverse pas de fragments en cours d'exécution, ne le colle pas deux fois, n'enveloppe pas de commentaire autour de lui.

**Partie 3 : résumé (après le HTML, sans commentaire entre les deux) :**

```
✅ Exécution de l'optimiseur terminée : voici ce qui a changé :
Rapport : {decisions-folder}/{YYYY-MM-DD}-vault-audit.html
Annexe JSON : {decisions-folder}/{YYYY-MM-DD}-vault-audit-findings.json
Score (F1–G7) : {score_before} → {score_after} ({delta_sign}{delta}) : « {interpretation_before} » → « {interpretation_after} »
{N} fichiers audités · {applied_total} corrections appliquées · {failed_total} échecs mécaniques (voir le rapport pour les raisons)
Tirets cadratins : {em_before} → {em_after} (-{em_delta}) · Couverture frontmatter : {fm_pct_before} → {fm_pct_after} (+{fm_pct_delta}pp) · Couverture d'index de dossier : {idx_pct_before} → {idx_pct_after} (+{idx_pct_delta}pp)
Tokens économisés : {tokens_saved} ({tokens_pct_saved}% de réduction de la charge agent) · ~{annual_savings} tokens/an économisés à {sessions}/semaine
```

Le résumé présente l'exécution de manière rétrospective : ce qui *a été fait*, ce qui *a changé*. En mode application en masse, les lignes « saved-to-plan » et « declined » sont absentes ; seuls `applied` et `failed` existent. En mode revue sélective, ajoute les décomptes enregistrer-dans-le-plan et refusés comme lignes supplémentaires.

Arrête. Ne propose pas d'actions de suivi.

---

## Règles : ce qu'il ne faut jamais faire

- **Ne jamais** présumer un dossier ou un fichier par son nom. Toujours résoudre via le registre des rôles construit à l'étape 1.5. `Context/`, `Projects/`, `Daily/`, `Plot.md`, `me.md`, etc. sont des *exemples* dans la documentation ; l'exécution résout les rôles abstraits vers ce que l'utilisateur possède réellement.
- **Ne jamais** ignorer un dossier parce qu'il ne correspond pas à un rôle standard. Chaque dossier est classifié, standard ou personnalisé, avec une couche. Les rôles personnalisés (`Building/`, `Garden/`, tout ce qui est spécifique à l'utilisateur) sont des participants de premier rang dans chaque framework qui opère sur leur couche. Les sauter va à l'encontre de la passe de découverte.
- **Ne jamais** traiter le registre des rôles comme éphémère. Persiste les attributions confirmées dans `.claude/vault-roles.json` à la fin de l'étape 1.5 afin que les exécutions futures démarrent depuis la vue confirmée par l'utilisateur de son vault, pas depuis une reclassification à neuf qui redemande tout.
- **Ne jamais** proposer un changement structurel uniquement parce que AI5D utilise cette convention. Les constats F9.0 de niveau 2 (améliorations fonctionnelles suggérant les conventions AI5D) doivent citer une preuve spécifique dans *ce* vault que la convention résoudrait un problème réel. Les propositions de réorganisation F9.6 doivent citer les finalités de dossier déclarées par l'utilisateur. « AI5D fait X » n'est une justification valide nulle part dans l'optimiseur. La taxonomie AI5D est une grille de reconnaissance et une référence optionnelle, jamais une forme cible à laquelle l'utilisateur doit se conformer.
- **Ne jamais** abandonner un constat de niveau 2 quand l'utilisateur a déjà un rôle personnalisé qui sert la fonction. Si `Lab/` (curated personnalisé) joue déjà le rôle projects, aucun constat F9.0 de niveau 2 ne se déclenche pour « vous devriez ajouter un dossier projects ». La fonction prime sur le nom.
- **Ne jamais** décider unilatéralement qu'un constat est « test fixture », « l'utilisateur ne voulait probablement pas ça », « jugement requis par section », ou toute autre raison inventée par l'agent pour router un constat vers autre chose que `applied` / `failed` en mode application en masse. L'utilisateur a dit appliquer tout ; cela veut dire tout. Si appliquer est mécaniquement dangereux, tente la version mécanique la plus sûre (ex. découper un fichier de 222KB aux frontières de H2 avec un marqueur de découpe automatique, marquer `applied_with_caveat: true`). La seule échappatoire valide est un échec mécanique avec un `failure_reason` consigné.
- **Ne jamais** rendre le tableau de bord avec le score AVANT. L'étape 5.9 est obligatoire : re-mesurer après les corrections, recalculer le score, peupler `{{SCORE_AFTER}}` et les colonnes après par rôle/par framework. Un tableau de bord qui ne montre que les chiffres de l'état AVANT est un bug.
- **Ne jamais** présenter le résumé final comme « ce que nous devons faire » en mode application en masse. Le résumé est rétrospectif : ce qui a changé, de combien, ce qui a échoué mécaniquement et pourquoi. Les formulations enregistrer-dans-le-plan / refusé / « pour plus tard » n'appartiennent qu'au mode revue sélective.
- **Ne jamais** appliquer un changement que l'utilisateur n'a pas approuvé via la revue `AskUserQuestion` à l'étape 4.
- **Ne jamais** réécrire automatiquement un CLAUDE.md sans confirmation utilisateur item par item. Les éditions F1.x, les réécritures de routage F9.1 et les ajouts d'orientation F9.7 se font en revue uniquement : l'utilisateur examine et confirme chaque édition rédigée avant qu'elle n'atterrisse.
- **Ne jamais** supprimer un fichier avant d'avoir grep les références.
- **Ne jamais** modifier les fichiers de la liste d'exclusion technique (`.git`, `.obsidian`, `.trash`, `node_modules`, `dist`, `build`).
- **Ne jamais** appliquer de substitutions à l'intérieur des zones protégées (code, URLs, chemins, frontmatter, wikilinks, titres, délimiteurs de tableau, dates).
- **Ne jamais** sauter un fichier à cause de son rôle. La classification route les règles ; elle n'exclut jamais de fichiers.
- **Ne jamais** sauter un framework. F1–F9 s'exécutent tous à chaque audit.
- **Ne jamais** laisser un constat ouvert. Chaque constat termine l'étape 5 avec `fix_status` défini à `applied`, `saved_to_plan`, `declined` ou `failed`. Non défini est un bug.
- **Ne jamais** appliquer en masse des corrections sémantiques (wikilinks F2.2, références croisées F2.4, reflection F8.x, architecture F9.1–F9.7). Chaque correction sémantique se parcourt item par item avec l'utilisateur confirmant la cible/le gagnant/la destination/la formulation.
- **Ne jamais** laisser F8 ou F9 modifier un SKILL.md ou quoi que ce soit dans `.claude/rules/`. Le périmètre de F8 est la couche curatée ; le périmètre de F9 exclut SKILL.md et `.claude/rules/`.
- **Ne jamais** terminer une fusion F8.2 ou une fusion de dossier F9.5 sans vérifier (a) la taille du fichier fusionné contre le budget par fichier de F5 et (b) zéro nouveau wikilink mort. Échouer à l'un ou l'autre annule et définit `fix_status: failed`.
- **Ne jamais** générer un Plot.md qui dépasse 8KB ou qui a des tirets cadratins. Si la liste des enfants d'un dossier ferait passer Plot.md au-dessus du budget, route vers une proposition de réorganisation F9.6 (découper le dossier) à la place.
- **Ne jamais** supprimer une source de fusion. F8.2 / F9.5 archivent les sources vers `Intelligence/archive/{date}-merged/`.
- **Ne jamais** intégrer F8 ou F9 dans le score de santé. Tous deux font remonter des opportunités d'optimisation, pas des échecs de lint ; le score couvre F1–G7 uniquement.
- **Ne jamais** paraphraser une règle de framework en faisant remonter un constat. Lis le fichier d'implémentation de la passe pertinent quand tu exécutes ses vérifications ; cite le framework dans le constat. Le contrôle de cohérence de l'étape 2.2 l'impose.
- **Ne jamais** s'exécuter silencieusement. L'étape 0.5 crée la liste de tâches visible ; la tâche de chaque framework transite de `in_progress` à `completed` au fil de l'exécution. Sauter la liste de tâches va à l'encontre du skill.
- **Ne jamais** livrer un template à moitié rendu. S'il reste un `{{placeholder}}` → échoue.
- **Ne jamais** déverser de HTML partiel en cours d'exécution. Le HTML complet apparaît une seule fois, à la fin (étape 6.4 partie 2). Ne colle pas le HTML deux fois ; ne colle pas de fragments partiels pendant que les constats sont collectés.

---

## Pourquoi ce skill existe

Huit frameworks plus des notes de terrain de praticiens convergent vers une conclusion : les vaults pourrissent sans maintenance active. Karpathy appelle cela le **lint**. Anthropic appelle cela le **pruning test** + le **memory budget**. Chroma appelle cela le **context rot**. Caveman appelle cela la **token discipline**. La divulgation progressive appelle cela le **layering**. Anthropic Dreams appelle cela la **reflection** : lire la couche curatée aux côtés des sessions récentes et faire remonter ce que le lint par fichier ne peut pas voir. F9 ajoute l'**architecture & discoverability** : parcourir le chemin que Claude collègue emprunte réellement, vérifier que le routage correspond à la réalité des dossiers, s'assurer que chaque dossier a un Plot.md à jour, et confirmer que chaque fichier est atteignable depuis la racine en ≤ 3 sauts. L'audit encode chaque signal auditable comme une vérification discrète, les applique toutes à chaque exécution, parcourt chaque constat via appliquer-maintenant / enregistrer-dans-le-plan / refusé, et enregistre un tableau de bord HTML catégorisé avec la lecture architecturale de l'agent en haut.

Lance-le chaque semaine tant que le vault grandit. Chaque mois une fois stabilisé. Chaque exécution ajoute un tableau de bord daté afin que l'utilisateur puisse regarder le score grimper.

Pour les règles détaillées derrière chaque passe, lis les `references/{framework}.md` et `references/passes-{framework}.md` pertinents. Chacun est complet en lui-même : TOC, règles complètes, regex exacte, format de constat, URLs sources.
