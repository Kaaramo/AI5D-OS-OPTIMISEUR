# F9 : Architecture & Découvrabilité (le pourquoi)

**Source :** notes de terrain de praticiens exploitant des seconds cerveaux sous Claude collègue. Pas issu d'un framework canonique unique : distillé à partir des points où F1–G7 + F8 laissent encore des lacunes.

## Thèse centrale

Le lint fichier par fichier ne peut pas vous dire si le **vault dans son ensemble oriente un agent fraîchement arrivé**. F1 audite la qualité du contenu d'un CLAUDE.md. F2 audite l'intégrité des wikilinks. F8 audite la synthèse sémantique. Aucun d'eux ne parcourt le chemin qu'un Claude collègue emprunte réellement :

> `CLAUDE.md` racine → entrée de routage → fichier d'index de dossier (quelle que soit la convention de l'utilisateur) → fichier

Si un maillon de cette chaîne est erroné, périmé ou manquant, l'agent ne trouve jamais le fichier, même si le fichier lui-même est parfaitement rédigé. F9 parcourt cette chaîne de bout en bout et fait remonter chaque rupture.

**F9 ne présume jamais des noms de dossiers, et n'ignore jamais les dossiers qui ne correspondent pas à un schéma standard.** La passe de découverte des rôles de l'Étape 1.5 classe chaque dossier. Les rôles standards (identité, contexte, projets, décisions, notes quotidiennes, réunions, transcriptions, ressources, skills, archive) sont des schémas que l'agent reconnaît. Tout ce qui ne correspond pas devient un **rôle personnalisé** : `Building/`, `Garden/`, `Inbox/`, `Sandbox/`, `Lab/`, quel que soit ce que l'utilisateur possède, avec une *couche* inférée (curatée / session / archive / méta / inconnue) et une finalité en 1 ligne. Les deux types participent pleinement à F9 : la vérification de la véracité de la table de routage contrôle la couverture de chaque dossier ; l'application de l'index de dossier s'applique à chaque dossier non trivial ; la revue de découvrabilité inclut chaque dossier, indépendamment de son appartenance à un rôle ; les propositions de réorganisation couvrent l'ensemble du vault.

F9.0 traite les besoins structurels en trois paliers, selon la force avec laquelle l'agent recommande l'action :

1. **Lacunes fonctionnelles** (sévérité : fail) : le travail propre de l'optimiseur est menacé. L'agent ne trouve aucun contexte opérateur/identité ; la racine n'a aucun routage d'aucune forme ; la couche d'un rôle personnalisé est inclassable. Indifférent à la forme : l'identité dans le frontmatter du CLAUDE.md convient, le routage exprimé en prose d'agent convient.
2. **Améliorations fonctionnelles** (sévérité : warn) : l'agent a *jugé que l'adoption d'une convention AI5D améliorerait significativement ce vault précis* compte tenu de l'état actuel, avec un raisonnement propre au cas. (« Vous avez 14 notes de forme professionnelle dispersées dans 6 dossiers sans hub clair ; les centraliser sous un dossier `Projects/` (ou équivalent) rendrait l'état des projets découvrable en un seul saut. ») Si la structure existante de l'utilisateur remplit déjà la fonction, par exemple un dossier personnalisé `Lab/` joue déjà le rôle de projets, aucun constat de palier 2 ne se déclenche.
3. **Inspiration** (sévérité : info, refus par défaut) : la taxonomie standard AI5D présentée comme une référence unique de niveau info. L'utilisateur retient celles qui correspondent à sa façon de travailler ; le reste persiste comme refusé et ne re-sollicite pas.

La position de l'optimiseur sur la taxonomie AI5D : testée, utile, optionnelle. L'agent l'applique comme grille de reconnaissance (rôles standards de l'Étape 1.5) et comme source de recommandations pour les améliorations de palier 2 *lorsqu'une preuve concrète montre qu'elle aiderait ce vault*. Jamais comme une forme cible à laquelle l'utilisateur doit se conformer. Les rôles personnalisés que possède l'utilisateur sont de première classe : jamais ignorés, jamais rétrogradés.

Rien de ce que possède l'utilisateur n'est ignoré simplement parce que cela ne correspond pas à un schéma. Rien n'est prescrit simplement parce que AI5D l'utilise.

## Ce que F9 détecte et que F1–F8 ne peuvent pas

| Préoccupation | Pourquoi F1–F8 le manquent | Comment F9 comble la lacune |
|---|---|---|
| Une entrée de routage pointe vers un dossier qui n'existe plus | F2.6 vérifie seulement que la table est présente et que les dossiers de premier niveau sont mappés | F9.1 vérifie que le chemin de chaque entrée de routage se résout |
| Une description de routage prétend que `Projects/` contient X alors qu'il contient en réalité Y | Aucun framework ne lit le contenu des dossiers pour le comparer à la description | F9.1 lit 3 à 5 fichiers échantillons par dossier et juge de la cohérence |
| Un dossier n'a aucun fichier d'index de dossier | Rien ne vérifie la présence d'un index par dossier | F9.2 applique la convention découverte à chaque dossier non trivial ; F9.0 propose d'en adopter une si aucune convention n'existe encore |
| Un index de dossier existe mais liste des fichiers qui n'existent plus ou omet de nouveaux fichiers | Aucune détection de dérive entre l'index et la réalité | F9.2 compare les enfants de l'index à la sortie de `ls` |
| Un fichier est atteignable par wikilink mais pas par navigation | Les orphelins de F2.3 utilisent les wikilinks, pas la chaîne de routage | F9.3 simule la revue de navigation et signale les orphelins par nombre de sauts |
| Un fichier est dans le mauvais dossier bien qu'il passe F2.6 (techniquement valide, conceptuellement erroné) | F2.6 est structurel ; ceci est sémantique | F9.4 lit le fichier comparé à la finalité du dossier parent et juge |
| Deux dossiers font le même travail sous des noms différents | Aucune comparaison sémantique inter-dossiers | F9.5 compare les finalités déclarées d'un dossier à l'autre |
| Le vault pourrait être réorganisé pour plus de clarté mais aucune règle individuelle n'est enfreinte | Tous les frameworks sont des vérifications atomiques | F9.6 propose 1 à 3 changements structurels à fort impact, avec raisonnement |
| Le `CLAUDE.md` racine paraît correct mais un agent fraîchement arrivé ne parvient pas réellement à s'orienter à partir de lui | F1 est un lint de qualité du contenu, pas un test d'aptitude | F9.7 lit d'abord Context/, construit des questions d'orientation propres au vault, puis tente d'y répondre à froid à partir du CLAUDE.md |

## Principes de fonctionnement

- **Parcourir la chaîne que parcourt Claude collègue.** F9 simule le chemin de découverte réel ; les échecs de *ce* chemin sont des constats, indépendamment de l'intégrité des wikilinks.
- **Chaque constat livre un correctif concret.** Pas de simple signalement. Aucun avertissement ne reste un avertissement. Lorsque l'utilisateur exécute l'optimiseur en mode application, chaque constat F9 devient soit une édition appliquée, soit une étape de migration enregistrée dans le plan de réorganisation daté : jamais reporté indéfiniment.
- **Orientation propre au vault, pas générique.** F9.7 construit ses questions d'orientation à partir des rôles d'identité + de contexte découverts chez l'utilisateur (quels que soient les dossiers ou fichiers qui jouent ces parties dans *ce* vault) : ce vers quoi le vault de *cet* utilisateur devrait orienter un agent. Le vault d'un fondateur oriente différemment de celui d'un chercheur.
- **Deux modes pour chaque correctif :** *appliquer maintenant* (parcourir et confirmer chaque étape dans cette exécution) ou *enregistrer dans le plan* (écrire dans le dossier équivalent-décisions découvert pour une exécution échelonnée ; se rabat sur `audits/` à la racine du vault si aucun rôle de décisions n'existe, avec un constat F9.0 pour en formaliser un). L'utilisateur choisit par item. Les deux modes engagent l'utilisateur à aller au bout ; ni l'un ni l'autre ne laisse le constat traîner en simple signalement.
- **La convention d'index de dossier est découverte.** Fichier markdown léger sous le nom que l'utilisateur emploie déjà (`README.md`, `Plot.md`, `index.md`, etc.) : finalité du dossier en 1 ligne, liste des enfants avec une description en 1 ligne chacun, horodatage de mise à jour. Auto-généré lorsqu'il manque ; auto-régénéré lorsqu'il est périmé ; l'utilisateur confirme la formulation pour chaque dossier.
- **Les rôles manquants sont des constats, pas des suppositions silencieuses.** Pas d'identité ? Pas de contexte ? Pas de dossier de décisions ? F9.0 fait remonter chaque lacune avec une proposition d'adoption concrète. L'utilisateur peut accepter, refuser (enregistré comme une décision explicite), ou demander à l'agent d'attribuer le rôle à un dossier existant qui joue déjà cette partie de façon informelle.
- **Ne jamais éditer un CLAUDE.md sans approbation de l'utilisateur item par item.** Les réécritures de routage de F9.1 et les ajouts d'orientation de F9.7 procèdent tous deux section par section.

## Pourquoi cela ne pourrait pas être une passe regex

Chaque vérification F9 exige que l'agent :
1. **Lise la structure** (dossiers, fichiers, ce que contient chacun).
2. **Lise l'intention déclarée de l'utilisateur** (routage du CLAUDE.md, finalités des Plot.md de dossiers, Context/ pour le contexte personnel).
3. **Raisonne sur la cohérence** entre (1) et (2).
4. **Propose un changement concret** qui comble la lacune.

Une regex peut détecter un chemin manquant ; seul un agent peut décider si la description correspond à la réalité, si deux dossiers dupliquent le travail, ou si la chaîne d'orientation oriente réellement.
