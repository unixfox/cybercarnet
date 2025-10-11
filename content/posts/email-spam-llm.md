---
title: "Fighting Email Spam on your Mail Server with LLMs — Privately"
date: "2025-10-11"
description: "Use LLMs to reject Spam on your mail server using Rspamd and Ollama."
tags: ["email", "spam", "ai", "privacy", "ollama", "rspamd", "mailcow"]
ShowToc: true
---

Spam emails have significantly increased over the years. Nowadays, your email address is in all spammers lists. Spammers are better at SPF, DKIM, and DMARC than everyone else.[^1] Traditional protections don't work anymore.

Here is how to effectively reduce the spam on your own mail server by scanning your emails with local LLMs.

---

> - We will use [Rspamd](https://rspamd.com) with [Mailcow](https://docs.mailcow.email/), but this also works with a standalone Rspamd.
> - [Ollama](https://ollama.com) will be used for its simplicity, but this also works with any other OpenAI compatible API.

---

## Why did I start using LLMs to scan my emails?

I have been self-hosting my own mail server for 10 years. The default Rspamd configuration of Mailcow has always exceptionally blocked 95% of the spam.

But one year ago, I noticed the success rate of blocking those spams had reduced by a good margin. To the point I'm getting annoyed by spam again, something I only experienced when I used public email services like Outlook and Gmail.

Until I discovered the [GPT plugin](https://docs.rspamd.com/modules/gpt/) in Rspamd.

## Where is the private part? The GPT plugin recommends me to use OpenAI GPT o4-mini.

The GPT plugin works great by default with any mainstream LLMs like o4-mini or Gemini 2.5 flash. But I do not want to send my entire email to the cloud, much less to big companies like OpenAI or Google. Not knowing what they will do with my confidential info.

So I experimented with Ollama and its diverse number of open source LLMs. I tried a collection of spam and not-spam emails on Gemma, Qwen, LLama, and Mistral. And even distilled ones from mainstream LLMs, like Gemma 3 distilled with OpenAI o4 mini.

I was looking for a small LLM (size under 10GB) that could run on a moderately priced GPU. Especially because Rspamd requires all plugins to respond in under 30 seconds, so a small LLM gives a quick answer.

What worked great was a combination of Google Gemma 3 12B, a very detailed prompt (not the default prompt from Rspamd) and feeding the LLM with extra context from web searches.

I learned small LLMs tend to only work great on complicated tasks when you give them a detailed prompt of the task. I asked OpenAI o4 to write me a prompt, and it gave me a perfect prompt that worked in all cases, spam and not-spam classification.

> {{< collapse summary="Expand to read the prompt." >}}

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

And for the Web searches. This idea came up by thinking:

> What would I do if I had little knowledge about a company emailing me, and I wanted to fact-check if they were trustworthy?

This is especially important because small LLMs have less knowledge about the Web than a big LLM. Based on my testing, they were unable to correctly identify that the real estate company I rent my apartment from is legitimate, not a scam. Because they don't know about it, this company is too little known to have landed in their trained dataset.

For Web searches, you can use any search engine API. Proprietary API from Google, Brave, Bing. Or APIs that scrape public search engines like [SearXNG Search API](https://docs.searxng.org/dev/search_api.html).

I personally went with the "unofficial" API of [Mullvad Leta](https://leta.mullvad.net). This service fetches search results from Google or Brave. And Mullvad has a good track record for user privacy, and I wanted a reliable API. My mail server and my own Nextcloud are the two things that need to work reliably.

The Rspamd GPT plugin doesn't yet offer the ability to dynamically add more context from an external service, so I created a proxy that intercepts the request between the plugin and the Ollama API. This proxy adds the web searches in the context.

## The workflow in summary

1. If Rspamd has doubts if it's a spam or not. It sends a request to my proxy.
2. My proxy request from Mullvad Leta the search results for who sent the email and the links included in the email.
3. The proxy sends the request to Ollama API.
4. Ollama respond, the proxy forward the request to Rspamd.
5. Based on the score given by the LLM, Rspamd classify the email as spam or not.

## How to replicate my setup.

### Prerequisites

- [Mailcow](https://mailcow.email) installed.
- [Ollama](https://ollama.com) installed locally or on a remote server.
- Ollama need to run on a GPU, LLMs takes too much time to respond on a CPU.

### Instructions

#### A) Configure Ollama

I had good success with these LLMs: `gemma3:12b`, `llama3.3:70b` and `qwen3:32b`. I did stick with `gemma3:12b` which have the smallest size.

Prefetch the LLM: `ollama pull gemma3:12b`.

#### B) Configure GPT plugin in Rspamd in Mailcow

1. Go to the Mailcow installation folder: `/opt/mailcow-dockerized`.
2. Edit the file in `data/conf/rspamd/local.d/gpt.conf`.
3. Replace the file with this config file:

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

4. Restart rspamd with `docker compose restart rspamd-mailcow`

#### C) Configure the proxy

The program source code is located at https://github.com/unixfox/mailcow-rspamd-ollama.

1. Git clone the proxy to /opt with:  
   ```
   git clone https://github.com/unixfox/mailcow-rspamd-ollama.git /opt/mailcow-rspamd-ollama
   ```
2. Go to `/opt/mailcow-rspamd-ollama`.
3. Configure the Ollama API URL if you are using a remote ollama instance by changing the environment variable `OLLAMA_API`.
4. Run the program with `docker compose up -d`.

#### That's all.

You can now try if it works for you. My go-to website to test if the setup works is to register a dummy account on https://chatgpt.com. ChatGPT registration emails always have a score of 2 in Rspamd which results in triggering the GPT plugin.

Use the Rspamd administration page to see if the email was scanned by the LLM: https://yourmailcowdomain/rspamd/.

![regular](images/2025-07-19_19-49-rspamd-admin-gpt.png)

### What if I don't have a GPU powerful enough to run a local LLM?

The proxy works with any OpenAI compatible.

If you are conscious about privacy, you can use European providers like [Nebius AI Studio](https://studio.nebius.com) (Netherlands) or [Mistral AI API](https://mistral.ai) (France).

Or use a dedicated GPU in the cloud, where you pay per seconds/minutes of usage. [Runpod](https://www.runpod.io), [Fly.io](https://fly.io/gpu), [Vast.ai](https://vast.ai) or any other providers.

### Feedback after 6 months of using it

It works great. I have had very few false positives; I would say 5 out of the 2000 emails scanned by the program.

For the few false positives, they land in my spam folder so they don't get lost, and I can still read them. Rspamd scoring system is great; by default, if an email has a score between 6 and 15, it gets moved to the spam folder. Higher than that, the email is rejected.

In most cases, spam receives a score higher than 15, as the GPT plugin can assign up to 8 points for spam and phishing detection combined.

![regular](images/2025-10-11_10-53.png)

### Troubleshooting

#### Ollama API is too slow to respond for Rspamd

Increase all the plugins timeout to 60s by tuning this file `/opt/mailcow-dockerized/data/conf/rspamd/override.d/worker-normal.custom.inc`.

Change to `task_timeout = 60s;`.

[^1]: https://toad.social/@grumpybozo/114213600922816869 ([Web Archive](https://web.archive.org/web/20250409191904/https://toad.social/@grumpybozo/114213600922816869)) ([Hackernews comments](https://news.ycombinator.com/item?id=43468995))
