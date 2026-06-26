# Anthropic : Managed Agents Memory (April 2026)

## Sommaire

1. [Thèse centrale](#thèse-centrale)
2. [Pourquoi cela valide le vault basé sur des fichiers](#pourquoi-cela-valide-le-vault-basé-sur-des-fichiers)
3. [Faits d'architecture](#faits-darchitecture)
4. [Règles strictes](#règles-strictes)
5. [Versioning et piste d'audit](#versioning-et-piste-daudit)
6. [Modèle de concurrence](#modèle-de-concurrence)
7. [Là où la mémoire excelle](#là-où-la-mémoire-excelle)
8. [Là où la mémoire NE remplace PAS la recherche vectorielle](#là-où-la-mémoire-ne-remplace-pas-la-recherche-vectorielle)
9. [Résultats rapportés par les premiers adoptants](#résultats-rapportés-par-les-premiers-adoptants)
10. [Recommandations de nommage et de structure des fichiers](#recommandations-de-nommage-et-de-structure-des-fichiers)
11. [À faire](#à-faire)
12. [À ne pas faire](#à-ne-pas-faire)
13. [Citations textuelles](#citations-textuelles)
14. [Signaux auditables](#signaux-auditables)
15. [Sources](#sources)

---

## Thèse centrale

La mémoire dans les Managed Agents d'Anthropic se monte comme un **véritable système de fichiers à `/mnt/memory/`**. Claude lit et écrit la mémoire à l'aide des **mêmes outils bash et fichiers qu'il utilise déjà pour tout le reste** : `cat`, `ls`, `Read`, `Write`. Aucun nouveau paradigme, aucun modèle d'embedding, aucune infrastructure de récupération.

> *"Files, not vectors. Claude reads and writes memory with the same bash and file tools it uses for everything else — no new paradigm, no embedding model, no retrieval infrastructure."*

C'est le déploiement en production du même principe qui sous-tend le Karpathy LLM Wiki et chaque pattern de second cerveau qui fonctionne : **des fichiers markdown dans des dossiers, parcourus par leur nom et leur structure, constituent le bon substrat pour faire fructifier la connaissance d'un agent.**

## Pourquoi cela valide le vault basé sur des fichiers

Pendant deux ans, l'hypothèse dominante était que la mémoire d'un agent en production nécessitait des embeddings vectoriels, des pipelines RAG et des stratégies de chunking. Avril 2026 : Anthropic a livré l'inverse.

Le pari sous-jacent :

- **La connaissance qui se cumule** (préférences, apprentissages, historique de tâches, patterns structurés) est mieux servie par des fichiers nommés qu'un agent parcourt explicitement.
- **La recherche par similarité sur des milliers d'entrées** bénéficie encore des magasins vectoriels, mais c'est un problème différent de la mémoire.
- **Les fichiers sont débogables.** Un humain peut lire `/mnt/memory/preferences.md` et savoir exactement ce que l'agent verra. C'est impossible avec des embeddings.
- **Les fichiers sont versionnables** avec l'outillage standard (git, diff, journaux d'audit).

Pour quiconque exploite un vault basé sur Obsidian avec Claude Code, c'est le signal le plus fort possible que l'architecture est la bonne. Le laboratoire qui construit le modèle a lui aussi choisi les fichiers.

## Faits d'architecture

| Fait | Valeur |
|---|---|
| Chemin de montage | `/mnt/memory/` |
| Limite de taille par fichier | **100 Ko (~25K tokens)** |
| Taille recommandée par fichier | **<10 Ko** |
| En-tête beta | `managed-agents-2026-04-01` |
| Ensemble d'outils requis | `agent_toolset_20260401` |
| Plafond total du magasin | **Aucun plafond documenté** en beta publique |
| Plafond du nombre de fichiers | **Aucun plafond documenté** en beta publique |
| Cycle de vie de l'attachement | La mémoire ne s'attache qu'à la création de la session, via le tableau `resources` |
| Inter-session | Portée au workspace, pas à la session : plusieurs agents peuvent s'attacher simultanément |
| Modes d'accès concurrents | Lecture seule et lecture-écriture supportés dans le même pool de sessions |
| Lancement | 23 avril 2026 (beta publique) |

## Règles strictes

| # | Règle | Auditable dans le contexte du vault |
|---|---|---|
| 1 | Limite de taille par fichier : 100 Ko / ~25K tokens | ✅ Passe 1 (taille) |
| 2 | Taille recommandée par fichier : <10 Ko | ✅ Passe 1 (candidat au découpage) |
| 3 | La mémoire ne s'attache **qu'à la création de la session** : impossible d'ajouter en cours de session | — |
| 4 | Écritures concurrentes : **last-write-wins** par défaut | — |
| 5 | La production devrait utiliser des vérifications de précondition `content_sha256` | — |
| 6 | Chaque écriture produit une version nommée immuable (`memver_…`) | — |
| 7 | Tous les changements remontent en tant qu'événements Console (piste d'audit) | — |
| 8 | Plusieurs fichiers ciblés > un seul méga-fichier | ✅ Passe 1 / Passe 9 |

## Versioning et piste d'audit

Chaque écriture dans un fichier de mémoire produit une **version nommée immuable** (`memver_…`). Cela donne :

- **Piste d'audit** : qui/quoi a écrit quoi, et quand
- **Récupération à un point dans le temps** : revenir à n'importe quelle version antérieure
- **Caviardage sans détruire la chaîne** : marquer une version comme caviardée tout en préservant les versions suivantes

C'est le même pattern que git fournit pour le code. Anthropic a explicitement modelé la mémoire d'après les systèmes de fichiers sous contrôle de version.

Pour les équivalents vault : un vault Obsidian correctement géré avec git ou la synchronisation Relay obtient les mêmes propriétés. L'audit du vault n'impose pas le versioning (c'est le rôle de la couche de synchronisation) mais signale quand un vault n'est *pas* sous contrôle de version.

## Modèle de concurrence

La mémoire est **portée au workspace**, pas à la session. Plusieurs agents peuvent s'attacher au même magasin de mémoire simultanément.

| Mode | Quand l'utiliser |
|---|---|
| **Lecture seule** | Sous-agents qui n'ont besoin que du contexte, ne modifient jamais |
| **Lecture-écriture** | Agents principaux qui mettent à jour la mémoire après l'achèvement d'une tâche |

Résolution de conflit par défaut : **last-write-wins**. Pour la production :

```python
client.memory.create(
    name="preferences.md",
    content=new_content,
    parent="memver_abc123"  # version parente explicite
)
# OU
client.memory.create(
    name="preferences.md",
    content=new_content,
    content_sha256=expected_sha  # concurrence optimiste
)
```

Sans ces garde-fous, deux écritures concurrentes peuvent silencieusement s'écraser l'une l'autre.

## Là où la mémoire excelle

✅ **Préférences structurées** : style de l'utilisateur, ton de communication, patterns récurrents
✅ **Historique des tâches** : ce qui a été fait, quand, avec quel résultat
✅ **Constantes** : clés d'API (chiffrées), URLs d'endpoints, chemins de datasets
✅ **Apprentissage dans le temps** : les corrections s'accumulent, l'agent s'améliore
✅ **Cohérence inter-session** : le même agent reste cohérent au fil des jours
✅ **Connaissance auditable** : les humains peuvent inspecter ce dont l'agent "se souvient"

Ce sont exactement les types de contenu que détiennent un CLAUDE.md, un profil par équipe, ou un journal `Intelligence/decisions/`.

## Là où la mémoire NE remplace PAS la recherche vectorielle

❌ **Recherche par similarité sur des milliers d'entrées.** "Trouve-moi toutes les réclamations clients similaires à celle-ci" requiert des embeddings, pas une recherche de fichier.
❌ **Requêtes sémantiques floues** sur de larges corpus. Les fichiers se parcourent par nom et par dossier, pas par sens.
❌ **Extraction de concepts inter-documents** à grande échelle. Les pipelines RAG l'emportent encore ici.
❌ **Requêtes analytiques en temps réel.** La mémoire n'est pas une base de données.

C'est pourquoi les stacks hybrides existent. Markdown pour le narratif + Postgres pour les entités + pgvector pour la similarité = le pattern de production. La mémoire remplace *la couche narrative + préférences*, pas la couche analytique.

## Résultats rapportés par les premiers adoptants

Le billet de lancement d'Anthropic et la couverture qui a suivi citaient :

- **Netflix**
- **Rakuten**
- **Wisedocs**
- **Ando**

Résultats rapportés pour les workflows de vérification de documents :
- *"97 percent reduction in first-pass errors"*
- *"30 percent speed increase"*

Source : billet de lancement, résumé par buildfastwithai.com, SDTimes, EdTech Innovation Hub, Techzine, DataCenter Knowledge.

> Mise en garde : le billet de blog Anthropic original n'était pas directement récupérable au moment de l'audit. Les chiffres de 100 Ko / 25K tokens / `/mnt/memory/` sont largement rapportés et cohérents à travers les sources secondaires mais devraient être vérifiés par rapport à l'annonce canonique d'Anthropic lorsqu'elle est accessible.

## Recommandations de nommage et de structure des fichiers

Le nommage compte parce que l'agent parcourt par nom. D'après les recommandations de lancement :

✅ **Bons noms :**
- `preferences.md`
- `failed-payment-recovery-log-2026-04-25.md`
- `client-onboarding-checklist.md`
- `voice-tone-rules.md`
- `task-history-2026-q1.md`

❌ **Mauvais noms :**
- `notes.md`
- `notes-2.md`
- `temp.md`
- `untitled.md`
- `file-1.md`

Le même principe s'applique aux fichiers du vault : descriptifs, datés là où c'est approprié, sans numérotation ambiguë.

### Structure recommandée

```
/mnt/memory/
├── INDEX.md              # ce qui se trouve où (~équivalent de MEMORY.md)
├── preferences.md        # préférences utilisateur
├── tone-rules.md
├── task-history/
│   ├── 2026-q1.md
│   └── 2026-q2.md
├── projects/
│   ├── alpha.md
│   └── beta.md
└── learnings.md          # corrections accumulées au fil du temps
```

Dans un vault : `Context/`, `Team/`, `Projects/`, plus un CLAUDE.md de routage au niveau racine, c'est le même pattern avec des noms à la sauce vault.

## À faire

- Découper la mémoire en fichiers ciblés ; garder chacun <10 Ko.
- Utiliser des noms de fichiers descriptifs que l'agent peut trouver par nom (pas `notes-2.md`).
- Utiliser des vérifications de précondition `content_sha256` en production.
- Traiter la mémoire comme une infrastructure partagée au niveau du workspace.
- Injecter un fichier d'index au niveau racine (`INDEX.md` ou `MEMORY.md`) pour que l'agent puisse naviguer.
- Utiliser des noms de fichiers horodatés pour le contenu sensible au temps (logs, rapports).
- Traiter les écritures en mémoire comme des commits git : descriptives, atomiques, faciles à auditer.
- Faire correspondre la structure des fichiers à la façon dont un humain organiserait le même contenu.

## À ne pas faire

- Ne pas tout mettre dans un seul fichier. Le plafond de 100 Ko est un plafond, pas un objectif.
- Ne pas s'attendre à une recherche sémantique de type vectoriel.
- Ne pas attacher la mémoire en cours de session. L'attacher uniquement à la création.
- Ne pas se reposer sur last-write-wins en cas de concurrence. Utiliser `content_sha256`.
- Ne pas utiliser de noms de fichiers ambigus. L'agent parcourt par nom.
- Ne pas stocker de données transactionnelles/relationnelles en mémoire. Utiliser une base de données.
- Ne pas stocker de blobs binaires ou de gros fichiers. La mémoire est faite pour du texte structuré.
- Ne pas sauter le fichier d'index. Sans lui, l'agent ne sait pas ce qui est disponible.

## Citations textuelles

> *"Files, not vectors. Claude reads and writes memory with the same bash and file tools it uses for everything else — no new paradigm, no embedding model, no retrieval infrastructure."*

> *"Memory on Claude Managed Agents is now in public beta on the Claude Platform, letting agents learn and improve across different sessions."*

## Signaux auditables

Quand ce skill exécute la Passe 1 (taille) et la Passe 9 (anti-patterns) pour les fichiers de type mémoire dans un vault :

- **Taille par fichier** : signaler tout fichier > 100 Ko ou > 25K tokens (hors budget de type mémoire pour tout fichier individuel).
- **Sous-budget** : signaler > 10 Ko comme "candidat au découpage" : trop volumineux pour la taille recommandée par fichier.
- **Mono-fichiers thématiques** : détecter les fichiers uniques contenant plusieurs sujets sans rapport. Heuristique : compter les titres h2 distincts ; signaler si un fichier comporte >5 sections h2 sans rapport, ce qui suggère qu'il devrait être découpé.
- **Caractère descriptif du nommage des fichiers** : signaler les fichiers correspondant à des patterns ambigus : `notes\d*\.md`, `untitled.*\.md`, `temp.*\.md`, `file-\d+\.md`, `new-document.*\.md`.
- **Présence d'un index de mémoire** : dans tout dossier contenant >5 fichiers, signaler s'il n'y a pas d'`index.md`, `INDEX.md`, `MEMORY.md`, ou `README.md` pour naviguer par nom.
- **Cohérence du nommage par date** : dans les dossiers qui devraient être datés (Daily/, Intelligence/meetings/), signaler les fichiers qui ne suivent pas le préfixe `YYYY-MM-DD`.
- **Versioning** : signaler si le vault n'est pas sous git ou une couche de synchronisation (Relay, etc.). Pas de versioning = pas de piste d'audit = pas de rollback.

## Sources

- https://www.anthropic.com/news/memory (billet de lancement canonique : accessible mais n'a pas renvoyé le contenu de la mémoire via WebFetch à la date de l'audit)
- https://claude.com/blog/memory (cible de redirection)
- https://www.buildfastwithai.com/blogs/claude-managed-agents-memory-2026 (détails opérationnels extraits)
- https://www.testingcatalog.com/anthropic-launches-memory-in-claude-agents-for-enterprise/
- https://sdtimes.com/anthropic/anthropic-adds-memory-to-claude-managed-agents/

> Note de vérification : lorsque vous travaillez avec ce fichier, privilégiez la récupération directe de l'annonce Anthropic originale pour confirmer les limites et l'en-tête beta. Les sources secondaires sont cohérentes mais tierces.
