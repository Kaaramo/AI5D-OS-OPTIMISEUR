# F8 : Reflection (implémentation de la passe)

**Référence (le pourquoi) :** `references/anthropic-dreams.md` (inspiré de Anthropic Managed Agents Dreams).
**S'applique à :** la couche curatée du vault, découverte via le registre des rôles (Étape 1.5). La couche curatée correspond à **chaque rôle avec `layer == "curated"`** (standard ou personnalisé : `Building/`, `Garden/`, les dossiers utilisateur personnalisés participent tous s'ils ont été classés curatés). La couche de session (lue comme preuve, jamais écrite par les sorties de F8) correspond à **chaque rôle avec `layer == "session"`**. Tout rôle standard marqué `missing` est silencieusement ignoré : son absence n'est pas un échec F8 (c'est un constat F9.0). Les rôles personnalisés ne produisent jamais de constats "missing" ; ils sont soit présents, soit absents.

**Hors périmètre (jamais touché par les correctifs F8) :** CLAUDE.md / claude.md (tout dossier), SKILL.md, .claude/rules/, les fichiers plus anciens que la fenêtre de session dans tout dossier de rôle de session, tout ce qui figure dans la liste d'exclusion technique.

**Résolution des chemins à l'exécution :** chaque référence ci-dessous à "la couche curatée" ou "la couche de session" se résout via `[role.path for role in registry if role.layer == X]`. Les références à des couches spécifiques ("le dossier des décisions", "le dossier des notes quotidiennes") recherchent `[role for role in registry if role.name == Y]` : et si `Y` est absent, la vérification utilise la meilleure alternative de l'agent dans la bonne couche, ou est ignorée. Les exemples de ce document utilisent des noms d'emplacement comme `Context/`, `Building/` pour la lisibilité : à l'exécution, on ne suppose jamais de noms littéraux.

## Fonctionnement de cette passe

Synthèse. Contrairement à F1–G7, le déclencheur n'est pas "scanner un fichier avec une regex". Le déclencheur est "construire des clusters inter-fichiers, puis juger chaque cluster". Les constats émergent du cluster, pas des lignes individuelles.

Chaque constat F8 fournit une proposition de correctif concrète. Les correctifs F8 sont en **revue pas à pas uniquement** : l'application en masse n'est pas sûre, car chaque contradiction, fusion ou promotion nécessite que l'utilisateur choisisse la cible gagnante.

## Sommaire

1. [Setup : fenêtres, clusters, périmètre](#setup--fenêtres-clusters-périmètre)
2. [F8.1 : Contradictions](#f81--contradictions)
3. [F8.2 : Candidats à la fusion](#f82--candidats-à-la-fusion)
4. [F8.3 : Entrées périmées](#f83--entrées-périmées)
5. [F8.4 : Thèmes émergents](#f84--thèmes-émergents)
6. [F8.5 : Promotions](#f85--promotions)
7. [Contraintes inter-frameworks](#contraintes-inter-frameworks)
8. [Schéma des constats](#schéma-des-constats)

---

## Setup : fenêtres, clusters, périmètre

**Fenêtre de session :** les 30 derniers jours, selon la mtime du fichier ou le préfixe de date dans le nom de fichier (`YYYY-MM-DD`). Remplaçable via `instructions` si l'utilisateur spécifie une autre cadence.

**Ensemble de la couche curatée :** chaque `.md` sous chaque rôle où `layer == "curated"`. Cela inclut les rôles curatés standard (context, projects, decisions, resources, identity, skills) et les rôles curatés personnalisés de l'utilisateur (`Building/`, `Garden/`, tout autre élément classé curaté à l'Étape 1.5.3).

**Ensemble de session :** chaque `.md` sous chaque rôle où `layer == "session"` dont la mtime ou le préfixe de date tombe dans la fenêtre de session. Inclut les rôles de session standard (daily, meetings, transcripts) et les rôles de session personnalisés (`Inbox/`, dossiers de capture personnalisés).

**Couches exclues :** `archive`, `meta`, `unknown`. Le contenu d'archive reste archivé ; meta relève de l'outillage/système ; les rôles unknown reçoivent une demande de clarification via F9.0 avant de participer.

Si le registre des rôles n'a pas d'`identity`/`context` (standard ou personnalisé), F8 s'exécute quand même, mais ses jugements perdent leur ancrage dans le monde déclaré par l'utilisateur. Le raisonnement de chaque constat F8 doit explicitement signaler quand l'ancrage contextuel est indisponible.

**Clusters thématiques :** regrouper les fichiers par recouvrement de :
- cibles de wikilink partagées (≥2 références `[[Target]]` communes),
- tags de frontmatter partagés (≥2 tags communs),
- similarité des tokens de nom de fichier (Jaccard ≥ 0.5 sur les basenames tokenisés),
- noms propres / entités répétés (≥3 expressions capitalisées partagées).

Un cluster est `{file_a, file_b, …, evidence}`. Lire les 1500 premiers caractères de chaque membre + l'index des en-têtes avant de juger.

---

## F8.1 : Contradictions

**Règle du framework :** deux notes ou plus affirment des faits opposés sur la même entité, décision ou principe.

**Heuristique de déclenchement :** au sein d'un cluster, rechercher les paires opposées :
- désaccord numérique sur la même métrique (`50% B2B` vs `100% B2C`),
- revirement de décision (`going with Postgres` vs `migrating to DynamoDB`),
- inversion de principe (`always X` vs `never X`),
- discordance de date/propriétaire sur le même projet.

**Jugement de l'agent :** lire les deux passages avec leur contexte environnant. Décider :
- **Vraie contradiction** → les deux fichiers prétendent faire autorité, aucun marqueur d'évolution, aucune différence de périmètre.
- **Évolution, pas contradiction** → le fichier le plus récent supplante explicitement l'ancien ; l'ancien fichier devrait être marqué comme périmé (router vers F8.3 à la place).
- **Faux positif** → les deux affirmations s'appliquent à des périmètres, des périodes ou des audiences différents. Abandonner.

**Proposition de correctif :** identifier le *gagnant* (par défaut, la source faisant autorité la plus récente ; l'utilisateur confirme). Réécrire le perdant pour soit (a) déférer au gagnant via un wikilink, soit (b) supprimer entièrement la ligne contredite. Si les deux fichiers sont porteurs, consigner la résolution dans `roles.decisions.path` (ou, si le rôle decisions est absent, faire remonter cette lacune dans la proposition de correctif : "aucun dossier de décisions découvert ; recommander d'en créer un").

**Sévérité :** fail.
**Corrigible :** true (revue pas à pas uniquement). L'utilisateur choisit le gagnant pour chaque item.

---

## F8.2 : Candidats à la fusion

**Règle du framework :** N notes (N ≥ 2) couvrent essentiellement le même concept. Le vault est fragmenté ; la couche curatée devrait avoir une seule entrée canonique.

**Heuristique de déclenchement :** clusters où :
- similarité de titre ≥ 0.6 (Jaccard de tokens) **et**
- recouvrement des cibles de wikilink partagées ≥ 0.4 **et**
- nombre combiné de liens entrants ≥ 1 (au moins une note du cluster est référencée ailleurs).

**Jugement de l'agent :** lire le corps de chaque membre du cluster. Décider :
- **Fusionner** → ≥60% de recouvrement de contenu ; aucun membre ne contient un sous-thème unique justifiant une séparation.
- **Lier sans fusionner** → les membres couvrent des facettes liées mais distinctes. Router vers F2.4 (références croisées manquantes) à la place.
- **Garder séparés** → les membres ont des audiences distinctes (par ex. `Context/brand.md` pour la voix, `Resources/swipe/copy.md` pour les exemples).

**Proposition de correctif :** choisir la cible canonique (par défaut, le plus grand nombre de liens entrants ; l'utilisateur confirme). Fusionner le contenu unique des sources dans la cible, rediriger chaque wikilink entrant `[[Source]]` vers `[[Target]]`, archiver les sources vers `{roles.archive.path}/{date}-merged/` (ou `archive/{date}-merged/` à la racine du vault si le rôle archive est absent, et faire remonter cette lacune à F9.0). Ne jamais supprimer : l'archivage préserve la trace.

**Vérification du budget F5 :** si le fichier fusionné dépassait le budget par fichier recommandé par F5 (10KB), proposer une **fusion partielle** (déplacer uniquement les sections qui se recouvrent ; garder séparés les fichiers de sous-thèmes distincts) ou rétrograder en simple signalement avec justification.

**Sévérité :** warn.
**Corrigible :** true (revue pas à pas uniquement). L'utilisateur choisit la cible canonique pour chaque cluster.

---

## F8.3 : Entrées périmées

**Règle du framework :** une entrée de la couche curatée énonce une hypothèse que du contenu de session récent a supplantée.

**Heuristique de déclenchement :**
- Pour chaque fichier de la couche curatée, extraire les phrases d'affirmation (faits déclaratifs, au présent, taggés en frontmatter ou ancrés en H2).
- Pour chaque affirmation, chercher dans l'ensemble de session un langage contradictoire ou supplantant (mots-clés de décision : `decided`, `pivot`, `now we`, `changed to`, `replaced`, `instead of`).
- Apparier par recouvrement d'entités (≥2 noms propres partagés ou ≥1 cible de wikilink partagée).

**Jugement de l'agent :** lire les deux. Décider :
- **Périmé** → l'affirmation curatée est contredite par une source de la couche de session que l'utilisateur a écrite il y a ≥7 jours et qu'il n'a pas révoquée depuis.
- **Dépassé mais toujours valide** → l'affirmation est encore opérationnellement vraie ; la session est un cas isolé. Abandonner.
- **Désaccord actif** → plusieurs sessions sont en désaccord. Router vers F8.1 (contradiction) à la place.

**Proposition de correctif :** réécrire l'entrée curatée avec le nouvel état. Citer la source supplantante avec un wikilink. Déplacer l'ancienne formulation vers une section `## History` si elle porte un contexte décisionnel, sinon l'abandonner.

**Sévérité :** fail.
**Corrigible :** true (revue pas à pas uniquement). L'utilisateur approuve la réécriture pour chaque item.

---

## F8.4 : Thèmes émergents

**Règle du framework :** ≥3 notes dans la fenêtre de session convergent vers un sujet qui n'a aucune entrée canonique dans la couche curatée.

**Heuristique de déclenchement :**
- Clusteriser les fichiers de l'ensemble de session par recouvrement d'entités/wikilinks (mêmes règles que le setup).
- Pour chaque cluster de ≥3 fichiers, chercher dans la couche curatée une entrée canonique existante (correspondance de nom de fichier, correspondance de H1, ou cible de wikilink).
- Si aucune n'existe → candidat.

**Jugement de l'agent :** lire le cluster. Décider :
- **Thème durable** → sujet récurrent avec contenu concret ; mérite une entrée Context ou une MOC.
- **Bavardage ponctuel** → le cluster est accessoire (même client mentionné dans 3 réunions sans rapport). Abandonner.
- **Relève d'une entrée existante** → le thème recoupe une entrée curatée que le déclencheur a manquée. Router vers F8.5 (promotion) en ciblant cette entrée.

**Proposition de correctif :** créer l'entrée du thème dans le rôle non absent le plus approprié : `roles.context.path/{theme-slug}.md` pour les thèmes de connaissance canonique, `roles.resources.path/{theme-slug}-MOC.md` pour les thèmes de type index. Si le bon rôle est absent → proposer de l'adopter dans le cadre du correctif (le faire remonter à F9.0 simultanément) avant de créer le fichier. Synthétiser le contenu du cluster en une entrée concise : définition en 1 ligne, 3 à 5 points clés, wikilinks vers les notes sources. Le frontmatter suit les conventions G7.2.

**Sévérité :** warn.
**Corrigible :** true (revue pas à pas uniquement). L'utilisateur confirme le thème + le chemin cible pour chaque item.

---

## F8.5 : Promotions

**Règle du framework :** une ligne ou un paragraphe spécifique dans un fichier éphémère de la couche de session se lit comme une connaissance durable (décision, principe, apprentissage) qui a sa place dans la couche curatée.

**Heuristique de déclenchement :** au sein de l'ensemble de session, rechercher des expressions marqueuses :
- décisions : `decided`, `chose`, `going with`, `we'll use`,
- principes : `always`, `never`, `rule:`, `principle:`,
- apprentissages : `learned`, `realized`, `key insight`, `takeaway`,
- encarts : `> [!note]`, `> [!important]`.

**Jugement de l'agent :** lire les 10 à 20 lignes environnantes. Décider :
- **Promouvoir** → durable, généralisable, non lié à un seul moment.
- **Garder à la source** → l'insight n'a de sens qu'à l'intérieur du journal de session où il se trouve.
- **Déjà promu** → la couche curatée contient déjà ce fait. Abandonner.

**Proposition de correctif :** choisir la destination parmi les rôles découverts de l'utilisateur, typiquement `roles.context.path/{x}.md` (connaissance durable), `roles.resources.path/{x}.md` (actifs réutilisables), ou `roles.decisions.path/{date}-{slug}.md` (décisions). L'agent présente à l'utilisateur les rôles qui existent ; les rôles cibles absents déclenchent un constat F9.0 en parallèle. Ajouter le contenu durable avec un wikilink vers la source. Dans la source, remplacer les lignes d'origine par un stub de wikilink (`See [[Target#section]]`).

**Sévérité :** info.
**Corrigible :** true (revue pas à pas uniquement). L'utilisateur choisit la destination pour chaque item.

---

## Contraintes inter-frameworks

Ces règles empêchent F8 d'entrer en conflit avec F1–G7. Les appliquer lors de l'émission des constats ; rétrograder en simple signalement en cas de violation.

| Contrainte | Pourquoi | Ce que fait F8 |
|---|---|---|
| F1 : ne jamais auto-éditer CLAUDE.md / claude.md | La règle de simple signalement de F1 est absolue | Si un correctif F8 modifiait un CLAUDE.md → mettre `fixable: false`, consigner le raisonnement, faire remonter en revue manuelle |
| F5 : ne jamais pousser un fichier au-delà du budget par fichier (10KB recommandé, 100KB strict) | F5 signalerait le fichier fusionné au prochain run | F8.2 vérifie la taille fusionnée avant d'appliquer ; si dépassée → proposer une fusion partielle ou rétrograder en simple signalement |
| F2 : ne jamais créer de wikilinks morts | F2.2 les signalerait | F8.2 redirige chaque `[[Source]]` entrant vers `[[Target]]` avant d'archiver la source ; vérifier zéro lien mort ensuite |
| F2 : archives orphelines | Les fichiers archivés deviennent orphelins | F8.2 archive vers `Intelligence/archive/{date}-merged/`, exclu des vérifications d'orphelins de F2.3 (convention de sous-dossier) |
| F3 : style de la couche agent sur le contenu généré | F3 signalerait le remplissage | Le texte généré par F8.4 / F8.5 suit la discipline caveman au moment de l'écriture |
| G7.1 : pas de tirets cadratins dans le contenu généré | G7 les signalerait | La création de fichiers F8 n'émet jamais `—` ni `–` (les plages numériques sont acceptables) |
| G7.2 : frontmatter sur les nouveaux fichiers | G7 signalerait les champs manquants | Les nouveaux fichiers F8.4 / F8.5 incluent `status:`, `tags:`, `type:`, `date:` |
| F6 : périmètre de SKILL.md | F6 est responsable du découpage de SKILL.md | F8 ne cible jamais les fichiers SKILL.md (périmètre couche curatée uniquement) |

Si une vérification de contrainte échoue pendant l'application d'un correctif → interrompre le correctif de ce constat, mettre `fixed: false`, consigner la raison dans `reasoning_post`. Le rendu HTML l'affiche comme FIXABLE · NOT APPLIED avec la raison de l'interruption.

---

## Schéma des constats

```json
{
  "framework": "F8",
  "check_id": "F8.2",
  "check_name": "Merge candidates",
  "path": "./Context/strategy.md",
  "cluster": [
    "./Context/strategy.md",
    "./Notes/2026-q1-strategy.md",
    "./Projects/positioning/research/strategy-thoughts.md"
  ],
  "line": null,
  "severity": "warn",
  "excerpt": "3 notes covering 'B2B positioning' with 72% content overlap; combined inbound links: 4",
  "reasoning": "These three files all define the B2B positioning thesis; Notes/2026-q1-strategy.md restates Context/strategy.md verbatim in 4 paragraphs; Projects/.../strategy-thoughts.md adds two unique sub-points (vertical-by-vertical breakdown) that should fold into the canonical Context entry. Merging keeps one source of truth without losing the unique material.",
  "action": "Merge Notes/2026-q1-strategy.md and Projects/.../strategy-thoughts.md into Context/strategy.md (canonical, highest inbound count). Redirect 4 inbound wikilinks. Archive sources to Intelligence/archive/2026-05-08-merged/. Estimated merged size: 7.2KB (under F5 10KB budget).",
  "proposed_target": "./Context/strategy.md",
  "estimated_size_after": 7234,
  "fixable": true,
  "fixed": false,
  "citation": "anthropic-dreams.md → F8.2 Merge candidates"
}
```

Le champ `reasoning` est obligatoire et doit expliquer le motif de recouvrement du cluster, pas seulement répéter "ceux-ci sont similaires".

Le tableau `cluster` liste chaque fichier du cluster (pour F8.1 et F8.2) ou juste `[path]` pour les constats sur un seul fichier (F8.3, F8.5). Pour F8.4, le cluster liste les notes sources qui ont déclenché le thème.

`proposed_target` est le chemin de destination que le correctif créerait ou dans lequel il écrirait. Pour F8.1 et F8.3, c'est le fichier en cours de réécriture ; pour F8.2, la cible de fusion canonique ; pour F8.4, le chemin du nouveau fichier ; pour F8.5, la destination de la promotion.

`estimated_size_after` est requis pour F8.2 (porte du budget F5). Optionnel ailleurs.
