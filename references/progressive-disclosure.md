# Divulgation progressive : bonnes pratiques des skills Anthropic

## Sommaire

1. [Thèse centrale](#thèse-centrale)
2. [Le modèle de chargement à trois niveaux](#le-modèle-de-chargement-à-trois-niveaux)
3. [Coût en tokens par niveau](#coût-en-tokens-par-niveau)
4. [Règles strictes : frontmatter](#règles-strictes--frontmatter)
5. [Règles strictes : corps du SKILL.md](#règles-strictes--corps-du-skillmd)
6. [Règles strictes : références](#règles-strictes--références)
7. [Conventions de nommage](#conventions-de-nommage)
8. [Rédaction de la description](#rédaction-de-la-description)
9. [Degrés de liberté ajustés à la fragilité](#degrés-de-liberté-ajustés-à-la-fragilité)
10. [Exemples de patterns](#exemples-de-patterns)
11. [Construire les évaluations d'abord](#construire-les-évaluations-dabord)
12. [Tester sur tous les niveaux de modèle](#tester-sur-tous-les-niveaux-de-modèle)
13. [La checklist verbatim de 21 points d'Anthropic](#la-checklist-verbatim-de-21-points-danthropic)
14. [Pourquoi cela compte pour les vaults](#pourquoi-cela-compte-pour-les-vaults)
15. [À faire](#à-faire)
16. [À éviter](#à-éviter)
17. [Citations verbatim](#citations-verbatim)
18. [Signaux auditables](#signaux-auditables)
19. [Sources](#sources)

---

## Thèse centrale

> *"The context window is a public good. Your Skill shares the context window with everything else Claude needs to know."*

Les skills doivent être **découvrables à faible coût** (métadonnées seulement, ~30 à 50 tokens par skill) et ne charger le détail qu'au déclenchement. Le mécanisme : **trois niveaux de divulgation progressive**.

C'est ce pattern qui permet à une installation de Claude Code d'héberger plus de 100 skills sans s'étouffer au démarrage. Seuls 30 à 50 tokens × 100 skills = ~5K tokens sont payés d'emblée. Chaque corps de SKILL.md individuel (souvent 200 à 500 lignes / ~2K à 5K tokens) ne se charge que lorsqu'il est pertinent.

## Le modèle de chargement à trois niveaux

| Niveau | Contenu | Quand chargé | Coût en tokens |
|---|---|---|---|
| **L1 : Métadonnées** | `name` + `description` du frontmatter YAML | **Toujours**, au démarrage | ~30 à 50 tokens par skill |
| **L2 : Corps du SKILL.md** | Instructions principales | Quand Claude juge le skill **pertinent** pour le prompt | Chargé une fois au déclenchement (corps entier) |
| **L3 : Références / scripts** | `references/*.md`, scripts exécutables | **À la demande**, uniquement quand le SKILL.md y renvoie et que Claude les lit | Zéro contexte tant qu'ils ne sont pas consultés |

Le même pattern s'applique à :
- **Outils MCP** : seuls les noms des outils se chargent au démarrage ; les schémas complets sont différés
- **Règles CLAUDE.md à portée de chemin** : `.claude/rules/*.md` avec un frontmatter `paths:` ne se chargent que lorsque les fichiers correspondants sont lus
- **CLAUDE.md de sous-répertoire** : ne se charge que lorsque Claude travaille dans ce répertoire

## Coût en tokens par niveau

Anthropic publie des chiffres approximatifs :

- **L1 (métadonnées) :** ~30 à 50 tokens par skill. Avec 30 skills installés, ~1 200 à 1 500 tokens d'emblée.
- **L2 (corps du SKILL.md) :** typiquement 200 à 500 lignes (~2K à 5K tokens). Chargé au déclenchement.
- **L3 (références) :** sans limite. Chargé uniquement à la lecture.

À titre de comparaison, une session Claude Code typique de 200K tokens charge ~8K de surcharge au démarrage (prompt système + mémoire + métadonnées des skills + CLAUDE.md). Soit 4 % de la fenêtre, ce qui laisse 96 % pour la conversation proprement dite.

Si vous cassiez le pattern de divulgation progressive (par exemple en chargeant tous les corps de SKILL.md d'emblée), 30 skills × 3K tokens = 90K. La moitié de la fenêtre disparaît avant le premier message de l'utilisateur.

## Règles strictes : frontmatter

| Champ | Limite | Notes |
|---|---|---|
| `name` | **64 caractères maximum** | Lettres minuscules, chiffres, tirets uniquement. Pas de balises XML. **Pas de mots réservés ("anthropic", "claude").** |
| `description` | **1024 caractères maximum** | Non vide. Pas de balises XML. |
| Style de `description` | Troisième personne | "Processes Excel files," et non "I can help…" ou "Use me when…" |
| Contenu de `description` | À la fois le **quoi** et le **quand** | Doit inclure des déclencheurs explicites |

Modes de défaillance :

- Nom > 64 caractères → skill rejeté au chargement
- Description > 1024 caractères → skill rejeté au chargement (c'est l'erreur que rencontrent les utilisateurs qui écrivent des scalaires repliés YAML de plusieurs paragraphes)
- Mots réservés → skill rejeté au chargement
- Description à la première personne → le skill fonctionne mais Claude est moins susceptible de l'invoquer correctement

## Règles strictes : corps du SKILL.md

| Règle | Source |
|---|---|
| Corps du SKILL.md **sous 500 lignes** | Anthropic |
| Utiliser des sections, en-têtes et listes clairs | Anthropic |
| Un seul sujet par skill | Anthropic |
| Éviter les infos sensibles au temps en ligne ("After August 2025…") | Anthropic |
| Utiliser des blocs repliables `<details>` pour les patterns hérités/anciens | Anthropic |
| Utiliser uniquement des barres obliques avant, même sous Windows | Anthropic |
| Outils MCP toujours pleinement qualifiés : `ServerName:tool_name` | Anthropic |
| Pas de nombres magiques / "constantes vaudou" sans commentaires `# why` | Anthropic |
| Terminologie cohérente : choisir un terme ("API endpoint") et l'utiliser partout | Anthropic |
| Ne pas refiler les erreurs à Claude dans les scripts : les gérer explicitement | Anthropic |

## Règles strictes : références

> *"Keep references one level deep from SKILL.md."*

C'est la règle la plus souvent enfreinte. La hiérarchie :

✅ **Autorisé :** `SKILL.md` → `references/foo.md`
❌ **Cassé :** `SKILL.md` → `references/foo.md` → `references/bar.md`
❌ **Cassé :** `SKILL.md` → `references/index.md` → `references/foo.md`

Pourquoi : une imbrication plus profonde pousse Claude à utiliser des **lectures partielles** (`head -100`) et à manquer du contenu. Les références doivent être atteignables en un seul saut pour que Claude lise le fichier entier quand il en a besoin.

Autres règles concernant les références :

| Règle | Source |
|---|---|
| Les fichiers de référence > 100 lignes ont besoin d'un **sommaire en haut** | Anthropic |
| Utiliser une organisation spécifique au domaine (ex : `references/finance.md`, `references/sales.md`) | Anthropic |
| Chaque fichier de référence doit être **complet en lui-même** | Anthropic |
| Les scripts sont exécutés, pas lus dans le contexte : ils consomment zéro contexte tant qu'aucune sortie n'est générée | Anthropic |

## Conventions de nommage

✅ **Bon** (forme en -ing, orientée action) :
- `processing-pdfs`
- `analyzing-spreadsheets`
- `responding-to-dms`
- `auditing-vault-health`

❌ **À éviter** (vague, générique) :
- `helper`
- `utils`
- `tools`
- `documents`
- `data`
- `manager`

Des noms spécifiques et orientés action aident Claude à sélectionner le bon skill parmi plus de 100 disponibles.

## Rédaction de la description

La description est **le champ le plus important** : c'est ainsi que Claude décide de charger ou non le corps du skill.

> *"Each Skill has exactly one description field. The description is critical for skill selection: Claude uses it to choose the right Skill from potentially 100+ available Skills."*

### Composants requis

1. **Quoi** : ce que fait le skill (troisième personne, spécifique)
2. **Quand** : quand l'utiliser (déclencheurs explicites, plusieurs formulations)
3. **Prérequis** s'il y en a (ex : "Run from vault root")

### Exemple : mauvais

```yaml
description: "Helps with PDFs"
```

Vague, pas de déclencheurs, ne sera pas sélectionné de manière fiable.

### Exemple : bon

```yaml
description: "Extracts structured data from PDF documents. Identifies tables, headers, and key fields. TRIGGERS: extract from PDF, parse PDF, get data from PDF, scan PDF, PDF to structured data, PDF to JSON. REQUIREMENT: PDF must be text-based, not scanned image."
```

Spécifique, déclencheurs multi-formulés, prérequis explicité.

## Degrés de liberté ajustés à la fragilité

Différentes opérations nécessitent différents niveaux de contrainte :

| Niveau de liberté | Forme | À utiliser quand |
|---|---|---|
| **Élevé** | Instructions textuelles | Plusieurs approches valides ; la tâche tolère la variation |
| **Moyen** | Pseudo-code, scripts paramétrés | Un pattern préféré existe, mais la variation est acceptable |
| **Faible** | Scripts spécifiques sans paramètres | Opérations fragiles/critiques (ex : migrations de BD, vérifications de sécurité) |

Ajustez la forme à la fragilité. N'écrivez pas de scripts rigides pour des tâches créatives. N'écrivez pas de texte lâche pour des tâches où la déviation casse la production.

## Exemples de patterns

### Pattern 1 : guide de haut niveau avec références

```
SKILL.md          # Démarrage rapide, cas courants
references/
├── FORMS.md      # Référence spécifique aux formulaires
├── REFERENCE.md  # API complète
└── EXAMPLES.md   # Exemples concrets
```

SKILL.md est l'index. Le contenu approfondi spécifique au domaine vit dans les références.

### Pattern 2 : organisation spécifique au domaine

```
SKILL.md
references/
├── finance.md
├── sales.md
├── engineering.md
└── operations.md
```

Une requête liée aux ventes ne charge jamais la référence finance. La divulgation progressive la plus propre.

### Pattern 3 : détails conditionnels

```
SKILL.md          # Contenu de base en ligne
references/
└── advanced.md   # Lié depuis SKILL.md quand des cas "avancés" surviennent
```

Le cas le plus courant est dans le corps ; les cas limites sont liés.

## Construire les évaluations d'abord

La forte recommandation d'Anthropic :

> Construire au moins **3 évaluations AVANT de rédiger une documentation extensive.**

Les évaluations vous indiquent :
- Si la description est assez spécifique pour se déclencher de manière fiable
- Si le corps du SKILL.md est assez clair pour que le modèle le suive
- Où le skill échoue actuellement

Sans évaluations, vous devinez.

## Tester sur tous les niveaux de modèle

> *"What works for Opus may need more detail for Haiku."*

Un skill qui fonctionne sur Opus 4.7 peut échouer sur Haiku 4.5 parce que :
- Haiku a moins de marge de raisonnement pour les instructions vagues
- Haiku suit moins d'instructions simultanées
- Haiku peut nécessiter des exemples plus explicites

Testez au minimum : Haiku, Sonnet, Opus. Si le skill est destiné à un usage en production sur plusieurs niveaux de modèle, c'est non négociable.

## La checklist verbatim de 21 points d'Anthropic

Reproduite depuis la page officielle des bonnes pratiques des skills :

- [ ] La description est spécifique et inclut des termes clés
- [ ] La description inclut à la fois ce que fait le skill et quand l'utiliser
- [ ] Le corps du SKILL.md est sous 500 lignes
- [ ] Les détails supplémentaires sont dans des fichiers séparés (si nécessaire)
- [ ] Pas d'information sensible au temps (ou dans une section "old patterns")
- [ ] Terminologie cohérente partout
- [ ] Les exemples sont concrets, pas abstraits
- [ ] Les références de fichiers sont à un niveau de profondeur
- [ ] Divulgation progressive utilisée de manière appropriée
- [ ] Les workflows ont des étapes claires
- [ ] Les scripts résolvent les problèmes plutôt que de les refiler à Claude
- [ ] La gestion des erreurs est explicite et utile
- [ ] Pas de "constantes vaudou"
- [ ] Les packages requis sont listés et vérifiés
- [ ] Les scripts ont une documentation claire
- [ ] Pas de chemins de style Windows
- [ ] Des étapes de validation/vérification pour les opérations critiques
- [ ] Des boucles de rétroaction incluses pour les tâches critiques en qualité
- [ ] Au moins trois évaluations créées
- [ ] Testé avec Haiku, Sonnet et Opus
- [ ] Testé avec des scénarios d'usage réels

Cette checklist est ce que la Passe 8 du skill vault-audit encode partiellement. Certains items sont auditables par programme (taille, profondeur des références, chemins) ; certains nécessitent une revue humaine (nombre d'évaluations, test sur les modèles, test de scénarios).

## Pourquoi cela compte pour les vaults

Le pattern de divulgation progressive est **la même architecture** que celle qu'utilise le vault, simplement à une couche différente :

| Couche skill | Couche vault |
|---|---|
| Métadonnées L1 (toujours chargées) | CLAUDE.md racine (toujours chargé au démarrage de session) |
| Corps L2 du SKILL.md (selon pertinence) | CLAUDE.md par dossier (se charge quand Claude travaille dans ce dossier) |
| Références L3 (à la demande) | Notes spécifiques que Claude lit pour répondre à une requête |

Voilà pourquoi un vault bien conçu et une suite de skills bien conçue partagent des principes :

- Racine épurée / SKILL.md épuré
- Détails par dossier / références par domaine
- Un niveau de profondeur
- Descriptions et nommage spécifiques
- L'ensemble testé sous charge

Quand le skill audite un vault, il vérifie les mêmes patterns qu'Anthropic utilise pour auditer les skills.

## À faire

- Pré-charger uniquement les métadonnées ; différer le corps et les références.
- Garder le SKILL.md ≤ 500 lignes.
- Garder les références à un niveau de profondeur.
- Ajouter un sommaire aux fichiers de référence > 100 lignes.
- Rédiger les descriptions à la troisième personne avec des déclencheurs explicites.
- Utiliser des noms en forme -ing orientés action.
- Utiliser des barres obliques avant partout (même sous Windows).
- Construire les évaluations d'abord (≥ 3 par skill).
- Tester sur Haiku, Sonnet, Opus.
- Ajuster les degrés de liberté à la fragilité (scripts rigides pour les opérations fragiles ; texte lâche pour les créatives).
- Utiliser des blocs `<details>` pour les patterns hérités/anciens.
- Toujours qualifier pleinement les outils MCP (`ServerName:tool_name`).
- Inclure des étapes de validation/vérification pour les opérations critiques.

## À éviter

- Ne pas imbriquer les références sur plus d'un niveau (`SKILL.md → adv.md → details.md` est cassé).
- Ne pas inclure de dates sensibles au temps en ligne ("After August 2025…") en dehors de `<details>`.
- Ne pas utiliser de noms vagues : `helper`, `utils`, `tools`.
- Ne pas utiliser la première/deuxième personne dans les descriptions.
- Ne pas inclure de mots réservés ("anthropic", "claude") dans les noms de skill.
- Ne pas refiler les erreurs à Claude dans les scripts ; les gérer explicitement.
- Ne pas utiliser de nombres magiques sans commentaires `# why`.
- Ne pas écrire de corps de SKILL.md > 500 lignes.
- Ne pas sauter les évaluations.
- Ne pas livrer sans tester au moins sur Haiku et Sonnet.
- Ne pas utiliser de chemins à barre oblique inverse de style Windows.
- Ne pas présenter 5 alternatives sans choisir un défaut.

## Citations verbatim

> *"The context window is a public good. Your Skill shares the context window with everything else Claude needs to know."*

> *"Default assumption: Claude is already very smart. Only add context Claude doesn't already have."*

> *"Keep references one level deep from SKILL.md."*

> *"Keep SKILL.md body under 500 lines for optimal performance."*

> *"For reference files longer than 100 lines, include a table of contents at the top."*

> *"Each Skill has exactly one description field. The description is critical for skill selection: Claude uses it to choose the right Skill from potentially 100+ available Skills."*

## Signaux auditables

Quand ce skill exécute la Passe 8 (audit skill-vault) et la Passe 1 (taille pour SKILL.md) :

- **Nombre de lignes du corps du SKILL.md** > 500 → warn.
- **Longueur de `name`** > 64 caractères → fail.
- **`name` contient des majuscules, des espaces ou des caractères spéciaux (autres que les tirets)** → fail.
- **`name` contient des mots réservés** ("anthropic", "claude") → fail.
- **Longueur de `description`** > 1024 caractères → fail.
- **`description` vide ou manquante** → fail.
- **`description` à la première/deuxième personne** ("I can…", "You can…", "Use me…") → warn.
- **`description` sans mots-clés déclencheurs** ("when", "TRIGGERS", "Use this skill") → warn.
- **Profondeur des références** : parcourir les liens markdown depuis SKILL.md. Toute référence atteignable uniquement via 2+ sauts → fail.
- **Fichier de référence > 100 lignes sans sommaire en haut** → warn.
- **Chemins de style Windows** (`\\` ou `C:\`) → fail.
- **Références d'outils MCP sans préfixe `ServerName:`** → warn.
- **Nombres magiques dans les scripts sans commentaires `# why`** → warn (heuristique).
- **Langage sensible au temps** ("After 2025", "Before Q3") en dehors de `<details>` → warn.
- **Terminologie incohérente** : un même concept désigné par plusieurs termes dans un seul SKILL.md → warn (heuristique).
- **Dossier `references/` du skill** contenant des fichiers qui correspondent aux noms de fichiers `Context/` du vault (icp, brand, voice, services) → fail (Passe 8 : pointer vers le vault à la place).

## Sources

- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices (canonique)
- https://code.claude.com/docs/en/skills (vue d'ensemble de la rédaction de skills)
- Anthropic, vidéo "Claude Agent Skills Explained" (2025-11-26) : présentation officielle du modèle L1/L2/L3
- "Progressive Disclosure in Claude Code" (Developers Digest, 2026-01-12) : note sur la convergence du secteur
