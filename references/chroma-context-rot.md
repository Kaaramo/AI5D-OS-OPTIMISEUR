# Chroma : recherche sur le Context Rot

## Sommaire

1. [Thèse centrale](#thèse-centrale)
2. [Les 18 modèles testés](#les-18-modèles-testés)
3. [Méthodologie : six expériences](#méthodologie--six-expériences)
4. [Constats solides](#constats-solides)
5. [Le paradoxe mélangé vs structuré](#le-paradoxe-mélangé-vs-structuré)
6. [Effets des distracteurs](#effets-des-distracteurs)
7. [Effet de position](#effet-de-position)
8. [Différences au niveau des familles](#différences-au-niveau-des-familles)
9. [LongMemEval : contexte ciblé vs complet](#longmemeval--contexte-ciblé-vs-complet)
10. [Le concept de budget d'attention](#le-concept-de-budget-dattention)
11. [Implications pour l'architecture du vault](#implications-pour-larchitecture-du-vault)
12. [À faire](#à-faire)
13. [À ne pas faire](#à-ne-pas-faire)
14. [Citations textuelles](#citations-textuelles)
15. [Signaux auditables](#signaux-auditables)
16. [Sources](#sources)

---

## Thèse centrale

Les benchmarks NIAH standard (Needle In A Haystack) donnent l'impression que les LLM modernes sont fiables sur le contexte long. Ils ne le sont pas.

Sur **18 modèles de pointe** couvrant Anthropic, OpenAI, Google et Alibaba, l'étude Chroma a constaté :

> *"Model performance consistently degrades with increasing input length."*

Pire encore : **un contexte structuré (logiquement cohérent) peut donner de moins bons résultats qu'un contexte mélangé.** Ce qui compte, ce n'est pas de savoir si l'information est présente dans le contexte : c'est la manière dont elle est présentée et la quantité d'éléments qui l'entourent.

Ce seul constat réécrit la marche à suivre pour la conception d'un vault. "Il suffit de le mettre dans le contexte" est faux. La curation, la position et le rapport signal/bruit comptent davantage que le volume total d'information.

## Les 18 modèles testés

| Famille | Modèles |
|---|---|
| **Anthropic (5)** | Claude Opus 4, Claude Sonnet 4, Claude Sonnet 3.7, Claude Sonnet 3.5, Claude Haiku 3.5 |
| **OpenAI (7)** | o3, GPT-4.1, GPT-4.1 mini, GPT-4.1 nano, GPT-4o, GPT-4 Turbo, GPT-3.5 Turbo |
| **Google (3)** | Gemini 2.5 Pro, Gemini 2.5 Flash, Gemini 2.0 Flash |
| **Alibaba (3)** | Qwen3-235B-A22B, Qwen3-32B, Qwen3-8B |

Le schéma s'est vérifié dans **chaque famille de modèles**. Il n'y a pas eu de "vainqueur" échappant à la dégradation : seulement des différences dans la pente de dégradation.

## Méthodologie : six expériences

L'étude a mené six expériences contrôlées pour isoler les facteurs qui pilotent réellement la dégradation :

| Expérience | Ce qu'elle faisait varier | Ce qu'elle testait |
|---|---|---|
| **1. Similarité aiguille-question** | Similarité cosinus 0.445–0.829 sur 5 modèles d'embedding | La similarité sémantique entre l'aiguille et la question affecte-t-elle la récupération ? |
| **2. Impact des distracteurs** | Référence (aiguille seule) vs 1 distracteur vs 4 distracteurs | Ajouter du contenu similaire mais erroné nuit-il ? |
| **3. Similarité aiguille-meule de foin** | Alignement thématique entre le remplissage et la cible | La relation thématique du remplissage compte-t-elle ? |
| **4. Structure de la meule de foin** | Ordonnancement cohérent vs ordonnancement aléatoire des phrases | La structure logique aide-t-elle ou nuit-elle ? |
| **5. LongMemEval** | QA conversationnelle à ~113k tokens vs variantes ciblées de ~300 tokens | Quel est le coût réel d'un contexte long en conditions réelles ? |
| **6. Mots répétés** | Réplication de texte, séquences de 25 à 10 000 mots avec mots uniques intégrés | Effet de position : où les mots uniques survivent-ils ? |

Ce protocole isole les variables. Les benchmarks de type NIAH les confondent. C'est pourquoi les modèles paraissent meilleurs sur NIAH que ce qu'ils donnent en pratique.

## Constats solides

| Constat | Implication |
|---|---|
| **La longueur dégrade tous les modèles** dans chaque expérience | Le contexte est un budget fini : dépensez-le sur le signal |
| **Mélangé > structuré** sur les 18 modèles | Ne supposez pas que la structure logique aide la récupération |
| **Un seul distracteur nuit de façon mesurable** à la référence ; **quatre distracteurs s'additionnent** | Le remplissage n'est pas gratuit : chaque bloc non pertinent a un coût |
| **Les distracteurs 2 et 3** (positions spécifiques) apparaissent le plus souvent dans les réponses hallucinées | Certains distracteurs sont pires que d'autres |
| **Famille Claude** : taux d'hallucination les plus bas dans l'ensemble | Adaptez la famille de modèle au risque de la tâche |
| **Famille GPT** : taux d'hallucination les plus élevés, confiants mais incorrects | Ne déployez pas GPT dans des pipelines critiques sur les faits sans garde-fous |
| **Effet de position** : les mots uniques placés tôt ont une meilleure précision que ceux enfouis | Commencez par l'information critique, pas par le préambule |
| **LongMemEval** : les prompts ciblés de 300 tokens ont nettement surpassé les contextes complets de 113k | La récupération juste-à-temps bat les déversements en amont |
| **Claude Opus 4** a montré la divergence ciblé-vs-complet la plus prononcée | Même le modèle le plus puissant perd des capacités significatives quand le contexte enfle |
| **Faible similarité aiguille-question** → courbes de dégradation nettement plus raides | L'alignement sémantique compte davantage à grande longueur |

## Le paradoxe mélangé vs structuré

Le constat le plus contre-intuitif :

> *"Across all 18 models and needle-haystack configurations, we observe a consistent pattern that models perform better on shuffled haystacks."*

Pourquoi la structure logique nuit-elle ? Parce que des documents adjacents dans un ensemble structuré logiquement partagent une terminologie et des schémas : ils ressemblent à des **distracteurs plausibles**. Le modèle ne peut pas distinguer facilement la cible de ses voisins. Le contexte mélangé perturbe cette similarité, ce qui fait ressortir la cible.

**Implication pour les vaults :** l'intuition de la "structure de dossiers bien organisée" n'est pas gratuite. Si deux fichiers adjacents utilisent un vocabulaire similaire (par exemple, `voice.md` et `brand.md` traitent tous deux du ton), ils se distraient mutuellement lorsqu'ils se chargent ensemble. Au choix :
- Les consolider
- Différencier le vocabulaire
- N'en charger qu'un à la fois (CLAUDE.md par dossier / divulgation progressive)

## Effets des distracteurs

L'étude a testé la référence (aiguille seule), 1 distracteur et 4 distracteurs :

- **1 distracteur réduit la performance** sous la référence, de façon mesurable et constante.
- **4 distracteurs additionnent** la dégradation, mais pas de façon linéaire. Chaque distracteur supplémentaire nuit moins que le premier.
- **Les distracteurs aux positions 2 et 3** (tôt mais pas en premier dans la meule de foin) apparaissent le plus dans les réponses hallucinées. Peut-être parce que le modèle prête une forte attention au contenu précoce, mais que le tout premier item bénéficie d'un traitement particulier.

**Implication pour les vaults :** chaque fichier non pertinent que vous chargez est un distracteur. Le coût n'est pas linéaire en nombre de fichiers : le *premier* fichier non pertinent est le pire. C'est pourquoi la divulgation progressive (CLAUDE.md de dossier chargé à la demande) compte davantage qu'un simple "CLAUDE.md racine plus petit".

## Effet de position

L'expérience des mots répétés a montré que les mots uniques placés **tôt** dans une longue séquence avaient une meilleure précision que ceux placés plus profondément.

**Implication pour tout fichier markdown :**
- Les premiers ~10–20 % reçoivent le plus d'attention.
- Les derniers ~10–20 % reçoivent une attention notable.
- Le milieu reçoit le moins d'attention.

Cela correspond à la recommandation d'Anthropic pour CLAUDE.md de placer les règles critiques en haut.

Pour les docs de vault :
- **Commencez par la décision**, pas par le raisonnement.
- **Commencez par la règle**, puis le raisonnement.
- **Commencez par l'action**, puis le contexte.
- Utilisez des encadrés en haut pour les résumés `> [!important]`.

## Différences au niveau des familles

Tous les modèles ne se dégradent pas de la même façon :

- **Famille Claude** : dégradation progressive, hallucination la plus basse. En cas d'incertitude, plus enclin à s'abstenir ou à demander. Le meilleur choix pour la récupération de faits à haut risque.
- **Famille GPT** : hallucination confiante à grande longueur. Génère des réponses plausibles mais incorrectes. Nécessite une couche de validation en production.
- **Famille Gemini** : médiocre sur les deux axes ; une grande fenêtre de contexte ne se traduit pas par une qualité sur contexte long.
- **Famille Qwen** : performante à des longueurs plus courtes ; se dégrade plus vite que Claude/GPT aux longueurs extrêmes.

À retenir : grande fenêtre de contexte ≠ capacité sur contexte long. Choisissez les modèles d'après la performance observée sur contexte long, pas d'après la taille de fenêtre annoncée.

## LongMemEval : contexte ciblé vs complet

LongMemEval a testé la QA conversationnelle en deux formats :

1. **Ciblé** : prompt de ~300 tokens avec uniquement le tour antérieur pertinent
2. **Complet** : historique complet de conversation de ~113k tokens

Résultat : **le ciblé a surpassé de façon spectaculaire le complet** sur tous les modèles testés. Claude Opus 4 a eu la plus grande divergence : autrement dit, même le modèle le plus puissant perd des capacités significatives lorsqu'on le force à trouver une aiguille dans 113k tokens d'historique.

**Implication pour les vaults :** la tentation de "simplement charger tout l'historique de conversation" ou "inclure la note quotidienne complète de la semaine dernière" est mauvaise. Curez vers les 300 tokens pertinents. Le schéma compaction + récupération juste-à-temps recommandé par Anthropic correspond directement à ce constat.

## Le concept de budget d'attention

L'attention des transformeurs crée **n² relations par paire pour n tokens** : autrement dit, l'attention s'amincit par paire à mesure que n croît. C'est la raison structurelle du context rot.

> L'attention est un budget fini. À mesure que n croît, la "part" d'attention de chaque token rétrécit. Au-delà d'un certain n, les règles structurantes de votre CLAUDE.md ne reçoivent pas assez d'attention pour influencer le comportement.

C'est aussi pourquoi même les modèles dotés de fenêtres de contexte de 200k+ montrent une dégradation bien avant que la fenêtre soit pleine. Le budget d'attention pourrit plus vite que la fenêtre ne se remplit.

## Implications pour l'architecture du vault

| Constat Chroma | Règle de conception du vault |
|---|---|
| La longueur dégrade tous les modèles | Garder le CLAUDE.md racine sous 200 lignes |
| Mélangé > structuré | Soit consolider les fichiers similaires, soit les charger à la demande, pas ensemble |
| Les distracteurs nuisent ; le premier est le pire | Chaque fichier non pertinent dans le contexte coûte cher |
| La position compte | Règles critiques en haut de chaque fichier |
| 300 tokens ciblés > 113k complets | Utiliser la divulgation progressive (CLAUDE.md par dossier, références à la demande) |
| Famille Claude, hallucination la plus basse | Adapter le modèle au profil de risque |
| L'attention est en n² | Traiter le contexte comme un budget fini |

## À faire

- Curer le contexte vers la plus petite tranche pertinente.
- Placer l'information la plus importante **en premier** dans chaque fichier (pas seulement CLAUDE.md).
- Utiliser la divulgation progressive : CLAUDE.md par dossier, références à la demande.
- Garder le CLAUDE.md racine léger : il se charge à chaque session.
- Utiliser la famille Claude quand le coût d'une hallucination est élevé.
- Traiter le budget d'attention comme fini même quand la fenêtre ne l'est pas.
- Privilégier la récupération juste-à-temps aux déversements de tout le corpus.
- Lorsque vous avez des fichiers thématiquement similaires, soit les consolider, soit les charger à la demande.
- Compacter agressivement à 60–70 % d'utilisation du contexte.

## À ne pas faire

- Ne supposez pas que "dans le contexte = récupérable". Ce n'est pas le cas.
- N'ajoutez pas de remplissage "par souci d'exhaustivité" : chaque bloc non pertinent est un distracteur.
- Ne comptez pas sur la structure logique pour vous sauver (elle peut nuire à la récupération).
- N'enfouissez pas de faits critiques au fond de longs fichiers.
- Ne saturez pas la fenêtre de contexte simplement parce qu'elle est disponible.
- Ne chargez pas l'historique quotidien/conversationnel complet quand vous pouvez curer vers le tour pertinent.
- Ne confondez pas taille de fenêtre et capacité : testez toujours la performance sur contexte long pour votre tâche.
- Ne placez pas deux fichiers thématiquement similaires (par exemple, `voice.md` + `brand.md`) dans le même chemin de chargement sans différenciation.

## Citations textuelles

> *"Whether relevant information is present in a model's context is not all that matters; what matters more is how that information is presented."*

> *"Model performance consistently degrades with increasing input length."*

> *"Across all 18 models and needle-haystack configurations, we observe a consistent pattern that models perform better on shuffled haystacks."*

> *"Even a single distractor reduces performance relative to the baseline."*

> Claude models *"consistently exhibit the lowest hallucination rates"*; GPT models *"show the highest rates of hallucination, often generating confident but incorrect responses."*

## Signaux auditables

Lorsque ce skill exécute la Passe 4 (estimation de tokens) et la Passe 9 (balayage des anti-patterns) pour les signaux de context rot :

- **Distribution de la longueur des fichiers** : calculer la taille médiane des fichiers markdown ; signaler les fichiers ≫ médiane du projet (probablement chargés de distracteurs, candidats au découpage).
- **Position de l'info critique** : dans CLAUDE.md et les fichiers de routage, détecter les règles/décisions qui apparaissent après la ligne N (où N = ~30 % du total des lignes). Suggérer de les remonter.
- **Préambule d'amorce** : détecter les débuts de notes avec mise en contexte/installation avant la décision réelle. Suggérer de commencer par la décision.
- **Densité de distracteurs** : notes comportant de nombreuses entités/sujets similaires dans un même fichier ; ceux-ci se chargent comme des distracteurs collectifs à la lecture du fichier.
- **Taille du contexte chargé** : somme des fichiers qui se chargent au démarrage de la session (CLAUDE.md racine + CLAUDE.md parcourus en remontant + MEMORY.md s'il est présent). Signaler si > budget souple (~3k tokens).
- **Paires de fichiers à sujet similaire** : heuristique de nom de fichier : `voice.md` + `brand.md` + `tone.md` se recouvrent probablement et se distraient mutuellement lorsqu'ils sont tous chargés.
- **Encadré en haut de fichier** : la règle du vault stipule que chaque doc doit avoir un encadré `> [!type]` pour la structure visuelle. Vérifier précisément que les décisions critiques en ont un près du haut.
- **Taille des fichiers quotidiens/conversationnels** : dans `Daily/`, signaler les fichiers dépassant ~2k tokens ; cela suggère un gonflement dû au copier-coller d'historique de conversation plutôt qu'à une curation.

## Sources

- https://www.trychroma.com/research/context-rot (étude de référence)
- https://research.trychroma.com/context-rot (redirige vers l'étude ci-dessus)
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents (cadrage Anthropic du "budget d'attention")
