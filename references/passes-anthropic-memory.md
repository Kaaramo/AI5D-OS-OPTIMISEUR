# F5 : Anthropic Memory (pass implementation)

**Référence (le pourquoi) :** `references/anthropic-managed-memory.md`.
**S'applique à :** chaque `.md` du vault.

## Fonctionnement de cette passe

Agentique. Les vérifications de taille et de nom de fichier ont des déclencheurs mécaniques, mais l'agent lit chaque candidat pour produire un raisonnement qui explique *pourquoi* la taille ou le nom de ce fichier précis pose problème dans le contexte du projet. Voir l'introduction de F1 pour le motif complet.

## Sommaire

1. [F5.1 : Taille par fichier](#f51--taille-par-fichier)
2. [F5.2 : Mono-fichiers thématiques](#f52--mono-fichiers-thématiques)
3. [F5.3 : Caractère descriptif des noms de fichiers](#f53--caractère-descriptif-des-noms-de-fichiers)
4. [F5.4 : Présence d'un index mémoire](#f54--présence-dun-index-mémoire)
5. [F5.5 : Cohérence du nommage par date](#f55--cohérence-du-nommage-par-date)
6. [F5.6 : Vérification du versionnement](#f56--vérification-du-versionnement)
7. [Schéma de constat](#schéma-de-constat)

---

## F5.1 : Taille par fichier

**Règle du framework :** par fichier ≤100KB / ~25K tokens (plafond strict). Recommandé <10KB.

**Heuristique de déclenchement :** taille en octets ; tokens ≈ octets/4.

**Jugement de l'agent :** pour chaque candidat surdimensionné, lire le fichier :
- Est-ce un seul sujet ciblé développé en longueur (un long document de référence, une transcription) ? → signaler avec un raisonnement sur la taille mais une sévérité plus basse.
- Est-ce plusieurs sujets sans rapport entassés ensemble ? → signalement fort ; le raisonnement identifie les sujets via les titres H2.
- Est-ce une transcription ou une source brute où la longueur est structurellement attendue ? → ignorer (les transcriptions ont leurs propres conventions).

**Faux positifs à ignorer :**
- Fichiers dans `*transcripts*/` ou marqués `type: transcript`.
- Fichiers servant explicitement de référence longue (ex. docs de l'API Anthropic mises en cache localement).

**Sévérité :**
- > 100KB ou > 25K tokens → fail
- > 10KB → warn

**Format du constat :**
```
{path} — {KB}KB ({tokens}t)
Reasoning: {what the file holds — single topic at length vs multiple topics; specific H2 sections present}
Action: {specific: "split at H2 sections {names}" OR "leave; legitimately long for {reason}"}
Citation: anthropic-managed-memory.md → Architecture facts
```

**Auto-correction :** aucune.

---

## F5.2 : Mono-fichiers thématiques

**Règle du framework :** plusieurs fichiers ciblés > un seul méga-fichier.

**Heuristique de déclenchement :** compter les titres H2 dans chaque fichier. >5 H2 → candidat.

**Jugement de l'agent :** pour chaque candidat, lire les titres H2 et une phrase de chaque section :
- Les H2 sont-ils des sous-sujets d'un thème cohérent unique (ex. chapitres d'un même guide) ? → pas un mono-fichier ; ignorer.
- Les H2 sont-ils des sujets sans rapport (ex. « Stripe setup », « Email automation », « Onboarding flow ») ? → mono-fichier ; signaler.
- Le raisonnement liste les H2 et juge si la scission aide à la navigation.

**Faux positifs à ignorer :**
- Documents de référence où un gros fichier est la forme canonique (ex. une référence d'API unique).
- Fichiers d'index qui listent légitimement de nombreux sous-sujets.

**Sévérité :** warn.

**Format du constat :**
```
{path} — {n} H2 sections (likely covers multiple topics)
Sections:
  - {H2-1}
  - {H2-2}
  …
Reasoning: {why these H2s appear unrelated — what would the user gain from splitting}
Action: split into {n} focused files: {suggested filenames per H2}
```

**Auto-correction :** aucune.

---

## F5.3 : Caractère descriptif des noms de fichiers

**Règle du framework :** l'agent navigue par le nom. Mauvais : `notes.md`, `untitled.md`, `temp.md`.

**Heuristique de déclenchement :** correspondance avec les motifs :
```
^notes-?\d*\.md$
^untitled.*\.md$
^temp.*\.md$
^file-?\d+\.md$
^new[-\s]?document.*\.md$
^draft\d*\.md$
^document\d*\.md$
^copy.*\.md$
^final-?(final-?)*.*\.md$
^\d+\.md$
^scratch\d*\.md$
^misc\.md$
^stuff\.md$
^random\.md$
```

**Jugement de l'agent :** pour chaque candidat, lire le fichier :
- De quoi parle-t-il réellement ? Le raisonnement propose un slug descriptif fondé sur le contenu réel.
- Est-ce une ébauche qui devrait être supprimée plutôt que renommée ? Recouper avec F2.7.
- Pour les fichiers `notes.md` dans des contextes de vault personnel où l'utilisateur s'approprie le dossier organiquement, juger si le renommage aiderait réellement.

**Faux positifs à ignorer :**
- Fichiers nommés par date (`\d{4}-\d{2}-\d{2}.*\.md`).
- Fichiers de cours/leçons numérotés dans des dossiers d'apprentissage évidents.

**Sévérité :** warn.

**Format du constat :**
```
{path} — ambiguous filename
Reasoning: file content is about {topic}; current name doesn't convey this and the agent can't find it by name
Suggested rename: {descriptive-slug}.md
Action: rename
Citation: anthropic-managed-memory.md → File-naming and structure
```

**Auto-correction :** aucune (le renommage nécessite l'approbation de l'utilisateur).

---

## F5.4 : Présence d'un index mémoire

**Règle du framework :** injecter un index de premier niveau pour que l'agent puisse naviguer par le nom.

**Heuristique de déclenchement :** dossiers avec >5 enfants `.md` directs et sans `index.md`/`README.md`/`CLAUDE.md`/`MEMORY.md`/`_index.md` (insensible à la casse).

**Jugement de l'agent :** pour chaque dossier candidat :
- Lire 3 à 5 fichiers échantillons. Sont-ils manifestement liés (un seul type de contenu) ou mélangés ?
- Si mélangés → signalement fort ; un index les unifie.
- S'ils sont indexés par date (ex. notes quotidiennes), la reconnaissance de motif de date de l'agent gère la navigation ; sévérité plus basse.
- Le raisonnement explique quel type d'index aiderait (une table de fichiers vs une prose de routage vs un CLAUDE.md par rôle).

**Faux positifs à ignorer :**
- Dossiers de date (Daily/, meetings/{type}/) où les noms de fichiers sont des dates.
- `Intelligence/archive/` : les archives sont uniquement des dépotoirs.

**Sévérité :** warn.

**Format du constat :**
```
{folder}/ — {N} files, no index file
Reasoning: {what the folder contains, why an index would help — cite specific files that are hard to find by name alone}
Action: create {index|README|CLAUDE}.md listing the files with one-line descriptions
```

**Auto-correction :** aucune.

---

## F5.5 : Cohérence du nommage par date

**Règle du framework :** horodater par date le contenu sensible au temps ; l'agent le trouve par la date.

**Heuristique de déclenchement :** pour les dossiers de date (`Daily/`, `**/meetings/`, `**/journal/`, `**/log/`), vérifier le préfixe `\d{4}-\d{2}-\d{2}` sur les noms de fichiers enfants directs.

**Jugement de l'agent :** pour chaque fichier non conforme, le lire :
- Est-ce réellement du contenu sensible au temps qui n'a simplement pas été daté ? → signaler, suggérer la date à partir du frontmatter ou des métadonnées de première ligne.
- Est-ce un fichier d'index non temporel qui se trouve dans le dossier de date par accident (ex. un CLAUDE.md de niveau dossier) ? → suggérer de le sortir du dossier de date.
- Le raisonnement décrit le type de contenu du fichier et la bonne destination.

**Faux positifs à ignorer :**
- Fichiers index/README/CLAUDE.md à l'intérieur des dossiers de date (ceux-ci routent la navigation, pas des événements).

**Sévérité :** warn.

**Format du constat :**
```
{path} — file in date-folder {folder}/ without YYYY-MM-DD prefix
Reasoning: {what the file is — dated event vs misplaced index}
Action: {specific: "rename to {YYYY-MM-DD}-{slug}.md (date inferred from {frontmatter|first line})" OR "move to {non-date folder}"}
```

**Auto-correction :** aucune.

---

## F5.6 : Vérification du versionnement

**Règle du framework :** les écritures en mémoire produisent des versions nommées immuables ; les analogues côté vault sont git ou la synchronisation Relay.

**Heuristique de déclenchement :**
- `.git/` existe à la racine du vault → versionné.
- `.obsidian/plugins/system3-relay/` ou `.relay/` → synchronisé par Relay.
- Aucun des deux → candidat.

**Jugement de l'agent :** surtout mécanique, mais l'agent lit pour :
- Vérifier `.gitignore` si `.git/` existe : le vault est-il réellement commité (pas entièrement ignoré) ?
- Le raisonnement explique le risque de piste d'audit propre à ce vault (ex. « vous avez 80 fichiers de contenu substantiel sans chemin de retour arrière ; une seule mauvaise édition en masse et le contenu est perdu »).

**Sévérité :** warn.

**Format du constat :**
```
Vault has no version control
Reasoning: {N} files of substantive content with no audit trail or rollback path; bulk operations are irreversible
Action: `git init` and commit, OR install the Relay sync plugin
Citation: anthropic-managed-memory.md → Versioning and audit trail
```

**Auto-correction :** aucune.

---

## Schéma de constat

Même forme que F1 : chaque constat a un `reasoning`. Voir SKILL.md, étape 2.4.
