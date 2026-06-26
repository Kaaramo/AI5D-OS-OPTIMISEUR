# Notes de terrain des praticiens

Synthèse inter-créateurs. Ces notes ne remplacent pas les frameworks officiels : ce sont des observations complémentaires de créateurs qui font tourner le même setup à grande échelle. De la friction de production, pas de la théorie.

## Sommaire

1. [camelCase : "Stop Writing Bad CLAUDE.md Files"](#camelcase--stop-writing-bad-claudemd-files-2026-02-04)
2. [Mark Kashef : "Claude Code Turned Obsidian Into My Dream Second Brain"](#mark-kashef--claude-code-turned-obsidian-into-my-dream-second-brain-2026-03-15)
3. [Dave Ebbelaar : "Effective Context Engineering for AI Agents"](#dave-ebbelaar--effective-context-engineering-for-ai-agents-2025-12-19)
4. [Nate B Jones : "Karpathy's Wiki vs. Open Brain"](#nate-b-jones--karpathys-wiki-vs-open-brain-2026-04-22)
5. [Anthropic : "Claude Agent Skills Explained"](#anthropic--claude-agent-skills-explained-2025-11-26)
6. [Developers Digest : "Progressive Disclosure in Claude Code"](#developers-digest--progressive-disclosure-in-claude-code-2026-01-12)
7. [Patterns inter-créateurs](#patterns-inter-créateurs)
8. [Signaux auditables dérivés des notes de praticiens](#signaux-auditables-dérivés-des-notes-de-praticiens)

---

## camelCase : "Stop Writing Bad CLAUDE.md Files" *(2026-02-04)*

Source: https://www.youtube.com/watch?v=lJNjDoJi6hQ

### Constat phare

> *"One line of bad code only affects exactly that code location. One line of bad CLAUDE.md has negative effects on every prompt and every action that Claude executes for you and for other developers — for years."*

### Enseignements concrets

- **N'exécutez pas `/init`.** Il génère un CLAUDE.md verbeux par défaut. La documentation générée par IA a tendance à être longue, prudente et vague : l'inverse de ce qui fonctionne. Utilisez-la uniquement comme échafaudage, puis élaguez sans pitié.
- **N'incluez pas de règles de style de code** (`use 2 spaces`, `single quotes`, `JS uses camelCase`). Le style de code est un problème déterministe : utilisez des linters et des formateurs avec le hook `post tool use` dans `.claude/settings.json`. Dépenser des tokens de CLAUDE.md pour le style est doublement mauvais : Claude déduirait de toute façon l'essentiel du style à partir du code existant, et Claude n'appliquera pas le style de manière déterministe (votre pipeline CI s'interrompra quand même).
- **Trois blocs de construction de base dont tout CLAUDE.md a besoin :**
  1. **Une phrase** décrivant le projet (framework, langage, audience, domaine)
  2. **Les commandes bash clés** utilisées dans le travail quotidien (build, test, typecheck, lint, deploy)
  3. **Mises en garde** : avertissements non évidents propres au projet ("Never modify schema.prisma directly, run `npm run db:generate`")
- **Utilisez `.claude/rules/`** avec le frontmatter `paths:` pour des règles à portée de chemin. Elles ne se chargent que lorsque Claude lit des fichiers correspondants.
- **Planifiez un audit mensuel de CLAUDE.md.** C'est un document vivant, le code dérive, les instructions deviennent obsolètes.
- **L'intégration GitHub d'Anthropic** vous permet de mentionner Claude dans un commentaire de revue de PR et de lui demander de mettre à jour CLAUDE.md à partir de ce qui a émergé. Réduit la charge.
- **Effet de position :** les LLM portent une attention excessive au début et à la fin, négligent le milieu. Placez les règles les plus importantes en haut.
- **Le system reminder d'Anthropic lui-même est révélateur :** *"This context may or may not be relevant to your tasks."* On dit au modèle que le contenu de CLAUDE.md pourrait être hors de propos. La spécificité surmonte ce phénomène ; le flou non.

### Ce qu'il NE faut PAS faire

- N'exécutez pas `/init` pour ensuite livrer le résultat.
- Ne mettez pas de règles de style de code dans CLAUDE.md.
- N'écrivez pas de fichiers CLAUDE.md de plus de 300 lignes (le plafond pratique pour l'adhérence).
- Ne listez pas chaque script npm : uniquement ceux utilisés dans le travail quotidien.
- N'incluez pas de documentation de schéma détaillée en ligne ; référencez-la par divulgation progressive (`Read docs/schema.md when modifying models`).

### Citations à conserver

> *"Less is more. Under 300 lines, but ideally even shorter."*

> *"The most powerful large language models like Claude Opus 4.5 can follow about 150 to 200 instructions with reasonable consistency."*

> *"What happens when Claude gets too many instructions? Unfortunately, it's not the case that Claude just ignores the 151st instruction but takes the others perfectly. The overall quality of responses degrades massively."*

> *"One line of bad CLAUDE.md has negative effects on every prompt and every action that Claude executes — for years."*

---

## Mark Kashef : "Claude Code Turned Obsidian Into My Dream Second Brain" *(2026-03-15)*

Source: https://www.youtube.com/watch?v=2kbINqpluM0

### Constat phare

Mark a abandonné Obsidian cinq fois avant de l'associer à Claude Code. Une fois que Claude Code est devenu la couche d'écriture, le problème d'abandon a disparu : car la friction d'organiser les notes manuellement était le véritable blocage.

> *"This changed on the sixth time when I combined Obsidian with Cloud Code."*

### Enseignements concrets

- **Pattern du dossier Inbox** : les idées orphelines atterrissent dans `Inbox/`, sont autocatégorisées plus tard par Claude. Réduit la friction de capture.
- **Les slash commands méritent leur place :** `/daily` pour un brief quotidien, `/standup` pour un point de projet, `/tldr` pour les captures de fin de conversation. Le pattern tldr est particulièrement bon : tapez-le à la fin de n'importe quelle conversation et une note structurée atterrit dans Obsidian.
- **Pipeline d'ingestion PDF/binaire** : ne stockez pas de PDF bruts dans le vault que Claude lit activement :
  1. Fichiers bruts → organisation en dossiers (regroupement par type de fichier)
  2. LLM bon marché à grande fenêtre de contexte (Gemini 3 Flash) → fiches mémo markdown
  3. Fiches mémo → vault
- **L'Obsidian CLI** compte 95 commandes. Claude Code peut toutes les piloter. Les skills peuvent encapsuler les plus utilisées.
- **Les prompts à choix multiples** via `AskUserInput` réduisent la friction de configuration plus que le texte libre. Particulièrement pertinent pour les skills d'onboarding (configuration de vault, configuration de profil).
- **Le skill de configuration du vault pose 4 questions :** que faites-vous comme travail ? qu'est-ce qui passe entre les mailles ? travail uniquement ou personnel aussi ? des fichiers existants à importer ?
- **Fonctionnalité Canvas** : Obsidian dispose d'un canvas natif (diagrammes mermaid, style NotebookLM). Les skills peuvent remplir des canvas à partir de prompts.

### Pattern d'architecture observé

```
Obsidian Vault/
├── Inbox/                  # orphan capture, autocategorized later
├── Work/                   # business categories
├── Personal/               # life categories
├── Projects/{name}/
└── .obsidian/             # plugin config (Relay, etc.)
```

Claude Code pointe sur la racine du vault. Les skills encapsulent l'Obsidian CLI pour un accès programmatique.

### Citations

> *"If you ever have a brand new idea that doesn't necessarily have a designated home, it can land in the inbox until you, or in this case, Cloud Code can autocategorize and place it where it fits best."*

> *"You can literally tell your CLAUDE.md, hey, always refer to insert path of all of these markdown files when I ask you about XYZ."*

---

## Dave Ebbelaar : "Effective Context Engineering for AI Agents" *(2025-12-19)*

Source: https://www.youtube.com/watch?v=nkJXADeI62c

### Constat phare

> *"In most of those cases, those AI agents fail not really because they can't reason or because the model isn't smart enough, but because they're reasoning over bad context."*

Le principal mode d'échec en production pour les agents n'est pas le modèle : c'est un mauvais context engineering. Et un mauvais context engineering est invisible pendant le développement parce que les sessions de dev sont courtes. Il n'apparaît qu'au tour 10 ou au tour 20.

### Enseignements concrets

- **Les LLM sont mauvais avec les exemples négatifs.** Remplacez `don't do X` par `do Y`. Montrez à quoi ressemble le bon résultat avec des exemples positifs. *"A picture says more than a thousand words. A positive example is so much better than a negative example saying don't do this."*
- **Mode d'échec de l'évolution du system prompt :**
  1. Départ : system prompt vague
  2. Mise en production
  3. Les utilisateurs se plaignent
  4. L'ingénieur code en dur un if/else pour chaque plainte
  5. Le system prompt gonfle, les performances chutent
  6. On répète
- **Solution : scinder en router + sous-prompts.** Un appel LLM de routage décide quel sous-prompt s'applique. Chaque sous-prompt est petit et ciblé.
- **Les machines à états battent un unique prompt géant.** Suivez l'état de l'utilisateur dans une base de données. À chaque message utilisateur, tirez le system prompt qui correspond à l'état. N'essayez pas d'encoder tous les états dans un seul prompt avec des branches "first do A, then if X do B".
- **Utilisez des outils de traçage (Langfuse) pour visualiser les traces complètes de conversation.** La plupart des problèmes "l'IA est bête" sont en réalité "le contexte est cassé" : visible uniquement quand vous voyez ensemble le system prompt + les appels d'outils + l'historique.
- **Les outils doivent être courts, descriptifs, ciblés, sans chevauchement.** Une description d'outil gonflée est une taxe de contexte à chaque appel. N'ayez pas 20 outils là où 5 suffiraient.
- **Mémoire de conversation :** élaguez ou résumez dès que l'historique devient long. Les problèmes émergent généralement au tour 10–20, jamais au tour 1–2 (la fenêtre de test de développement).
- **Workflows vs agents :** la plupart des cas d'usage sont des workflows (séquences déterministes avec des appels LLM), pas des agents (boucles autonomes d'usage d'outils). Ne sortez pas l'"agent" quand un "workflow" suffit.
- **Savoir quand utiliser un workflow plutôt qu'un agent est la compétence la plus sous-estimée en ingénierie IA.**

### Citations

> *"Calibrating the system prompt is all about being just specific enough but also letting the LLM be creative rather than giving it line by line if-else statements."*

> *"LLMs are not really good at handling negative examples. They really thrive on giving them positive examples."*

> *"In most of those cases, those AI agents fail not because they can't reason but because they're reasoning over bad context."*

> *"The key to building effective AI systems is that it doesn't need to pass once. It doesn't need to pass twice, but it also needs to pass on turn 10 and also on turn 20."*

---

## Nate B Jones : "Karpathy's Wiki vs. Open Brain" *(2026-04-22)*

Source: https://www.youtube.com/watch?v=dxq7WtWxi44

### Constat phare

> *"The Wiki idea is great when you want to think in connections. It is fragile when you need exact retrieval from structured data."*

L'approche wiki est un outil, pas le seul outil. Associez-la à la bonne couche de données structurées pour les tâches où le lookup exact compte.

### Enseignements concrets

- **Le wiki sert aux connexions et à la synthèse.** Références croisées, raisonnement accumulé, savoir narratif.
- **Le wiki est fragile pour le lookup de données structurées.** Requêtes de catalogue, fiches clients, transactions financières, tout ce qui nécessite agrégation/filtrage.
- **Les stacks hybrides sont le pattern de production :**
  - Wiki markdown pour le narratif
  - Postgres pour les entités et la recherche plein texte
  - pgvector pour la recherche de similarité (quand nécessaire)
  - Base de données graphe (Neo4j, Graphiti) pour le raisonnement relationnel (optionnel)
  - Le tout unifié via un serveur MCP
- **Ne mettez pas de catalogues de produits ou de journaux de transactions en markdown.** Mauvaise forme : le markdown stocke des chaînes, pas des entités.
- **Vous, en tant qu'ingénieur/PM, devez quand même décider.** Aucun outil ne se substitue à la réflexion architecturale.

### Citations

> *"It is up to you. It is not up to me. We all have to wrestle with this. And if you are an engineer thinking about this or a product manager thinking about this in an org, you cannot substitute for that level of thoughtfulness. I'm sorry, you got to do the thinking."*

---

## Anthropic : "Claude Agent Skills Explained" *(2025-11-26)*

Source: https://www.youtube.com/watch?v=fOxC44g8vig

### Constat phare

La présentation officielle du modèle de divulgation progressive L1/L2/L3. Le fait opérationnel clé :

> *"At startup only the name and description of every installed skill is loaded in the system prompt. This is going to consume about 30 to 50 tokens per skill."*

### Enseignements concrets

- **Les métadonnées L1 (~30–50 tokens par skill)** se chargent au démarrage. Uniquement les `name` et `description` du frontmatter.
- **Le corps L2 de SKILL.md** se charge quand Claude juge le skill pertinent pour le prompt.
- **Les références L3** ne se chargent que lorsque SKILL.md y renvoie et que Claude les lit.
- **Les skills sont portables** entre Claude Code, l'API et claude.ai.
- **Les serveurs MCP connectent les données ; les sous-agents se spécialisent dans des rôles ; les skills apportent l'expertise.** Trois couches différentes, souvent confondues.

### Cas d'usage mis en avant

- Onboarding de nouvelles recrues aux standards de code
- Garantir que chaque PR respecte les bonnes pratiques de sécurité
- Partager une méthodologie d'analyse de données au sein d'une équipe

### Citation

> *"Skills let you package workflows into reusable capabilities."*

---

## Developers Digest : "Progressive Disclosure in Claude Code" *(2026-01-12)*

Source: https://www.youtube.com/watch?v=DQHFow2NoQc

### Constat phare

> *"Tools as files, loaded on demand, bash + filesystem, progressive disclosure. Bash is all you need."*

L'industrie converge. Cloudflare, Anthropic, Vercel, Cursor : indépendamment, tous sont arrivés à la même architecture pour construire des agents.

### Enseignements concrets

- **La convergence de l'industrie est réelle.** Plusieurs grandes entreprises d'infrastructure IA sont arrivées indépendamment à :
  - Des outils représentés comme des fichiers
  - Chargés à la demande
  - Bash + système de fichiers comme interface universelle
  - La divulgation progressive comme modèle de chargement
- **C'est une réponse contre-intuitive.** Il y a 6 mois, la sagesse conventionnelle disait "utilisez des embeddings vectoriels pour tout" et "les agents ont besoin de frameworks d'orchestration." Cette hypothèse s'est effondrée.
- **Le pattern fonctionne parce que les systèmes de fichiers sont débogables.** Un humain peut faire `cat` sur le même fichier que l'agent lit. Vous ne pouvez pas faire cela avec des embeddings ou un état d'orchestration caché.

### Citation

> *"Tools as files, loaded on demand, bills, progressive disclosure, bash is all you need."*

---

## Patterns inter-créateurs

Quand on empile les notes de praticiens, plusieurs patterns reviennent :

1. **La spécificité bat le flou partout.** La recherche d'Anthropic, les conseils CLAUDE.md de camelCase, le prompt engineering de Dave, la configuration de vault de Mark : tous convergent vers *concret > général*.

2. **La divulgation progressive est désormais standard.** Anthropic la livre pour les skills. Karpathy y arrive indépendamment pour les wikis. Cloudflare/Vercel/Cursor convergent vers la même. Le vault l'obtient via un CLAUDE.md par dossier.

3. **Le pari architectural, c'est fichiers-pas-vecteurs.** La Managed Memory d'Anthropic, le Wiki de Karpathy, l'Obsidian de Mark, la note de convergence de Developers Digest : tous tournent sur du fichier.

4. **Stacks hybrides pour problèmes hybrides.** La mise en garde de Nate B Jones : n'essayez pas de faire faire au markdown ce qu'une base de données fait. Associez le wiki à des stores structurés quand nécessaire.

5. **Les documents vivants exigent une maintenance planifiée.** L'audit mensuel de camelCase, l'opération lint de Karpathy, le test d'élagage d'Anthropic, la conception de ce skill : tous supposent que le système pourrit sans soin actif.

6. **Les modes d'échec en production sont le contexte, pas le modèle.** La thèse principale de Dave Ebbelaar rejoint le cadrage "context engineering" d'Anthropic. Le modèle est rarement le goulot d'étranglement. Le contexte fourni au modèle l'est généralement.

7. **Le premier mois est une corvée.** C'est un thème récurrent dans les commentaires des créateurs sur l'adoption. En solo pendant 2–3 semaines, puis ajoutez un coéquipier curieux, puis étendez. Ne déployez pas de manière descendante.

## Signaux auditables dérivés des notes de praticiens

Ils complètent les signaux dérivés des frameworks dans les autres fichiers de référence :

- **Duplication skill-vault** : un skill embarque ses propres ICP/voix/offre alors que le vault les a dans `Context/` (Pass 8). C'est ce que démontre le skill linkedin-writer-vault.
- **Règles de style de code dans CLAUDE.md** : détecter `\b(use|prefer)\s+\d+\s+(space|tab)`, les règles d'indentation, les règles de guillemets. Suggérer de déplacer vers la config du linter + `.claude/rules/`.
- **Empreinte d'un CLAUDE.md généré par `/init`** : détecter les longs paragraphes de préambule ("This project is...", "The codebase contains..."), les descriptions fichier par fichier, les platitudes génériques. Suggérer un élagage ou une réécriture à partir de zéro.
- **Exemples négatifs dans les skills/CLAUDE.md** : détecter les règles `\bdon't\b.*` sans exemple positif correspondant. Suggérer une conversion vers la forme "do this".
- **Présence d'un dossier Inbox** : dans les vaults se présentant comme un second cerveau, signaler s'il n'y a pas de dossier de capture d'orphelins (`Inbox/`, `Capture/`, `New/`). Suggérer d'en créer un.
- **Slash commands présentes** : un vault sain a souvent des commandes de type `/daily`, `/tldr`, `/standup`. Simple signalement : à signaler si absentes dans un vault actif depuis plus de 2 semaines.
- **Binaires bruts (PDF, slides) à la racine du vault ou dans les dossiers actifs** : le pattern de Mark dit de synthétiser d'abord. Signaler les fichiers binaires dans les dossiers actifs de Claude.
- **Fichiers de journal de conversation dépassant 2k tokens** : le mode d'échec "tour 10–20" de Dave s'applique ici. Signaler les longues captures de conversation non traitées.
