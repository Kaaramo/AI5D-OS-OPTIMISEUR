# Karpathy : LLM Wiki (April 2026)

## Sommaire

1. [Thèse centrale](#thèse-centrale)
2. [Pourquoi les humains abandonnent les wikis mais pas les LLM](#pourquoi-les-humains-abandonnent-les-wikis-mais-pas-les-llm)
3. [Architecture à trois couches](#architecture-à-trois-couches)
4. [Trois opérations](#trois-opérations)
5. [Ingest : le flux canonique](#ingest--le-flux-canonique)
6. [Query : le flux canonique](#query--le-flux-canonique)
7. [Lint : ce qu'il détecte précisément](#lint--ce-quil-détecte-précisément)
8. [Pourquoi la synthèse au moment de l'ingest bat le RAG au moment de la query](#pourquoi-la-synthèse-au-moment-de-lingest-bat-le-rag-au-moment-de-la-query)
9. [Structure du document de schéma](#structure-du-document-de-schéma)
10. [Règles strictes](#règles-strictes)
11. [Le fan-out de 10 à 15 pages](#le-fan-out-de-10-à-15-pages)
12. [Ce que possède l'humain vs ce que possède le LLM](#ce-que-possède-lhumain-vs-ce-que-possède-le-llm)
13. [À faire](#à-faire)
14. [À ne pas faire](#à-ne-pas-faire)
15. [Citations verbatim](#citations-verbatim)
16. [Signaux auditables](#signaux-auditables)
17. [Critiques de praticiens](#critiques-de-praticiens)
18. [Sources](#sources)

---

## Thèse centrale

Le RAG traditionnel force le LLM à **redécouvrir la connaissance à partir de zéro à chaque query**. Chaque récupération extrait des chunks bruts ; le LLM doit ensuite re-résumer, re-connecter, re-raisonner. Il n'y a aucune accumulation.

Le LLM Wiki de Karpathy inverse cette logique : **pré-digérer les sources au moment de l'ingest** dans un wiki markdown maintenu et inter-lié que le LLM possède. La connaissance est compilée une seule fois puis tenue à jour, pas re-dérivée à chaque query. À ~100 articles / 400 000+ mots, l'index complet du wiki tient dans la fenêtre de contexte d'un LLM moderne, ce qui permet la détection de doublons et la vérification des contradictions sans aucun système de récupération.

> *"the LLM is rediscovering knowledge from scratch on every question. There's no accumulation."*

> *"The knowledge is compiled once and then kept current, not re-derived on every query."*

## Pourquoi les humains abandonnent les wikis mais pas les LLM

Les humains abandonnent les wikis parce que la charge de maintenance dépasse la valeur. Les références croisées pourrissent, les résumés deviennent obsolètes, les modifications se propagent de façon incohérente. Wikipedia survit grâce à des millions de contributeurs ; les wikis privés meurent en silence.

Les LLM n'ont pas ce problème :

- **Ne s'ennuient pas** : la maintenance est la même tâche que ce soit un mardi matin ou à 23h
- **N'oublient pas les références croisées** : ils peuvent scanner tout l'index en une seule passe
- **Peuvent toucher 15 fichiers en une seule ingestion** sans perdre le contexte
- **Ne dérivent pas sur les conventions** : le document de schéma est relu à chaque opération

> *"Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass."*

## Architecture à trois couches

| Couche | Objectif | Qui modifie |
|---|---|---|
| **Raw Sources** | Documents curatés immuables : articles, papers, transcriptions, captures d'écran, données | **Humains uniquement.** Le LLM lit, ne modifie jamais. |
| **The Wiki** | Markdown généré par le LLM, organisé par type de contenu : résumés, pages d'entités, pages de concepts, comparaisons, synthèses | **Le LLM possède entièrement cette couche.** |
| **The Schema (ex : CLAUDE.md)** | Document de configuration qui *"tells the LLM how the wiki is structured, what the conventions are, and what workflows to follow"* | Maintenu par l'humain. Lu par le LLM à chaque opération. |

La couche de schéma est ce qui fait fonctionner le système. Elle définit :
- Comment les pages du wiki sont nommées
- Quel dossier reçoit quel type de contenu
- À quoi ressemble l'ingest
- À quoi ressemble la query
- À quoi ressemble le lint

Dans un vault basé sur Obsidian : le **schéma = CLAUDE.md racine + fichiers CLAUDE.md par dossier**. Même architecture, nom différent.

## Trois opérations

| Op | Déclencheur | Ce qu'elle fait |
|---|---|---|
| **Ingest** | Une nouvelle source arrive | Lire la source → discuter des enseignements avec l'utilisateur → écrire une page de résumé → mettre à jour l'index → réviser les pages d'entités/concepts pertinentes → ajouter au log |
| **Query** | L'utilisateur pose une question | Chercher dans les pages du wiki (pas les sources brutes) → synthétiser une réponse avec citations → optionnellement reverser les explorations utiles sous forme de nouvelles pages du wiki pour que les découvertes se cumulent |
| **Lint** | Périodique, planifié | Bilan de santé : identifier les contradictions entre pages, les affirmations obsolètes, les pages orphelines sans liens entrants, les références croisées manquantes |

## Ingest : le flux canonique

Quand une nouvelle source entre dans le système :

1. **Lire la source.** Lecture complète, pas seulement l'abstract.
2. **Discuter des enseignements avec l'utilisateur.** Qu'est-ce qui est réellement important ici ? Qu'est-ce qui nous a surpris ? Qu'est-ce qui contredit ce que nous avons déjà ?
3. **Écrire une page de résumé.** Nouveau fichier dans le wiki, nommé selon le schéma.
4. **Mettre à jour l'index.** Ajouter la nouvelle entrée aux pages d'index qui référencent ce type de contenu.
5. **Réviser les pages d'entités pertinentes.** Si la source mentionne une personne, un projet ou un concept pour lequel le wiki a déjà une page, mettre à jour ces pages avec la nouvelle information et créer un lien retour.
6. **Réviser les pages de concepts pertinentes.** Pareil que ci-dessus pour les idées/concepts.
7. **Ajouter au log d'ingest.** Piste d'audit : ce qui a été ingéré, quand, quelles pages ont été touchées.

**Une source touche typiquement 10 à 15 pages du wiki.** Si une source ne produit qu'une seule page, c'est qu'elle n'a pas été pleinement digérée.

## Query : le flux canonique

Quand l'utilisateur pose une question :

1. **Chercher dans les pages du wiki, pas les sources brutes.** Le wiki est déjà pré-digéré.
2. **Synthétiser une réponse avec citations.** Chaque affirmation pointe vers la page du wiki qui la soutient.
3. **Si la query révèle une lacune ou une nouvelle connexion**, la reverser dans le wiki sous forme de nouvelle page ou d'amendement. La connaissance se cumule : les queries de demain seront meilleures parce que l'exploration d'aujourd'hui a été capturée.

Cette dernière étape est critique. Les queries ne sont pas jetables. C'est ainsi que le wiki croît organiquement.

## Lint : ce qu'il détecte précisément

L'opération de lint est un bilan de santé périodique. Le gist de Karpathy nomme explicitement ces modes de défaillance :

| Mode de défaillance | À quoi cela ressemble |
|---|---|
| **Contradictions entre pages** | La page A dit "X happens monthly", la page B dit "X happens weekly" |
| **Affirmations obsolètes** | Une nouvelle source remplace une ancienne affirmation, mais l'ancienne page conserve toujours l'ancienne affirmation sans lien vers le remplacement |
| **Pages orphelines** | Une page du wiki qui n'a aucun lien entrant depuis une autre page du wiki |
| **Références croisées manquantes** | Un nom d'entité apparaît en texte brut dans le corps d'une page mais n'est pas un `[[wikilink]]` vers la page de cette entité |
| **Non-conformité au schéma** | Une page se trouve dans le mauvais dossier selon le document de schéma, ou a le mauvais format de nom de fichier |
| **Sources non digérées** | Une transcription de réunion ou un article a été ingéré mais n'a produit qu'une seule page de résumé ; aucun fan-out vers les entités/concepts |
| **Duplication** | Deux pages prétendant être l'enregistrement canonique pour la même entité (ex : `Brand Voice.md` et `Voice Guidelines.md`) |

C'est ce qu'encodent la Passe 5 (wikilinks) et la Passe 6 (duplication) du skill d'audit.

## Pourquoi la synthèse au moment de l'ingest bat le RAG au moment de la query

| Dimension | RAG (au moment de la query) | LLM Wiki (au moment de l'ingest) |
|---|---|---|
| Coût de calcul par query | Élevé : le LLM re-dérive à chaque fois | Faible : le wiki est pré-digéré |
| Qualité dans le temps | Statique : mêmes chunks, mêmes réponses | Cumulative : les découvertes au moment de la query sont reversées |
| Références croisées | Aucune par nature | Obligatoires ; les orphelins sont des échecs de lint |
| Contradictions | Invisibles jusqu'à ce qu'elles tombent dans la même query | Détectées par le lint périodique |
| Information obsolète | Persiste silencieusement | Détectée par le lint quand des sources plus récentes arrivent |
| Provenance | Chaque chunk → citation de source | Page du wiki → citation de source, avec le raisonnement préservé |
| Maintenance | Nulle (et c'est bien là le problème) | Réelle, mais automatisée : le LLM s'en charge |

## Structure du document de schéma

Un document de schéma fonctionnel couvre :

- **Organisation des dossiers** : quel type de contenu va où
- **Conventions de nommage** : `YYYY-MM-DD-title.md`, `Person Name.md`, etc.
- **Types de pages** : résumé, entité, concept, comparaison, décision, note de réunion
- **Règles de références croisées** : chaque nom d'entité dans un corps devient un `[[wikilink]]` ; les fichiers orphelins sont signalés
- **Workflow d'ingest** : le flux en 7 étapes ci-dessus
- **Workflow de query** : chercher d'abord dans le wiki, citer, reverser les constats
- **Workflow de lint** : ce qu'il faut scanner, à quelle fréquence
- **Standard de frontmatter** : champs requis par type de page

Dans un vault construit sur ce modèle, le CLAUDE.md racine est le document de schéma.

## Règles strictes

| # | Règle | Auditable |
|---|---|---|
| 1 | Le document de schéma définit la structure, les conventions de nommage et les trois workflows | ✅ Passe 4 (couverture du routage) |
| 2 | Chaque source ingérée produit plusieurs modifications en aval, pas une seule note | ⚠️ Passe 3 / informatif |
| 3 | Les références croisées sont **obligatoires** ; les orphelins sont des échecs de lint | ✅ Passe 5 / 9 |
| 4 | Les affirmations obsolètes (remplacées par des sources plus récentes) sont des échecs de lint | ⚠️ Difficile à détecter : signaler pour revue manuelle |
| 5 | Les contradictions entre pages sont des échecs de lint | ✅ Passe 6 (duplication) |
| 6 | Les découvertes au moment de la query sont reversées pour que le wiki se cumule | — |
| 7 | L'humain possède : la curation des sources, le fait de poser de bonnes questions. Le LLM possède : l'écriture, le linkage, la maintenance | — |
| 8 | Les sources brutes sont **immuables**. Jamais modifiées par le LLM | ✅ (l'audit peut signaler si Raw/ a été édité) |
| 9 | À ~100 articles / 400k mots, l'index du wiki tient dans les fenêtres de contexte modernes | — |

## Le fan-out de 10 à 15 pages

La règle la plus opérationnelle du gist : **un ingest touche typiquement 10 à 15 pages du wiki.** Ce n'est pas un nombre arbitraire : c'est à quoi ressemble une "digestion approfondie" dans la pratique.

Si votre skill ou workflow d'ingest n'écrit qu'un seul fichier de résumé par source :

- La source n'a **pas été pleinement digérée**
- Les queries futures rateront le contexte
- Le wiki ne se cumulera pas

À quoi ressemble le fan-out pour une transcription de réunion :

1. Nouveau fichier : `meetings/team-standups/2026-04-30.md` (le résumé)
2. Mise à jour : la page de profil de chaque participant avec les action items
3. Mise à jour : toutes les pages de projet mentionnées dans la réunion
4. Mise à jour : toutes les pages de décision qui ont été évoquées
5. Mise à jour : l'index des réunions récentes
6. Nouveau : un enregistrement de décision si une décision a été prise
7. Mise à jour : la liste des "active topics" du schéma si un nouveau sujet a émergé

Facilement plus de 10 modifications.

## Ce que possède l'humain vs ce que possède le LLM

| L'humain possède | Le LLM possède |
|---|---|
| La curation des sources (qu'est-ce qui vaut la peine d'être ingéré ?) | L'écriture des résumés |
| Poser de bonnes questions | Les liens croisés |
| La direction de l'analyse | La maintenance de l'index |
| Les modifications stratégiques du schéma | L'exécution du lint |
| La résolution des contradictions une fois remontées | Le signalement des contradictions |
| La curation du log d'ingest | L'ajout au log d'ingest |

Le travail de l'humain, c'est la **direction**. Le travail du LLM, c'est la **maintenance**. C'est exactement pourquoi les humains abandonnent les wikis (ils détestent la maintenance) et pourquoi les LLM rendent les wikis viables.

## À faire

- Pré-digérer à l'ingest, pas à la query.
- Maintenir religieusement le document de schéma / CLAUDE.md : c'est la constitution.
- Exécuter le lint périodiquement (ce skill en est un).
- Reverser les constats des queries dans le wiki.
- Toucher 10 à 15 pages par ingestion.
- Utiliser le wiki pour la connaissance cumulative ; réserver les sources brutes comme canon immuable.
- Quand une source contredit une page existante, lier la nouvelle page → l'ancienne, marquer l'ancienne comme remplacée.
- Quand une query fait émerger une nouvelle connexion, écrire une nouvelle page du wiki ou un amendement.
- Utiliser l'opération de lint comme une corvée récurrente, pas un coup unique.

## À ne pas faire

- Ne pas laisser le RAG tout re-dériver à chaque query.
- Ne pas autoriser les pages orphelines : chaque page du wiki a besoin d'au moins un lien entrant.
- Ne pas laisser des affirmations obsolètes cohabiter avec des affirmations à jour sans lier le remplacement.
- Ne pas ingérer une source et n'écrire qu'un seul résumé : propager vers les entités/concepts.
- Ne pas modifier les sources brutes.
- Ne pas laisser les humains faire la maintenance : les LLM sont meilleurs pour cela.
- Ne pas négliger le document de schéma. Sans lui, le wiki dérive vers des dossiers que personne ne retrouve.

## Citations verbatim

> *"the LLM is rediscovering knowledge from scratch on every question. There's no accumulation."*

> *"The knowledge is compiled once and then kept current, not re-derived on every query."*

> *"Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass."*

> Lint identifies *"contradictions between pages, stale claims that newer sources have superseded, orphan pages with no inbound links."*

## Signaux auditables

Quand ce skill exécute la Passe 5 (lint des wikilinks) et la Passe 6 (duplication) pour la couche wiki :

- **Notes orphelines** : tout fichier `.md` avec zéro `[[wikilinks]]` entrant. (Réserve : les fichiers d'index et les notes quotidiennes sont des orphelins intentionnels ; exclure `index.md`, `Daily/`, les fichiers correspondant à `YYYY-MM-DD.md`.)
- **Wikilinks morts** : `[[target]]` où aucun fichier de ce nom n'existe. Suggérer le nom de fichier le plus proche par distance de Levenshtein.
- **Références croisées manquantes** : quand un nom d'entité connu apparaît en texte brut dans un corps, suggérer la conversion en `[[wikilink]]`. (Heuristique : entité connue = un fichier portant exactement ce nom existe dans le vault.)
- **Doublons de même rôle** : heuristique de nom de fichier : `voice.md`, `brand.md` et `tone.md` couvrent probablement le même rôle ; signaler à l'utilisateur pour consolider ou différencier.
- **Ingests à résumé unique** : fichiers de réunion/transcription où la seule modification est un fichier de résumé (aucune édition d'entité/projet/concept en aval le même jour). Signaler comme "potentiellement non digéré".
- **Non-conformité au schéma** : fichier dans un dossier non listé dans la table de routage du document de schéma.
- **Modification de source brute** : si un dossier `Raw/` ou `sources/` existe, signaler tout fichier qui y présente des modifications récentes par le LLM (vérifier le git log pour des commits non humains).

## Critiques de praticiens

D'après Nate B Jones, "Karpathy's Wiki vs. Open Brain" (2026-04-22) :

- L'approche wiki est **excellente pour les connexions et la synthèse**, mais **fragile quand on a besoin d'une récupération exacte depuis des données structurées**.
- L'associer à une base de données graphe ou à un store relationnel pour interroger des données tabulaires (catalogues, fiches clients, transactions financières).
- Le wiki est une couche cerveau. Ne pas mettre dans du markdown des données qui nécessitent agrégation, filtrage ou intégrité transactionnelle.

C'est pourquoi `agentic-rag-knowledge-graph` de Cole Medin et des stacks hybrides similaires existent : markdown pour le narratif + Postgres/Neo4j pour les données structurées + serveur MCP comme interface unifiée.

L'audit du vault n'impose pas cette frontière : c'est une décision architecturale propre à chaque projet, mais le skill **signale** bien quand de longues données structurées apparaissent dans du markdown (Passe 4 / Passe 9).

## Sources

- https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f (the original gist)
- https://venturebeat.com/data/karpathy-shares-llm-knowledge-base-architecture-that-bypasses-rag-with-an
- Nate B Jones, "Karpathy's Wiki vs. Open Brain. One Fails When You Need It Most." (2026-04-22)
- Corey Ganim / Nick B Zark, "Claude + Karpathy's Second Brain is INSANE" (2026-04-07)
- Brad Bonanno, "Build Your Own Self Improving AI Wiki in 11 Minutes" (2026-04-15)
