# AI Support Prototype: Chatwoot + n8n + Qdrant + Gemini

A local prototype of an AI layer for customer support. A customer chats through a website widget. An AI bot answers from your business documents and hands the chat to a human agent when it can't answer.

```
Website widget ─► Chatwoot ──webhook──► n8n ─┬─► Qdrant (document search)
                     ▲                       └─► Google Gemini (embeddings + answer)
                     └──── reply OR handoff ◄────┘
```

Everything runs in Docker on one laptop. No Chatwoot code is changed. The bot is a standard Chatwoot **Agent Bot**, and its logic lives in n8n workflows.

## What it does

- **Answers from documents.** Upload a `.txt` or `.pdf` through an n8n form. It is split into chunks, embedded with `gemini-embedding-001` and stored in Qdrant. The bot answers only from those chunks.
- **Hands off to a human.** If the answer isn't in the documents, or the customer asks for a person, the bot says so, opens the conversation for agents, adds a private note with the customer's question and adds the `ai-handoff` label.
- **Stays quiet after handoff.** The bot only replies while a conversation is `pending`, so it never talks over a human. It also ignores its own messages, so it can't loop.

## Stack

| Part | Choice |
|---|---|
| Support platform | Chatwoot `v4.19.1-ce` (prebuilt image) |
| AI logic | n8n (self-hosted) |
| Vector DB | Qdrant `v1.19.0`, collection `knowledge_base` |
| Embeddings | Gemini `gemini-embedding-001` |
| LLM | Gemini `gemini-3.5-flash-lite` |
| Channel | Chatwoot Website inbox |

## Repo layout

```
docker-compose.yml   all 6 services (rails, sidekiq, postgres, redis, n8n, qdrant)
.env.example         copy to .env and fill in secrets
workflows/           n8n workflow exports
  01-ingest.json       "Padhai": form upload → Qdrant
  02-answer.json       "Jawab + Handoff": webhook → AI agent → reply or handoff
docs/                dummy business documents (faq.txt, demo-offer.txt)
test-widget.html     a fake shop page that loads the Chatwoot widget
test-screens/        screenshots from the end-to-end tests
AGENTS.md            full build brief (plan, verified facts, rules)
PROGRESS.md          step-by-step build log with test results
```

## Run it

You need Docker Desktop and a free Gemini API key from https://aistudio.google.com.

1. `cp .env.example .env` and replace every `change_me` with a random value (for example `openssl rand -hex 32`).
2. Set up the database, then start everything:
   ```
   docker compose pull
   docker compose run --rm rails bundle exec rails db:chatwoot_prepare
   docker compose up -d
   ```
3. Chatwoot at http://localhost:3000: create the admin account, a **Website** inbox, and an **Agent Bot** with webhook URL `http://n8n:5678/webhook/chatwoot-bot`. Attach the bot to the inbox.
4. n8n at http://localhost:5678: create the owner account and three credentials: Google Gemini (PaLM) API, Qdrant (`http://qdrant:6333`, no key), and Header Auth named "Chatwoot Bot" (header `api_access_token`, value = the bot's access token).
5. Import both files from `workflows/`, select your credentials in each node, then activate **02 Jawab + Handoff**.
6. Run **01 Padhai** and upload `docs/faq.txt`.
7. In `test-widget.html`, put your inbox's website token. Serve the page with `python -m http.server 8080 --bind 127.0.0.1` and open http://localhost:8080/test-widget.html.

Ask "Refund kitne din mein milta hai?" for an answer from the FAQ, or "Kya aap mobile phone bechte ho?" for a handoff.

## Notes

- `SAFE_FETCH_ALLOW_PRIVATE_NETWORK=true` is required. Without it, Chatwoot refuses to send webhooks to the n8n container's private IP.
- Chatwoot's webhook timeout is 5 seconds. Right after n8n restarts, the first webhook can be slow; wait about 20 seconds before testing.
- Use dummy data only. The Gemini free tier may use the data it receives.
- This is a local prototype. It isn't hardened for production: there's no TLS, and the webhook signature isn't checked.
