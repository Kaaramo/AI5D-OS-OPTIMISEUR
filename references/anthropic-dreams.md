# F8 : Reflection (the why)

**Source :** Anthropic Managed Agents : Dreams (Research Preview).
https://platform.claude.com/docs/en/managed-agents/dreams

## Thèse centrale

Les mémoires accumulent doublons, contradictions et entrées périmées à mesure qu'elles grossissent. Le lint par fichier ne peut pas voir cela : chaque vérification est locale à un seul fichier. Reflection lit la **couche curatée** du vault parallèlement à la **couche de session récente** (journaux quotidiens, notes de réunion, décisions) et fait remonter des constats inter-fichiers qui n'émergent que de la synthèse.

Un dream prend :
- une mémoire préexistante (ici : chaque dossier où `layer == "curated"` dans le registre de l'Étape 1.5, qu'il s'agisse de rôles standards comme context/projects/resources ou de tout rôle personnalisé que l'utilisateur possède, comme `Building/` ou `Garden/`, que l'agent a classé comme curaté),
- optionnellement jusqu'à 100 sessions (ici : chaque dossier où `layer == "session"`, rôles standards comme daily/meetings et rôles de session personnalisés comme `Inbox/`, limités à la fenêtre temporelle configurée, par défaut 30 jours),

et produit une sortie curatée : doublons fusionnés, entrées périmées remplacées par la dernière valeur, nouveaux constats intégrés. L'entrée n'est jamais modifiée : les propositions sont revues avant d'être appliquées.

**F8 ne code jamais en dur les noms de dossiers, et n'ignore jamais les dossiers qui ne correspondent pas à un rôle standard.** Quelle que soit la façon dont l'utilisateur nomme les choses dans *son* vault (`Knowledge/`, `Library/`, `Journal/`, `Logs/`, `Building/`, `Garden/`, n'importe quoi de personnalisé), le registre des rôles attribue à chacun une couche, et F8 interroge par couche. Les dossiers curatés personnalisés sont exploités pour en extraire contradictions, fusions, entrées périmées et thèmes, exactement comme les dossiers standards. Les rôles standards manquants sont silencieusement ignorés (F9.0 traite le constat de lacune) ; F8 s'exécute quand même sur tout le contenu curaté et de session que le vault possède réellement.

## Pourquoi c'est un framework corrigeable, pas un simple signalement

L'utilisateur lance `/os-optimizer` pour *optimiser* le vault. Chaque constat F8 est donc livré avec une proposition de correction concrète (cible de fusion, texte de remplacement, nouveau chemin de fichier, destination de promotion). L'utilisateur approuve item par item via `AskUserQuestion` ; rien n'est appliqué en masse, car chaque contradiction/fusion/promotion nécessite un jugement humain sur quelle note l'emporte et comment formuler le résultat.

## Ce que F8 attrape et que F1–G7 ne peuvent pas attraper

| Limite de F1–G7 | F8 comble la lacune |
|---|---|
| Règles par fichier | Motifs à l'échelle du vault (clusters, contradictions, thèmes) |
| Portée statique | Sensible au temps : utilise l'activité récente pour détecter ce qui est supplanté |
| Signaux de lint | Signaux de synthèse : ce que le vault dans son ensemble n'a pas |
| Déclencheurs + jugement sur le contexte local | Clustering thématique sur de nombreux fichiers + jugement sur le cluster |

## Cinq catégories F8

1. **Contradictions** : deux notes sont en désaccord sur le même fait.
2. **Candidats à la fusion** : N notes couvrant le même concept qui devraient se réduire à une seule entrée canonique.
3. **Entrées périmées** : hypothèses de `Context/` supplantées par des décisions ou des conclusions de réunion récentes.
4. **Thèmes émergents** : ≥3 notes sur les 30 derniers jours sur un sujet sans entrée Context canonique ni MOC.
5. **Promotions** : connaissances durables enfouies dans des journaux quotidiens/de réunion éphémères, qui ont leur place dans `Context/` ou `Resources/`.

## Principes de fonctionnement

- **L'entrée n'est jamais modifiée par l'analyse** : tous les changements proposés passent par l'Étape 4 (`AskUserQuestion`) avant qu'aucune édition ne soit appliquée.
- **Revue pas à pas uniquement, item par item** : chaque correction F8 nécessite que l'utilisateur choisisse la bonne cible. Pas de mode d'application en masse.
- **La fenêtre temporelle compte** : les sessions plus anciennes que la fenêtre configurée (par défaut 30 jours) ne sont pas considérées comme « récentes ». Le contenu plus ancien vit dans la couche curatée ou est archivé.
- **Respecter le budget de F5** : une fusion qui ferait dépasser le budget par fichier de F5 est rétrogradée en simple signalement avec justification.
- **Ne jamais toucher à un CLAUDE.md** : les constats F8 qui ciblent un CLAUDE.md ou claude.md sont rétrogradés en simple signalement. L'interdiction de réécriture automatique de F1 l'emporte.
