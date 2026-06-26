# F2 : Karpathy LLM Wiki (implémentation de la passe)

**Référence (le pourquoi) :** `references/karpathy-llm-wiki.md`.
**S'applique à :** la couche de contenu du wiki, c'est-à-dire chaque fichier classé comme `note`, `context`, `decision`, `meeting`, `index`, `readme`. F2.6 s'exécute aussi sur `root-claude` (vérification du document de schéma).

## Fonctionnement de cette passe

Agentique. Chaque vérification associe une **heuristique de déclenchement** (remontée de candidats peu coûteuse) à un **jugement de l'agent** (lire le contexte, raisonner, décider). Les constats incluent un `reasoning` spécifique au cas. Voir l'introduction de F1 pour le pattern complet.

Construire les index décrits à l'étape 1.3 de SKILL.md une seule fois avant de lancer F2 (index des noms de fichiers, index des liens entrants, table de routage).

## Sommaire

1. [F2.1 : Complétude du document de schéma](#f21--complétude-du-document-de-schéma)
2. [F2.2 : Wikilinks morts](#f22--wikilinks-morts)
3. [F2.3 : Pages orphelines](#f23--pages-orphelines)
4. [F2.4 : Références croisées manquantes](#f24--références-croisées-manquantes)
5. [F2.5 : Doublons de même rôle](#f25--doublons-de-même-rôle)
6. [F2.6 : Non-conformité au schéma (routage)](#f26--non-conformité-au-schéma-routage)
7. [F2.7 : Notes ébauches](#f27--notes-ébauches)
8. [F2.8 : Sources non digérées](#f28--sources-non-digérées)
9. [F2.9 : Modification de source brute](#f29--modification-de-source-brute)
10. [Schéma de constat](#schéma-de-constat)

---

## F2.1 : Complétude du document de schéma

**Règle du framework :** le document de schéma (CLAUDE.md racine) définit l'organisation des dossiers, le nommage, les types de pages, les règles de référence croisée, les workflows d'ingestion/requête/lint, le frontmatter. Sans lui, le wiki dérive vers des dossiers que personne ne retrouve.

**Heuristique de déclenchement :** lire le CLAUDE.md racine. Évaluer la présence des sept composants du schéma d'après les titres de section et le contenu du corps (pas seulement la correspondance de titre : un titre sans contenu dans le corps ne compte pas).

**Jugement de l'agent :**
- Pour chaque composant manquant, décider s'il est réellement absent ou si le projet le gère différemment (par exemple, des conventions de nommage documentées dans un fichier séparé `Resources/conventions.md` et liées depuis CLAUDE.md → compte quand même).
- Le raisonnement explique *quels* composants sont faibles et quelle en est la conséquence pour les utilisateurs qui exécutent les workflows d'ingestion/requête.

**Faux positifs à ignorer :**
- Les vaults minuscules (<20 fichiers) où un schéma formel est superflu : signaler avec une sévérité moindre, recommander un schéma léger.

**Sévérité :** fail si <4 composants présents, warn si 4 à 6.

**Format du constat :**
```
./CLAUDE.md — schema doc missing components: {list}
Reasoning: {what specifically is missing and how it affects ingest/query — e.g., "no naming convention defined → meeting files named inconsistently across folders, search by name fails"}
Action: add the missing components (link to existing docs if they live elsewhere)
Citation: karpathy-llm-wiki.md → Schema doc structure
```

**Correction automatique :** aucune.

---

## F2.2 : Wikilinks morts

**Règle du framework :** les liens morts sont des échecs de lint.

**Heuristique de déclenchement :** pour chaque `[[target]]` (regex `\[\[([^\]|#]+)(?:\|[^\]]+)?\]\]`), retirer les ancres de section et les alias, puis chercher dans `vault_filename_index`. Si absent → candidat.

**Jugement de l'agent :** pour chaque candidat mort :
- Lire la ligne environnante. Le lien est-il une vraie référence, ou se trouve-t-il dans un bloc de code illustrant la syntaxe wikilink ? Ignorer les démonstrations de syntaxe.
- Calculer les 3 noms de fichiers les plus proches par distance de Levenshtein.
- Lire brièvement les fichiers cibles candidats. Choisir le meilleur réacheminement en raisonnant sur le sens de la ligne qui entoure le lien.
- S'il n'existe pas de bon réacheminement → recommander « retirer le lien, garder le texte ».
- Raisonnement : pourquoi ce réacheminement précis plutôt que les autres.

**Faux positifs à ignorer :**
- Les wikilinks à l'intérieur de blocs de code ou de code inline (démonstrations de syntaxe).
- Les wikilinks vers des systèmes externes (patterns `[[external://something]]`).

**Sévérité :** warn.

**Format du constat :**
```
{file}:{line} — dead wikilink [[{target}]]
Excerpt: "{line content}"
Reasoning: {why the suggested repoint is the right one — what the surrounding sentence is referring to}
Closest matches: {top-3}
Action: (a) repoint to [[{best-match}]] OR (b) remove link, keep text
Citation: karpathy-llm-wiki.md → Lint catches dead links
```

**Correction automatique : corrigible** avec confirmation de l'utilisateur pour chaque constat.

---

## F2.3 : Pages orphelines

**Règle du framework :** chaque page de wiki a besoin d'au moins 1 lien entrant.

**Heuristique de déclenchement :** les fichiers où `inbound_link_index[file] == 0`.

**Jugement de l'agent :** pour chaque candidat orphelin :
- Est-ce intentionnel ? (Note quotidienne indexée par date, racine de profil, archive, transcriptions.) Le fichier de passe liste les patterns d'orphelins intentionnels connus ; l'agent confirme en lisant le rôle et le contenu du fichier.
- Si réellement inaccessible, que faut-il faire ? Options :
  - Lier depuis une note parente/index (laquelle ?)
  - Déplacer vers l'archive (le contenu est-il obsolète ?)
  - Supprimer (le contenu est-il redondant ?)
- Le raisonnement choisit la bonne action en lisant le contenu du fichier.

**Orphelins intentionnels** (exclus des constats) :
- `Daily/*.md`, `**/*\d{4}-\d{2}-\d{2}\.md`
- `**/index.md`, `**/README.md`, `**/CLAUDE.md`, `**/MEMORY.md`
- Les fichiers dans `Intelligence/archive/`
- Les fichiers racine de profil (`Team/*/Profiles/*/{Name}.md`)

**Sévérité :** warn.

**Format du constat :**
```
{path} — orphan (no inbound wikilinks)
Reasoning: {what the file contains and why it should be reachable — e.g., "this captures the X decision but no other note links to it; it'd be lost if anyone looked for X-related context"}
Action: {specific recommendation: "link from {index file}", "move to archive", or "delete"}
```

**Correction automatique :** aucune.

---

## F2.4 : Références croisées manquantes

**Règle du framework :** un nom d'entité apparaît en texte brut alors qu'un fichier portant ce nom existe → devrait être un `[[wikilink]]`.

**Heuristique de déclenchement :** pour chaque nom de fichier en forme d'entité (plusieurs mots, PascalCase, ou comportant des majuscules ; ignorer les génériques comme `notes`, `index`, `readme`), chercher dans le vault les occurrences en texte brut qui ne sont PAS à l'intérieur de `[[...]]`, `[...](...)`, blocs de code ou code inline.

**Jugement de l'agent :** pour chaque occurrence en texte brut :
- Lire la phrase complète. Le nom est-il utilisé comme l'entité (renvoie à la page du wiki) ou de façon incidente/dans une citation ?
- « Stripe » dans « Set up Stripe webhook » → référence d'entité, la lier.
- « Stripe » dans « We're not using Stripe anymore » → reste une référence d'entité, la lier.
- « Stripe » dans un message cité d'un utilisateur : cas de jugement ; si la citation parle de la même entité Stripe, la lier ; si c'est une simple coïncidence de mot, ignorer.
- Le raisonnement explique pourquoi cette occurrence précise devrait ou non être liée.

**Faux positifs à ignorer :**
- Les titres (le titre lui-même ne devrait généralement pas être un wikilink).
- Les valeurs de frontmatter.
- Le fichier qui EST la page de l'entité (ne pas lier un fichier à lui-même).
- Les correspondances de mots génériques (ignorer les noms de fichiers < 4 caractères, ignorer les mots isolés ambigus).

**Sévérité :** warn.

**Format du constat :**
```
{file}:{line} — entity "{name}" appears as plain text; {target}.md exists
Excerpt: "{surrounding sentence}"
Reasoning: {why this specific occurrence is an entity reference, not a generic word}
Action: convert to [[{name}]]
Citation: karpathy-llm-wiki.md → Missing cross-references
```

**Correction automatique : corrigible sur acceptation de l'utilisateur**, par occurrence (ne pas remplacer en masse ; certaines occurrences sont intentionnellement en texte brut).

---

## F2.5 : Doublons de même rôle

**Règle du framework :** deux pages prétendant être l'enregistrement canonique de la même entité constituent un échec de lint.

**Heuristique de déclenchement :** clusters de noms de fichiers dans `Context/` (voice/brand/tone, icp/customer-profile/audience, services/offers/products, me/profile/operator, strategy/goals/okrs, team/org). Plus les paires, dans n'importe quel dossier, où la distance de Levenshtein ≤3 entre les noms de base.

**Jugement de l'agent :** pour chaque paire candidate, **lire les deux fichiers**.
- Se recouvrent-ils réellement, ou couvrent-ils des aspects différents (par exemple, `voice.md` = comment nous sonnons, `brand.md` = identité visuelle + voix + valeurs) ?
- Identifier les sections qui se recouvrent. Le raisonnement cite les sections en chevauchement par leur titre.
- S'ils sont complémentaires → recommander une différenciation par la portée du frontmatter ou un renommage pour lever l'ambiguïté.
- S'ils se recouvrent → recommander une consolidation ; choisir le nom de fichier canonique en raisonnant sur celui qui est le plus conventionnel / le plus lié.

**Faux positifs à ignorer :**
- Les fichiers où l'un est un brouillon en cours et l'autre est archivé (lire le `status:` du frontmatter).
- Les paires où un fichier est clairement un index parent et l'autre une feuille.

**Sévérité :** warn.

**Format du constat :**
```
Potential overlap in {folder}/:
  - {path1} ({bytes-a}B)
  - {path2} ({bytes-b}B)
Reasoning: {what the overlap is — name 2-3 sections present in both, paraphrase what they say in each}
Action: {specific: "consolidate into {chosen-canonical}.md" OR "differentiate by adding scope: {field} to frontmatter" OR "rename {path2} to {disambiguated-name}.md"}
```

**Correction automatique :** aucune.

---

## F2.6 : Non-conformité au schéma (routage)

**Règle du framework :** le document de schéma définit le routage par dossier ; les fichiers non conformes sont des échecs de lint.

**Heuristique de déclenchement :**
1. Lire le CLAUDE.md racine, extraire la table de routage / routage des connaissances.
2. Lister chaque entrée de premier niveau (dossier, fichier).
3. Les fichiers à la racine du vault qui ne sont pas `CLAUDE.md`/`README.md`/`MEMORY.md`/`index.md` → candidat.
4. Les dossiers de premier niveau absents de la table de routage → candidat.

**Jugement de l'agent :**
- Pour chaque fichier à la racine du vault : le lire. Décider s'il est mal placé (le déplacer où ?) ou légitime (rare ; recommander de l'ajouter à la liste des exceptions autorisées).
- Pour chaque dossier non mappé : lire son contenu. Décider s'il faut étendre la table de routage ou relocaliser le contenu du dossier.
- Le raisonnement explique la bonne destination pour chaque élément non conforme.

**Faux positifs à ignorer :**
- `LICENSE.md`, `CHANGELOG.md`, `CONTRIBUTING.md` : fichiers conventionnels de racine de dépôt.
- `.github/`, `.vscode/`, `.cursor/`, `.claude/` : dossiers d'outillage.

**Sévérité :**
- violation par fichier à la racine du vault → fail
- dossier non mappé → warn
- pas de table de routage → fail (un seul constat au total)

**Format du constat (fichier à la racine du vault) :**
```
./{filename} — file in vault root
Reasoning: {what the file contains and where it belongs per the routing table}
Action: move to {specific folder}, or delete if obsolete
```

**Format du constat (dossier non mappé) :**
```
./{folder}/ — top-level folder not in routing
Reasoning: {what the folder holds and how it relates to mapped folders}
Action: add to routing table OR relocate contents to {specific mapped folder}
```

**Format du constat (pas de table de routage) :**
```
./CLAUDE.md — no routing table found
Reasoning: without a routing table, the wiki schema is unenforceable; new content drifts into ad-hoc folders
Action: add a routing table that maps every top-level folder to a content type
```

**Correction automatique :** aucune.

---

## F2.7 : Notes ébauches

**Règle du framework :** les notes non digérées ou vides polluent le wiki.

**Heuristique de déclenchement :**
- taille en octets < 200 après retrait du frontmatter (ébauche stricte)
- Le corps correspond à : un simple H1, uniquement le frontmatter, ou seulement `TODO`, `WIP`, `Coming soon`, `Placeholder`, `Lorem ipsum`, `Draft`, `xxx`.

**Jugement de l'agent :**
- Pour chaque candidat, lire le fichier. Est-ce intentionnellement un index d'une ligne (par exemple, `Resources/quick-links.md` avec trois puces) ? → ce n'est pas une ébauche.
- Est-ce une vraie ébauche, c'est-à-dire un fichier créé mais jamais rempli ? → signaler.
- Le raisonnement juge si le fichier est récupérable (à compléter), redondant (à supprimer) ou en cours (à déplacer vers drafts/).

**Faux positifs à ignorer :**
- Les fichiers de onboarding/templates/ (modèles par nature).
- Les fichiers index/README qui routent légitimement en 1 ligne.
- Les fichiers explicitement marqués `status: draft` dans le frontmatter (déjà connus comme en cours).

**Sévérité :** warn.

**Format du constat :**
```
{path} — stub ({bytes}B, {n} content lines)
Excerpt: "{first 80 chars}"
Reasoning: {what the file appears to be aiming at, based on filename + any present headings; what the user can recover}
Action: {specific: "fill in: {what's missing}" OR "delete (redundant with {other-file})" OR "move to drafts/"}
```

**Correction automatique :** aucune.

---

## F2.8 : Sources non digérées

**Règle du framework :** une ingestion touche généralement 10 à 15 pages de wiki. Une source qui ne produit qu'un seul fichier de résumé n'a pas été pleinement digérée.

**Heuristique de déclenchement :** pour les fichiers `meeting` et `transcript` :
1. Extraire la date de la source.
2. Examiner le git log (ou le mtime du système de fichiers si git n'est pas disponible) pour les fichiers modifiés dans une fenêtre de ±2 jours.
3. Compter les fichiers distincts modifiés hors du dossier de la source elle-même. <3 → candidat.

**Jugement de l'agent :** pour chaque candidat :
- Lire le fichier source. Quelles entités, projets, décisions mentionne-t-il ?
- Vérifier si ces fichiers d'entité/projet existent dans le vault et ont été mis à jour à une date proche de celle de la source.
- Si la source est courte/peu importante (par exemple, un appel de 5 min) → un faible essaimage peut être acceptable. Ignorer.
- Si la source est substantielle mais qu'aucune activité en aval n'a suivi → signaler avec un raisonnement listant les propagations manquées.

**Faux positifs à ignorer :**
- Les fichiers source explicitement marqués `processed: true` dans le frontmatter.
- Les sources triviales (transcriptions très courtes).
- Les vaults sans historique git (ignorer ; l'agent n'a aucun signal).

**Sévérité :** warn.

**Format du constat :**
```
{path} — possibly undigested source
Reasoning: source mentions {entities/projects/decisions} but only {n} downstream files modified near the source date; {specific entity-file} appears to need updating
Action: revisit; update {entity-file-1}, {project-file-2}, etc.
Citation: karpathy-llm-wiki.md → 10–15 page fan-out
```

**Correction automatique :** aucune.

---

## F2.9 : Modification de source brute

**Règle du framework :** les sources brutes sont immuables.

**Heuristique de déclenchement :** détecter les dossiers de sources brutes (`Raw/`, `Sources/`, `sources/`, `raw-sources/`, `inputs/`). Vérifier le git log pour des auteurs de commit non humains.

**Jugement de l'agent :** pour chaque source brute modifiée :
- Lire le git diff. Le changement porte-t-il sur le contenu (le LLM a réécrit la source) ou sur les métadonnées (le LLM a ajouté du frontmatter) ? Les deux sont des violations, mais le raisonnement diffère.
- Le raisonnement décrit ce qui a été modifié et pourquoi cela importe.

**Faux positifs à ignorer :**
- Les vaults sans git → ignorer silencieusement.
- Les dossiers qui *portent le nom* `raw` mais ne sont pas réellement des archives de sources.

**Sévérité :** fail.

**Format du constat :**
```
{path} — raw source modified by automated commit
Reasoning: {what was changed and why immutability matters for this source's downstream wiki pages}
Action: revert; raw sources are immutable canon
```

**Correction automatique :** aucune.

---

## Schéma de constat

Même forme qu'en F1 : chaque constat possède un `reasoning`. Voir l'étape 2.4 de SKILL.md.
