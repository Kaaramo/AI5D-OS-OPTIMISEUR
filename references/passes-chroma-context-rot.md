# F4 : Chroma Context Rot (implémentation de la passe)

**Référence (le pourquoi) :** `references/chroma-context-rot.md`.
**S'applique à :** chaque `.md`. Les vérifications de position (F4.3, F4.4) priorisent les fichiers qui se chargent tôt. Les vérifications de distracteurs (F4.5) opèrent sur les paires en auto-chargement et dans un même dossier.

## Fonctionnement de cette passe

Agentique. La longueur et la position sont surtout mécaniques, mais **ce qui constitue un problème dépend du rôle et du contenu du fichier**. Un transcript de 200 Ko ne pose aucun souci ; un CLAUDE.md de 200 Ko est un désastre. Une amorce en préambule dans une note quotidienne est normale ; dans un index de routage, c'est un problème de règle enfouie. L'agent lit chaque candidat et produit un raisonnement qui distingue les deux cas.

Voir l'introduction de F1 pour le schéma complet déclencheur → jugement → raisonnement.

## Sommaire

1. [F4.1 : Distribution des longueurs de fichiers](#f41--distribution-des-longueurs-de-fichiers)
2. [F4.2 : Budget de taille du contexte chargé](#f42--budget-de-taille-du-contexte-chargé)
3. [F4.3 : Position de l'information critique](#f43--position-de-linformation-critique)
4. [F4.4 : Amorce en préambule](#f44--amorce-en-préambule)
5. [F4.5 : Densité de distracteurs / paires de sujets similaires](#f45--densité-de-distracteurs--paires-de-sujets-similaires)
6. [F4.6 : Callout en tête de fichier](#f46--callout-en-tête-de-fichier)
7. [F4.7 : Taille des notes quotidiennes](#f47--taille-des-notes-quotidiennes)
8. [Schéma de constat](#schéma-de-constat)

---

## F4.1 : Distribution des longueurs de fichiers

**Règle du framework :** les fichiers plus longs se dégradent davantage ; les valeurs aberrantes agissent comme des distracteurs collectifs.

**Heuristique de déclenchement :** calculer la taille médiane en octets du projet. Fichiers > médiane × 5 → candidat.

**Jugement de l'agent :** pour chaque candidat, lire le fichier (ou le survoler s'il est énorme) :
- Est-il intentionnellement long (transcripts, documents de référence, contenu de cours) ? → peut être acceptable ; signaler tout de même s'il est chargé en auto-contexte.
- Est-il surchargé (plusieurs sujets sans rapport dans un seul fichier) ? → signalement fort ; recommander une scission avec des points de découpe de sections précis.
- Le raisonnement décrit la structure du fichier et confirme soit qu'il doit être scindé, soit qu'il est légitimement long.

**Faux positifs à ignorer :**
- Fichiers dans `*transcript*/`, `*archive*/` où la longueur est attendue.

**Sévérité :**
- > médiane × 10 → fail
- > médiane × 5 → warn

**Format du constat :**
```
{path} — {KB}KB ({Nx} project median)
Reasoning: {what's actually in the file — number of H2 sections, range of topics; whether it's one topic at length vs many topics jammed together}
Action: {specific: "split at {section breakpoints}" OR "leave as-is — legitimately long for {reason}"}
```

**Auto-correction :** aucune.

---

## F4.2 : Budget de taille du contexte chargé

**Règle du framework :** le contexte en auto-chargement constitue un budget d'attention fini ; cible souple ~3K tokens.

**Heuristique de déclenchement :** sommer les octets du CLAUDE.md racine (le seul fichier en auto-chargement selon la règle Anthropic 13 : les CLAUDE.md de dossier se chargent à la demande). Tokens ≈ octets/4. Comparer au budget de 3 000 tokens.

**Jugement de l'agent :** si le budget est dépassé, lire le CLAUDE.md racine et identifier :
- Quelles sections sont du pur bruit (pourraient être déplacées vers references/, .claude/rules/, ou des skills) ?
- Quelles sections sont porteuses pour chaque session, par opposition aux sections de niche ?
- Le raisonnement nomme les intitulés de sections précis à déplacer et leur destination.

**Faux positifs à ignorer :**
- Vaults où l'utilisateur a explicitement opté pour un auto-chargement volumineux (rare ; demander s'il existe une préférence affichée).

**Sévérité :**
- > 6 000 tokens → fail
- > 3 000 tokens → warn

**Format du constat :**
```
Auto-load context = {tokens}t (root CLAUDE.md = {bytes}B)
Reasoning: {names of sections that could move and where} — together those would save ~{est}t
Action: extract {section-A} → references/, {section-B} → .claude/rules/{topic}.md with paths: scope
Citation: chroma-context-rot.md → Loaded context size
```

**Auto-correction :** aucune.

---

## F4.3 : Position de l'information critique

**Règle du framework :** les mots uniques placés tôt obtiennent une meilleure précision. Les règles enfouies sont négligées.

**Heuristique de déclenchement :** pour `root-claude`, `folder-claude`, `index`, `readme`, parcourir toutes les lignes et marquer celles qui semblent porteuses (impératifs en début de ligne, lignes de table de routage, callouts, marqueurs `IMPORTANT:`). Pour chaque ligne porteuse au-delà du seuil de 30 % → candidat.

**Jugement de l'agent :** lire la structure du fichier :
- La règle enfouie est-elle réellement critique ? (Certains impératifs comme « Save the report » sont opérationnels mais sans enjeu élevé.) Ne pas sur-signaler.
- Le fichier est-il assez long pour que « 30 % » soit un seuil significatif ? Dans un fichier de 50 lignes, la ligne 20 n'est pas vraiment « enfouie ».
- Le raisonnement nomme la règle enfouie précise et ce qui occupe le haut du fichier à sa place.

**Faux positifs à ignorer :**
- Fichiers où le haut est intentionnellement un frontmatter + un callout de résumé, avec les règles juste en dessous.
- Fichiers < 30 lignes où la position compte moins.

**Sévérité :** warn.

**Format du constat :**
```
{path} — {n} load-bearing rules buried past 30%
Examples:
  L{line} ({pct}% in): "{excerpt}"
Reasoning: {what the top of the file currently contains, and why the buried rule deserves higher placement}
Action: move {specific rules} to the top; demote {currently-top section} below
```

**Auto-correction :** aucune.

---

## F4.4 : Amorce en préambule

**Règle du framework :** commencer par la règle, pas par la justification.

**Heuristique de déclenchement :** lire les 30 premières lignes (hors frontmatter et H1). Détecter les motifs de préambule :
```
\b(This document explains|In this guide|Welcome to|The purpose of this|Below you will find|We will cover|This file describes|Overview of)\b
```
Si présent et qu'aucun impératif/table de routage n'apparaît avant la ligne 20 → candidat.

**Jugement de l'agent :** lire les 30 premières lignes.
- Le préambule pose-t-il un contexte nécessaire (par exemple, nommer la portée du fichier) ? → peut être acceptable avec une sévérité faible.
- Le préambule n'est-il que du remplissage ? → signaler.
- Le raisonnement explique ce que dit actuellement le préambule et ce qui devrait le remplacer.

**Faux positifs à ignorer :**
- Fichiers où le préambule est un résumé de portée d'une ligne suivi immédiatement de contenu porteur.

**Sévérité :** warn.

**Format du constat :**
```
{path} — preamble before any load-bearing content
Excerpt: "{first 100 chars}…"
Reasoning: {what the preamble adds — likely nothing operational; what's actually load-bearing further down}
Action: lead with {specific lower section}; move preamble below or delete
```

**Auto-correction :** aucune.

---

## F4.5 : Densité de distracteurs / paires de sujets similaires

**Règle du framework :** des fichiers similaires dans le même chemin de chargement se distraient mutuellement (constat « shuffled > structured »).

**Heuristique de déclenchement :** pour les paires de fichiers dans le même dossier où la distance de Levenshtein ≤4 entre les noms de base ET le Jaccard de vocabulaire >0,4 (après suppression des mots vides, en excluant les zones protégées).

**Jugement de l'agent :** pour chaque paire candidate, lire brièvement les deux fichiers :
- Se recoupent-ils réellement sur le plan thématique, ou partagent-ils un vocabulaire générique parce que ce sont tous deux des notes ?
- S'ils se chargent toujours ensemble (par exemple, tous deux référencés depuis le même CLAUDE.md), l'effet de distracteur est réel → signaler avec raisonnement.
- S'ils se chargent indépendamment (chemins de routage différents), le risque de distracteur est plus faible → signaler avec une sévérité moindre, ou ignorer.

**Faux positifs à ignorer :**
- Paires où l'un est clairement un index de l'autre.
- Paires explicitement différenciées par la portée du frontmatter.

**Sévérité :** warn.

**Format du constat :**
```
Distractor pair in {folder}/:
  - {path1}
  - {path2}
Reasoning: vocabulary overlap = {jaccard}; both files are referenced from {load path}, so they load together; specific overlapping content: {section names}
Action: consolidate, differentiate vocabulary explicitly, or split load paths (move one to .claude/rules/{topic}.md with paths: scope)
```

**Auto-correction :** aucune.

---

## F4.6 : Callout en tête de fichier

**Règle du framework :** les décisions critiques bénéficient d'un callout `> [!type]` près du haut.

**Heuristique de déclenchement :** pour les fichiers classés comme `decision` OU contenant `decision`/`rule`/`policy` dans le H1 ou le frontmatter, vérifier les 30 premières lignes pour `^>\s*\[!(important|warning|note|info|tip|caution)\]`.

**Jugement de l'agent :** pour chaque candidat sans callout :
- Lire le fichier. Identifier l'affirmation porteuse : existe-t-il une phrase unique qui capture la décision/règle ?
- Le raisonnement fournit cette phrase comme corps de callout suggéré.

**Faux positifs à ignorer :**
- Fichiers qui commencent déjà par un H1 + un résumé d'une phrase (fonctionnellement un callout sans la syntaxe).

**Sévérité :** warn.

**Format du constat :**
```
{path} — no top-of-file callout
Reasoning: this is a {decision/rule/policy} file; the load-bearing claim appears to be "{one-sentence summary the agent extracts}"
Action: add `> [!important]\n> {summary}` at the top
```

**Auto-correction :** aucune.

---

## F4.7 : Taille des notes quotidiennes

**Règle du framework :** des prompts ciblés de 300 tokens battent des contextes complets de 113K ; le gonflement des notes quotidiennes est un anti-pattern connu.

**Heuristique de déclenchement :** chaque fichier `daily`. Tokens ≈ octets/4. Budget souple de 2 000 tokens.

**Jugement de l'agent :** pour chaque candidat dépassant le budget, lire le fichier :
- Qu'est-ce qui le gonfle ? Un historique de conversation collé ? De longs transcripts de réunion ? Des actions à mener qui devraient être ailleurs ?
- Le raisonnement nomme la source du gonflement et recommande où elle devrait se trouver.

**Faux positifs à ignorer :**
- Entrées quotidiennes qui capturent un unique événement substantiel (par exemple, une décision majeure consignée en détail) : signaler avec une sévérité faible.

**Sévérité :**
- > 4 000 tokens → fail
- > 2 000 tokens → warn

**Format du constat :**
```
{path} — daily note {tokens}t (>{budget}t)
Reasoning: {what's bloating it — pasted X, transcript of Y, etc.}
Action: extract {specific content type} to {specific destination} (e.g., decisions to Intelligence/decisions/, meeting transcripts to Intelligence/meetings/)
```

**Auto-correction :** aucune.

---

## Schéma de constat

Même forme que F1 : chaque constat possède un `reasoning`. Voir SKILL.md Étape 2.4.
