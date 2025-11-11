---
title: "Protégez votre serveur mail des spams grâce aux LLMs auto-hébergés"
date: "2025-10-11"
description: "Utilisez les LLM pour bloquer les spams sur votre serveur de messagerie à l'aide de Rspamd et Ollama."
tags: ["email", "spam", "ia", "vie privée", "ollama", "rspamd", "mailcow"]
ShowToc: true
---

Les spams sont devenus de plus en plus présents ces dernières années. Votre adresse email est désormais dans toutes les listes des spammeurs. Les spammeurs sont plus forts que les autres pour configurer correctement le SPF, DKIM et DMARC.[^1] Les protections classiques ne fonctionnent plus.

Voici comment vous protéger contre les spams en analysant vos emails avec des LLMs auto-hébergés.

---

> - [Rspamd](https://rspamd.com) sera utilisé avec [Mailcow](https://docs.mailcow.email/), mais cela fonctionne aussi avec un Rspamd tout seul/isolé.
> - [Ollama](https://ollama.com) sera utilisé pour sa simplicité, mais cela fonctionne aussi avec d'autres API compatible OpenAI API.

---

## Pourquoi j'ai commencé à analyser mes emails avec des LLMs ?

Cela fait 10 ans que j'héberge mon propre serveur mail. La configuration par défaut de Rspamd sur Mailcow a toujours fonctionné avec un taux de blocage des spams à 95%.

Mais il y a un an, j'ai remarqué que le blocage des spams était beaucoup moins effectif qu'auparavant. À tel point que j'étais de nouveau embêté par les spams, ce qui m'était uniquement arrivé lorsque j'utilisais des services publics comme Outlook ou Gmail.

Jusqu'à ce que je découvre le [plugin GPT](https://docs.rspamd.com/modules/gpt/) de Rspamd.

## Est-ce que mes données sont envoyées vers le cloud propriétaire ? Le plugin GPT me recommande d'utiliser le modèle d'OpenAI GPT o4-mini.

Le plugin GPT fonctionne par défaut très bien avec les modèles populaires comme o4-mini ou Gemini 2.5 flash. Mais je n'ai pas envie d'envoyer tous mes emails dans le cloud propriétaire, encore moins à OpenAI ou Google. Sans savoir ce qu'ils feront de mes données personnelles.

Du coup, j'ai expérimenté avec Ollama et ses nombreux LLMs open source. J'ai testé une série de spam et de courriers légitimes sur les modèles Gemma, Qwen, LLama, Mistral. Et même des LLMs distillés de LLMs populaires, comme Gemma 3 distillé avec le modèle OpenAI o4 mini.

Je recherchais un petit LLM (en dessous de 10GB) qui peut tourner confortablement sur un GPU de moyenne gamme. Principalement parce que Rspamd exige que tous les plugins répondent en moins de 30 secondes, ce qui fait que les petits LLM produisent des réponses plus rapidement.

Ce qui fonctionnait bien était une combinaison de Google Gemma 3 12B, un prompt détaillé (pas le prompt par défaut de Rspamd) et de lui donner du contexte supplémentaire de recherches Web.

J'ai appris que les petits LLms ne fonctionnent bien qu'avec des tâches compliquées lors que le prompt est très détaillé à propos de la tâche. Du coup, j'ai demandé à OpenAI o4 de m'écrire un prompt et voici un prompt qu'il m'a donné dont il fonctionne très bien pour toutes les situations :

> {{< collapse summary="Cliquez pour lire le prompt." >}}

You are an expert email evaluator analyzing messages for spam or malicious intent. Use the full content of the email, sender information, and web presence to assess its legitimacy.
Assumptions:
- DKIM authentication is valid; the sender address has not been spoofed.
- You will be provided with the sender domain's web presence for additional context.

Evaluation Criteria:
1. Domain legitimacy and sender identity (based on matching domain and content).
2. Language, tone, and structure: assess if it resembles common phishing or scam tactics.
3. External context: if the domain has a strong, legitimate online presence, this is a positive signal.
4. Check if the domain activities is related to the domain used to send the email.

Output exactly 3 lines:
1. Numeric spam/malicious probability from 0.00 to 1.00.
2. One-sentence justification based on the strongest risk signal (or clearest sign of legitimacy).
3. (Only if score > 0.5, the concern category) phishing, scam, malware, or marketing. Leave blank otherwise.

{{</ collapse >}}

Et pour les recherches Web. J'ai eu cette idée en me disant :

> Que ferais-je si je ne connaissais pas grand-chose à propos d'une entreprise qui m'envoie un e-mail et que je voulais vérifier si elle est digne de confiance ?

C'est particulièrement important, car les petits LLMs ont beaucoup moins de connaissances à propos du Web qu'un LLM plus large. D'après mes expérimentations, les LLMs n'arrivaient pas à correctement identifier l'agence immobilière à laquelle je loue mon appartement. Ils pensaient que c'était une arnaque. Parce qu'ils ne la connaissent pas, cette agence est peu connue pour faire part de leurs données d'entrainement.

Pour faire des recherches Web, vous pouvez utiliser n'importe quelle API de recherche. Des API propriétaires comme celle de Google, Brave ou Bing. Ou alors une API qui scrape les résultats de moteurs de recherche publics comme celle de [SearXNG Search API](https://docs.searxng.org/dev/search_api.html).

~~J'ai plutôt utilisé l'API non officielle de [Mullvad Leta](https://leta.mullvad.net). Ce service a comme source les résultats de Google ou Brave. Et Mullvad a fait ses preuves en matière de confidentialité des utilisateurs, et je voulais une API fiable. Mon serveur mail et mon Nextcloud sont les deux services qui doivent toujours bien fonctionner.~~

Depuis la fermeture de [Mullvad Leta](https://mullvad.net/en/blog/shutting-down-our-search-proxy-leta), je suis passé à la bibliothèque [ddgs](https://github.com/deedy5/ddgs) qui permet d'utiliser plusieurs moteurs de recherches (ceux qui respectent ma vie privée).

Le plugin GPT de Rspamd n'offre pas encore la possibilité d'ajouter du contexte supplémentaire d'un service externe, donc j'ai créé un proxy qui intercepte les requêtes entre le plugin et l'API de Ollama. Ce proxy ajoute au contexte les résultats de recherche Web.

## En résumé, le fonctionnement par étape

1. Si Rspamd a des doutes si l'email est du spam ou non. Il envoie une requête à mon proxy.
2. Mon proxy fait une requête sur le service Mullvad Leta avec des informations comme qui a envoyé l'email et les liens contenus dans l'email.
3. Le proxy envoie une requête à l'API de Ollama.
4. Ollama répond, le proxy redirige la réponse à Rspamd.
5. En fonction du score donné par le LLM, Rspamd classifie si l'email est du spam ou non.

## Guide d'installation

### Pré-requis

- [Mailcow](https://mailcow.email) d'installé.
- [Ollama](https://ollama.com) installé localement ou sur un serveur distant.
- Ollama a besoin d'utiliser un GPU parce que sur CPU cela prend trop de temps à traiter.

### Instructions

#### A) Configurer Ollama

Ces LLMs suivants fonctionnent très bien : `gemma3:12b`, `llama3.3:70b` and `qwen3:32b`. J'ai choisi `gemma3:12b` parce qu'il a la plus petite taille.

Récupèrer le LLM: `ollama pull gemma3:12b`.

#### B) Configurer le plugin GPT dans Rspamd dans Mailcow

1. Aller dans le dossier d'installation de Mailcow: `/opt/mailcow-dockerized`.
2. Ouvrir le fichier `data/conf/rspamd/local.d/gpt.conf`.
3. Replacer le fichier avec ce fichier de config:

```
enabled = true; # Enable the plugin
type = "openai"; # Supported types: openai, ollama
api_key = "dummy"
model = "gemma3:12b"
temperature = 0.0;
autolearn = false;
max_tokens = 1000; # Maximum tokens to generate
timeout = 120s; # Timeout for requests
prompt = "You are an expert email evaluator analyzing messages for spam or malicious intent. Use the full content of the email, sender information, and web presence to assess its legitimacy. \n Assumptions: \n - DKIM authentication is valid; the sender address has not been spoofed. \n - You will be provided with the sender domain's web presence for additional context. \n Evaluation Criteria: \n 1. Domain legitimacy and sender identity (based on matching domain and content). \n 2. Language, tone, and structure: assess if it resembles common phishing or scam tactics. \n 3. External context: if the domain has a strong, legitimate online presence, this is a positive signal. \n Output exactly 3 lines: \n 1. Numeric spam/malicious probability from 0.00 to 1.00, less than 0.25 is ham and more than 0.75 is spam (if you find a concern category it's a spam). \n 2. One-sentence justification based on the strongest risk signal (or clearest sign of legitimacy). \n 3. (Only if score > 0.5, the concern category) phishing, scam, malware, or marketing. Leave blank otherwise."
url = "http://rspamdgpt:8080/api/v1/chat/completions"; # URL for the API
allow_passthrough = false; # Check messages with passthrough result
allow_ham = false; # Check messages that are apparent ham (no action and negative score)
reason_header = "X-GPT-Reason"; # Add header with reason (null to disable)
symbols_to_except = { MAILCOW_BLACK = 1998, BAYES_SPAM = 0.9, MAILCOW_FUZZY_DENIED = 1, FUZZY_DENIED = 1, WHITELIST_SPF = -1, WHITELIST_DKIM = -1, WHITELIST_DMARC = -1, REPLY = -1, }
json = false; # Use JSON format for response
```

4. Redémarrer Rspamd avec `docker compose restart rspamd-mailcow`

#### C) Configurer le proxy

Le code source du programme se trouve dans https://github.com/unixfox/mailcow-rspamd-ollama.

1. Faire un git clone du proxy dans /opt:  
   ```
   git clone https://github.com/unixfox/mailcow-rspamd-ollama.git /opt/mailcow-rspamd-ollama
   ```
2. Aller dans le dossier `/opt/mailcow-rspamd-ollama`.
3. Configurer l'API de Ollama si vous utilisez une instance distante en changeant la variable d'environnement `OLLAMA_API`.
4. Exécuter le programme avec `docker compose up -d`.

#### Configuration terminée

Vous pouvez tester si cela fonctionne. Mon site préféré pour tester si le setup fonctionne est de s'inscrire sur ChatGPT : https://chatgpt.com. Les emails de ChatGPT ont toujours un score de 2 dans Rspamd, ce qui déclenche la vérification par le plugin GPT.

Utiliser la page d'administration de Rspamd pour vérifier que l'email a été analysé par le LLM : https://yourmailcowdomain/rspamd/.

![regular](images/2025-07-19_19-49-rspamd-admin-gpt.png)

### Que faire si je ne dispose pas d'un GPU suffisamment puissant pour exécuter un LLM local ?

Ce proxy fonctionne avec n'importe quel API compatible OpenAI API.

Si vous souhaitez faire attention à votre vie privée, vous pouvez utiliser un fournisseur Européen comme [Nebius AI Studio](https://studio.nebius.com) (Pays-Bas) ou [Mistral AI API](https://mistral.ai) (France).

Ou bien utiliser un GPU dédié dans le cloud, où vous paierez à l'usage par secondes/minutes d'utilisation. [Runpod](https://www.runpod.io), [Fly.io](https://fly.io/gpu), [Vast.ai](https://vast.ai) ou d'autres fournisseurs.

### Retour après 6 mois d'utilisation

Le setup fonctionne très bien. J'ai eu très peu de faux positifs. Environ 5 sur 2000 emails reçus et analysés par le programme.

Pour les quelques faux positifs, ils sont accessibles dans le dossier spam, donc je peux toujours les lire. Le système de score de Rspam est vraiment bon. Par défaut, si un email a un score entre 6 et 15, il est mis dans le dossier spam. Si le score est plus haut que 15, l'email est rejetté. 

Dans la plupart des cas, les spams ont un score supérieur à 15 parce que le plugin GPT peut donner jusqu'à 8 points pour la classification de spam et d'hameçonnage en même temps.

![regular](images/2025-10-11_10-53.png)

### Solutions à des problèmes

#### L'API Ollama est trop lente pour répondre à Rspamd.

Augmentez le délai d'expiration de tous les plugins à 60 secondes en modifiant ce fichier : `/opt/mailcow-dockerized/data/conf/rspamd/override.d/worker-normal.custom.inc`.

Changer à `task_timeout = 60s;`.

[^1]: https://toad.social/@grumpybozo/114213600922816869 ([Web Archive](https://web.archive.org/web/20250409191904/https://toad.social/@grumpybozo/114213600922816869)) ([Commentaires Hackernews](https://news.ycombinator.com/item?id=43468995))
