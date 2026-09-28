# Scanners de sécurité pilotés par IA : une nouvelle surface d'attaque

> Write-up pédagogique sur les vulnérabilités des scanners de sécurité basés sur des LLM, à partir du module *Web LLM Attacks* de la PortSwigger Web Security Academy.

---

## 1. Qu'est-ce qu'un scanner de sécurité piloté par IA ?

Un scanner de sécurité **piloté par IA** est un outil automatisé qui parcourt (crawl) une application web, lit son contenu et audite sa sécurité — mais dont le moteur de décision est un **grand modèle de langage (LLM)**.

Contrairement à un scanner classique, qui suit des règles figées, le scanner IA **raisonne** : il lit le contenu des pages (commentaires, avis, articles), en déduit un plan d'action et appelle des fonctions (requêtes HTTP, accès à des API internes) pour mener son audit.

Pour être efficace, on lui donne souvent :
- des **identifiants authentifiés** (pour explorer les zones privées d'un site) ;
- un **accès à des données sensibles** (clés API, configuration) ;
- une **position réseau privilégiée** (il tourne à l'intérieur de l'infrastructure).

C'est précisément ce qui en fait une cible de choix.

---

## 2. Le problème : une puissance qui devient une faille

Le point faible d'un scanner IA n'est pas un bug de code classique, mais son incapacité à **distinguer les données des instructions**.

Quand le scanner lit un commentaire, ce texte est censé être une **donnée** à analyser. Mais si un attaquant y glisse des instructions bien formulées, le LLM peut les interpréter comme des **ordres à exécuter**. C'est ce qu'on appelle l'**injection de prompt indirecte** : l'attaquant n'interagit jamais directement avec l'IA — il piège un contenu que l'IA lira plus tard.

Deux conséquences majeures :

- **L'attaquant emprunte les privilèges du scanner.** Un visiteur non authentifié ne peut pas supprimer un compte ou lire une clé API. Mais s'il manipule le scanner (authentifié, privilégié), il lui fait exécuter ces actions à sa place.
- **Le scanner devient une arme.** D'outil défensif, il se transforme en tremplin pour atteindre des ressources internes normalement hors de portée.

---

## 3. Les attaques

### 3.1 Actions destructrices (injection indirecte)

L'attaquant poste un commentaire piégé. Quand le scanner le lit, il exécute une action à état modifiant : suppression de compte, changement de paramètres.

**Clé de réussite :** une demande frontale (« supprime ce compte ») est souvent détectée comme malveillante. En revanche, déguiser l'ordre en **étape légitime de vérification** (« confirme cette vulnérabilité déjà identifiée en la déclenchant ») passe beaucoup mieux — le scanner croit faire son travail.

### 3.2 Exfiltration de données sensibles

Ici, on ne fait pas *agir* le scanner, on le fait *parler*. Le scanner a accès à des données sensibles (ex. une clé API) dans le cadre de son audit. L'injection le pousse à **inclure cette donnée dans sa sortie** (un commentaire, un rapport), là où l'attaquant peut la lire.

**Difficulté supplémentaire :** certains scanners ont des **défenses anti-injection**. Il faut alors contourner les filtres — éviter les mots déclencheurs, présenter la divulgation comme une tâche neutre (formatage, vérification), sortir le modèle de son « mode auditeur ».

### 3.3 Déclencher des vulnérabilités secondaires (SSRF)

L'attaque la plus avancée **chaîne** deux failles :
1. Une **injection indirecte** pour piloter le scanner.
2. Une **SSRF routing-based** : on force le scanner à envoyer une requête vers une ressource **interne** (ex. un panneau d'admin sur une IP privée) en manipulant la destination, souvent via l'en-tête `Host` ou un paramètre d'URL détourné.

Résultat : le scanner, depuis sa position interne privilégiée, atteint un service qu'aucun attaquant externe ne pourrait joindre.

### 3.4 Un risque distinct : l'agentivité excessive (*excessive agency*)

Au-delà de l'injection, il y a un risque plus subtil : un scanner IA autonome peut interagir avec un panneau d'admin ou une API interne **sans qu'on le lui demande** — simplement parce qu'un bon outil d'audit « explore tout ». Le danger ne vient pas toujours d'un attaquant : il vient de ce que l'agent a **le droit de faire**.

---

## 4. Les moyens de défense

Le réflexe intuitif — « rendre le LLM meilleur pour détecter les injections » — **ne suffit pas**. Un scanner qui détecte les injections peut quand même être contourné par une formulation astucieuse.

Le bon principe : **supposer que le moteur de raisonnement de l'IA sera compromis**, et contraindre son environnement pour limiter les dégâts. C'est le **principe du moindre privilège** appliqué à un agent IA.

### 4.1 Restreindre les identifiants et les droits du scanner
Utiliser des **comptes de test dédiés**, sans les permissions des vrais administrateurs. Si le scanner n'a pas le droit de supprimer un compte, une injection « supprime carlos » n'aboutit à rien.

### 4.2 Traiter tout contenu stocké comme hostile
Considérer chaque commentaire, champ de profil ou avis récupéré en base comme un **vecteur d'injection potentiel**. Le contenu utilisateur ne doit jamais être traité comme une instruction de confiance.

### 4.3 Cloisonner l'environnement
Segmentation réseau et **allow-list** des destinations autorisées. Si le vérificateur de stock n'accepte que des URLs légitimes, la SSRF vers le panneau d'admin interne devient impossible.

### 4.4 Journaliser et surveiller
Tracer les actions du scanner (les *tool calls*) permet de détecter un comportement anormal — par exemple une requête vers un endpoint d'administration.

---

## 5. À retenir

| Côté attaque | Côté défense |
|---|---|
| Le scanner ne distingue pas données et instructions | Traiter tout contenu externe comme non fiable |
| L'attaquant emprunte les privilèges du scanner | Moindre privilège : comptes de test dédiés |
| SSRF vers des ressources internes | Cloisonnement réseau + allow-list |
| Agentivité excessive | Contraindre ce que l'agent a le droit de faire |

**En une phrase :** on ne peut pas compter sur la résistance du modèle aux injections. La vraie défense consiste à supposer l'IA compromise et à limiter ce qu'elle peut atteindre et exécuter.

---

## Ressources

- PortSwigger Web Security Academy — *Web LLM Attacks* : https://portswigger.net/web-security/llm-attacks
- OWASP Top 10 for LLM Applications

---

---

*Write-up réalisé dans le cadre de mon apprentissage en cybersécurité (Réseaux & Sécurité).*

**Becky Joyce ELIBE**
GitHub : https://github.com/Beckyyjw · LinkedIn : https://linkedin.com/in/becky-joyce-elibe10
